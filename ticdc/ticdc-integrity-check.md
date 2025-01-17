---
title: TiCDC 单行数据正确性校验
summary: 介绍 TiCDC 数据正确性校验功能的实现原理和使用方法。
---

# TiCDC 单行数据正确性校验

从 v7.1.0 开始，TiCDC 引入了单行数据正确性校验功能。该功能基于 Checksum 算法，校验一行数据从 TiDB 写入、通过 TiCDC 同步，到写入 Kafka 集群的过程中数据内容是否发生错误。TiCDC 数据正确性校验功能仅支持下游是 Kafka 的 Changefeed，目前支持 Avro 协议。

## 实现原理

在启用单行数据 Checksum 正确性校验功能后，TiDB 使用 CRC32 算法计算该行数据的 Checksum 值，并将其一并写入 TiKV。TiCDC 从 TiKV 读取数据，根据相同的算法重新计算 Checksum，如果该值与 TiDB 写入的值相同，则可以证明数据在 TiDB 至 TiCDC 的传输过程中是正确的。

TiCDC 将数据编码成特定格式并发送至 Kafka。Kafka Consumer 读取数据后，可以使用与 TiDB 相同的算法计算得到新的 Checksum，将此值与数据中携带的 Checksum 值进行比较，若二者一致，则可证明从 TiCDC 至 Kafka Consumer 的传输链路上的数据是正确的。

关于 Checksum 值的计算规则，请参考 [Checksum 计算规则](#checksum-计算规则)。

## 启用功能

TiCDC 数据正确性校验功能默认关闭，要使用该功能，请执行以下步骤：

1. 首先，你需要在上游 TiDB 中开启行数据 Checksum 功能 ([`tidb_enable_row_level_checksum`](/system-variables.md#tidb_enable_row_level_checksum-从-v710-版本开始引入))：

    ```sql
    SET GLOBAL tidb_enable_row_level_checksum = ON;
    ```

    上述配置仅对新创建的会话生效，因此需要重新连接 TiDB。

2. 在创建 Changefeed 的 `--config` 参数所指定的[配置文件中](/ticdc/ticdc-changefeed-config.md#ticdc-changefeed-配置文件说明)，添加如下配置：

    ```toml
    [integrity]
    integrity-check-level = "correctness"
    corruption-handle-level = "warn"
    ```

3. 当使用 Avro 作为数据编码格式时，你需要在 [`sink-uri`](/ticdc/ticdc-sink-to-kafka.md#sink-uri-配置-kafka) 中设置 [`enable-tidb-extension=true`](/ticdc/ticdc-sink-to-kafka.md#sink-uri-配置-kafka)。同时，为了防止数值类型在网络传输过程中发生精度丢失，导致 Checksum 校验失败，还需要设置 [`avro-decimal-handling-mode=string`](/ticdc/ticdc-sink-to-kafka.md#sink-uri-配置-kafka) 和 [`avro-bigint-unsigned-handling-mode=string`](/ticdc/ticdc-sink-to-kafka.md#sink-uri-配置-kafka)。下面是一个配置示例：

    ```shell
    cdc cli changefeed create --server=http://127.0.0.1:8300 --changefeed-id="kafka-avro-checksum" --sink-uri="kafka://127.0.0.1:9092/topic-name?protocol=avro&enable-tidb-extension=true&avro-decimal-handling-mode=string&avro-bigint-unsigned-handling-mode=string" --schema-registry=http://127.0.0.1:8081 --config changefeed_config.toml
    ```

    通过上述配置，Changefeed 会在每条写入 Kafka 的消息中携带该消息对应数据的 Checksum，你可以根据此 Checksum 的值进行数据一致性校验。

    > **注意：**
    >
    > 对于已有 Changefeed，如果未设置 `avro-decimal-handling-mode` 和 `avro-bigint-unsigned-handling-mode`，开启 Checksum 校验功能时会引起 Schema 不兼容问题。可以通过修改 Schema Registry 的兼容性为 `NONE` 解决该问题。详情可参考 [Schema 兼容性](https://docs.confluent.io/platform/current/schema-registry/fundamentals/avro.html#no-compatibility-checking)。

## 关闭功能

TiCDC 默认关闭单行数据的 Checksum 校验功能。若要在开启此功能后将其关闭，请执行以下步骤：

1. 首先，按照 [TiCDC 更新同步任务配置](/ticdc/ticdc-manage-changefeed.md#更新同步任务配置)的说明，按照 `暂停任务 -> 修改配置 -> 恢复任务` 的流程更新 Changefeed 的配置内容。在 Changefeed 的 `--config` 参数所指定的配置文件中调整 `[integrity]` 的配置内容为：

    ```toml
    [integrity]
    integrity-check-level = "none"
    corruption-handle-level = "warn"
    ```

2. 在上游 TiDB 中关闭行数据 Checksum 功能 ([`tidb_enable_row_level_checksum`](/system-variables.md#tidb_enable_row_level_checksum-从-v710-版本开始引入))，执行如下 SQL 语句：

    ```sql
    SET GLOBAL tidb_enable_row_level_checksum = OFF;
    ```

    上述配置仅对新创建的会话生效。在所有写入 TiDB 的客户端都完成数据库连接重建后，Changefeed 写入 Kafka 的消息中将不再携带该条消息对应数据的 Checksum 值。

## Checksum 计算规则

Checksum 计算算法的伪代码如下：

```
fn checksum(columns) {
    let result = 0
    for column in sort_by_schema_order(columns) {
        result = crc32.update(result, encode(column))
    }
    return result
}
```

* `columns` 应该按照 column ID 排序。在 Avro schema 中，各个字段已经按照 column ID 的顺序排序，因此可以直接按照此顺序排序 `columns`。

* `encode(column)` 函数将 column 的值编码为字节，编码规则取决于该 column 的数据类型。具体规则如下：

    * TINYINT、SMALLINT、INT、BIGINT、MEDIUMINT 和 YEAR 类型会被转换为 UINT64 类型，并按照小端序编码。例如，数字 `0x0123456789abcdef` 会被编码为 `hex'0x0123456789abcdef'`。
    * FLOAT 和 DOUBLE 类型会被转换为 DOUBLE 类型，然后转换为 IEEE754 格式的 UINT64 类型。
    * BIT、ENUM 和 SET 类型会被转换为 UINT64 类型。

        * BIT 类型按照二进制转换为 UINT64 类型。
        * ENUM 和 SET 类型按照其对应的 INT 值转换为 UINT64 类型。例如，`SET('a','b','c')` 类型 column 的数据值为 `'a,c'`，则该值将被编码为 `0b101`，即 `5`。

    * TIMESTAMP、DATE、DURATION、DATETIME、JSON 和 DECIMAL 类型会被转换为 STRING 类型，然后转换为字节。
    * CHAR、VARCHAR、VARSTRING、STRING、TEXT、BLOB（包括 TINY、MEDIUM 和 LONG）等字符类型，会直接使用字节。
    * NULL 和 GEOMETRY 类型不会被纳入到 Checksum 计算中，返回空字节。

## 基于 Golang 的 Avro 数据消费和 Checksum 计算过程解释

TiCDC 提供了基于 Golang 的 Checksum 计算过程，你可以参考该过程实现自己的 Checksum 计算逻辑。主要代码逻辑位于 TiCDC Avro Decoder 中实现的 [NextRowChangedEvent](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L100) 方法。该方法的具体工作过程如下：

1. 假设已经从 Kafka 读取到消息，并设置了 key 和 value 字段。分别对 key 和 value 进行解码操作，得到解码之后的数据值和 schema。具体的过程可以参考 [`decodeKey`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L395) 和 [`decodeValue`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L419) 方法。
2. 使用解码后的 key、value 和 schema 等内容，重新构建每一列的数据内容 `RowChangedEvent`。详情可见 [`assembleEvent`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L176) 方法。构建 `RowChangedEvent` 的过程如下：

    1. 获取 schema 中所有的 `fields` 内容，遍历 `fields` 中的每一个元素 `field` 以构建对应的列。其中 `fields` 已经按照 Column ID 排序。
    2. 利用 `field` 中包含的每一列的类型信息，重建每一列的 MySQL Type。并通过 keyMap 识别到 Handle Key 列，然后设置相应的 flag。
    3. valueMap 中的值需要经过 [`getColumnValue`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L299) 转换，这是因为在编码过程中，某些列允许为 NULL，此时会把 value 编码成一个 map。因此，解码时需要从 map 中获取具体的值，即 map 中的第一个元素。如果该列是 `mysql.TypeEnum` 或 `mysql.TypeSet` 类型，还需要映射到它们的数字形式表示上。
    4. 遍历完 `fields` 后，就获取了所有列的数据。对于 Delete 事件，将解码得到的列设置为 `PreColumns`；对于 Insert 和 Update 事件，将解码得到的列设置为 `Columns`。

Checksum 计算和校验的过程如下：

1. 调用 [`extractExpectedChecksum`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L281) 方法，获取期望的 Checksum 值。如果该方法返回 `false`，则说明该事件不需要进行 Checksum 校验，因为上游并没有发送 Checksum。这可能发生在 TiCDC 开启了 Checksum 但 TiDB 没有开启该功能时，或者当前事件发生在 Checksum 校验功能开启之前等场景。
2. 调用 [`calculateChecksum`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L461) 方法，遍历之前重建出来的所有列。利用 [buildChecksumBytes](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L482) 方法将每一列的 value 和 MySQL Type 编码为一个字节切片，然后使用该字节切片更新 Checksum 值。
3. 通过 [`verifyChecksum`](https://github.com/pingcap/tiflow/blob/eb04aecaf8e61f7f9d67597c2d2ef1f44583dd79/pkg/sink/codec/avro/decoder.go#L444) 方法，进行 Checksum 计算和校验。将步骤 2 计算的值与步骤 1 获取的期望值进行比较。如果不相等，则说明 Checksum 校验失败，数据可能存在损坏的情况。

> **注意：**
>
> - 开启 Checksum 校验功能后，DECIMAL 和 UNSIGNED BIGINT 类型的数据会被转换为字符串类型。因此在下游消费者代码中需要将其转换为对应的数值类型，然后进行 Checksum 相关计算。
> - Delete 事件只含有 Handle Key 列的内容，而 Checksum 是基于所有列计算的，所以 Delete 事件不参与到 Checksum 的校验中。
启动 broker 时在 `server.properties` 中禁用自动创建 topic：

```properties
# 确保在所有 Broker 的配置中设置此项
auto.create.topics.enable=false
```

另外可以在 broker 设置 topic 默认策略：

```properties
# 保证高可用, 消息至少同步指定副本数, 才认为生产者消息投递 topic 成功, 建议至少 2 个
min.insync.replicas=<num>
```

创建 Topic 基本示例：

```bash
kafka-topics.sh \
--bootstrap-server [<broker:port>...] \
--create \
--topic <topic_name> \
--partitions <num> \                   # 分区数根据吞吐量评估, 通常至少为 3
--replication-factor <num> \           # 消息同步副本数, 生产环境建议至少为 3 个
--config min.insync.replicas=<num>     # 保证高可用, 消息至少同步指定副本数, 才认为生产者消息投递 topic 成功, 建议至少 2 个
--config retention.ms=604800000 \      # 日志保留时常, -1 表示永久保留
--config segment.bytes=1073741824      # 单日志分段最大大小(1GB)
--config cleanup.policy=compact \      # 日志处理策略, 根据需求选择 delete 或 compact
--config unclean.leader.election.enable=false  # 禁止不完整副本成为Leader
```

常规业务（大多数业务）：

```bash
kafka-topics.sh --create \
    --topic user-activities \
    --partitions 3 \
    --replication-factor 3 \
    --config min.insync.replicas=2
```

关键数据（如金融、交易类），对数据一致性要求极高的场景：

```bash
kafka-topics.sh --create \
    --topic financial-transactions \
    --partitions 16 \
    --replication-factor 3 \
    --config min.insync.replicas=2 \
    --config cleanup.policy=compact \
    --config retention.ms=-1  # 永久保留
```

日志、监控等：

```bash
kafka-topics.sh --create \
    --topic application-logs \
    --partitions 6 \
    --replication-factor 2 \      # 可以接受2副本
    --config min.insync.replicas=1 \
    --config retention.ms=86400000  # 保留1天
```

# Topic 消息保留策略

Kafka 的日志由许多 segment 文件组成，生产者将消息写入 topic 后，消息不会立马删除，而是依据提供的保留策略根据条件慢慢清理。下面是主要的策略参数：

## retention.ms

这是最直接的保留时间。意思是：一条消息从写入开始，到达到 `retention.ms` 后，就允许 Kafka 将其删除。

注意是“允许”，不是“立刻删除”，只是被标记为可删除。真正的清理会在 log cleaner 或 log retention 线程运行时执行。

例如：

```bash
kafka-topics.sh --create \
    --topic <topic_name> \
    --config retention.ms=7天
```

那么消息大致会在 7 天后被标记为可清除。

|**Note**|
|:-------|
| `retention.ms` 的单位是 毫秒，这里设置 7天 只是便于理解。|

## retention.bytes

控制 topic 日志大小，如果 topic 所有 segment 累计大小超过 `retention.bytes`，Kafka 会从最老的 segment 往前删。

例如：

```bash
kafka-topics.sh --create \
    --topic <topic_name> \
    --config retention.bytes=1GB
```

当前 topic 日志有 1.2GB，旧的 segment 会被删除直到重新低于 1GB。

`retention.bytes` 和 `retention.ms` 谁先触发，就按谁来删除。

|**Note**|
|:-------|
| `retention.bytes` 的单位是 byte，这里设置 1GB 只是便于理解。|

## cleanup.policy

控制日志如何“老去”。

* delete：按时间/大小进行删除
* compact：按 key 去重，只保留每个 key 的最后一条消息

简单地说就是：

```
cleanup.policy=delete：按时间/空间删
cleanup.policy=compact：按 key 保留最新版本
```

另外，也可以设置成 `delete,compact` 的组合：

```bash
kafka-topics.sh --create \
    --topic <topic_name> \
    --config cleanup.policy=delete,compact
```

## delete.retention.ms

该参数是对 `cleanup.policy=compact` 的扩充，在 `cleanup.policy=delete` 模式下不起作用。

compact 模式运行一个特殊线程叫 Log Cleaner，它的任务是扫描日志，把所有旧的 key 替换掉，只保留每个 key 的最新值。

比如：

```
key=A, value=1
key=A, value=2
key=A, value=3
```

最终 compact 后只会留下：

```
key=A, value=3
```

要删除一条 key，Kafka 写入的不是“删掉这条”，而是：

key=A, value=null → 这叫 tombstone

null 值 = 删除标记，Kafka 看到它就知道你想删除 key=A。

delete.retention.ms → 控制 tombstone 保留多久

Kafka 不会看到 tombstone 就立即删对应的旧数据。它会先把 tombstone 保留 一段时间，让 consumer 有机会看到 key 真的被删除过。

这段“墓碑停留时间”就是 delete.retention.ms，默认一般是 86400000ms（24 小时）。

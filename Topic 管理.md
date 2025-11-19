
# 禁用自动创建 topic

如果当前运行的是 cluster 模式，在集群启动之前，需要将所有的选举节点都设置为紧张自动创建 topic。也就是在 `server.properties` 都做如下配置：

```properties
auto.create.topics.enable=false
```

所谓的选举节点，就是 `server.properties` 的指定 roles 为 controller 的节点（下面两种配置都表示 broker 是选举节点）：

```
process.roles=controller
process.roles=broker,controller
```

如果是 standalone 模式，只需要在 `server.properties` 中添加该配置即可。

# broker 设置副本默认策略

在创建 topic 时，为了保证高可用，通常会设置多个消息副本（`--replication-factor <num>`），也就是说消息写入 Leader 之后会继续将消息同步到其他副本。默认情况下，消息写入 Leader 副本成功之后就认为生产者将消息投递 topic 成功了。

而为了保证真正的高可用（防止消息丢失），在创建 topic 时通常还会指定最少同步副本数（`--config min.insync.replicas=<num>`）。该配置解决的问题是，当消息写入 Leader 副本之后，还要继续同步其他副本，只有当消息至少成功写入 `min.insync.replicas` 个副本（含 Leader）才认为消息写入 topic 成功。

为了防止创建 topic 时遗漏，我们可以直接在 broker 的配置文件 `server.properties` 中设置默认的最小同步副本数：

```properties
min.insync.replicas=<num>
```
当然，如果是 cluster 模式，需要在所有的 broker 节点都设置。在 broker 级别设置，主要是增加一层保险而已。

# 消息分区

partitions 指的是 topic 消息的分区数。分区数越多，并发能力越强，当然要根据业务评估。

replication-factor 指的是分区同步副本数。当 topic 根据 key 路由将消息写入某个分区，该分区就是 Leader 副本。该配置用于指定分区总副本（备份）数，除了 Leader 副本，其他都是 Follower 副本。生产者将消息投递到 topic 后，首先会将消息写入 Leader 副本，之后再同步到 Follower 副本。

min.insync.replicas 指的是最少同步副本数。默认情况下，生产者将消息投递到 topic，写入 Leader 副本后就认为消息投递成功了，至于 Follower 副本有没有同步成功并不关心。该配置用于指定当消息写入分区后，最少同步指定个副本（含 Leader 副本），才认为消息投递 topic 成功，依此来达到真正的高可用。

需要注意的是，min.insync.replicas 设置的最少同步副本数不能大于 replication-factor。

# 创建 topic 语法

通用的创建 topic 命令参数如下，可根据需要自行指定：

```bash
kafka-topics.sh \
--bootstrap-server [<broker:port>...] \
--create \
--topic <topic_name> \
--partitions <num> \                   # 分区数, 根据吞吐量评估, 通常至少为 3
--replication-factor <num> \           # 每个分区同步副本数, 生产环境建议至少为 3 个
--config min.insync.replicas=<num>     # 消息最小同步分区副本数, 保证高可用, 消息至少同步指定个副本, 才认为生产者消息投递 topic 成功, 建议至少 2 个
--config retention.ms=604800000 \      # 日志保留时长(7天), -1 表示永久保留
--config segment.bytes=1073741824      # 单日志分段最大大小(1GB)
--config cleanup.policy=compact \      # 日志处理策略, 根据需求选择 delete 或 compact
--config unclean.leader.election.enable=false  # 禁止不完整副本成为Leader
```

下面是一些不同场景的推荐设置。

- **常规业务（大多数业务）：**

```bash
kafka-topics.sh --create \
    --topic user-activities \
    --partitions 3 \
    --replication-factor 3 \
    --config min.insync.replicas=2
```

-  **关键数据（如金融、交易类），对数据一致性要求极高的场景：**

```bash
kafka-topics.sh --create \
    --topic financial-transactions \
    --partitions 16 \
    --replication-factor 3 \
    --config min.insync.replicas=2 \
    --config cleanup.policy=compact \
    --config retention.ms=-1  # 永久保留
```

- **日志、监控类等：**

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

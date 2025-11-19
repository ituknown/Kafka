
```bash
$ mkdir standalone-cluster

.
├── broker_1
│   ├── broker.properties
│   └── data
├── broker_2
│   ├── broker.properties
│   └── data
└── broker_3
    ├── broker.properties
    └── data
```


```properties
# The role of this server. Setting this puts us in KRaft mode
process.roles=broker,controller

# 是否允许自动创建 topic
auto.create.topics.enable=false

# 节点ID
node.id=1

# 控制节点, 用于告诉 Broker 首次启动时去哪里获取集群元数据信息. 
# 等同于 --bootstrap-server=
#
# 这里使用单机部署集群, 所以直接使用 localhost.
# 如果是多机器部署, 一定要将 localhost 替换为机器可对外访问的 ip 地址
controller.quorum.bootstrap.servers=localhost:19093,localhost:29093,localhost:39093

# 真正的控制节点
#
# 这里使用单机部署集群, 所以直接使用 localhost.
# 如果是多机器部署, 一定要将 localhost 替换为机器可对外访问的 ip 地址
controller.quorum.voters=1@localhost:19093,2@localhost:29093,3@localhost:39093

# 监听本机端口
listeners=PLAINTEXT://0.0.0.0:19092,CONTROLLER://0.0.0.0:19093

# 对外开放地址
#
# 这里使用单机部署集群, 所以直接使用 localhost.
# 如果是多机器部署, 一定要将 localhost 替换为机器可对外访问的 ip 地址
advertised.listeners=PLAINTEXT://localhost:19092,CONTROLLER://localhost:19093

# 日志输入地址
# 推荐使用绝对路径, 别使用相对路径
# 如果配置为绝对路径, 输出的日志地址会是基于执行命令的相对路径
log.dirs=/usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_1/data
```


```bash
$ bin/kafka-storage.sh random-uuid
i1KwsyLMSr6-Mfx6deLpkg
```


```bash
$ bin/kafka-storage.sh format \
--clusster-id i1KwsyLMSr6-Mfx6deLpkg \
--config standalone-cluster/broker_1/broker.properties
```

执行成功，输出结果：

```
Formatting metadata directory /usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_1/data with metadata.version 4.1-IV1.
```

```bash
$ ls standalone-cluster/broker_1/data/
bootstrap.checkpoint  meta.properties
```

```
$ bin/kafka-server-start.sh standalone-cluster/broker_1/broker.properties

...
[2025-11-19 10:38:27,861] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:38:27,862] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
```

```bash
$ bin/kafka-server-start.sh standalone-cluster/broker_2/broker.properties

...
[2025-11-19 10:37:58,236] INFO [BrokerServer id=2] Waiting for all of the authorizer futures to be completed (kafka.server.BrokerServer)
[2025-11-19 10:37:58,236] INFO [BrokerServer id=2] Finished waiting for all of the authorizer futures to be completed (kafka.server.BrokerServer)
[2025-11-19 10:37:58,236] INFO [BrokerServer id=2] Waiting for all of the SocketServer Acceptors to be started (kafka.server.BrokerServer)
[2025-11-19 10:37:58,236] INFO [BrokerServer id=2] Finished waiting for all of the SocketServer Acceptors to be started (kafka.server.BrokerServer)
[2025-11-19 10:37:58,236] INFO [BrokerServer id=2] Transition from STARTING to STARTED (kafka.server.BrokerServer)
[2025-11-19 10:37:58,237] INFO Kafka version: 4.1.1 (org.apache.kafka.common.utils.AppInfoParser)
[2025-11-19 10:37:58,237] INFO Kafka commitId: be816b82d25370ce (org.apache.kafka.common.utils.AppInfoParser)
[2025-11-19 10:37:58,237] INFO Kafka startTimeMs: 1763519878236 (org.apache.kafka.common.utils.AppInfoParser)
[2025-11-19 10:37:58,238] INFO [KafkaRaftServer nodeId=2] Kafka Server started (kafka.server.KafkaRaftServer
```

```bash
$ bin/kafka-server-start.sh standalone-cluster/broker_3/broker.properties
```

```plaintext
...
[2025-11-19 10:40:58,892] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:58,892] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:59,893] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:59,893] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:41:02,532] INFO [RaftManager id=1] Updated in-memory voters from VoterSet(voters={1=VoterNode(voterKey=ReplicaKey(id=1, directoryId=eep6_l2c2cfyygUhb3YTtA), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:19093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 2=VoterNode(voterKey=ReplicaKey(id=2, directoryId=yB30f_T5xHVSMlXaxIvB0Q), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:29093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 3=VoterNode(voterKey=ReplicaKey(id=3, directoryId=<undefined>), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/127.0.0.1:39093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:0])}) to VoterSet(voters={1=VoterNode(voterKey=ReplicaKey(id=1, directoryId=eep6_l2c2cfyygUhb3YTtA), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:19093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 2=VoterNode(voterKey=ReplicaKey(id=2, directoryId=yB30f_T5xHVSMlXaxIvB0Q), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:29093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 3=VoterNode(voterKey=ReplicaKey(id=3, directoryId=L15uNU9Ut5zR_yI7uYz4vQ), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:39093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1])}) (org.apache.kafka.raft.internals.UpdateVoterHandler)
```


```bash
bin/kafka-topics.sh \
--bootstrap-server localhost:19092,localhost:29092 \
--create \
--topic order.paid \
--partitions 3 \
--replication-factor 3 \
--config min.insync.replicas=3 \
--config cleanup.policy=delete \
--config retention.ms=2592000000 \
--config unclean.leader.election.enable=false
```

输出结果：

```plaintext
WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.
Created topic order.paid. <== topic 创建成功
```

前面的 WARNING 并不是错误，而是 KAFKA 友善的提醒你：在创建 topic 时不要混用 `.` 和 `-`。

这事源自 Kafka 的度量指标（metrics）名字会把 topic 名嵌进去，而早期某些系统会把 `.` 和 `_` 都当成同一个分隔符。

也就是说创建 topic 时 `order.paid` 和 `order_paid` 可能会生成相同的 metric 名，造成“撞名”。这并不是什么错误，仅仅只是友善的提示你不要同时混用 `.` 和 `-`，尽量保持统一的命名规范。

查看已创建的 topic：

```bash
$ bin/kafka-topics.sh \
--bootstrap-server localhost:19092,localhost:29092 \
--list

order.paid
```

查看 topic 相信信息：

```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:19092,localhost:29092 --topic order.paid --describe

Topic: order.paid	TopicId: tVFQoD0UR4CvWrIgLU0bDA	PartitionCount: 3	ReplicationFactor: 3	Configs: min.insync.replicas=3,cleanup.policy=delete,segment.bytes=1073741824,retention.ms=2592000000,unclean.leader.election.enable=false
	Topic: order.paid	Partition: 0	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2	Elr: 	LastKnownElr:
	Topic: order.paid	Partition: 1	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3	Elr: 	LastKnownElr:
	Topic: order.paid	Partition: 2	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1	Elr: 	LastKnownElr:
```

删除 topic：

```bash
$ bin/kafka-topics.sh --bootstrap-server localhost:19092,localhost:29092 --delete --topic order.paid
```

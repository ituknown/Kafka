
先到官网下载页面（[https://kafka.apache.org/downloads](https://kafka.apache.org/downloads)）下载想要部署的版本，这里以最新版（v4.1.1）为例：

```bash
$ wget https://dlcdn.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz
```

需要强调一点，kafka 与 JDK 版本有要求。如果机器安装的 JDK 不满足 kafka 运行要求，将无法正常运行，具体可以查看官方文档说明。

kafka 在各自的版本文档中，都有对 JDK 要求的详细说明，比如最新版（v4.1.1）对 JDK 要求说明：[https://kafka.apache.org/documentation/#java](https://kafka.apache.org/documentation/#java)（所以我这里就按照要求使用 JDK21）。

kafka 下载成功后，开始配置环境变量：

```bash
# Java
export JAVA_HOME=/usr/local/lib/java/jdk21
export PATH=$JAVA_HOME/bin:$PATH

# Kafka
export KAFKA_HOME=/usr/local/lib/kafka/kafka_4_1_1
export PATH=$KAFKA_HOME/bin:$PATH
```

正常情况下，都是在多机器部署集群，每台机器都是一个 broker 节点。但是因为我只有一台机器，所以我就通过不同的配置文件来实现单机部署集群。

在任意目录下创建几个（集群节点建议最少为 3 个）broker 文件夹，每个文件夹都作为一个 broker 节点使用，并在 broker 下创建一个 data 目录和 broker.properties 配置文件

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

**Note：** broker.properties 无需手动创建，直接将 KAFKA_HOME 下的配置文件拷贝过来即可。

```bash
$ cp $KAFKA_HOME/config/broker.properties broker_1/
$ cp $KAFKA_HOME/config/broker.properties broker_2/
$ cp $KAFKA_HOME/config/broker.properties broker_3/
```

之后修改 broker.properties 配置文件。

**broker_1 配置文件：**

```properties
# 节点角色
# - controller: 元数据节点
# - broker: 数据存储节点
#
# 这里由于机器原因, 每个节点都将承担两种角色使用.
#
# 如果机器足够多, 建议将 controller 和 broker 分开部署(尤其是生产环境).
# controller 节点推荐至少部署 3 个
# broker 节点也建议至少部署 3 个
process.roles=broker,controller

# 是否允许自动创建 topic
# 如果想早点下班, 生产环境一定要设置 false
auto.create.topics.enable=false

# 节点ID
# 集群中每个节点 ID 都是唯一的, 建议从第一个节点自增使用
node.id=1

# 将当前节点注册集群到集群
# 用于告诉 broker 首次初始化去哪里获取集群元数据信息
#
# 这里建议将要所有 controller 节点都配置上去, 实际只要配置任意一个能正常通信的 controller 节点即可(主要是防止意外情况)
#
# 这里使用单机部署集群, 所以直接使用 localhost.
# 如果是多机器部署, 一定要将 localhost 替换为机器可对外访问的 ip 地址
controller.quorum.bootstrap.servers=localhost:19093,localhost:29093,localhost:39093

# 配置真正的 controller 节点(选举节点)
# controller.quorum.bootstrap.servers 只是初始化使用, 该配置才是用于指定集群运行时真正的选举节点
# 一定要将所有 controller 节点都配置上去!
# 另外配置格式为: 节点ID@IP:PORT
#
# 这里使用单机部署集群, 所以直接使用 localhost.
# 如果是多机器部署, 一定要将 localhost 替换为机器可对外访问的 ip 地址
controller.quorum.voters=1@localhost:19093,2@localhost:29093,3@localhost:39093

# 监听本机端口通信协议
#
# 如果当前节点仅作为 broker 使用, 只需要配置一个 PLAINTEXT 即可
# 如果当前节点仅作为 controller 使用, 只需要配置一个 CONTROLLER 即可
# 但是如果当前节点同时承担两种角色, 那这里就要配置两个, 注意端口号不要冲突
#
# 除了这里列出的两种通信协议, 还可以配置 SSL 通信协议(端口不能被占用)
# 具体可查询文档对 listener.security.protocol.map 配置项的说明
listeners=PLAINTEXT://0.0.0.0:19092,CONTROLLER://0.0.0.0:19093

# 对外开放地址
#
# 每个通信协议真正用于可被外部访问的地址
# 一定要设置为可被外部方式的地址(IP/域名)
advertised.listeners=PLAINTEXT://172.21.11.1:19092,CONTROLLER://172.21.11.1:19093

# 数据写入目录
#
# 最好使用机器绝对路径, 别使用相对路径
# 如果使用相对路径, 数据将写入执行命令所在目录的相对目录
log.dirs=/usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_1/data
```

**broker_2 配置文件：**

```properties
process.roles=broker,controller
auto.create.topics.enable=false

node.id=2

controller.quorum.bootstrap.servers=localhost:19093,localhost:29093,localhost:39093
controller.quorum.voters=1@localhost:19093,2@localhost:29093,3@localhost:39093

listeners=PLAINTEXT://0.0.0.0:29092,CONTROLLER://0.0.0.0:29093
advertised.listeners=PLAINTEXT://172.21.11.1:29092,CONTROLLER://172.21.11.1:29093

log.dirs=/usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_2/data
```

**broker_3 配置文件：**

```properties
process.roles=broker,controller
auto.create.topics.enable=false

node.id=3

controller.quorum.bootstrap.servers=localhost:19093,localhost:29093,localhost:39093
controller.quorum.voters=1@localhost:19093,2@localhost:29093,3@localhost:39093

listeners=PLAINTEXT://0.0.0.0:39092,CONTROLLER://0.0.0.0:39093
advertised.listeners=PLAINTEXT://172.21.11.1:39092,CONTROLLER://172.21.11.1:39093

log.dirs=/usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_3/data
```

配置文件都调整完成后，就可以生成 cluster id 了。只需要生成一次即可，实际可使用任意一台几点生成：

```bash
$ bin/kafka-storage.sh random-uuid

i1KwsyLMSr6-Mfx6deLpkg # cluster id
```

集群ID（cluster）生成成功后，就可以初始化 broker 元数据了：

- broker_1：

```bash
$ bin/kafka-storage.sh format \
--clusster-id i1KwsyLMSr6-Mfx6deLpkg \
--config standalone-cluster/broker_1/broker.properties
```

如果执行成功，会输出类似如下结果：

```
Formatting metadata directory /usr/local/lib/kafka/kafka_4_1_1/standalone-cluster/broker_1/data with metadata.version 4.1-IV1.
```

并且 data 目录下会有两个文件：

```bash
$ ls standalone-cluster/broker_1/data/
bootstrap.checkpoint  meta.properties
```

其他两个 broker 也执行同样操作：

```bash
# broker_2
$ bin/kafka-storage.sh format \
--clusster-id i1KwsyLMSr6-Mfx6deLpkg \
--config standalone-cluster/broker_2/broker.properties

# broker_3
$ bin/kafka-storage.sh format \
--clusster-id i1KwsyLMSr6-Mfx6deLpkg \
--config standalone-cluster/broker_3/broker.properties
```

所有准备工作都完成后，就可以启动节点了：

```bash
$ bin/kafka-server-start.sh standalone-cluster/broker_1/broker.properties
```

或加上 -daemon 参数后台运行：

```bash
$ bin/kafka-server-start.sh -daemon standalone-cluster/broker_1/broker.properties
```

因为演示使用，前台运行即可。

输出示例：

```
...
[2025-11-19 10:38:27,861] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:38:27,862] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
```

现在只启动一个 broker 节点，会提示其他节点还没找到。继续启动 broker_2：

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

此时 broker_1 的输出日志信息就变了：

```plaintext
...
[2025-11-19 10:40:58,892] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:58,892] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:59,893] INFO [RaftManager id=1] Node 3 disconnected. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:40:59,893] WARN [RaftManager id=1] Connection to node 3 (localhost/127.0.0.1:39093) could not be established. Node may not be available. (org.apache.kafka.clients.NetworkClient)
[2025-11-19 10:41:02,532] INFO [RaftManager id=1] Updated in-memory voters from VoterSet(voters={1=VoterNode(voterKey=ReplicaKey(id=1, directoryId=eep6_l2c2cfyygUhb3YTtA), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:19093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 2=VoterNode(voterKey=ReplicaKey(id=2, directoryId=yB30f_T5xHVSMlXaxIvB0Q), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:29093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 3=VoterNode(voterKey=ReplicaKey(id=3, directoryId=<undefined>), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/127.0.0.1:39093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:0])}) to VoterSet(voters={1=VoterNode(voterKey=ReplicaKey(id=1, directoryId=eep6_l2c2cfyygUhb3YTtA), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:19093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 2=VoterNode(voterKey=ReplicaKey(id=2, directoryId=yB30f_T5xHVSMlXaxIvB0Q), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:29093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1]), 3=VoterNode(voterKey=ReplicaKey(id=3, directoryId=L15uNU9Ut5zR_yI7uYz4vQ), listeners=Endpoints(endpoints={ListenerName(CONTROLLER)=localhost/<unresolved>:39093}), supportedKRaftVersion=SupportedVersionRange[min_version:0, max_version:1])}) (org.apache.kafka.raft.internals.UpdateVoterHandler)
```

继续启动 broker_3：

```bash
$ bin/kafka-server-start.sh standalone-cluster/broker_3/broker.properties
```

正常启动，就大功告成了！

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

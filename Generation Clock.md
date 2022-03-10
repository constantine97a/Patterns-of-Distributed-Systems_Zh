# Generation Clock

一个单调递增的数字，表示服务器的代数。又名：次、时代和世代。

[TOC]

## 问题

​	在领导者和追随者设置中，领导者可能会暂时与追随者断开连接。 领导进程中可能存在*垃圾收集暂停*，或者*临时网络中断*导致领导者与追随者断开连接。 这种情况下*leader*进程还在运行，在暂停或网络中断结束后，它会尝试向*follower*发送复制请求。 这很危险，因为同时集群的其余部分可能已经选择了一个新的领导者并接受了来自客户端的请求。 对于集群的其余部分来说，检测来自旧领导者的任何请求非常重要。 旧领导者本身也应该能够检测到它暂时与集群断开连接，并采取必要的纠正措施来退出领导层。

```asciiarmor
生成时钟模式是 Lamport 时间戳的一个示例：一种用于确定跨一组进程的事件顺序的简单技术，而不依赖于系统时钟。 每个进程维护一个整数计数器，该计数器在进程执行的每个操作后递增。 每个进程还会将此整数与进程交换的消息一起发送给其他进程。 接收消息的进程通过获取它自己的计数器和消息的整数值之间的最大值来设置它自己的整数计数器。 这样，任何进程都可以通过比较相关的整数来确定哪个动作发生在另一个之前。 如果消息是在进程之间交换的，那么跨多个进程的操作也可以进行比较。 可以以这种方式比较的行为被称为“因果关系”。
```



## 解决方案

​	保持一个单调递增的数字表示服务器的世代。 每次发生新的领导者选举时，都应该以增加代数为标志。 该生成需要在服务器重新启动后可用，因此它与预写日志（WAL）中的每个条目一起存储。 如 High-Water Mark 中所述，追随者使用此信息在其日志中查找冲突条目。

​	在启动时，服务器从日志中读取最后一个已知的代。

class ReplicatedLog…

```java
this.replicationState = new ReplicationState(config, wal.getLastLogEntryGeneration());
```

每次有新的领导者选举时，领导者和追随者服务器都会增加一代。

class ReplicatedLog…

```java
private void startLeaderElection() {
    replicationState.setGeneration(replicationState.getGeneration() + 1);
    registerSelfVote();
    requestVoteFrom(followers);
}
```

​	服务器将生成作为投票请求的一部分发送到其他服务器。 这样，在一次成功的领导者选举之后，所有的服务器都具有同一代。

follower (class ReplicatedLog...)

```java

  private void becomeFollower(int leaderId, Long generation) {
      replicationState.reset();
      replicationState.setGeneration(generation);
      replicationState.setLeaderId(leaderId);
      transitionTo(ServerRole.FOLLOWING);
  }
```

​	此后，领导者在发送给追随者的每个请求中都包含代数。 它包含在每个 [HeartBeat]() 消息以及发送给追随者的复制请求中。

领导者将[代数]()与其预写日志中的每个条目一起保存

leader (class ReplicatedLog...)

```java

  Long appendToLocalLog(byte[] data) {
      Long generation = replicationState.getGeneration();
      return appendToLocalLog(data, generation);
  }

  Long appendToLocalLog(byte[] data, Long generation) {
      var logEntryId = wal.getLastLogIndex() + 1;
      var logEntry = new WALEntry(logEntryId, data, EntryType.DATA, generation);
      return wal.writeEntry(logEntry);
  }
```

​		这样，它也作为Leader和Followers复制机制的一部分持久化在follower日志中

​		如果追随者从被废黜的领导人那里得到消息，追随者可以分辨出来，因为它的世代太低了。 跟随者然后回复失败响应。

follower (class ReplicatedLog...)

```java
  Long currentGeneration = replicationState.getGeneration();
  if (currentGeneration > request.getGeneration()) {
      return new ReplicationResponse(FAILED, serverId(), currentGeneration, wal.getLastLogIndex());
  }
```

​		当一个领导者收到这样的失败响应时，它就会成为一个追随者，并期待与新领导者的沟通。

Old leader (class ReplicatedLog...)

```java
  if (!response.isSucceeded()) {
      if (response.getGeneration() > replicationState.getGeneration()) {
          becomeFollower(LEADER_NOT_KNOWN, response.getGeneration());
          return;
      }
```

​	考虑以下示例。 在三服务器集群中，*leader1* 是现有的*leader*。 集群中所有服务器的世代为1。*Leader1*向*Follower*发送连续的心跳。 *Leader1* 有很长的垃圾收集暂停，比如 5 秒。 追随者没有得到心跳，超时选举新的领导者。 新的领导者将世代递增到 2。垃圾收集暂停结束后，领导者 1 继续向其他服务器发送请求。 位于第 2 代的追随者和新领导者拒绝请求并与第 2 代一起发送失败响应。领导者1 处理失败响应并降级成为追随者，并更新为第 2 代。

![img](.\images\generation1.png)

![img](.\images\generation2.png)

## 例子
### Raft

​	Raft 使用 Term 的概念来标记领导者的生成。

### Zab

​	在 Zookeeper 中，每个事务 id 都会维护一个 epoch 编号。所以在 Zookeeper 中持久化的每一笔交易都有一个以 epoch 为标志的世代。

### Cassandra

​	在 [Cassandra](http://cassandra.apache.org/) 中，每台服务器都存储一个世代编号，每次服务器重新启动时该编号都会递增。generation 信息保存在系统密钥空间中，并作为*gossip*消息的一部分传播到其他服务器。接收*gossip*消息的服务器然后可以比较它知道的世代值和*gossip*消息中的世代值。如果 gossip 消息中的生成更高，则它知道服务器已重新启动，然后丢弃它为该服务器维护的所有状态并请求新状态。

### Epoch in Kafka

​	在 [Kafka](https://kafka.apache.org/)中，每次为 kafka 集群选择新的 Controller 时，都会创建一个 epoch 编号并将其存储在 Zookeeper 中。 epoch 包含在从Controller 发送到集群中其他服务器的每个请求中。维护另一个称为 [LeaderEpoch](https://cwiki.apache.org/confluence/display/KAFKA/KIP-101+-+Alter+Replication+Protocol+to+use+Leader+Epoch+rather+than+High+Watermark+for+Truncation)的 epoch，以了解分区的追随者是否落后于其[High-Water Mark](https://martinfowler.com/articles/patterns-of-distributed-systems/high-watermark.html).。


# 复制的日志

通过使用复制到所有集群节点的预写日志(WAL)来保持多个节点的状态同步。

[TOC]



## 问题

​		当多个节点共享一个状态时，需要同步状态。 所有集群节点都需要就相同的状态达成一致，即使某些节点崩溃或断开连接也是如此。 这需要就每个状态更改请求达成共识。

​		但就个人要求达成共识是不够的。 每个副本也需要以相同的顺序执行请求，否则不同的副本可能会进入不同的最终状态，即使它们对单个请求达成共识。



## 解决方案

```asciiarmor
失败假设
	根据失败假设，使用不同的算法在日志条目上建立共识。最常用的假设是碰撞故障。使用碰撞故障，当集群节点出现故障时，它会停止工作。一个更复杂的故障假设是拜占庭故障。对于拜占庭故障，故障集群节点可以任意行为。它们可能被保持节点正常运行但故意发送带有错误数据的请求或响应的对手节点控制，例如欺诈性交易以窃取资金。
	大多数企业系统，如数据库、消息代理，甚至企业区块链产品（如超级账本结构）都承担崩溃故障。因此，几乎总是使用像 Raft 和 Paxos 这样的基于崩溃故障假设的共识算法。
	像 pbft 这样的算法用于需要允许拜占庭故障的系统。虽然 pbft 算法以类似的方式使用 log，但为了容忍拜占庭故障，它需要三个阶段执行和 3f + 1 的仲裁，其中 f 是容忍故障的数量。
```

​		集群节点维护一个预写日志。每个日志条目存储共识所需的状态以及用户请求。他们协调建立对日志条目的共识，以便所有集群节点具有完全相同的预写日志。然后请求按照日志顺序执行。因为所有集群节点都同意每个日志条目，所以它们以相同的顺序执行相同的请求。这确保了所有集群节点共享相同的状态。

使用 Quorum 的容错共识构建机制需要两个阶段。

* 建立Generation Clock并了解在先前法定人数（Quorum）中复制的日志条目的阶段。

* 在所有集群节点上复制请求的阶段。

​		为每个状态更改请求执行两个阶段效率不高。所以集群节点在启动时选择一个领导者。领导者选举阶段建立[Generation Clock](https://martinfowler.com/articles/patterns-of-distributed-systems/generation.html)并检测前一个 法定人数 中的所有日志条目。 （前一个领导者可能已经复制到大多数集群节点的条目。）一旦有一个稳定的领导者，只有领导者协调复制。客户与领导沟通。领导者将每个请求添加到日志中，并确保将其复制到所有追随者上。*一旦日志条目成功复制到大多数追随者，就会达成共识*。这样，当有一个稳定的领导者时，每个状态更改操作只需要一个阶段执行来达成共识。

### Multi-Paxos and Raft

Multi-Paxos 和 Raft 是最流行的实现复制日志的算法。 Multi-Paxos 只是在学术论文中模糊描述。 Spanner 和 Cosmos DB 等云数据库使用 Multi-Paxos，但实现细节没有很好的文档记录。 Raft 非常清楚地记录了所有实现细节，并且是大多数开源系统的首选实现选择，尽管 Paxos 及其变体在学术界讨论得更多。

以下部分描述了 Raft 如何实现复制日志。

### Replicating client requests

![img](.\images\raft-replication.png)

​																									*Figure 1: Replication*

对于每个日志条目，领导者将其附加到其本地预写日志中，然后将其发送给所有追随者。

**leader (class ReplicatedLog...)**

```java
  private Long appendAndReplicate(byte[] data) {
      Long lastLogEntryIndex = appendToLocalLog(data);
      replicateOnFollowers(lastLogEntryIndex);
      return lastLogEntryIndex;
  }


  private void replicateOnFollowers(Long entryAtIndex) {
      for (final FollowerHandler follower : followers) {
          replicateOn(follower, entryAtIndex); //send replication requests to followers
      }
  }
```

​	追随者处理复制请求并将日志条目附加到其本地日志中。 在成功附加日志条目后，它们会以他们拥有的最新日志条目的索引来响应领导者。 该响应还包括服务器的当前生成时钟。

​	追随者还会检查条目是否已经存在，或者是否存在正在复制的条目之外的条目。 它忽略已经存在的条目。 但是，如果有来自不同代的条目，它们会删除冲突的条目。

**follower (class ReplicatedLog...)**

```java
  void maybeTruncate(ReplicationRequest replicationRequest) {
      replicationRequest.getEntries().stream()
              .filter(entry -> wal.getLastLogIndex() >= entry.getEntryIndex() &&
                      entry.getGeneration() != wal.readAt(entry.getEntryIndex()).getGeneration())
              .forEach(entry -> wal.truncate(entry.getEntryIndex()));
  }
```

**follower (class ReplicatedLog...)**

```java
  private ReplicationResponse appendEntries(ReplicationRequest replicationRequest) {
      List<WALEntry> entries = replicationRequest.getEntries();
      entries.stream()
              .filter(e -> !wal.exists(e))
              .forEach(e -> wal.writeEntry(e));
      return new ReplicationResponse(SUCCEEDED, serverId(), replicationState.getGeneration(), wal.getLastLogIndex());
  }
```

当请求中的世代号低于服务器知道的最新一代时，跟随者拒绝复制请求。 这通知领导者下台并成为追随者。

follower (class ReplicatedLog...)

```java
  Long currentGeneration = replicationState.getGeneration();
  if (currentGeneration > request.getGeneration()) {
      return new ReplicationResponse(FAILED, serverId(), currentGeneration, wal.getLastLogIndex());
  }
```

​	当收到（复制）响应时，*Leader* 会跟踪在每个服务器上复制的日志索引。 它使用它来跟踪成功复制到 *Quorum* 法定人数的日志条目，并将索引作为 *commitIndex* 进行跟踪。 *commitIndex* 是日志中的高水位线。

**leader (class ReplicatedLog...)**

```java
  logger.info("Updating matchIndex for " + response.getServerId() + " to " + response.getReplicatedLogIndex());
  updateMatchingLogIndex(response.getServerId(), response.getReplicatedLogIndex());
  var logIndexAtQuorum = computeHighwaterMark(logIndexesAtAllServers(), config.numberOfServers());
  var currentHighWaterMark = replicationState.getHighWaterMark();
  if (logIndexAtQuorum > currentHighWaterMark && logIndexAtQuorum != 0) {
      applyLogEntries(currentHighWaterMark, logIndexAtQuorum);
      replicationState.setHighWaterMark(logIndexAtQuorum);
  }
```

**leader (class ReplicatedLog...)**

```java
  Long computeHighwaterMark(List<Long> serverLogIndexes, int noOfServers) {
      serverLogIndexes.sort(Long::compareTo);
      //获取中位的水位线
      return serverLogIndexes.get(noOfServers / 2);
  }
```

**leader (class ReplicatedLog...)**

```java
  private void updateMatchingLogIndex(int serverId, long replicatedLogIndex) {
      FollowerHandler follower = getFollowerHandler(serverId);
      follower.updateLastReplicationIndex(replicatedLogIndex);
  }
```

**leader (class ReplicatedLog...)**

```java
  public void updateLastReplicationIndex(long lastReplicatedLogIndex) {
      this.matchIndex = lastReplicatedLogIndex;
  }
```

### 完全复制

​		有一点非常重要，就是要确保所有的节点都能收到来自领导者所有的日志条目，即便是节点断开连接，或是崩溃之后又恢复之后。Raft 有个机制能确保所有的集群节点能够收到来自领导者的所有日志条目。

​		在Raft 中的每个复制请求中，领导者还会发送正在复制日志条目的前一项日志条目的日志Index及其世代Generation。 如果追随者发现之前的日志条目Index和世代与其本地日志不匹配，则追随者拒绝该请求。 这向领导者表明，追随者的日志需要同步一些较早的日志条目。

**follower (class ReplicatedLog...)**

```java
  if (!wal.isEmpty() && request.getPrevLogIndex() >= wal.getLogStartIndex() &&
           generationAt(request.getPrevLogIndex()) != request.getPrevLogGeneration()) {
      return new ReplicationResponse(FAILED, serverId(), replicationState.getGeneration(), wal.getLastLogIndex());
  }
```

**follower (class ReplicatedLog...)**

```java
  private Long generationAt(long prevLogIndex) {
      WALEntry walEntry = wal.readAt(prevLogIndex);

      return walEntry.getGeneration();
  }
```

这样，领导者会递减匹配索引（matchIndex），并尝试发送较低索引的日志条目。它会一直这么做，直到追随者接受复制请求。

leader (class ReplicatedLog...)

```java
  //rejected because of conflicting entries, decrement matchIndex
  FollowerHandler peer = getFollowerHandler(response.getServerId());
  logger.info("decrementing nextIndex for peer " + peer.getId() + " from " + peer.getNextIndex());
  peer.decrementNextIndex();
  replicateOn(peer, peer.getNextIndex());
```

对前一项日志Index和世代的检查允许领导者发现两件事。

- 追随者是否存在日志条目缺失。例如，如果追随者只有一个条目，而领导者要开始复制第三个条目，那么，这个请求就会遭到拒绝，直到领导者复制第二个条目。
- 日志中的前一个是否来自不同的世代，与领导者日志中的对应条目相比，是高还是低。领导者会尝试复制索引较低的日志条目，直到请求得到接受。追随者会截断世代不匹配的日志条目。

按照这种方式，领导者通过使用前一项的Index检测缺失或冲突的日志条目，尝试将自己的日志推送给所有的追随者。这就确保了所有的集群节点最终都能收到来自领导者的所有日志条目，即使它们断开了一段时间的连接。

Raft 没有单独的提交消息，而是将提交索引（commitIndex）作为常规复制请求的一部分进行发送。空的复制请求也可以当做心跳发送。因此，commitIndex 会当做心跳请求的一部分发送给追随者。

#### 日志条目以日志顺序执行

一旦领导者更新了它的 commitIndex，它就会按顺序执行日志条目，从上一个 commitIndex 的值执行到最新的 commitIndex 值。一旦日志条目执行完毕，客户端请求就完成了，应答会返回给客户端。

class ReplicatedLog…

```java
  private void applyLogEntries(Long previousCommitIndex, Long commitIndex) {
      for (long index = previousCommitIndex + 1; index <= commitIndex; index++) {
          WALEntry walEntry = wal.readAt(index);
          var responses = stateMachine.applyEntries(Arrays.asList(walEntry));
          completeActiveProposals(index, responses);
      }
  }
```

领导者还会在它发送给追随者的心跳请求中发送 commitIndex。追随者会更新 commitIndex，并以同样的方式应用这些日志条目。

class ReplicatedLog…

```java
  private void updateHighWaterMark(ReplicationRequest request) {
      if (request.getHighWaterMark() > replicationState.getHighWaterMark()) {
          var previousHighWaterMark = replicationState.getHighWaterMark();
          replicationState.setHighWaterMark(request.getHighWaterMark());
          applyLogEntries(previousHighWaterMark, request.getHighWaterMark());
      }
  }
```



### 领导者选举

领导者选举就是检测到日志条目在前一个 Quorum 中完成提交的阶段。 每个集群节点都以三种状态运行：候选者、领导者或追随者。 集群节点以跟随者状态开始，期望来自现有领导者的心跳。 如果一个追随者在预定的时间段内没有收到任何领导者的消息，它就会进入候选状态并开始领导者选举。 领导者选举算法建立一个新的世代时钟值。 Raft 将 Generation Clock 称为任期(term)。

领导者选举机制也确保当选的领导者拥有 Quorum 所规定的最新日志条目。这是Raft所做的一个优化，避免了日志条目要从以前的 Quorum 转移新的领导者上。

新领导者选举的启动要通过向每个对等服务器发送消息，请求开始投票。

class ReplicatedLog…

```java
  private void startLeaderElection() {
      replicationState.setGeneration(replicationState.getGeneration() + 1);
      registerSelfVote();
      requestVoteFrom(followers);
  }
```

一旦一个服务器节点在某一[世代时钟（Generation Clock）](https://github.com/dreamhead/patterns-of-distributed-systems/blob/master/content/generation-clock.md)投票中得到投票，在同一个世代中总能获得一样的投票。这就确保了在选举成功发生的情况下，如果其它服务器以同样的世代请求投票，它是不会当选的。投票请求的处理过程如下：

class ReplicatedLog…

```java
  VoteResponse handleVoteRequest(VoteRequest voteRequest) {
      //for higher generation request become follower.
      // But we do not know who the leader is yet.
      if (voteRequest.getGeneration() > replicationState.getGeneration()) {
          becomeFollower(LEADER_NOT_KNOWN, voteRequest.getGeneration());
      }

      VoteTracker voteTracker = replicationState.getVoteTracker();
      if (voteRequest.getGeneration() == replicationState.getGeneration() && !replicationState.hasLeader()) {
              if(isUptoDate(voteRequest) && !voteTracker.alreadyVoted()) {
                  voteTracker.registerVote(voteRequest.getServerId());
                  return grantVote();
              }
              if (voteTracker.alreadyVoted()) {
                  return voteTracker.votedFor == voteRequest.getServerId() ?
                          grantVote():rejectVote();

              }
      }
      return rejectVote();
  }

  private boolean isUptoDate(VoteRequest voteRequest) {
      boolean result = voteRequest.getLastLogEntryGeneration() > wal.getLastLogEntryGeneration()
              || (voteRequest.getLastLogEntryGeneration() == wal.getLastLogEntryGeneration() &&
              voteRequest.getLastLogEntryIndex() >= wal.getLastLogIndex());
      return result;
  }
```

​		收到大多数服务器投票的服务器会切换到领导者状态。这里的大多数是按照 [Quorum]() 讨论的方式确定的。一旦当选，领导者会持续地向所有的追随者发送[心跳（HeartBeat）]()。如果追随者在指定的时间间隔内没有收到[心跳（HeartBeat）]()，就会触发新的领导者选举。

### 上一代的日志条目

​		如上节所述，共识算法的第一阶段会检测既有的值，这些值在算法的前几次运行中已经复制过了。另一个关键点是，这些值就会提议为领导者最新世代的值。第二阶段会决定，只有当这些值提议为当前世代的值时，这些值才会得到提交。Raft 不会更新既有日志条目的世代数。因此，如果领导者拥有来自上一世代的日志条目，而这些条目在一些追随者中是缺失的，它不会仅仅根据大多数的 Quorum 就将这些条目标记为已提交。这是因为有其它服务器可能此时处于不可用的状态，但其拥有同样索引但更高世代的条目。如果领导者在没有复制其当前世代这些日志条目的情况下宕机了，这些条目就会被新的领导者改写。所以，在 Raft 中，新的领导者必须在其任期（term）内提交至少一个条目。然后，它可以安全地提交所有以前的条目。大多数实际的 Raft 实现都在领导者选举后，立即提交一个空操作（no-op）的日志项，这个动作会在领导者得到承认为客户端请求提供服务之前。详情请参考 [raft-phd](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf) 3.6.1节。

#### 一次领导者选举的示例

考虑有五个服务器：雅典（athens）、拜占庭（byzantium）、锡兰（cyrene）、德尔菲（delphi）和以弗所（ephesus）。以弗所是第一代的领导者。它已经把日志条目复制了其自身、德尔菲和雅典。

![img](.\images\raft-lost-heartbeats.png)

​																		*Figure 2: 失去心跳触发选举*

此时，以弗所（ephesus）和德尔菲（delphi）同集群的其它节点失去连接。

拜占庭（byzantium）有最小的选举超时，因此，它会把[世代时钟]（Generation Clock）递增到 2，由此触发选举。锡兰（cyrene）其世代小于 2，而且它也有同拜占庭（byzantium）同样的日志条目。因此，它会批准这次投票。但是，雅典（athens）其日志中有额外的条目。因此，它会拒绝这次投票。

因为拜占庭（byzantium）无法获得多数的 3 票，所以，它就失去了选举权，回到追随者状态，如下图所示。

![img](.\images\raft-election-timeout.png)

​																	*Figure 3: 日志不是最新的失去选举*

雅典（athens）超时，触发下一轮选举。它将[世代时钟（Generation Clock）]()递增到 3，并向拜占庭（byzantium）和锡兰（cyrene）发送了投票请求。因为拜占庭（byzantium）和锡兰（cyrene）的世代数比较低，也比雅典（athens）的日志条目少，二者都批准了雅典（athens）的投票。一旦雅典（athens）获得了大多数的投票，它就会变成领导者，开始向拜占庭（byzantium）和锡兰（cyrene）发送心跳。一旦拜占庭（byzantium）和锡兰（cyrene）接收到了来自更高世代的心跳，它们会递增他们的世代。这就确认了雅典（athens）的领导者地位，如下图所示。雅典（athens）随后就会将自己的日志复制给拜占庭（byzantium）和锡兰（cyrene）。

![img](https://martinfowler.com/articles/patterns-of-distributed-systems/raft-won-election.png)

​																*Figure 4: 拥有最新日志的节点赢得选举*

雅典（athens）现在将来自世代 1 的 Entry2 复制给拜占庭（byzantium）和锡兰（cyrene）。但由于它是上一代的日志条目，即便 Entry2 成功的在大多数 Quorum 上复制，它也不会更新提交索引（commitIndex）。

![img](D:\workspace\patterns-of-distributed-system-zh_CN\images\raft-leader-commitIndex-previous-term.png)

雅典（athens）在其本地日志中追加了一个空操作（no-op）的条目。在这个第 3 代的新条目成功复制后，它会更新提交索引（commitIndex）。

如果以弗所（ephesus）回来或是恢复了网络连接，它会向锡兰（cyrene）发送请求。因为锡兰（cyrene）现在是第 3 代了，它会拒绝这个请求。以弗所（ephesus）会在拒绝应答中得到新的任期（term），下台成为一个追随者。

![img](.\images\raft-leader-stepdown.png)

​																					Figure 7: 领导下台

### 技术考量

以下是任何复制日志机制都需要有的一些重要技术考量。

- 任何共识建立机制的第一阶段都需要了解日志条目在上一个 Quorum 上可能已经复制过了。领导者需要了解所有这些条目，确保它们复制到集群的每个节点上。

Raft 会确保当选领导者的集群节点拥有同服务器的 Quorum 拥有同样的最新日志，所以，日志条目无需从其它集群节点传给新的领导者。

有可能一些条目存在冲突。在这种情况下，追随者日志中冲突的条目会被覆盖。

- 有可能集群中的一些集群节点落后了，可能是因为它们崩溃后重新启动，可能是与领导者断开了连接。领导者需要跟踪每个集群节点，确保它发送了所有缺失的日志条目。

Raft 会为每个集群节点维护一个状态，以便了解在每个节点上都已成功复制的日志条目的索引。向每个节点发送的复制请求都会包含从这个日志索引开始的所有条目，确保每个集群节点获得所有的日志条目。

- 客户端如何与复制日志进行交互，以找到领导，这个实现在[一致性内核（Consistent Core）]()中讨论过。

在客户端重试的情况下，集群会检测重复的请求，通过采用[幂等接收者（Idempotent Receiver）]()就可以进行处理。

- 日志通常会用[低水位标记（Low-Water Mark）]()进行压缩。复制日志会周期性地进行存储快照，比如，几千个条目之后就快照一次。然后，快照索引之前的日志就可以丢弃了。缓慢的追随者，或是新加入的服务器，需要发送完整的日志，发给它们的就是快照，而非单独的日志条目。
- 这里的一个关键假设，所有的请求都是严格有序的。这可能并非总能满足的需求。例如，一个键值存储可能不需要对不同键值的请求进行排序。在这种情况下，有可能为每个键值运行一个不同的共识实例。这样一来，就不需要对所有的请求都有单一的领导者了。

[EPaxos](https://www.cs.cmu.edu/~dga/papers/epaxos-sosp2013.pdf) 就是一种不依赖单一领导者对请求进行排序的算法。

在像 [MongoDB](https://www.mongodb.com/) 这样的分区数据库中，每个分区都会维护一个复制日志。因此，请求是按分区排序的，而非跨分区。

### 推送（Push） vs. 拉取（Pull）

在这里解释的 [Raft](https://raft.github.io/) 复制机制中，领导者可以将所有日志条目推送给追随者，也可以让追随者来拉取日志条目。[Kafka](https://kafka.apache.org/) 的 [Raft 实现](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A+A+Raft+Protocol+for+the+Metadata+Quorum)就遵循了基于**拉取**的复制。

### 日志里有什么？

复制日志机制广泛地用于各种应用之中，从键值存储到[区块链](https://en.wikipedia.org/wiki/Blockchain)。

对键值存储而言，日志条目是关于设置键值与值的。对于[租约（Lease）](https://github.com/dreamhead/patterns-of-distributed-systems/blob/master/content/lease.md)而言，日志条目是关于设置命名租约的。对于区块链而言，日志条目是区块链中的区块，它需要以同样的顺序提供给所有的对等体（peer）。对于像 [MongoDB](https://www.mongodb.com/) 这样的数据库而言，日志条目就是需要持续复制的数据。

## 示例

复制日志是 [Raft](https://raft.github.io/)、[多 Paxos](https://www.youtube.com/watch?v=JEpsBg0AO6o&t=1920s)、[Zab](https://zookeeper.apache.org/doc/r3.4.13/zookeeperInternals.html#sc_atomicBroadcast) 和 [viewstamped 复制](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)协议使用的机制。这种技术被称为[状态机复制](https://en.wikipedia.org/wiki/State_machine_replication)，各个副本都以以相同的顺序执行相同的命令。[一致性内核（Consistent Core）](https://github.com/dreamhead/patterns-of-distributed-systems/blob/master/content/consistent-core.md)通常是用状态机复制机制构建出来的。

像 [hyperledger fabric](https://github.com/hyperledger/fabric)这样的区块链实现有一个排序组件，它是基于复制日志的机制。之前版本的 hyperledger fabric 使用 [Kafka](https://kafka.apache.org/)对区块链中的区块进行排序。最近的版本则使用 [Raft](https://raft.github.io/) 达成同样的目的。
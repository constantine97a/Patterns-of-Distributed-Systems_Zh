# 一致的核心

​		维护一个较小的集群，提供更强的一致性，以允许大型数据集群在不实施基于Quorum的算法的情况下协调服务器活动。

[TOC]



## 问题

​		当一个集群需要处理大量数据时，它需要越来越多的服务器。对于一个服务器集群，有一些共同的需求，比如选择特定的服务器作为特定任务的master，管理组成员信息，数据分区到服务器的映射等。这些需求需要*强一致性保证*，即*线性化*  linearizability. 实现还需要容错。一种常见的方法是使用基于 *Quorum* 的容错共识算法。<u>但是在基于*Quorum*的系统中，吞吐量会随着集群的大小而降低</u>。

```asciiarmor
线性化是最强的一致性保证，保证所有客户端都能看到最新提交的数据更新。<u>提供线性化和容错需要在服务器上实现 Raft、Zab 或 Paxos 等共识算法?
虽然共识算法是实现一致核心的基本要求，但客户端交互的各个方面——例如客户端如何找到领导者、如何处理重复请求等——都是重要的实现决策。还有一些关于安全性和活性的重要实施注意事项。 Paxos 只定义了共识算法，但这些其他实现方面在 Paxos 文献中没有很好地记录。 Raft 非常清楚地记录了各种实现方面以及参考实现，因此是当今使用最广泛的算法。
```



## 解决方案

​	实施一个较小的 3 到 5 个节点集群，提供线性化保证和容错。 [1] 单独的数据集群可以使用小型一致集群来管理元数据，并使用 Lease 等原语做出集群范围的决策。 这样，数据集群可以增长到大量的服务器，但仍然可以使用较小的元数据集群来执行某些需要强一致性保证的操作。

![img](.\images\ConsistentCore.png)

*Figure 1: Consistent Core*

A typical interface of consistent core looks like this:

```java
public interface ConsistentCore {
    CompletableFuture put(String key, String value);

    List<String> get(String keyPrefix);

    CompletableFuture registerLease(String name, long ttl);

    void refreshLease(String name);

    void watch(String name, Consumer<WatchEvent> watchCallback);
}
```

至少，Consistent Core 提供了一个简单的键值存储机制。 它用于存储元数据。

## 元数据存储

​		存储是使用 Raft 等共识算法实现的。这是 Replicated Write Ahead Log 实现的一个示例，其中复制由 Leader 和 Followers和High-Water Mark 两种模式处理， 用于使用 Quorum 跟踪成功的复制

### 支持分层存储

​	Consistent Core 通常用于存储以下数据：组成员身份或跨服务器的任务分配。一种常见的使用模式是使用前缀来限定元数据的类型。例如对于组成员，密钥将全部存储为 /servers/1、servers/2 等。对于分配给服务器的任务，密钥可以是 /tasks/task1、/tasks/task2。通常使用具有特定前缀的所有键读取此数据。例如，要获取有关集群中所有服务器的信息，将读取所有带有前缀 /servers 的键。

一个示例用法如下：

服务器可以通过使用前缀 /servers 创建自己的密钥来向一致核心注册自己。

```java
client1.setValue("/servers/1", "{address:192.168.199.10, port:8000}");

client2.setValue("/servers/2", "{address:192.168.199.11, port:8000}");

client3.setValue("/servers/3", "{address:192.168.199.12, port:8000}");
```


然后，客户端可以通过读取键前缀 /servers 来了解集群中的所有服务器，如下所示：

```java
assertEquals(client1.getValue("/servers"), Arrays.asList("{address:192.168.199.12, port:8000}",
                                                            "{address:192.168.199.11, port:8000}",
                                                            "{address:192.168.199.10, port:8000}"));
```

由于数据存储的这种分层特性，[zookeeper]、[chubby] 等产品提供了类似文件系统的界面，用户在其中创建目录和文件或节点，具有父节点和子节点的概念。 [etcd3] 有一个扁平的键空间，能够获取一系列键。



### 处理客户交互

​		一致核心功能的关键要求之一是客户端如何与核心交互。以下方面对于客户使用 Consistent Core 至关重要。

#### 寻找领导者

​		重要的是所有操作都在领导者上执行，因此客户端库需要首先找到领导者服务器。有两种方法可以满足这一要求。

* 一致性核心中的follower服务器知道当前的leader，因此如果客户端连接到follower，它可以返回leader的地址。然后客户端可以直接连接到响应中标识的领导者。需要注意的是，当客户端尝试连接时，服务器可能正在选举领导者。在这种情况下，服务器无法返回领导者地址，客户端需要等待并尝试另一台服务器。

* 服务器可以实现转发机制，将所有客户端请求转发给领导者。这允许客户端连接到任何服务器。同样，如果服务器正在选举领导者，那么客户端需要重试，直到领导者选举成功并建立合法领导者。
  zookeeper 和 etcd 等产品实现了这种方法，因为它们允许跟随服务器处理一些只读请求；当大量客户端是只读的时，这避免了领导者的瓶颈。这降低了客户端根据请求类型连接到领导者或追随者的复杂性。

找到领导者的一个简单机制是尝试连接到每个服务器并尝试发送请求，如果不是领导者，服务器会以重定向响应进行响应。

```
串行化和线性化
当追随者服务器处理读取请求时，客户端可能会获得陈旧的数据，因为来自领导者的最新提交尚未到达追随者。客户端接收更新的顺序仍然保持不变，但更新可能不是最新的。这是[serializability]串行化保证，而不是线性化 linearizability。线性化保证每个客户端都能获得最新的更新。当客户端只需要读取元数据并且可以容忍过时的元数据一段时间时，客户端可以使用可串行化保证。对于像 Lease 这样的操作，严格需要线性化。

如果领导者与集群的其余部分分开，客户端可以从领导者那里获得陈旧的值，Raft 描述了一种提供线性化读取的机制。参见 readIndex 的 etcd 实现。 YugabyteDB 使用一种称为 [yugabyte-leader-lease] 的技术来实现相同的目的。

类似的情况也可能发生在分区的追随者身上。跟随者可能被分区并且可能不会将最新值返回给客户端。为了确保follower不被分区并且与leader保持同步，需要查询leader知道最新的更新，并等到收到最新的更新后再响应client，见提议的kafka以设计为例。
```

找到领导者的一个简单机制是尝试连接到每个服务器并尝试发送请求，如果不是领导者，服务器会以重定向响应进行响应。

```java
private void establishConnectionToLeader(List<InetAddressAndPort> servers) {
    for (InetAddressAndPort server : servers) {
        try {
            SingleSocketChannel socketChannel = new SingleSocketChannel(server, 10);
            logger.info("Trying to connect to " + server);
            RequestOrResponse response = sendConnectRequest(socketChannel);
            if (isRedirectResponse(response)) {
                redirectToLeader(response);
                break;
            } else if (isLookingForLeader(response)) {
                logger.info("Server is looking for leader. Trying next server");
                continue;
            } else { //we know the leader
                logger.info("Found leader. Establishing a new connection.");
                newPipelinedConnection(server);
                break;
            }
        } catch (IOException e) {
            logger.info("Unable to connect to " + server);
            //try next server
        }
    }
}

private boolean isLookingForLeader(RequestOrResponse requestOrResponse) {
    return requestOrResponse.getRequestId() == RequestId.LookingForLeader.getId();
}

private void redirectToLeader(RequestOrResponse response) {
    RedirectToLeaderResponse redirectResponse = deserialize(response);
    newPipelinedConnection(redirectResponse.leaderAddress);

    logger.info("Connected to the new leader "
            + redirectResponse.leaderServerId
            + " " + redirectResponse.leaderAddress
            + ". Checking connection");
}


private boolean isRedirectResponse(RequestOrResponse requestOrResponse) {
    return requestOrResponse.getRequestId() == RequestId.RedirectToLeader.getId();
}
```

​		仅仅建立 TCP 连接是不够的，我们需要知道服务器是否可以处理我们的请求。 因此客户端向服务器发送一个特殊的连接请求，以确认它是否可以服务请求或重定向到领导服务器。

```java
private RequestOrResponse sendConnectRequest(SingleSocketChannel socketChannel) throws IOException {
    RequestOrResponse request
            = new RequestOrResponse(RequestId.ConnectRequest.getId(), JsonSerDes.serialize("CONNECT"), 0);
    try {
        return socketChannel.blockingSend(request);
    } catch (IOException e) {
        resetConnectionToLeader();
        throw e;
    }
}
```

一旦连接，客户端维护一个到领导服务器的单套接字通道[Single Socket Channel](Single Socket Channel.md).

#### 处理重复请求

在失败的情况下，客户端可能会尝试连接到新的领导者，重新发送请求。 但是，如果这些请求在失败之前已经由失败的领导者处理，则可能会导致重复。 因此，在服务器上有一种机制来忽略重复请求是很重要的。 幂等接收器模式 [Idempotent Receiver](https://martinfowler.com/articles/patterns-of-distributed-systems/idempotent-receiver.html) 用于实现重复检测。

可以使用 Lease 跨一组服务器协调任务。 同样可以用来实现组成员身份和故障检测机制。

[State Watch](https://martinfowler.com/articles/patterns-of-distributed-systems/state-watch.html)用于获取元数据或时间限制租约更改的通知。



## 例子

众所周知，谷歌使用 [chubby] 锁定服务进行协调和元数据管理。

Kafka 使用 [zookeeper] 来管理元数据并做出决策，例如集群主节点的领导选举。 Kafka 中提议的架构更改将用自己的基于 Raft 的控制器集群取代 zookeeper。

[bookkeeper] 使用 Zookeeper 管理集群元数据。

[kubernetes] 使用 [etcd] 进行协调，管理集群元数据和组成员信息。

所有的大数据存储和处理系统，如 [hdfs]、[spark]、[flink] 都使用 [zookeeper] 来实现高可用性和集群协调。

### Note

1：因为整个集群都依赖于 Consistent Core，所以了解所使用的共识算法的细节至关重要。在一些棘手的网络分区情况下，共识实现可能会遇到活性问题。例如，一个 Raft 集群可能会被分区的服务器中断，这可能会持续触发领导者选举，除非特别小心。 Cloudflare 最近发生的这起事件就是一个值得学习的好例子。
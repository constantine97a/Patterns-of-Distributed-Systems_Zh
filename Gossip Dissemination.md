# Gossip 传播

[TOC]

## 问题

​		在节点集群中，每个节点都需要将其拥有的元数据信息传递给集群中的所有其他节点，而不依赖于共享存储。 在大型集群中，如果所有服务器都与所有其他服务器通信，则会消耗大量网络带宽。 即使某些网络链接出现问题，信息也应该到达所有节点。

## 解决方案

​		集群节点使用 gossip 风格的通信来传播状态更新。每个节点选择一个随机节点来传递它拥有的信息。这是定期进行的，比如每 1 秒。每次，都会选择*一个*随机节点来传递信息。

在大型集群中，需要考虑以下事项：

* 对每台服务器生成的消息数量设置一个固定限制
* 消息不应消耗大量网络带宽。应该有一个几百 Kbs 的上限，以确保应用程序的数据传输不受集群中太多消息的影响。
* 元数据传播应该容忍网络和一些服务器故障。即使有几个网络链接断开，或者一些服务器出现故障，它也应该到达所有集群节点。
  正如边栏中所讨论的，Gossip 式通信满足所有这些要求。

```asciiarmor
流行病、谣言和计算机通信
目前，我们都在经历像 Covid19 这样的流行病在全球蔓延的速度有多快。流行病的数学特性描述了为什么它们传播得如此之快。流行病学的数学分支研究流行病或谣言如何在社会中传播。Gossip传播基于流行病学的数学模型。流行病或谣言的关键特征是传播速度非常快，即使每个人随机接触的人也很少。即使只有很少的总互动，整个人群也可能被感染。更具体地说，如果 n 是给定人口中的总人数，则它的交互作用与每个个体的 log(n) 成正比。正如 Indranil Gupta 教授在他的 Gossip Analysis 中所讨论的，log(n) 几乎可以被视为一个常数。

流行病传播的这种特性对于在一组 processes 中传播信息非常有用。即使一个给定的process随机只与几个 process通信，在极少数的通信轮次中，集群中的所有节点都会拥有相同的信息。 Hashicorp 有一个非常好的convergence simulator[https://www.serf.io/docs/internals/simulator.html] 来演示信息在整个集群中传播的速度，即使出现一些网络丢失和节点故障。
```

每个集群节点将元数据存储为与集群中每个节点关联的键值对列表，如下所示：

class Gossip…

```java
  Map<NodeId, NodeState> clusterMetadata = new HashMap<>();
```

class NodeState…

```java
  Map<String, VersionedValue> values = new HashMap<>();
```

在启动时，每个集群节点都会添加关于自身的元数据，这些元数据需要传播到其他节点。 元数据的一个例子可以是节点监听的 IP 地址和端口、它负责的分区等。Gossip 实例需要知道至少一个其他节点才能启动 gossip 通信。 用于初始化 Gossip 实例的众所周知的集群节点称为种子节点或引入者。 任何节点都可以充当引入者。

class Gossip…

```java
  public Gossip(InetAddressAndPort listenAddress,
                List<InetAddressAndPort> seedNodes,
                String nodeId) throws IOException {
      this.listenAddress = listenAddress;
      // filter this node itself in case its part of the seed nodes
      this.seedNodes = removeSelfAddress(seedNodes);
      this.nodeId = new NodeId(nodeId);
      addLocalState(GossipKeys.ADDRESS, listenAddress.toString());
      this.socketServer = new NIOSocketListener(newGossipRequestConsumer(), listenAddress);
  }

  private void addLocalState(String key, String value) {
      NodeState nodeState = clusterMetadata.get(listenAddress);
      if (nodeState == null) {
          nodeState = new NodeState();
          clusterMetadata.put(nodeId, nodeState);
      }
      nodeState.add(key, new VersionedValue(value, incremenetVersion()));
  }
```

每个集群节点安排一个作业，以定期将其拥有的元数据传输到其他节点。

class Gossip…

```java
  private ScheduledThreadPoolExecutor gossipExecutor = new ScheduledThreadPoolExecutor(1);
  private long gossipIntervalMs = 1000;
  private ScheduledFuture<?> taskFuture;
  public void start() {
      socketServer.start();
      taskFuture = gossipExecutor.scheduleAtFixedRate(()-> doGossip(),
                  gossipIntervalMs,
                  gossipIntervalMs,
                  TimeUnit.MILLISECONDS);
  }
```

​	当调度任务被调用时，它会从元数据映射的服务器列表中选择一小组随机节点。 一个小的常数，定义为 Gossip fanout，决定了选择多少节点作为 gossip 目标。 如果尚无任何信息，它会选择一个随机种子节点并将其拥有的元数据映射发送到该节点。

class Gossip…

```java
  public void doGossip() {
      List<InetAddressAndPort> knownClusterNodes = liveNodes();
      if (knownClusterNodes.isEmpty()) {
          sendGossip(seedNodes, gossipFanout);
      } else {
          sendGossip(knownClusterNodes, gossipFanout);
      }
  }

  private List<InetAddressAndPort> liveNodes() {
      Set<InetAddressAndPort> nodes
              = clusterMetadata.values()
              .stream()
              .map(n -> InetAddressAndPort.parse(n.get(GossipKeys.ADDRESS).getValue()))
              .collect(Collectors.toSet());
      return removeSelfAddress(nodes);
  }
```

```java
private void sendGossip(List<InetAddressAndPort> knownClusterNodes, int gossipFanout) {
    if (knownClusterNodes.isEmpty()) {
        return;
    }

    for (int i = 0; i < gossipFanout; i++) {
        InetAddressAndPort nodeAddress = pickRandomNode(knownClusterNodes);
        sendGossipTo(nodeAddress);
    }
}

private void sendGossipTo(InetAddressAndPort nodeAddress) {
    try {
        getLogger().info("Sending gossip state to " + nodeAddress);
        SocketClient<RequestOrResponse> socketClient = new SocketClient(nodeAddress);
        GossipStateMessage gossipStateMessage
                = new GossipStateMessage(this.clusterMetadata);
        RequestOrResponse request
                = createGossipStateRequest(gossipStateMessage);
        byte[] responseBytes = socketClient.blockingSend(request);
        GossipStateMessage responseState = deserialize(responseBytes);
        merge(responseState.getNodeStates());

    } catch (IOException e) {
        getLogger().error("IO error while sending gossip state to " + nodeAddress, e);
    }
}

private RequestOrResponse createGossipStateRequest(GossipStateMessage gossipStateMessage) {
    return new RequestOrResponse(RequestId.PushPullGossipState.getId(),
            JsonSerDes.serialize(gossipStateMessage), correlationId++);
}
```



```asciiarmor
使用 UDP 或 TCP

Gossip 通信假设网络不可靠，因此它可以使用 UDP 作为传输机制。 但是集群节点通常需要一些状态快速收敛的保证，因此使用基于 TCP 的传输来交换gossip状态。
```



接收Gossip消息的集群节点检查它拥有的元数据并发现三件事。

* 传入消息中的值在在本地节点的状态图（state map）中没有
* 节点具有但传入的 Gossip 消息没有的值
* 当节点具有传入消息中存在的值时，选择更高的版本值

然后它将缺失值添加到自己的状态图中。 传入消息中缺少的任何值都将作为响应返回。

发送 Gossip 消息的集群节点会将它从 gossip 响应中获得的值添加到自己的状态中。

*class Gossip…*

```java
  private void handleGossipRequest(org.distrib.patterns.common.Message<RequestOrResponse> request) {
      GossipStateMessage gossipStateMessage = deserialize(request.getRequest());
      Map<NodeId, NodeState> gossipedState = gossipStateMessage.getNodeStates();
      getLogger().info("Merging state from " + request.getClientSocket());
      merge(gossipedState);

      Map<NodeId, NodeState> diff = delta(this.clusterMetadata, gossipedState);
      GossipStateMessage diffResponse = new GossipStateMessage(diff);
      getLogger().info("Sending diff response " + diff);
      request.getClientSocket().write(new RequestOrResponse(RequestId.PushPullGossipState.getId(),
                      JsonSerDes.serialize(diffResponse),
                      request.getRequest().getCorrelationId()));
  }
public Map<NodeId, NodeState> delta(Map<NodeId, NodeState> fromMap, Map<NodeId, NodeState> toMap) {
    Map<NodeId, NodeState> delta = new HashMap<>();
    for (NodeId key : fromMap.keySet()) {
        if (!toMap.containsKey(key)) {
            delta.put(key, fromMap.get(key));
            continue;
        }
        NodeState fromStates = fromMap.get(key);
        NodeState toStates = toMap.get(key);
        NodeState diffStates = fromStates.diff(toStates);
        if (!diffStates.isEmpty()) {
            delta.put(key, diffStates);
        }
    }
    return delta;
}
public void merge(Map<NodeId, NodeState> otherState) {
    Map<NodeId, NodeState> diff = delta(otherState, this.clusterMetadata);
    for (NodeId diffKey : diff.keySet()) {
        if(!this.clusterMetadata.containsKey(diffKey)) {
            this.clusterMetadata.put(diffKey, diff.get(diffKey));
        } else {
            NodeState stateMap = this.clusterMetadata.get(diffKey);
            stateMap.putAll(diff.get(diffKey));
        }
    }
}
```

这个过程在每个集群节点上每隔一秒发生一次，每次选择一个不同的节点来交换状态。



### 避免不必要的状态交换

上面的代码示例显示了节点的完整状态是在 Gossip 消息中发送的。 这对于新加入的节点来说很好，但是一旦状态是最新的，就没有必要发送完整的状态。 集群节点只需要发送自上次 Gossip 以来的状态变化。 为了实现这一点，每个节点都维护一个版本号，每次在本地添加新的元数据条目时，该版本号都会增加。

class Gossip…

```java
  private int gossipStateVersion = 1;


  private int incremenetVersion() {
      return gossipStateVersion++;
  }
```

集群元数据中的每个值都使用版本号进行维护。 这是模式版本化值的示例。

class VersionedValue…

```java
  long version;
  String value;

  public VersionedValue(String value, long version) {
      this.version = version;
      this.value = value;
  }

  public long getVersion() {
      return version;
  }

  public String getValue() {
      return value;
  }
```

然后，每个 Gossip 循环都可以交换特定版本的状态。

class Gossip…

```java
  private void sendKnownVersions(InetAddressAndPort gossipTo) throws IOException {
      Map<NodeId, Long> maxKnownNodeVersions = getMaxKnownNodeVersions();
      RequestOrResponse knownVersionRequest = new RequestOrResponse(RequestId.GossipVersions.getId(),
              JsonSerDes.serialize(new GossipStateVersions(maxKnownNodeVersions)), 0);
      SocketClient<RequestOrResponse> socketClient = new SocketClient(gossipTo);
      byte[] knownVersionResponseBytes = socketClient.blockingSend(knownVersionRequest);
  }

  private Map<NodeId, Long> getMaxKnownNodeVersions() {
      return clusterMetadata.entrySet()
              .stream()
              .collect(Collectors.toMap(e -> e.getKey(), e -> e.getValue().maxVersion()));
  }
```

class NodeState…

```java
  public long maxVersion() {
      return values.values().stream().map(v -> v.getVersion()).max(Comparator.naturalOrder()).orElse(Long.valueOf(0));
  }
```

只有当版本大于请求中的版本时，接收节点才能发送这些值。

class Gossip…

```java
  Map<NodeId, NodeState> getMissingAndNodeStatesHigherThan(Map<NodeId, Long> nodeMaxVersions) {
      Map<NodeId, NodeState> delta = new HashMap<>();
      delta.putAll(higherVersionedNodeStates(nodeMaxVersions));
      delta.putAll(missingNodeStates(nodeMaxVersions));
      return delta;
  }

  private Map<NodeId, NodeState> missingNodeStates(Map<NodeId, Long> nodeMaxVersions) {
      Map<NodeId, NodeState> delta = new HashMap<>();
      List<NodeId> missingKeys = clusterMetadata.keySet().stream().filter(key -> !nodeMaxVersions.containsKey(key)).collect(Collectors.toList());
      for (NodeId missingKey : missingKeys) {
          delta.put(missingKey, clusterMetadata.get(missingKey));
      }
      return delta;
  }

  private Map<NodeId, NodeState> higherVersionedNodeStates(Map<NodeId, Long> nodeMaxVersions) {
      Map<NodeId, NodeState> delta = new HashMap<>();
      Set<NodeId> keySet = nodeMaxVersions.keySet();
      for (NodeId node : keySet) {
          Long maxVersion = nodeMaxVersions.get(node);
          NodeState nodeState = clusterMetadata.get(node);
          if (nodeState == null) {
              continue;
          }
          NodeState deltaState = nodeState.statesGreaterThan(maxVersion);
          if (!deltaState.isEmpty()) {
              delta.put(node, deltaState);
          }
      }
      return delta;
  }
```



### Gossip 的节点选择标准

集群节点随机选择节点发送 Gossip 消息。 Java 中的示例实现可以使用 java.util.Random，如下所示：

class Gossip…

```java
  private Random random = new Random();
  private InetAddressAndPort pickRandomNode(List<InetAddressAndPort> knownClusterNodes) {
      int randomNodeIndex = random.nextInt(knownClusterNodes.size());
      InetAddressAndPort gossipTo = knownClusterNodes.get(randomNodeIndex);
      return gossipTo;
  }
```


可能还有其他考虑因素，例如接触最少的节点。例如 Cockroachdb 中的 Gossip 协议就是这样选择节点的。

Gossip 目标选择的网络拓扑感知方式也存在。这些中的任何一个都可以在 pickRandomNode() 方法中模块化实现。

### 组成员和故障检测

```asciiarmor
最终一致性
与 Gossip 协议的信息交换最终本质上是一致的。即使它的 Gossip 状态收敛非常快，在新节点被整个集群识别或检测到节点故障之前，也会有一些延迟。使用 Gossip 协议进行信息交换的实现需要容忍最终的一致性。
对于需要强一致性的操作，需要使用Consistent Core。
在同一个集群中使用两者是一种常见的做法。例如，[consul] 使用 Gossip 协议进行组成员资格和故障检测，但使用基于 Raft 的 Consistent Core 来存储强一致的服务目录。
维护集群中可用节点的列表是 Gossip 协议最常见的用法之一。有两种方法在使用。
```



* [swim-gossip] 使用一个单独的探测组件，它不断地探测集群中的不同节点以检测它们是否可用。如果它检测到节点是活动的还是死的，该结果将通过 Gossip 通信传播到整个集群。探测器随机选择一个节点来发送 Gossip 消息。如果接收节点检测到这是新信息，它会立即将消息发送到随机选择的节点。这样，集群中的一个节点或新加入的节点的故障，很快就会被整个集群知道。
* 集群节点可以定期更新自己的状态以反映其心跳。然后通过交换的Gossip消息将该状态传播到整个集群。然后，每个集群节点可以检查它是否在固定时间内接收到特定集群节点的任何更新，或者将该节点标记为关闭。在这种情况下，每个集群节点独立确定一个节点是启动还是关闭。



### 处理节点重启

如果节点崩溃或重新启动，版本化的值将无法正常工作，因为所有内存中的状态都会丢失。更重要的是，对于同一个键，节点可以有不同的值。例如，集群节点可以从不同的 IP 地址和端口开始，或者可以从不同的配置开始。 Generation Clock 可以用每个值来标记generation，这样当元数据状态被发送到一个随机的集群节点时，接收节点不仅可以通过版本号来检测变化，还可以通过generation来检测变化。

需要注意的是，核心 Gossip 协议不需要这种机制。但它在实践中实施以确保正确跟踪状态更改。

## 例子

[cassandra] 使用 Gossip 协议进行集群节点的组成员和故障检测。每个集群节点的元数据，例如分配给每个集群节点的令牌，也使用 Gossip 协议传输。

[consul] 使用 [swim-gossip] 协议进行组成员身份和 consul 代理的故障检测。

CockroachDB 使用 Gossip 协议来传播节点元数据。

Hyperledger Fabric 等区块链实现使用 Gossip 协议进行组成员身份和发送分类帐元数据。
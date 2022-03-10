# Version Vector

维护一个计数器列表，每个集群节点一个，以检测并发更新

[TOC]

## 问题

​		如果多个服务器允许更新相同的键，则检测值何时跨一组副本同时更新很重要。



# 解决方案

​		每个键值都与一个版本向量相关联，该版本向量为每个集群节点维护一个数字。本质上，版本向量是一组计数器，每个节点一个。 三个节点（蓝色、绿色、黑色）的版本向量看起来像 [blue: 43, green: 54, black: 12]。 每次节点有内部更新时，它都会更新自己的计数器，因此绿色节点中的更新会将向量更改为 [blue: 43, green: 55, black: 12]。 每当两个节点通信时，它们都会同步它们的向量标记，从而使它们能够检测到任何同时更新。

```
与矢量时钟的区别
[vector-clock] 实现类似。 但是矢量时钟用于跟踪服务器上发生的每个事件。 相比之下，版本向量用于检测一组副本中对同一键的并发更新。 因此，版本向量的实例是按键存储的，而不是按服务器存储的。 像 [riak] 这样的数据库使用术语版本向量而不是向量时钟来实现它们。 有关详细信息，请参阅 [version-vectors-are-not-vector-clocks]。
```

一个典型的版本向量实现如下：

class VersionVector…

```java
  private final TreeMap<String, Long> versions;

  public VersionVector() {
      this(new TreeMap<>());
  }

  public VersionVector(TreeMap<String, Long> versions) {
      this.versions = versions;
  }

  public VersionVector increment(String nodeId) {
      TreeMap<String, Long> versions = new TreeMap<>();
      versions.putAll(this.versions);
      Long version = versions.get(nodeId);
      if(version == null) {
          version = 1L;
      } else {
          version = version + 1L;
      }
      versions.put(nodeId, version);
      return new VersionVector(versions);
  }
```

Note： 是一个值对象，所以每次修改值都需要返回新的对象

存储在服务器上的每个值都与一个版本向量相关联

```
class VersionedValue…

  public class VersionedValue {
      String value;
      VersionVector versionVector;
  
      public VersionedValue(String value, VersionVector versionVector) {
          this.value = value;
          this.versionVector = versionVector;
      }
  
      @Override
      public boolean equals(Object o) {
          if (this == o) return true;
          if (o == null || getClass() != o.getClass()) return false;
          VersionedValue that = (VersionedValue) o;
          return Objects.equal(value, that.value) && Objects.equal(versionVector, that.versionVector);
      }
  
      @Override
      public int hashCode() {
          return Objects.hashCode(value, versionVector);
      }
```





## 比较版本向量

​		通过比较每个节点的版本号来比较版本向量。 如果两个版本向量都具有相同集群节点的版本号，并且每个版本号都高于另一个向量中的版本号，则认为版本向量高于另一个，反之亦然。 如果两个向量的所有版本号都更高，或者它们具有不同集群节点的版本号，则它们被认为是并发的。

以下是一些示例比较

|                           |                    |                            |
| :------------------------ | :----------------- | :------------------------- |
| {blue:2, green:1}         | is greater than    | {blue:1, green:1}          |
| {blue:2, green:1}         | is concurrent with | {blue:1, green:2}          |
| {blue:1, green:1, red: 1} | is greater than    | {blue:1, green:1}          |
| {blue:1, green:1, red: 1} | is concurrent with | {blue:1, green:1, pink: 1} |

比较实现如下：

```
public enum Ordering {
    Before,
    After,
    Concurrent
}
```

class VersionVector…

```
  //This is exact code for Voldermort implementation of VectorClock comparison.
  //https://github.com/voldemort/voldemort/blob/master/src/java/voldemort/versioning/VectorClockUtils.java
  public static Ordering compare(VersionVector v1, VersionVector v2) {
      if(v1 == null || v2 == null)
          throw new IllegalArgumentException("Can't compare null vector clocks!");
      // We do two checks: v1 <= v2 and v2 <= v1 if both are true then
      boolean v1Bigger = false;
      boolean v2Bigger = false;

      SortedSet<String> v1Nodes = v1.getVersions().navigableKeySet();
      SortedSet<String> v2Nodes = v2.getVersions().navigableKeySet();      
      SortedSet<String> commonNodes = getCommonNodes(v1Nodes, v2Nodes);      
      // if v1 has more nodes than common nodes
      // v1 has clocks that v2 does not
      if(v1Nodes.size() > commonNodes.size()) {
          v1Bigger = true;
      }
      // if v2 has more nodes than common nodes
      // v2 has clocks that v1 does not
      if(v2Nodes.size() > commonNodes.size()) {
          v2Bigger = true;
      }
      // compare the common parts
      for(String nodeId: commonNodes) {
          // no need to compare more
          if(v1Bigger && v2Bigger) {
              break;
          }
          long v1Version = v1.getVersions().get(nodeId);
          long v2Version = v2.getVersions().get(nodeId);
          if(v1Version > v2Version) {
              v1Bigger = true;
          } else if(v1Version < v2Version) {
              v2Bigger = true;
          }
      }

      /*
       * This is the case where they are equal. Consciously return BEFORE, so
       * that the we would throw back an ObsoleteVersionException for online
       * writes with the same clock.
       */
      if(!v1Bigger && !v2Bigger)
          return Ordering.Before;
          /* This is the case where v1 is a successor clock to v2 */
      else if(v1Bigger && !v2Bigger)
          return Ordering.After;
          /* This is the case where v2 is a successor clock to v1 */
      else if(!v1Bigger && v2Bigger)
          return Ordering.Before;
          /* This is the case where both clocks are parallel to one another */
      else
          return Ordering.Concurrent;
  }

  private static SortedSet<String> getCommonNodes(SortedSet<String> v1Nodes, SortedSet<String> v2Nodes) {
      // get clocks(nodeIds) that both v1 and v2 has
      SortedSet<String> commonNodes = Sets.newTreeSet(v1Nodes);
      commonNodes.retainAll(v2Nodes);
      return commonNodes;
  }


  public boolean descents(VersionVector other) {
      return other.compareTo(this) == Ordering.Before;
  }
```

## 在键值存储中使用版本向量

​		版本向量可用于键值存储，如下所示。 需要一个版本化值列表，因为可以有多个并发值。

​		当客户端想要存储一个值时，它首先读取给定Key的最新已知版本。 然后它会根据密钥选择集群节点来存储值。 在存储值时，客户端传回已知版本。 请求流程如下图所示。 有两个服务器名为蓝色和绿色。 对于键“名称”，蓝色是主服务器。

![img](.\images\versioned-vector-put.png)

​		在无领导复制方案中，客户端或协调器节点根据Key选择节点写入数据。 版本向量根据Key映射到的主集群节点进行更新。 在其他集群节点上复制具有相同版本向量的值以进行复制。 如果映射到Key的集群节点不可用，则选择下一个节点。 版本向量仅针对保存该值的第一个集群节点递增。 所有其他节点保存数据的副本。 在 [[voldemort]()] 等数据库中递增版本向量的代码如下所示：

class ClusterClient…

```
  public void put(String key, String value, VersionVector existingVersion) {
      List<Integer> allReplicas = findReplicas(key);
      int nodeIndex = 0;
      List<Exception> failures = new ArrayList<>();
      VersionedValue valueWrittenToPrimary = null;
      for (; nodeIndex < allReplicas.size(); nodeIndex++) {
          try {
              ClusterNode node = clusterNodes.get(nodeIndex);
              //the node which is the primary holder of the key value is responsible for incrementing version number.
              //作为键值的主要持有者的节点负责增加版本号。
              valueWrittenToPrimary = node.putAsPrimary(key, value, existingVersion);
              break;
          } catch (Exception e) {
              //if there is exception writing the value to the node, try other replica.
              failures.add(e);
          }
      }

      if (valueWrittenToPrimary == null) {
          throw new NotEnoughNodesAvailable("No node succeeded in writing the value.", failures);
      }

      //Succeded in writing the first node, copy the same to other nodes.
      nodeIndex++;
      for (; nodeIndex < allReplicas.size(); nodeIndex++) {
          ClusterNode node = clusterNodes.get(nodeIndex);
          node.put(key, valueWrittenToPrimary);
      }
  }
```

作为主节点的节点是增加版本号的节点。

```
public VersionedValue putAsPrimary(String key, String value, VersionVector existingVersion) {
    VersionVector newVersion = existingVersion.increment(nodeId);
    VersionedValue versionedValue = new VersionedValue(value, newVersion);
    put(key, versionedValue);
    return versionedValue;
}

public void put(String key, VersionedValue value) {
    versionVectorKvStore.put(key, value);
}
```

​		从上面的代码中可以看出，不同的客户端可以在不同的节点上更新相同的Key，例如当客户端无法到达特定节点时。 这会造成不同节点具有不同值的情况，这些值根据它们的版本向量被认为是“并发的”。

​		如下图所示，client1 和 client2 都在尝试写入键“name”。 如果 client1 无法写入绿色服务器，则绿色服务器将缺少 client1 写入的值。 当 client2 尝试写入，但无法连接到服务器 blue 时，它将写入服务器 green。 键“名称”的版本向量将反映服务器（蓝色和绿色）具有并发写入。

![img](.\images\vector-clock-concurrent-updates.png)

​		因此，当版本被认为是并发的时，基于版本向量的存储为任何Key保留多个版本。

class VersionVectorKVStore…

```
  public void put(String key, VersionedValue newValue) {
      List<VersionedValue> existingValues = kv.get(key);
      if (existingValues == null) {
          existingValues = new ArrayList<>();
      }

      rejectIfOldWrite(key, newValue, existingValues);
      List<VersionedValue> newValues = merge(newValue, existingValues);
      kv.put(key, newValues);
  }

  //If the newValue is older than existing one reject it.
  private void rejectIfOldWrite(String key, VersionedValue newValue, List<VersionedValue> existingValues) {
      for (VersionedValue existingValue : existingValues) {
          if (existingValue.descendsVersion(newValue)) {
              throw new ObsoleteVersionException("Obsolete version for key '" + key
                      + "': " + newValue.versionVector);
          }
      }
  }

  //Merge new value with existing values. Remove values with lower version than the newValue.
  //If the old value is neither before or after (concurrent) with the newValue. It will be preserved
  private List<VersionedValue> merge(VersionedValue newValue, List<VersionedValue> existingValues) {
      List<VersionedValue> retainedValues = removeOlderVersions(newValue, existingValues);
      retainedValues.add(newValue);
      return retainedValues;
  }

  private List<VersionedValue> removeOlderVersions(VersionedValue newValue, List<VersionedValue> existingValues) {
      List<VersionedValue> retainedValues = existingValues
              .stream()
              .filter(v -> !newValue.descendsVersion(v)) //keep versions which are not directly dominated by newValue.
              .collect(Collectors.toList());
      return retainedValues;
  }
```

如果在从多个节点读取时检测到并发值，则会引发错误，从而允许客户端进行可能的冲突解决。

class ClusterClient…

```
  public List<VersionedValue> get(String key) {
      List<Integer> allReplicas = findReplicas(key);

      List<VersionedValue> allValues = new ArrayList<>();
      for (Integer index : allReplicas) {
          ClusterNode clusterNode = clusterNodes.get(index);
          List<VersionedValue> nodeVersions = clusterNode.get(key);

          allValues.addAll(nodeVersions);
      }

      return latestValuesAcrossReplicas(allValues);
  }

  private List<VersionedValue> latestValuesAcrossReplicas(List<VersionedValue> allValues) {
      List<VersionedValue> uniqueValues = removeDuplicates(allValues);
      return retainOnlyLatestValues(uniqueValues);
  }

  private List<VersionedValue> retainOnlyLatestValues(List<VersionedValue> versionedValues) {
      for (int i = 0; i < versionedValues.size(); i++) {
          VersionedValue v1 = versionedValues.get(i);
          versionedValues.removeAll(getPredecessors(v1, versionedValues));
      }
      return versionedValues;
  }

  private List<VersionedValue> getPredecessors(VersionedValue v1, List<VersionedValue> versionedValues) {
      List<VersionedValue> predecessors = new ArrayList<>();
      for (VersionedValue v2 : versionedValues) {
          if (!v1.sameVersion(v2) && v1.descendsVersion(v2)) {
              predecessors.add(v2);
          }
      }
      return predecessors;
  }

  private List<VersionedValue> removeDuplicates(List<VersionedValue> allValues) {
      return allValues.stream().distinct().collect(Collectors.toList());
  }
```

例如，[[riak\]](https://riak.com/posts/technical/vector-clocks-revisited/index.html?p=9545.html) 允许应用程序提供冲突解决程序，如[此处所述](https://docs.riak.com/riak/kv/latest/developing/usage/conflict-resolution/java/index.html)。



### Last Write Wins (LWW) Conflict Resolution

```
Cassandra和 LWW
[cassandra] 虽然在架构上与 [riak] 或 [voldemort] 相同，但根本不使用版本向量，并且仅支持最后写入获胜冲突解决策略。 Casasndra 是一个列族数据库，而不是一个简单的键值存储，它存储每列的时间戳，而不是一个整体的值。 虽然这将解决冲突的负担从用户身上移开，但用户需要确保 [ntp] 服务已配置并跨 cassandra 节点正常工作。 在最坏的情况下，由于时钟漂移，一些最新值可能会被旧值覆盖。
```

虽然版本向量允许检测跨不同服务器集的并发写入，但它们本身并不能帮助客户端确定在发生冲突时选择哪个值。 解决问题的负担在客户身上。 有时客户更喜欢键值存储基于时间戳进行冲突解决。 虽然跨服务器的时间戳存在已知问题，但这种方法的简单性使其成为客户端的首选，即使存在由于跨服务器时间戳问题而丢失一些更新的风险。 它们完全依赖于 NTP 等服务进行良好配置并在整个集群中工作。 [riak] 和 [voldemort] 等数据库允许用户选择“最后写入胜出”的冲突解决策略。

为了支持 LWW 冲突解决，在写入每个值时都会存储一个时间戳。

class TimestampedVersionedValue…

```
  class TimestampedVersionedValue {
      String value;
      VersionVector versionVector;
      long timestamp;
  
      public TimestampedVersionedValue(String value, VersionVector versionVector, long timestamp) {
          this.value = value;
          this.versionVector = versionVector;
          this.timestamp = timestamp;
      }
```

在读取值时，客户端可以使用时间戳来获取最新值。 在这种情况下，版本向量被完全忽略。

class ClusterClient…

```
  public Optional<TimestampedVersionedValue> getWithLWWW(List<TimestampedVersionedValue> values) {
      return values.stream().max(Comparator.comparingLong(v -> v.timestamp));
  }
```



### 读取修复
​		虽然允许任何集群节点接受写入请求可以提高可用性，但重要的是最终所有副本都具有相同的数据。 修复副本的常用方法之一发生在客户端读取数据时。

​		解决冲突后，还可以检测哪些节点具有旧版本。 作为来自客户端的读取请求处理的一部分，可以向具有旧版本的节点发送最新版本。 这称为读取修复。

​		考虑下图中显示的场景。 两个节点，蓝色和绿色，具有键“名称”的值。 绿色节点具有版本向量 [blue: 1, green:1] 的最新版本。 当从副本（蓝色和绿色）中读取值时，它们会进行比较以找出哪个节点缺少最新版本，并将最新版本的放置请求发送到集群节点。

![img](.\images\read-repair.png)

*Figure 3: Read repair*



### 允许在同一个集群节点上进行并发更新

有可能两个客户端同时写入同一个节点。 在上面显示的默认实现中，第二次写入将被拒绝。 在这种情况下，每个集群节点的版本号的基本实现是不够的。

考虑以下场景。 当两个客户端尝试更新相同的密钥时，第二个客户端将收到异常，因为它在其 put 请求中传递的版本是过时的。

![img](.\images\concurrent-update-with-server-versions.png)

像 [riak] 这样的数据库为客户端提供了灵活性，以允许此类并发写入并且不希望得到错误响应。



### 使用客户端 ID 而不是服务器 ID
如果每个集群客户端都可以有唯一的 ID，则可以使用客户端 ID。 每个客户端 ID 都会存储一个版本号。 每次客户端写入值时，它首先读取现有版本，增加与客户端 ID 关联的数字并将其写入服务器。

class ClusterClient…

```
  private VersionedValue putWithClientId(String clientId, int nodeIndex, String key, String value, VersionVector version) {
      ClusterNode node = clusterNodes.get(nodeIndex);
      VersionVector newVersion = version.increment(clientId);
      VersionedValue versionedValue = new VersionedValue(value, newVersion);
      node.put(key, versionedValue);
      return versionedValue;
  }
```

因为每个客户端都会增加自己的计数器，并发写入会在服务器上创建兄弟值，但并发写入永远不会失败。

上面提到的场景给第二个客户端带来了错误，它的工作原理如下：

![img](.\images\concurrent-update-with-client-versions.png)



### 虚线版本向量

​		基于客户端 ID 的版本向量的主要问题之一是版本向量的大小直接取决于客户端的数量。 这会导致集群节点随着时间的推移为给定键累积太多并发值。 这个问题被称为兄弟爆炸[sibling explosion](https://docs.riak.com/riak/kv/2.2.3/learn/concepts/causal-context/index.html#sibling-explosion)。 为了解决这个问题并仍然允许基于集群节点的版本向量，[[riak\]](https://riak.com/posts/technical/vector-clocks-revisited/index.html?p=9545.html) 使用了一种版本向量的变体，称为点分版本向量 [dotted version vector](https://riak.com/posts/technical/vector-clocks-revisited-part-2-dotted-version-vectors/index.html)。



## 例子
[[voldemort\]](https://www.project-voldemort.com/voldemort/) 以这里描述的方式使用版本向量。 它允许基于时间戳的最后一次写入赢得冲突解决。

[[riak\]](https://riak.com/posts/technical/vector-clocks-revisited/index.html?p=9545.html) 开始使用基于客户端 ID 的版本向量，但移至基于集群节点的版本向量，并最终使用点式版本向量。 Riak 还支持基于系统时间戳的最后一次写入胜出冲突解决。

[[cassandra\]](http://cassandra.apache.org/) 不使用版本向量，它仅支持基于系统时间戳的 last write wins 冲突解决。
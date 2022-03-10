# Write-Ahead Log

通过将每个状态更改作为命令持久保存到仅附加日志，提供持久性保证，无需将存储数据结构刷新到磁盘。



[TOC]



## 问题

​		即使在存储数据的服务器机器发生故障的情况下，也需要强大的持久性保证。 一旦服务器同意执行一项操作，即使它失败并重新启动，它也应该这样做，从而丢失其所有内存状态。



## 解决方案

![img](D:\workspace\patterns-of-distributed-system-zh_CN\images\wal.png)

​		将每个状态更改作为命令存储在硬盘上的文件中。 为每个按顺序附加的服务器进程维护一个日志。 单个日志按顺序附加，简化了重启时日志的处理和后续在线操作（当日志附加新命令时）。 每个日志条目都有一个唯一的标识符。 唯一的日志标识符有助于在日志上实现某些其他操作，如分段日志或使用低水位标记清理日志等。日志更新可以通过单一更新队列实现

​		典型的日志条目结构如下所示

​	class WALEntry…

```
  private final Long entryIndex;
  private final byte[] data;
  private final EntryType entryType;
  private final long timeStamp;
```

每次重新启动时都可以读取该文件，并且可以通过重播所有日志条目来恢复状态。

考虑一个简单的内存键值存储：

class KVStore…

```java
  private Map<String, String> kv = new HashMap<>();

  public String get(String key) {
      return kv.get(key);
  }

  public void put(String key, String value) {
      appendLog(key, value);
      kv.put(key, value);
  }

  private Long appendLog(String key, String value) {
      return wal.writeEntry(new SetValueCommand(key, value).serialize());
  }
```

put 操作表示为 Command，在更新内存中的 hashmap 之前将其序列化并存储在日志中。

```java
class SetValueCommand…

  final String key;
  final String value;
  final String attachLease;
  public SetValueCommand(String key, String value) {
      this(key, value, "");
  }
  public SetValueCommand(String key, String value, String attachLease) {
      this.key = key;
      this.value = value;
      this.attachLease = attachLease;
  }

  @Override
  public void serialize(DataOutputStream os) throws IOException {
      os.writeInt(Command.SetValueType);
      os.writeUTF(key);
      os.writeUTF(value);
      os.writeUTF(attachLease);
  }

  public static SetValueCommand deserialize(InputStream is) {
      try {
          DataInputStream dataInputStream = new DataInputStream(is);
          return new SetValueCommand(dataInputStream.readUTF(), dataInputStream.readUTF(), dataInputStream.readUTF());
      } catch (IOException e) {
          throw new RuntimeException(e);
      }
  }
```

这样可以确保一旦 put 方法成功返回，即使持有 KVStore 的进程崩溃，也可以通过在启动时读取日志文件来恢复其状态。

class KVStore…

```java
  public KVStore(Config config) {
      this.config = config;
      this.wal = WriteAheadLog.openWAL(config);
      //启动重新获取Log信息
      this.applyLog();
  }

  public void applyLog() {
      List<WALEntry> walEntries = wal.readAll();
      applyEntries(walEntries);
  }

  private void applyEntries(List<WALEntry> walEntries) {
      for (WALEntry walEntry : walEntries) {
          Command command = deserialize(walEntry);
          if (command instanceof SetValueCommand) {
              SetValueCommand setValueCommand = (SetValueCommand)command;
              kv.put(setValueCommand.key, setValueCommand.value);
          }
      }
  }

  public void initialiseFromSnapshot(SnapShot snapShot) {
      kv.putAll(snapShot.deserializeState());
  }
```



### 实施注意事项

​	在实现 Log 时有一些重要的注意事项。确保写入日志文件的条目实际保存在物理介质上很重要。所有编程语言中提供的文件处理库都提供了一种强制操作系统将文件更改“刷新”到物理介质的机制。在使用刷新机制时需要考虑权衡。

​	刷新每个写入磁盘的日志可以提供强大的持久性保证（这是首先拥有日志的主要目的），但这会严重限制性能并很快成为瓶颈。如果刷新被延迟或异步完成，它会提高性能，但如果服务器在刷新条目之前崩溃，则存在丢失日志条目的风险。大多数实现使用像批处理这样的技术来限制刷新操作的影响。

​	另一个注意事项是确保在读取日志时检测到损坏的日志文件。为了处理这个问题，日志条目通常与 CRC 记录一起写入，然后可以在读取文件时对其进行验证。

​	单个日志文件可能会变得难以管理，并且会迅速消耗所有存储空间。为了解决这个问题，使用了分段日志和低水位标记等技术。

​	预写日志是仅附加的。由于这种行为，在客户端通信失败和重试的情况下，日志可能包含重复条目。应用日志条目时，需要确保忽略重复项。如果最终状态类似于 HashMap，其中对同一键的更新是幂等的，则不需要特殊机制。如果不是，则需要实施某种机制来用唯一标识符标记每个请求并检测重复项。



### 对比 Event Sourcing

用于变更记录的日志的使用类似于事件溯源中的事件日志。 事实上，当一个事件源系统使用它的日志来同步多个系统时，它使用它的日志作为一个复制的日志。 然而，事件溯源系统不仅仅使用它的日志，例如重建历史上先前点的状态的能力。 为此，事件溯源日志是事实的持久来源，并且日志条目会保留很长时间，通常是无限期的。

但是，复制日志的条目仅用于状态同步。 因此，当所有节点都确认更新时，它们可以被丢弃，即低于低水位线。



## 例子

 [Zookeeper](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/persistence/FileTxnLog.java) 和 [RAFT](https://github.com/etcd-io/etcd/blob/master/server/wal/wal.go)等所有 Consensus 算法中的日志实现都类似于预写日志。
[Kafka ](https://github.com/axbaretto/kafka/blob/master/core/src/main/scala/kafka/log/Log.scala) 中的存储实现与数据库中的提交日志的结构类似。
所有数据库，包括像 Cassandra 这样的 nosql 数据库都使用[write ahead log technique ](https://github.com/apache/cassandra/blob/trunk/src/java/org/apache/cassandra/db/commitlog/CommitLog.java)来保证持久性。
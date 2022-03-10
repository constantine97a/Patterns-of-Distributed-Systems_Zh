# Low-Water Mark

预写日志中的索引，显示可以丢弃日志的哪个部分。



[TOC]

## 问题

​		预写日志维护对持久存储的每次更新。 随着时间的推移，它可以无限增长。 分段日志允许一次处理较小的文件，但如果不检查，总磁盘存储可能会无限增长。

## 解决方案

有一种机制来告诉日志记录机器可以安全地丢弃日志的哪一部分。 该机制给出了最低偏移量或低水位线，在此之前可以丢弃日志。 有一个任务在后台运行，在一个单独的线程中，它不断检查日志的哪些部分可以被丢弃并删除磁盘上的文件。

```java
this.logCleaner = newLogCleaner(config);
this.logCleaner.startup();
```

日志清理器可以实现为计划任务

```java
public void startup() {
    scheduleLogCleaning();
}

private void scheduleLogCleaning() {
    singleThreadedExecutor.schedule(() -> {
        cleanLogs();
    }, config.getCleanTaskIntervalMs(), TimeUnit.MILLISECONDS);
}
```



### 基于快照的低水位线

​	大多数共识实现，如 *Zookeeper* 或 *etcd*（在 RAFT 中定义），都实现了快照机制。 在此实现中，存储引擎会定期拍摄快照。 与快照一起，它还存储成功应用的日志索引。 参考 Write-Ahead Log 模式中的简单键值存储实现，快照可以如下所示：

```java
public SnapShot takeSnapshot() {
    Long snapShotTakenAtLogIndex = wal.getLastLogIndex();
    return new SnapShot(serializeState(kv), snapShotTakenAtLogIndex);
}
```

一旦快照成功保存在磁盘上，日志管理器将获得低水位标记以丢弃较旧的日志。

```java
List<WALSegment> getSegmentsBefore(Long snapshotIndex) {
    List<WALSegment> markedForDeletion = new ArrayList<>();
    List<WALSegment> sortedSavedSegments = wal.sortedSavedSegments;
    for (WALSegment sortedSavedSegment : sortedSavedSegments) {
        if (sortedSavedSegment.getLastLogEntryIndex() < snapshotIndex) {
            markedForDeletion.add(sortedSavedSegment);
        }
    }
    return markedForDeletion;
}
```



###  基于时间的低水位线

​		在某些系统中，日志不一定用于更新系统的状态，可以在给定时间窗口后丢弃日志，而无需等待任何其他子系统共享可以删除的最低日志索引。 例如，在 Kafka 这样的系统中，日志维护 *7 周*； 所有消息超过 *7 周*的日志段都将被丢弃。 对于此实现，每个日志条目还包括创建时的时间戳。 然后，日志清理器可以检查每个日志段的最后一个条目，并丢弃早于配置的时间窗口的段。

```java
private List<WALSegment> getSegmentsPast(Long logMaxDurationMs) {
    long now = System.currentTimeMillis();
    List<WALSegment> markedForDeletion = new ArrayList<>();
    List<WALSegment> sortedSavedSegments = wal.sortedSavedSegments;
    for (WALSegment sortedSavedSegment : sortedSavedSegments) {
        if (timeElaspedSince(now, sortedSavedSegment.getLastLogEntryTimestamp()) > logMaxDurationMs) {
            markedForDeletion.add(sortedSavedSegment);
        }
    }
    return markedForDeletion;
}

private long timeElaspedSince(long now, long lastLogEntryTimestamp) {
    return now - lastLogEntryTimestamp;
}
```



## 例子

*  [Zookeeper](https://github.com/apache/zookeeper/blob/master/zookeeper-server/src/main/java/org/apache/zookeeper/server/persistence/FileTxnLog.java)和 [RAFT](https://github.com/etcd-io/etcd/blob/master/wal/wal.go)等所有 Consensus 算法中的日志实现都实现了基于快照的日志清理.
*  [Kafka](https://github.com/axbaretto/kafka/blob/master/core/src/main/scala/kafka/log/Log.scala)中的存储实现遵循基于时间的日志清理.
# Hybrid Clock

使用系统时间戳和逻辑时间戳的组合将版本作为日期时间，有序的。



[TOC]

## 问题

当 Lamport Clock 用作 Versioned Value 中的版本时，客户端不知道存储特定版本时的实际日期时间。 客户端使用日期时间（如 01-01-2020）而不是使用整数（如 1、2、3）访问版本很有用。



## 解决方法

混合逻辑时钟提供了一种方法，使版本像一个简单的整数一样单调递增，但也与实际日期时间有关。 mongodb 或 cockroachdb 等数据库在实践中使用混合时钟。

混合逻辑时钟实现如下：

class HybridClock…

```java
  public class HybridClock {
      private final SystemClock systemClock;
      private HybridTimestamp latestTime;
      public HybridClock(SystemClock systemClock) {
          this.systemClock = systemClock;
          this.latestTime = new HybridTimestamp(systemClock.now(), 0);
      }
```

它维护最新时间作为混合时间戳的一个实例，HybridTimestamp是通过使用系统时间和整数计数器构造的。

class HybridTimestamp…

```java
  public class HybridTimestamp implements Comparable<HybridTimestamp> {
      private final long wallClockTime;
      private final int ticks;
  
      public HybridTimestamp(long systemTime, int ticks) {
          this.wallClockTime = systemTime;
          this.ticks = ticks;
      }
  
      public static HybridTimestamp fromSystemTime(long systemTime) {
          return new HybridTimestamp(systemTime, -1); //initializing with -1 so that addTicks resets it to 0
      }
  
      public HybridTimestamp max(HybridTimestamp other) {
          if (this.getWallClockTime() == other.getWallClockTime()) {
              return this.getTicks() > other.getTicks()? this:other;
          }
          return this.getWallClockTime() > other.getWallClockTime()?this:other;
      }
  
      public long getWallClockTime() {
          return wallClockTime;
      }
  
      public HybridTimestamp addTicks(int ticks) {
          return new HybridTimestamp(wallClockTime, this.ticks + ticks);
      }
  
      public int getTicks() {
          return ticks;
      }
  
      @Override
      public int compareTo(HybridTimestamp other) {
          if (this.wallClockTime == other.wallClockTime) {
              return Integer.compare(this.ticks, other.ticks);
          }
          return Long.compare(this.wallClockTime, other.wallClockTime);
      }
```

混合时钟的使用方式与 Lamport Clock 版本完全相同。 每个服务器都拥有一个混合时钟的实例。

class Server…

```java
  HybridClockMVCCStore mvccStore;
  HybridClock clock;

  public Server(HybridClockMVCCStore mvccStore) {
      this.clock = new HybridClock(new SystemClock());
      this.mvccStore = mvccStore;
  }
```

​	每次写入值时，都会关联一个混合时间戳。 诀窍是检查系统时间值是否及时返回，如果是，则增加另一个表示组件逻辑部分的数字以反映时钟进度。

class HybridClock…

```java
  public synchronized HybridTimestamp now() {
      long currentTimeMillis = systemClock.now();
      if (latestTime.getWallClockTime() >= currentTimeMillis) {
           latestTime = latestTime.addTicks(1);
      } else {
          latestTime = new HybridTimestamp(currentTimeMillis, 0);
      }
      return latestTime;
  }
```

​	服务器从客户端接收到的每个写请求都带有一个时间戳。 接收服务器将自己的时间戳与请求的时间戳进行比较，并将其自己的时间戳设置为两者中较高的一个

class Server…

```java
  public HybridTimestamp write(String key, String value, HybridTimestamp requestTimestamp) {
      //update own clock to reflect causality
      HybridTimestamp writeAtTimestamp = clock.tick(requestTimestamp);
      mvccStore.put(key, writeAtTimestamp, value);
      return writeAtTimestamp;
  }
```

class HybridClock…

```java
  public synchronized HybridTimestamp tick(HybridTimestamp requestTime) {
      long nowMillis = systemClock.now();
      //set ticks to -1, so that, if this is the max, the next addTicks reset it to zero.
      HybridTimestamp now = HybridTimestamp.fromSystemTime(nowMillis);
      latestTime = max(now, requestTime, latestTime);
      latestTime = latestTime.addTicks(1);
      return latestTime;
  }

  private HybridTimestamp max(HybridTimestamp ...times) {
      HybridTimestamp maxTime = times[0];
      for (int i = 1; i < times.length; i++) {
          maxTime = maxTime.max(times[i]);
      }
      return maxTime;
  }
```

​	用于写入值的时间戳返回给客户端。 请求客户端更新其自己的时间戳，然后使用此时间戳发出任何进一步的写入。



class Client…

```java
  HybridClock clock = new HybridClock(new SystemClock());
  public void write() {
      HybridTimestamp server1WrittenAt = server1.write("name", "Alice", clock.now());
      clock.tick(server1WrittenAt);

      HybridTimestamp server2WrittenAt = server2.write("title", "Microservices", clock.now());

      assertTrue(server2WrittenAt.compareTo(server1WrittenAt) > 0);
  }
```



### 具有混合时钟的多版本存储

当值存储在键值存储中时，混合时间戳可以用作版本。 这些值按照版本化值中的讨论进行存储。

class HybridClockReplicatedKVStore…

```java
  private Response applySetValueCommand(VersionedSetValueCommand setValueCommand) {
      mvccStore.put(setValueCommand.getKey(), setValueCommand.timestamp, setValueCommand.value);
      Response response = Response.success(setValueCommand.timestamp);
      return response;
  }
```

class HybridClockMVCCStore…

```java
  ConcurrentSkipListMap<HybridClockKey, String> kv = new ConcurrentSkipListMap<>();

  public void put(String key, HybridTimestamp version, String value) {
      kv.put(new HybridClockKey(key, version), value);
  }
```

class HybridClockKey…

```java
  public class HybridClockKey implements Comparable<HybridClockKey> {
      private String key;
      private HybridTimestamp version;
  
      public HybridClockKey(String key, HybridTimestamp version) {
          this.key = key;
          this.version = version;
      }
  
      public String getKey() {
          return key;
      }
  
      public HybridTimestamp getVersion() {
          return version;
      }
  
      @Override
      public int compareTo(HybridClockKey o) {
          int keyCompare = this.key.compareTo(o.key);
          if (keyCompare == 0) {
              return this.version.compareTo(o.version);
          }
          return keyCompare;
      }
```

​	这些值的读取完全按照版本化键的排序中的讨论。 版本化密钥的排列方式是通过使用混合时间戳作为密钥的后缀来形成自然排序。 此实现使我们能够使用可导航地图 API 获取特定版本的值。
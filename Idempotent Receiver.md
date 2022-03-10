# Idempotent Receiver

[TOC]

唯一识别来自客户端的请求，以便在客户端重试时忽略重复请求

## 问题

客户端向服务器发送请求，但可能得不到响应。 客户端不可能在处理请求之前知道响应是否丢失或服务器崩溃。 为了确保请求得到处理，客户端必须重新发送请求。

如果服务器已经处理了请求并在该服务器将收到来自客户端的重复请求后崩溃，则当客户端重试时。

## 解决方案

通过为每个客户分配一个唯一的 id 来唯一地识别一个客户。 在发送任何请求之前，客户端会向服务器注册自己。

**class ConsistentCoreClient…**

```java
  private void registerWithLeader() {
      RequestOrResponse request
              = new RequestOrResponse(RequestId.RegisterClientRequest.getId(),
              correlationId.incrementAndGet());

      //blockingSend will attempt to create a new connection if there is a network error.
      RequestOrResponse response = blockingSend(request);
      RegisterClientResponse registerClientResponse
              = JsonSerDes.deserialize(response.getMessageBodyJson(),
              RegisterClientResponse.class);
      this.clientId = registerClientResponse.getClientId();
  }
```

​		当服务器收到客户端注册请求时，它会为客户端分配一个唯一的 id。 如果服务器是 Consistent Core，它可以将 Write-Ahead Log 索引分配为客户端标识符。

**class ReplicatedKVStore…**

```java
  private Map<Long, Session> clientSessions = new ConcurrentHashMap<>();

  private RegisterClientResponse registerClient(WALEntry walEntry) {

      Long clientId = walEntry.getEntryIndex();
      //clientId to store client responses.
      clientSessions.put(clientId, new Session(clock.nanoTime()));

      return new RegisterClientResponse(clientId);

  }
```

服务器创建一个会话来存储注册客户端请求的响应。 它还跟踪创建会话的时间，以便可以丢弃不活动的会话，如后面部分所述。

```java
public class Session {
    long lastAccessTimestamp;
    Queue<Response> clientResponses = new ArrayDeque<>();

    public Session(long lastAccessTimestamp) {
        this.lastAccessTimestamp = lastAccessTimestamp;
    }

    public long getLastAccessTimestamp() {
        return lastAccessTimestamp;
    }

    public Optional<Response> getResponse(int requestNumber) {
        return clientResponses.stream().
                filter(r -> requestNumber == r.getRequestNumber()).findFirst();

    }

    private static final int MAX_SAVED_RESPONSES = 5;

    public void addResponse(Response response) {
        if (clientResponses.size() == MAX_SAVED_RESPONSES) {
            clientResponses.remove(); //remove the oldest request
        }
        clientResponses.add(response);
    }

    public void refresh(long nanoTime) {
        this.lastAccessTimestamp = nanoTime;
    }
}
```



​	对于一致性核心，客户端注册请求也作为一致性算法的一部分进行复制。 因此，即使现有领导者失败，客户端注册也是可用的。 然后，服务器还存储发送给客户端的响应以供后续请求。

```asciiarmor
幂等和非幂等请求
需要注意的是，一些请求本质上是幂等的。例如，在键值存储中设置键和值，自然是幂等的。即使多次设置相同的键和值，也不会产生问题。

另一方面，创建租约不是幂等的。如果已创建租约，则重试创建租约的请求将失败。这是个问题。考虑以下场景。客户端发送创建租约的请求；服务器成功创建租约，但随后崩溃，或者在响应发送到客户端之前连接失败。客户端再次创建连接，并重试创建租约；因为服务器已经有一个给定名称的租约，所以它返回一个错误。所以客户认为它没有租约。这显然不是我们期望的行为。

使用幂等接收器，客户端将发送具有相同请求号(RequestId)的租用请求。因为已经处理的请求的响应保存在服务器上，所以返回相同的响应。这样，如果客户端可以在连接失败之前成功创建租约，它将在重试相同请求后得到响应。
```

对于服务器接收到的每个非幂等请求（如上所述），它会在成功执行后将响应存储在客户端会话中。

**class ReplicatedKVStore…**

```java
  private Response applyRegisterLeaseCommand(WALEntry walEntry, RegisterLeaseCommand command) {
      logger.info("Creating lease with id " + command.getName()
              + "with timeout " + command.getTimeout()
              + " on server " + getReplicatedLog().getServerId());
      try {
          leaseTracker.addLease(command.getName(),
                  command.getTimeout());
          Response success = Response.success(walEntry.getEntryIndex());
          if (command.hasClientId()) {
              Session session = clientSessions.get(command.getClientId());
              session.addResponse(success.withRequestNumber(command.getRequestNumber()));
          }
          return success;

      } catch (DuplicateLeaseException e) {
          return Response.error(1, e.getMessage(), walEntry.getEntryIndex());
      }
  }
```

​		客户端将客户端标识符与发送到服务器的每个请求一起发送。 客户端还保留一个计数器，为发送到服务器的每个请求分配请求编号。

**class ConsistentCoreClient…**

```java
  int nextRequestNumber = 1;

  public void registerLease(String name, long ttl) {
      RegisterLeaseRequest registerLeaseRequest
              = new RegisterLeaseRequest(clientId, nextRequestNumber, name, ttl);

      nextRequestNumber++; //increment request number for next request.

      var serializedRequest = serialize(registerLeaseRequest);

      logger.info("Sending RegisterLeaseRequest for " + name);
      blockingSendWithRetries(serializedRequest);

  }
 private static final int MAX_RETRIES = 3;

 private RequestOrResponse blockingSendWithRetries(RequestOrResponse request) {
      for (int i = 0; i <= MAX_RETRIES; i++) {
          try {
              //blockingSend will attempt to create a new connection is there is no connection.
              return blockingSend(request);

          } catch (NetworkException e) {
              resetConnectionToLeader();
              logger.error("Failed sending request  " + request + ". Try " + i, e);
          }
      }

      throw new NetworkException("Timed out after " + MAX_RETRIES + " retries");
  }
```

​		当服务器接收到请求时，它会检查来自同一客户端的具有给定请求号的请求是否已被处理。 如果它找到保存的响应，它会向客户端返回相同的响应，而不需要再次处理请求。

**class ReplicatedKVStore…**

```java
  private Response applyWalEntry(WALEntry walEntry) {
      Command command = deserialize(walEntry);
      if (command.hasClientId()) {
          Session session = clientSessions.get(command.getClientId());
          Optional<Response> savedResponse = session.getResponse(command.getRequestNumber());
          if(savedResponse.isPresent()) {
              return savedResponse.get();
          } //else continue and execute this command.
      }
```



### 使保存的客户端请求过期

​		每个客户端存储的请求不能永远存储。请求可以通过多种方式过期。在 Raft 的参考实现中，客户端保留一个单独的编号来记录成功接收响应的请求编号。然后这个号码会随每个请求一起发送到服务器。服务器可以安全地丢弃任何请求数小于此数的请求。

​		如果保证客户端只有在收到前一个请求的响应后才发送下一个请求，那么一旦服务器收到来自客户端的新请求，它就可以安全地删除所有先前的请求。使用请求管道时会出现问题，因为可能有多个正在进行的请求，客户端可能没有收到响应。如果服务器知道客户端可以拥有的最大正在进行的请求数，它可以只存储那些响应，并删除所有其他响应。例如，Kafka 对其生产者最多可以有五个正在进行的请求，因此它最多存储五个先前的响应。

​		服务器启动一个计划任务来定期检查过期的会话并删除过期的会话。

class ReplicatedKVStore…

```java
  private long heartBeatIntervalMs = TimeUnit.SECONDS.toMillis(10);
  private long sessionTimeoutNanos = TimeUnit.MINUTES.toNanos(5);

  private void startSessionCheckerTask() {
      scheduledTask = executor.scheduleWithFixedDelay(()->{
          removeExpiredSession();
      }, heartBeatIntervalMs, heartBeatIntervalMs, TimeUnit.MILLISECONDS);
  }
  private void removeExpiredSession() {
      long now = System.nanoTime();
      for (Long clientId : clientSessions.keySet()) {
          Session session = clientSessions.get(clientId);
          long elapsedNanosSinceLastAccess = now - session.getLastAccessTimestamp();
          if (elapsedNanosSinceLastAccess > sessionTimeoutNanos) {
              clientSessions.remove(clientId);
          }
      }
  }
```





## 例子

Raft 有参考实现，具有提供线性化动作的幂等性。

Kafka 允许 Idempotent Producer，它允许客户端重试请求并忽略重复请求。

Zookeeper 有 Sessions 和 zxid 的概念，允许客户端恢复。 Hbase 有一个 [hbase-recoverable-zookeeper] 包装器，它遵循 [zookeeper-error-handling] 的指导方针实现幂等操作
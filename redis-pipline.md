title: redis shared pipline
date: 2015/10/16 20:00:00
tags: [redis,jedis]
categories: redis
---

使用redis批量处理多个请求的时候, 可以使用[pipline](http://redis.io/topics/pipelining)提高效率.
但是在使用的时候可能会有一些坑, 我们来看一下.

## 问题
我们先来看问题, 有下面这样一组key, 分布在不同的分片(shared)上, 分别执行exists返回结果如下:

> exists test_k1 => false
> exists test_k2 => false
> exists k1 => false
> exists k2 => false
> exists k3 => true


现在需要分别查test和普通的key, 为了效率我们现在使用pipeline(ShardedJedisPipeline):
``` java
ShardedJedis jedis = shardedJedisPool.getResource();
ShardedJedisPipeline pipeline1 = jedis.pipelined();
ShardedJedisPipeline pipeline2 = jedis.pipelined();
for (String key : keys) {
  if (key.startsWith("test")) {
    pipeline1.exists(key);
  } else {
    pipeline2.exists(key);
  }
}
List<Object> results2 = pipeline2.syncAndReturnAll();
List<Object> results1 = pipeline1.syncAndReturnAll();
System.out.println(results2);
System.out.println(results1);
```
看起来很完美吧...
我们期望返回的结果应该是:
> false false true
> false false

然而实际的结果却是:
> false false false
> true false


## 原因
接下来我们分析原因:

我们跟代码看下jedis是怎么处理分片的pipline.
假设我们一组Redis分片下有多个redis节点, 在执行`PipelineBase.exists(String key)`方法时:
``` java
// redis.clients.jedis.PipelineBase.exists(String)
public Response<Boolean> exists(String key) {
  getClient(key).exists(key);
  return getResponse(BuilderFactory.BOOLEAN);
}
```
有以下几个步骤:

1. A1: 首先会通过一致性hash计算落在哪个节点(client), 并且会缓存client在LinkedList中.
  ``` java
  private Queue<Client> clients = new LinkedList<Client>();

  // redis.clients.jedis.ShardedJedisPipeline.getClient(String)
  protected Client getClient(String key) {
    Client client = jedis.getShard(key).getClient();
    clients.add(client);
    results.add(new FutureResult(client));
    return client;
  }
  ```

2. A2: 执行`exists`方法时会调用`Connection.sendCommand(Command, byte[]...)`方法, 可以看到这里只是把命令和参数写到了`outputStream`但是并没有`flush`.
  ``` java
  // redis.clients.jedis.Connection.sendCommand(Command, byte[]...)
  protected Connection sendCommand(final Command cmd, final byte[]... args) {
    connect();
    Protocol.sendCommand(outputStream, cmd, args);
    pipelinedCommands++;
    return this;
  }
```

3. A3: 最后`getResponse`时会缓存, 也是在LinkedList中.
  ``` java
  private Queue<Response<?>> pipelinedResponses = new LinkedList<Response<?>>();

  // redis.clients.jedis.Queable.getResponse(Builder<T>)
  protected <T> Response<T> getResponse(Builder<T> builder) {
    Response<T> lr = new Response<T>(builder);
    pipelinedResponses.add(lr);
    return lr;
  }
  ```


接下来我们看`ShardedJedisPipeline.syncAndReturnAll()`这个方法
``` java
// redis.clients.jedis.ShardedJedisPipeline.syncAndReturnAll()
public List<Object> syncAndReturnAll() {
  List<Object> formatted = new ArrayList<Object>();
  for (Client client : clients) {
    formatted.add(generateResponse(client.getOne()).get());
  }
  return formatted;
}
```
也是分了几个步骤:
1. B1: 对应上面的A2步骤
`client.getOne()`方法会flush outputStream, 把之前缓存在outputStream中的 **所有** 命令和参数都提交到服务器.
这个flush会被循环遍历执行多次, 但第一次flush完后面再执行的时候outputStream为空就不会提交数据到服务器了.
  ``` java
  public Object getOne() {
    flush();
    pipelinedCommands--;
    return Protocol.read(inputStream);
  }
  ```
2. B2: 对应上面的A3步骤
`generateResponse`从缓存队列里拿一个response设置值以后返回
  ```java
  protected Response<?> generateResponse(Object data) {
    Response<?> response = pipelinedResponses.poll();
    if (response != null) {
      response.set(data);
    }
    return response;
  }
  ```

3. 最后返回总的结果集

整个jedis分片处理pipline的过程就是这样, 最后我们再来看问题.
我们在一个jedis连接里分别开启了两个pipline, 所以第一次`syncAndReturnAll`的时候实际上已经把同一个分片下的所有命令和参数都已经提交到服务器了, 第二次再执行的时候就只是处理结果了.
顺序会乱掉就是因为我们不同的key通过一致性hash落到了不同的节点上.

那这个顺序有规律吗?
没有规律, 但是有规则可循. 主要是看k3落在哪个节点上, 什么时候flush提交对应的命令到服务器.

## 解决办法
1. 用不同的连接打开各自的pipline
``` java
ShardedJedis jedis1 = shardedJedisPool.getResource();
ShardedJedis jedis2 = shardedJedisPool.getResource();
ShardedJedisPipeline pipeline1 = jedis1.pipelined();
ShardedJedisPipeline pipeline2 = jedis2.pipelined();
```

2. 同一个连接的piplie一定要先关闭才能打开新的
``` java
for (String key : keys) {
  if (key.startsWith("test")) {
    pipeline1.exists(key);
    keys.remove(key);
  }
}
List<Object> results1 = pipeline1.syncAndReturnAll();
System.out.println(results1);
for (String key : keys) {
  if (!key.startsWith("test")) {
    pipeline1.exists(key);
  }
}
List<Object> results2 = pipeline2.syncAndReturnAll();
System.out.println(results2);
```

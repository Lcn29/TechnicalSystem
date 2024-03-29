# Redis - 实战

## 1 缓存

使用 Redis 作为缓存的时候, 一般流程是这样的  
> 1. **命中缓存**, 返回 Redis 的存储结果 
> 2. **未命中缓存**, 到数据库查询, 查询到结果的话, 将结果写入 Redis, 返回数据库的存储结果


### 1.1 缓存一致性问题

因为这些数据是很少修改的, 所以在绝大部分的情况下可以命中缓存。 但是, 一旦被缓存的数据发生变化的时候, 既要操作数据库的数据, 也要操作 Redis 的数据。  

现在我们有两种选择  
> 1. 先操作 Redis 的数据再操作数据库的数据
> 2. 先操作数据库的数据再操作 Redis 的数据

无论哪种选择, 需要达到的效果都是 2 个操作同时成功。  
但是 Redis 的数据和数据库的数据是不可能通过事务达到统一的, 我们只能根据**相应的场景**和**所需要付出的代价**来采取一些措施降低数据不一致的问题出现的概率, 在数据一致性和性能之间取得一个权衡。


#### 1.1.1 Redis 操作

操作 Redis 数据的话, 可以在细分为 2 种
> 1. 直接更新 Redis
> 2. 直接删除 Redis, 后续在访问的时候, 进行填充

至于选择哪一种处理方式, 主要还是考虑更新缓存的代价。  
更新缓存之前, 是不是要经过其他表的查询, 接口调用, 计算才能得到最新的数据, 而不是直接从数据库拿到的值。  
如果是的话, 建议直接删除缓存, 

在一般情况下, 推荐使用删除的方案。
原因: 
> 1. 删除一个数据, 相比更新一个数据更加轻量级，出问题的概率更小。 有时候缓存不是直接读取表数据就行的, 经过其他表的查询, 接口调用, 计算才能得到最新的数据。
> 2. 不是所有的缓存数据都是频繁访问的, 更新后的缓存可能会长时间不被访问。下次查询命中再填充缓存，是一个更好的方案，类似于 Lazy Loading 的思想。


#### 1.1.2 先更新数据库，再删除缓存

异常情况
> 1. 更新数据库失败，程序捕获异常，不会走到下一步，所以数据不会出现不一致
> 2. 更新数据库成功，删除缓存失败。数据库是新数据，缓存是旧数据，发生了不一致的情况

解决方案一: **重试的机制**  

> 1. 如果删除缓存失败, 捕获这个异常, 把需要删除的 key 发送到消息队列
> 2. 自己创建一个消费者消费，尝试再次删除这个 key

缺点: 对业务代码造成入侵


解决方案二: **异步更新缓存**  

> 1. 因为更新数据库时会往 binlog 写入日志, 可以通过一个服务来监听 binlog 的变化 (比如阿里的 cancal)
> 2. 然后在客户端完成删除 key 的操作
> 3. 如果删除失败的话， 再发送到消息队列

#### 1.1.3 先删除缓存，再更新数据库

异常情况: 
> 1. 删除缓存失败，程序捕获异常，不会走到下一步，所以数据不会出现不一致
> 2. 删除缓存成功，更新数据库失败, 因为以数据库的数据为准，所以不存在数据不一致的情况 ?

在高并发的情况, 异常情况 2 还是会出现问题
> 1. 线程 A 需要更新数据，首先删除了 Redis 缓存
> 2. 线程 B 查询数据，发现缓存不存在，到数据库查询旧值，写入 Redis，返回
> 3. 线程 A 更新了数据库
> 4. 这个时候, Redis 是旧的值，数据库是新的值，发生了数据不一致的情况


能不能让对同一条数据的访问串行化呢？  
代码肯定保证不了，因为有多个线程，即使做了任务队列也可能有多个服务实例。  
数据库也保证不了，因为会有多个数据库的连接, 只有一个数据库只提供一个连接的情况下，才能保证读写的操作是串行的。

解决方案: **延迟双删**  
在写入数据之后，再删除一次缓存。
> 1. 删除缓存
> 2. 更新数据库
> 3. 休眠 500ms（这个时间，依据读取数据的耗时而定）
> 4. 再次删除缓存



### 1.2 高并发下的热点数据问题

有两种情况可能会导致热点问题的产生: 
> 1. 一个是用户集中访问的数据, 比如抢购的商品, 明星结婚和明星出轨的微博
> 2. 在数据进行分片的情况下, 负载不均衡, 超过了单个服务器的承受能力

热点问题可能引起缓存服务的不可用，最终造成压力堆积到数据库


#### 1.2.1 热点数据的发现

**手动统计**  

在项目中手动统计 Redis key 计算。这样每个地方都要修改, 重复代码较多, 同时只能统计当前客户端的热点 key

**代理层进行统计**  
TwemProxy 和 Codis 这些中间件有提供 Redis key 统计的功能

**基于 Redis的 monitor 的命令**  
Redis 有个一个 monitor 的命令, 可以监控到 Redis 执行的命令

```java
jedis.monitor(new JedisMonitor() {

   @Override
   public void onCommand(String command) {
       System.out.println("执行的命令" + command);
   }
});
```

Factbook 的 开源项目 redis-faina 就是基于这个原来实现的, 可以分析 moitor 的数据

```sh
redis-cli -p 6379 monitor | head -n 100000 | ./redis-faina.py
```

这种方法也有 2 个问题 
> 1. monitor 命令在高并发的场景下, 会影响性能
> 2. 只能统计一个 Redis 节点的热点 key


**基于机器层面监控**  

通过对 TCP 协议进行抓包, 同样也有一些开源软件 ELK 的 packetbeat 插件

#### 1.2.2 缓存雪崩

Redis 的大量热点数据同时过期 (失效), 因为设置了相同的过期时间, 刚好这个时候 Redis 请求的并发量又很大, 导致所有的请求落到数据库

解决方案
> 1. 加互斥锁或者使用队列, 针对同一个 key 只允许一个线程到数据库查询
> 2. 缓存定时预先更新, 避免同时失效
> 3. 通过加随机数, 使 key 在不同的时间过期
> 4. 缓存永不过期

#### 1.2.2 缓存穿透

在高并发下, 频繁地查询缓存和数据库中都没有的数据，由于缓存是不命中, 每次请求都落到了数据上，失去了缓存的意义。

解决方案
> 1. 缓存空数据
> 2. 缓存特殊字符串, 比如 &&

在缓存中缓存一个空字符串或者特殊的字符串, 在应用中里面可以拿到这个特殊字符串的时候, 就知道数据库没有值，就不需要再到数据库查询

上面能解决的情况是 应用重复查询同一个不存在的值的情况。如果应用每一次查询的不存在的值是不一样, 每次都缓存特殊字符串也没有作用。

解决: 
利用位图, 一个有序的数组, 存在对应的 key 是否存在, 0: 数据不存在, 1: 数据存在

key 的问题
> 1. key 的长度是不固定的, 不同 key 的输入，可以得到固定长度的输出
> 2. 转换成下标的时候，希望在这个邮箱数组里面是分布均匀的

可以利用 Hash 函数解决, 但是引入 Hash 函数会引入一个新的问题 **Hash 冲突或 Hash 碰撞**

Hash 冲突的解决  
> 1. 扩大数组的长度, 也就是位图的容量, 函数是分布均匀的, 所以位图容量越大, 同一个位置发生 Hash 碰撞的概率越低, 但是越大的位图容量, 越大的内存消耗
> 2. 多个不同的 Hash 计算, 每次结果的位置都设置为 1, 填满位图的需要更多空间, 计算需要消耗时间

**布隆过滤器**

即使的多次 hash 计算，还是可能出现 hash 冲突。  
比如三个 hash 函数，对元素 a, 进行计算, 得到的三次的位置都是为 1, 那么 a 会被判断为存在。但是这时候还是可能有误判性, Hash 碰撞是不可避免的
比如对元素 b, 进行计算, 得到的三次的位置为 1, 0, 1, 这是完全可以肯定 b 一定不存在。

布隆过滤器的特点
> 1. 如果布隆过滤器判断元素在集合中存在, 不一定存在
> 2. 如果布隆过滤器判断不存在，一定不存在
> 3. 如果元素实际存在, 布隆过滤器一定判断为存在
> 4. 如果元素实际不存在, 布隆过滤器可能判断为存在

基于第二个的特性, 可以解决缓存穿透的问题 (谷歌的 Guava)

> 1. 项目启动时, 加载数据库中所有的数据库
> 2. 布隆在实例的使用中，先查询布隆，布隆说不存在, 那么一定不存在，如果说存在, 那么再走之前的流程

其他的使用场景， 爬虫，url 跑过了, 不需要爬了
邮箱服务器，发送垃圾邮件的账号叫做 spamer， 判断一个账号是不是为 spamer。


#### 1.2.3 缓存击穿

某一个热点 key，在缓存过期的一瞬间，同时有大量的请求打进来，由于此时缓存过期了，所以请求最终都会走到数据库，造成瞬时数据库请求量大、压力骤增。

解决方案  
> 1. 加互斥锁
> 2. 热点数据不过期


## 2 分布式锁

> 1. 互斥 set nx
> 2. 死锁，超时
> 3. 持有锁的, 才能释放锁, 线程id
> 4. 可重入
> 5. 自动续期


---
title: "Redis Interview"
date: 2023-08-02T19:07:39+08:00
draft: false
author: ["Younger"] #作者
categories:
- 分类1
- 分类2
tags:
- redis
- interview
description: "" #描述
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
draft: false # 是否为草稿
comments: true #是否展示评论
showToc: true # 显示目录
TocOpen: true # 自动展开目录
hidemeta: false # 是否隐藏文章的元信息，如发布日期、作者等
disableShare: true # 底部不显示分享栏
showbreadcrumbs: true #顶部显示当前路径
cover:
image: "" #图片路径：posts/tech/文章1/picture.png
caption: "" #图片底部描述
alt: ""
relative: false
---

### 1. 持久化怎么实现的
- aof：append only file。持续写文件到buffer ring中，然后根据参数完成fsync操作。aof文件会有重写操作，节省空间，以及宕机恢复操作的时间。
- rdb：内存快照。对某一时刻的redis的数据情况进行快照存储。


### 2. zset怎么做延迟队列
zadd key Val score：job数据存储在Val，score代表任务执行时间。
zrangebyscore time，拿当前的时间戳做比较。取可执行的任务。
zremrange 移除这些任务。

### 3. 哪些操作会触发Redis单线程处理IO请求性能瓶颈。
耗时的操作包括：
- **操作bigkey**：写入一个bigkey在分配内存时需要消耗更多的时间，同样，删除bigkey释放内存同样会产生耗时
- **使用复杂度过高的命令**：例如SORT/SUNION/ZUNIONSTORE，或者O(N)命令，但是N很大，例如lrange key 0 -1一次查询全量数据
- **大量key集中过期**：Redis的过期机制也是在主线程中执行的，大量key集中过期会导致处理一个请求时，耗时都在删除过期key，耗时变长
- **淘汰策略**：淘汰策略也是在主线程执行的，当内存超过Redis内存上限后，每次写入都需要淘汰一些key，也会 造成耗时变长。
- **AOF刷盘开启always机制**：每次写入都需要把这个操作刷到磁盘，写磁盘的速度远比写内存慢，会拖慢Redis的性能
- **主从全量同步生成RDB**：虽然采用fork子进程生成数据快照，但fork这一瞬间也是会阻塞整个线程的，实例越大，阻塞时间越久

- Redis在6.0推出了多线程，可以在高并发场景下利用CPU多核多线程读写客户端数据，进一步提升server性能。当然，只针对客户端的读写是并行的，每个命令的真正操作依旧是单线程的


### 4. redis分布式锁用过吗？说下咋用的，哪些场景需要用

   ```c
   SET lock_key unique_Val NX PX 10000 
   ```

   - lock_key 就是 key 键；
   - unique_Val 是客户端生成的唯一的标识，区分来自不同客户端的锁操作；
   - NX 代表只在 lock_key 不存在时，才对 lock_key 进行设置操作；
   - PX 10000 表示设置 lock_key 的过期时间为 10s，这是为了避免客户端发生异常而无法释放锁。

   解锁时因为要判断是否是加锁者来解锁，需要完成判断和解锁两个操作。故借助lua脚本完成。

   难点：

   1. 超时时间不好设置。设置短了，任务还未执行完成，其他的线程就可以拿到锁。设置长了，影响性能。为了解决这个问题可以加一个看门狗机制。也就是新增一个守护线程专门监控这种情况。当任务未执行完成，锁快要过期时，续约锁时间。当任务执行完成，但锁还未过期，直接销毁锁即可
   2. 主从模式的时候，redis异步复制数据，这将导致分布式锁的不可靠性。如果在主节点获取锁后，其他节点还未同步到锁。主节点宕机了。那么新的主节点还是可以获取到锁。那么如何解决呢。就是redis的redlock红锁
   

   ***Redis的redlock***

   为了保证集群环境下分布式锁的可靠性，Redis 官方已经设计了一个分布式锁算法 Redlock（红锁）。

   它是基于**多个 Redis 节点**的分布式锁，即使有节点发生了故障，锁变量仍然是存在的，客户端还是可以完成锁操作。官方推荐是至少部署 5 个 Redis 节点，而且都是主节点，它们之间没有任何关系，都是一个个孤立的节点。

   Redlock 算法的基本思路，**是让客户端和多个独立的 Redis 节点依次请求申请加锁，如果客户端能够和半数以上的节点成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁，否则加锁失败**。

   这样一来，即使有某个 Redis 节点发生故障，因为锁的数据在其他节点上也有保存，所以客户端仍然可以正常地进行锁操作，锁的数据也不会丢失。

   Redlock 算法加锁三个过程：

   - 第一步是，客户端获取当前时间（t1）。
   - 第二步是，客户端按顺序依次向 N 个 Redis 节点执行加锁操作：
     - 加锁操作使用 SET 命令，带上 NX，EX/PX 选项，以及带上客户端的唯一标识。
     - 如果某个 Redis 节点发生故障了，为了保证在这种情况下，Redlock 算法能够继续运行，我们需要给「加锁操作」设置一个超时时间（不是对「锁」设置超时时间，而是对「加锁操作」设置超时时间），加锁操作的超时时间需要远远地小于锁的过期时间，一般也就是设置为几十毫秒。
   - 第三步是，一旦客户端从超过半数（大于等于 N/2+1）的 Redis 节点上成功获取到了锁，就再次获取当前时间（t2），然后计算计算整个加锁过程的总耗时（t2-t1）。如果 t2-t1 < 锁的过期时间，此时，认为客户端加锁成功，否则认为加锁失败。

   可以看到，加锁成功要同时满足两个条件（*简述：如果有超过半数的 Redis 节点成功的获取到了锁，并且总耗时没有超过锁的有效时间，那么就是加锁成功*）：

   - 条件一：客户端从超过半数（大于等于 N/2+1）的 Redis 节点上成功获取到了锁；
   - 条件二：客户端从大多数节点获取锁的总耗时（t2-t1）小于锁设置的过期时间。

   加锁成功后，客户端需要重新计算这把锁的有效时间，计算的结果是「锁最初设置的过期时间」减去「客户端从大多数节点获取锁的总耗时（t2-t1）」。如果计算的结果已经来不及完成共享数据的操作了，我们可以释放锁，以免出现还没完成数据操作，锁就过期了的情况。

   加锁失败后，客户端向**所有 Redis 节点发起释放锁的操作**，释放锁的操作和在单节点上释放锁的操作一样，只要执行释放锁的 Lua 脚本就可以了。

   

### 5. 缓存穿透、雪崩的解决方案

   缓存穿透：同一时刻大量请求访问不存在的数据。

   - 缓存空值：可以在缓存中缓存空值来防止压力给到mysql。但是这也不是万能的。缓存空值会浪费我们的空间，如果缓存的空值比较多。还会影响redis的命中率。淘汰了我们正常的热点数据。所以生产环境，我们需要监控一些null空值的数量。避免浪费空间和对正常热数据的影响。我们还可以使用布隆过滤器来处理。
   - 布隆过滤器：查询布隆过滤器，判断数据是否存在。
     - 用法：存储数据时，布隆过滤器会先以hash函数来判断数据的落位bit，以bit位0,1来标示数据是否存在。读取数据时，就可以根据0，1判断是否数据是否存在，而不用去缓存和数据库查找数据。
     - 缺点：1.有hash函数就有hash碰撞，所以就有可能将并不在集合中的元素判断在里面。2.不支持删除元素
     - 解决方案：针对第一点，可以采用多个hash函数来确定bit位，减少误差。多个hash都是1的时候，数据真正存在。(存在疑问？多个hash函数不是更加加大了碰撞的几率嘛。确实是这样的。但是前期数据量比较少的时候，多个hash函数可以显著增加误判率，但是随着越来越多的“1”被占用。碰撞的几率也会越来越大。所以需要去关注布隆过滤器的负载量)
     - 另外布隆过滤器有误判的可能。布隆过滤器判断存在数据存在时，不一定存在(因为hash碰撞占位)。数据不存在时，肯定是不存在，这个是准确的。

   

   缓存雪崩：有大量的key在同一时间过期，或者redis故障

   - 分散key的失效时间。设置过期时间时，可以加一个时间偏移量
   - 设置key不过期。这个方式比较暴力。但是key并不是真正的会一直存在。redis的多种内存淘汰机制可能会清理掉数据。因此我们需要其他的机制来监控缓存的数据。
     1. 可以将缓存更新的任务交由后台线程完成。定时扫描key是否存在。是否需要更新最新数据。业务不负责缓存数据的更新。但是定时扫描的定时需要trade-off，均衡业务的容忍度。
     2. 业务请求时发现数据不存在，往mq中写一条消息。交由消费者完成数据的更新和加载。当然执行前可以检测下，数据是否已经存在。
   - 互斥锁。保证不会有大量的请求打到mysql。只有一个请求完成cache aside方式的数据回写。

   

   缓存击穿：热点key数据过期

   - 互斥锁
   - 任务key不过期

   

### 6. 如何保证缓存-db一致性

   - cache aside方式。选择先更新db，再删除缓存。另外可加消费mysql binlog 的方式或者后台任务。保证更新db和删除操作都执行成功。

### 7. redis怎么做限流

   可以依赖redis单线程和其支持的数据结构完成限流操作。例如：依赖list结构实现令牌桶，固定时间生成token，当令牌token为空时，新来的请求阻塞或者直接返回失败。下面是go-zero框架lua脚本实现令牌桶的方式：

   ```lua
   script = `local rate = tonumber(ARGV[1])
   local capacity = tonumber(ARGV[2])
   local now = tonumber(ARGV[3])
   local requested = tonumber(ARGV[4])
   
   //过期到下次填充前的时间
   local fill_time = capacity/rate
   local ttl = math.floor(fill_time*2)
   
   local last_tokens = tonumber(redis.call("get", KEYS[1]))
   if last_tokens == nil then
       last_tokens = capacity
   end
   
   local last_refreshed = tonumber(redis.call("get", KEYS[2]))
   if last_refreshed == nil then
       last_refreshed = 0
   end
   
   //计算上次更新token的时间差，计算出需要补多少token。判断是否满足请求需要的request
   //计算剩余的token量
   local delta = math.max(0, now-last_refreshed)
   local filled_tokens = math.min(capacity, last_tokens+(delta*rate))
   local allowed = filled_tokens >= requested
   local new_tokens = filled_tokens
   if allowed then
       new_tokens = filled_tokens - requested
   end
   
   redis.call("setex", KEYS[1], ttl, new_tokens)
   redis.call("setex", KEYS[2], ttl, now)
   
   return allowed`
   ```

   

### 8. redis中server和client通信方式？

   - 简单文本通信协议

### 9. redis淘汰策略
- noeviction（不淘汰策略）：当内存不足以容纳新写入数据时，新写入操作会报错"OOM（Out of Memory）"。默认情况下，Redis就采用了此种淘汰策略。
- allkeys-lru（最近最少使用淘汰策略）：Redis会根据键的最近最少使用时间来淘汰数据。当内存不足以容纳新写入数据时，从所有键中选择最近最少使用的数据淘汰。
- volatile-lru（设置过期时间的最近最少使用淘汰策略）：Redis会根据键的最近最少使用时间来淘汰键值对，但只会针对设定了过期时间的键值对进行淘汰。
- volatile-ttl（设置过期时间淘汰策略）：Redis会根据键值对的剩余存活时间来淘汰数据。当内存不足以容纳新写入数据时，优先淘汰存活时间较短的数据。此种淘汰策略适用于只关心最近活跃数据的场景。


### 10. 缓存预热？如何做？

    缓存预热：redis启动前缓存热门数据

    nginx+lua将流量请求打到kafka中，让kafka抗高并发的量。然后数据写入storm中，storm完成M份Top N数据的生成。然后聚合生成热门数据存到zk中，新起任务完成读取数据，写入redis。完成预热缓存。

### 11. Redis 应用题: 设计一个每日日活用户统计功能
    需求: 我们的网站每天用户访问之后，UserID 将会被收集到我们的 Redis 数据存储（需要你思考如何存储），我们的后台需要支持一个功能:
    1.API - 支持查询 某一天访问的用户 & 前一天并没有访问我们的网站
    Request : {"date": "20230216"}
    Response: {"user_ids": [1,3,4]}
    2.API 拓展 - 在支持(1) 的基础上，支持根据指定批量日期范围查询这个数据
    Request : {"dates": ["20230216", "20230217"]}
    Response: {"list": [{"date": "20230216", "user_ids": [1,3,4]}, {"date": "20230216", "user_ids": [2,9]}]}
    数据量级:
    Level0: 每日访问用户数在 1K 以内
    Level1: 每日访问用户数在 10K 以内
    考察点1: redis 数据存储如何设计，应该使用什么数据结构，怎么记录和实现功能

    答案：
    1.按照天的维度将用户放到set结构中(key是日期，value是UserID)，针对查询需求，可以将两天的set做diff操作，生成到一个new set中。
    2.如果要支持指定批量日期范围查询，首先需要明确最早日期，日期数量是否有限制？qps是多少(每条执行多少请求和计算量有关)，如果不高的话，可以实时diff计算，
    一般情况下，latency会比较长。
    latency过长的解决方案：
    1.缓存：new set已经放入到了redis中，如果请求中的日期过多，影响了latency的性能，那么可以使用进程内缓存，此时就要考虑缓存的大小如何设计？(设计的标准应该是满足当前qps，多少数据从redis拿，多少从进程缓存拿，可以满足qps需求)，考虑设计缓存淘汰方式(LRU、LFU or 其他？，这里选择LFU，淘汰使用频率最少的比较合理)。
    2.升级机器配置，升级cpu配置，升级redis配置。使得机器cpu不是瓶颈(内存应该不是瓶颈，不需要考虑)，升级redis的cpu配置，主从或者分片。

    数量级问题需要注意的点：1k以内的set，userid是int类型的话8byte，一个日期的key存储占8k(set是hash结构和intset，那就按照正常的数组大小评估，就是8k)，不算是大key(大key的标准和结构中的数量和元素的大小相关)，那么diff完成的也不是大key。这个时候就还是考虑latency的问题了，按照上面方案解决就行。

    如果真的数量级到了大key，那么一般方案也就是拆分成多个key，此时内存友好，但是cpu就不友好了。解决的话，可以升级配置。或者redis集群部署，分散压力。

### 12. Redis的主从同步流程
- 全量同步：
  - 触发条件：首次同步，master节点进程id发生变化，master缓存中没有slave当前同步的offsetID(滞后太多)
  - 同步过程：
     1. slave向master发起链接建立。
     2. slave向master发起sync命令，请求复制数据
     3. master收到PSYNC命令后，会在后台进行数据持久化；
        - 通过bgsave生成最新的rdb快照文件
        - bgsave期间，将客户端发送的命令（会修改数据集的）缓存到内存中；
     4. 持久化完毕后，master将这份RDB数据发送给slave；
     5. slave会把接收到的数据进行持久化生成RDB，然后再加载到内存中。
     6. master继续将之前缓存在内存中的命令通过socket发送给slave。
- 增量，断点续传同步：
  - 同步过程：
    1. 断开重连后，slave向master发送psync命令同步数据
    2. master会在内存中维护命令队列repl backlong buff，并且master拥有各slave的数据下标offset，slave有master进程id
    3. 重连后，从下标开始接受master缓冲队列的命令，完成同步
    4. 如果master进程节点id变化，或者从节点数据下标offset太久，已经不在master的缓存队列里，则会进行一次全量数据复制
   
注意点：
1. 主从复制时，如果有过多的从节点，为了缓解主从复制风暴，多个从节点同事复制主节点导致主节点压力过大。可以让部分从节点作为假主，关联其他从库。也就是主-从(假主)-从
2. RDB用于全量复制，全量复制完后，后续的执行指令，master会将repl backlong buff中缓存的指令通过socket发送给从库。
3. 主从复制的时候，针对slave进行全量数据同步，slave在加载master的RDB文件前，会运行flushall来清理自己当前的数据。slave-lazy-flush参数设置决定是否采用异步flush的机制。异步flush清空从节点本地数据库，可减少全量同步耗时，从而减少主库因输出缓冲区爆炸引起的内存使用增长。

### 13. Redis多线程网络IO模型(下面的内容都来自潘少的[blog](https://strikefreedom.top/archives/multiple-threaded-network-model-in-redis))
   1. redis为什么这么快？
      - C语言支持
      - 内存型数据库
      - 网络IO多路复用
      - 单线程模型，避免多线程频繁上下文切换，额外的同步机制
      - 高性能的数据结构设计
   2. redis为什么选择单线程
      - 避免过多的上下文切换开销
        - 多线程调度必然会有上下文切换的开销，具体的开销在于程序计数器，堆栈指针和程序状态等一系列寄存器置换，程序堆栈重置等，还会影响cpu cache，TLB快表的淘汰置换。单一进程的多线程切换还好一些，因为多个线程共享进程地址空间。对比跨进程(多进程)的调度，那么开销就会很大，需要切换掉整个进程的地址空间。
      - 避免同步机制的开销
        - 多线程模型下，势必会有一些同步机制，用来保证数据的状态。简单来说就是需要加锁了。因为redis有丰富的数据类型，此时锁的粒度又会有很多取舍。这样不仅会有锁的开销，程序的复杂度也会极具增长。与redis简单设计的理念不符
   3. redis多线程发展历程
      - redis 4.0的时候引入了多线程处理异步任务。这些任务包括数据清理，数据备份bgsave等。主要的目的避免一些阻塞的命令，影响单线程执行模型。比如新增了UNLINK、FLUSHALL ASYNC等。但网络IO处理还是单线程的。
      - redis 6.0。
        -  Reactor 单进程 / 线程，不用考虑进程间通信以及数据同步的问题，因此实现起来比较简单，这种方案的缺陷在于无法充分利用多核 CPU，而且处理业务逻辑的时间不能太长，否则会延迟响应，所以不适用于计算机密集型的场景，适用于业务处理快速的场景，比如 Redis（6.0之前 ） 采用的是单 Reactor 单进程的方案。
        - 发展到这个阶段，多线程来提升整体性能已经是必定的路了。因为redis的单线程模型会导致系统消耗很多cpu在网络IO上，从而降低了吞吐量，那么为了解决这个问题，只能1.优化网络IO 2.提高机器内存读写速度。显然完成条件1是首先考虑的。优化1是手段有1.零拷贝 2.利用多核优势。零拷贝无法应对redis这一类复杂的网络IO场景。所以顺理成章的我们需要利用多核优势了。
        - 6.0的时候，增加多线程来处理网络IO的读写，就是多线程去处理IO读写数据，但执行操作还是单线程的。官方的数据显示增加多线程后，性能提升一倍。
   4. cpu亲和性
      - redis6.0后，多线程执行很多异步任务，并发度已经有很大提升，而且redis是对吞吐量和延迟很敏感的系统。故此需要考虑cpu切换时的性能损耗，为了避免这个情况，redis启动的时候设置了cpu亲和性，也就是绑核，将某些线程/进程绑定到固定的cpu上，其他任务就不会占用这些任务的cpu时间片，能更高效率的工作，也能极大限度借助cpu cache提升性能。
   5. free lock的设计
      - 引入多线程，不可避免的会有共享资源竞争，就会引出锁来保护共享资源。但是redis通过原子操作和时空交错访问来实现无锁的多线程模型。记住redis6.0的源码中，IO读写多线程只完成读写数据的操作。
      - 下面是多线程读的流程。
        - 当有client就绪时，就会将client放入队列clients_pending_read
        - 主线程遍历待读取的client队列，通过 RR 策略把所有任务分配给 I/O 线程和主线程去读取和解析客户端命令。
        - 忙轮询等待所有 I/O 线程完成任务。
        - 完成任务后，再遍历 clients_pending_read，执行所有 client 的命令。
        - 执行完成后，需要回复的内容已经写入每个client的buf中了。然后将client放入clients_pending_write队列中，等待调度，将响应内容返回给客户端。
      - 下面是多线程写的流程：
        - 检查当前任务负载，如果当前的任务数量不足以用多线程模式处理的话，则休眠 I/O 线程并且直接同步将响应数据回写到客户端。
        - 唤醒正在休眠的 I/O 线程（如果有的话）。
        - 遍历待写出的 client 队列 clients_pending_write，通过 RR 策略把所有任务分配给 I/O 线程和主线程去将响应数据写回到客户端。
        - 忙轮询等待所有 I/O 线程完成任务。
        - 最后再遍历 clients_pending_write，为那些还残留有响应数据的 client 注册命令回复处理器 sendReplyToClient，等待客户端可写之后在事件循环中继续回写残余的响应数据。
      - 为什么是lock free呢？
        - 主线程和 I/O 线程之间共享的变量有三个：io_threads_pending 计数器、io_threads_op I/O 标识符和 io_threads_list 线程本地任务队列。io_threads_pending 是原子变量，不需要加锁保护，io_threads_op 和 io_threads_list 这两个变量则是通过控制主线程和 I/O 线程交错访问来规避共享数据竞争问题：I/O 线程启动之后会通过忙轮询和锁休眠等待主线程的信号，在这之前它不会去访问自己的本地任务队列 io_threads_list[id]，而主线程会在分配完所有任务到各个 I/O 线程的本地队列之后才去唤醒 I/O 线程开始工作，并且主线程之后在 I/O 线程运行期间只会访问自己的本地任务队列 io_threads_list[0] 而不会再去访问 I/O 线程的本地队列，这也就保证了主线程永远会在 I/O 线程之前访问 io_threads_list 并且之后不再访问，保证了交错访问。io_threads_op 同理，主线程会在唤醒 I/O 线程之前先设置好 io_threads_op 的值，并且在 I/O 线程运行期间不会再去访问这个变量。
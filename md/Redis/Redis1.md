## 单线程/并发竞争

- 为什么Redis是单线程的
  - CPU不是Redis的瓶颈，Redis的瓶颈最有可能是机器内存的大小或者网络带宽
  - 这里我们一直在强调的单线程，只是在处理我们的网络请求的时候只有一个线程来处理
    - Redis进行持久化的时候会以子进程或者子线程的方式执行
  - 因为是单一线程，所以同一时刻只有一个操作在进行，所以，耗时的命令会导致并发的下降，不只是读并发，写并发也会下降。而单一线程也只能用到一个CPU核心
    - 我们使用单线程的方式是无法发挥多核CPU 性能，不过我们可以通过在单机开多个Redis 实例来完善
    - 在多处理器情况下，不能充分利用其他CPU。可以的解决方法是开启多个redis服务实例，通过复制和修改配置文件，可以在多个端口上开启多个redis服务实例，这样就可以利用其他CPU来处理连接流
    - 所以可以在同一个多核的服务器中，可以启动多个实例，组成master-master或者master-slave的形式，耗时的读命令可以完全在slave进行
    - 由于是单线程模型，Redis 更喜欢大缓存快速 CPU， 而不是多核

- 并发竞争

  - 多客户端同时并发写一个key，可能本来应该先到的数据后到了，导致数据版本错了。或者是多客户端同时获取一个key，修改值之后再写回去，只要顺序错了，数据就错了。

  - 并发写竞争解决方案

    - 利用redis自带的incr命令

      - 数字值在 Redis 中以字符串的形式保存

      - 可以通过组合使用 INCR 和 EXPIRE，来达到只在规定的生存时间内进行计数(counting)的目的。

      - 客户端可以通过使用 GETSET命令原子性地获取计数器的当前值并将计数器清零

      - 使用其他自增/自减操作，比如 DECR 和 INCRBY ，用户可以通过执行不同的操作增加

        或减少计数器的值，比如在游戏中的记分器就可能用到这些命令

    - 独占锁的方式，类似操作系统的mutex机制

    - 乐观锁的方式进行解决（成本较低，非阻塞，性能较高）

      - 使用redis的命令watch进行构造条件

      - watch这里表示监控该key值，后面的事务是有条件的执行，如果watch的key对应的value值被修改了，则事务不会执行

      - > T1
        > set key1 value1
        > 初始化key1
        > T2
        > watch key1
        > 监控 key1 的键值对
        > T3
        > multi
        > 开启事务
        > T4
        > set key2 value2
        > 设置 key2 的值
        > T5
        > exec
        > 提交事务，Redis 会在这个时间点检测 key1 的值在 T2 时刻后，有没有被其他命令修改过，如果没有，则提交事务去执行

    - 针对客户端来的，在代码里要对redis操作的时候，针对同一key的资源，就先进行加锁（java里的synchronized或lock）。

    - 利用redis的set（使用set来获取锁, Lua 脚本来释放锁）

      - 考虑可以使用SETNX，将 key 的值设为 value ，当且仅当 key 不存在。

      - ```java
        /* 第一个为key，我们使用key来当锁名
           第二个为value，我们传的是uid，唯一随机数，也可以使用本机mac地址 + uuid
           第三个为NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作 
        	第四个为PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定 
        	第五个为time，代表key的过期时间，对应第四个参数 PX毫秒，EX秒
        */
        String result = jedis.set(key, value, "NX", "PX", expireMillis);
        if (result != null && result.equalsIgnoreCase("OK")) {
        	flag = true;
        }
        
        
        // ---
        
        // 执行脚本的常用命令为 EVAL。 
        // 原子操作：Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入
        // redis.call 函数的返回值就是redis命令的执行结果
        
        
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script,Collections.singletonList(fullKey),Collections.singletonList(value));
        if (Objects.equals(UNLOCK_SUCCESS, result)) {
            flag = true;
        }
        
        
        ```

        







## 多路I/O复用模型

- 复用是指一个线程可以服务多条IO流，I/O multiplexing 这里面的 multiplexing 指的其实是在单个线程通过记录跟踪每一个Socket(I/O流)的状态来同时管理多个I/O流。
  - IO多路复用的优势在于，当处理的消耗对比IO几乎可以忽略不计时，可以处理大量的并发IO，而不用消耗太多CPU/内存
  - IO多路复用 + 单进（线）程有个额外的好处，就不会有并发编程的各种坑问题，比如在nginx里，redis里，编程实现都会很简单很多。
  - Java世界里，因为JDBC这个东西是BIO的，所以在我们常见的Java服务里没有办法做到全部NIO化，必须得弄成多线程模型。如果要在做Java web服务这个大场景下享受IO多路复用的好处，要不就是不需要DB的，要不就是得用Vert.X一类的纯NIO框架把DB IO访问也得框进来。
- 进\线程并不是在socket上阻塞  而是在select/epoll上阻塞   socket是否阻塞是交给内核来判断然后给进程发送信号
- Epoll代理
  - 没有最大并发连接的限制，能打开的fd上限远大于1024（1G的内存能监听约10万个端口）
  - 采用回调的方式，效率提升。只有活跃可用的fd才会调用callback函数，也就是说 epoll 只管你“活跃”的连接，而跟连接总数无关，因此在实际的网络环境中，epoll的效率就会远远高于select和poll。只轮询发出了事件的流，哪个流发生了怎样的IO事件会通知处理线程，因此对这些流的操作都是有意义的，复杂度降低到了O(1)
  - 内存拷贝。使用mmap()文件映射内存来加速与内核空间的消息传递，减少复制开销
- CPU本来就是线性的不论什么都需要顺序处理并行只能是多核CPU
- io多路复用本来就是用来解决对多个I/O监听时,一个I/O阻塞影响其他I/O的问题,跟多线程没关系.
- 跟多线程相比较,线程切换需要切换到内核进行线程切换,需要消耗时间和资源.而I/O多路复用不需要切换线/进程,效率相对较高,特别是对高并发的应用nginx就是用I/O多路复用,故而性能极佳.但多线程编程逻辑和处理上比I/O多路复用简单.而I/O多路复用处理起来较为复杂







## 线程模型

- Redis基于Reactor模式开发了网络事件处理器，这个处理器被称为文件事件处理器。它的组成结构为4部分：多个套接字、IO多路复用、文件事件分派器、事件处理器。因为文件事件分派器队列的消费是单线程的，所以Redis才叫单线程模型。
- 消息处理流程
  - I/O多路复用(multiplexing)程序来同时监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
  - 当被监听的套接字准备好执行连接应答(accept)、读取(read)、写入(write)、关闭(close)等操作时，与操作相对应的文件事件就会产生，这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。
  - 尽管多个文件事件可能会并发地出现，但I/O多路复用程序总是会将所有产生事件的套接字都推到一个队列里面
  - 当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕）， I/O多路复用程序才会继续向文件事件分派器传送下一个套接字
- 文件事件的处理器
  - 连接应答处理器
  - 命令请求处理器
  - 命令回复处理器







## rehash

- 在Redis中，键值对（Key-Value Pair）存储方式是由字典（Dict）保存的，而字典底层是通过哈希表来实现的。通过哈希表中的节点保存字典中的键值对。类似Java中的HashMap，将Key通过哈希函数映射到哈希表节点位置。

- rehash的步骤

  - 为字典的ht[1]哈希表分配空间

    - > 该哈希表已有节点的数量
      > unsigned long used;
      >
      > 
      >
      > 扩展
      > 	ht[1]的大小为>=ht[0].used*2>=2^n
      > 收缩
      > 	ht[1]的大小为>=ht[0].used>=2^n

    - 将保存在ht[0]中的所有键值对rehash到ht[1]中，rehash指重新计算键的哈希值和索引值，然后将键值对放置到ht[1]哈希表的指定位置上

    - 释放ht[0]，将ht[1]设置为ht[0],新建空白的哈希表ht[1]，以备下次rehash使用

- 渐进式 rehash 执行期间的哈希表操作

  - 因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。
  - 另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。
  - 渐进式rehash避免了redis阻塞，可以说非常完美，但是由于在rehash时，需要分配一个新的hash表，在rehash期间，同时有两个hash表在使用，会使得redis内存使用量瞬间突增，在Redis 满容状态下由于Rehash会导致大量Key驱逐。
  - 除了导致满容驱逐淘汰，Redis Rehash还会引起其他一些问题：
    - 在tablesize级别与现有Keys数量不在同一个区间内，主从切换后，由于Redis全量同步，从库tablesize降为与现有Key匹配值，导致内存倾斜；
    - Redis Cluster下的某个分片由于Key数量相对较多提前Resize，导致集群分片内存不均

- Redis使用Scan清理Key由于Rehash导致清理数据不彻底

  - 为了高效地匹配出数据库中所有符合给定模式的Key，Redis提供了Scan命令。

  - Redis官方定义Scan特点如下：

    - 整个遍历从开始到结束期间， 一直存在于Redis数据集内的且符合匹配模式的所有Key都会被返回；
    - 如果发生了rehash，同一个元素可能会被返回多次，遍历过程中新增或者删除的Key可能会被返回，也可能不会。

  - 那么在Dict非稳定状态，即发生Rehash的情况下，Scan要如何保证原有的Key都能遍历出来，又尽少可能重复扫描呢？Redis Scan通过Hash桶掩码的高位顺序访问来解决。

    - Scan采用高位序访问的原因，就是为了实现Redis Dict在Rehash时尽可能少重复扫描返回Key。

      - > 000-100-010-110-001-101-011-111

    - 举个例子，如果Dict的tablesize从8扩展到了16，梳理一下Scan扫描方式:

      - > Dict(8) 从Cursor 0开始扫描；
        >
        > 准备扫描Cursor 6时发生Resize，扩展为之前的2倍，并完成Rehash；
        >
        > 客户端这时开始从Dict(16)的Cursor 6继续迭代；
        >
        > 这时按照 6→14→1→9→5→13→3→11→7→15 Scan完成。





## 跳表

- 它不要求上下相邻两层链表之间的节点个数有严格的对应关系，而是为每个节点随机出一个层数(level)

  - > 如果一个节点有第i层(i>=1)指针，么它有第(i+1)层指针的概率为p
    > 节点最大的层数不允许超过一个最大值，记为MaxLevel
    >
    > 
    >
    > level := 1    // random()返回一个[0...1)的随机数    
    > while random() < p and level < MaxLevel 
    > do level := level + 1 return level 
    >
    > 
    >
    > 在Redis的skiplist实现中，这两个参数的取值为：
    > 	p = 1/4
    > MaxLevel = 32

- skiplist与平衡树、哈希表的比较

  - skiplist和各种平衡树（如AVL、红黑树等）的元素是有序排列的，而哈希表不是有序的，哈希表上只能做单个key的查找
  - 在做范围查找的时候，平衡树比skiplist操作要复杂
    - 平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点
    - skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。
  - 平衡树的插入和删除操作可能引发子树的调整，逻辑复杂
  - 从内存占用上来说，skiplist比平衡树更灵活一些
    - 平衡树每个节点包含2个指针（分别指向左右子树）
    - Redis里的实现一样，取p=1/4，那么平均每个节点包含1.33个指针，比平衡树更有优势
  - 查找单个key，skiplist和平衡树的时间复杂度都为O(log n)。哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。所以我们平常使用的各种Map或dictionary结构，大都是基于哈希表实现的
  - 从算法实现难度上来比较，skiplist比平衡树要简单得多

- Redis中的skiplist实现

  - sorted set底层不仅仅使用了skiplist，还使用了ziplist和dict

  - skiplist的数据结构定义

    - zskiplistNode定义了skiplist的节点结构

      - > backward字段是指向链表前一个节点的指针（前向指针）。节点只有1个前向指针，所以只有第1层链表是一个双向链表。
        >
        > level[]存放指向各层链表后一个节点的指针（后向指针）
        >
        > 每层对应1个后向指针，用forward字段表示。另外，每个后向指针还对应了一个span值，它表示当前的指针跨越了多少个节点。span用于计算元素排名(rank)

    - zskiplist定义了真正的skiplist结构

      - > 头指针header和尾指针tail
        > 链表长度length
        > level表示skiplist的总层数，即所有节点层数的最大值

  - Redis中skiplist实现的特殊性
    - 当数据较少时，sorted set是  由一个ziplist来实现的
    - 当数据多的时候，sorted set是由一个dict + 一个skiplist来实现的
      - dict用来查询数据到分数的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）
    - 在如下两个条件之一满足的时候，ziplist会转成zset
      - 元素个数，即(数据, score)对的数目超过128的时候，也就是ziplist数据项超过256的时候
      - 当sorted set中插入的任意一个数据的长度超过了64的时候

- Redis中的skiplist跟前面介绍的经典的skiplist相比，有如下不同

  - 分数(score)允许重复，即skiplist的key允许重复。这在最开始介绍的经典skiplist中是不允许的
  - 在比较时，不仅比较分数（相当于skiplist的key），还比较数据本身。在Redis的skiplist实现中，数据本身的内容唯一标识这份数据，而不是由key来唯一标识
  - 第1层链表不是一个单向链表，而是一个双向链表。这是为了方便以倒序方式获取一个范围内的元素

  

### 主从复制

&emsp; 为了分担Master的读压力，Redis支持主从架构。一个Master可以连接多个Slave，一个Slave也可以连接多个Slave。主从复制主要分为：全量同步和增量同步。

* 全量同步
  1. Slave连接Master，发送SYNC命令；
  2. Master接收到SYNC，开始执行BGSAVE生成RDB快照文件，并使用缓冲区记录此后执行的所有写命令；
  3. Master执行BGSAVE完成后，向从服务器发送RDB快照文件，并在发送期间继续使用缓冲区记录执行的写命令；
  4. Slave接收到RDB快照文件后，丢弃旧数据，载入收到的快照数据；
  5. Master发送RDB文件完成后开始向Slave发送缓冲区的写命令；
  6. 从服务器完成对RDB快照文件的载入，开始接受命令请求，并执行来自Master缓冲区的写命令；
  7. 缓冲区写命令执行完成，全量同步结束。从服务器可以接收来自用户的请求。
* 部分同步
  从Redis2.8开始支持部分同步。部分同步使用PSYNC命令。Master为复制流维护一个内存缓冲区（in-memory backlog）。主从服务器都维护一个复制偏移量（offset）和master run id。Slave断开重连后，会请求继续复制，加入主从服务器的两个master run id相同，并且指定的偏移量在内存缓冲区中还有效，复制就会从上次中断的地方开始继续。
  ***在Master内部为每个Slave维护了一份同步日志和同步标识，每个Slave跟Master进行同步时都会携带自己的同步标识和上次同步的最后位置。当主从链接断掉后，Slave每隔1s主动尝试和Master进行链接，如果从服务器携带的偏移量标识还在Master的同步备份日志中，那么久从Slave发送的偏移量开始继续上次的同步操作。如果不在则需要进行全量同步。部分同步中，Master会将本地保存的同步备份日志中记录的指令依次发送给Slave从而达到数据一致性。***
* 增量同步
  增量同步的过程主要是Master每执行一个写命令就会向Slave发送相同的写命令，Slave接收到命令并执行命令。
* 无盘复制
  如果Master的磁盘有限，无法保存RDB文件，此时可以采用无盘复制来解决。Master直接开启一个Socket将rdb文件发送给Slave。无盘复制一般应用在磁盘有限但是网络良好的情况下。

**注意事项**

1. 如果多个slave断线重启。因为只要Slave启动就会发送Sync请求和Master全量同步，当有多个Slave的时候可能造成Master的IO剧增而宕机。
2. Master执行BGSAVE是通过fork子进程的方式异步执行，全量同步不会阻塞Master。也不会阻塞Slave。
3. 从Redis2.8开始Slave会周期性的应答从复制流中处理的数据量。以实现部分重新同步策略。
4. 主从架构可以解决Master的IO问题，在Master上数据不做持久化（关闭AOF、RDB），在从服务器上开启持久化。（需要避免Master重启，一旦Master重启，Master上的数据丢失，此时Slave会从master同步数据，则Slave上的数据也会丢失）
5. Slave不处理过期的键，由Master维护过期键的处理，并将相关删除命令发送给Slave。
6. 多个Slave同时发送SYNC命令，Master只生成一个RDB文件。

### 数据持久化

* RDB
  通过快照完成持久化，RDB是redis默认的持久化方式，在配置文件中设置了三个条件

  > > > save 900 1      #900秒内有至少1个键被更改则进行快照
  > > > save 300 10     #300秒内有至少10个键被更改则进行快照
  > > > save 60 10000   #60秒内有至少10000个键被更改则进行快照
  > > > 当满足上面三个条件中的任意一个则会生成快照文件。快照的实现过程如下所示：
  > > >
  > > > 1. Redis使用fork函数复制一份当前进程的副本（子进程），父子进程共享同一内存数据，如果父进程需要修改内存中的数据，操作系统会将该片数据复制一份以保证子进程数据不受影响，所以rdb文件存储的是执行fork时刻的内存数据；
  > > > 2. 父进程继续接受并处理客户端发送来的命令，子进程开始讲内存的数据写入硬盘中的临时文件；
  > > > 3. 子进程写完所有的数据后会用该临时文件替换原来的rdb文件，快照操作完成；

* AOF
  默认情况下Redis没有开启aof方式的持久化，可以通过参数appendonly yes来开启。开启AOF后redis重启后装载数据会比较慢，每执行一条更新redis中的数据命令，redis就将命令写入aof文件中。AOF文件会不断增加，可以通过配置对AOF文件进行重写。AOF开启后会降低Redis的性能，通过设置AOF的同步策略来提高效率。例如：appendfsync always \appendfsync everysec  \appendfsync no  等方式。

### 管道模式

&emsp; Redis2.6版本提供了管道功能，客户端可以向服务器发送多个命令，而不必等待答复，直到最后一个步骤中读取答复。

### Redis与DB数据不一致

* 写流程
  1) 淘汰cache； 2)写db
* 读流程
  1) 读cache，命中则返回； 2) cache未命中，读db；3) 将db中的数据写入cache；

&emsp; 当两个线程操作同一个数据，写线程A先将cache淘汰，然后写db，写db之前进行了GC导致暂停了一会；读线程B读cache未命中，查找db，此时找到db中旧数据，然后写cache；此时写线程A执行更新db操作。线程AB操作完成后，db和cache中的值就不一致了。解决方案如下：

1. 分布式锁，对同一个数据的操作必须拥有该数据的锁；
2. 保证相同数据的请求在同一个服务中，对同一个数据的读写操作在同一个数据库链接中；

### 分布式锁

* SETNX key value + EXPIRE key value
* SET key value [EX seconds] [PX milliseconds] [NX|XX]（redis2.6版本开始支持）

### 数据丢失问题

&emsp; 数据丢失主要有两种情况：

* 异步复制过程中。master更新数据后还没有复制到slave，master宕机了；
* 脑裂，master所在机器脱离了集群，跟其他slave不能链接，但是master还运行着；此时哨兵就认为master宕机了，开启选举，其他slave切换成了master，这个时候集群里有两个master；

&emsp; 问题解决：

* min-slaves-to-write 1
* min-slaves-max-lag 10

&emsp; 参数含义：要求至少有1个slave，数据复制和同步的延迟不能超过10秒。
&emsp; 有了min-slaves-max-lag这个配置，就可以确保说，一旦slave复制数据和ack延时太长，就认为可能master宕机后损失的数据太多了，那么就拒绝写请求，这样可以把master宕机时由于部分数据未同步到slave导致的数据丢失降低的可控范围内;
&emsp; 如果一个master出现了脑裂，跟其他slave丢了连接，那么上面两个配置可以确保说，如果不能继续给指定数量的slave发送数据，而且slave超过10秒没有给自己ack消息，那么就直接拒绝客户端的写请求;

### Redis Cluster

Redis 3.0支持Redis Cluster架构。自动将数据进行分片，每个master上放一部分数据；提供内置高可用，当部分master不可用时，可以继续工作。采用Cluster架构需要开放两个端口6379和16379,16379负责集群通信。

可以采用一致性hash（自动缓存迁移） + 虚拟节点（自动负载均衡）的策略。

RedisCluster有固定的16384个hash slot，对每个key计算出CRC16值，然后对16384取模，可以获取对应key的hash slot。ResisCluster中的每个master都存储部分slot（可以用户指定，也可以集群初始化的时候自动生成redis-trib.rd脚本），增加一个master就将别的master上的hash slot移动新的master上，减少一个master，就将他的hash slots移动到其它master上去。

Master维护一个16384/8字节的位序列，master节点用bit来标识自己是否拥有某个槽，例如对于编号为1的slot，master只需要判断序列的第二位是不是1即可。集群同时还维护着槽到集群节点的映射，是由长度为16384、类型为节点的数组实现的，这样就可以很快通过slot编号找到负责这个槽的节点。

客户端可以请求集群中的任意节点，节点找到对应key的slot，如果该节点负责key对应的slot，则直接返回结果；否则返回MOVED {HOST} {PORT}，客户端接受返回后再次请求对应的节点即可获取数据。客户端可以缓存链接，避免两次查询。

* slots迁移
  迁移的slot状态设置为MIGRATING状态，当客户请求的key对应的slot为MIGRATING状态时，如果key存在则成功处理；如果key不存在则返回客户端ASK，将请求转向另外一个节点，并不刷新客户端的映射关系，下次客户端请求该key的时候还会选择该master节点；如果key包含多个命令，如果都存在则成功处理；如果都不存在则返回ASK到另一节点处理；如果部分存在，则通知客户端稍后重试。

  slot在新节点的状态为IMPORTING状态；当客户请求的key对应的slot为IMPORTING状态，如果是正常命令则返回MOVE重定向，如果是ASKING命令则会被执行，从而key没有在老节点的情况可以被顺利处理，如果key不存在则新建。

参考文章：<https://www.cnblogs.com/wxd0108/p/5729754.html>


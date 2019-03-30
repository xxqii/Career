### 优点
1. 负载在brokers之间的自动分布；
2. broker借助zero-copy实现零拷贝发送到消费者；
3. 当有消费者加入或者离开consumer group的时候自动负载均衡；
4. kafka streams api将状态存储自动备份到集群；
5. broker故障的时候partition主动重新选举；

### 优化吞吐量

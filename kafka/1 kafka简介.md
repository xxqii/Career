### 线程模型
&emsp; Kafka采用一个Acceptor线程处理客户端的链接请求，然后轮询processors列表，将网络io事件交由processor处理。每个Acceptor和Processor都有自己的selector。  
&emsp; processor中的read方法读取请求内容，write将返回内容发送给客户端。如果read读取完毕，则构造Request对象，放入RequestChannel里等待处理。  
&emsp; KafkaRequestHandlerPool负责处理RequestChannel的requestQueue里面的请求。线程池的数量通过num.io.threads设置(默认8个线程)。获取请求后调用ApiUtils处理请求。
&emsp; KafkaApis会判断用户的请求类型RequestKeys，然后对应的处理函数处理请求，并得到response，通过调用RequestChannel的sendRespoonse将respoonse放入RequestChannel的responseQueue中。processor获取response并将结果返回客户端。  

* processors：默认为3，处理socket的读写事件  
* RequestChannel：包括两个队列requestQueue(ArrayBlockingQueue实现)和responseQueues(数组+BlockingQueue实现)。  
* requestQueue：默认大小500,内部实现为ArrayBlockingQueue缓存请求  
* responseQueues：数组+BlockingQueue实现，数组的长度和processor的数量相同，每个processors只处理自己的response  
### 相关代码  
1. SocketServer创建  
>
	class SocketServer(val brokerId: Int,
                   val host: String,
                   val port: Int,
                   val numProcessorThreads: Int,
                   val maxQueuedRequests: Int,
                   val sendBufferSize: Int,
                   val recvBufferSize: Int,
                   val maxRequestSize: Int = Int.MaxValue,
                   val maxConnectionsPerIp: Int = Int.MaxValue,
                   val connectionsMaxIdleMs: Long,
                   val maxConnectionsPerIpOverrides: Map[String, Int] ) extends Logging with KafkaMetricsGroup {
	  this.logIdent = "[Socket Server on Broker " + brokerId + "], "
	  private val time = SystemTime
	  //处理网络请求的线程
	  private val processors = new Array[Processor](numProcessorThreads)
	  //处理accept请求的线程
	  @volatile private var acceptor: Acceptor = null
	  //请求队列。Acceptor将请求放入requestChannel，processors从requestChannel中获取请求并处理。
	  val requestChannel = new RequestChannel(numProcessorThreads, maxQueuedRequests)

	  /* a meter to track the average free capacity of the network processors */
	  private val aggregateIdleMeter = newMeter("NetworkProcessorAvgIdlePercent", "percent", TimeUnit.NANOSECONDS)


	  def startup() {
		val quotas = new ConnectionQuotas(maxConnectionsPerIp, maxConnectionsPerIpOverrides)
		//创建processor，默认3个线程。
		for(i <- 0 until numProcessorThreads) {
		  processors(i) = new Processor(i, 
										time, 
										maxRequestSize, 
										aggregateIdleMeter,
										newMeter("IdlePercent", "percent", TimeUnit.NANOSECONDS, Map("networkProcessor" -> i.toString)),
										numProcessorThreads, 
										requestChannel,
										quotas,
										connectionsMaxIdleMs)
		  Utils.newThread("kafka-network-thread-%d-%d".format(port, i), processors(i), false).start()
		}

		newGauge("ResponsesBeingSent", new Gauge[Int] {
		  def value = processors.foldLeft(0) { (total, p) => total + p.countInterestOps(SelectionKey.OP_WRITE) }
		})

		// register the processor threads for notification of responses
		requestChannel.addResponseListener((id:Int) => processors(id).wakeup())
	   
		// start accepting connections
		//创建acceptor，并启动
		this.acceptor = new Acceptor(host, port, processors, sendBufferSize, recvBufferSize, quotas)
		Utils.newThread("kafka-socket-acceptor", acceptor, false).start()
		acceptor.awaitStartup
		info("Started")
	  }

	  def shutdown() = {
		info("Shutting down")
		if(acceptor != null)
		  acceptor.shutdown()
		for(processor <- processors)
		  processor.shutdown()
		info("Shutdown completed")
	  }
	}

2. Acceptor启动  
>
	def run() {	
		//注册selector
		serverChannel.register(selector, SelectionKey.OP_ACCEPT);
		startupComplete()
		var currentProcessor = 0
		while(isRunning) {
		  val ready = selector.select(500)
		  if(ready > 0) {
			val keys = selector.selectedKeys()
			val iter = keys.iterator()
			while(iter.hasNext && isRunning) {
			  var key: SelectionKey = null
			  try {
				key = iter.next
				iter.remove()
				//处理accept请求
				if(key.isAcceptable)
				   accept(key, processors(currentProcessor))
				else
				   throw new IllegalStateException("Unrecognized key state for acceptor thread.")

				// round robin to the next processor thread
				//记录当前处理的processor，采用轮询的方式负载均衡。
				currentProcessor = (currentProcessor + 1) % processors.length
			  } catch {
				case e: Throwable => error("Error while accepting connection", e)
			  }
			}
		  }
		}
		debug("Closing server socket and selector.")
		swallowError(serverChannel.close())
		swallowError(selector.close())
		shutdownComplete()
	  }
3. Acceptor的accept方法  
>
	def accept(key: SelectionKey, processor: Processor) {
		val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
		val socketChannel = serverSocketChannel.accept()
		try {
		  connectionQuotas.inc(socketChannel.socket().getInetAddress)
		  socketChannel.configureBlocking(false)
		  socketChannel.socket().setTcpNoDelay(true)
		  socketChannel.socket().setSendBufferSize(sendBufferSize)

		  debug("Accepted connection from %s on %s. sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
				.format(socketChannel.socket.getInetAddress, socketChannel.socket.getLocalSocketAddress,
					  socketChannel.socket.getSendBufferSize, sendBufferSize,
					  socketChannel.socket.getReceiveBufferSize, recvBufferSize))

		  processor.accept(socketChannel)
		} catch {
		  case e: TooManyConnectionsException =>
			info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
			close(socketChannel)
		}
	  }

4. Processor的run方法  
>
	override def run() {
		startupComplete()
		while(isRunning) {
		  // setup any new connections that have been queued up
		  configureNewConnections()
		  // register any new responses for writing
		  processNewResponses()
		  val startSelectTime = SystemTime.nanoseconds
		  val ready = selector.select(300)
		  currentTimeNanos = SystemTime.nanoseconds
		  val idleTime = currentTimeNanos - startSelectTime
		  idleMeter.mark(idleTime)
		  // We use a single meter for aggregate idle percentage for the thread pool.
		  // Since meter is calculated as total_recorded_value / time_window and
		  // time_window is independent of the number of threads, each recorded idle
		  // time should be discounted by # threads.
		  aggregateIdleMeter.mark(idleTime / totalProcessorThreads)

		  trace("Processor id " + id + " selection time = " + idleTime + " ns")
		  if(ready > 0) {
			val keys = selector.selectedKeys()
			val iter = keys.iterator()
			while(iter.hasNext && isRunning) {
			  var key: SelectionKey = null
			  try {
				key = iter.next
				iter.remove()
				if(key.isReadable)
				  read(key)
				else if(key.isWritable)
				  write(key)
				else if(!key.isValid)
				  close(key)
				else
				  throw new IllegalStateException("Unrecognized key state for processor thread.")
			  } catch {
				case e: EOFException => {
				  info("Closing socket connection to %s.".format(channelFor(key).socket.getInetAddress))
				  close(key)
				} case e: InvalidRequestException => {
				  info("Closing socket connection to %s due to invalid request: %s".format(channelFor(key).socket.getInetAddress, e.getMessage))
				  close(key)
				} case e: Throwable => {
				  error("Closing socket for " + channelFor(key).socket.getInetAddress + " because of error", e)
				  close(key)
				}
			  }
			}
		  }
		  maybeCloseOldestConnection
		}
		debug("Closing selector.")
		closeAll()
		swallowError(selector.close())
		shutdownComplete()
	  }
  
5. Processor处理网络IO请求  
>
	def read(key: SelectionKey) {
		lruConnections.put(key, currentTimeNanos)
		val socketChannel = channelFor(key)
		var receive = key.attachment.asInstanceOf[Receive]
		if(key.attachment == null) {
		  receive = new BoundedByteBufferReceive(maxRequestSize)
		  key.attach(receive)
		}
		val read = receive.readFrom(socketChannel)
		val address = socketChannel.socket.getRemoteSocketAddress();
		trace(read + " bytes read from " + address)
		if(read < 0) {
		  close(key)
		} else if(receive.complete) {
		  val req = RequestChannel.Request(processor = id, requestKey = key, buffer = receive.buffer, startTimeMs = time.milliseconds, remoteAddress = address)
		  requestChannel.sendRequest(req)
		  key.attach(null)
		  // explicitly reset interest ops to not READ, no need to wake up the selector just yet
		  key.interestOps(key.interestOps & (~SelectionKey.OP_READ))
		} else {
		  // more reading to be done
		  trace("Did not finish reading, registering for read again on connection " + socketChannel.socket.getRemoteSocketAddress())
		  key.interestOps(SelectionKey.OP_READ)
		  wakeup()
		}
	  }
6. RequestChannel入队列  
>
	def sendRequest(request: RequestChannel.Request) {
		requestQueue.put(request)
	  }

7. KafkaRequestHandler  
>
	class KafkaRequestHandlerPool(val brokerId: Int,
                              val requestChannel: RequestChannel,
                              val apis: KafkaApis,
                              numThreads: Int) extends Logging with KafkaMetricsGroup {

	  /* a meter to track the average free capacity of the request handlers */
	  private val aggregateIdleMeter = newMeter("RequestHandlerAvgIdlePercent", "percent", TimeUnit.NANOSECONDS)

	  this.logIdent = "[Kafka Request Handler on Broker " + brokerId + "], "
	  val threads = new Array[Thread](numThreads)
	  val runnables = new Array[KafkaRequestHandler](numThreads)
	  for(i <- 0 until numThreads) {
		runnables(i) = new KafkaRequestHandler(i, brokerId, aggregateIdleMeter, numThreads, requestChannel, apis)
		threads(i) = Utils.daemonThread("kafka-request-handler-" + i, runnables(i))
		threads(i).start()
	  }
	  
8. KafkaRequestHandler  
>
	class KafkaRequestHandler(id: Int,
                          brokerId: Int,
                          val aggregateIdleMeter: Meter,
                          val totalHandlerThreads: Int,
                          val requestChannel: RequestChannel,
                          apis: KafkaApis) extends Runnable with Logging {
	  this.logIdent = "[Kafka Request Handler " + id + " on Broker " + brokerId + "], "

	  def run() {
		while(true) {
		  try {
			var req : RequestChannel.Request = null
			while (req == null) {
			  // We use a single meter for aggregate idle percentage for the thread pool.
			  // Since meter is calculated as total_recorded_value / time_window and
			  // time_window is independent of the number of threads, each recorded idle
			  // time should be discounted by # threads.
			  val startSelectTime = SystemTime.nanoseconds
			  req = requestChannel.receiveRequest(300)
			  val idleTime = SystemTime.nanoseconds - startSelectTime
			  aggregateIdleMeter.mark(idleTime / totalHandlerThreads)
			}

			if(req eq RequestChannel.AllDone) {
			  debug("Kafka request handler %d on broker %d received shut down command".format(
				id, brokerId))
			  return
			}
			req.requestDequeueTimeMs = SystemTime.milliseconds
			trace("Kafka request handler %d on broker %d handling request %s".format(id, brokerId, req))
			apis.handle(req)
		  } catch {
			case e: Throwable => error("Exception when handling request", e)
		  }
		}
	  }

	  def shutdown(): Unit = requestChannel.sendRequest(RequestChannel.AllDone)
	}

9. KafkaApis处理用户请求  
>
	def handle(request: RequestChannel.Request) {
		try{
		  trace("Handling request: " + request.requestObj + " from client: " + request.remoteAddress)
		  request.requestId match {
			case RequestKeys.ProduceKey => handleProducerOrOffsetCommitRequest(request)
			case RequestKeys.FetchKey => handleFetchRequest(request)
			case RequestKeys.OffsetsKey => handleOffsetRequest(request)
			case RequestKeys.MetadataKey => handleTopicMetadataRequest(request)
			case RequestKeys.LeaderAndIsrKey => handleLeaderAndIsrRequest(request)
			case RequestKeys.StopReplicaKey => handleStopReplicaRequest(request)
			case RequestKeys.UpdateMetadataKey => handleUpdateMetadataRequest(request)
			case RequestKeys.ControlledShutdownKey => handleControlledShutdownRequest(request)
			case RequestKeys.OffsetCommitKey => handleOffsetCommitRequest(request)
			case RequestKeys.OffsetFetchKey => handleOffsetFetchRequest(request)
			case RequestKeys.ConsumerMetadataKey => handleConsumerMetadataRequest(request)
			case requestId => throw new KafkaException("Unknown api code " + requestId)
		  }
		} catch {
		  case e: Throwable =>
			request.requestObj.handleError(e, requestChannel, request)
			error("error when handling request %s".format(request.requestObj), e)
		} finally
		  request.apiLocalCompleteTimeMs = SystemTime.milliseconds
	  }
### Kafka partition数据倾斜
&emsp; Kafka0.8版本，客户端发送消息的时候如果在消息中没有指定key，则在DefaultEventHandler中选择消息发送的partition的时候会从sendPartitionPerTopicCache中获取缓存的partiton，缓存在partition丢失、leader不存在或者topic.metadata.refresh.interval.ms
（默认10分钟）后失效，则10min内所有的消息都会发送到同一个分区，造成kafka中partition的数据倾斜。官方解释是：减少服务器端的socket连接数。代码如下所示：

	private def getPartition(topic: String, key: Any, topicPartitionList: Seq[PartitionAndLeader]): Int = {
    val numPartitions = topicPartitionList.size
    if(numPartitions <= 0)
      throw new UnknownTopicOrPartitionException("Topic " + topic + " doesn't exist")
    val partition =
      if(key == null) {
        // If the key is null, we don't really need a partitioner
        // So we look up in the send partition cache for the topic to decide the target partition
        val id = sendPartitionPerTopicCache.get(topic)
        id match {
          case Some(partitionId) =>
            // directly return the partitionId without checking availability of the leader,
            // since we want to postpone the failure until the send operation anyways
            partitionId
          case None =>
            val availablePartitions = topicPartitionList.filter(_.leaderBrokerIdOpt.isDefined)
            if (availablePartitions.isEmpty)
              throw new LeaderNotAvailableException("No leader for any partition in topic " + topic)
            val index = Utils.abs(Random.nextInt) % availablePartitions.size
            val partitionId = availablePartitions(index).partitionId
            sendPartitionPerTopicCache.put(topic, partitionId)
            partitionId
        }
      } else
        partitioner.partition(key, numPartitions)
    if(partition < 0 || partition >= numPartitions)
      throw new UnknownTopicOrPartitionException("Invalid partition id: " + partition + " for topic " + topic +
        "; Valid values are in the inclusive range of [0, " + (numPartitions-1) + "]")
    trace("Assigning message of topic %s and key %s to a selected partition %d".format(topic, if (key == null) "[none]" else key.toString, partition))
    partition
  }
 
&emsp;官方配置文件说明：  
* key: topic.metadata.refresh.interval.ms	
* value: 600 * 1000	
* desc: The producer generally refreshes the topic metadata from brokers when there is a failure (partition missing, leader not available...). It will also poll regularly (default: every 10min so 600000ms). If you set this to a negative value, metadata will only get refreshed on failure. If you set this to zero, the metadata will get refreshed after each message sent (not recommended). Important note: the refresh happen only AFTER the message is sent, so if the producer never sends a message the metadata is never refreshed

### 优点
1. 负载在brokers之间的自动分布；
2. broker借助zero-copy实现零拷贝发送到消费者；
3. 当有消费者加入或者离开consumer group的时候自动负载均衡；
4. kafka streams api将状态存储自动备份到集群；
5. broker故障的时候partition主动重新选举；

### 启动流程
&emsp; Kafka启动主要在server.KafkaServer的startup方法中实现，具体步骤如下所示：  
1. kafkaScheduler.startup()  
&emsp; 启动任务调度线程池，默认10个线程，主要负责后台任务的处理，比如：日志清理工作。  
2. initZk  
&emsp; 连接到zk服务器；创建通用节点。  
3. createLogManager  
&emsp; LogManager是kafka的子系统，负责log的创建，检索及清理。所有的读写操作由单个的日志实例来代理。  
4. socketServer.startup  
&emsp;SocketServer是nio的socket服务器，线程模型是：1个Acceptor线程处理新连接，Acceptor还有多个处理器线程，每个处理器线程拥有自己的selector和多个读socket请求Handler线程。handler线程处理请求并产生响应写给处理器线程。   
&emsp; num.network.threads:处理网络请求的线程数，默认3个；num.io.threads:处理IO的线程数，默认8个；  
5. replicaManager  
&emsp; 副本管理器   
6. offsetManager  
&emsp;创建offset管理器  
7. kafkaController  
&emsp;创建controller  
8. 创建apis和requestHandlerPool  
&emsp;主要用来处理用户的请求  
9. topicConfigManager  
&emsp; topic管理器  
10. kafkaHealthcheck  
&emsp; 心跳检查  
11. 

### 优化  
* request.required.acks  
* min.insync.replicas：当request.required.acks设置为-1时生效。  
&emsp; 要保证数据写入到Kafka是安全的，高可靠的，需要如下的配置：  
* topic的配置：replication.factor>=3,即副本数至少是3个；2<=min.insync.replicas<=replication.factor  
* broker的配置：leader的选举条件unclean.leader.election.enable=false  
* producer的配置：request.required.acks=-1(all)，producer.type=sync  
### 消息传输保证
At most once: 消息可能会丢，但绝不会重复传输   
At least once：消息绝不会丢，但可能会重复传输  
Exactly once：每条消息肯定会被传输一次且仅传输一次  

1、网络和io操作线程配置优化  
# broker处理消息的最大线程数（默认为3）  
num.network.threads=cpu核数+1  
# broker处理磁盘IO的线程数   
num.io.threads=cpu核数*2  
 
2、log数据文件刷盘策略   
# 每当producer写入10000条消息时，刷数据到磁盘  
log.flush.interval.messages=10000  
# 每间隔1秒钟时间，刷数据到磁盘  
log.flush.interval.ms=1000  

3、日志保留策略配置  
# 保留三天，也可以更短 （log.cleaner.delete.retention.ms）  
log.retention.hours=72  
# 段文件配置1GB，有利于快速回收磁盘空间，重启kafka加载也会加快(如果文件过小，则文件数量比较多，kafka启动时是单线程扫描目录(log.dir)下所有数据文件  
log.segment.bytes=1073741824  

4、Replica相关配置  
default.replication.factor:3  

### Que

* Kafka的用途有哪些？使用场景如何？
Kafka中的ISR、AR又代表什么？ISR的伸缩又指什么
Kafka中的HW、LEO、LSO、LW等分别代表什么？
Kafka中是怎么体现消息顺序性的？
Kafka中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？
Kafka生产者客户端的整体结构是什么样子的？
Kafka生产者客户端中使用了几个线程来处理？分别是什么？
Kafka的旧版Scala的消费者客户端的设计有什么缺陷？
“消费组中的消费者个数如果超过topic的分区，那么就会有消费者消费不到数据”这句话是否正确？如果不正确，那么有没有什么hack的手段？
消费者提交消费位移时提交的是当前消费到的最新消息的offset还是offset+1?
有哪些情形会造成重复消费？
那些情景下会造成消息漏消费？
KafkaConsumer是非线程安全的，那么怎么样实现多线程消费？
简述消费者与消费组之间的关系
当你使用kafka-topics.sh创建（删除）了一个topic之后，Kafka背后会执行什么逻辑？
topic的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？
topic的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？
创建topic时如何选择合适的分区数？
Kafka目前有那些内部topic，它们都有什么特征？各自的作用又是什么？
优先副本是什么？它有什么特殊的作用？
Kafka有哪几处地方有分区分配的概念？简述大致的过程及原理
简述Kafka的日志目录结构
Kafka中有那些索引文件？
如果我指定了一个offset，Kafka怎么查找到对应的消息？
如果我指定了一个timestamp，Kafka怎么查找到对应的消息？
聊一聊你对Kafka的Log Retention的理解
聊一聊你对Kafka的Log Compaction的理解
聊一聊你对Kafka底层存储的理解（页缓存、内核层、块层、设备层）
聊一聊Kafka的延时操作的原理
聊一聊Kafka控制器的作用
消费再均衡的原理是什么？（提示：消费者协调器和消费组协调器）
Kafka中的幂等是怎么实现的
Kafka中的事务是怎么实现的（这题我去面试6加被问4次，照着答案念也要念十几分钟，面试官简直凑不要脸）
Kafka中有那些地方需要选举？这些地方的选举策略又有哪些？
失效副本是指什么？有那些应对措施？
多副本下，各个副本中的HW和LEO的演变过程
为什么Kafka不支持读写分离？
Kafka在可靠性方面做了哪些改进？（HW, LeaderEpoch）
Kafka中怎么实现死信队列和重试队列？
Kafka中的延迟队列怎么实现（这题被问的比事务那题还要多！！！听说你会Kafka，那你说说延迟队列怎么实现？）
Kafka中怎么做消息审计？
Kafka中怎么做消息轨迹？
Kafka中有那些配置参数比较有意思？聊一聊你的看法
Kafka中有那些命名比较有意思？聊一聊你的看法
Kafka有哪些指标需要着重关注？
怎么计算Lag？(注意read_uncommitted和read_committed状态下的不同)
Kafka的那些设计让它有如此高的性能？
Kafka有什么优缺点？
还用过什么同质类的其它产品，与Kafka相比有什么优缺点？
为什么选择Kafka?
在使用Kafka的过程中遇到过什么困难？怎么解决的？
怎么样才能确保Kafka极大程度上的可靠性？
聊一聊你对Kafka生态的理解
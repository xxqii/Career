### MemoryChannel
&emsp; MemoryChannel内部采用LinkedBlockingDeque实现。
### 变量定义

	//队列锁
	private Object queueLock = new Object();
	//消息队列
	private LinkedBlockingDeque<Event> queue;
	//跟踪队列剩余空间
	private Semaphore queueRemaining;
	//预定队列中的空间
	private Semaphore queueStored;
	//记录channel容量、大小等统计数据
	private ChannelCounter channelCounter;

### ChannelCounter
	
	//channel中保存数据的大小
	private static final String COUNTER_CHANNEL_SIZE = "channel.current.size";
	//put操作的数量
	private static final String COUNTER_EVENT_PUT_ATTEMPT = "channel.event.put.attempt";
	//take操作的数量
	private static final String COUNTER_EVENT_TAKE_ATTEMPT = "channel.event.take.attempt";
	//put成功的数量
	private static final String COUNTER_EVENT_PUT_SUCCESS = "channel.event.put.success";
	//take成功的数量
	private static final String COUNTER_EVENT_TAKE_SUCCESS = "channel.event.take.success";
	//channel的容量大小
	private static final String COUNTER_CHANNEL_CAPACITY = "channel.capacity";
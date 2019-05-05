1. Kafka如何保证消息的可靠性？
   消费者丢了数据：当消费者先提交offset，然后处理消息的时候可能会发生提交offset，然后消费者宕机了，此时消息就会丢失。此种情况可以先处理消息，然后提交offset。这样会避免消息丢失，但是可能发生消息重复。可以通过消费者自己保证幂等性来避免消息重复的影响（redis、mysql主键、全局id去重）
   Kafka丢了数据：主要是partition的leader挂了，follower还没来得及同步数据，然后follower成为leader，此时会发生kafka丢数据的情况。通过配置一下参数可以避免kafka丢失数据。

   * 给topic设置replication.factor参数，值必须大于1，保证每个partition至少2个副本；
   * min.insync.replicas：值必须大于1，leader感知至少有一个follower和自己联系；
   * producer端设置ack=all：每条数据必须是写入所有的replica之后才认为是成功的；
   * producer端设置retries=MAX（无限重试），一旦消息失败就无限重试，程序卡在这里；

   生成者丢失数据：设置了ack=all和retries=MAX后生产者是不会丢失数据的。

2. 如何保证消息的顺序性？
   在RabbitMQ中可以拆分多个queue，每个queue一个消费者。
   在kafka中可以一个topic、一个partition、一个consumer，单线程串行化操作，保证有序性，但是吞吐量太低；写N个内存queue，相同的key写入同一个queue，然后N个线程分别处理对应的queue，可以保证顺序性。
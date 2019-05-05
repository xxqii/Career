1. RDD由哪些特性？
   弹性分布式数据集。代表一个不可变、可分区、里面的元素可以并行计算的集合。

2. RDD的宽窄依赖?
   窄依赖：子RDD的每个分区依赖常数个父RDD分区；
   宽依赖：子RDD的每个分区依赖所有的父RDD分区；

3. spark如何划分stage？
   spark根据RDD之间的依赖关系，形成一个有向无环图（DAG），DAG会提交给DAGScheduler，DAGSchduler会把DAG划分为互相依赖的多个stage，划分的依据时宽、窄依赖，如果遇到宽依赖就划分stage，每个stage包含多个task，然后将这些task以TaskSet的形式提交给TaskSchuduler运行，stage是由一组并行的task组成。

4. spark的pipeline计算模式？
   spark采用pipeline计算模式，也就是来一条数据计算一条数据，然后把所有的逻辑走完，完后落地。而MapReduce是一步一步执行的，每步执行完毕都需要落地。spark中间计算完全基于内存，所以比mapreduce快。

5. pipeline中的RDD何时落地?
   shuffle write和persist的时候会落地。

6. DAGScheduler的做用？
   接收用户提交了job；将job划分为不同的stage，并在每个stage内产生一系列的task，封装成taskSet；决定每个task的最佳位置，任务在数据所在节点运行，将TaskSet提交给TaskScheduler；重新提交shuffle数据丢失的task给TaskScheduler；

7. Job生成？
   一旦Driver程序中出现Action操作，就生成一个Job，Driver向DAGScheduler提交Job，DAGScheduler将Job从后向前分割为多个stage，每个stage包含多个task，DAGSchduler将taskset提交给TaskScheduler，TaskScheduler负责task的执行、监控以及失败充实等。DAGScheduler负责task执行的位置，以及提交shuffle过程中失败的task。

8. Driver的功能？
   一个Spark作业运行时包含一个driver进程，就是作业的主进程，具有main函数，并且由SparkContext实例，是程序的入口。负责向集群申请资源，向master注册信息，负责作业的解析、调度，生成stage并调度task到executor上，包括DAGScheduler和TaskScheduler。

9. DataFrame 和 RDD 最大的区别？
   左侧的RDD[Person]虽然以Person为类型参数，但Spark框架本身不了解Person类的内部结构。而右侧的DataFrame却提供了详细的结构信息，使得Spark SQL可以清楚地知道该数据集中包含哪些列，每列的名称和类型各是什么。DataFrame多了数据的结构信息，即schema。RDD是分布式的Java对象的集合。DataFrame是分布式的Row对象的集合。DataFrame除了提供了比RDD更丰富的算子以外，更重要的特点是提升执行效率、减少数据读取以及执行计划的优化，比如filter下推、裁剪等。
   RDD API是函数式的，强调不变性，在大部分场景下倾向于创建新对象而不是修改老对象。Spark SQL在框架内部已经在各种可能的情况下尽量重用对象，这样做虽然在内部会打破了不变性，但在将数据返回给用户时，还会重新转为不可变数据。

   SparkSql的查询优化器会优化DAG

10. spark和flink的区别?
    都支持实时计算和批处理。spark功能更强大；flink效率更高；
    flink支持增量迭代，在迭代运算上比spark稍好；
    spark流计算时通过将RDD进行小批量处理，延迟在100ms，flink是一行一行处理（类似storm），支持毫秒级别；
    Flink基于分布式快照与可部分重发的数据源实现了容错。用户可自定义对整个Job进行快照的时间间隔，当任务失败时，Flink会将整个Job恢复到最近一次快照，并从数据源重发快照之后的数据。

11. apache beam？
    统一数据批处理（batch）和流处理（streaming）编程范式；
    能在任何执行引擎上执行（例如：spark/flink）
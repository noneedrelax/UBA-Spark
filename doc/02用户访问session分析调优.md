## Spark性能调优

1. 性能调优相关的原理讲解、经验总结；
2. 掌握一整套Spark企业级性能调优解决方案；而不只是简单的一些性能调优技巧。
3. 针对写好的spark作业，实施一整套数据倾斜解决方案：实际经验中积累的数据倾斜现象的表现，以及处理后的效果总结。

调优前首先要对spark的作业流程清楚：
* Driver到Executor的结构；
```
Master: Driver
    |-- Worker: Executor
            |-- job
                 |-- stage
                       |-- Task Task 
```
* 一个Stage内，**最终的RDD有多少个partition，就会产生多少个task**，一个task处理一个partition的数据；
* 作业划分为task分到Executor上，然后一个cpu core执行一个task；
* BlockManager负责Executor，task的数据管理，task来它这里拿数据；



## 1.1 资源分配

性能调优的王道：分配更多资源。
* 分配哪些资源？ executor、cpu per executor、memory per executor、driver memory
* 在哪里分配这些资源？
在我们在生产环境中，提交spark作业时，用的spark-submit shell脚本，里面调整对应的参数
```
/usr/local/spark/bin/spark-submit \
--class cn.spark.sparktest.core.WordCountCluster \
--driver-memory 1000m \     #driver的内存，影响不大，只要不出driver oom
--num-executors 3 \         #executor的数量
--executor-memory 100m \    #每个executor的内存大小
--executor-cores 3 \        #每个executor的cpu core数量
/usr/local/SparkTest-0.0.1-SNAPSHOT-jar-with-dependencies.jar \
```

### 如何调节资源分配

* 第一种，Spark Standalone 模式下资源分配。

假如说一共20台机器，每台机器能使用4G内存，2个cpu core。
```
executor = 20，对应worker节点数量；
executor-memory = 4G内存，对应每个worker能用的最大内存；
executor-cores = 2，对应每个worker给每个executor能用的最多个cpu core。
```

* 第二种，Yarn模式资源队列下资源调度。

应该去查看spark作业，要提交到的资源队列，大概有多少资源？
一个原则，你能使用的资源有多大，就尽量去调节到最大的大小（executor的数量，几十个到上百个不等；


### 为什么调节资源以后，性能可以提升？

* num-executors：增加executor个数

    如果executor数量比较少，能够并行执行的task就少，Application的并行执行的能力就很弱。

* executor-cores：增加每个executor的cpu core，增加了执行的并行能力

    原本20个executor，各有2个cpu core。能够并行40个task。
    现在每个executor的cpu core，增加到了5个。就能够并行执行100个task。
    执行的速度，提升了2.5倍。
    **但不超过服务器的cpu core数，不然会waiting**。

* executor-memory：增加每个executor的内存量

    增加了内存量以后，对性能的提升有以下几点，**但是不超过分配各每个worker的内存**：
    1. 如果需要对RDD进行cache，可以缓存更多的数据，将更少的数据写入磁盘。
    2. 对于shuffle操作减少了磁盘IO，reduce端需要内存来存放拉取的数据并进行聚合。如果内存不够，会写入磁盘。
    3. 对于task的执行，会创建很多对象。如果内存比较小，可能会频繁导致JVM堆内存满了，然后频繁GC，垃圾回收，minor GC和full GC。速度变慢。


## 1.2 调节并行度

并行度：Spark作业中根据宽窄依赖拆成多个stage，各个stage的task数量，也就代表了job在各个阶段（stage）的并行度。
```
SparkConf conf = new SparkConf()
  .set("spark.default.parallelism", "500")
```

**spark自己的算法给task选择Executor，Execuotr进程里面包含着task线程**。
举例（wordCount）：
    map阶段：每个task根据去找到自己需要的数据写到文件去处理。生成的文件一定是存放相同的key对应的values，相同key的values一定是进入同一个文件。
    reduce阶段：每个stage1的task，会去上面生成的文件拉取数据；拉取到的数据，一定是相同key对应的数据。对相同的key，对应的values，才能去执行我们自定义的function操作（_ + _）

假设资源调到上限了，如果不调节并行度，导致并行度过低，会怎么样？

假设task设置了100个task。有50个executor，每个executor有3个cpu core。
则Application任何一个stage运行的时候，都有总数在150个cpu core，可以并行运行。
同时在运行的task只有100个。每个executor剩下的一个cpu core，并行度没有与资源相匹配，就浪费掉了。

合理的并行度的设置，应该要设置到可以完全合理的利用你的集群资源；
比如上面的例子，总共集群有150个cpu core，可以并行运行150个task。
即可以同时并行运行，还可以让每个task要处理的数据量变少。
最终，就是提升你的整个Spark作业的性能和运行速度。

**官方是推荐，task数量，设置成spark application总cpu core数量的2~3倍，比如150个cpu core，基本要设置task数量为300~500；**

因为有些task会运行的快一点，比如50s就完了，有些task，可能会慢一点，要1分半才运行完，如果task数量设置成cpu core总数的2~3倍，那么一个task运行完了以后，另一个task马上可以补上来，就尽量不让cpu core空闲，尽量提升spark作业运行的效率和速度，提升性能。


## 1.3 重构RDD架构以及RDD持久化

1. RDD架构重构与优化：尽量复用RDD，差不多的RDD抽取为一个共同的RDD，供后面的RDD计算时，反复使用。
2. 公共RDD一定要实现持久化，将RDD的数据缓存到内存中/磁盘中，（BlockManager），以后无论对这个RDD做多少次计算，那么都是直接取这个RDD的同一份数据。
3. 持久化是可以进行序列化的，如果正常将数据持久化在内存中，可能会导致内存的占用过大，导致OOM。
4. 为了数据的高可靠性，而且内存充足，可以使用双副本机制，进行持久化。


## 1.4 大变量进行广播，使用Kryo序列化，本地化等待时间

**广播变量**
session分析模块中随机抽取部分，time2sessionsRDD.flatMapToPair()，取session2extractlistMap中对应时间的list。
task执行的算子flatMapToPair算子，**使用了外部的变量session2extractlistMap**，每个task都要通过网络的传输获取一份变量的副本占网络资源占内存。

* 在driver上会有一份初始的副本。
* task在运行的时候，想要使用广播变量中的数据，此时首先会在自己本地的Executor对应的BlockManager中（负责管理某个Executor对应的内存和磁盘上的数据），尝试获取变量副本；
* 如果本地没有，那么就从Driver远程拉取变量副本，并保存在本地的BlockManager中；
* 此后这个executor上的task，都会直接使用本地的BlockManager中的副本。
* BlockManager除了从driver上拉取，也可能从其他节点的BlockManager上拉取变量副本，距离越近越好。

==============

**优化序列化格式**
默认情况下，Spark内部是使用Java的对象输入输出流序列化机制，ObjectOutputStream / ObjectInputStream
这种默认序列化机制的好处在于，处理起来比较方便；也不需要我们手动去做什么事情，只是，你在算子里面使用的变量，必须是实现Serializable接口的，可序列化即可。
但是默认的序列化机制的效率不高速度慢；序列化数据占用的内存空间大。

Spark支持使用Kryo序列化机制。
Kryo序列化机制，比默认的Java序列化机制，速度要快，序列化后的数据要更小，大概是Java序列化机制的1/10。让网络传输的数据变少；耗费的内存资源大大减少。

Kryo序列化机制，一旦启用以后，会生效的几个地方：
1、算子函数中使用到的外部变量
2、持久化RDD时进行序列化，StorageLevel.MEMORY_ONLY_SER
3、shuffle

第一步：
```
SparkConf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```
第二步，注册你使用到的，需要通过Kryo序列化的自定义类。
```
SparkConf.registerKryoClasses(new Class[]{CategorySortKey.class});
```
Kryo要求，如果要达到它的最佳性能的话，那么就一定要注册你自定义的类（比如，你的算子函数中使用到了外部自定义类型的对象变量，就要求必须注册你的类，否则Kryo达不到最佳性能）。

============

**fastutil优化**
由于java的集合类型在每个数据中除了数据，还有元素的位置长度等都要占用了空间，所以一般不推荐使用集合，而是使用java数组。

fastutil是扩展了Java标准集合框架（Map、List、Set；HashMap、ArrayList、HashSet）的类库，提供了特殊类型的map、set、list和queue；能够提供更小的内存占用，更快的存取速度。

Spark中应用fastutil的场景：
1、如果算子函数使用了外部变量是某种比较大的集合，那么可以考虑使用fastutil改写外部变量，首先从源头上就减少内存的占用，通过广播变量进一步减少内存占用，再通过Kryo序列化类库进一步减少内存占用。
2、在你的算子函数里，也就是task要执行的计算逻辑里面，要创建比较大的Map、List等集合，可以考虑将这些集合类型使用fastutil类库重写，减少task创建出来的集合类型的内存占用。

fastutil的使用，在pom.xml中引用fastutil的包
```List<Integer> => IntList

```
基本都是类似于IntList的格式，前缀就是集合的元素类型；特殊的就是Map，Int2IntMap，代表了key-value映射的元素类型。


==============

**调节数据本地化等待时长**

背景：所谓任务跟着数据跑，Driver对Application的每一个stage的task进行分配之前，都会计算出每个task要计算的是哪个分片数据。
分配算法会希望每个task正好分配到它要计算的数据所在的节点。
但是可能某task要分配过去的那个节点的计算资源和计算能力都满了，Spark会等待一段时间，默认情况下是3s。
超过时间了会选择一个比较差的本地化级别，将task分配到离要计算的数据所在节点比较近的一个节点，然后进行计算。

task会通过其所在节点的BlockManager来获取数据，BlockManager发现自己本地没有数据，
会通过一个getRemote()方法，通过TransferService（网络数据传输组件）从数据所在节点的BlockManager中，获取数据，通过网络传输回task所在节点。

所以我们可以调节等待时长就是让spark再等待多一下，不要到低一级的本地化级别。
```
spark.locality.wait.process
spark.locality.wait.node
spark.locality.wait.rack

new SparkConf()
  .set("spark.locality.wait", "10")
```

先用client模式，在本地就直接可以看到比较全的日志。
观察日志，spark作业的运行日志显示，观察大部分task的数据本地化级别。
如果是发现，好多的级别都是NODE_LOCAL、ANY，那么最好就去调节一下数据本地化的等待时长。
看看大部分的task的本地化级别有没有提升，spark作业的运行时间有没有缩短。
如果spark作业的运行时间反而增加了，那就还是不要调节了。

**本地化级别**
* PROCESS_LOCAL：进程本地化，代码和数据在同一个进程中，也就是在同一个executor中；计算数据的task由executor执行，数据在executor的BlockManager中；性能最好
* NODE_LOCAL：节点本地化，代码和数据在同一个节点中；比如说，数据作为一个HDFS block块，就在节点上，而task在节点上某个executor中运行；或者是，数据和task在一个节点上的不同executor中；数据需要在进程间进行传输
* NO_PREF：对于task来说，数据从哪里获取都一样，没有好坏之分
* RACK_LOCAL：机架本地化，数据和task在一个机架的两个节点上；数据需要通过网络在节点之间进行传输
* ANY：数据和task可能在集群中的任何地方，而且不在一个机架中，性能最差


## 1.5 JVM调优

### 首先估计GC的影响

GC调优的第一步就是去统计GC发生的频率和GC消耗时间。
通过添加：
```
./bin/spark-submit \
--name "My app" \
--master local[4] \
--conf spark.eventLog.enabled=false \
--conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" \
myApp.jar
```
在作业运行的时候能够看到worker的日志里面在每次GC的时候就打印出GC信息。

**使用jvisualvm来监控Spark应用程序**

可以看到Spark应用程序堆，线程的使用情况，看不到GC回收的次数时间什么的，从而根据这些数据去优化您的程序。
* 在$SPARK_HOME/conf目录下配置spark-default.conf文件，加入如下配置：
   ```
   spark.driver.extraJavaOptions   -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false
   ```
* 启动Spark应用程序
* 打开jvisualvm.exe监控
    在JDK安装的bin目录下有个jvisualvm.exe，进行配置，依次选择 文件-> 添加JMX连接，在连接文本框填上Driver机器的地址和端口


### 了解GC相关原理
* GC调优的目标
    1. 保证在Old generation只存储有长期存活的RDD；
    2. Young generation有足够的空间存储短期的对象，**避免full GC将作业执行时创建的短期的对象也回收掉**。
    3. 避免Young generation频繁minor GC

* GC调优做法：
    1. 通过收集GC状态检查是否有大量的垃圾回收，如果在作业完成前有多次 full GC，意味着没有足够的内存给执行的task。
    2. 如果进行了多次 minorGC，分配更多的内存给Eden也许会有帮助。只要将Eden的内存E设置成大于每个task需要的内存大小，Young generation 则设置成 -Xmn=4 / 3 * E。
    3. 如果OldGen将要完全占满，可以减少spark.memory.fraction。改变JVM的NewRatio参数，默认设置是2，表示Old generation占用了 2/3的 heap内存，应该设置得足够大超过并超过spark.memory.fraction。另外可以考虑减少Young generation的大小通过调低 -Xmn。
    4. 尝试使用G1GC 收集器 通过配置 -XX:+UseG1GC 。如果executor需要大的堆内存，那么通过配置-XX:G1HeapRegionSize来提高G1 region size 是很重要的。
    5. 如果作业从HDFS中读取数据，可通过作业使用的block大小推测使用的内存大小，读取出来的block通常是存储大小的2-3倍。如果有4个作业去使用一个HDFS的128MB的block，我们预估Eden需要4 * 3 * 128MB。

spark-submit脚本里面，去用--conf的方式，去添加配置：
```
--conf spark.memory.fraction，0.6 -> 0.5 -> 0.4 -> 0.2
--conf spark.shuffle.memoryFraction=0.3
```



### 调节executor堆外内存

有时候，如果你的spark作业处理的数据量特别特别大；然后spark作业一运行，时不时的报错，shuffle file cannot find，executor、task lost，out of memory（内存溢出）；

可能是说executor的堆外内存不太够用，导致executor在运行的过程中会内存溢出；

然后导致后续的stage的task在运行的时候，可能要从一些executor中去拉取shuffle map output文件，但是executor可能已经挂掉了，关联的Block manager也没有了；spark作业彻底崩溃。

上述情况下，就可以去考虑调节一下executor的堆外内存。避免掉某些JVM OOM的异常问题。
此外，堆外内存调节的比较大的时候，对于性能来说，也会带来一定的提升。


spark-submit脚本里面，去用--conf的方式，去添加基于yarn的提交模式配置；
```
--conf spark.yarn.executor.memoryOverhead=2048
```
默认情况下，这个堆外内存上限大概是300多M；真正处理大数据的时候，这里都会出现问题，导致spark作业反复崩溃，无法运行；此时就会去调节这个参数到至少1G（1024M），甚至说2G、4G。

### 调节连接等待时长

我们知道 Executor，优先从自己本地关联的BlockManager中获取某份数据。
如果本地block manager没有的话，那么会通过TransferService，去远程连接其他节点上executor的block manager去获取。
正好碰到那个 exeuctor 的 JVM 在垃圾回收，就会没有响应，无法建立网络连接；
spark默认的网络连接的超时时长，是60s；
就会出现某某file。一串file id。uuid（dsfsfd-2342vs--sdf--sdfsd）。not found。file lost。
报错几次，几次都拉取不到数据的话，可能会导致spark作业的崩溃。也可能会导致DAGScheduler，反复提交几次stage。TaskScheduler，反复提交几次task。大大延长我们的spark作业的运行时间。

可以考虑在spark-submit脚本里面，调节连接的超时时长：
```
--conf spark.core.connection.ack.wait.timeout=300
```



## OOM相关

Spark中的OOM问题不外乎两种情况：
* map执行中内存溢出
* shuffle后内存溢出，shuffle后内存溢出的shuffle操作包括join，reduceByKey，repartition等操作。

可以先理一下Spark内存模型，再总结各种OOM的情况相对应的解决办法和性能优化方面的总结。

* map过程产生大量对象导致内存溢出

这种溢出的原因是在单个map中产生了大量的对象导致的，
```
例如：rdd.map(x=>for(i <- 1 to 10000) yield i.toString)
```
这个操作在rdd每个对象都产生了10000个对象，这肯定很容易产生内存溢出的问题。
针对这种问题，**在不增加内存的情况下，可以通过减少每个Task的大小，让Executor的内存也能够装得下。**
具体做法可以在会产生大量对象的map操作之前调用repartition方法，分区成更小的块传入map。

```
例如：rdd.repartition(10000).map(x=>for(i <- 1 to 10000) yield i.toString)。
```
面对这种问题注意，不能使用rdd.coalesce方法，这个方法只能减少分区，不能增加分区，不会有shuffle的过程。

* 数据不平衡导致内存溢出

也可如上面进行repartition

* coalesce减少分区导致内存溢出

因为hdfs中不适合存小文件，所以Spark计算后如果产生的文件太小，会调用coalesce合并文件再存入hdfs中。
这会导致一个问题：有100个文件，现在调用coalesce(10)，意味着能够有100个Task，最后只产生10个文件，
因为coalesce并不是shuffle操作，意味着coalesce并不是先执行100个Task，再将Task的执行结果合并成10个，
而是从头到位只有10个Task在执行，每个Task同时一次读取10个文件，使用的内存是原来的10倍，这导致了OOM。

解决这个问题的方法是令程序按照我们想的先执行100个Task再将结果合并成10个文件，
可以通过repartition解决，调用repartition(10)，
因为这就有一个shuffle的过程，shuffle前后是两个Stage，一个100个分区，一个是10个分区，就能按照我们的想法执行。

* shuffle后内存溢出：

shuffle内存溢出的情况可以说都是shuffle后，单个文件过大导致的。在shuffle的使用，需要传入一个partitioner，默认的partitioner都是HashPatitioner，默认值是父RDD中最大的分区数，这个参数通过spark.default.parallelism控制 (只对HashPartitioner有效，在spark-sql中用spark.sql.shuffle.partitions) 。如果是别的partitioner导致的shuffle内存溢出，就需要从partitioner的代码增加partitions的数量。



## 1.6 Suffle调优

### 1.6.1 shuffle流程
shuffle，一定是分为两个stage来完成的。
因为这其实是个逆向的过程，不是stage决定shuffle，是shuffle决定stage。
在某个action触发job的时候，DAGScheduler会负责划分job为多个stage。划分的依据，就是如果发现有会触发shuffle操作的算子（reduceByKey）。

每一个shuffle的前半部分stage的task，**每个task**都会**创建下一个stage的task数量相同的文件**，
比如下一个stage会有10个task，那么当前stage每个task都会创建10份文件；
会将同一个key对应的values，写入同一个文件中的；
不同节点上的task，也一定会将同一个key对应的values，写入下一个stage的同一个task对应的文件中。

shuffle的后半部分stage的task，每个task都会从各个节点上的task写的属于自己的那一份文件中，拉取key, value对；
然后task会有一个内存缓冲区，然后会用HashMap，进行key, values的汇聚；
task会用我们自己定义的聚合函数reduceByKey(_+_)，把所有values进行一对一的累加；聚合出来最终的值。


### 1.6.2 合并map端输出文件（优化的hashShuffle)

```
sparConf().set("spark.shuffle.consolidateFiles", "true")
```

开启了map端输出文件的合并机制之后：

第一个stage，同时运行cpu core个task，比如cpu core是2个，并行运行2个task；每个task都创建下一个stage的task数量个文件；

第一个stage，并行运行的2个task执行完以后；就会执行另外两个task；**另外2个task不会再重新创建输出文件；而是复用之前的task创建的map端输出文件，将数据写入上一批task的输出文件中**。

第二个stage，task在拉取数据的时候，就不会去拉取上一个stage每一个task为自己创建的那份输出文件了；而是拉取少量的输出文件，每个输出文件中，可能包含了多个task给自己的map端输出。


### 1.6.3 调节map端内存缓冲与reduce端内存占比

深入一下shuffle原理：
shuffle的map task：
输出到磁盘文件的时候，统一都会先写入每个task自己关联的一个内存缓冲区。
这个缓冲区大小，默认是32kb。当**内存缓冲区满溢之后，会进行spill溢写操作到磁盘文件中去**。
数据量比较大的情况下可能导致多次溢写。

shuffle的reduce端task：
在拉取到数据之后，会用hashmap的数据格式，来对各个key对应的values进行汇聚的时候。
使用的就是自己对应的executor的内存，executor（jvm进程，堆），默认executor内存中划分给reduce task进行聚合的比例，是0.2。
要是拉取过来的数据很多，那么在内存中，放不下；就将在内存放不下的数据，都spill（溢写）到磁盘文件中去。
数据大的时候磁盘上溢写的数据量越大，后面在进行聚合操作的时候，很可能会多次读取磁盘中的数据，进行聚合。

**怎么调节？**
看Spark UI，shuffle的磁盘读写的数据量很大，就意味着最好调节一些shuffle的参数。进行调优。

```
set(spark.shuffle.file.buffer，64)      // 默认32k  每次扩大一倍，看看效果。
set(spark.shuffle.memoryFraction，0.2)  // 每次提高0.1，看看效果。
```

很多资料都会说这两个参数，是调节shuffle性能的不二选择，实际上不是这样的。
以实际的生产经验来说，这两个参数没有那么重要，shuffle的性能不是因为这方面的原因导致的。


### 1.6.4 HashShuffleManager与SortShuffleManager

**上面我们所讲shuffle原理是指HashShuffleManager**，是很过时的shuffle manager。
之前讲解的一些调优的点，比如consolidateFiles机制、map端缓冲、reduce端内存占比。这些对任何shuffle manager都是有用的。

在spark 1.5.x后，又出来了一种tungsten-sort shuffleMnager。效果跟sort shuffle manager是差不多的。
唯一的不同之处在于tungsten-sort shuffleMnager，是使用了自己实现的一套内存管理机制以及堆内存，性能上有很大的提升可以避免shuffle过程中产生的大量的OOM，GC。

SortShuffleManager与HashShuffleManager两点不同：
1. SortShuffleManager会对每个reduce task要处理的数据，进行排序（默认的）。
2. SortShuffleManager会一个task，只会写入一个磁盘文件，不同reduce task的数据，用offset来划分界定。


**hash、sort、tungsten-sort。如何来选择？**
1. 需不需要数据默认就让spark给你进行排序？就好像mapreduce，默认就是有按照key的排序。
如果不需要的话，其实还是建议搭建就使用最基本的HashShuffleManager，因为最开始就是考虑的是不排序，换取高性能；

2、什么时候需要用sort shuffle manager？如果你需要你的那些数据按key排序了，那么就选择这种吧。
而且要注意，reduce task的数量应该是超过200的，这样sort、merge（多个文件合并成一个）的机制，才能生效把。
但是这里要注意，你一定要自己考量一下，有没有必要在shuffle的过程中，就做这个事情，毕竟对性能是有影响的。

3、如果你不需要排序，而且你希望你的每个task输出的文件最终是会合并成一份的，你自己认为可以减少性能开销；
可以去调节bypassMergeThreshold这个阈值，比如你的reduce task数量是500，默认阈值是200，所以默认还是会进行sort和直接merge的；
可以将阈值调节成550，不会进行sort，按照hash的做法，每个reduce task创建一份输出文件，最后合并成一份文件。
（一定要提醒大家，这个参数，其实我们通常不会在生产环境里去使用，也没有经过验证说，这样的方式，到底有多少性能的提升）

4、如果你想选用sort based shuffle manager，而且你们公司的spark版本比较高，
是1.5.x版本的，那么可以考虑去尝试使用tungsten-sort shuffle manager。
看看性能的提升与稳定性怎么样。（唉，开源出来的项目都是落后了快五年了的）


总结：
1、在生产环境中，不建议大家贸然使用第三点和第四点：
2、如果你不想要你的数据在shuffle时排序，那么就自己设置一下，用hash shuffle manager。
3、如果你的确是需要你的数据在shuffle时进行排序的，那么就默认不用动，默认就是sort shuffle manager；或者是什么？如果你压根儿不care是否排序这个事儿，那么就默认让他就是sort的。调节一些其他的参数（consolidation机制）。（80%，都是用这种）

```
new SparkConf().set("spark.shuffle.manager", "hash")  // hash、tungsten-sort、默认为sort
new SparkConf().set("spark.shuffle.sort.bypassMergeThreshold", "550")   // 默认200
```
当reduce task数量少于等于200；map task创建的输出文件小于等于200的；会将所有的输出文件合并为一份文件。且不进行sort排序，节省了性能开销。


## 1.7 算子调优

### 1.7.1 MapPartitions提升Map类操作性能

这里需要稍微讲一下RDD和DataFrame的区别。
RDD强调的是不可变对象，每个RDD都是不可变的，当调用RDD的map类型操作的时候，都是产生一个新的对象。
这就导致如果对一个RDD调用大量的map类型操作的话，每个map操作会产生一个到多个RDD对象，
这虽然不一定会导致内存溢出，但是会产生大量的中间数据，增加了gc操作。
另外RDD在调用action操作的时候，会出发Stage的划分，但是在每个Stage内部可优化的部分是不会进行优化的，
例如rdd.map(_+1).map(_+1)，这个操作在数值型RDD中是等价于rdd.map(_+2)的，但是RDD内部不会对这个过程进行优化。

DataFrame则不同，DataFrame由于有类型信息所以是可变的，并且在可以使用sql的程序中，都有除了解释器外，都会有一个sql优化器Catalyst，

上面说到的这些RDD的弊端，有一部分就可以使用mapPartitions进行优化，mapPartitions可以同时替代rdd.map, rdd.filter, rdd.flatMap的作用，
所以在长操作中，可以在mapPartitons中将RDD大量的操作写在一起，避免产生大量的中间rdd对象，
另外是mapPartitions在一个partition中可以复用可变类型，这也能够避免频繁的创建新对象。


普通的mapToPair，当一个partition中有1万条数据，function要执行和计算1万次。
但是，使用MapPartitions操作之后，一个task仅仅会执行一次function，function一次接收所有的partition数据。只要执行一次就可以了，性能比较高。
但是MapPartitions操作，对于大量数据来说，一次传入一个function以后，**可能一下子内存不够而且又没法腾出内存空间的话，可能就OOM！**

在项目中，自己先去估算一下RDD的数据量，以及每个partition的量，还有自己分配给每个executor的内存资源，看看一下子内存容纳所有的partition数据。


### 1.7.2 使用coalesce减少分区数量

从源码中可以看出repartition方法其实就是调用了coalesce方法，shuffle为 true 的情况. 现在假设 RDD 有X个分区,需要重新划分成Y个分区.

1.如果x<y,说明x个分区里有数据分布不均匀的情况,利用HashPartitioner把x个分区重新划分成了y个分区,此时,需要把shuffle设置成true才行,因为如果设置成false,不会进行shuffle操作,此时父RDD和子RDD之间是窄依赖,这时并不会增加RDD的分区.

2.如果x>y,需要先把x分区中的某些个分区合并成一个新的分区,然后最终合并成y个分区,此时,需要把coalesce方法的shuffle设置成false.

总结:如果想要增加分区的时候,可以用repartition或者coalesce+true。但是一定要有shuffle操作,分区数量才会增加。


RDD这种filter之后，RDD中的每个partition的数据量，可能都不太一样了。
问题：
1、每个partition数据量变少了，但是在后面进行处理的时候，还跟partition数量一样数量的task，来进行处理；有点浪费task计算资源。
2、每个partition的数据量不一样，会导致后面的每个task处理每个partition的时候，每个task要处理的数据量就不同，处理速度相差大，导致数据倾斜。。。。

针对上述的两个问题，能够怎么解决呢？

1、针对第一个问题，希望可以进行partition的压缩吧，因为数据量变少了，那么partition其实也完全可以对应的变少。
比如原来是4个partition，现在完全可以变成2个partition。
那么就只要用后面的2个task来处理即可。就不会造成task计算资源的浪费。

2、针对第二个问题，其实解决方案跟第一个问题是一样的；也是去压缩partition，尽量让每个partition的数据量差不多。
那么这样的话，后面的task分配到的partition的数据量也就差不多。
不会造成有的task运行速度特别慢，有的task运行速度特别快。避免了数据倾斜的问题。

主要就是用于在filter操作之后，添加coalesce算子，针对每个partition的数据量各不相同的情况，来压缩partition的数量。
减少partition的数量，而且让每个partition的数据量都尽量均匀紧凑。


### 1.7.3 foreachPartition优化写数据库性能
默认的foreach的性能缺陷在哪里？
1. 对于每条数据，都要单独去调用一次function，task为每个数据都要去执行一次function函数。如果100万条数据，（一个partition），调用100万次。性能比较差。
2. 浪费数据库连接资源。

在生产环境中，都使用foreachPartition来写数据库
1、对于我们写的function函数，就调用一次，一次传入一个partition所有的数据，但是太多也容易OOM。
2、主要创建或者获取一个数据库连接就可以
3、只要向数据库发送一次SQL语句和多组参数即可

**local模式跑的时候foreachPartition批量入库会卡住**，可能资源不足，因为用standalone集群跑的时候不会出现。


### 1.7.4 repartition 解决Spark SQL低并行度的性能问题
并行度：可以这样调节：
1、spark.default.parallelism   指定为全部executor的cpu core总数的2~3倍
2、textFile()，传入第二个参数，指定partition数量（比较少用）
**但是通过spark.default.parallelism参数指定的并行度，只会在没有Spark SQL的stage中生效**。
Spark SQL自己会默认根据hive表对应的hdfs文件的block，自动设置Spark SQL查询所在的那个stage的并行度。

比如第一个stage，用了Spark SQL从hive表中查询出了一些数据，然后做了一些transformation操作，接着做了一个shuffle操作（groupByKey），这些都不会应用指定的并行度可能会有非常复杂的业务逻辑和算法，就会导致第一个stage的速度，特别慢；下一个stage，在shuffle操作之后，做了一些transformation操作才会变成你自己设置的那个并行度。

解决上述Spark SQL无法设置并行度和task数量的办法，是什么呢？

repartition算子，可以将你用Spark SQL查询出来的RDD，使用repartition算子时，去重新进行分区，此时可以分区成多个partition，比如从20个partition分区成100个。
就可以避免跟Spark SQL绑定在一个stage中的算子，只能使用少量的task去处理大量数据以及复杂的算法逻辑。


### 1.7.5 reduceByKey本地聚合介绍
reduceByKey，相较于普通的shuffle操作（比如groupByKey），它有一个特点：底层基于CombineByKey，会进行map端的本地聚合。
对map端给下个stage每个task创建的输出文件中，写数据之前，就会进行本地的combiner操作，也就是说对每一个key，对应的values，都会执行你的算子函数。

用reduceByKey对性能的提升：
1、在本地进行聚合以后，在map端的数据量就变少了，减少磁盘IO。而且可以减少磁盘空间的占用。
2、下一个stage，拉取数据的量，也就变少了。减少网络的数据传输的性能消耗。
3、在reduce端进行数据缓存的内存占用变少了，要进行聚合的数据量也变少了。

reduceByKey在什么情况下使用呢？
1、简单的wordcount程序。
2、对于一些类似于要对每个key进行一些字符串拼接的这种较为复杂的操作，可以自己衡量一下，其实有时，也是可以使用reduceByKey来实现的。
但是不太好实现。如果真能够实现出来，对性能绝对是有帮助的。


## 1.8 troubleshooting调优

### 1.8.1 控制shuffle reduce端缓冲大小以避免OOM
map端的task是不断的输出数据的，数据量可能是很大的。
但是，在map端写过来一点数据，reduce端task就会拉取一小部分数据，先放在buffer中，立即进行后面的聚合、算子函数的应用。
每次reduece能够拉取多少数据，就由buffer来决定。然后才用后面的executor分配的堆内存占比（0.2），hashmap，去进行后续的聚合、函数的执行。

reduce端缓冲默认是48MB（buffer），可能会出什么问题？
缓冲达到最大极限值，再加上你的reduce端执行的聚合函数的代码，可能会创建大量的对象。reduce端的内存中，就会发生内存溢出的问题。
这个时候，就应该减少reduce端task缓冲的大小。我宁愿多拉取几次，但是每次同时能够拉取到reduce端每个task的数量，比较少，就不容易发生OOM内存溢出的问题。

另外，如果你的Map端输出的数据量也不是特别大，然后你的**整个application的资源也特别充足，就可以尝试去增加这个reduce端缓冲大小的，比如从48M，变成96M**。
这样每次reduce task能够拉取的数据量就很大。需要拉取的次数也就变少了。
最终达到的效果，就应该是性能上的一定程度上的提升。
设置
spark.reducer.maxSizeInFlight


### 1.8.2 解决JVM GC导致的shuffle文件拉取失败
比如，executor的JVM进程，内存不够了，发生GC，导致BlockManger，netty通信都停了。
下一个stage的executor，可能是还没有停止掉的，task想要去上一个stage的task所在的exeuctor，去拉取属于自己的数据，结果由于对方正在GC，就导致拉取了半天没有拉取到。
可能会报错shuffle file not found。
但是，可能下一个stage又重新提交了stage或task以后，再执行就没有问题了，因为可能第二次就没有碰到JVM在gc了。
有的时候，出现这种情况以后，会重新去提交stage、task。重新执行一遍，发现就好了。没有这种错误了。


spark.shuffle.io.maxRetries 3
spark.shuffle.io.retryWait 5s

针对这种情况，我们完全可以进行预备性的参数调节。
增大上述两个参数的值，达到比较大的一个值，尽量保证第二个stage的task，一定能够拉取到上一个stage的输出文件。
尽量避免因为gc导致的shuffle file not found，无法拉取到的问题。


### 1.8.3 yarn-cluster模式的JVM内存溢出无法执行问题
总结一下yarn-client和yarn-cluster模式的不同之处：
yarn-client模式，driver运行在本地机器上的；yarn-cluster模式，driver是运行在yarn集群上某个nodemanager节点上面的。
yarn-client的driver运行在本地，通常来说本地机器跟yarn集群都不会在一个机房的，所以说性能可能不是特别好；yarn-cluster模式下，driver是跟yarn集群运行在一个机房内，性能上来说，也会好一些。

实践经验，碰到的yarn-cluster的问题：

有的时候，运行一些包含了spark sql的spark作业，可能会碰到yarn-client模式下，可以正常提交运行；yarn-cluster模式下，可能是无法提交运行的，会报出JVM的PermGen（永久代）的内存溢出，会报出PermGen Out of Memory error log。

yarn-client模式下，driver是运行在本地机器上的，spark使用的JVM的PermGen的配置，是本地的spark-class文件（spark客户端是默认有配置的），JVM的永久代的大小是128M，这个是没有问题的；但是在yarn-cluster模式下，driver是运行在yarn集群的某个节点上的，使用的是没有经过配置的默认设置（PermGen永久代大小），82M。

spark-submit脚本中，加入以下配置即可：
--conf spark.driver.extraJavaOptions="-XX:PermSize=128M -XX:MaxPermSize=256M"

另外，sql有大量的or语句。可能就会出现一个driver端的jvm stack overflow。
基本上就是由于调用的方法层级过多，因为产生了非常深的，超出了JVM栈深度限制的，递归。spark sql内部源码中，在解析sqlor特别多的话，就会发生大量的递归。
建议不要搞那么复杂的spark sql语句。
采用替代方案：将一条sql语句，拆解成多条sql语句来执行。
每条sql语句，就只有100个or子句以内；一条一条SQL语句来执行。



## 1.9 数据倾斜调优

有key倾斜了，再怎么repartition也没用啊?

可以的，只要你机器无限多，给他reptition的无限小
我只会一招就是repartiton，谁oom都给谁repartition，repartition不行就count然后filter
repartition不行就count然后filter啊，读信息不要读一半啊
如果repartition不管用就是严重数据倾斜，那么首先count找到引发倾斜的key，然后看到底是什么原因引起的，然后单独处理
repartition够用就是资源不够，又不可能加资源，那就慢点跑呗
如果对计算指标没影响直接过滤掉
有影响该干啥干啥，网上搜一下很全了

业务其实更多的是非法值，加key打散也是浪费资源，非法值就是裹一层Option，模式匹配一把梭

### 1.9.1 数据倾斜的原理、现象、产生原因与定位
**原因**
shuffle操作是按照key，来进行values的数据的输出、拉取和聚合的。所以**同一个key的values，一定是分配到一个reduce task进行处理的**。当其中一个key的数据量很大就会发生数据倾斜。

定位：在自己的程序里面找找，哪些地方用了会产生shuffle的算子，groupByKey、countByKey、reduceByKey、join。
看log，看看是执行到了第几个stage。哪一个stage，task特别慢或者只有一部分task在工作，就能够从spark代码的stage划分，通过stage定位到你的代码，哪里发生了数据倾斜。



**第一个方案：聚合源数据**
做一些聚合的操作：groupByKey、reduceByKey，说白了就是对每个key对应的values执行一定的计算。

spark作业的数据来源如果是hive，可以直接在生成hive表的hive etl中，对数据进行聚合。
比如按key来分组，将key对应的所有的values，全部用一种特殊的格式，拼接到一个字符串里面去，每个key就只对应一条数据。比如

```
key=sessionid, value: action_seq=1|user_id=1|search_keyword=火锅|category_id=001;action_seq=2|user_id=1|search_keyword=涮肉|category_id=001”。

然后在spark中，拿到key=sessionid，values<Iterable>。

```
或者在hive里面就进行reduceByKey计算。
spark中就不需要再去执行groupByKey+map这种操作了。
直接对每个key对应的values字符串进行map进行你需要的操作即可。也就根本不可能导致数据倾斜。

具体怎么去在hive etl中聚合和操作，就得根据你碰到数据倾斜问题的时候，你的spark作业的源hive表的具体情况，具体需求，具体功能，具体分析。

具体对于我们的程序来说，完全可以将aggregateBySession()这一步操作，放在一个hive etl中来做，形成一个新的表。
对每天的用户访问行为数据，都按session粒度进行聚合，写一个hive sql。

**在spark程序中，就不要去做groupByKey+mapToPair这种算子了**。
直接从当天的session聚合表中，用SparkSQL查询出来对应的数据，即可。
这个RDD在后面就可以使用了。


**第二个方案：过滤导致倾斜的key**
如果你能够接受某些数据，在spark作业中直接就摒弃掉，不使用。
比如说，总共有100万个key。只有2个key，是数据量达到10万的。其他所有的key，对应的数量都是几十。
这个时候，你自己可以去取舍，如果业务和需求可以理解和接受的话，在你从hive表查询源数据的时候，直接在sql中用where条件，过滤掉某几个key。
那么这几个原先有大量数据，会导致数据倾斜的key，被过滤掉之后，那么在你的spark作业中，自然就不会发生数据倾斜了。


### 1.9.2 提高shuffle操作的reduce并行度
**将reduce task的数量，变多，就可以让每个reduce task分配到更少的数据量。**
这样的话，也许就可以缓解，或者甚至是基本解决掉数据倾斜的问题。
提升shuffle reduce端并行度，怎么来操作？
在调用shuffle算子的时候，传入进去一个参数。
就代表了那个shuffle操作的reduce端的并行度。那么在进行shuffle操作的时候，就会对应着创建指定数量的reduce task。
按照log，找到发生数据倾斜的shuffle操作，给它传入一个并行度数字，这样的话，原先那个task分配到的数据，肯定会变少。就至少可以避免OOM的情况，程序至少是可以跑的。

但是**没有从根本上改变数据倾斜的本质和问题。**
不像第一个和第二个方案（直接避免了数据倾斜的发生）。
原理没有改变，只是说，尽可能地去缓解和减轻shuffle reduce task的数据压力，以及数据倾斜的问题。

实际生产环境中的经验。
1、如果最理想的情况下，提升并行度以后，减轻了数据倾斜的问题，那么就最好。就不用做其他的数据倾斜解决方案了。
2、不太理想的情况下，就是比如之前某个task运行特别慢，要5个小时，现在稍微快了一点，变成了4个小时；或者是原先运行到某个task，直接OOM，现在至少不会OOM了，但是那个task运行特别慢，要5个小时才能跑完。
那么，如果出现第二种情况的话，各位，就立即放弃第三种方案，开始去尝试和选择后面的方案。


### 1.9.3 使用随机key实现双重聚合
1、原理
第一轮聚合的时候，对key进行打散，将原先一样的key，变成不一样的key，相当于是将每个key分为多组；比如原来是
```
(5,44)、(6,45)、(7,45)
就可以对key添加一个随机数
(1_5,44)、(3_6,45)、(2_7,45)
针对多个组，进行key的局部聚合；
接着，再去除掉每个key的前缀，恢复成
(5,44)、(6,45)、(7,45)
然后对所有的key，进行全局的聚合。
```
对groupByKey、reduceByKey造成的数据倾斜，有比较好的效果。

2、使用场景
（1）groupByKey
（2）reduceByKey

### 1.9.4 将导致倾斜的key单独进行join
这个方案关键之处在于: 
将发生数据倾斜的key，单独拉出来，放到一个RDD中去；
用这个原本会倾斜的key RDD跟其他RDD，单独去join一下，
key对应的数据，可能就会分散到多个task中去进行join操作。
这个key跟之前其他的key混合在一个RDD中时，肯定是会导致一个key对应的所有数据，都到一个task中去，就会导致数据倾斜。

**这种方案什么时候适合使用？**

针对你的RDD的数据，你可以自己把它转换成一个中间表，或者是直接用countByKey()的方式，你可以看一下这个RDD各个key对应的数据量；
RDD有一个或少数几个key，是对应的数据量特别多；
此时可以采用咱们的这种方案，单拉出来那个最多的key；
单独进行join，尽可能地将key分散到各个task上去进行join操作。

### 1.9.5  使用随机数以及扩容表进行join
这个方案是没办法彻底解决数据倾斜的，更多的，是一种对数据倾斜的缓解。
局限性：
1、因为join两个RDD都很大，就没有办法去将某一个RDD扩的特别大，一般是10倍。
2、如果就是10倍的话，那么数据倾斜问题，只能说是缓解和减轻，不能说彻底解决。

步骤：
1、选择一个RDD，要用flatMap，进行扩容，将每条数据，映射为多条数据，每个映射出来的数据，都带了一个n以内的随机数，通常来说，会选择10。
2、将另外一个RDD，做普通的map映射操作，每条数据，都打上一个10以内的随机数。
3、最后，将两个处理后的RDD，进行join操作。
4、join完以后，可以执行map操作，去将之前打上的随机数给去掉，然后再和另外一个普通RDD join以后的结果，进行union操作。

sample采样倾斜key并单独进行join
将key，从另外一个RDD中过滤出的数据，可能只有一条，或者几条，此时，咱们可以任意进行扩容，扩成1000倍。
将从第一个RDD中拆分出来的那个倾斜key RDD，打上1000以内的一个随机数。
打散成100份，甚至1000份去进行join，那么就肯定没有数据倾斜的问题了吧。
这种情况下，还可以配合上，提升shuffle reduce并行度，join(rdd, 1000)。












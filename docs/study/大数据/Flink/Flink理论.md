# Flink理论

# 一、Flink简介

Apache Flink 是一个框架和分布式处理引擎，用于对无界和有界**数据流**进行**有状态**计算。

目标：低延迟、高吞吐、结果的准确性和良好的容错性



处理流数据行业：电商和市场营销（数据报表、广告投放、业务流程）

​                               物联网（传感器实时数据采集显示、实时报警、交通运输）

​                                电信业（基站流量调配）

​                                银行和金融业（实时结算和通知推送，实时监测异常行为）



## 1.1 架构

### 1.1.1 传统数据处理架构

![image-20211008170501860](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008170501860.png)

管理系统、订单系统、wed应用等，读取并存储大量数据在数据库中。

数据量大的时候，数据库做联表查询时代价就会很高。

特点：实时性好，但是海量数据、高并发的时候不行。

### 1.1.2 离线分析处理架构

![image-20211008170856808](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008170856808.png)

特点：高并发可以，但是低延迟做不到



### 1.1.3 有状态的流式处理（第一代流处理器）

![image-20211008183614233](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183614233.png)

**低延迟：**事务处理原则，来一个处理一个（关系数据库关联查询很麻烦）

**高吞吐：**所以直接把数据存在本地内存，存成本地状态。来了事务后，与本地状态进行计算输出，状态更新，就改。

**容错：** 数据在内存里，不稳定。节点挂了，数据丢失。所以需要存盘和故障恢复机制。所以设计了周期性的检查点（periodic checkpoint）,定期保存，出现故障，恢复到上个存盘点。检查点放在远程的持久化空间。

**问题：**分布式有延迟，数据的时间顺序保证不了



### 1.1.4 第二代流处理器

![image-20211008183551176](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183551176.png)

**lambda架构：**用两套系统（批处理和流处理），同时保证低延迟和结果正确。流处理保证低延迟，批处理保证正确结果。

**效果：**用户会很快得到一个结果，但不一定结果正确。过一段时间后，获得正确结果。

**特点：**1. 需要维护两套系统



### 1.1.5 第三代流处理器

![image-20211008183940037](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183940037.png)

**Storm**是第一代的流处理器（特点：**低延迟**，吞吐量不行，时间顺序不保证），**spark streaming**是批处理演变过来的（特点：**高吞吐**，保证正确，但不低延迟，时间顺序不保证）。



## 1.2 Flink特点

* 事件驱动：用户有事件，然后存到事件日志，读到Flink内部。然后查询本地状态（定期存盘，放在远程持久化存储空间）。输入和本地状态计算，得到结果，触发操作或者输出到输出事件日志。

![image-20211008184455395](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008184455395.png)

* 基于流的世界观

  离线数据是有界流，实时数据是无界流

* 分层API

  ![image-20211008190354059](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008190354059.png)

顶层：表

第二层：无界流主要用的是DataStream API，有界主要是DataSet API.

底层：可以拿到事件、状态、定时间。

* 支持事件时间（event-time）和处理时间（processing-time）

* 精确一次（exactly-once）的状态一致性保证

  结果正确（恢复后也要保证）

* 低延迟，每秒处理数百万个事件，毫秒级延迟

* 与众多常用存储系统的连接

* 高可用，动态扩展，实现7*24小时全天候运行



## 1.3 Flink vs Spack Streaming

* 流与微批

  Flink 流 毫秒级，Spark Streaming 微批 秒级。

* 数据模型

  spark：RDD模型，Spark streaming 的 DStream实际上就是一组组小批数据RDD的集合

  flink基本数据模型是数据流，以及事件

* 运行时架构

  spark是批计算,将DAG划分为不同的stage,一个完成后才可以计算下一个

  flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理。



# 二、Flink wordcount

## 2.1 maven pom.xml 依赖配置

```
    <dependencies>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>1.13.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.12</artifactId>
            <version>1.13.1</version>
<!--            <scope>provided</scope>-->
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_2.12</artifactId>
            <version>1.13.1</version>
        </dependency>
    </dependencies>
```

## 2.2 flink  wordcount

```
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        //创建运行时流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //线程数（分区数），默认是计算机的核数。输出前的3> 表示在那个分区上计算的
        env.setParallelism(4);
        //从文件中读取数据
        String inputPath = "src/main/resources/wc.txt";
        DataStream<String> inputDataStream = env.readTextFile(inputPath);
        //用parameter tool工具从程序启动参数中提取配置项
//        ParameterTool parameterTool = ParameterTool.fromArgs(args);
//        String host = parameterTool.get("host");
//        int port = parameterTool.getInt("port");
        //从socket文本流读取数据
//        env.socketTextStream(host,port);

        //基于数据流进行转换计算
        SingleOutputStreamOperator<Tuple2<String, Integer>> sum = inputDataStream.flatMap(new MyFlatMapperFunction())
                .keyBy(0)
                .sum(1);
        //任务定义
        sum.print();
        //任务执行
        env.execute();

    }
    static class MyFlatMapperFunction implements FlatMapFunction<String, Tuple2<String,Integer>>{
        @Override
        public void flatMap(String s, Collector<Tuple2<String, Integer>> collector) throws Exception {
            String[] words = s.split(" ");
            for(String word : words){
                collector.collect(new Tuple2<>(word,1));
            }
        }
    }
}
```



# 三、Flink运行时架构

## 3.1 Flink四大组件

![image-20211207094614608](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211207094614608.png)

### 3.1.1 作业管理器（JobManager）

 控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的JobManager所控制执行。

 JobManager会先接收到**要执行的应用程序**，这个应用程序会包括：

* 作业图（JobGraph）
* 逻辑数据流图（logical dataflow graph）
* 打包了所有的类、库和其它资源的JAR包。

 JobManager会把JobGraph转换成一个物理层面的数据流图，这个图被叫做“**执行图**”（ExecutionGraph），包含了所有可以并发执行的任务。

 **JobManager会向资源管理器（ResourceManager）请求执行任务必要的资源，也就是任务管理器（TaskManager）上的插槽（slot）。一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的TaskManager上**。

 在运行过程中，JobManager会负责所有需要中央协调的操作，比如说检查点（checkpoints）的协调。

​       

### 3.1.2任务管理器（TaskManager）

 Flink中的工作进程。通常在Flink中会有多个TaskManager运行，每一个TaskManager都包含了一定数量的插槽（slots）。**插槽的数量限制了TaskManager能够执行的任务数量**。

 启动之后，TaskManager会向资源管理器**注册它的插槽**；收到资源管理器的指令后，TaskManager就会将一个或者多个插槽提供给JobManager调用。JobManager就可以向插槽分配任务（tasks）来执行了。

 **在执行过程中，一个TaskManager可以跟其它运行同一应用程序的TaskManager交换数据**。



### 3.1.3资源管理器（ResourceManager）

主要负责管理任务管理器（TaskManager）的**插槽（slot）**，TaskManger插槽是Flink中定义的处理资源单元。Flink为不同的环境和资源管理工具提供了不同资源管理器，比如YARN、Mesos、K8s，以及standalone部署。 **当JobManager申请插槽资源时，ResourceManager会将有空闲插槽的TaskManager分配给JobManager**。如果ResourceManager没有足够的插槽来满足JobManager的请求，它还可以向资源提供平台发起会话，以提供启动TaskManager进程的容器。

 另外，**ResourceManager还负责终止空闲的TaskManager，释放计算资源**。



### 3.1.4分发器（Dispatcher）

 可以跨作业运行，它为应用提交提供了REST接口。

 当一个应用被提交执行时，分发器就会启动并将应用**移交**给一个JobManager。由于是REST接口，所以Dispatcher可以作为集群的一个**HTTP接入点**，这样就能够**不受防火墙阻挡**。Dispatcher也会启动一个**Web UI**，用来方便地展示和监控作业执行的信息。

 *Dispatcher在架构中可能并不是必需的，这取决于应用提交运行的方式。*





## 3.2 任务提交流程

![image-20211207223625524](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211207223625524.png)

首先，提交应用给Dispatcher

Dispatcher启动一个JobManager,并提交应用程序。

JobManager知道了任务数、并行数。然后申请资源。

ResourceManager收到请求，要用资源，才去启动TaskManager

TaskManager启动后，注册slot.

ResourceManager根据需要的slot向多个Taskmanager发出提供slot的指令

TaskManager提供slot给JobManager,JobManager提交要执行的任务 。



### 3.2.1 Yarn上作业提交流程

![image-20211213093432995](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211213093432995.png)

1. Flink任务提交后，Client向HDFS上传Flink的Jar包和配置
2. 之后客户端向Yarn ResourceManager提交任务，ResourceManager分配Container资源并通知对应的NodeManager启动ApplicationMaster
3. ApplicationMaster启动后加载Flink的Jar包和配置构建环境，去启动JobManager，之后**JobManager向Flink自身的RM进行申请资源，自身的RM向Yarn 的ResourceManager申请资源(因为是yarn模式，所有资源归yarn RM管理)启动TaskManager**
4. Yarn ResourceManager分配Container资源后，由ApplicationMaster通知资源所在节点的NodeManager启动TaskManager
5. NodeManager加载Flink的Jar包和配置构建环境并启动TaskManager，TaskManager启动后向JobManager发送心跳包，并等待JobManager向其分配任务。

## 3.3 任务调度原理

![image-20211207234948465](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211207234948465.png)



1. 客户端不是运行时和程序执行的一部分，但它用于准备并发送dataflow(JobGraph)给Master(JobManager)，然后，客户端断开连接或者维持连接以等待接收计算结果。而Job Manager会产生一个**执行图(**Dataflow Graph)

2. 当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的 TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个 TaskManager 去执行，然后 TaskManager 将**心跳和统计信息**汇报给 JobManager。TaskManager 之间以**流**的形式进行数据的传输。上述三者均为独立的 JVM 进程。

3. Client 为提交 Job 的客户端，可以是运行在任何机器上（与 JobManager 环境连通即可）。提交 Job 后，Client 可以结束进程（Streaming的任务），也可以不结束并等待结果返回。

4. JobManager 主要负责**调度 Job** 并协调 Task **做 checkpoint**，职责上很像 Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行。

5. TaskManager 在启动的时候就设置好了槽位数（Slot），**每个 slot 能启动一个 Task**，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 **Netty 连接**，接收数据并处理。

   *注：如果一个Slot中启动多个线程，那么这几个线程类似CPU调度一样共用同一个slot*

### 3.3.1 TaskManger与Slots

要点：

+ 考虑到Slot分组，所以实际运行Job时所需的Slot总数 = 每个Slot组中的最大并行度。

  eg（1，1，2，1）,其中第一个归为组“red”、第二个归组“blue”、第三个和第四归组“green”，那么运行所需的slot即max（1）+max（1）+max（2，1） =  1+1+2  = 4

  可以在代码中通过算子的`.slotSharingGroup("组名")`指定算子所在的Slot组名，默认每一个算子的SlotGroup和上一个算子相同，而默认的SlotGroup就是"default"。

  **同一个SlotGroup的算子能共享同一个slot，不同组则必须另外分配独立的Slot。**



+ Flink中每一个worker(TaskManager)都是一个**JVM**进程，它可能会在独立的线程上执行一个或多个subtask。

+ 为了控制一个worker能接收多少个task，worker通过task slot来进行控制（一个worker至少有一个task slot）。

+ 默认情况下，Flink允许子任务共享slot，即使它们是不同任务的子任务（前提需要来自同一个Job）。这样结果是，**一个slot可以保存作业的整个管道pipeline**。

  + **不同任务共享同一个Slot的前提：这几个任务前后顺序不同，如上图中Source和keyBy是两个不同步骤顺序的任务，所以可以在同一个Slot执行**。
  + 一个slot可以保存作业的整个管道的好处：
    + 如果有某个slot执行完了整个任务流程，那么其他任务就可以不用继续了，这样也省去了跨slot、跨TaskManager的通信损耗（降低了并行度）
    + 同时slot能够保存整个管道，使得整个任务执行健壮性更高，因为某些slot执行出异常也能有其他slot补上。
    + 有些slot分配到的子任务非CPU密集型，有些则CPU密集型，如果每个slot只完成自己的子任务，将出现某些slot太闲，某些slot过忙的现象。

  + *假设拆分的多个Source子任务放到同一个Slot，那么任务不能并行执行了=>因为多个相同步骤的子任务需要抢占的具体资源相同，比如抢占某个锁，这样就不能并行。*

+ Task Slot是静态的概念，是指TaskManager具有的并发执行能力，可以通过参数`taskmanager.numberOfTaskSlots`进行配置。

  *而并行度**parallelism**是动态概念，即**TaskManager**运行程序时实际使用的并发能力，可以通过参数`parallelism.default`进行配置。*

​	每个task slot表示TaskManager拥有资源的一个固定大小的子集。假如一个TaskManager有三个slot，那么它会将其管理的内存分成三份给各个slot。资源slot化意味着一个subtask将不需要跟来自其他job的subtask竞争被管理的内存，取而代之的是它将拥有一定数量的内存储备。

​	**需要注意的是，这里不会涉及到CPU的隔离，slot目前仅仅用来隔离task的受管理的内存**。

​	通过调整task slot的数量，允许用户定义subtask之间如何互相隔离。如果一个TaskManager一个slot，那将意味着每个task group运行在独立的JVM中（该JVM可能是通过一个特定的容器启动的），而一个TaskManager多个slot意味着更多的subtask可以共享同一个JVM。<u>而在同一个JVM进程中的task将共享TCP连接（基于多路复用）和心跳消息。它们也可能共享数据集和数据结构，因此这减少了每个task的负载。</u>

### 3.3.2 Slot和并行度

1. **一个特定算子的 子任务（subtask）的个数被称之为其并行度（parallelism）**，我们可以对单独的每个算子进行设置并行度，也可以直接用env设置全局的并行度，更可以在页面中去指定并行度。
2. 最后，由于并行度是实际Task Manager处理task 的能力，而一般情况下，**一个 stream 的并行度，可以认为就是其所有算子中最大的并行度**，则可以得出**在设置Slot时，在所有设置中的最大设置的并行度大小则就是所需要设置的Slot的数量。**（如果Slot分组，则需要为每组Slot并行度最大值的和）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215554488.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020052421520496.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215251380.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	假设一共有3个TaskManager，每一个TaskManager中的分配3个TaskSlot，也就是每个TaskManager可以接收3个task，一共9个TaskSlot，如果我们设置`parallelism.default=1`，即运行程序默认的并行度为1，9个TaskSlot只用了1个，有8个空闲，因此，设置合适的并行度才能提高效率。

​	*ps：上图最后一个因为是输出到文件，避免多个Slot（多线程）里的算子都输出到同一个文件互相覆盖等混乱问题，直接设置sink的并行度为1。*

### 3.3.3 程序和数据流（DataFlow）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524215944234.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ **所有的Flink程序都是由三部分组成的： Source 、Transformation 和 Sink。**

+ Source 负责读取数据源，Transformation 利用各种算子进行处理加工，Sink 负责输出

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220037630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

+ 在运行时，Flink上运行的程序会被映射成“逻辑数据流”（dataflows），它包含了这三部分

+ 每一个dataflow以一个或多个sources开始以一个或多个sinks结束。dataflow类似于任意的有向无环图（DAG）

+ 在大部分情况下，程序中的转换运算（transformations）跟dataflow中的算子（operator）是一一对应的关系

### 3.3.4 执行图（**ExecutionGraph**）

​	由Flink程序直接映射成的数据流图是StreamGraph，也被称为**逻辑流图**，因为它们表示的是计算逻辑的高级视图。为了执行一个流处理程序，Flink需要将**逻辑流图**转换为**物理数据流图**（也叫**执行图**），详细说明程序的执行方式。

+ Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

  + **StreamGraph**：是根据用户通过Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。

  + **JobGraph**：StreamGraph经过优化后生成了JobGraph，提交给JobManager 的数据结构。主要的优化为，将多个符合条件的节点chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。

  + **ExecutionGraph**：JobManager 根据JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。

  + 物理执行图：JobManager 根据ExecutionGraph 对Job 进行调度后，在各个TaskManager 上部署Task 后形成的“图”，并不是一个具体的数据结构。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220232635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

### 3.3.5 数据传输形式

+ 一个程序中，不同的算子可能具有不同的并行度

+ 算子之间传输数据的形式可以是 one-to-one (forwarding) 的模式也可以是redistributing 的模式，具体是哪一种形式，取决于算子的种类

  + **One-to-one**：stream维护着分区以及元素的顺序（比如source和map之间）。这意味着map 算子的子任务看到的元素的个数以及顺序跟 source 算子的子任务生产的元素的个数、顺序相同。**map、fliter、flatMap等算子都是one-to-one的对应关系**。

  + **Redistributing**：stream的分区会发生改变。每一个算子的子任务依据所选择的transformation发送数据到不同的目标任务。例如，keyBy 基于 hashCode 重分区、而 broadcast 和 rebalance 会随机重新分区，这些算子都会引起redistribute过程，而 redistribute 过程就类似于 Spark 中的 shuffle 过程。

### 3.3.6 任务链（OperatorChains）

​	Flink 采用了一种称为任务链的优化技术，可以在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或多个算子设为**相同的并行度**，并通过本地转发（local forward）的方式进行连接

+ **相同并行度**的 **one-to-one 操作**，Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的 subtask
  + 并行度相同、并且是 one-to-one 操作，两个条件缺一不可

​	**为什么需要并行度相同，因为若flatMap并行度为1，到了之后的map并行度为2，从flatMap到map的数据涉及到数据由于并行度map为2会往两个slot处理，数据会分散，所产生的元素个数和顺序发生的改变所以有2个单独的task，不能成为任务链**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200524220815415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

​	**如果前后任务逻辑上可以是OneToOne，且并行度一致，那么就能合并在一个Slot里**（并行度原本是多少就是多少，两者并行度一致）执行。

+ keyBy需要根据Hash值分配给不同slot执行，所以只能Hash，不能OneToOne。
+ 逻辑上可OneToOne但是并行度不同，那么就会Rebalance，轮询形式分配给下一个任务的多个slot。

---

+ **代码中如果`算子.disableChaining()`，能够强制当前算子的子任务不参与任务链的合并，即不和其他Slot资源合并，但是仍然可以保留“Slot共享”的特性**。

+ **如果`StreamExecutionEnvironment env.disableOperatorChaining()`则当前执行环境全局设置算子不参与"任务链的合并"。**

+ **如果`算子.startNewChain()`表示不管前面任务链合并与否，从当前算子往后重新计算任务链的合并。通常用于前面强制不要任务链合并，而当前往后又需要任务链合并的特殊场景。**

*ps：如果`算子.shuffle()`，能够强制算子之后重分区到不同slot执行下一个算子操作，逻辑上也实现了任务不参与任务链合并=>但是仅为“不参与任务链的合并”，这个明显不是最优解操作*

> [Flink slotSharingGroup disableChain startNewChain 用法案例](https://blog.csdn.net/qq_31866793/article/details/102786249)









# 四、Flink状态管理

![image-20220109235904442](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220109235904442.png)

1. 由一个任务维护，并且用来计算某个结果的所有数据，都属于这个任务的状态；
2. 可以认为任务状态就是一个本地变量，可以被任务的业务逻辑访问；
3. Flink 会进行状态管理，包括状态一致性、故障处理以及高效存储和访问，以便于开发人员可以专注于应用程序的逻辑；
4. 在Flink中，状态始终与特定算子相关联；
5. 为了使运行时的Flink了解算子的状态，算子需要预先注册其状态；



有两种类型的状态：

* 算子状态（Operator State）

  算子状态的作用范围限定为算子任务，不能跨任务访问

* 键控状态（Keyed State）

  根据输入数据流中定义的键（key）来维护和访问





## 4.1 算子状态（Operator State）

![image-20220110222303777](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220110222303777.png)

* 算子状态的作用范围限定为算子任务，同一并行任务所处理的所有数据都可以访问到相同的状态。
* 状态对于**同一任务**而言是共享的。（**不能跨slot**）
* 状态算子不能由相同或不同算子的另一个任务访问。

### 4.1.1 算子状态数据结构

* 列表状态(List state)
  * 将状态表示为一组数据的列表
* 联合列表状态(Union list state)
  * 也将状态表示未数据的列表。它与常规列表状态的区别在于，在发生故障时，或者从保存点(savepoint)启动应用程序时如何恢复
* 广播状态(Broadcast state)
  * 如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态



## 4.2 键控状态（Keyed State）

![image-20220113235444881](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220113235444881.png)

* 键控状态是根据输入数据流中定义的键（key）来维护和访问的 

* Flink 为**每个 key** 维护**一个状态实例**，并将具有相同键的所有数据，都分区到同一个算子任务中，这个任务会维护和处理这个 key 对应的状态 

* 当任务处理一条数据时，它会自动将状态的访问范围限定为当前数据的 key



### 4.2.1 键控状态数据结构

* 值状态(value state)
  * 将状态表示为单个的值
* 列表状态(List state)
  * 将状态表示为一组数据的列表
* 映射状态(Map state)
  * 将状态表示为一组key-value对
* **聚合状态(Reducing state & Aggregating State)**
  * 将状态表示为一个用于聚合操作的列表



### 4.2.2 键控状态的使用

![image-20220116014219016](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116014219016.png)







### 4.2.3 示例代码

```java
package com.mng.stateTest;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.RichFlatMapFunction;
import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.runtime.state.FunctionInitializationContext;
import org.apache.flink.runtime.state.FunctionSnapshotContext;
import org.apache.flink.streaming.api.checkpoint.CheckpointedFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class opstate {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> stringDS = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<SensorReading> sensorStream = stringDS.map(line -> {
            String[] strings = line.split(",");
            return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));

        });
        sensorStream.keyBy(sensorReading -> sensorReading.getId())
                .flatMap(new MyMapfunction(10D))
                .print();
        env.execute();
    }
}

class MyMapfunction extends RichFlatMapFunction<SensorReading, SensorReading> {
    private Double thresold;

    ValueState<Double> lasttemp;

    public MyMapfunction(Double thresold) {
        this.thresold = thresold;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        lasttemp = getRuntimeContext().getState(new ValueStateDescriptor<Double>("l_t",Double.class));

    }

    @Override
    public void close() throws Exception {
        super.close();
        lasttemp.clear();
    }

    @Override
    public void flatMap(SensorReading sensorReading, Collector<SensorReading> collector) throws Exception {
        Double lastTempV = lasttemp.value();
        Double nowTemp = sensorReading.getTemperaturre();
        lasttemp.update(nowTemp);
        if(lastTempV != null){
            if(Math.abs(lastTempV-nowTemp)>=this.thresold){
                collector.collect(sensorReading);
            }
        }
    }
}
```

输入：

```
nc -lp 7777
```

```
sensor_1,1547718199,35.8
sensor_1,1547718199,32.4
sensor_1,1547718199,42.4
sensor_10,1547718205,52.6   
sensor_10,1547718205,22.5
sensor_7,1547718202,6.7
sensor_7,1547718202,9.9
sensor_1,1547718207,36.3
sensor_7,1547718202,19.9
sensor_7,1547718202,30
```

输出：

![image-20220116014304430](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116014304430.png)



## 4.3 状态后端（State Backends）

### 4.3.1 概述

* 每传入一条数据，有状态的算子任务都会读取和更新状态
* 由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行 任务都会在本地维护其状态，以确保快速的状态访问
* 状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就 叫做状态后端（state backend） 
* 状态后端主要负责两件事：本地的状态管理，以及将检查点 （checkpoint）状态写入远程存储



### 4.3.2 选择一个状态后端

* MemoryStateBackend
  * 内存级的状态后端，会将键控状态作为内存中的对象进行管理，将它们存储在TaskManager的JVM堆上，而将checkpoint存储在JobManager的内存中
  * 特点：快速、低延迟，但不稳定
* FsStateBackend（默认）
  * 将checkpoint存到远程的持久化文件系统（FileSystem）上，而对于本地状态，跟MemoryStateBackend一样，也会存在TaskManager的JVM堆上
  * 同时拥有内存级的本地访问速度，和更好的容错保证
* RocksDBStateBackend
  * 将所有状态序列化后，存入本地的RocksDB中存储



### 4.3.3 配置文件

flink-conf.yaml

![image-20220116015145134](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116015145134.png)



### 4.3.4 示例代码

![image-20220116015236952](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116015236952.png)





# 五、容错机制

## 5.1一致性检查点（checkpoint）

![image-20220116203001731](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116203001731.png)

* Flink 故障恢复机制的核心，就是应用状态的一致性检查点

* 有状态流应用的一致检查点，其实就是所有任务的状态，在某个时间点的一份拷贝（一份快照）；这个时间点，应该是所有任务都恰好处理完一个相同的输 入数据的时候



## 5.2 从检查点恢复状态

* 在执行流应用程序期间，Flink 会定期保存状态的一致检查点

* 如果发生故障， Flink 将会使用最近的检查点来一致恢复应用程序的状态，并 重新启动处理流程



（**如下图所示，7这个数据被source读到了，准备传给奇数流时，奇数流宕机了，数据传输发生中断**）

![image-20220116233538105](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116233538105.png)

* 遇到故障之后，第一步就是重启应用

![image-20220116233611314](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116233611314.png)

* 第二步是从 checkpoint 中读取状态，将状态重置

从检查点重新启动应用程序后，其内部状态与检查点完成时的状态完全相同

![image-20220116233701994](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116233701994.png)

* 第三步：开始消费并处理检查点到发生故障之间的所有数据 

这种检查点的保存和恢复机制可以为应用程序状态提供“精确一次” （exactly-once）的一致性，因为所有算子都会保存检查点并恢复其所有状态，这样一来所有的输入流就都会被重置到检查点完成时的位置

![image-20220116233642658](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220116233642658.png)

*（这里要求source源也能记录状态，回退到读取数据7的状态，kafka有相应的偏移指针能完成该操作）*





## 5.3 Flink检查点算法

### 概述

**checkpoint和Watermark一样，都会以广播的形式告诉所有下游。**

---

+ 一种简单的想法

  暂停应用，保存状态到检查点，再重新恢复应用（当然Flink 不是采用这种简单粗暴的方式）

+ Flink的改进实现

  + 基于Chandy-Lamport算法的分布式快照
  + 将检查点的保存和数据处理分离开，不暂停整个应用

  （就是每个任务单独拍摄自己的快照到内存，之后再到jobManager整合）

---

+ 检查点分界线（Checkpoint Barrier）
  + Flink的检查点算法用到了一种称为分界线（barrier）的特殊数据形式，用来把一条流上数据按照不同的检查点分开
  + **分界线之前到来的数据导致的状态更改，都会被包含在当前分界线所属的检查点中；而基于分界线之后的数据导致的所有更改，就会被包含在之后的检查点中**

### 具体讲解

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529224034243.png)

+ 现在是一个有两个输入流的应用程序，用并行的两个 Source 任务来读取

+ 两条自然数数据流，蓝色数据流已经输出完`蓝3`了，黄色数据流输出完`黄4`了

+ 在Souce端 Source1 接收到了数据`蓝3` 正在往下游发向一个数据`蓝2 和 蓝3`； Source2 接受到了数据`黄4`，且往下游发送数据`黄4`

+ 偶数流已经处理完`黄2` 所以后面显示为2， 奇数流处理完`蓝1 和 黄1 黄3` 所以为5，并分别往下游发送每次聚合后的结果给Sink

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529224517502.png)

+ **JobManager 会向每个 source 任务发送一条带有新检查点 ID 的消息**，通过这种方式来启动检查点

  *（这个带有新检查点ID的东西为**barrier**，由图中三角型表示，数值2只是ID）*

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529224705177.png)

+ 数据源将它们的状态写入检查点，并发出一个检查点barrier
+ 状态后端在状态存入检查点之后，会返回通知给source任务，source任务就会向JobManager确认检查点完成

​	*上图，在Source端接受到barrier后，将自己此身的3 和 4 的数据的状态写入检查点，且向JobManager发送checkpoint成功的消息，然后向下游分别发出一个检查点 barrier*

​	*可以看出在Source接受barrier时，数据流也在不断的处理，不会进行中断*

​	*此时的偶数流已经处理完`蓝2`变成了4，但是还没处理到`黄4`，只是下游sink发送了一个数据4，而奇数流已经处理完`蓝3`变成了8（黄1+蓝1+黄3+蓝3），并向下游sink发送了8*

​	*此时检查点barrier都还未到Sum_odd奇数流和Sum_even偶数流*

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529225235834.png)

+ **分界线对齐：barrier向下游传递，sum任务会等待所有输入分区的barrier到达**
+ **对于barrier已经达到的分区，继续到达的数据会被缓存**
+ **而barrier尚未到达的分区，数据会被正常处理**

​	*此时蓝色流的barrier先一步抵达了偶数流，黄色的barrier还未到，但是因为数据的不中断一直处理，此时的先到的蓝色的barrier会将此时的偶数流的数据4进行缓存处理，流接着处理接下来的数据等待着黄色的barrier的到来，而黄色barrier之前的数据将会对缓存的数据相加*

​	*这次处理的总结：**分界线对齐**：**barrier 向下游传递，sum 任务会等待所有输入分区的 barrier 到达，对于barrier已经到达的分区，继续到达的数据会被缓存。而barrier尚未到达的分区，数据会被正常处理***

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529225656902.png)

+ **当收到所有输入分区的 barrier 时，任务就将其状态保存到状态后端的检查点中，然后将 barrier 继续向下游转发**

​	*当蓝色的barrier和黄色的barrier(所有分区的)都到达后，进行状态保存到远程仓库，**然后对JobManager发送消息，说自己的检查点保存完毕了***

​	*此时的偶数流和奇数流都为8*

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200529230413317.png)

+ 向下游转发检查点 barrier 后，任务继续正常的数据处理

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/2020052923042436.png)

+ **Sink 任务向 JobManager 确认状态保存到 checkpoint 完毕**
+ **当所有任务都确认已成功将状态保存到检查点时，检查点就真正完成了**

## 5.4 保存点(Savepoints)

**CheckPoint为自动保存，SavePoint为手动保存**

+ Flink还提供了可以自定义的镜像保存功能，就是保存点（save points）
+ 原则上，创建保存点使用的算法与检查点完全相同，因此保存点可以认为就是具有一些额外元数据的检查点
+ Flink不会自动创建保存点，因此用户（或者外部调度程序）必须明确地触发创建操作
+ 保存点是一个强大的功能。除了故障恢复外，保存点可以用于：有计划的手动备份、更新应用程序、版本迁移、暂停和重启程序，等等

## 5.5 检查点和重启策略配置

+ java样例代码

  ```java
  package com.mng.check;
  
  import com.mng.dto.SensorReading;
  import org.apache.flink.api.common.restartstrategy.RestartStrategies;
  import org.apache.flink.api.common.time.Time;
  import org.apache.flink.contrib.streaming.state.RocksDBStateBackend;
  import org.apache.flink.runtime.state.filesystem.FsStateBackend;
  import org.apache.flink.runtime.state.memory.MemoryStateBackend;
  import org.apache.flink.streaming.api.CheckpointingMode;
  import org.apache.flink.streaming.api.datastream.DataStream;
  import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
  
  
  public class CheckTest {
    public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
  
      // 1. 状态后端配置
      env.setStateBackend(new MemoryStateBackend());
      env.setStateBackend(new FsStateBackend(""));
      // 这个需要另外导入依赖
      env.setStateBackend(new RocksDBStateBackend(""));
  
      // 2. 检查点配置 (每300ms让jobManager进行一次checkpoint检查)
      env.enableCheckpointing(300);
  
      // 高级选项
      env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
      //Checkpoint的处理超时时间
      env.getCheckpointConfig().setCheckpointTimeout(60000L);
      // 最大允许同时处理几个Checkpoint(比如上一个处理到一半，这里又收到一个待处理的Checkpoint事件)
      env.getCheckpointConfig().setMaxConcurrentCheckpoints(2);
      // 与上面setMaxConcurrentCheckpoints(2) 冲突，这个时间间隔是 当前checkpoint的处理完成时间与接收最新一个checkpoint之间的时间间隔
      env.getCheckpointConfig().setMinPauseBetweenCheckpoints(100L);
      // 如果同时开启了savepoint且有更新的备份，是否倾向于使用更老的自动备份checkpoint来恢复，默认false
      env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
      // 最多能容忍几次checkpoint处理失败（默认0，即checkpoint处理失败，就当作程序执行异常）
      env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0);
  
      // 3. 重启策略配置
      // 固定延迟重启(最多尝试3次，每次间隔10s)
      env.setRestartStrategy(RestartStrategies.fixedDelayRestart(3, 10000L));
      // 失败率重启(在10分钟内最多尝试3次，每次至少间隔1分钟)
      env.setRestartStrategy(RestartStrategies.failureRateRestart(3, Time.minutes(10), Time.minutes(1)));
  
      // socket文本流
      DataStream<String> inputStream = env.socketTextStream("localhost", 7777);
  
      // 转换成SensorReading类型
      DataStream<SensorReading> dataStream = inputStream.map(line -> {
        String[] fields = line.split(",");
        return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
      });
  
      dataStream.print();
      env.execute();
    }
  }
  ```



# 六、状态一致性

## 6.1 概述

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530181851687.png)

+ 有状态的流处理，内部每个算子任务都可以有自己的状态

+ 对于流处理器内部来说，所谓的状态一致性，其实就是我们所说的计算结果要保证准确。

+ 一条数据不应该丢失，也不应该重复计算

+ 在遇到故障时可以恢复状态，恢复以后的重新计算，结果应该也是完全正确的。

## 6.2 分类

​	**Flink的一个重大价值在于，它既保证了exactly-once，也具有低延迟和高吞吐的处理能力。**

1. AT-MOST-ONCE（最多一次）
   当任务故障时，最简单的做法是什么都不干，既不恢复丢失的状态，也不重播丢失的数据。At-most-once 语义的含义是最多处理一次事件。

   *这其实是没有正确性保障的委婉说法——故障发生之后，计算结果可能丢失。类似的比如网络协议的udp。*

2. AT-LEAST-ONCE（至少一次）
   在大多数的真实应用场景，我们希望不丢失事件。这种类型的保障称为 at-least-once，意思是所有的事件都得到了处理，而一些事件还可能被处理多次。

   *这表示计数结果可能大于正确值，但绝不会小于正确值。也就是说，计数程序在发生故障后可能多算，但是绝不会少算。*

3. EXACTLY-ONCE（精确一次）
   **恰好处理一次是最严格的保证，也是最难实现的。恰好处理一次语义不仅仅意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。**

   *这指的是系统保证在发生故障后得到的计数结果与正确值一致。*

## 6.3 一致性检查点(Checkpoints)

+ Flink使用了一种轻量级快照机制——检查点（checkpoint）来保证exactly-once语义
+ 有状态流应用的一致检查点，其实就是：所有任务的状态，在某个时间点的一份备份（一份快照）。而这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时间。
+ 应用状态的一致检查点，是Flink故障恢复机制的核心

### 端到端(end-to-end)状态一致性

+ 目前我们看到的一致性保证都是由流处理器实现的，也就是说都是在Flink流处理器内部保证的；而在真实应用中，流处理应用除了流处理器以外还包含了数据源（例如Kafka）和输出到持久化系统

+ 端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终；每一个组件都保证了它自己的一致性
+ **整个端到端的一致性级别取决于所有组件中一致性最弱的组件**

### 端到端 exactly-once

+ 内部保证——checkpoint
+ source端——可重设数据的读取位置
+ sink端——从故障恢复时，数据不会重复写入外部系统
  + 幂等写入
  + 事务写入

#### 幂等写入

+ 所谓幂等操作，是说一个操作，可以重复执行很多次，但只导致一次结果更改，也就是说，后面再重复执行就不起作用了。

  *（中间可能会存在不正确的情况，只能保证最后结果正确。比如5=>10=>15=>5=>10=>15，虽然最后是恢复到了15，但是中间有个恢复的过程，如果这个过程能够被读取，就会出问题。）*

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/2020053019091138.png)

#### 事务写入

+ 事务（Transaction）

  + 应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤销
  + 具有原子性：一个事务中的一系列的操作要么全部成功，要么一个都不做

+ 实现思想

  **构建的事务对应着checkpoint，等到checkpoint真正完成的时候，才把所有对应的结果写入sink系统中**。

+ 实现方式

  + 预习日志
  + 两阶段提交

##### 预写日志(Write-Ahead-Log，WAL)

+ 把结果数据先当成状态保存，然后在收到checkpoint完成的通知时，一次性写入sink系统
+ 简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么sink系统，都能用这种方式一批搞定
+ DataStream API提供了一个模版类：GenericWriteAheadSink，来实现这种事务性sink

##### 两阶段提交(Two-Phase-Commit，2PC)

+ 对于每个checkpoint，sink任务会启动一个事务，并将接下来所有接收到的数据添加到事务里
+ 然后将这些数据写入外部sink系统，但不提交它们——这时只是"预提交"
+ **这种方式真正实现了exactly-once，它需要一个提供事务支持的外部sink系统**。Flink提供了TwoPhaseCommitSinkFunction接口

### 不同Source和Sink的一致性保证

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530194322578.png)



## 6.4 Flink+Kafka 端到端状态一致性的保证

> [Flink-状态一致性 | 状态一致性分类 | 端到端状态一致性 | 幂等写入 | 事务写入 | WAL | 2PC](https://blog.csdn.net/qq_40180229/article/details/106445029)

+ 内部——利用checkpoint机制，把状态存盘，发生故障的时候可以恢复，保证内部的状态一致性
+ source——kafka consumer作为source，可以将偏移量保存下来，如果后续任务出现了故障，恢复的时候可以由连接器重制偏移量，重新消费数据，保证一致性
+ sink——kafka producer作为sink，采用两阶段提交sink，需要实现一个TwoPhaseCommitSinkFunction

### Exactly-once 两阶段提交

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530194434435.png)

+ JobManager 协调各个 TaskManager 进行 checkpoint 存储
+ checkpoint保存在 StateBackend中，默认StateBackend是内存级的，也可以改为文件级的进行持久化保存

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530194627287.png)

+ 当 checkpoint 启动时，JobManager 会将检查点分界线（barrier）注入数据流
+ barrier会在算子间传递下去

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530194657186.png)

+ 每个算子会对当前的状态做个快照，保存到状态后端
+ checkpoint 机制可以保证内部的状态一致性

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200530194835593.png)

+ 每个内部的 transform 任务遇到 barrier 时，都会把状态存到 checkpoint 里

+ sink 任务首先把数据写入外部 kafka，**这些数据都属于预提交的事务**；**遇到 barrier 时，把状态保存到状态后端，并开启新的预提交事务**

  *(barrier之前的数据还是在之前的事务中没关闭事务，遇到barrier后的数据另外新开启一个事务)*

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/2020053019485194.png)

+ 当所有算子任务的快照完成，也就是这次的 checkpoint 完成时，JobManager 会向所有任务发通知，确认这次 checkpoint 完成
+ sink 任务收到确认通知，正式提交之前的事务，kafka 中未确认数据改为“已确认”

### Exactly-once 两阶段提交步骤总结

1. 第一条数据来了之后，开启一个 kafka 的事务（transaction），正常写入 kafka 分区日志但标记为未提交，这就是“预提交”
2. jobmanager 触发 checkpoint 操作，barrier 从 source 开始向下传递，遇到 barrier 的算子将状态存入状态后端，并通知 jobmanager
3. sink 连接器收到 barrier，保存当前状态，存入 checkpoint，通知 jobmanager，并开启下一阶段的事务，用于提交下个检查点的数据
4. jobmanager 收到所有任务的通知，发出确认信息，表示 checkpoint 完成
5. sink 任务收到 jobmanager 的确认信息，正式提交这段时间的数据
6. 外部kafka关闭事务，提交的数据可以正常消费了。







# 七、时间特性(Time Attributes)

## 7.1 概述

+ 基于时间的操作（比如 Table API 和 SQL 中窗口操作），需要定义相关的时间语义和时间数据来源的信息
+ Table 可以提供一个逻辑上的时间字段，用于在表处理程序中，指示时间和访问相应的时间戳
+ **时间属性，可以是每个表schema的一部分。一旦定义了时间属性，它就可以作为一个字段引用，并且可以在基于时间的操作中使用**
+ 时间属性的行为类似于常规时间戳，可以访问，并且进行计算

## 7.2 定义处理时间(Processing Time)

+ 处理时间语义下，允许表处理程序根据机器的本地时间生成结果。它是时间的最简单概念。它既不需要提取时间戳，也不需要生成 watermark

### 由DataStream转换成表时指定

+ 在定义 Table Schema 期间，可以使用`.proctime`，指定字段名定义处理时间字段

+ **这个proctime属性只能通过附加逻辑字段，来扩展物理schema。因此，只能在schema定义的末尾定义它**

  ```java
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, temperature, pt.proctime");
  ```

### 定义Table Schema时指定

```java
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp",DataTypes.BIGINT())
            .field("temperature",DataTypes.DOUBLE())
            .field("pt",DataTypes.TIMESTAMP(3))
            .proctime()
           )
```

### 创建表的DDL中定义

```java
String sinkDDL = 
  "create table dataTable (" +
  " id varchar(20) not null, " +
  " ts bigint, " +
  " temperature double, " +
  " pt AS PROCTIME() " +
  " ) with (" +
  " 'connector.type' = 'filesystem', " +
  " 'connector.path' = '/sensor.txt', " +
  " 'format.type' = 'csv')";
tableEnv.sqlUpdate(sinkDDL);
```

### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 4. 将流转换成表，定义时间特性
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");

    dataTable.printSchema();
    tableEnv.toAppendStream(dataTable, Row.class).print();

    env.execute();
  }
}
```

输出如下：

```shell
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- pt: TIMESTAMP(3) *PROCTIME*

sensor_1,1547718199,35.8,2021-02-03T16:50:58.048
sensor_6,1547718201,15.4,2021-02-03T16:50:58.048
sensor_7,1547718202,6.7,2021-02-03T16:50:58.050
sensor_10,1547718205,38.1,2021-02-03T16:50:58.050
```

## 7.3 定义事件事件(Event Time)

+ 事件时间语义，允许表处理程序根据每个记录中包含的时间生成结果。这样即使在有乱序事件或者延迟事件时，也可以获得正确的结果。
+ **为了处理无序事件，并区分流中的准时和迟到事件；Flink需要从事件数据中，提取时间戳，并用来推送事件时间的进展**
+ 定义事件事件，同样有三种方法：
  + 由DataStream转换成表时指定
  + 定义Table Schema时指定
  + 在创建表的DDL中定义

### 由DataStream转换成表时指定

+ 由DataStream转换成表时指定（推荐）

+ 在DataStream转换成Table，使用`.rowtime`可以定义事件事件属性

  ```java
  // 将DataStream转换为Table，并指定时间字段
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, timestamp.rowtime, temperature");
  // 或者，直接追加时间字段
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                            "id, temperature, timestamp, rt.rowtime");
  ```

### 定义Table Schema时指定

```java
.withSchema(new Schema()
            .field("id", DataTypes.STRING())
            .field("timestamp",DataTypes.BIGINT())
            .rowtime(
              new Rowtime()
              .timestampsFromField("timestamp") // 从字段中提取时间戳
              .watermarksPeriodicBounded(1000) // watermark延迟1秒
            )
            .field("temperature",DataTypes.DOUBLE())
           )
```

### 创建表的DDL中定义

```java
String sinkDDL = 
  "create table dataTable (" +
  " id varchar(20) not null, " +
  " ts bigint, " +
  " temperature double, " +
  " rt AS TO_TIMESTAMP( FROM_UNIXTIME(ts) ), " +
  " watermark for rt as rt - interval '1' second"
  " ) with (" +
  " 'connector.type' = 'filesystem', " +
  " 'connector.path' = '/sensor.txt', " +
  " 'format.type' = 'csv')";
tableEnv.sqlUpdate(sinkDDL);
```

### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    dataTable.printSchema();

    tableEnv.toAppendStream(dataTable,Row.class).print();

    env.execute();
  }
}
```

输出如下：

*注：这里最后一列rt里显示的是EventTime，而不是Processing Time*

```shell
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- rt: TIMESTAMP(3) *ROWTIME*

sensor_1,1547718199,35.8,2019-01-17T09:43:19
sensor_6,1547718201,15.4,2019-01-17T09:43:21
sensor_7,1547718202,6.7,2019-01-17T09:43:22
sensor_10,1547718205,38.1,2019-01-17T09:43:25
sensor_1,1547718207,36.3,2019-01-17T09:43:27
```

## 7.4 窗口

+ 时间语义，要配合窗口操作才能发挥作用。
+ 在Table API和SQL中，主要有两种窗口
  + Group Windows（分组窗口）
    + **根据时间戳或行计数间隔，将行聚合到有限的组（Group）中，并对每个组的数据执行一次聚合函数**
  + Over Windows
    + 针对每个输入行，计算相邻行范围内的聚合

### 7.4.1 Group Windows

+ Group Windows 是使用 window（w:GroupWindow）子句定义的，并且**必须由as子句指定一个别名**。

+ 为了按窗口对表进行分组，窗口的别名必须在 group by 子句中，像常规的分组字段一样引用

  ```scala
  Table table = input
  .window([w:GroupWindow] as "w") // 定义窗口，别名为w
  .groupBy("w, a") // 按照字段 a和窗口 w分组
  .select("a,b.sum"); // 聚合
  ```

+ Table API 提供了一组具有特定语义的预定义 Window 类，这些类会被转换为底层 DataStream 或 DataSet 的窗口操作

+ 分组窗口分为三种：

  + 滚动窗口
  + 滑动窗口
  + 会话窗口

#### 滚动窗口(Tumbling windows)

+ 滚动窗口（Tumbling windows）要用Tumble类来定义

```java
// Tumbling Event-time Window（事件时间字段rowtime）
.window(Tumble.over("10.minutes").on("rowtime").as("w"))

// Tumbling Processing-time Window（处理时间字段proctime）
.window(Tumble.over("10.minutes").on("proctime").as("w"))

// Tumbling Row-count Window (类似于计数窗口，按处理时间排序，10行一组)
.window(Tumble.over("10.rows").on("proctime").as("w"))
```

+ over：定义窗口长度
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

#### 滑动窗口(Sliding windows)

+ 滑动窗口（Sliding windows）要用Slide类来定义

```java
// Sliding Event-time Window
.window(Slide.over("10.minutes").every("5.minutes").on("rowtime").as("w"))

// Sliding Processing-time window 
.window(Slide.over("10.minutes").every("5.minutes").on("proctime").as("w"))

// Sliding Row-count window
.window(Slide.over("10.rows").every("5.rows").on("proctime").as("w"))
```

+ over：定义窗口长度
+ every：定义滑动步长
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

#### 会话窗口(Session windows)

+ 会话窗口（Session windows）要用Session类来定义

```java
// Session Event-time Window
.window(Session.withGap("10.minutes").on("rowtime").as("w"))

// Session Processing-time Window 
.window(Session.withGap("10.minutes").on("proctime").as("w"))
```

+ withGap：会话时间间隔
+ on：用来分组（按时间间隔）或者排序（按行数）的时间字段
+ as：别名，必须出现在后面的groupBy中

### 7.4.2 SQL中的Group Windows

Group Windows定义在SQL查询的Group By子句中

+ TUMBLE(time_attr, interval)
  + 定义一个滚动窗口，每一个参数是时间字段，第二个参数是窗口长度
+ HOP(time_attr，interval，interval)
  + 定义一个滑动窗口，第一个参数是时间字段，**第二个参数是窗口滑动步长，第三个是窗口长度**
+ SESSION(time_attr，interval)
  + 定义一个绘画窗口，第一个参数是时间字段，第二个参数是窗口间隔

#### 测试代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    //        dataTable.printSchema();
    //
    //        tableEnv.toAppendStream(dataTable,Row.class).print();

    tableEnv.createTemporaryView("sensor", dataTable);

    // 5. 窗口操作
    // 5.1 Group Window
    // table API
    Table resultTable = dataTable.window(Tumble.over("10.seconds").on("rt").as("tw"))
      .groupBy("id, tw")
      .select("id, id.count, temp.avg, tw.end");

    // SQL
    Table resultSqlTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp, tumble_end(rt, interval '10' second) " +
                                             "from sensor group by id, tumble(rt, interval '10' second)");

    dataTable.printSchema();
    tableEnv.toAppendStream(resultTable, Row.class).print("result");
    tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }
}
```

输出：

```java
root
 |-- id: STRING
 |-- ts: BIGINT
 |-- temp: DOUBLE
 |-- rt: TIMESTAMP(3) *ROWTIME*

result> sensor_1,1,35.8,2019-01-17T09:43:20
result> sensor_6,1,15.4,2019-01-17T09:43:30
result> sensor_1,2,34.55,2019-01-17T09:43:30
result> sensor_10,1,38.1,2019-01-17T09:43:30
result> sensor_7,1,6.7,2019-01-17T09:43:30
sql> (true,sensor_1,1,35.8,2019-01-17T09:43:20)
result> sensor_1,1,37.1,2019-01-17T09:43:40
sql> (true,sensor_6,1,15.4,2019-01-17T09:43:30)
sql> (true,sensor_1,2,34.55,2019-01-17T09:43:30)
sql> (true,sensor_10,1,38.1,2019-01-17T09:43:30)
```

### 7.4.3 Over Windows

> [SQL中over的用法](https://blog.csdn.net/liuyuehui110/article/details/42736667)
>
> [sql over的作用及用法](https://www.cnblogs.com/xiayang/articles/1886372.html)

+ **Over window 聚合是标准 SQL 中已有的（over 子句），可以在查询的 SELECT 子句中定义**

+ Over window 聚合，会**针对每个输入行**，计算相邻行范围内的聚合

+ Over windows 使用 window（w:overwindows*）子句定义，并在 select（）方法中通过**别名**来引用

  ```scala
  Table table = input
  .window([w: OverWindow] as "w")
  .select("a, b.sum over w, c.min over w");
  ```

+ Table API 提供了 Over 类，来配置 Over 窗口的属性

#### 无界Over Windows

+ 可以在事件时间或处理时间，以及指定为时间间隔、或行计数的范围内，定义 Over windows
+ 无界的 over window 是使用常量指定的

```java
// 无界的事件时间over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_RANGE).as("w"))

//无界的处理时间over window (时间字段"proctime")
.window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_RANGE).as("w"))

// 无界的事件时间Row-count over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("rowtime").preceding(UNBOUNDED_ROW).as("w"))

//无界的处理时间Row-count over window (时间字段 "rowtime")
.window(Over.partitionBy("a").orderBy("proctime").preceding(UNBOUNDED_ROW).as("w"))
```

*partitionBy是可选项*

#### 有界Over Windows

+ 有界的over window是用间隔的大小指定的

```java
// 有界的事件时间over window (时间字段 "rowtime"，之前1分钟)
.window(Over.partitionBy("a").orderBy("rowtime").preceding("1.minutes").as("w"))

// 有界的处理时间over window (时间字段 "rowtime"，之前1分钟)
.window(Over.partitionBy("a").orderBy("porctime").preceding("1.minutes").as("w"))

// 有界的事件时间Row-count over window (时间字段 "rowtime"，之前10行)
.window(Over.partitionBy("a").orderBy("rowtime").preceding("10.rows").as("w"))

// 有界的处理时间Row-count over window (时间字段 "rowtime"，之前10行)
.window(Over.partitionBy("a").orderBy("proctime").preceding("10.rows").as("w"))
```

### 7.4.4 SQL中的Over Windows

+ 用 Over 做窗口聚合时，所有聚合必须在同一窗口上定义，也就是说必须是相同的分区、排序和范围
+ 目前仅支持在当前行范围之前的窗口
+ ORDER BY 必须在单一的时间属性上指定

```sql
SELECT COUNT(amount) OVER (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
FROM Orders

// 也可以做多个聚合
SELECT COUNT(amount) OVER w, SUM(amount) OVER w
FROM Orders
WINDOW w AS (
  PARTITION BY user
  ORDER BY proctime
  ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

#### 测试代码

java代码

```java
package apitest.tableapi;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.table.api.Over;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.Tumble;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

public class TableTest5_TimeAndWindow {
  public static void main(String[] args) throws Exception {
    // 1. 创建环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 2. 读入文件数据，得到DataStream
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 3. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
        @Override
        public long extractTimestamp(SensorReading element) {
          return element.getTimestamp() * 1000L;
        }
      });

    // 4. 将流转换成表，定义时间特性
    //        Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, pt.proctime");
    Table dataTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp, rt.rowtime");

    //        dataTable.printSchema();
    //
    //        tableEnv.toAppendStream(dataTable,Row.class).print();

    tableEnv.createTemporaryView("sensor", dataTable);

    // 5. 窗口操作
    // 5.1 Group Window
    // table API
    Table resultTable = dataTable.window(Tumble.over("10.seconds").on("rt").as("tw"))
      .groupBy("id, tw")
      .select("id, id.count, temp.avg, tw.end");

    // SQL
    Table resultSqlTable = tableEnv.sqlQuery("select id, count(id) as cnt, avg(temp) as avgTemp, tumble_end(rt, interval '10' second) " +
                                             "from sensor group by id, tumble(rt, interval '10' second)");

    // 5.2 Over Window
    // table API
    Table overResult = dataTable.window(Over.partitionBy("id").orderBy("rt").preceding("2.rows").as("ow"))
      .select("id, rt, id.count over ow, temp.avg over ow");

    // SQL
    Table overSqlResult = tableEnv.sqlQuery("select id, rt, count(id) over ow, avg(temp) over ow " +
                                            " from sensor " +
                                            " window ow as (partition by id order by rt rows between 2 preceding and current row)");

    //        dataTable.printSchema();
    //        tableEnv.toAppendStream(resultTable, Row.class).print("result");
    //        tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");
    tableEnv.toAppendStream(overResult, Row.class).print("result");
    tableEnv.toRetractStream(overSqlResult, Row.class).print("sql");

    env.execute();
  }
}
```

输出:

*因为`partition by id order by rt rows between 2 preceding and current row`，所以最后2次关于`sensor_1`的输出的`count(id)`都是3,但是计算出来的平均值不一样。（前者计算倒数3条sensor_1的数据，后者计算最后最新的3条sensor_1数据的平均值）*

```shell
result> sensor_1,2019-01-17T09:43:19,1,35.8
sql> (true,sensor_1,2019-01-17T09:43:19,1,35.8)
result> sensor_6,2019-01-17T09:43:21,1,15.4
sql> (true,sensor_6,2019-01-17T09:43:21,1,15.4)
result> sensor_7,2019-01-17T09:43:22,1,6.7
sql> (true,sensor_7,2019-01-17T09:43:22,1,6.7)
result> sensor_10,2019-01-17T09:43:25,1,38.1
sql> (true,sensor_10,2019-01-17T09:43:25,1,38.1)
result> sensor_1,2019-01-17T09:43:27,2,36.05
sql> (true,sensor_1,2019-01-17T09:43:27,2,36.05)
sql> (true,sensor_1,2019-01-17T09:43:29,3,34.96666666666666)
result> sensor_1,2019-01-17T09:43:29,3,34.96666666666666
result> sensor_1,2019-01-17T09:43:32,3,35.4
sql> (true,sensor_1,2019-01-17T09:43:32,3,35.4)
```

# 八、 函数(Functions)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200601214323293.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQwMTgwMjI5,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/2020060121433777.png)

## 8.1 用户自定义函数(UDF)

+ 用户定义函数（User-defined Functions，UDF）是一个重要的特性，它们显著地扩展了查询的表达能力

  *一些系统内置函数无法解决的需求，我们可以用UDF来自定义实现*

+ **在大多数情况下，用户定义的函数必须先注册，然后才能在查询中使用**

+ 函数通过调用 `registerFunction()` 方法在 TableEnvironment 中注册。当用户定义的函数被注册时，它被插入到 TableEnvironment 的函数目录中，这样Table API 或 SQL 解析器就可以识别并正确地解释它

### 8.1.1 标量函数(Scalar Functions)

**Scalar Funcion类似于map，一对一**

**Table Function类似flatMap，一对多**

---

+ 用户定义的标量函数，可以将0、1或多个标量值，映射到新的标量值

+ 为了定义标量函数，必须在 org.apache.flink.table.functions 中扩展基类Scalar Function，并实现（一个或多个）求值（eval）方法

+ **标量函数的行为由求值方法决定，求值方法必须public公开声明并命名为 eval**

  ```java
  public static class HashCode extends ScalarFunction {
  
    private int factor = 13;
  
    public HashCode(int factor) {
      this.factor = factor;
    }
  
    public int eval(String id) {
      return id.hashCode() * 13;
    }
  }
  ```

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.ScalarFunction;
import org.apache.flink.types.Row;

public class UdfTest1_ScalarFunction {
  public static void main(String[] args) throws Exception {
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 并行度设置为1
    env.setParallelism(1);

    // 创建Table执行环境
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换为表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id,timestamp as ts,temperature");

    // 4. 自定义标量函数，实现求id的hash值
    HashCode hashCode = new HashCode(23);
    // 注册UDF
    tableEnv.registerFunction("hashCode", hashCode);

    // 4.1 table API
    Table resultTable = sensorTable.select("id, ts, hashCode(id)");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, ts, hashCode(id) from sensor");

    // 打印输出
    tableEnv.toAppendStream(resultTable, Row.class).print();
    tableEnv.toAppendStream(resultSqlTable, Row.class).print();

    env.execute();
  }

  public static class HashCode extends ScalarFunction {

    private int factor = 13;

    public HashCode(int factor) {
      this.factor = factor;
    }

    public int eval(String id) {
      return id.hashCode() * 13;
    }
  }
}
```

输出结果

```shell
sensor_1,1547718199,-772373508
sensor_1,1547718199,-772373508
sensor_6,1547718201,-772373443
sensor_6,1547718201,-772373443
sensor_7,1547718202,-772373430
sensor_7,1547718202,-772373430
sensor_10,1547718205,1826225652
sensor_10,1547718205,1826225652
sensor_1,1547718207,-772373508
sensor_1,1547718207,-772373508
sensor_1,1547718209,-772373508
sensor_1,1547718209,-772373508
sensor_1,1547718212,-772373508
sensor_1,1547718212,-772373508
```

### 8.1.2 表函数(Table Fcuntions)

**Scalar Funcion类似于map，一对一**

**Table Function类似flatMap，一对多**

---

+ 用户定义的表函数，也可以将0、1或多个标量值作为输入参数；**与标量函数不同的是，它可以返回任意数量的行作为输出，而不是单个值**

+ 为了定义一个表函数，必须扩展 org.apache.flink.table.functions 中的基类 TableFunction 并实现（一个或多个）求值方法

+ **表函数的行为由其求值方法决定，求值方法必须是 public 的，并命名为 eval**

  ```java
  public static class Split extends TableFunction<Tuple2<String, Integer>> {
  
    // 定义属性，分隔符
    private String separator = ",";
  
    public Split(String separator) {
      this.separator = separator;
    }
  
    public void eval(String str) {
      for (String s : str.split(separator)) {
        collect(new Tuple2<>(s, s.length()));
      }
    }
  }
  ```

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.TableFunction;
import org.apache.flink.types.Row;

public class UdfTest2_TableFunction {
  public static void main(String[] args) throws Exception {
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 并行度设置为1
    env.setParallelism(1);

    // 创建Table执行环境
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换为表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id,timestamp as ts,temperature");

    // 4. 自定义表函数，实现将id拆分，并输出（word, length）
    Split split = new Split("_");
    // 需要在环境中注册UDF
    tableEnv.registerFunction("split", split);

    // 4.1 table API
    Table resultTable = sensorTable
      .joinLateral("split(id) as (word, length)")
      .select("id, ts, word, length");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, ts, word, length " +
                                             " from sensor, lateral table(split(id)) as splitid(word, length)");

    // 打印输出
    tableEnv.toAppendStream(resultTable, Row.class).print("result");
    tableEnv.toAppendStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }

  // 实现自定义 Table Function
  public static class Split extends TableFunction<Tuple2<String, Integer>> {

    // 定义属性，分隔符
    private String separator = ",";

    public Split(String separator) {
      this.separator = separator;
    }

    public void eval(String str) {
      for (String s : str.split(separator)) {
        collect(new Tuple2<>(s, s.length()));
      }
    }
  }
}
```

输出结果

```shell
result> sensor_1,1547718199,sensor,6
result> sensor_1,1547718199,1,1
sql> sensor_1,1547718199,sensor,6
sql> sensor_1,1547718199,1,1
result> sensor_6,1547718201,sensor,6
result> sensor_6,1547718201,6,1
sql> sensor_6,1547718201,sensor,6
sql> sensor_6,1547718201,6,1
result> sensor_7,1547718202,sensor,6
result> sensor_7,1547718202,7,1
sql> sensor_7,1547718202,sensor,6
sql> sensor_7,1547718202,7,1
result> sensor_10,1547718205,sensor,6
result> sensor_10,1547718205,10,2
sql> sensor_10,1547718205,sensor,6
sql> sensor_10,1547718205,10,2
result> sensor_1,1547718207,sensor,6
result> sensor_1,1547718207,1,1
sql> sensor_1,1547718207,sensor,6
sql> sensor_1,1547718207,1,1
result> sensor_1,1547718209,sensor,6
result> sensor_1,1547718209,1,1
sql> sensor_1,1547718209,sensor,6
sql> sensor_1,1547718209,1,1
result> sensor_1,1547718212,sensor,6
result> sensor_1,1547718212,1,1
sql> sensor_1,1547718212,sensor,6
sql> sensor_1,1547718212,1,1
```

### 8.1.3 聚合函数(Aggregate Functions)

**聚合，多对一，类似前面的窗口聚合**

---

+ 用户自定义聚合函数（User-Defined Aggregate Functions，UDAGGs）可以把一个表中的数据，聚合成一个标量值
+ 用户定义的聚合函数，是通过继承 AggregateFunction 抽象类实现的

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200601221643915.png)

+ AggregationFunction要求必须实现的方法
  + `createAccumulator()`
  + `accumulate()`
  + `getValue()`
+ AggregateFunction 的工作原理如下：
  + 首先，它需要一个累加器（Accumulator），用来保存聚合中间结果的数据结构；可以通过调用 `createAccumulator()` 方法创建空累加器
  + 随后，对每个输入行调用函数的 `accumulate()` 方法来更新累加器
  + 处理完所有行后，将调用函数的 `getValue()` 方法来计算并返回最终结果

#### 测试代码

```java
package apitest.tableapi.udf;

import apitest.beans.SensorReading;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.functions.AggregateFunction;
import org.apache.flink.types.Row;

public class UdfTest3_AggregateFunction {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);

    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

    // 1. 读取数据
    DataStream<String> inputStream = env.readTextFile("/tmp/Flink_Tutorial/src/main/resources/sensor.txt");

    // 2. 转换成POJO
    DataStream<SensorReading> dataStream = inputStream.map(line -> {
      String[] fields = line.split(",");
      return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
    });

    // 3. 将流转换成表
    Table sensorTable = tableEnv.fromDataStream(dataStream, "id, timestamp as ts, temperature as temp");

    // 4. 自定义聚合函数，求当前传感器的平均温度值
    // 4.1 table API
    AvgTemp avgTemp = new AvgTemp();

    // 需要在环境中注册UDF
    tableEnv.registerFunction("avgTemp", avgTemp);
    Table resultTable = sensorTable
      .groupBy("id")
      .aggregate("avgTemp(temp) as avgtemp")
      .select("id, avgtemp");

    // 4.2 SQL
    tableEnv.createTemporaryView("sensor", sensorTable);
    Table resultSqlTable = tableEnv.sqlQuery("select id, avgTemp(temp) " +
                                             " from sensor group by id");

    // 打印输出
    tableEnv.toRetractStream(resultTable, Row.class).print("result");
    tableEnv.toRetractStream(resultSqlTable, Row.class).print("sql");

    env.execute();
  }

  // 实现自定义的AggregateFunction
  public static class AvgTemp extends AggregateFunction<Double, Tuple2<Double, Integer>> {
    @Override
    public Double getValue(Tuple2<Double, Integer> accumulator) {
      return accumulator.f0 / accumulator.f1;
    }

    @Override
    public Tuple2<Double, Integer> createAccumulator() {
      return new Tuple2<>(0.0, 0);
    }

    // 必须实现一个accumulate方法，来数据之后更新状态
    // 这里方法名必须是这个，且必须public。
    // 累加器参数，必须得是第一个参数；随后的才是我们自己传的入参
    public void accumulate(Tuple2<Double, Integer> accumulator, Double temp) {
      accumulator.f0 += temp;
      accumulator.f1 += 1;
    }
  }
}
```

输出结果：

```shell
result> (true,sensor_1,35.8)
result> (true,sensor_6,15.4)
result> (true,sensor_7,6.7)
result> (true,sensor_10,38.1)
result> (false,sensor_1,35.8)
result> (true,sensor_1,36.05)
sql> (true,sensor_1,35.8)
result> (false,sensor_1,36.05)
sql> (true,sensor_6,15.4)
result> (true,sensor_1,34.96666666666666)
sql> (true,sensor_7,6.7)
result> (false,sensor_1,34.96666666666666)
sql> (true,sensor_10,38.1)
result> (true,sensor_1,35.5)
sql> (false,sensor_1,35.8)
sql> (true,sensor_1,36.05)
sql> (false,sensor_1,36.05)
sql> (true,sensor_1,34.96666666666666)
sql> (false,sensor_1,34.96666666666666)
sql> (true,sensor_1,35.5)
```

### 8.1.4 表聚合函数

+ 用户定义的表聚合函数（User-Defined Table Aggregate Functions，UDTAGGs），可以把一个表中数据，聚合为具有多行和多列的结果表
+ 用户定义表聚合函数，是通过继承 TableAggregateFunction 抽象类来实现的

![在这里插入图片描述](https://gitee.com/minan-palace/md_images/raw/master/images2/20200601223517314.png)

+ AggregationFunction 要求必须实现的方法：
  + `createAccumulator()`
  + `accumulate()`
  + `emitValue()`
+ TableAggregateFunction 的工作原理如下：
  + 首先，它同样需要一个累加器（Accumulator），它是保存聚合中间结果的数据结构。通过调用 `createAccumulator()` 方法可以创建空累加器。
  + 随后，对每个输入行调用函数的 `accumulate()` 方法来更新累加器。
  + 处理完所有行后，将调用函数的 `emitValue()` 方法来计算并返回最终结果。

#### 测试代码

> [Flink-函数 | 用户自定义函数（UDF）标量函数 | 表函数 | 聚合函数 | 表聚合函数](https://blog.csdn.net/qq_40180229/article/details/106482550)

```scala
import com.atguigu.bean.SensorReading
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.table.api.Table
import org.apache.flink.table.api.scala._
import org.apache.flink.table.functions.TableAggregateFunction
import org.apache.flink.types.Row
import org.apache.flink.util.Collector

object TableAggregateFunctionTest {
  def main(args: Array[String]): Unit = {

    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 开启事件时间语义
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    // 创建表环境
    val tableEnv: StreamTableEnvironment = StreamTableEnvironment.create(env)

    val inputDStream: DataStream[String] = env.readTextFile("D:\\MyWork\\WorkSpaceIDEA\\flink-tutorial\\src\\main\\resources\\SensorReading.txt")

    val dataDStream: DataStream[SensorReading] = inputDStream.map(
      data => {
        val dataArray: Array[String] = data.split(",")
        SensorReading(dataArray(0), dataArray(1).toLong, dataArray(2).toDouble)
      })
    .assignTimestampsAndWatermarks( new BoundedOutOfOrdernessTimestampExtractor[SensorReading]
                                   ( Time.seconds(1) ) {
                                     override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000L
                                   } )

    // 用proctime定义处理时间
    val dataTable: Table = tableEnv
    .fromDataStream(dataDStream, 'id, 'temperature, 'timestamp.rowtime as 'ts)

    // 使用自定义的hash函数，求id的哈希值
    val myAggTabTemp = MyAggTabTemp()

    // 查询 Table API 方式
    val resultTable: Table = dataTable
    .groupBy('id)
    .flatAggregate( myAggTabTemp('temperature) as ('temp, 'rank) )
    .select('id, 'temp, 'rank)


    // SQL调用方式，首先要注册表
    tableEnv.createTemporaryView("dataTable", dataTable)
    // 注册函数
    tableEnv.registerFunction("myAggTabTemp", myAggTabTemp)

    /*
    val resultSqlTable: Table = tableEnv.sqlQuery(
      """
        |select id, temp, `rank`
        |from dataTable, lateral table(myAggTabTemp(temperature)) as aggtab(temp, `rank`)
        |group by id
        |""".stripMargin)
*/


    // 测试输出
    resultTable.toRetractStream[ Row ].print( "scalar" )
    //resultSqlTable.toAppendStream[ Row ].print( "scalar_sql" )
    // 查看表结构
    dataTable.printSchema()

    env.execute(" table ProcessingTime test job")
  }
}

// 自定义状态类
case class AggTabTempAcc() {
  var highestTemp: Double = Double.MinValue
  var secondHighestTemp: Double = Double.MinValue
}

case class MyAggTabTemp() extends TableAggregateFunction[(Double, Int), AggTabTempAcc]{
  // 初始化状态
  override def createAccumulator(): AggTabTempAcc = new AggTabTempAcc()

  // 每来一个数据后，聚合计算的操作
  def accumulate( acc: AggTabTempAcc, temp: Double ): Unit ={
    // 将当前温度值，跟状态中的最高温和第二高温比较，如果大的话就替换
    if( temp > acc.highestTemp ){
      // 如果比最高温还高，就排第一，其它温度依次后移
      acc.secondHighestTemp = acc.highestTemp
      acc.highestTemp = temp
    } else if( temp > acc.secondHighestTemp ){
      acc.secondHighestTemp = temp
    }
  }

  // 实现一个输出数据的方法，写入结果表中
  def emitValue( acc: AggTabTempAcc, out: Collector[(Double, Int)] ): Unit ={
    out.collect((acc.highestTemp, 1))
    out.collect((acc.secondHighestTemp, 2))
  }
}
```


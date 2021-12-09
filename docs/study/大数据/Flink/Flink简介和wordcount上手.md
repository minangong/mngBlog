# Flink简介

Apache Flink 是一个框架和分布式处理引擎，用于对无界和有界**数据流**进行**有状态**计算。

目标：低延迟、高吞吐、结果的准确性和良好的容错性



处理流数据行业：电商和市场营销（数据报表、广告投放、业务流程）

​                               物联网（传感器实时数据采集显示、实时报警、交通运输）

​                                电信业（基站流量调配）

​                                银行和金融业（实时结算和通知推送，实时监测异常行为）



## 架构

### 传统数据处理架构

![image-20211008170501860](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008170501860.png)

管理系统、订单系统、wed应用等，读取并存储大量数据在数据库中。

数据量大的时候，数据库做联表查询时代价就会很高。

特点：实时性好，但是海量数据、高并发的时候不行。

### 离线分析处理架构

![image-20211008170856808](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008170856808.png)

特点：高并发可以，但是低延迟做不到



### 有状态的流式处理（第一代流处理器）

![image-20211008183614233](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183614233.png)

**低延迟：**事务处理原则，来一个处理一个（关系数据库关联查询很麻烦）

**高吞吐：**所以直接把数据存在本地内存，存成本地状态。来了事务后，与本地状态进行计算输出，状态更新，就改。

**容错：** 数据在内存里，不稳定。节点挂了，数据丢失。所以需要存盘和故障恢复机制。所以设计了周期性的检查点（periodic checkpoint）,定期保存，出现故障，恢复到上个存盘点。检查点放在远程的持久化空间。

**问题：**分布式有延迟，数据的时间顺序保证不了



### 第二代流处理器

![image-20211008183551176](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183551176.png)

**lambda架构：**用两套系统（批处理和流处理），同时保证低延迟和结果正确。流处理保证低延迟，批处理保证正确结果。

**效果：**用户会很快得到一个结果，但不一定结果正确。过一段时间后，获得正确结果。

**特点：**1. 需要维护两套系统



### 第三代流处理器

![image-20211008183940037](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211008183940037.png)

**Storm**是第一代的流处理器（特点：**低延迟**，吞吐量不行，时间顺序不保证），**spark streaming**是批处理演变过来的（特点：**高吞吐**，保证正确，但不低延迟，时间顺序不保证）。



## Flink特点

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



## Flink vs Spack Streaming

* 流与微批

  Flink 流 毫秒级，Spark Streaming 微批 秒级。

* 数据模型

  spark：RDD模型，Spark streaming 的 DStream实际上就是一组组小批数据RDD的集合

  flink基本数据模型是数据流，以及事件

* 运行时架构

  spark是批计算,将DAG划分为不同的stage,一个完成后才可以计算下一个

  flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理。



# Flink wordcount

## 1.maven pom.xml 依赖配置

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

## 2. flink  wordcount

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




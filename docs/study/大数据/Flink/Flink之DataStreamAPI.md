# Flink之DataStreamAPI

# 一、Flink流处理API



![image-20211213190036179](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211213190036179.png)

首先，获取环境。

Flink程序由三部分组成：Source、Transformation 和 Sink。

Source负责读取数据源、Transfromation利用各种算子进行处理加工，Sink负责输出。

## 1.1 Environment

### 1.1.1 getExecutionEnvironment

 创建一个执行环境，表示当前执行程序的上下文。

 如果程序是独立调用的，则此方法返回本地执行环境；如果从命令行客户端调用程序以提交到集群，则此方法返回此集群的执行环境。也就是说，getExecutionEnvironment会根据查询运行的方式决定返回什么样的运行环境。

```java
//创建运行时流处理环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
//创建批处理运行环境
ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
```



### 1.1.2 creatLocalEnvironment

返回本地执行环境，可以指定默认的并行度

```java
LocalStreamEnvironment localEnvironment = StreamExecutionEnvironment.createLocalEnvironment(1);
```



### 1.1.3 createRemoteEnvironment

返回集群执行环境，将Jar提交到远程服务器。需要在调用时指定JobManager的IP和端口号，并指定要在集群中运行的Jar包。

![image-20211213205056851](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211213205056851.png)



## 1.2 Source



### 1.2.1 从集合中读取数据

```java
        //创建运行时流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度
        env.setParallelism(4);
        DataStream<SensorReading> dataStream = env.fromCollection(
            Arrays.asList(
                new SensorReading("sensor_1",154771819L,35.8),
                new SensorReading("sensor_6",1547718201L,15.4),
                new SensorReading("sensor_1",1547718209L,32.8),
                new SensorReading("sensor_1",1547718212L,37.1)
            )
        );
        DataStream<String> stringDataStream = env.fromElements("sensor1","sensor2","sensor3");

        dataStream.print("haha1");
        stringDataStream.print("haha2");
        env.execute("mng");   //jobname
```

输出：

![image-20211214203407296](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211214203407296.png)

### 1.2.2 从socket中读取数据

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
        ParameterTool parameterTool = ParameterTool.fromArgs(args);
        String host = parameterTool.get("host");
        int port = parameterTool.getInt("port");
//        从socket文本流读取数据
        DataStream<String> inputDataStream = env.socketTextStream(host,port);

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

打包上传后。

socket传递string（回退 ctrl + backspace）

实则传了  hello world   hello  mng

![image-20211206214631776](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206214631776.png)

结果：

![image-20211206221343550](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206221343550.png)

### 1.2.3 从文件中读取数据

```
        //创建运行时流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度
        env.setParallelism(4);
        DataStream<String> stringDataStream = env.readTextFile("src/main/resources/wc.txt");
        stringDataStream.print("haha2");
        env.execute("mng");
```

输出：

![image-20211214211652304](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211214211652304.png)

### 1.2.4 从kafka读取数据

**具体在7.1**

1.pom依赖

```
        <!-- kafka -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
```

2.启动zookeeper

```shell
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

3.启动kafka服务

```shell
$ bin/kafka-server-start.sh config/server.properties
```

4.启动kafka生产者

```shell
$ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
```

5.编写java代码

```java
package apitest.source;

import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;

import java.util.Properties;

/**
 * @author : Ashiamd email: ashiamd@foxmail.com
 * @date : 2021/1/31 5:44 PM
 */
public class SourceTest3_Kafka {

    public static void main(String[] args) throws Exception {
        // 创建执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 设置并行度1
        env.setParallelism(1);

        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers", "47...:9092");
        // 下面这些次要参数
        properties.setProperty("group.id", "consumer-group");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");

        // flink添加外部数据源
        DataStream<String> dataStream = env.addSource(new FlinkKafkaConsumer<String>("sensor", new SimpleStringSchema(),properties));

        // 打印输出
        dataStream.print();

        env.execute();
    }
}
```

6.运行java代码，在Kafka生产者console中输入

```shell
$ bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
>sensor_1,1547718199,35.8
>sensor_6,1547718201,15.4
>
```

7.java输出

```shell
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
```



### 1.2.5 自定义Source

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.utils.ParameterTool;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.util.Collector;

import java.util.Arrays;
import java.util.HashMap;
import java.util.Random;

public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        //创建运行时流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度
        env.setParallelism(1);

        DataStream<SensorReading> sensorReadingDataStream = env.addSource(new MySource());
        sensorReadingDataStream.print("MySource");
        env.execute("mng");
    }
    public static class MySource implements SourceFunction<SensorReading>{
        //标志位
        private volatile boolean running = true;
        @Override
        public void run(SourceContext<SensorReading> sourceContext) throws Exception {
            Random random = new Random();
            HashMap<String,Double> sensorTempMap = new HashMap<>();
            for(int i = 0;i<10;++i){
                sensorTempMap.put("sensor"+(i+1),60+random.nextGaussian()*20);
            }
            while (running){
                for (String sensorId : sensorTempMap.keySet()) {
                    // 在当前温度基础上随机波动
                    Double newTemp = sensorTempMap.get(sensorId) + random.nextGaussian();
                    //sensorTempMap.put(sensorId, newTemp);
                    sourceContext.collect(new SensorReading(sensorId,System.currentTimeMillis(),newTemp));
                }
                cancel();
                // 控制输出评率
                Thread.sleep(2000L);
            }

        }

        @Override
        public void cancel() {
            this.running = false;

        }
    }
}

```

输出：

![image-20211214232716796](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211214232716796.png)

## 1.3 Transform

### 1.3.1 基本转换算子（map/flatMap/filter）

map：把数组流中的每一个值，使用所提供的函数执行一遍，一一对应。得到**元素个数相同**的数组流。

flatmap：把数组流中的每一个值，使用所提供的函数执行一遍，一一对应。得到**元素个数不同/相同**的数组流。

```java
import org.apache.flink.api.common.functions.FilterFunction;
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;


public class StreamWordCount {
    public static void main(String[] args) throws Exception {
        //创建运行时流处理环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //设置并行度
        env.setParallelism(4);
        DataStream<String> stringDataStream = env.readTextFile("src/main/resources/wc.txt");
        //map string -->SensorReading
        DataStream<SensorReading> sensorReadingDataStream = stringDataStream.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
                String[] strings = s.split(",");
                String id = strings[0];
                Long timestamp = Long.parseLong(strings[1]);
                Double temp = Double.parseDouble(strings[2]);
                return new SensorReading(id,timestamp,temp);
            }
        });
        //flatmap String --> String.split
        DataStream<String> stringDataStreamWithFlat = stringDataStream.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public void flatMap(String s, Collector<String> collector) throws Exception {
                String[] strings = s.split(",");
                for(String str : strings){
                    collector.collect(str);
                }
            }
        });
        //filter  sensorReading.id start with sensor_1
        DataStream<SensorReading> dataStreamWithFilter = sensorReadingDataStream.filter(new FilterFunction<SensorReading>() {
            @Override
            public boolean filter(SensorReading sensorReading) throws Exception {
                String id = sensorReading.getId();
                return id.startsWith("sensor_1");
            }
        });
        sensorReadingDataStream.print("map");
        stringDataStreamWithFlat.print("flatmap");
        dataStreamWithFilter.print("filter");
        env.execute();
    }
    
}

```

输出：

![image-20211215092816012](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211215092816012.png)







### 1.3.2 聚合操作算子

DataStream 没有reduce、sum等聚合操作的方法。所以必须**先分组在聚合**。先keyBy得到KeyedStream, 然后调用reduce、sum等聚合操作算法。

常见聚合操作算子有：

* keyBy
* 滚动聚合算子Rolling Aggregation
* reduce



#### 1.3.2.1 keyBy

![image-20211224210641128](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211224210641128.png)

**DataStream -> KeyedStream**：逻辑地将一个流拆分成不相交的分区，每个分区包含具有相同key的元素，在内部以hash的形式(再取模)实现的。

1、KeyBy会重新分区； 2、不同的key有可能分到一起，因为是通过hash原理+取模实现的；



#### 1.3.2.2 滚动聚合算子

这些算子可以针对KeyedStream的每一个支流做聚合。

* sum()
* min()
* max()：只有比较的字段更新，其他字段不变
* minBy()
* maxBy()：其他字段也会更新成最新的



```java
package com.mng;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

public class KeyStreamTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<String> stringDataStream = env.readTextFile("src/main/resources/wc.txt");
        env.readTextFile("src/main/resources/wc.txt");

        DataStream<SensorReading> sensorReadingDataStream = stringDataStream.flatMap(new FlatMapFunction<String, SensorReading>() {

            @Override
            public void flatMap(String s, Collector<SensorReading> collector) throws Exception {
                String[] strings = s.split(",");
                String id = strings[0];
                Long timestamp = Long.parseLong(strings[1]);
                Double temp = Double.parseDouble(strings[2]);
                collector.collect(new SensorReading(id,timestamp,temp));
            }
        });
        //聚合
        KeyedStream<SensorReading, String> sensorReadingStringKeyedStream = sensorReadingDataStream.keyBy(sensorReading -> sensorReading.id);
        //max 
        DataStream<SensorReading> sKey1 = sensorReadingStringKeyedStream.max("temperaturre");
        sKey1.print("result");
        env.execute();
    }
}

```

结果：可以看到timestamp字段没有更新

![image-20211224221351080](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211224221351080.png)

**maxBy结果**

![image-20211224221126557](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211224221126557.png)

**sum结果**

![image-20211224222045840](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211224222045840.png)

#### 1.3.2.3 reduce

![image-20211225080913459](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211225080913459.png)



```java
        DataStream<SensorReading> sKey1 = sensorReadingStringKeyedStream.reduce(
                new ReduceFunction<SensorReading>() {
                    @Override
                    public SensorReading reduce(SensorReading sensorReading, SensorReading t1) throws Exception {
                        t1.setTemperaturre(Math.max(sensorReading.getTemperaturre(),t1.getTemperaturre()));
                        return t1;
                    }
                }
        );
```

![image-20211225105547333](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211225105547333.png)







### 1.3.3 多流转换算子

#### 1.3.3.1 connect算子

![image-20211226070004967](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211226070004967.png)

**DataStream,DataStream -> ConnectedStreams**: 连接两个保持他们类型的数据流，两个数据流被Connect 之后，只是被放在了一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立。



#### 1.3.3.2 CoMap/CoFlatMap

![image-20211226070239753](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211226070239753.png)

**ConnectedStreams -> DataStream**: 作用于ConnectedStreams 上，功能与map和flatMap一样，对ConnectedStreams 中的**每一个Stream分别进行map和flatMap操作**；



示例：

```java
package com.mng;

import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.ConnectedStreams;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.CoMapFunction;

public class TansformTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> stringDS = env.readTextFile("src/main/resources/wc.txt");

        SingleOutputStreamOperator<SensorReading> sensorDS = stringDS.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
                String[] strings = s.split(",");
                return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));
            }
        });
        ConnectedStreams<String, SensorReading> connect = stringDS.connect(sensorDS);
        SingleOutputStreamOperator<Object> connectResult = connect.map(new CoMapFunction<String, SensorReading, Object>() {
            @Override
            public Object map1(String s) throws Exception {
                return s;
            }

            @Override
            public Object map2(SensorReading sensorReading) throws Exception {
                return sensorReading;
            }
        });
        connectResult.print();
        env.execute();
    }
}

```

输出结果：

![image-20211226074246512](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211226074246512.png)





#### 1.3.3.3 union算子

![image-20211226074455018](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211226074455018.png)

**DataStream -> DataStream**：对**两个或者两个以上**的DataStream进行Union操作，产生一个包含多有DataStream元素的新DataStream。

**问题：和Connect的区别？**

1. Connect 的数据类型可以不同，**Connect 只能合并两个流**；
2. **Union可以合并多条流，Union的数据结构必须是一样的**；

```
        DataStream<SensorReading> union1 = sensorDS.union(sensorDS);
```



## 1.4 flink支持的数据类型

 Flink流应用程序处理的是以数据对象表示的事件流。所以在Flink内部，我们需要能够处理这些对象。它们**需要被序列化和反序列化**，以便通过网络传送它们；或者从状态后端、检查点和保存点读取它们。为了有效地做到这一点，Flink需要明确知道应用程序所处理的数据类型。Flink使用类型信息的概念来表示数据类型，并为每个数据类型生成特定的序列化器、反序列化器和比较器。

 Flink还具有一个类型提取系统，该系统分析函数的输入和返回类型，以自动获取类型信息，从而获得序列化器和反序列化器。但是，在某些情况下，例如lambda函数或泛型类型，需要显式地提供类型信息，才能使应用程序正常工作或提高其性能。

 Flink支持Java和Scala中所有常见数据类型。

### 1.4.1 基础数据类型

Int, Double, Long, String, …

### 1.4.2 Java和scala元组（Tuples）

java不像Scala天生支持元组Tuple类型，java的元组类型由Flink的包提供，默认提供Tuple0~Tuple25

```
DataStream<Tuple2<String, Integer>> personStream = env.fromElements( 
  new Tuple2("Adam", 17), 
  new Tuple2("Sarah", 23) 
); 
personStream.filter(p -> p.f1 > 18);
```

### 1.4.3 Scala样例类(case classes)

```scala
case class Person(name:String,age:Int)

val numbers: DataStream[(String,Integer)] = env.fromElements(
  Person("张三",12),
  Person("李四"，23)
)
```

### 1.4.4 Java简单对象(POJO)

java的POJO这里要求必须提供无参构造函数

* 成员变量要求都是public（或者private但是提供get、set方法）

### 1.4.5 其他(Arrays, Lists, Maps, Enums,等等)

Flink对Java和Scala中的一些特殊目的的类型也都是支持的，比如Java的ArrayList，HashMap，Enum等等。



## 1.5 实现UDF函数——更细粒度的控制流

### 1.5.1 函数类(Function Classes)

 Flink暴露了所有UDF函数的接口(实现方式为接口或者抽象类)。例如MapFunction, FilterFunction, ProcessFunction等等。

 下面例子实现了FilterFunction接口：

```java
DataStream<String> flinkTweets = tweets.filter(new FlinkFilter()); 
public static class FlinkFilter implements FilterFunction<String> { 
  @Override public boolean filter(String value) throws Exception { 
    return value.contains("flink");
  }
}
```

 还可以将函数实现成匿名类

```java
DataStream<String> flinkTweets = tweets.filter(
  new FilterFunction<String>() { 
    @Override public boolean filter(String value) throws Exception { 
      return value.contains("flink"); 
    }
  }
);
```

 filter的字符串"flink"还可以当作参数传进去。

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE "); 
DataStream<String> flinkTweets = tweets.filter(new KeyWordFilter("flink")); 
public static class KeyWordFilter implements FilterFunction<String> { 
  private String keyWord; 

  KeyWordFilter(String keyWord) { 
    this.keyWord = keyWord; 
  } 

  @Override public boolean filter(String value) throws Exception { 
    return value.contains(this.keyWord); 
  } 
}
```

### 1.5.2 匿名函数(Lambda Functions)

```java
DataStream<String> tweets = env.readTextFile("INPUT_FILE"); 
DataStream<String> flinkTweets = tweets.filter( tweet -> tweet.contains("flink") );
```

### 1.5.3 富函数(Rich Functions)

 “富函数”是DataStream API提供的一个函数类的接口，所有Flink函数类都有其Rich版本。

 **它与常规函数的不同在于，可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能**。

* RichMapFunction
* RichFlatMapFunction
* RichFilterFunction
* …

 Rich Function有一个**生命周期**的概念。典型的生命周期方法有：

* **`open()`方法是rich function的初始化方法，当一个算子例如map或者filter被调用之前`open()`会被调用。**
* **`close()`方法是生命周期中的最后一个调用的方法，做一些清理工作。**
* **`getRuntimeContext()`方法提供了函数的RuntimeContext的一些信息，例如函数执行的并行度，任务的名字，以及state状态**



## 1.6 数据重分区

**其中`partitionCustom(...)`方法用于自定义重分区**。

```
        sensorDS.partitionCustom(
                new Partitioner<Object>() {
                    @Override
                    public int partition(Object o, int i) {
                        return 0;
                    }
                }, 
                new KeySelector<SensorReading, Object>() {
                    @Override
                    public Object getKey(SensorReading sensorReading) throws Exception {
                        return null;
                    }
                });
```

```
    // SingleOutputStreamOperator多并行度默认就rebalance,轮询方式分配
    dataStream.print("input");

    // 1. shuffle (并非批处理中的获取一批后才打乱，这里每次获取到直接打乱且分区)
    DataStream<String> shuffleStream = inputStream.shuffle();
    shuffleStream.print("shuffle");

    // 2. keyBy (Hash，然后取模)
    dataStream.keyBy(SensorReading::getId).print("keyBy");

    // 3. global (直接发送给第一个分区，少数特殊情况才用)
    dataStream.global().print("global");
```





## 1.7 Sink

Flink没有类似于spark中foreach方法，让用户进行迭代的操作。虽有对外的输出操作都要利用Sink完成。最后通过类似如下方式完成整个任务最终输出操作。

```
myDstream.addSink(new MySink(xxxx)) 
```



 官方提供了一部分的框架的sink。除此以外，需要用户自定义实现sink。

<div align="left"><img src=https://gitee.com/minan-palace/md_images/raw/master/images/image-20211227071141532.png ></div>



### 1.7.1 kafka

配置

```xml
        <!-- kafka -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka_${scala.binary.version}</artifactId>
            <version>${flink.version}</version>
        </dependency>
```

代码：（注意修改ip）

```java
package com.mng.sink;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;

import java.util.Properties;

public class KafkaSinkTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        // 设置并行度1
        env.setParallelism(1);
        Properties properties = new Properties();
        properties.setProperty("bootstrap.servers","47...:9092");
        properties.setProperty("group.id", "default");
        properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        properties.setProperty("auto.offset.reset", "latest");
        // 从Kafka中读取数据
        DataStream<String> inputStream = env.addSource( new FlinkKafkaConsumer<String>("sensor", new SimpleStringSchema(), properties));
        // 序列化从Kafka中读取的数据
        DataStream<String> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            SensorReading sensorReading = new SensorReading(fields[0], Long.parseLong(fields[1]), Double.parseDouble(fields[2]));
            //System.out.println(sensorReading);
            return sensorReading.toString();
        });
        dataStream.print();
        // 将数据写入Kafka
        dataStream.addSink( new FlinkKafkaProducer<String>("47...:9092", "sinktest", new SimpleStringSchema()));
        env.execute();

    }
}
```



1.新建kafka生产者

```
bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
```

2.新建kafka消费者

```
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sinktest
```



ps: kafka 需要设置监听端口(config/server.properties)

![image-20211230185812523](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211230185812523.png)



3.运行Flink程序，生产者输入数据，查看消费者输出结果

生产者输入：

```
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
```

![image-20211230190306492](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211230190306492.png)

消费者输出：

![image-20211230190318630](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211230190318630.png)



idea输出：（dataStream.print()结果）

![image-20211230190335880](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211230190335880.png)







### 1.7.2 redis



### 1.7.3 Elasticsearch



### 1.7.4 JDBC 自定义sink（包括JDBC source）

通过富函数实现

```java
package com.mng.sink;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.RichSinkFunction;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;

public class JdbcSinkTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<SensorReading> sensorDs = env.addSource(new SensorSourceFromMysql());
        sensorDs.print("input");
        sensorDs.addSink(new SensorSinkToMysql());
        env.execute();
    }

}

class SensorSourceFromMysql extends RichSourceFunction<SensorReading> {

    String driver = "com.mysql.cj.jdbc.Driver";
    String url = "jdbc:mysql://localhost:3306/flinksinktest?serverTimezone=GMT";
    String username = "root";
    String password = "1234";
    Connection connection = null;
    PreparedStatement ps = null;

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        connection = getConnection();
        String sql = "select * from sensorinput;";
        //获取执行语句
        ps = connection.prepareStatement(sql);
    }
    @Override
    public void run(SourceContext<SensorReading> sourceContext) throws Exception {
        ResultSet resultSet = ps.executeQuery();
        while(resultSet.next()){
            SensorReading sensorReading = new SensorReading(
                    resultSet.getString("id"),
                    resultSet.getLong("timestamp"),
                    resultSet.getDouble("temperaturre")
            );
            sourceContext.collect(sensorReading);
        }
    }

    @Override
    public void cancel() {

    }


    @Override
    public void close() throws Exception {
        super.close();
        if(connection != null){
            connection.close();
        }
        if (ps != null){
            ps.close();
        }

    }

    //获取mysql连接配置
    public Connection getConnection() {
        try {
            //加载驱动
            Class.forName(driver);
            //创建连接
            connection = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            System.out.println("********mysql get connection occur exception, msg = " + e.getMessage());
            e.printStackTrace();
        }
        return connection;
    }
}

class SensorSinkToMysql extends RichSinkFunction<SensorReading>{
    String driver = "com.mysql.cj.jdbc.Driver";
    String url = "jdbc:mysql://localhost:3306/flinksinktest?serverTimezone=GMT";
    String username = "root";
    String password = "1234";
    Connection connection = null;
    PreparedStatement ps = null;
    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        connection = getConnection();
        String sql = "insert into sensoroutput values (?,?,?);";
        ps = connection.prepareStatement(sql);
    }

    @Override
    public void close() throws Exception {
        super.close();
        ps.close();
        connection.close();
    }

    @Override
    public void invoke(SensorReading value, Context context) throws Exception {
        ps.setString(1,value.getId());
        ps.setLong(2,value.getTimestamp());
        ps.setDouble(3,value.getTemperaturre());
        ps.execute();
    }

    //获取mysql连接配置
    public Connection getConnection() {
        try {
            //加载驱动
            Class.forName(driver);
            //创建连接
            connection = DriverManager.getConnection(url, username, password);
        } catch (Exception e) {
            System.out.println("********mysql get connection occur exception, msg = " + e.getMessage());
            e.printStackTrace();
        }
        return connection;
    }
}
```

结果：

![image-20211231111434599](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211231111434599.png)

![image-20211231111421483](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211231111421483.png)









# 二、Flink Window API

## 2.1 window 概念

窗口（window）就是将无限流切割为有限流的一种方式，它会将流数据分发到有限大小的桶（bucket）中进行分析。



## 2.2 window 类型

* 时间窗口

  * 滚动时间窗口：按固定的窗口长度对数据切分，时间对齐，没有重叠。

  ![image-20211231210031084](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211231210031084.png)

  * 滑动时间窗口：固定窗口长度、滑动间隔，可以重叠

  ![image-20211231211158709](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211231211158709.png)

  * 会话窗口：由一系列事件组合一个指定时间长度的timeout间隙组成，一段时间没有接收到新数据就会生成新矿口。时间无对齐。

  ![image-20211231212034533](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211231212034533.png)

* 技术窗口

  * 滚动计数窗口
  * 滑动计数窗口





## 2.3 window API

* 窗口分配器----window方法

* 可以用.window()来定义一个窗口，再做一些聚合或者其他处理操作，window()方法必须在keyBy之后才能用。
* Flink提供了更简单的.timeWindow和.countWindow方法，用于定义时间窗口和计数窗口。

> PS:时间分为处理时间（Processing time 执行算子操作的机器系统时间）、事件时间（Event time 每个单独的事件在其生产设备上发生的时间，通常内置于数据记录中）

### 2.3.1 创建不同类型的窗口

滚动时间窗口(tumbling time window)

时间间隔：Time.milliseconds(),Time.seconds(),Time.minutes()等

```java
.timeWindow(Time.seconds(15))
```

滑动时间窗口：(sliding time window)

每5s就计算输出结果一次，每一次计算的window范围是15s内的所有元素。

```java
.timeWindow(Time.seconds(15),Time.seconds(5))
```

会话窗口(session window)

```java
.window(EventTimeSessionWindows.withGap(Time.minutes(10)))
```

滚动计数窗口(tumbling count window)

```java
.countWindow(5)
```

滑动计数窗口(sliding count window)

每收到两个相同key的数据就计算一次，每次计算的范围是10哥元素。

```java
.countWindow(10,2)
```



### 2.3.2 窗口函数

window function定义了要对窗口中收集的数据做的计算操作

分为：

* 增量聚合函数：每条数据到来就进行计算

  有ReduceFunction,AggregateFunction

* 全窗口函数：先把窗口所有数据收集起来，计算时遍历所有数据

  ProcessWindowFunction,WindowFunction

  ProcessWindowFunction获得一个包含窗口所有元素的可迭代器，以及一个具有时间和状态信息访问权的上下文对象，这使得它比其他窗口函数提供更大的灵活性。这是以性能和资源消耗为代价的，因为元素不能增量地聚合，而是需要在内部缓冲，直到认为窗口可以处理为止，使用该函数需要注意数据量，数据量太大，全量数据保存在内存中，会造成内存溢出。

### 2.3.3 其他API

* .trigger  ——  触发器

  定义window什么时候关闭，出发计算并输出结果

* .evictor——移除器

  定义移除某些数据的逻辑

* .allowedLateness——允许处理迟到的数据

* .sideOutputLateData——将迟到的数据放入侧输出流

* .getSideOutput——获取侧输出流



## 2.4 代码示例

### 2.4.1 滚动时间窗口的增量聚合函数

```
package com.mng.window;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.java.functions.KeySelector;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;

public class windowTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> stringDs = env.socketTextStream("localhost",7777);

        SingleOutputStreamOperator<SensorReading> sensorDs = stringDs.map(line ->{
            String[] strings = line.split(",");
            return new SensorReading(strings[0],Long.parseLong(strings[1]),Double.parseDouble(strings[2]));
            }
        );
        sensorDs.keyBy(sensor -> sensor.getId())
                .window(TumblingProcessingTimeWindows.of(Time.seconds(30)))
                .aggregate(new AggregateFunction<SensorReading, SensorReading, SensorReading>() {
                    //新建的累加器
                    @Override
                    public SensorReading createAccumulator() {
                        return null;
                    }

                    //每个数据在上次的基础上累加
                    @Override
                    public SensorReading add(SensorReading sensorReading, SensorReading o) {
                        if(o == null || sensorReading.getTemperaturre()>o.getTemperaturre()) {
                            return sensorReading;
                        }
                        return o;
                    }

                    //返回结果值
                    @Override
                    public SensorReading getResult(SensorReading o) {
                        return o;
                    }

                    // 分区合并结果(TimeWindow一般用不到，SessionWindow可能需要考虑合并)
                    @Override
                    public SensorReading merge(SensorReading o, SensorReading acc1) {
                        return null;
                    }
                })
                .print();
                //.max("temperaturre")
                //.print();
        env.execute();
    }
}

```





windows下

```shell
nc -l -p 7777
```

输入：

![image-20220101232224327](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220101232224327.png)

输出：

![image-20220101232239565](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220101232239565.png)









### 2.4.2 滑动时间窗口的全局聚合函数

```
package com.mng.window;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.SlidingProcessingTimeWindows;
import org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;

public class processWindowTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        DataStreamSource<String> stringDs = env.socketTextStream("localhost",7777);

        SingleOutputStreamOperator<SensorReading> sensorDs = stringDs.map(line ->{
                    String[] strings = line.split(",");
                    return new SensorReading(strings[0],Long.parseLong(strings[1]),Double.parseDouble(strings[2]));
                }
        );
        sensorDs.keyBy(sensor -> sensor.getId())
                .window(SlidingProcessingTimeWindows.of(Time.seconds(30),Time.seconds(10)))
                .process(new ProcessWindowFunction<SensorReading, SensorReading, String, TimeWindow>() {
                    @Override
                    public void clear(Context context) throws Exception {
                        super.clear(context);
                    }

                    @Override
                    public void process(String s, Context context, Iterable<SensorReading> iterable, Collector<SensorReading> collector) throws Exception {
                        Long maxTime = 0L;
                        Double sumTem = 0D;
                        Long count = 0L;
                        for(SensorReading sensorReading  : iterable){
                            if(sensorReading.getTimestamp() > maxTime){
                                maxTime = sensorReading.getTimestamp();
                            }
                            sumTem += sensorReading.getTemperaturre();
                            count +=1;
                        }
                        collector.collect(new SensorReading(s,maxTime,sumTem/count));
                    }
                })
                .print();
        env.execute();
    }
}

```



windows下

```
nc -l -p 7777
```

输入：

![image-20220102002746707](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220102002746707.png)

输出：

![image-20220102002941702](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220102002941702.png)



# 三、时间语义和Watermark

## 3.1 Flink中的时间语义

![image-20220102131909104](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220102131909104.png)

Event Time：事件创建的时间,一般使用更多。

Ingestion Time：数据进入Flink的时间

Processing Time：执行操作算子的本地系统时间



## 3.2 EventTime的引入

 **在Flink的流式处理中，绝大部分的业务都会使用eventTime**，一般只在eventTime无法使用时，才会被迫使用ProcessingTime或者IngestionTime。

默认环境里使用的是Processing,如果要使用EventTime,那么需要引入EventTime的时间属性。

新版

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 从调用时刻开始给env创建的每一个stream追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```



## 3.3 Watermark

### 3.3.1 乱序数据

* flink以Event Time模式处理数据流时，会根据数据中的时间戳处理基于时间的算子
* 由于网络、分布式等原因，会导致乱序数据的产生
* 乱序数据会让窗口计算不准确

![image-20220103192601355](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220103192601355.png)



Flink对于迟到数据有三层保障,先来后到的保障顺序是：

* WaterMark => 约等于放宽窗口标准
* allowedLateness => 允许迟到（ProcessingTime超时，但是EventTime没超时）
* sideOutputLateData => 超过迟到时间，另外捕获，之后可以自己批处理合并先前的数据



### 3.3.2 水位线（watermark）

* 遇到一个时间戳到达了窗口关闭时间，不应该立刻触发窗口计算，而是等待一段时间，等迟到的数据来了再关闭窗口

* Watermark是一种衡量Event Time进展的机制，可以设定延迟触发
* Watermark是用于处理乱序事件的，正确的处理乱序事件，通常用Watermark机制结合window来实现
* watermark用来让程序自己平衡延迟和结果正确性

定义：WaterMark是一种特殊的时间戳，它会被插入到数据流中，用于表示EventTime小于Watermark的事件全部落入到了相应的窗口中。**如果有窗口的停止时间等于maxEventTime – t，那么这个窗口被触发执行。**

例如：延迟时间为2s,当event time为12的数据到来时，生成了event time - t = 10的watermark ,系统认为所有event time<=10的数据到达，所以关闭5-10的窗口.

![image-20220103194149936](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220103194149936.png)





## 3.4 Watermark的引入

调用 assignTimestampAndWatermarks 方法，传入一个

*  BoundedOutOfOrdernessTimestampExtractor，可以指定延迟时间和时间戳。
*  AscendingTimestampExtractor,，可以指定时间戳。





## 3.5 TimestampAssigner

Flink 暴露了 TimestampAssigner 接口供我们实现，使我们可以自定义如何从事件数据中抽取时间戳和生成watermark 

MyAssigner 可以有两种类型，都继承自 TimestampAssigner，TimestampAssigner 定义了抽取时间戳，以及生成 watermark 的方法，有两种类型 

*  AssignerWithPeriodicWatermarks：
   * 周期性的生成 watermark：系统会周期性的将 watermark 插入到流中 
   * 默认周期是200毫秒，可以使用 ExecutionConfig.setAutoWatermarkInterval() 方法进行设置 
   * 升序和前面乱序的处理 
   * BoundedOutOfOrdernessTimestampExtractor， 都是基于周期性 watermark 的。
*  AssignerWithPunctuatedWatermarks 
   * 没有时间周期规律，可打断的生成 watermark







## 3.6 代码示例

```java
package com.mng.watermark;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.TimeCharacteristic;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.OutputTag;

import java.time.Duration;

public class EventTimeTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(4);

        //env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
        env.getConfig().setAutoWatermarkInterval(100);

        // socket文本流
        DataStream<String> inputStream = env.socketTextStream("localhost", 7777);

        // 转换成SensorReading类型，分配时间戳和watermark
        DataStream<SensorReading> dataStream = inputStream.map(line -> {
            String[] fields = line.split(",");
            return new SensorReading(fields[0], new Long(fields[1]), new Double(fields[2]));
        })
                .assignTimestampsAndWatermarks(WatermarkStrategy
                        .<SensorReading>forBoundedOutOfOrderness(Duration.ofSeconds(2))
                .withTimestampAssigner(new SerializableTimestampAssigner<SensorReading>() {
                    @Override
                    public long extractTimestamp(SensorReading sensorReading, long l) {
                        return sensorReading.getTimestamp()*1000L;
                    }
                }));

//                // 乱序数据设置时间戳和watermark
//                .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<SensorReading>(Time.seconds(2)) {
//                    @Override
//                    public long extractTimestamp(SensorReading element) {
//                        return element.getTimestamp() * 1000L;
//                    }
//                });

        OutputTag<SensorReading> outputTag = new OutputTag<SensorReading>("late") {
        };

        // 基于事件时间的开窗聚合，统计15秒内温度的最小值
        SingleOutputStreamOperator<SensorReading> minTempStream = dataStream.keyBy("id")
                //.timeWindow(Time.seconds(15))
                .window(TumblingEventTimeWindows.of(Time.seconds(15)))
                .allowedLateness(Time.minutes(1))
                .sideOutputLateData(outputTag)
                .minBy("temperaturre");

        minTempStream.print("minTemp");
        minTempStream.getSideOutput(outputTag).print("late");

        env.execute();
    }
}

```



windows打开socket

```
nc -lp 7777
```

输入：

```
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_1,1547718207,36.3
sensor_1,1547718211,34
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
sensor_1,1547718212,31.9
```

输出：（上面全部输入后得到）

![image-20220104134634722](https://gitee.com/minan-palace/md_images/raw/master/images2/image-20220104134634722.png)

分析：

1.**计算窗口起始位置Start和结束位置End**

从TumblingProcessingTimeWindows类里的**assignWindows**方法，我们可以得知窗口的起点计算方法如下： 

​                                 $$ start = timestamp - (timestamp -offset+WindowSize) % WindowSize $$ 

由于我们没有设置offset，所以这里start=第一个数据的时间戳**1547718199-(1547718199-0+15)%15=1547718195**

计算得到窗口初始位置为**Start = 1547718195**，那么这个窗口理论上本应该在1547718195+15的位置关闭，也就是**End=1547718210**



2.**计算修正后的Window输出结果的时间**

测试代码中Watermark设置的`maxOutOfOrderness`最大乱序程度是2s，所以实际获取到End+2s的时间戳数据时（达到Watermark），才认为Window需要输出计算的结果（不关闭，因为设置了允许迟到1min）

**所以实际应该是1547718212的数据到来时才触发Window输出计算结果。**





为什么上面输入中，最后连续四条相同输入，才触发Window输出结果？

* **Watermark会向子任务广播**
  * 我们在map才设置Watermark，map根据Rebalance轮询方式分配数据。所以前4个输入分别到4个slot中，4个slot计算得出的Watermark不同（分别是1547718199-2，1547718201-2，1547718202-2，1547718205-2）
* **Watermark传递时，会选择当前接收到的最小一个作为自己的Watermark**
  * 前4次输入中，有些map子任务还没有接收到数据，所以其下游的keyBy后的slot里watermark就是`Long.MIN_VALUE`（因为4个上游的Watermark广播最小值就是默认的`Long.MIN_VALUE`）
  * 并行度4，在最后4个相同的输入，使得Rebalance到4个map子任务的数据的`currentMaxTimestamp`都是1547718212，经过`getCurrentWatermark()`的计算（`currentMaxTimestamp-maxOutOfOrderness`），4个子任务都计算得到watermark=1547718210，4个map子任务向4个keyBy子任务广播`watermark=1547718210`，使得keyBy子任务们获取到4个上游的Watermark最小值就是1547718210，然后4个KeyBy子任务都更新自己的Watermark为1547718210。
* **根据Watermark的定义，我们认为>=Watermark的数据都已经到达。由于此时watermark >= 窗口End，所以Window输出计算结果（4个子任务，4个结果）。**






























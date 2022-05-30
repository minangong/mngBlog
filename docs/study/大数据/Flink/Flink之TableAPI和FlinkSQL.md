# Flink之TableAPI和FlinkSQL



# 一、概述

* Flink对批处理和流处理，提供了统一的上层API

* Table API是一套内嵌在Java和Scala语言中的查询API，它允许以非常直观的方式组合来自一些关系运算符的查询

* Flink的SQL支持基于实现了SQL标准的Apache Calcite

  ![image-20220120170245054](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220120170245054.png)



## 代码示例

导入依赖

```
<!-- Table API 和 Flink SQL -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table-planner-blink_${scala.binary.version}</artifactId>
  <version>${flink.version}</version>
</dependency>
```

```java
package com.mng.tableAPI;

import com.mng.dto.SensorReading;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.types.Row;

import static org.apache.flink.table.api.Expressions.$;

public class easyTest {
    public static void main(String[] args) throws Exception {
        // 1.创建流执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        // 2. 读取数据
        DataStreamSource<String> stringDS = env.readTextFile("src/main/resources/wc.txt");
        // 3.转化成POJO
        SingleOutputStreamOperator<SensorReading> sensorStream = stringDS.map(line -> {
            String[] strings = line.split(",");
            return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));

        });
        // 4.创建表执行环境
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
        // 5.基于流创建一张表
        Table sensorTable = tableEnv.fromDataStream(sensorStream);

        // 6.调用table API进行转换操作
        Table result1 = sensorTable.select($("id"), $("temperaturre")) //Expression
                .where($("id").isEqual("sensor_1"));

        // 7.执行Sal
        tableEnv.createTemporaryView("sensor",sensorStream);
        String sql = "select id,temperaturre from sensor where id = 'sensor_1'";
        Table tableResult = tableEnv.sqlQuery(sql);

        // 8.输出
        tableEnv.toAppendStream(result1, Row.class).print("API");
        tableEnv.toAppendStream(tableResult, Row.class).print("sql");

        env.execute();

    }
}

```

输出：

![image-20220121235838742](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220121235838742.png)

# 二、基本程序结构

```
// 创建表的执行环境
StreamTableEnvironment tableEnv = ... 

// 创建一张表，用于读取数据
tableEnv.connect(...).createTemporaryTable("inputTable");

// 注册一张表，用于把计算结果输出
tableEnv.connect(...).createTemporaryTable("outputTable");

// 通过 Table API 查询算子，得到一张结果表
Table result = tableEnv.from("inputTable").select(...);

// 通过SQL查询语句，得到一张结果表
Table sqlResult = tableEnv.sqlQuery("SELECT ... FROM inputTable ...");

// 将结果表写入输出表中
result.insertInto("outputTable");
```





# 三、Table API批处理和流处理

## 3.1 表环境配置

Blink planner和old planner有许多不同的特点，具体列举如下：

* Blink planner将批处理作业看做是流处理作业的特例。所以，不支持Table 与DataSet之间的转换，批处理的作业也不会被转成DataSet程序，而是被转为DataStream程序。
* Blink planner不支持 `BatchTableSource`，使用的是有界的StreamTableSource。
* Blink planner仅支持新的 `Catalog`，不支持`ExternalCatalog` (已过时)。
* 对于FilterableTableSource的实现，两种Planner是不同的。old planner会谓词下推到`PlannerExpression`(未来会被移除)，而Blink planner 会谓词下推到 `Expression`(表示一个产生计算结果的逻辑树)。
* 仅仅Blink planner支持key-value形式的配置，即通过Configuration进行参数设置。
* 关于PlannerConfig的实现，两种planner有所不同。
* Blink planner 会将多个sink优化成一个DAG(仅支持TableEnvironment，StreamTableEnvironment不支持)，old planner总是将每一个sink优化成一个新的DAG，每一个DAG都是相互独立的。
* old planner不支持catalog统计，Blink planner支持catalog统计。

```
    // 创建执行环境
    StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
    // 设置并行度为1
    env.setParallelism(1);

    //1.11后默认blink版本planner
    StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);


    // 1.1 基于老版本planner的流处理
    EnvironmentSettings oldStreamSettings = EnvironmentSettings.newInstance()
      .useOldPlanner()
      .inStreamingMode()
      .build();
    StreamTableEnvironment oldStreamTableEnv = StreamTableEnvironment.create(env,oldStreamSettings);

    // 1.2 基于老版本planner的批处理
    ExecutionEnvironment batchEnv = ExecutionEnvironment.getExecutionEnvironment();
    BatchTableEnvironment oldBatchTableEnv = BatchTableEnvironment.create(batchEnv);

    // 1.3 基于Blink的流处理
    EnvironmentSettings blinkStreamSettings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inStreamingMode()
      .build();
    StreamTableEnvironment blinkStreamTableEnv = StreamTableEnvironment.create(env,blinkStreamSettings);

    // 1.4 基于Blink的批处理
    EnvironmentSettings blinkBatchSettings = EnvironmentSettings.newInstance()
      .useBlinkPlanner()
      .inBatchMode()
      .build();
    TableEnvironment blinkBatchTableEnv = TableEnvironment.create(blinkBatchSettings);
  }


```



## 3.2 创建表

* TableEnvironment可以注册目录Catalog，并可以基于Catalog注册表
* **表(Table)是由一个"标示符"(identifier)来指定的，由3部分组成：Catalog名、数据库(database)名和对象名**（未定义，则是default）
* 表可以是常规的，也可以是虚拟的(视图，View)
* 常规表(Table)一般可以用来描述外部数据，比如文件、数据库表或消息队列的数据，也可以直接从DataStream转换而来
* 视图(View)可以从现有的表中创建，通常是table API或者SQL查询的一个结果集





* TableEnvironment可以调用`connect()`方法，连接外部系统，并调用`.createTemporaryTable()`方法，在Catalog中注册表

  ```java
  tableEnv
    .connect(...)    //    定义表的数据来源，和外部系统建立连接
    .withFormat(...)    //    定义数据格式化方法
    .withSchema(...)    //    定义表结构
    .createTemporaryTable("MyTable");    //    创建临时表
  ```



### 3.2.1 基于DataSteam流来创建

此时先创建的一个Table对象，如果使用Table APi操作的话就可以直接操作了，如果要使用Sql的方式则需要先注册成一个view然后再操作

```java
        Table sensorTable = tableEnv.fromDataStream(sensorStream);
```

```java
//SQL的方式 需要先将dataStream注册成一张表
tableEnv.createTemporaryView("sensor_table", mapDataStream);
```



### 3.2.2 基于connect+withFormat+withSchema方式

此时是先注册成一个view，如果使用SQL操作的话就可以直接操作了，如果要使用Api的方式则需要使用from语句获得Table对象

#### 3.2.2.1 从文件中创建表

```xml
<!-- csv -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-csv</artifactId>
  <version>${flink.version}</version>
</dependency>
```

```java
package com.mng.tableAPI;

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.DataTypes;
import org.apache.flink.table.api.EnvironmentSettings;
import org.apache.flink.table.api.Table;
import org.apache.flink.table.api.TableEnvironment;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.descriptors.Csv;
import org.apache.flink.table.descriptors.FileSystem;
import org.apache.flink.table.descriptors.Schema;
import org.apache.flink.types.Row;

public class TableTest {
    public static void main(String[] args) throws Exception{
        EnvironmentSettings settings = EnvironmentSettings.newInstance()
                .useBlinkPlanner()
                .inStreamingMode()
                .build();
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();


        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings);
        tableEnv.connect(new FileSystem().path("src/main/resources/wc.txt"))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("id", DataTypes.STRING())
                        .field("timestamp", DataTypes.BIGINT())
                        .field("temperaturre", DataTypes.DOUBLE())
                )
                .createTemporaryTable("inputTable");
        Table inputTable = tableEnv.from("inputTable");
     
        inputTable.printSchema();

        tableEnv.toAppendStream(inputTable, Row.class).print();
        env.execute();
    }
}
```

输出：

![image-20220124232847085](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220124232847085.png)

#### 3.2.2.2 从kafka中创建表

```java
        tableEnv.connect(new Kafka()
                    .version("universal")//Flink附带了提供了多个Kafka连接器：universal通用版本，0.10，0.11
                    .topic("sensor")
                    .startFromEarliest()
//                    .property("zookeeper.connect", "47.94.161.48:2181")
                    .property("bootstrap.servers", "47.94.161.48:9092"))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("id", DataTypes.STRING())
                        .field("timestamp", DataTypes.BIGINT())
                        .field("temperaturre", DataTypes.DOUBLE())
                )
                .createTemporaryTable("inputTable");
        Table inputTable = tableEnv.from("inputTable");
        inputTable.printSchema();
        tableEnv.toAppendStream(inputTable, Row.class).print();
```

kafka创建生产者并输入数据：

```
bin/kafka-console-producer.sh --broker-list 47.94.161.48:9092  --topic sensor
```

![image-20220125014248780](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220125014248780.png)

输出：

![image-20220125014322587](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220125014322587.png)

### 3.2.3 基于DDL建表语句，就是Create Table方式

此时和方式2一样，是先注册成一个view，如果使用SQL操作的话就可以直接操作了，如果要使用Api的方式则需要使用from语句获得Table对象

```
String sinkDDL=
"create table jdbcOutputTable (" +
" id varchar(20) not null, " +
" cnt bigint not null " +
") with (" +
" 'connector.type' = 'jdbc', " +
" 'connector.url' = 'jdbc:mysql://localhost:3306/test', " +
" 'connector.table' = 'sensor_count', " +
" 'connector.driver' = 'com.mysql.jdbc.Driver', " +
" 'connector.username' = 'root', " +
" 'connector.password' = '123456' )";
tableEnv.sqlUpdate(sinkDDL)
```



```

    /**
     * 官网API网址：https://ci.apache.org/projects/flink/flink-docs-release-1.11/zh/dev/table/connectors/kafka.html#how-to-create-a-kafka-table
     *
     *所有参数：
     * connector
     * topic
     * properties.bootstrap.servers
     * properties.group.id
     * format
     *  scan.startup.mode
     *  scan.startup.specific-offsets
     *  scan.startup.timestamp-millis
     *  sink.partitioner
     *
     */
    private static final String KAFKA_SQL = "CREATE TABLE kafkaTable (\n" +
            " code STRING," +
            " total_emp INT" +
            ") WITH (" +
            " 'connector' = 'kafka'," +
            " 'topic' = 'flink_dwd_test1'," +
            " 'properties.bootstrap.servers' = 'local:9092'," +
            " 'properties.group.id' = 'test1'," +
            " 'format' = 'json'," +
            " 'scan.startup.mode' = 'earliest-offset'" +
            ")";
————————————————
版权声明：本文为CSDN博主「黄瓜炖啤酒鸭」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_31866793/article/details/107312914
```





## 3.3 表的查询

### 3.3.1 table API

* Table API 是集成在 Scala 和 Java 语言内的查询 API 

* Table API 基于代表“表”的 Table 类，并提供一整套操作处理的方法 API；这些方法会返回一个新的 Table 对象，表示对输入表应用转换操作的结果

* 有些关系型转换操作，可以由多个方法调用组成，构成链式调用结构

```java
        
Table inputTable = tableEnv.from("inputTable");
Table sensor1Table = inputTable.select($("id"), $("temperaturre"))
                .where($("id").isEqual("serson_1"));
```



### 3.3.2 SQL

* Flink 的 SQL 集成，基于实现 了SQL 标准的 Apache Calcite 
* 在 Flink 中，用常规字符串来定义 SQL 查询语句 
* SQL 查询的结果，也是一个新的 Table

```java
        tableEnv.createTemporaryView("sensor",sensorStream);
        String sql = "select id,temperaturre from sensor where id = 'sensor_1'";
        Table tableResult = tableEnv.sqlQuery(sql);
```



## 3.4 输出表

* 表的输出，是通过将数据写入 TableSink 来实现的

* TableSink 是一个通用接口，可以支持不同的文件格式、存储数据库和消息队 列
* 输出表最直接的方法，就是通过 Table.insertInto() 方法将一个 Table 写入注册过的



### 3.4.1 输出到文件

```java
        tableEnv.connect(new FileSystem().path("src/main/resources/wc2.txt"))
                .withFormat(new Csv())
                .withSchema(new Schema()
                        .field("id",DataTypes.STRING())
                        //.field("timestamp",DataTypes.BIGINT())
                        .field("temperaturre",DataTypes.DOUBLE())
                )

                .createTemporaryTable("output                                                                                             Table");

        sensor1Table.insertInto("outputTable");
```



对于文件已经存在问题，尚不清楚overwrite属性如何设置

```java
org.apache.flink.core.fs.FileSystem.WriteMode.OVERWRITE
```

### 3.4.2 输出到kafka

```java
    // 4. 建立kafka连接，输出到不同的topic下
    tableEnv.connect(new Kafka()
                     .version("universal")
                     .topic("sinktest")
//                     .property("zookeeper.connect", "localhost:2181")
                     .property("bootstrap.servers", "localhost:9092")
                    )
      .withFormat(new Csv())
      .withSchema(new Schema()
                  .field("id", DataTypes.STRING())
                  .field("timestamp", DataTypes.BIGINT())
                  .field("temp", DataTypes.DOUBLE())
                 )
      .createTemporaryTable("outputTable");

    resultTable.insertInto("outputTable");
```





kafka生产者：

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092  --topic sensor
```

![image-20220125213441927](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220125213441927.png)

kafka消费者：

```shell
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic sinktest
```

![image-20220125213409141](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220125213409141.png)



### 3.4.2 输出到ES

* 可以创建Table来描述ES中的数据，作为输出的TableSink

  ```java
  tableEnv.connect(
    new Elasticsearch()
    .version("6")
    .host("localhost",9200,"http")
    .index("sensor")
    .documentType("temp")
  )
    .inUpsertMode()
    .withFormat(new Json())
    .withSchema(new Schema()
                .field("id",DataTypes.STRING())
                .field("count",DataTypes.BIGINT())
               )
    .createTemporaryTable("esOutputTable");
  aggResultTable.insertInto("esOutputTable");
  ```

### 3.4.3 输出到MySQL

* 需要的pom依赖

  Flink专门为Table API的jdbc连接提供了flink-jdbc连接器，需要先引入依赖

  ```xml
  <!-- https://mvnrepository.com/artifact/org.apache.flink/flink-jdbc -->
  <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-jdbc_2.12</artifactId>
      <version>1.10.3</version>
      <scope>provided</scope>
  </dependency>
  ```

* 可以创建Table来描述MySql中的数据，作为输入和输出

  ```java
  String sinkDDL = 
    "create table jdbcOutputTable (" +
    " id varchar(20) not null, " +
    " cnt bigint not null " +
    ") with (" +
    " 'connector.type' = 'jdbc', " +
    " 'connector.url' = 'jdbc:mysql://localhost:3306/test', " +
    " 'connector.table' = 'sensor_count', " +
    " 'connector.driver' = 'com.mysql.jdbc.Driver', " +
    " 'connector.username' = 'root', " +
    " 'connector.password' = '123456' )";
  tableEnv.sqlUpdate(sinkDDL);    // 执行DDL创建表
  aggResultSqlTable.insertInto("jdbcOutputTable");
  ```



## 3.5 更新模式

* 对于流式查询，需要声明如何在表和外部连接器之间执行转换
* 与外部系统交换的消息类型，由更新模式（Uadate Mode）指定
* 追加（Append）模式
  * 表只做插入操作，和外部连接器只交换插入（Insert）消息
* 撤回（Retract）模式
  * 表和外部连接器交换添加（Add）和撤回（Retract）消息
  * 插入操作（Insert）编码为Add消息；删除（Delete）编码为Retract消息；**更新（Update）编码为上一条的Retract和下一条的Add消息**
* 更新插入（Upsert）模式
  * 更新和插入都被编码为Upsert消息；删除编码为Delete消息



## 3.6 表和流的转换

### 3.6.1 将Table转化为DataStream

* 表可以转换为 DataStream 或 DataSet ，这样自定义流处理或批处理程序就 可以继续在 Table API 或 SQL 查询的结果上运行了
* 将表转换为 DataStream 或 DataSet 时，需要指定生成的数据类型，即要将 表的每一行转换成的数据类型
* 表作为流式查询的结果，是动态更新的
* 转换有两种转换模式：追加（Append）模式和撤回（Retract）模式



* 追加模式（Append Mode） 

– 用于表只会被插入（Insert）操作更改的场景 

```
DataStream resultStream = tableEnv.toAppendStream(resultTable, Row.class); 
```

*  撤回模式（Retract Mode） 

– 用于任何场景。有些类似于更新模式中 Retract 模式，它只有 Insert 和 Delete 两类操作。

 – 得到的数据会增加一个 Boolean 类型的标识位（返回的第一个字段），用它来表示到底是 新增的数据（Insert），还是被删除的数据（Delete） 

```
DataStream aggResultStream = tableEnv.toRetractStream(aggResultTable , Row.class);
```



### 3.6.2将DataStream转换成表

* 对于一个DataStream，可以直接转换成Table，进而方便地调用Table API做转换操作

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream);
  ```

* 默认转换后的Table schema和DataStream中的字段定义一一对应，也可以单独指定出来

  ```java
  DataStream<SensorReading> dataStream = ...
  Table sensorTable = tableEnv.fromDataStream(dataStream,
                                             "id, timestamp as ts, temperature");
  ```



## 3.7 创建临时视图(Temporary View)

* 基于DataStream创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView",dataStream);
  tableEnv.createTemporaryView("sensorView",
                              dataStream, "id, timestamp as ts, temperature");
  ```

* 基于Table创建临时视图

  ```java
  tableEnv.createTemporaryView("sensorView", sensorTable);
  ```



## 3.9 查看执行计划

* Table API 提供了一种机制来解释计算表的逻辑和优化查询计划

* 查看执行计划，可以通过`TableEnvironment.explain(table)`方法或`TableEnvironment.explain()`方法完成，返回一个字符串，描述三个计划

  * 优化的逻辑查询计划
  * 优化后的逻辑查询计划
  * 实际执行计划

  ```java
  String explaination = tableEnv.explain(resultTable);
  System.out.println(explaination);
  ```





## 3.10 流处理和关系代数的区别

​	Table API和SQL，本质上还是基于关系型表的操作方式；而关系型表、关系代数，以及SQL本身，一般是有界的，更适合批处理的场景。这就导致在进行流处理的过程中，理解会稍微复杂一些，需要引入一些特殊概念。

​	可以看到，其实**关系代数（主要就是指关系型数据库中的表）和SQL，主要就是针对批处理的，这和流处理有天生的隔阂。**

|                           | 关系代数(表)/SQL           | 流处理                                       |
| ------------------------- | -------------------------- | -------------------------------------------- |
| 处理的数据对象            | 字段元组的有界集合         | 字段元组的无限序列                           |
| 查询（Query）对数据的访问 | 可以访问到完整的数据输入   | 无法访问所有数据，必须持续"等待"流式输入     |
| 查询终止条件              | 生成固定大小的结果集后终止 | 永不停止，根据持续收到的数据不断更新查询结果 |

## 3.11 动态表(Dynamic Tables)

​	我们可以**随着新数据的到来，不停地在之前的基础上更新结果**。这样得到的表，在Flink Table API概念里，就叫做“动态表”（Dynamic Tables）。

+ 动态表是 Flink 对流数据的 Table API 和 SQL 支持的核心概念
+ 与表示批处理数据的静态表不同，动态表是随时间变化的
+ 持续查询(Continuous Query)
  + 动态表可以像静态的批处理表一样进行查询，查询一个动态表会产生**持续查询（Continuous Query）**
  + **连续查询永远不会终止，并会生成另一个动态表**
  + 查询（Query）会不断更新其动态结果表，以反映其动态输入表上的更改。

### 3.11.1 动态表和持续查询

![在这里插入图片描述](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20200601190927869.png)

流式表查询的处理过程：

1. 流被转换为动态表

2. 对动态表计算连续查询，生成新的动态表

3. 生成的动态表被转换回流

### 3.11.2 将流转换成动态表

+ 为了处理带有关系查询的流，必须先将其转换为表
+ 从概念上讲，流的每个数据记录，都被解释为对结果表的插入（Insert）修改操作

*本质上，我们其实是从一个、只有插入操作的changelog（更新日志）流，来构建一个表*

*来一条数据插入一条数据*

![在这里插入图片描述](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20200601191016684.png)

### 3.11.3 持续查询

+ 持续查询，会在动态表上做计算处理，并作为结果生成新的动态表。

  *与批处理查询不同，连续查询从不终止，并根据输入表上的更新更新其结果表。*

  *在任何时间点，连续查询的结果在语义上，等同于在输入表的快照上，以批处理模式执行的同一查询的结果。*

​	下图为一个点击事件流的持续查询，是一个分组聚合做count统计的查询。

![在这里插入图片描述](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20200601191112183.png)

### 3.11.4 将动态表转换成 DataStream

+ 与常规的数据库表一样，动态表可以通过插入（Insert）、更新（Update）和删除（Delete）更改，进行持续的修改
+ **将动态表转换为流或将其写入外部系统时，需要对这些更改进行编码**

---

+ 仅追加（Append-only）流

  + 仅通过插入（Insert）更改来修改的动态表，可以直接转换为仅追加流

+ 撤回（Retract）流

  + 撤回流是包含两类消息的流：添加（Add）消息和撤回（Retract）消息

    *动态表通过将INSERT 编码为add消息、DELETE 编码为retract消息、UPDATE编码为被更改行（前一行）的retract消息和更新后行（新行）的add消息，转换为retract流。*

+ Upsert（更新插入流）

  + Upsert流也包含两种类型的消息：Upsert消息和删除（Delete）消息

    *通过将INSERT和UPDATE更改编码为upsert消息，将DELETE更改编码为DELETE消息，就可以将具有唯一键（Unique Key）的动态表转换为流。*

### 3.11.5 将动态表转换成DataStream

![在这里插入图片描述](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20200601191549610.png)

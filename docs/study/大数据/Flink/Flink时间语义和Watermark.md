# Flink时间语义和Watermark

# 一、时间语义和Watermark

## 1.1 Flink中的时间语义

![image-20220102131909104](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220102131909104.png)

Event Time：事件创建的时间,一般使用更多。

Ingestion Time：数据进入Flink的时间

Processing Time：执行操作算子的本地系统时间



## 1.2 EventTime的引入

 **在Flink的流式处理中，绝大部分的业务都会使用eventTime**，一般只在eventTime无法使用时，才会被迫使用ProcessingTime或者IngestionTime。

默认环境里使用的是Processing,如果要使用EventTime,那么需要引入EventTime的时间属性。

新版

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
// 从调用时刻开始给env创建的每一个stream追加时间特征
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```



## 1.3 Watermark

### 1.3.1 乱序数据

* flink以Event Time模式处理数据流时，会根据数据中的时间戳处理基于时间的算子
* 由于网络、分布式等原因，会导致乱序数据的产生
* 乱序数据会让窗口计算不准确

![image-20220103192601355](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220103192601355.png)



Flink对于迟到数据有三层保障,先来后到的保障顺序是：

* WaterMark => 约等于放宽窗口标准
* allowedLateness => 允许迟到（ProcessingTime超时，但是EventTime没超时）
* sideOutputLateData => 超过迟到时间，另外捕获，之后可以自己批处理合并先前的数据



### 1.3.2 水位线（watermark）

* 遇到一个时间戳到达了窗口关闭时间，不应该立刻触发窗口计算，而是等待一段时间，等迟到的数据来了再关闭窗口

* Watermark是一种衡量Event Time进展的机制，可以设定延迟触发
* Watermark是用于处理乱序事件的，正确的处理乱序事件，通常用Watermark机制结合window来实现
* watermark用来让程序自己平衡延迟和结果正确性

定义：WaterMark是一种特殊的时间戳，它会被插入到数据流中，用于表示EventTime小于Watermark的事件全部落入到了相应的窗口中。**如果有窗口的停止时间等于maxEventTime – t，那么这个窗口被触发执行。**

例如：延迟时间为2s,当event time为12的数据到来时，生成了event time - t = 10的watermark ,系统认为所有event time<=10的数据到达，所以关闭5-10的窗口.

![image-20220103194149936](https://gitee.com/minan-palace/md_images/raw/master/images/image-20220103194149936.png)





## 1.4 Watermark的引入

调用 assignTimestampAndWatermarks 方法，传入一个

*  BoundedOutOfOrdernessTimestampExtractor，可以指定延迟时间和时间戳。
* AscendingTimestampExtractor,，可以指定时间戳。





## 1.5 TimestampAssigner

Flink 暴露了 TimestampAssigner 接口供我们实现，使我们可以自定义如何从事件数据中抽取时间戳和生成watermark 

MyAssigner 可以有两种类型，都继承自 TimestampAssigner，TimestampAssigner 定义了抽取时间戳，以及生成 watermark 的方法，有两种类型 

*  AssignerWithPeriodicWatermarks：
  * 周期性的生成 watermark：系统会周期性的将 watermark 插入到流中 
  * 默认周期是200毫秒，可以使用 ExecutionConfig.setAutoWatermarkInterval() 方法进行设置 
  * 升序和前面乱序的处理 
  * BoundedOutOfOrdernessTimestampExtractor， 都是基于周期性 watermark 的。
* AssignerWithPunctuatedWatermarks 
  * 没有时间周期规律，可打断的生成 watermark







## 1.6 代码示例

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










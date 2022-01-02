# Flink Window API

# 一、window 概念

窗口（window）就是将无限流切割为有限流的一种方式，它会将流数据分发到有限大小的桶（bucket）中进行分析。



# 二、window 类型

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





# 三、window API

* 窗口分配器----window方法

* 可以用.window()来定义一个窗口，再做一些聚合或者其他处理操作，window()方法必须在keyBy之后才能用。
* Flink提供了更简单的.timeWindow和.countWindow方法，用于定义时间窗口和计数窗口。

> PS:时间分为处理时间（Processing time 执行算子操作的机器系统时间）、事件时间（Event time 每个单独的事件在其生产设备上发生的时间，通常内置于数据记录中）

## 3.1 创建不同类型的窗口

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



## 3.2 窗口函数

window function定义了要对窗口中收集的数据做的计算操作

分为：

* 增量聚合函数：每条数据到来就进行计算

  有ReduceFunction,AggregateFunction

* 全窗口函数：先把窗口所有数据收集起来，计算时遍历所有数据

  ProcessWindowFunction,WindowFunction
  
  ProcessWindowFunction获得一个包含窗口所有元素的可迭代器，以及一个具有时间和状态信息访问权的上下文对象，这使得它比其他窗口函数提供更大的灵活性。这是以性能和资源消耗为代价的，因为元素不能增量地聚合，而是需要在内部缓冲，直到认为窗口可以处理为止，使用该函数需要注意数据量，数据量太大，全量数据保存在内存中，会造成内存溢出。
  

## 3.3 其他API

* .trigger  ——  触发器

  定义window什么时候关闭，出发计算并输出结果

* .evictor——移除器

  定义移除某些数据的逻辑

* .allowedLateness——允许处理迟到的数据

* .sideOutputLateData——将迟到的数据放入侧输出流

* .getSideOutput——获取侧输出流



# 四、代码示例

## 4.1 滚动时间窗口的增量聚合函数

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









## 4.2 滑动时间窗口的全局聚合函数

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
























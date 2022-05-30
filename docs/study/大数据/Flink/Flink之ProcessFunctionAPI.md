# Flink之ProcessFunctionAPI

 我们之前学习的**转换算子**是无法访问事件的时间戳信息和水位线信息的。而这在一些应用场景下，极为重要。例如MapFunction这样的map转换算子就无法访问时间戳或者当前事件的事件时间。

 基于此，DataStream API提供了一系列的Low-Level转换算子。可以**访问时间戳**、**watermark**以及**注册定时事件**。还可以输出**特定的一些事件**，例如超时事件等。Process Function用来构建事件驱动的应用以及实现自定义的业务逻辑(使用之前的window函数和转换算子无法实现)。例如，FlinkSQL就是使用Process Function实现的。

Flink提供了8个Process Function：

* ProcessFunction
* KeyedProcessFunction
* CoProcessFunction
* ProcessJoinFunction
* BroadcastProcessFunction
* KeyedBroadcastProcessFunction
* ProcessWindowFunction
* ProcessAllWindowFunction

# 一、KeyedProcessFunction

 这个是相对比较常用的ProcessFunction，根据名字就可以知道是用在keyedStream上的。

 KeyedProcessFunction用来操作KeyedStream。KeyedProcessFunction会处理流的每一个元素，输出为0个、1个或者多个元素。所有的Process Function都继承自RichFunction接口，所以都有`open()`、`close()`和`getRuntimeContext()`等方法。而`KeyedProcessFunction<K, I, O>`还额外提供了两个方法:

* `processElement(I value, Context ctx, Collector<O> out)`，流中的每一个元素都会调用这个方法，调用结果将会放在Collector数据类型中输出。Context可以访问元素的时间戳，元素的 key ，以及TimerService 时间服务。 Context 还可以将结果输出到别的流(side outputs)。

  ![image-20220118010303365](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220118010303365.png)

  timeservice可以获得时间或者注册删除触发器（定时间）

* `onTimer(long timestamp, OnTimerContext ctx, Collector<O> out)`，是一个回调函数。当之前注册的定时器触发时调用。参数timestamp 为定时器所设定的触发的时间戳。Collector 为输出结果的集合。OnTimerContext和processElement的Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息(事件时间或者处理时间)





## 测试代码

```java
package com.mng.processFun;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class KeyedProcessFunTest {
    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> stringDS = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<SensorReading> sensorDs = stringDS.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
                String[] strings = s.split(",");
                return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));
            }
        });

        sensorDs.keyBy(sensorReading -> sensorReading.getId())
                .process(new MyProcess())
                .print();
        env.execute();
    }
}

class MyProcess extends KeyedProcessFunction<String,SensorReading,String>{
    @Override
    public void processElement(SensorReading sensorReading, Context context, Collector<String> collector) throws Exception {
        collector.collect(context.getCurrentKey());
        System.out.println(context.timerService().currentProcessingTime());
        context.timerService().registerProcessingTimeTimer(context.timerService().currentProcessingTime()+5000L);
    }
    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        super.onTimer(timestamp, ctx, out);
        out.collect(String.valueOf(timestamp));
    }
}
```

输入 

```
sensor_1,1547718207,36.3
```

输出

![image-20220118011017681](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220118011017681.png)



# 二、TimeService和定时器（Timer）

 Context 和OnTimerContext 所持有的TimerService 对象拥有以下方法：

* `long currentProcessingTime()` 返回当前处理时间
* `long currentWatermark()` 返回当前watermark 的时间戳
* `void registerProcessingTimeTimer( long timestamp)` 会注册当前key的processing time的定时器。当processing time 到达定时时间时，触发timer。
* **`void registerEventTimeTimer(long timestamp)` 会注册当前key 的event time 定时器。当Watermark水位线大于等于定时器注册的时间时，触发定时器执行回调函数。**
* `void deleteProcessingTimeTimer(long timestamp)` 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
* `void deleteEventTimeTimer(long timestamp)` 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。

 **当定时器timer 触发时，会执行回调函数onTimer()。注意定时器timer 只能在keyed streams 上面使用。**





## 测试代码

```java
package com.mng.processFun;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.time.Time;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;

public class TimerTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> stringDs = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<SensorReading> sensorDs = stringDs.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
                String[] strings = s.split(",");
                return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));
            }
        });
        sensorDs.keyBy(sensorReading -> sensorReading.getId())
                .process(new MyProcessFun(Time.seconds(10).toMilliseconds()))
                .print();
        env.execute();

    }
}

class MyProcessFun extends KeyedProcessFunction<String, SensorReading , String>{

    //检查时间段
    private Long checkTime;
    //上一次温度
    ValueState<Double> lastTemp;
    //触发器的时间
    ValueState<Long>  recentTimer;
    public MyProcessFun(Long checkTime) {
        this.checkTime = checkTime;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        lastTemp = getRuntimeContext().getState(new ValueStateDescriptor<Double>("lastTem",Double.class));
        recentTimer = getRuntimeContext().getState(new ValueStateDescriptor<Long>("reTimer",Long.class));
    }

    @Override
    public void close() throws Exception {
        super.close();
        lastTemp.clear();
        recentTimer.clear();
    }

    @Override
    public void processElement(SensorReading sensorReading, Context context, Collector<String> collector) throws Exception {
        Double currTempV = sensorReading.getTemperaturre();
        Double lastTempV = lastTemp.value() != null? lastTemp.value() : currTempV;

        //如果温度升高且没有设置触发器
        //那么设置触发器并更新触发器时间状态
        if(currTempV > lastTempV && recentTimer.value() == null){
            context.timerService().registerProcessingTimeTimer(
                    context.timerService().currentProcessingTime()+checkTime);
            recentTimer.update(context.timerService().currentProcessingTime()+checkTime);

        }
        //如果温度不上升，那么删除触发器，并清空状态
        else if(currTempV <= lastTempV && recentTimer.value() != null){
            context.timerService().deleteProcessingTimeTimer(recentTimer.value());
            recentTimer.clear();
        }
        //更新上一次温度
        lastTemp.update(currTempV);

    }
    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<String> out) throws Exception {
        // 触发报警，并且清除 定时器状态值
        out.collect("传感器" + ctx.getCurrentKey() + "温度值连续" + checkTime + "ms上升");
        recentTimer.clear();
    }
}

```



输入

```
sensor_1,1547718199,35.8
sensor_1,1547718199,34.1
sensor_1,1547718199,34.2
sensor_1,1547718199,35.1
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
sensor_10,1547718205,39  
sensor_6,1547718201,18  
sensor_7,1547718202,9.1
```

输出

![image-20220118230823416](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220118230823416.png)



# 三、侧输出流（SideOutPut）

* **一个数据可以被多个window包含，只有其不被任何window包含的时候(包含该数据的所有window都关闭之后)，才会被丢到侧输出流。**
* **简言之，如果一个数据被丢到侧输出流，那么所有包含该数据的window都由于已经超过了"允许的迟到时间"而关闭了，进而新来的迟到数据只能被丢到侧输出流！**

------

* 大部分的DataStream API 的算子的输出是单一输出，也就是某种数据类型的流。除了split 算子，可以将一条流分成多条流，这些流的数据类型也都相同。
* **processfunction 的side outputs 功能可以产生多条流，并且这些流的数据类型可以不一样。**
* 一个side output 可以定义为OutputTag[X]对象，X 是输出流的数据类型。
* processfunction 可以通过Context 对象发射一个事件到一个或者多个side outputs。



## 测试代码

场景：**高低温分流**

```java
package com.mng.processFun;

import com.mng.dto.SensorReading;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.time.Time;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;

public class SideOutPutTest {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        DataStreamSource<String> stringDs = env.socketTextStream("localhost", 7777);
        SingleOutputStreamOperator<SensorReading> sensorDs = stringDs.map(new MapFunction<String, SensorReading>() {
            @Override
            public SensorReading map(String s) throws Exception {
                String[] strings = s.split(",");
                return new SensorReading(strings[0], Long.parseLong(strings[1]), Double.parseDouble(strings[2]));
            }
        });

        OutputTag<SensorReading> lowTempTag = new OutputTag<SensorReading>("lowTemp"){};

        SingleOutputStreamOperator<SensorReading> highSensorDs = sensorDs.keyBy(sensorReading -> sensorReading.getId())
                .process(new KeyedProcessFunction<String, SensorReading, SensorReading>() {
                    @Override
                    public void processElement(SensorReading sensorReading, Context context, Collector<SensorReading> collector) throws Exception {
                        if (sensorReading.getTemperaturre() > 30) {
                            collector.collect(sensorReading);
                        } else {
                            context.output(lowTempTag, sensorReading);
                        }
                    }
                });
        highSensorDs.print("high");
        highSensorDs.getSideOutput(lowTempTag).print("low");
        env.execute();
    }
}


```

输入

```
sensor_1,1547718199,35.8
sensor_6,1547718201,15.4
sensor_7,1547718202,6.7
sensor_10,1547718205,38.1
```

输出

![image-20220119000340808](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220119000340808.png)



# 四、CoProcessFunction 

* 对于两条输入流，DataStream API 提供了CoProcessFunction 这样的low-level操作。CoProcessFunction 提供了操作每一个输入流的方法: `processElement1()`和`processElement2()`。
* **类似于ProcessFunction，这两种方法都通过Context 对象来调用**。这个Context对象可以访问事件数据，定时器时间戳，TimerService，以及side outputs。
* **CoProcessFunction 也提供了onTimer()回调函数**。

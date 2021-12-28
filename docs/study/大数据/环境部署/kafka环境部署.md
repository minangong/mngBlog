

# kafka环境部署



# 一、环境

Linux系统版本：CentOS Linux release 7.6.1810

jdk版本：[jdk-8u181-linux-x64](https://repo.huaweicloud.com/java/jdk/8u181-b13/)

zookeeper: [zookeeper-3.4.9](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.9/)

kafka:[kafka_2.12-2.8.1](https://kafka.apache.org/downloads.html)



# 二、安装步骤

## 1.安装包上传并解压

使用xftp将kafka_2.12-2.8.1.tgz传入服务器的/usr/local/mng目录下，并解压

```shell
//解压
tar -zxvf /usr/local/mng/kafka_2.12-2.8.1.tgz
//改名
mv /usr/local/mng/kafka_2.12-2.8.1.tgz /usr/local/mng/kafka
```



## 2.参数配置

```shell
#broker 的全局唯一编号，不能重复
broker.id=0
#删除 topic 功能使能
delete.topic.enable=true
#配置连接 Zookeeper 集群地址
zookeeper.connect=localhost:2181,.....
```



## 3.启动kafka

```shell
bin/kafka-server-start.sh -daemon config/server.properties
```

后台运行：ctrl+z，然后bg,回车

![image-20211228194940605](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211228194940605.png)



## 4.kafka使用

1）查看当前服务器中的所有 topic 

```
bin/kafka-topics.sh --zookeeper  localhost:2181 --list 
```

2）创建 topic 

```
bin/kafka-topics.sh --create --partitions 2 --replication-factor 1 --topic flinktest --zookeeper localhost:2181
```

--topic 定义 topic 名 

--replication-factor 定义副本数

 --partitions 定义分区数

3）删除 topic 

```
bin/kafka-topics.sh --zookeeper  localhost:2181 --delete --topic flinktest
```

4）发送消息 

```
bin/kafka-console-producer.sh --broker-list  localhost:9092 --topic flinktest
```

5）消费消息

```
bin/kafka-console-consumer.sh  --zookeeper localhost:2181 --topic flinktest --from-beginning 
```

--from-beginning 会把主题中以往所有的数据都读取出来。 

6）查看某个 Topic 的详情

```
bin/kafka-topics.sh --zookeeper  localhost:2181 --describe --topic flinktest
```

7）修改分区数 

```
bin/kafka-topics.sh --zookeeper  localhost:2181 --alter --topic flinktest --partitions 6
```

# 三、配置环境变量

```
vi ~/.bashrc
```

将下面几行加入最下面(PATH添加在PATH后面)

```
export CCP_HOME=/usr/local/mng
export KAFKA_HOME=$CCP_HOME/kafka
export PATH=$PATH:......:$KAFKA_HOME
```

让配置文件生效

```
source ~/.bashrc
```


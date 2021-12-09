# 一、环境

Linux系统版本：CentOS Linux release 7.6.1810

jdk版本：[jdk-8u181-linux-x64](https://repo.huaweicloud.com/java/jdk/8u181-b13/)

flink: [flink-1.14.0-bin-scala_2.12.tgz](https://flink.apache.org/zh/downloads.html)

hadoop支持:  flink-shaded-hadoop-2-uber-2.8.3-10.0.jar

# 二、安装步骤

## 1.减压安装包

```
//解压
tar -zxvf flink-1.14.0-bin-scala_2.12.tgz
//将hadoop支持jar包 导入flink的lib目录下
mv flink-shaded-hadoop-2-uber-2.8.3-10.0.jar  flink/lib/flink-shaded-hadoop-2-uber-2.8.3-10.0.jar
```

## 2. 配置文件解读

### conf/flink-conf.yaml

![image-20211205233649197](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211205233649197.png)

jobmanager进行任务的调度，而taskmanager完成工作。所以taskmanager的内存应该更大。

![image-20211206212521815](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206212521815.png)

最后的numberOfTaskSlots一个taskmanager（一台机器）的线程数。建议改成核心数。parallelism.default: 1，默认的线程数

### conf/masters

![image-20211206000047387](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206000047387.png)

jobmanager。进行任务提交调度的。

### conf/workers

![image-20211206000207629](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206000207629.png)

taskmanager

## 3. 配置环境变量

```
vi ~/.bashrc
```

将下面几行加入最下面(PATH添加在PATH后面)

```
export CCP_HOME=/usr/local/mng
export FLINK_HOME=$CCP_HOME/flink
export PATH=$PATH:......:$FLINK_HOME/bin
```

让配置文件生效

```
source ~/.bashrc
```

## 3.启动集群

```
start-cluster.sh
```

![image-20211206001610282](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206001610282.png)

```
flink dashboard界面  可以修改flink-conf.yaml中的rest.port来修改端口  默认为8081端口  

# 防火墙开启端口
firewall-cmd --zone=public --add-port=8081/tcp --permanent
#防火墙重新加载配置
firewall-cmd --reload
#查看开启的端口
firewall-cmd --list-ports
```

![image-20211206212626302](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206212626302.png)



# 三、提交Job

## web ui提交

将java程序编译打包后，

![image-20211206212100199](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206212100199.png)

设置属性后，运行，如果任务需要的Parallelism（各部分最大并行数），大于slot数，会超时报错，缺少资源（如果有socket 提前开放端口 并打开socket）（StreamWordCount --host localhost --port 7777）

![image-20211206213831955](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206213831955.png)

job running

![image-20211206214305536](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206214305536.png)

socket传递string（回退 ctrl + backspace）

实则传了  hello world   hello  mng

![image-20211206214631776](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206214631776.png)

结果：

![image-20211206221551528](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206221551528.png)

![image-20211206221343550](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206221343550.png)



## 命令行提交

```
./bin/flink  run -c com.atguigu.wc.StreamllordCount -p 3 /mnt/d/Projects/BigData/FlinkTutorial/target/FlinkTutorial-1.0-SNAPSHOT.jar  --host localhost --port 7777
```

![image-20211206222446451](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206222446451.png)



取消

```
flink list //列出jobid
flink cancel jobid //删除任务

```



# 四、yarn、k8s部署

## 1.yarn

### 1.1 Session-cluster模式

![image-20211206230932314](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206230932314.png)

-n没什么用了, 动态分配，可以不停提交Job

![image-20211206231352178](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231352178.png)

**有yarn session，默认向yarn提交，不然默认是standslone模式**

![image-20211206231548181](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231548181.png)

![image-20211206231701151](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231701151.png)

### 1.2 Job

![image-20211206231111713](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231111713.png)

![image-20211206231732341](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231732341.png)



## k8s

![image-20211206231839518](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231839518.png)

![image-20211206231956027](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211206231956027.png)


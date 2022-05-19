# hadoop环境部署

# 一、环境

Linux系统版本：CentOS Linux release 7.6.1810

jdk版本：[jdk-8u181-linux-x64](https://repo.huaweicloud.com/java/jdk/8u181-b13/)

zookeeper: [zookeeper-3.4.9](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.9/)

hadoop:[hadoop-3.2.2](https://dlcdn.apache.org/hadoop/common/hadoop-3.2.2/)

# 二、安装步骤

## 2.1 防火墙开启端口

防火墙开启2888、3888、2181、9000、50070、8485、50010端口

```
#查看防火墙状态 
firewall-cmd --state
# 打开防火墙
systemctl start firewalld.service
# 防火墙开机自启
systemctl enable firewalld.service
# 重启防火墙
systemctl restart firewalld.service
# 防火墙开启端口
firewall-cmd --zone=public --add-port=80/tcp --permanent

#防火墙重新加载配置
firewall-cmd --reload
#查看开启的端口
firewall-cmd --list-ports
#开启伪装ip
firewall-cmd --zone=public --add-masquerade --permanent
```

![image-20211010100416018](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010100416018.png)



## 2.2安装包上传并解压

使用xftp 将hadoop-3.2.2.tar.gz传入服务器的/usr/local/mng目录下，并解压

```
//解压
tar -zxvf /usr/local/mng/hadoop-3.2.2.tar.gz
//改名
mv /usr/local/mng/hadoop-3.2.2  /usr/local/mng/hadoop
```

## 2.3 相关参数配置

[Apache Hadoop 3.1.2 – Hadoop Cluster Setup](https://hadoop.apache.org/docs/r3.1.2/hadoop-project-dist/hadoop-common/ClusterSetup.html)

配置文件都在/usr/local/mng/hadoop/etc/hadoop中

### 2.3.1  core-site.xml

core-site.xml：配置通用属性，例如HDFS和MapReduce常用的I/O设置等

修改内容为：

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/usr/local/mng/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://master:9000</value>
    </property>
    <property>
        <name>io.file.buffer.size</name>
        <value>131072</value>
    </property>
</configuration>
```

参数说明：

* fs.defaultFS：默认文件系统，HDFS的客户端访问HDFS需要此参数
* hadoop.tmp.dir：指定Hadoop数据存储的临时目录，其它目录会基于此路径, 建议设置到一个足够空间的地方，而不是默认的/tmp下

### 2.3.2 hdfs-site.xml

```xml

<configuration>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>master:50070</value>
    </property>
    <property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>master:9000</value>
    </property>

    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/usr/local/mng/hadoop/dfs/name</value>
    </property>
 
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/usr/local/mng/hadoop/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.webhdfs.enabled</name>
        <value>true</value>
    </property>
</configuration>
```

参数说明：

* dfs.replication：数据块副本数
* dfs.namenode.name.dir：指定namenode节点的文件存储目录
* dfs.datanode.data.dir：指定datanode节点的文件存储目录
* dfs.namenode.secondary.http-address: The secondary namenode http server address and port.



### 2.3.3 mapred-site.xml

```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.address</name>
        <value>master:10020</value>
    </property>
    <property>
        <name>mapreduce.jobhistory.webapp.address</name>
        <value>master:50030</value>
    </property>
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>4096</value>
    </property>
</configuration>
```

### 2.3.4 workers

```
master
```

### 2.3.5 yarn-site.xml

```xml
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>master</value>
    </property>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value> 
    </property>
	<property>
		<name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
		<value>org.apache.hadoop.mapred.ShuffleHandler</value>
	</property>
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
</configuration>
```

### 2.3.6 hadoop-env.sh/yarn-env.sh

```
export JAVA_HOME=/usr/local/mng/java8
```

### 2.3.7 hadoop/sbin目录下start-dfs.sh ，stop-dfs.sh

添加配置

```
HDFS_DATANODE_USER=root
HDFS_DATANODE_SECURE_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root
```



### 2.3.8 hadoop/sbin目录下start-yarn.sh，stop-yarn.sh

```
YARN_RESOURCEMANAGER_USER=root
HADOOP_SECURE_DN_USER=yarn
YARN_NODEMANAGER_USER=root
```

# 三、启动hadoop的dfs和yarn

```
问题1：Cannot set priority of namenode process 9734
解决方法：在$HADOOP_HOME/etc/hadoop/hadoop-env.sh最后一行加上HADOOP_SHELL_EXECNAME=root
        vi $HADOOP_HOME/bin/hdfs，有个HADOOP_SHELL_EXECNAME="hdfs"，将hdfs改为root
        
问题2: but there is no ...... defined. Aborting operation
解决方法： 上文2.3.7和2.3.8

问题3：jps发现只有SecondaryNameNode，没有NameNode
解决方法： 重新namenode初始化，bin/hadoop namenode -format,之前有错，初始化不成功。      

```

```
//进入hadoop目录
bin/hadoop namenode -format
sbin/start-yarn.sh
sbin/start-yarn.sh
```



![image-20211010193449982](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010193449982.png)

Datanode、SecondaryNameNode是dfs的进程

NodeManager、ResourceManager是yarn的进程

# 四. 配置环境变量

```
vi ~/.bashrc
```

将下面几行加入最下面(PATH添加在PATH后面)

```
export HADOOP_HOME=$CCP_HOME/hadoop
export PATH=$PATH:$ZOOKEEPER_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

让配置文件生效

```
source ~/.bashrc
```


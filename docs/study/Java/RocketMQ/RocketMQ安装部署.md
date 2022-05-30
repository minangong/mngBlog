# RocketMQ安装部署

`ps:单机安装部署`

# 一、RocketMQ安装
## 1.安装包下载
[Apache RocketMQ  4.9.2](https://rocketmq.apache.org/release_notes/release-notes-4.9.2/)

## 2.安装包上传并解压
`ps:安装unzip`
```shell
yum install -y unzip
```

```shell
unzip rocketmq-all-4.9.2-bin-release.zip
```
## 3.修改配置
修改JAVA_HOME及Xms,Xmx,Xmn等内存配置，默认最小4G
```shell
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```
### 3.1 runserver.sh
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220209143633.png)

修改后：
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220209174541.png)


### 3.2 runbroker.sh
修改后：
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220209174430.png)


## 3.3 broker.conf
修改conf下的配置文件broker.conf，并修改启动命令
```
vi conf/broker.conf  
```
添加或者修改下面的两行
```
namesrvAddr=IP:9876  
brokerIP1=IP
```
启动broker命令加上
```
-c conf/broker.conf
```
## 4.启动
启动NameServer
```shell
nohup sh bin/mqnamesrv & tail -f ~/logs/rocketmqlogs/namesrv.log
```
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220209181136.png)


启动broker

```shell
nohup sh bin/mqbroker -n localhost:9876 -c conf/broker.conf & tail -f ~/logs/rocketmqlogs/broker.log
```


![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220209181256.png)


## 5.发送/接收消息测试

发送消息
```shell
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

接收消息

```shell
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```


## 6.关闭Server

无论是关闭name server还是broker，都是使用bin/mqshutdown命令。

```shell
[root@mqOS rocketmq]# sh bin/mqshutdown broker
The mqbroker(1740) is running...
Send shutdown request to mqbroker(1740) OK

[root@mqOS rocketmq]# sh bin/mqshutdown namesrv
The mqnamesrv(1692) is running...
Send shutdown request to mqnamesrv(1692) OK
[2]+ 退出 143 nohup sh bin/mqbroker -n localhost:9876
```

## 7 环境变量配置
```shell
vim /etc/profile
```

在profile文件的末尾加入如下命令
```shell
export ROCKETMQ_HOME=/usr/local/mng/rocketmq-4.9.2
export PATH=$PATH:$ROCKETMQ_HOME/bin
```

输入:wq! 保存并退出， 并使得配置立刻生效：
```shell
source /etc/profile
```

---
title: Linux下zookeeper环境部署
tags: 
   - 环境部署
categories:
   - 大数据
   - 环境部署
date: 2021/10/8
cover: /mngImg/zookeeper.jfif
top_img: /mngImg/yeWan.jpg
---

# 一、环境

Linux系统版本：CentOS Linux release 7.6.1810

jdk版本：[jdk-8u181-linux-x64](https://repo.huaweicloud.com/java/jdk/8u181-b13/)

zookeeper: [zookeeper-3.4.9](http://archive.apache.org/dist/zookeeper/zookeeper-3.4.9/)

# 二、安装步骤

## 1. 安装包上传并解压

使用xftp 将zookeeper-3.4.9.tar.gz传入服务器的/usr/local/mng目录下，并解压

```
//解压
tar -zxvf /usr/local/mng/zookeeper-3.4.9.tar.gz
//改名
mv /usr/local/mng/zookeeper-3.4.9  /usr/local/mng/zookeeper
```

## 2.相关参数配置

进入解压好的 zookeeper 目录中，将 conf/zoo_sample.cfg 改名为conf/zoo.cfg

```
cd zookeeper
cd conf
mv zoo_sample.cfg zoo.cfg
```

![image-20211009225058505](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211009225058505.png)

dataDir、dataLogDir参数修改

![image-20211010000856253](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010000856253.png)

文件末添加（zookeeper集群，server.2 、server.3.......）

![image-20211009225754585](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211009225754585.png)

在dataDir目录下新建myid文件作为主机标识。

```
cd /usr/local/mng/zookeeper/
mkdir data
cd data
echo 1 >> myid
cd ..
mkdir logs
cd logs
echo 1 >> myid
//注意：这个 id 是 zookeeper 的主机标识，每个主机 id 不同，第一台是1，第二台是 2，以此类推。
```

## 3.启动zookeeper

```
cd /usr/local/mng/zookeeper/bin
./zkServer.sh start


//查看zookeeper状态
./zkServer.sh status
```

![image-20211010000750938](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010000750938.png)

![image-20211010000735372](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010000735372.png)

zookeeper以单机模式运行，2354进程就是zookeeper进程

# 三. 配置环境变量

```
vi ~/.bashrc
```

将下面几行加入最下面

```
export CCP_HOME=/usr/local/mng
export ZOOKEEPER_HOME=$CCP_HOME/zookeeper
export PATH=$PATH:$ZOOKEEPER_HOME/bin
```

让配置文件生效

```
source ~/.bashrc
```

查看安装情况

```
./zkServer.sh status
```

![image-20211010003053877](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211010003053877.png)

不需要进入或者写 zookeeper/bin 了
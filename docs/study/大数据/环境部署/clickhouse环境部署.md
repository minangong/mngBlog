# 一、环境

Linux系统版本：CentOS Linux release 7.6.1810

clickhouse: [Release v21.8.8.29](https://github.com/ClickHouse/ClickHouse/releases/tag/v21.8.8.29-lts)

下载下面四个文件，尚硅谷的资源应该也可以用（具体看三、真正的问题与解决）

![image-20211013001349877](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013001349877.png)



# 二.安装步骤

## 2.1 开放防火墙端口8123、9001

```
# 防火墙开启端口
firewall-cmd --zone=public --add-port=8123/tcp --permanent
firewall-cmd --zone=public --add-port=9001/tcp --permanent
#防火墙重新加载配置
firewall-cmd --reload
#查看开启的端口
firewall-cmd --list-ports
```

## 2.2 centos取消打开文件数限制

![image-20211011214714451](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211011214714451.png)

* /etc/security/limits.conf 文件的末尾加入以下内容

```
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

* /etc/security/limits.d/20-nproc.conf 文件的末尾加入以下内容

```
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```



## 2.3 安装依赖

```
sudo yum install -y libtool
sudo yum install -y *unixODBC*
sudo yum install libicu.x86_64
```



## 2.4 Centos取消Selinux

修改/etc/selinux/config中的SELINUX=disabled

## 2.5导入包并安装

```
tar -xzvf clickhouse-common-static-21.8.8.29.tgz
sudo clickhouse-common-static-21.8.8.29/install/doinst.sh

tar -xzvf clickhouse-common-static-dbg-21.8.8.29.tgz
sudo clickhouse-common-static-dbg-$LATEST_VERSION/install/doinst.sh

tar -xzvf clickhouse-server-21.8.8.29.tgz
sudo clickhouse-server-21.8.8.29/install/doinst.sh

tar -xzvf clickhouse-client-21.8.8.29.tgz
sudo clickhouse-client-21.8.8.29/install/doinst.sh

```

## 2.6 修改配置文件

​       步骤2.5时，会问是否允许外部访问，但是由于我密码弄错了，ctrl+shift+c了一下，然后再doinst.sh就不问了，还得自己配config.xml。

```
 sudo vim /etc/clickhouse-server/config.xml
```

网上教程：把\<listen_host\>::\</listen_host\>的注释打开，这样的话才能让 ClickHouse 被除本 机以外的服务器访问

**如果是阿里云,不要打开这个注释，开放\<listen_host\>0.0.0.0\</listen_host\>, 即只开放ipv4**

![image-20211011234949278](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211011234949278.png)



如果担心hadoop的9000端口和clickhouse冲突，可以**将\<tcp_port\>改为9001**

![image-20211013002031091](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013002031091.png)

## 2.7 启动Server

```
sudo /etc/init.d/clickhouse-server start
```

![image-20211013001549744](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013001549744.png)

## 2.8 使用 client 连接 server

```
clickhouse-client -m  --port 9001
```

![image-20211013001608847](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013001608847.png)

## 2.9 主机访问

云服务器IP:8123/play

![image-20211013002517328](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013002517328.png)

# 三、问题与解决

## 搞笑的问题解决

```
3.1 连接server报错：Code: 210. DB::NetException: Connection refused (localhost:9000)
    原因：listen_host 注释取消错了  或者 删了 下面的listen_host
    解决方法：停止server,改config

3.2 别改/etc/clickhouse-server/users.d/default-password.xml下的密码，我设为空或者空的sha56值都     失败了，systemctl start clickhouse-server失败

3.3 如果不记得密码，或者不记得listen_host原本什么样子了
    解决方法：删了重来吧（心疼作者一秒）
    //查看安装了那些
    yum list installed | grep clickhouse
    //开始删除
    yum remove -y clickhouse-common-static.x86_64
    yum remove -y clickhouse-common-static-dbg.x86_64
    //删这个是因为我设了密码用client不太方便，所以删了，设密码时直接回车。不然留着就不能重设密码了
    rm -rf  /etc/clickhouse-server	        clickhouse 服务端配置文件目录
    //其余
    rm -rf  /etc/clickhouse-client	        clickhouse 客户端配置文件目录
    rm -rf  /var/lib/clickhouse	            clickhouse 默认数据目录
    rm -rf  /var/log/clickhouse-server	    clickhouse 默认日志目录
    rm -rf  /etc/init.d/clickhouse-server	clickhouse 服务端启动脚本
```

## 真正的问题与解决

​      开始是跟着尚硅谷的视频、笔记在部署clickhouse，最后**启动server、使用 client 连接 server** 都成功了，但是**windows主机连不到8123、或者9000端口**。开始以为hadoop的9000端口与clickhouse冲突了，于是把<tcp_port>改成了9001。

​      我在这里重启clickhouse-server,但是每次重启都失败了，于是按照上面搞笑的3.3: 删了，改config, 然后启动服务，但是发现：

![image-20211012235900777](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211012235900777.png)

可以看到：第一还是**监听9000端口**. 第二8123、9000监听的都是**127.0.0.1，是云服务器自己的请求**。

**所以，查看日志！！！**

![image-20211013000422158](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013000422158.png)

日志中有这么一段：

```
2021.10.12 23:17:25.332186 [ 3132 ] {} <Warning> Application: Listen [::1]:8123 failed: 
Poco::Exception. Code: 1000, e.code() = 99, e.displayText() = Net Exception: Cannot 
assign requested address: [::1]:8123 (version 21.8.8.29 (official build)). If it is an 
IPv6 or IPv4 address and your host has disabled IPv6 or IPv4, then consider to specify 
not disabled IPv4 or IPv6 address to listen in <listen_host> element of configuration 
file. Example for disabled IPv6: <listen_host>0.0.0.0</listen_host> . Example for 
disabled IPv4: <listen_host>::</listen_host>
```

Cannot assign requested address: [::1]:8123，需要是否区分支持IPV4或者IPV6，

**不支持IPV6：设为<listen_host>0.0.0.0</listen_host>，不支持IPV4： <listen_host>::</listen_host>**

**修改为<listen_host>0.0.0.0</listen_host>后，重启成功！！！！！！**

![image-20211013000957295](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013000957295.png)

**所以，看日志！！！！**之前只看了systemctl status clickhouse-server。



# 四、java+clickhouse

参考来源：[(1条消息) Clickhouse 入门教程（二）—— Java 连接示例_magicpenta的博客-CSDN博客](https://blog.csdn.net/magicpenta/article/details/89515550)

clickhouse的jdbc驱动：

```
官方驱动：
<dependency>
    <groupId>ru.yandex.clickhouse</groupId>
    <artifactId>clickhouse-jdbc</artifactId>
    <version>0.1.52</version>
</dependency>

三方提供的驱动：
<dependency>
    <groupId>com.github.housepower</groupId>
    <artifactId>clickhouse-native-jdbc</artifactId>
    <version>1.6-stable</version>
</dependency>
```

采用了三方提供的驱动，pom.xml中添加相应依赖

```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.Statement;

public class Test {
    public static void main(String[] args) {
        try{
            Class.forName("com.github.housepower.jdbc.ClickHouseDriver");
            //ip自行填写
            Connection connection = DriverManager.getConnection("jdbc:clickhouse://:9001?connectTimeout=60000&socketTimeout=300000");
            Statement statement = connection.createStatement();
            statement.executeQuery("create database test");
            statement.executeQuery("create table test.jdbc_example(day Date, name String, age UInt8) Engine=Log");
        }catch (Exception e){
            e.printStackTrace();
        }
    }
}

```

执行后，

![image-20211013003227104](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211013003227104.png)

**成功！！**


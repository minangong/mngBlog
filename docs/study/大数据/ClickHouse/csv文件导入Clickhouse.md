## 1.新建表
```
CREATE TABLE zhanguo(
name String COMMENT '姓名',
age UInt8 comment '年龄',
Country String comment '国家',
Job  String comment '职业'
)ENGINE = MergeTree() 
ORDER BY (name,age,Country)
SETTINGS index_granularity = 8192 ;
 
```

## 2.准备csv
```
嬴政,23,秦,王
赵括,24,赵,将军
王翦,53,秦,上将军
屈原,25,楚,士大夫
李冰,30,秦,郡守
韩非,32,韩,士大夫
赵高,31,秦,太监
李斯,33,秦,丞相
```

## 3.
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220502011315.png)

```
cat /dev/shm/data | clickhouse-client -h 123.456.1.789 --query="insert into database.table FORMAT CSV"```

或

```powershell
clickhouse-client -h localhost --query="insert into dataflow.zhanguo FORMAT CSV" < /dataflow/data/zhanguo.csv

```




## 4.
```
netstat  -nlp|grep 8082

kill

netstat -tlun
```


## 5.
非交互模式下，调用 bash 解释器，通过 bash -c 后接命令的形式来解释执行命令


我们使用 sudo 只是让 echo 命令具有了 root 权限，但是没有让 “>” 和 “>>” 命令也具有 root 权限，所以 bash 会认为这两个命令都没有像 test.csv文件写入信息的权限。
解决这一问题的途径有两种。

第一种是利用 “sh -c” 命令，它可以让 bash 将一个字串作为完整的命令来执行，这样就可以将 sudo 的影响范围扩展到整条命令。具体用法如下：
$ sudo /bin/sh -c ‘echo “hahah” >> test.asc’
————————————————
版权声明：本文为CSDN博主「平平无奇子」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/qq_44792624/article/details/107189110
```java
String cmd = "clickhouse-client -h 47.94.161.48 --query=\"insert into " +  
        "dataflow."+"zhanguo" +  
        " FORMAT CSV\" < " +  
        "/dataflow/data/"+fileName;  
  
System.out.println(cmd);  
//Process proc = Runtime.getRuntime().exec(cmd);  
  
Runtime run = Runtime.getRuntime();  
try {  
    String[] cmds = { "/bin/sh", "-c", cmd };  
    Process process = run.exec(cmds);  
    process.waitFor();  
    process.destroy();  
} catch (IOException e) {  
    e.printStackTrace();  
}
```

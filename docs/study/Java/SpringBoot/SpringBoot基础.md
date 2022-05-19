# SpringBoot基础

# 一、springBoot简介

  Springboot是由Pivotal团队提供的全新框架，其设计目的是用来简化Spring应用的初始
搭建以及开发过程

* Spring程序缺点：
   * 依赖设置繁琐
   * 配置繁琐
* SpringBoot程序优点
   * 起步依赖（简化依赖配置）
   * 自动配置（简化常用工程相关配置）
   * 辅助功能（内置服务器，.....）

# 二、快速上手Springboot
## 2.1 创建Springboot工程的四种方式
* 基于Idea创建SpringBoot工程
* 基于官网创建SpringBoot工程
* 基于阿里云创建SpringBoot工程
* 手工创建Maven工程修改为SpringBoot工程

## 2.2 隐藏指定文件/文件夹
Setting → File Types → Ignored Files and Folders
输入要隐藏的文件名，支持\*号通配符


## 2.3 parent starter

**parent:** 【版本控制，减少依赖冲突】
* 所有SpringBoot项目要继承的项目，定义了若干个坐标版本号（依赖管理，而非依赖），以达到减少依赖冲突的目的。帮助开发者统一的进行各种技术的版本管理。【不负责导入坐标，只进行版本管理】
* spring-boot-starter-parent各版本间存在着诸多坐标版本不同
**starter:**【减少依赖配置的书写量】
* 实际开发时，对于依赖坐标的使用往往都有一些固定的组合方式，比如使用spring-webmvc就一定要使用spring-web。SpringBoot将其统一集成进starter，应用者只需要在maven中引入starter依赖，SpringBoot就能自动扫描到要加载的信息并启动相应的默认配置。【快速配置、简化配置】

[SpringBoot应用篇（一）：自定义starter - 超级小小黑 - 博客园 (cnblogs.com)](https://www.cnblogs.com/hello-shf/p/10864977.html)


**实际开发：**
* 实际开发中如果需要用什么技术，先去找有没有这个技术对应的starter
    * 如果有对应的starter，直接写starter，而且无需指定版本，版本由parent提供
    * 如果没有对应的starter，手写坐标即可
* 实际开发中如果发现坐标出现了冲突现象，确认你要使用的可行的版本号，使用手工书写的方式添加对应依赖，覆盖SpringBoot提供给我们的配置管理
    * 方式一：直接写坐标
    * 方式二：覆盖\<properties\>中定义的版本号，就是下面这堆东西了，哪个冲突了覆盖哪个就OK了
* 使用任意坐标时，仅书写GAV中的G和A，V由SpringBoot提供，除非SpringBoot未提供对应版本V

## 2.4 引导类
Spring程序运行的基础是需要创建自己的Spring容器对象（IoC容器）并将所有的对象交给Spring的容器管理，也就是一个一个的Bean。
当前这个类运行后就会产生一个Spring容器对象，并且可以将这个对象保存起来，通过容器对象直接操作Bean。
```java
@SpringBootApplication
public class Springboot0101QuickstartApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext ctx = SpringApplication.run(Springboot0101QuickstartApplication.class, args);
        BookController bean = ctx.getBean(BookController.class);
        System.out.println("bean======>" + bean);
    }
}
```


## 2.5 内置服务器
### 2.5.1 内嵌Tomcat定义位置
Web的starter：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```
其中导入了：
```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <version>2.5.4</version>
        <scope>compile</scope>
    </dependency>
```
tomcat的starter导入了：
```xml
    <dependency>
        <groupId>org.apache.tomcat.embed</groupId>
        <artifactId>tomcat-embed-core</artifactId>
        <version>9.0.52</version>
        <scope>compile</scope>
        <exclusions>
            <exclusion>
                <artifactId>tomcat-annotations-api</artifactId>
                <groupId>org.apache.tomcat</groupId>
            </exclusion>
        </exclusions>
    </dependency>
```
这里面有一个核心的坐标，tomcat-embed-core，叫做tomcat内嵌核心。就是这个东西把tomcat功能引入到了我们的程序中。
### 2.5.2 内置服务器运行原理
Tomcat服务器是由Java语言开发的软件，运行靠的是对象。Tomcat服务器运行其实是以对象的形式在Spring容器中运行的。具体运行的就是上面的tomcat内嵌核心。


### 2.5.3 更换内嵌的服务器
* tomcat(默认)  apache出品，粉丝多，应用面广，负载了若干较重的组件
* jetty 更轻量级，负载性能远不及tomcat
* undertow 负载性能勉强跑赢tomcat

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>
</dependencies>
```

首先排除掉默认加载的tomcat,然后加入服务器的坐标（优先starter）。


# 三、springboot基础配置
## 3.1 属性配置
SpringBoot通过配置文件application.properties就可以修改默认的配置，简化开发者配置的书写位置，集中管理。

打开SpringBoot的官网，找到SpringBoot官方文档，打开查看附录中的Application Properties就可以获取到对应的配置项了，网址：[https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#application-properties)

另外，SpringBoot中导入对应starter后，才提供对应配置属性

## 3.2 配置文件分类
SpringBoot提供了3种配置文件的格式
-   properties（传统格式/默认格式）
```properties
server.port=80
```
-   **yml**（主流格式）
```yml
server:
  port: 81
```
-   yaml : yml和yaml文件格式就是一模一样的，只是文件后缀不同，
```yaml
  port: 82
```


## 3.3 配置文件优先级
三个文件如果共存的话，优先级为：
```
application.properties  >  application.yml  >  application.yaml
```
每个配置文件中的项都会生效，只不过如果多个配置文件中有相同类型的配置会优先级高的文件覆盖优先级低的文件中的配置。如果配置项不同的话，那所有的配置项都会生效。


## 3.4 自动提示功能消失解决方案
可能的原因是：Idea认为你现在写配置的文件不是个配置文件，所以拒绝给你提供提示功能

1. 【Files】→【Project Structure...】-->【Facets】,选择提示功能消失的模块
2. 点击Customize Spring Boot按钮，此时可以看到当前模块对应的配置文件是哪些了。如果没有你想要称为配置文件的文件格式，就有可能无法弹出提示
![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220303234727.png)
3. 选择添加配置文件，然后选中要作为配置文件的具体文件就OK了


## 3.5 yaml文件
1.  大小写敏感
2.  属性层级关系使用多行描述，**每行结尾使用冒号结束**
3.  使用缩进表示层级关系，同层级左侧对齐，只允许使用空格（不允许使用Tab键）
4.  属性值前面添加空格（属性名与属性值之间使用冒号+空格作为分隔）
5.  \#号 表示注释
6. 数组表示：
```yml
likes: [王者荣耀,刺激战场]			#数组书写缩略格式
users:							 #对象数组格式一
  - name: Tom
   	age: 4
  - name: Jerry
    age: 5
```


## 3.6 yaml数据提取

### 3.6.1 读取单一数据
yaml中保存的单个数据，可以使用Spring中的注解直接读取，使用@Value可以读取单个数据，属性名引用方式：**${一级属性名.二级属性名……}**
记得使用@Value注解时，要将该注入写在某一个指定的Spring管控的**bean的属性名**上方。现在就可以读取到对应的单一数据行了。

![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220304110331.png)

### 3.6.2 读取全部数据
1.  使用Environment对象封装全部配置信息
2.  使用@Autowired自动装配数据到Environment对象中
3.  调用时使用getProperties(String),参数填写属性名即可
![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220304111553.png)


### 3.6.3 读取对象数据
首先定义一个对象，并将该对象纳入Spring管控的范围，也就是定义成一个bean，然后使用注解@ConfigurationProperties指定该对象加载哪一组yaml中配置的信息。
调用时@Autowired注入
![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220304221940.png)



### 3.6.4 数据引用
写yaml文件时，会出现很多个文件都具有相同的目录前缀。可以使用引用格式来定义数据，其实就是搞了个变量名，然后引用变量。
```yaml
baseDir: /usr/local/fire
	center:
    dataDir: ${baseDir}/data
    tmpDir: ${baseDir}/tmp
    logDir: ${baseDir}/log
    msgDir: ${baseDir}/msgDir
```

还有一个注意事项，在书写字符串时，如果需要使用转义字符，需要将数据字符串使用双引号包裹起来
```yaml
lesson: "Spring\tboot\nlesson"
```




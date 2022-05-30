# JavaWeb

开始学习SpringMVC前，对JavaWeb做一个大致的了解

# 一、Tomcat

[servlet的本质是什么，它是如何工作的？](https://www.zhihu.com/question/21416727)

Tomcat其实是Web服务器和Servlet容器的结合体。

应用程序三个过程：接收请求，处理请求，响应请求。接受、响应请求是共性的、且没有差异性，抽取成Web服务器。请求处理的逻辑是不同的。抽取成Servlet，由程序员编写。之后出现三层架构，一些逻辑从Servlet中抽取出来，分担到Service和Dao。

Web服务器：将主机上的资源映射为一个URL供外界访问

servlet容器：要通过Web服务器映射的URL访问资源，肯定需要写程序处理请求。

![image-20211120234401544](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120234401544.png)

## 1. Tomcat结构图

原文链接：https://blog.csdn.net/u014231646/article/details/79482195

[Tomcat外传](https://zhuanlan.zhihu.com/p/54121733)

![image-20211119003140203](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211119003140203.png)

![image-20211120234545157](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120234545157.png)

​	 Tomcat主要组件：服务器Server，服务Service，连接器Connector、容器Container。其中，Service和Connector是核心组件

​      一个**Container容器**和一个或多个**Connector**组合在一起，加上其他组件共同**组成一个Service服务**，对外提供能力了。Server组件可以同时**管理**一个或多个Service服务



​     结合Tomcat的配置文件Server.xml来看

![preview](https://raw.githubusercontent.com/minangong/mng_images/main/images/v2-fb9aa5d024fa131bbec7fb9fe9d99739_r.jpg)

* 根目录是<Server\>，代表服务器，\<Server\>下的\<Service\>代表服务。

* \<Service\>下有多个\<Connector\>代表连接，可以配置监视的端口和协议。

  \<Service\>下有一个\<Engine\> Tomcat引擎处理请求

## 2. Tomcat组件--Connector

一个Connector在某个指定的端口上侦听客户请求。**接收**浏览器的tcp**连接请求**，创建Request和Response对象用于和请求端交换数据。

然后**产生一个线程**来处理这个请求并把产生的Request、Response对象传给处理Engine(Contain的一部分)，从Engine获得响应并返回给客户。

Connector 最重要的功能就是**接收连接请求然后分配线程让 Container 来处理这个请求**，所以这必然是**多线程**的，多线程的处理是Connect设计的核心



## 3.Tomcat组件--Container

![image-20211119220950456](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211119220950456.png)

​       Container是容器的父接口，该容器的设计用的是典型的责任链的设计模式，它由四个自容器组件构成，分别是Engine、Host、Context、Wrapper。这四个组件是负责关系，存在包含关系。通常**一个Servlet class对应一个Wrapper**，如果有多个Servlet定义多个Wrapper，如果有**多个Wrapper就要定义一个更高的Container**，如Context。 
​       Context 还可以定义在父容器 Host 中，Host 不是必须的，但是**要运行 war 程序，就必须要 Host**，因为 war 中必有 **web.xml 文件，这个文件的解析就需要 Host 了**，如果要有**多个 Host 就要定义一个 top 容器 Engine** 了。而 Engine 没有父容器了，**一个 Engine 代表一个完整的 Servlet 引擎**。

### Engine 容器 
Engine 容器比较简单，它只定义了一些基本的关联关系
### Host 容器 
Host 是 Engine 的子容器，**一个 Host 在 Engine 中代表一个虚拟主机**，这个虚拟主机的作用就是**运行多个应用**，它负责安装和展开这些应用，并且标识这个应用以便能够区分它们。它的子容器通常是 Context，它除了**关联子容器**外，还有就是**保存一个主机应该有的信息**。

### Context 容器

Context 代表 Servlet 的 Context，它具备了 **Servlet 运行的基本环境**，理论上只要有 Context 就能运行 Servlet 了。简单的 Tomcat 可以没有 Engine 和 Host。Context 最重要的功能就是**管理它里面的 Servlet 实例**，Servlet 实例在 Context 中是以 Wrapper 出现的，还有一点就是 Context 如何才能**找到正确的 Servlet** 来执行它呢？ Tomcat5 以前是通过一个 Mapper 类来管理的，Tomcat5 以后这个功能被移到了 **request** 中，在前面的时序图中就可以发现**获取子容器都是通过 request 来分配**的。

### Wrapper 容器

Wrapper 代表一个 Servlet，它负责**管理一个 Servlet，包括的 Servlet 的装载、初始化、执行以及资源回收**。Wrapper 是最底层的容器，它没有子容器了，所以调用它的 addChild 将会报错。 
Wrapper 的实现类是 StandardWrapper，StandardWrapper 还实现了拥有一个 Servlet 初始化信息的 ServletConfig，由此看出 StandardWrapper 将直接和 Servlet 的各种信息打交道

## 4.Tomcat运行机制

1、用户点击网页内容，请求被发送到本机端口8080，被在那里监听的Coyote HTTP/1.1 **Connector**获得。 
2、Connector把该请求交给它所在的Service的**Engine来处理**，并**等待Engine的回应**。 
3、Engine获得请求localhost/test/index.jsp，**匹配所有的虚拟主机Host**。 
4、Engine匹配到名为localhost的Host（即使匹配不到也把请求交给该Host处理，因为该Host被定义为该Engine的默认主机），名为localhost的**Host获得请求/test/index.jsp**，**匹配**它所拥有的所有的**Context**。Host匹配到路径为/test的Context（如果匹配不到就把该请求交给路径名为“ ”的Context去处理）。 
5、path=“/test”的Context获得请求/index.jsp，在它的**mapping table**中寻找出**对应的Servlet**。Context匹配到URL PATTERN为*.jsp的Servlet,对应于JspServlet类。 
6、**构造HttpServletRequest对象和HttpServletResponse对象**，作为参数**调用JspServlet的doGet（）或doPost（）**.执行业务逻辑、数据存储等程序。 
7、Context把执行完之后的**HttpServletResponse对象返回**给Host。 
8、Host把HttpServletResponse对象返回给Engine。 
9、Engine把HttpServletResponse对象返回Connector。 
10、Connector把HttpServletResponse对象返回给客户Browser。



**简化：**

用户发送Http请求到Tomcat 

Tomcat从磁盘中加载Servlet(第一次请求)

Tomcat解析Http请求为request对象               

转发request至相应的Servlet进行处理             Servlet.service(   )

Servlet处理后返回response

Tomcat将response转换成Http响应

Tomcat将Http响应返回给客户端





#  二、Servlet

​     是用Java编写的服务器端程序。主要功能在于交互式地浏览和修改数据，生成动态``内容。多数情况下用来扩展基于Http协议的Web服务器

​     狭义的Servlet是指Java语言实现的一个接口，广义的Servlet是指任何实现了这个Servlet接口的类。

​     等Spring家族出现后，Servlet开始退居幕后，取而代之的是方便的SpringMVC。SpringMVC的核心组件DispatcherServlet其实本质就是一个Servlet。但它已经自立门户，在原来HttpServlet的基础上，又封装了一条逻辑。



## Servlet处理请求

![image-20211121014604302](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211121014604302.png)

## 1.Servlet接口

Tomcat已经把解析HTTP请求、把结果转成HTTP响应做好了，封装成了request对象和response对象。还会传入servletConfig对象(web.xml)

```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by FernFlower decompiler)
//

package javax.servlet;

import java.io.IOException;

public interface Servlet {
  void init(ServletConfig var1) throws ServletException;

  ServletConfig getServletConfig();

  void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

  String getServletInfo();

  void destroy();
}
```

**init(ServletConfig var1)** 在Servlet第一次被请求时，Servlet会调用这个方法来初始化一个Servlet对象。只执行一次

**service(ServletRequest var1, ServletResponse var2)**   每当请求Servlet时，Servlet容器（例如Tomcat）就会调用这个方法。

**destory(  )** 销毁Servlet前，会调用这个方法。在卸载应用程序或者关闭Servlet容器时，会发生这种情况，会在这个方法里写一些清除代码

  getServletInfo（ ），这个方法会返回Servlet的一段描述，可以返回一段字符串。

  getServletConfig（ ），这个方法会返回由Servlet容器传给init（ ）方法的ServletConfig对象。





## 2.servletConfig

Servlet配置，web.xml中配置了Servlet.Tomcat解析web.xml，创建servletConfig实例并传入。

![preview](https://raw.githubusercontent.com/minangong/mng_images/main/images/v2-3dd656100783b3e9e62621ad8e2e9b04_r.jpg)

## 3.Request和Response

HTTP请求到了Tomcat后，Tomcat通过字符串解析，把请求头（Header）,请求地址（URL）,请求参数（QueryString）等都封装进了Request对象中。

通过以下方法可以得到发送的请求信息：(HttpServletResquest中的方法)

```
request.getHeader();
request.getUrl()；
request.getQueryString();
...
```



至于Response，Tomcat传给Servlet时，它还是**空的对象**。Servlet逻辑处理后得到结果，最终通过response.write()方法，将结果写入response内部的缓冲区。Tomcat会在**servlet处理结束**后，拿到response，遍历里面的信息，**组装成HTTP响应**发给客户端。



## 4.GeneicServlet

GenericServlet,定义了成员变量servletConfig，在init（）时，保存传入的servletConfig对象。

## 5.HttpServlet

在写service()处理逻辑时，需要判断是Get/Post请求。为了简化这个操作，不直接实现servlet接口。HttpServlet实现了service，根据request.getMethod()判断，调用相应的doGet()、doPost().



所以子类只需要重写doGet、doPost



![preview](https://raw.githubusercontent.com/minangong/mng_images/main/images/v2-73b703e690ce018ffe88280376a67dc0_r.jpg)

## 6.基于注解开发

Servlet3.0支持基于注解开发

在类上使用@WebServlet注解，进行配置
@WebServlet("/demo2") 注解接收参数是url匹配地址
注解配置详解
一个Servlet对应多个url @WebServlet({"/url1", “/url2”, “url3”})

# 三、Filter

原文链接： [JavaWeb三大组件_miracle-CSDN博客_](https://blog.csdn.net/qq_39013701/article/details/86691439)

## 1.概念：

一般用于完成通用的操作。如 登录验证，编码统一处理，敏感字符过滤

## 2.快速入门：

```
	1.步骤：
		1.定义一个类，实现接口Filter
		2.复写
		3.配置拦截路径
			两种配置方法
			1.web.xml 配置
				<filter>
			        <filter-name>demo2</filter-name>
			        <filter-class>cn.itcast.web.filter.FilterDemo2</filter-class>
			    </filter>
			    <filter-mapping>
			        <filter-name>demo2</filter-name>
			        <url-pattern>/*</url-pattern>
			    </filter-mapping>
			2.注解
				@WebFilter("/*") 里面是拦截规则


```

```
package cn.itcast.web.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;

@WebFilter("/*")
public class FilterDemo1 implements Filter {

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("filter");
        // 放行，如果不放行，那么被访问的资源将无法看见
        filterChain.doFilter(servletRequest, servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```



## 3.执行流程
请求首先到达过滤器的doFilter方法，然后对request增强，然后调用filterChain放行，等Servlet执行完毕后，对response增强

## 4.过滤器生命周期

```
1.init：在服务器启动后，会创建Filter对象，然后调用init方法。只执行一次。用于加载资源
2.doFilter：每一次请求被拦截资源时，会执行。执行多次
3.destroy：在服务器关闭后，Filter对象会被销毁。如果服务器是正常关闭，则会执行destory方法。只执行一次。用于释放资源
123
```

## 5.拦截配置

```
<filter>
	<filter-name>MyFirstFilter</filter-name>
	<filter-class>com.atguigu.filter.MyFirstFilter</filter-class>
</filter>

<filter-mapping>
	<filter-name>MyFirstFilter</filter-name>
	<url-pattern>/*</url-pattern>
	/**
	*url-pattern的三种写法（不能两两组合使用）
	*1、精确匹配
	* /pics/index.jsp   /template/login ;直接拦截指定路径
	*2、路径匹配
	* /pics/*  ;拦截pics下的所有请求
	* 3、后缀匹配
	*  *.jsp  ;拦截所有以.jsp结尾的请求
	*/
</filter-mapping>

```

* 拦截路径配置

```
1.具体资源路径拦截：/index.jsp	只有访问index.jsp资源时，过滤器才会被执行
2.拦截目录：/user/* 访问/user下的所有资源时，过滤器都会被执行
3.后缀名拦截：*.jsp 访问所有后缀名为jsp资源时 过滤器都会被执行
4.拦截所有资源：/* 访问所有资源时过滤器会被执行

```

* 拦截方式配置

```
* 注解配置
	1.REQUEST：默认值。浏览器直接请求资源
	2.FORWARD：转发访问资源
	3.INCLUDE：包含访问资源
	4.ERROR：错误跳转资源
	5.ASYNC：异步访问资源
* web.xml配置
	设置<dispatcher></dispatcher>标签即可
```

## 6.过滤器链(配置多个过滤器)
执行顺序：如果有两个过滤器：过滤器1和过滤器2

```
1.过滤器1
2.过滤器2
3.资源执行
4.过滤器2 
5.过滤器1
```

过滤器先后顺序问题：

```
1.注解配置：按照类名的字符串比较规则比较，值小的先执行
2.web.xml配置：谁定义在上边谁先执行
```

# 四、监听器

## 1.概念

```
* 事件监听机制
	* 事件	：一件事情
	* 事件源	：事件发生的地方
	* 监听器	：一个对象
	* 注册监听	：将事件，事件源，监听器绑定在一起。当事件源上发生某个事件后，执行监听器代码
```

## 2.实现

监听ServletContext对象的创建和销毁
包含两个方法
contextDestroyed(ServletContextEvent sce) : ServletContext对象被销毁之前会调用该方法
contextInitialized(ServletContextEvent sce) : ServletContext对象创建后会调用该方法
使用步骤

```
1.定义一个类，实现ServletContextListener接口
2.复写方法
3.配置
	1.web.xml
	2.注解
```

# 举例（结合网络）

![image-20211120234434055](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120234434055.png)

## 网络部分

通过域名访问时，首先本机hosts文件查询域名对应IP，查不到访问DNS，返回IP。根据IP和端口访问服务器。

![preview](https://raw.githubusercontent.com/minangong/mng_images/main/images/v2-1ebb772d4f718d6a2fefac68bb1a15d4_r.jpg)

访问的Request请求中还是会带上host。因为 **域名！= IP**。即**一个IP可以对应多个虚拟主机**。IP对应实体服务器，域名对应具体的网站。

![image-20211120233524665](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120233524665.png)



## Tomcat



![image-20211120233942116](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120233942116.png)







## servlet

![image-20211120234504093](https://raw.githubusercontent.com/minangong/mng_images/main/images/image-20211120234504093.png)










































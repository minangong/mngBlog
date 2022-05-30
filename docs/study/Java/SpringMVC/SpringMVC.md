# SpringMVC

# 一、SpringMVC简介

## 1.什么是MVC

MVC是一种软件架构的思想，将软件按照模型、视图、控制器来划分

**M：Model，模型层，指工程中的JavaBean，作用是处理数据**

JavaBean分为两类：

- 实体类Bean：专门存储业务数据的，如 Student、User 等
- 业务处理 Bean：指 Service 或 Dao 对象，专门用于处理业务逻辑和数据访问。

**V：View，视图层，指工程中的html或jsp等页面，作用是与用户进行交互，展示数据**

**C：Controller，控制层，指工程中的servlet，作用是接收请求和响应浏览器**

MVC的工作流程：
用户通过视图层发送请求到服务器，在服务器中请求被Controller接收，Controller调用相应的Model层处理请求，处理完毕将结果返回到Controller，Controller再根据请求处理的结果找到相应的View视图，渲染数据后最终响应给浏览器



## 2.什么是SpringMVC

SpringMVC 是 Spring 为表述层开发提供的一整套完备的解决方案。

> 注：三层架构分为表述层（或表示层）、业务逻辑层、数据访问层，表述层表示前台页面和后台servlet

## 3.SpringMVC特点

* Spring 家族原生产品，与 **IOC 容器**等基础设施**无缝对接** 
* 基于原生的Servlet，通过了功能强大的**前端控制器DispatcherServlet**，对请求和响应进行**统一处理**

- 表述层各细分领域需要解决的问题**全方位覆盖**，提供**全面解决方案**
- **代码清新简洁**，大幅度提升开发效率
- 内部组件化程度高，可插拔式组件**即插即用**，想要什么功能配置相应组件即可
- **性能卓著**，尤其适合现代大型、超大型互联网项目要求



# 二、Hello Word
[web.xml 中 Servlet 配置报错：cannot resolve servlet 'springmvc'](https://blog.csdn.net/qq_40147863/article/details/87717566)
## 2.1 文件格式
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301093900.png)

![image-20220301093637348](https://raw.githubusercontent.com/minangong/mng_images/main/images2/image-20220301093637348.png)
## 2.2 添加依赖
```xml
<dependencies>
    <!-- SpringMVC -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>5.3.1</version>
    </dependency>

    <!-- 日志 -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.3</version>
    </dependency>

    <!-- ServletAPI -->
    <dependency>
        <groupId>javax.servlet</groupId>
        <artifactId>javax.servlet-api</artifactId>
        <version>3.1.0</version>
        <scope>provided</scope>
    </dependency>

    <!-- Spring5和Thymeleaf整合包 -->
    <dependency>
        <groupId>org.thymeleaf</groupId>
        <artifactId>thymeleaf-spring5</artifactId>
        <version>3.0.12.RELEASE</version>
    </dependency>
</dependencies>

```

## 2.3 配置Web.xml
注册SpringMVC的前端控制器DispatcherServlet
```xml
<?xml version="1.0" encoding="UTF-8"?>  
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"  
 xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"  
 xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"  
 version="4.0">  
  
    <!--  
 注册springMVC的前端控制器，对浏览器所发送的请求统一进行处理  
 在此配置下，springMVC的配置文件具有默认的位置和名称  
 默认的位置：WEB-INF  
 默认的名称：<servlet-name>-servlet.xml  
 若要为springMVC的配置文件设置自定义的位置和名称  
 需要在servlet标签中添加init-param  
 <init-param> <param-name>contextConfigLocation</param-name> <param-value>classpath:springMVC.xml</param-value> </init-param> load-on-startup：将前端控制器DispatcherServlet的初始化时间提前到服务器启动时  
 -->  
 <servlet>  
        <servlet-name>springMVC</servlet-name>  
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
        <init-param>  
            <param-name>contextConfigLocation</param-name>  
            <param-value>classpath:springMVC.xml</param-value>  
        </init-param>  
        <load-on-startup>1</load-on-startup>  
    </servlet>  
  
    <!--  
 设置springMVC的核心控制器所能处理的请求的请求路径  
 
 /所匹配的请求可以是/login或.html或.js或.css方式的请求路径  
 但是/不能匹配.jsp请求路径的请求  
 
 /*则能够匹配所有请求
-->  
 <servlet-mapping>  
        <servlet-name>springMVC</servlet-name>  
        <url-pattern>/</url-pattern>  
    </servlet-mapping>  
  
</web-app>
```


## 2.4 创建请求控制器
由于前端控制器对浏览器发送的请求进行了统一的处理，但是具体的请求有不同的处理过程，因此需要创建处理具体请求的类，即请求控制器

请求控制器中每一个处理请求的方法成为控制器方法

因为SpringMVC的控制器由一个POJO（普通的Java类）担任，因此需要通过@Controller注解将其标识为一个控制层组件，交给Spring的IoC容器管理，此时SpringMVC才能够识别控制器的存在

```java
@Controller  
public class HelloController {  
    // 通过@RequestMapping注解，可以通过请求路径匹配要处理的具体的请求  
 //  /表示的当前工程的上下文路径  
 @RequestMapping("/")  
    public String index(){  
        return "index";  
    }  
  
    @RequestMapping("/target")  
    public String toTarget(){  
        return "target";  
    }  
}
```

## 2.5 MVC配置文件
```xml
<!-- 自动扫描包 -->
<context:component-scan base-package="com.atguigu.mvc.controller"/>

<!-- 配置Thymeleaf视图解析器 -->
<bean id="viewResolver" class="org.thymeleaf.spring5.view.ThymeleafViewResolver">
    <property name="order" value="1"/>
    <property name="characterEncoding" value="UTF-8"/>
    <property name="templateEngine">
        <bean class="org.thymeleaf.spring5.SpringTemplateEngine">
            <property name="templateResolver">
                <bean class="org.thymeleaf.spring5.templateresolver.SpringResourceTemplateResolver">
    
                    <!-- 视图前缀 -->
                    <property name="prefix" value="/WEB-INF/templates/"/>
    
                    <!-- 视图后缀 -->
                    <property name="suffix" value=".html"/>
                    <property name="templateMode" value="HTML5"/>
                    <property name="characterEncoding" value="UTF-8" />
                </bean>
            </property>
        </bean>
    </property>
</bean>

<!-- 
   处理静态资源，例如html、js、css、jpg
  若只设置该标签，则只能访问静态资源，其他请求则无法访问
  此时必须设置<mvc:annotation-driven/>解决问题
 -->
<mvc:default-servlet-handler/>

<!-- 开启mvc注解驱动 -->
<mvc:annotation-driven>
    <mvc:message-converters>
        <!-- 处理响应中文内容乱码 -->
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
            <property name="defaultCharset" value="UTF-8" />
            <property name="supportedMediaTypes">
                <list>
                    <value>text/html</value>
                    <value>application/json</value>
                </list>
            </property>
        </bean>
    </mvc:message-converters>
</mvc:annotation-driven>

```


## 2.6 tomcat 发布运行
[idea配置tomcat的url地址_长着个猫头的鹰的博客-CSDN博客_idea tomcat url配置](https://blog.csdn.net/qq_27523379/article/details/120432624?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_paycolumn_v3&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_paycolumn_v3&utm_relevant_index=2)


![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301213946.png)
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301213915.png)

## 2.7 总结
浏览器发送请求，若请求地址符合前端控制器的url-pattern，该请求就会被前端控制器DispatcherServlet处理。前端控制器会读取SpringMVC的核心配置文件，通过扫描组件找到控制器，将请求地址和控制器中@RequestMapping注解的value属性值进行匹配，若匹配成功，该注解所标识的控制器方法就是处理请求的方法。处理请求的方法需要返回一个字符串类型的视图名称，该视图名称会被视图解析器解析，加上前缀和后缀组成视图的路径，通过Thymeleaf对视图进行渲染，最终转发到视图所对应页面



# 三、@RequestMapping
## 3.1  功能
从注解名称上我们可以看到，@RequestMapping注解的作用就是将请求和处理请求的控制器方法关联起来，建立映射关系。

SpringMVC 接收到指定的请求，就会来找到在映射关系中对应的控制器方法来处理这个请求。


## 3.2 注解位置
@RequestMapping标识一个类：设置映射请求的请求路径的初始信息
@RequestMapping标识一个方法：设置映射请求请求路径的具体信息

## 3.3 value属性
@RequestMapping注解的value属性通过请求的请求地址匹配请求映射

@RequestMapping注解的value属性是一个字符串类型的数组，表示该请求映射能够匹配多个请求地址所对应的请求

@RequestMapping注解的value属性必须设置，至少通过请求地址匹配请求映射 

## 3.4 method属性

@RequestMapping注解的method属性通过请求的请求方式（get或post）匹配请求映射

@RequestMapping注解的method属性是一个RequestMethod类型的数组，表示该请求映射能够匹配多种请求方式的请求

若当前请求的请求地址满足请求映射的value属性，但是请求方式不满足method属性，则浏览器报错405：Request method 'POST' not supported

<a th:href="@{/test}">测试@RequestMapping的value属性-->/test</a><br>  

```java
@RequestMapping(  
 value = {"/testRequestMapping", "/test"},  
 method = {RequestMethod.GET, RequestMethod.POST}  
)  
public String testRequestMapping(){  
 return "success";  
}

```

> 注：
> 
> 1、对于处理指定请求方式的控制器方法，SpringMVC中提供了@RequestMapping的派生注解
> 
> 处理get请求的映射-->@GetMapping
> 
> 处理post请求的映射-->@PostMapping
> 
> 处理put请求的映射-->@PutMapping
> 
> 处理delete请求的映射-->@DeleteMapping
> 
> 2、常用的请求方式有get，post，put，delete
> 
> 但是目前浏览器只支持get和post，若在form表单提交时，为method设置了其他请求方式的字符串（put或delete），则按照默认的请求方式get处理
> 
> 若要发送put和delete请求，则需要通过spring提供的过滤器HiddenHttpMethodFilter，在RESTful部分会讲到


## 3.6 params属性（了解）
@RequestMapping注解的params属性通过请求的请求参数匹配请求映射

@RequestMapping注解的params属性是一个字符串类型的数组，可以通过四种表达式设置请求参数和请求映射的匹配关系

"param"：要求请求映射所匹配的请求必须携带param请求参数

"!param"：要求请求映射所匹配的请求必须不能携带param请求参数

"param=value"：要求请求映射所匹配的请求必须携带param请求参数且param=value

"param!=value"：要求请求映射所匹配的请求必须携带param请求参数但是param!=value

```java
@RequestMapping(
        value = {"/testRequestMapping", "/test"}
        ,method = {RequestMethod.GET, RequestMethod.POST}
        ,params = {"username","password!=123456"}
)
```

>注:
>若当前请求满足@RequestMapping注解的value和method属性，但是不满足params属性，此时页面回报错400：Parameter conditions "username, password!=123456" not met for actual request parameters: username={admin}, password={123456}


## 3.7 headers属性（了解）

@RequestMapping注解的headers属性通过请求的请求头信息匹配请求映射

@RequestMapping注解的headers属性是一个字符串类型的数组，可以通过四种表达式设置请求头信息和请求映射的匹配关系

"header"：要求请求映射所匹配的请求必须携带header请求头信息

"!header"：要求请求映射所匹配的请求必须不能携带header请求头信息

"header=value"：要求请求映射所匹配的请求必须携带header请求头信息且header=value

"header!=value"：要求请求映射所匹配的请求必须携带header请求头信息且header!=value

若当前请求满足@RequestMapping注解的value和method属性，但是不满足headers属性，此时页面显示404错误，即资源未找到

## 3.7 SpringMVC支持ant风格的路径
```txt
？  表示任意的单个字符

*   表示任意的0个或多个字符

**  表示任意的一层或多层目录

注意：在使用**时，只能使用/**/xxx的方式

```

## 3.8 SpringMVC支持路径中的占位符（重点）

原始方式：/deleteUser?id=1

rest方式：/deleteUser/1

SpringMVC路径中的占位符常用于RESTful风格中，当请求路径中将某些数据通过路径的方式传输到服务器中，就可以在相应的@RequestMapping注解的value属性中通过占位符{xxx}表示传输的数据，在通过@PathVariable注解，将占位符所表示的数据赋值给控制器方法的形参
```java
@RequestMapping("/testRest/{id}/{username}")  
public String testRest(@PathVariable("id") String id, @PathVariable("username") String username){  
 System.out.println("id:"+id+",username:"+username);  
 return "success";  
}  
//最终输出的内容为-->id:1,username:admin
```


# 四、SpringMVC获取请求参数

## 4.1 通过ServletAPI获取
将HttpServletRequest作为控制器方法的形参，此时HttpServletRequest类型的参数表示封装了当前请求的请求报文的对象。
```java
@RequestMapping("/testParam")
public String testParam(HttpServletRequest request){
    String username = request.getParameter("username");
    String password = request.getParameter("password");
    System.out.println("username:"+username+",password:"+password);
    return "success";
}
```


## 4.2 通过控制器方法的形参获取请求参数
在控制器方法的形参位置，设置和请求参数同名的形参，当浏览器发送请求，匹配到请求映射时，在DispatcherServlet中就会将请求参数赋值给相应的形参
```java
@RequestMapping("/testParam")
public String testParam(String username, String password){
 System.out.println("username:"+username+",password:"+password);
 return "success";
}
```

>注：
>
若请求所传输的请求参数中有多个同名的请求参数，此时可以在控制器方法的形参中设置字符串数组或者字符串类型的形参接收此请求参数
若使用字符串数组类型的形参，此参数的数组中包含了每一个数据
若使用字符串类型的形参，此参数的值为每个数据中间使用逗号拼接的结果

## 4.3 @RequestParam

@RequestParam是将请求参数和控制器方法的形参创建映射关系

@RequestParam注解一共有三个属性：

value：指定为形参赋值的请求参数的参数名

required：设置是否必须传输此请求参数，默认值为true

若设置为true时，则当前请求必须传输value所指定的请求参数，若没有传输该请求参数，且没有设置defaultValue属性，则页面报错400：Required String parameter 'xxx' is not present；若设置为false，则当前请求不是必须传输value所指定的请求参数，若没有传输，则注解所标识的形参的值为null

defaultValue：不管required属性值为true或false，当value所指定的请求参数没有传输或传输的值为""时，则使用默认值为形参赋值

## 4.4 @RequestHeader

@RequestHeader是将请求头信息和控制器方法的形参创建映射关系

@RequestHeader注解一共有三个属性：value、required、defaultValue，用法同@RequestParam

## 4.5 @CookieValue

@CookieValue是将cookie数据和控制器方法的形参创建映射关系

@CookieValue注解一共有三个属性：value、required、defaultValue，用法同@RequestParam

## 4.6 通过POJO获取请求参数
可以在控制器方法的形参位置设置一个实体类类型的形参，此时若浏览器传输的请求参数的参数名和实体类中的属性名一致，那么请求参数就会为此属性赋值

```html
<form th:action="@{/testpojo}" method="post">  
 用户名：<input type="text" name="username"><br>  
 密码：<input type="password" name="password"><br>  
 性别：<input type="radio" name="sex" value="男">男<input type="radio" name="sex" value="女">女<br>  
 年龄：<input type="text" name="age"><br>  
 邮箱：<input type="text" name="email"><br>  
 <input type="submit">  
</form>

```

```java
@RequestMapping("/testpojo")  
public String testPOJO(User user){  
 System.out.println(user);  
 return "success";  
}  
//最终结果-->User{id=null, username='张三', password='123', age=23, sex='男', email='123@qq.com'}
```


## 4.7 解决获取请求参数的乱码问题

解决获取请求参数的乱码问题，可以使用SpringMVC提供的编码过滤器CharacterEncodingFilter，但是必须在web.xml中进行注册

```xml
<!--配置springMVC的编码过滤器-->  
<filter>  
 <filter-name>CharacterEncodingFilter</filter-name>  
 <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
 <init-param>  
 <param-name>encoding</param-name>  
 <param-value>UTF-8</param-value>  
 </init-param>  
 <init-param>  
 <param-name>forceResponseEncoding</param-name>  
 <param-value>true</param-value>  
 </init-param>  
</filter>  
<filter-mapping>  
 <filter-name>CharacterEncodingFilter</filter-name>  
 <url-pattern>/*</url-pattern>  
</filter-mapping>
```

> 注：
> 
> SpringMVC中处理编码的过滤器一定要配置到其他过滤器之前，否则无效

# 五、域对象共享数据
## 5.1 使用ServletAPI向request域对象共享数据
```java
@RequestMapping("/testServletAPI")
public String testServletAPI(HttpServletRequest request){
    request.setAttribute("testScope", "hello,servletAPI");
    return "success";
}
```

## 5.2 使用ModelAndView向request域对象共享数据
```java
@RequestMapping("/testModelAndView")
public ModelAndView testModelAndView(){
    /**
     * ModelAndView有Model和View的功能
     * Model主要用于向请求域共享数据
     * View主要用于设置视图，实现页面跳转
     */
    ModelAndView mav = new ModelAndView();
    //向请求域共享数据
    mav.addObject("testScope", "hello,ModelAndView");
    //设置视图，实现页面跳转
    mav.setViewName("success");
    return mav;
}
```

## 5.3 使用Model向request域对象共享数据
```java
@RequestMapping("/testModel")
public String testModel(Model model){
    model.addAttribute("testScope", "hello,Model");
    return "success";
}
```

## 5.4 使用map向request域对象共享数据
```java
@RequestMapping("/testMap")
public String testMap(Map<String, Object> map){
    map.put("testScope", "hello,Map");
    return "success";
}
```

## 5.5 使用ModelMap向request域对象共享数据
```java
@RequestMapping("/testModelMap")  
public String testModelMap(ModelMap modelMap){  
 modelMap.addAttribute("testScope", "hello,ModelMap");  
 return "success";  
}
```

## 5.6 Model、ModelMap、Map的关系

Model、ModelMap、Map类型的参数其实本质上都是 BindingAwareModelMap 类型的
```
public interface Model{}  
public class ModelMap extends LinkedHashMap<String, Object> {}  
public class ExtendedModelMap extends ModelMap implements Model {}  
public class BindingAwareModelMap extends ExtendedModelMap {}
```

## 5.7 向session域共享数据
```java
@RequestMapping("/testSession")  
public String testSession(HttpSession session){  
 session.setAttribute("testSessionScope", "hello,session");  
 return "success";  
}
```
## 5.8 向application域共享数据
```java
@RequestMapping("/testApplication")  
public String testApplication(HttpSession session){  
 ServletContext application = session.getServletContext();  
 application.setAttribute("testApplicationScope", "hello,application");  
 return "success";  
}
```


# 六、SpringMVC视图
SpringMVC中的视图是View接口，视图的作用渲染数据，将模型Model中的数据展示给用户

SpringMVC视图的种类很多，默认有转发视图和重定向视图

当工程引入jstl的依赖，转发视图会自动转换为JstlView

若使用的视图技术为Thymeleaf，在SpringMVC的配置文件中配置了Thymeleaf的视图解析器，由此视图解析器解析之后所得到的是ThymeleafView

## 6.1 ThymeleafView

当控制器方法中所设置的视图名称没有任何前缀时，此时的视图名称会被SpringMVC配置文件中所配置的视图解析器解析，视图名称拼接视图前缀和视图后缀所得到的最终路径，会通过转发的方式实现跳转
```java
@RequestMapping("/testHello")  
public String testHello(){  
 return "hello";  
}
```

## 6.2 转发视图

SpringMVC中默认的转发视图是InternalResourceView

SpringMVC中创建转发视图的情况：

当控制器方法中所设置的视图名称以"forward:"为前缀时，创建InternalResourceView视图，此时的视图名称不会被SpringMVC配置文件中所配置的视图解析器解析，而是会将前缀"forward:"去掉，剩余部分作为最终路径通过转发的方式实现跳转

**若转到页面，需要加后缀.html**
例如"forward:/"，"forward:/employee"
```java
@RequestMapping("/testForward")  
public String testForward(){  
 return "forward:/testHello";  
}
```


## 6.3 重定向视图

SpringMVC中默认的重定向视图是RedirectView

当控制器方法中所设置的视图名称以"redirect:"为前缀时，创建RedirectView视图，此时的视图名称不会被SpringMVC配置文件中所配置的视图解析器解析，而是会将前缀"redirect:"去掉，剩余部分作为最终路径通过重定向的方式实现跳转

例如"redirect:/"，"redirect:/employee"
```java
@RequestMapping("/testRedirect")  
public String testRedirect(){  
 return "redirect:/testHello";  
}
```

>注：
>
重定向视图在解析时，会先将redirect:前缀去掉，然后会判断剩余部分是否以/开头，若是则会自动拼接上下文路径


## 6.4 重定向和转发区别
1、请求次数：重定向是浏览器向服务器发送一个请求并收到响应后再次向一个新地址发出请求，转发是服务器收到请求后为了完成响应跳转到一个新的地址；重定向至少请求两次，转发请求一次；

2、地址栏不同：重定向地址栏会发生变化，转发地址栏不会发生变化；（转发到链接会变，转发页面不变）

3、是否共享数据：重定向两次请求不共享数据，转发一次请求共享数据（在request级别使用信息共享，使用重定向必然出错）；

4、跳转限制：重定向可以跳转到任意URL，转发只能跳转本站点资源；

5、发生行为不同：重定向是客户端行为，转发是服务器端行为；



## 6.6 视图控制器view-controller

当控制器方法中，仅仅用来实现页面跳转，即只需要设置视图名称时，可以将处理器方法使用view-controller标签进行表示

<!--  
 path：设置处理的请求地址  
 view-name：设置请求地址所对应的视图名称  
-->  
<mvc:view-controller path="/testView" view-name="success"></mvc:view-controller>

> 注：
> 
> 当SpringMVC中设置任何一个view-controller时，其他控制器中的请求映射将全部失效，此时需要在SpringMVC的核心配置文件中设置开启mvc注解驱动的标签：
> 
> <mvc:annotation-driven />



# 七、RESTful
[RESTful到底是什么？ - 张瑞丰 - 博客园 (cnblogs.com)](https://www.cnblogs.com/zhangruifeng/p/13257731.html)
## 7.1 REST
REST（Representational State Transfer）表象化状态转变（表述性状态转变）
基于HTTP、URI、XML、JSON等标准和协议，支持轻量级、跨平台、跨语言的架构设计。是Web服务的一种新的架构风格（一种思想）。

## 7.2 RESTful架构的主要原则

-   对网络上所有的资源都有一个资源标志符。  （url）
-   对资源的操作不会改变标识符。
-   同一资源有多种表现形式（xml、json）
-   所有操作都是无状态的（Stateless，客户端和服务器端不必保存对方的详细信息）

符合上述REST原则的架构方式称为RESTful
RESTful是一种架构的规范与约束、原则，符合这种规范的架构就是RESTful架构。

## 7.3 RESTful实现

### 7.3.1 RESTful资源操作
| http方法 | 资源操作 | 幂等 | 安全 |
|:-------- |:-------- |:---- |:---- |
| GET      | SELECT   | 是   | 是   |
| POST     | INSERT   | 否   | 否   |
| PUT      | UPDATE   | 是   | 否   |
| DELETE   | DELETE   | 是   | 否   |


### 7.3.2 URL设计准则

* 宾语必须是名次
* 宾语复数较好
* 避免多级URL（/authors/12?categories=2，除了第一级，其他级别都用查询字符串表达。）


### 7.3.3 HiddenHttpMethodFilter

由于浏览器只支持发送get和post方式的请求，那么该如何发送put和delete请求呢？

SpringMVC 提供了 **HiddenHttpMethodFilter** 帮助我们**将 POST 请求转换为 DELETE 或 PUT 请求**

**HiddenHttpMethodFilter** 处理put和delete请求的条件：

a>当前请求的请求方式必须为post

b>当前请求必须传输请求参数_method

满足以上条件，**HiddenHttpMethodFilter** 过滤器就会将当前请求的请求方式转换为请求参数_method的值，因此请求参数_method的值才是最终的请求方式

在web.xml中注册**HiddenHttpMethodFilter**

<filter>  
 <filter-name>HiddenHttpMethodFilter</filter-name>  
 <filter-class>org.springframework.web.filter.HiddenHttpMethodFilter</filter-class>  
</filter>  
<filter-mapping>  
 <filter-name>HiddenHttpMethodFilter</filter-name>  
 <url-pattern>/*</url-pattern>  
</filter-mapping>

> 注：
> 
> 目前为止，SpringMVC中提供了两个过滤器：CharacterEncodingFilter和HiddenHttpMethodFilter
> 
> 在web.xml中注册时，必须先注册CharacterEncodingFilter，再注册HiddenHttpMethodFilter
> 
> 原因：
> 
> -   在 CharacterEncodingFilter 中通过 request.setCharacterEncoding(encoding) 方法设置字符集的
>     
> -   request.setCharacterEncoding(encoding) 方法要求前面不能有任何获取请求参数的操作
>     
> -   而 HiddenHttpMethodFilter 恰恰有一个获取请求方式的操作：
>     
> -   String paramValue = request.getParameter(this.methodParam);
>



# 八、HttpMessageConverter

HttpMessageConverter，报文信息转换器，将请求报文转换为Java对象，或将Java对象转换为响应报文

HttpMessageConverter提供了两个注解和两个类型：@RequestBody，@ResponseBody，RequestEntity，

## 8.1、@RequestBody

@RequestBody可以获取请求体，需要在控制器方法设置一个形参，使用@RequestBody进行标识，当前请求的请求体就会为当前注解所标识的形参赋值

<form th:action="@{/testRequestBody}" method="post">  
 用户名：<input type="text" name="username"><br>  
 密码：<input type="password" name="password"><br>  
 <input type="submit">  
</form>

```java
@RequestMapping("/testRequestBody")  
public String testRequestBody(@RequestBody String requestBody){  
 System.out.println("requestBody:"+requestBody);  
 return "success";  
}
```

输出结果：

requestBody:username=admin&password=123456

## 8.2 RequestEntity

RequestEntity封装请求报文的一种类型，需要在控制器方法的形参中设置该类型的形参，当前请求的请求报文就会赋值给该形参，可以通过getHeaders()获取请求头信息，通过getBody()获取请求体信息

```java
@RequestMapping("/testRequestEntity")  
public String testRequestEntity(RequestEntity<String> requestEntity){  
 System.out.println("requestHeader:"+requestEntity.getHeaders());  
 System.out.println("requestBody:"+requestEntity.getBody());  
 return "success";  
}
```

输出结果： requestHeader:[host:"localhost:8080", connection:"keep-alive", content-length:"27", cache-control:"max-age=0", sec-ch-ua:"" Not A;Brand";v="99", "Chromium";v="90", "Google Chrome";v="90"", sec-ch-ua-mobile:"?0", upgrade-insecure-requests:"1", origin:"[http://localhost:8080](http://localhost:8080)", user-agent:"Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.93 Safari/537.36"] requestBody:username=admin&password=123

## 8.3 @ResponseBody

@ResponseBody用于标识一个控制器方法，可以将该方法的返回值直接作为响应报文的响应体响应到浏览器

```
@RequestMapping("/testResponseBody")  
@ResponseBody  
public String testResponseBody(){  
 return "success";  
}
```


结果：浏览器页面显示success
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220228001316.png)

## 8.4 SpringMVC处理json

@ResponseBody处理json的步骤：

a>导入jackson的依赖
```yaml
<dependency>  
    <groupId>com.fasterxml.jackson.core</groupId>  
    <artifactId>jackson-databind</artifactId>  
    <version>2.12.1</version>  
</dependency>  
<dependency>  
    <groupId>com.fasterxml.jackson.core</groupId>  
    <artifactId>jackson-core</artifactId>  
    <version>2.12.1</version>  
</dependency>
```

b>在SpringMVC的核心配置文件中开启mvc的注解驱动，此时在HandlerAdaptor中会自动装配一个消息转换器：MappingJackson2HttpMessageConverter，可以将响应到浏览器的Java对象转换为Json格式的字符串
```xml
<mvc:annotation-driven />
```

c>在处理器方法上使用@ResponseBody注解进行标识

d>将Java对象直接作为控制器方法的返回值返回，就会自动转换为Json格式的字符串
```java
@RequestMapping("/success")  
@ResponseBody  
public User success(){  
    User user = new User("minangong",13);  
    return user;  
}
```

浏览器的页面中展示的结果：
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220228004808.png)

## 8.5 SpringMVC处理ajax
控制器方法：
```java
@RequestMapping("/testAjax")  
@ResponseBody  
public String testAjax(String username, String password){  
 System.out.println("username:"+username+",password:"+password);  
 return "hello,ajax";  
}
```


## 8.6 @RestController注解

@RestController注解是springMVC提供的一个复合注解，标识在控制器的类上，就相当于为类添加了@Controller注解，并且为其中的每个方法添加了@ResponseBody注解

## 8.7 ResponseEntity

ResponseEntity用于控制器方法的返回值类型，该控制器方法的返回值就是响应到浏览器的响应报文




# 九、文件上传与下载

## 9.1 文件上传
使用ResponseEntity实现下载文件的功能
```java
@RequestMapping("/testDown")  
public ResponseEntity<byte[]> testResponseEntity(HttpSession session) throws IOException {  
 //获取ServletContext对象  
 ServletContext servletContext = session.getServletContext();  
 //获取服务器中文件的真实路径  
 String realPath = servletContext.getRealPath("/static/img/1.jpg");  
 //创建输入流  
 InputStream is = new FileInputStream(realPath);  
 //创建字节数组  
 byte[] bytes = new byte[is.available()];  
 //将流读到字节数组中  
 is.read(bytes);  
 //创建HttpHeaders对象设置响应头信息  
 MultiValueMap<String, String> headers = new HttpHeaders();  
 //设置要下载方式以及下载文件的名字  
 headers.add("Content-Disposition", "attachment;filename=1.jpg");  
 //设置响应状态码  
 HttpStatus statusCode = HttpStatus.OK;  
 //创建ResponseEntity对象  
 ResponseEntity<byte[]> responseEntity = new ResponseEntity<>(bytes, headers, statusCode);  
 //关闭输入流  
 is.close();  
 return responseEntity;  
}

```

## 9.2 文件上传

文件上传要求form表单的请求方式必须为post，并且添加属性enctype="multipart/form-data"

SpringMVC中将上传的文件封装到MultipartFile对象中，通过此对象可以获取文件相关信息

上传步骤：

a>添加依赖：

```yaml
<!-- https://mvnrepository.com/artifact/commons-fileupload/commons-fileupload -->  
<dependency>  
 <groupId>commons-fileupload</groupId>  
 <artifactId>commons-fileupload</artifactId>  
 <version>1.3.1</version>  
</dependency>
```

b>在SpringMVC的配置文件中添加配置：
```xml
<!--必须通过文件解析器的解析才能将文件转换为MultipartFile对象-->  
<bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver"></bean>
```

c>控制器方法：

```java
@RequestMapping("/testUp")  
public String testUp(MultipartFile photo, HttpSession session) throws IOException {  
 //获取上传的文件的文件名  
 String fileName = photo.getOriginalFilename();  
 //处理文件重名问题  
 String hzName = fileName.substring(fileName.lastIndexOf("."));  
 fileName = UUID.randomUUID().toString() + hzName;  
 //获取服务器中photo目录的路径  
 ServletContext servletContext = session.getServletContext();  
 String photoPath = servletContext.getRealPath("photo");  
 File file = new File(photoPath);  
 if(!file.exists()){  
 file.mkdir();  
 }  
 String finalPath = photoPath + File.separator + fileName;  
 //实现上传功能  
 photo.transferTo(new File(finalPath));  
 return "success";  
}

```

# 十、拦截器
## 10.1 拦截器的配置

SpringMVC中的拦截器用于拦截控制器方法的执行

SpringMVC中的拦截器需要实现HandlerInterceptor

SpringMVC的拦截器必须在SpringMVC的配置文件中进行配置：
```xml
<bean class="com.atguigu.interceptor.FirstInterceptor"></bean>  
<ref bean="firstInterceptor"></ref>  
<!-- 以上两种配置方式都是对DispatcherServlet所处理的所有的请求进行拦截 -->  
<mvc:interceptor>  
 <mvc:mapping path="/**"/>  
 <mvc:exclude-mapping path="/testRequestEntity"/>  
 <ref bean="firstInterceptor"></ref>  
</mvc:interceptor>  
<!--   
 以上配置方式可以通过ref或bean标签设置拦截器，通过mvc:mapping设置需要拦截的请求，通过mvc:exclude-mapping设置需要排除的请求，即不需要拦截的请求  
-->
```

## 10.2 拦截器的三个抽象方法

SpringMVC中的拦截器有三个抽象方法：
preHandle：控制器方法执行之前执行preHandle()，其boolean类型的返回值表示是否拦截或放行，返回true为放行，即调用控制器方法；返回false表示拦截，即不调用控制器方法
postHandle：控制器方法执行之后执行postHandle()
afterComplation：处理完视图和模型数据，渲染视图完毕之后执行

## 10.3 多个拦截器的执行顺序

a>若每个拦截器的preHandle()都返回true

此时多个拦截器的执行顺序和拦截器在SpringMVC的配置文件的配置顺序有关：
preHandle()会按照配置的顺序执行，而postHandle()和afterComplation()会按照配置的反序执行

b>若某个拦截器的preHandle()返回了false

preHandle()返回false和它之前的拦截器的preHandle()都会执行，postHandle()都不执行，返回false的拦截器之前的拦截器的afterComplation()会执行


# 十一、异常处理器

## 11.1 基于配置的异常处理

SpringMVC提供了一个处理控制器方法执行过程中所出现的异常的接口：HandlerExceptionResolver

HandlerExceptionResolver接口的实现类有：DefaultHandlerExceptionResolver和SimpleMappingExceptionResolver

SpringMVC提供了自定义的异常处理器SimpleMappingExceptionResolver，使用方式：

```xml
<bean class="org.springframework.web.servlet.handler.SimpleMappingExceptionResolver">  
 <property name="exceptionMappings">  
 <props>  
 <!--  
 properties的键表示处理器方法执行过程中出现的异常  
 properties的值表示若出现指定异常时，设置一个新的视图名称，跳转到指定页面  
 -->  
 <prop key="java.lang.ArithmeticException">error</prop>  
 </props>  
 </property>  
 <!--  
 exceptionAttribute属性设置一个属性名，将出现的异常信息在请求域中进行共享  
 -->  
 <property name="exceptionAttribute" value="ex"></property>  
</bean>
```


控制器：（模拟错误）
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220228115933.png)

前端：
```html
<!DOCTYPE html>  
<html lang="en" xmlns:th="http://www.thymeleaf.org">  
<head>  
    <meta charset="UTF-8">  
    <title>Title</title>  
</head>  
<body>  
出现错误  
<p th:text="${ex}"></p>  
</body>  
</html>
```
结果：
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301003139.png)

## 11.2 基于注解的异常处理

```java
//@ControllerAdvice将当前类标识为异常处理的组件
@ControllerAdvice
public class ExceptionController {

    //@ExceptionHandler用于设置所标识方法处理的异常
    @ExceptionHandler(ArithmeticException.class)
    //ex表示当前请求处理中出现的异常对象
    public String handleArithmeticException(Exception ex, Model model){
        model.addAttribute("ex", ex);
        return "error";
    }

}
```


# 十二、注解配置SpringMVC

使用配置类和注解代替web.xml和SpringMVC配置文件的功能

## 12.1 创建初始化类，代替web.xml
```java
public class WebInit extends AbstractAnnotationConfigDispatcherServletInitializer {

    /**
     * 指定spring的配置类
     * @return
     */
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{SpringConfig.class};
    }

    /**
     * 指定SpringMVC的配置类
     * @return
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    /**
     * 指定DispatcherServlet的映射规则，即url-pattern
     * @return
     */
    @Override
    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

    /**
     * 添加过滤器
     * @return
     */
    @Override
    protected Filter[] getServletFilters() {
        CharacterEncodingFilter encodingFilter = new CharacterEncodingFilter();
        encodingFilter.setEncoding("UTF-8");
        encodingFilter.setForceRequestEncoding(true);
        HiddenHttpMethodFilter hiddenHttpMethodFilter = new HiddenHttpMethodFilter();
        return new Filter[]{encodingFilter, hiddenHttpMethodFilter};
    }
}
```
## 12.2 创建SpringConfig配置类，代替spring的配置文件

```java
@Configuration
public class SpringConfig {
	//ssm整合之后，spring的配置信息写在此类中
}
```

## 12.3 创建WebConfig配置类，代替SpringMVC的配置文件
```java
/**  
 * 代替SpringMVC的配置文件：  
 * 1、扫描组件 2、视图解析器 3、view-controller    4、default-servlet-handler  
 * 5、mvc注解驱动 6、文件上传解析器 7、异常处理 8、拦截器  
 */  
//将当前类标识为一个配置类  
@Configuration  
//1、扫描组件  
@ComponentScan("com.mng.controller")  
//5、mvc注解驱动  
@EnableWebMvc  
public class WebConfig implements WebMvcConfigurer {  
    //4.default-servlet-handler  
 @Override  
 public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {  
        configurer.enable();  
    }  
  
    //8、拦截器  
 @Override  
 public void addInterceptors(InterceptorRegistry registry) {  
        TestInterceptor testInterceptor = new TestInterceptor();  
        registry.addInterceptor(testInterceptor).addPathPatterns("/**");  
    }  
  
    //3.view-controller  
 @Override  
 public void addViewControllers(ViewControllerRegistry registry) {  
        registry.addViewController("/hello").setViewName("hello");  
    }  
  
    //6、文件上传解析器  
 @Bean  
 public MultipartResolver multipartResolver(){  
        CommonsMultipartResolver commonsMultipartResolver = new CommonsMultipartResolver();  
        return commonsMultipartResolver;  
    }  
  
    //7.异常处理  
 @Override  
 public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {  
  
        SimpleMappingExceptionResolver simpleMappingExceptionResolver = new SimpleMappingExceptionResolver();  
        Properties props = new Properties();  
        props.setProperty("java.lang.ArithmeticException", "error");  
        simpleMappingExceptionResolver.setExceptionMappings(props);  
        simpleMappingExceptionResolver.setExceptionAttribute("ex");  
        resolvers.add(simpleMappingExceptionResolver);  
    }  
  
    //模板解析器  
 @Bean  
 public ITemplateResolver templateResolver(){  
        WebApplicationContext webApplicationContext = ContextLoader.getCurrentWebApplicationContext();  
        ServletContextTemplateResolver templateResolver = new ServletContextTemplateResolver(webApplicationContext.getServletContext());  
        templateResolver.setPrefix("WEB-INF/templates/");  
        templateResolver.setSuffix(".html");  
        templateResolver.setCharacterEncoding("UTF-8");  
        templateResolver.setTemplateMode(TemplateMode.HTML);  
        return templateResolver;  
    }  
    //生成模板引擎 并为模板引擎注入模板解析器  
 @Bean  
 public SpringTemplateEngine templateEngine(ITemplateResolver templateResolver){  
        SpringTemplateEngine templateEngine = new SpringTemplateEngine();  
        templateEngine.addTemplateResolver(templateResolver);  
        return templateEngine;  
    }  
    //生成视图解析器 并注入模板引擎  
 @Bean  
 public ViewResolver viewResolver(SpringTemplateEngine templateEngine){  
        ThymeleafViewResolver viewResolver = new ThymeleafViewResolver();  
        viewResolver.setTemplateEngine(templateEngine);  
        return viewResolver;  
    }  
  
  
  
}
```


# 十三、SpringMVC执行流程

## 13.1 SpringMVC常用组件
-   DispatcherServlet：**前端控制器**，不需要工程师开发，由框架提供

作用：统一处理请求和响应，整个流程控制的中心，由它调用其它组件处理用户的请求

-   HandlerMapping：**处理器映射器**，不需要工程师开发，由框架提供

作用：根据请求的url、method等信息查找Handler，即控制器方法

-   Handler：**处理器**，需要工程师开发
    
作用：在DispatcherServlet的控制下Handler对具体的用户请求进行处理

-   HandlerAdapter：**处理器适配器**，不需要工程师开发，由框架提供
    
作用：通过HandlerAdapter对处理器（控制器方法）进行执行

-   ViewResolver：**视图解析器**，不需要工程师开发，由框架提供
    
作用：进行视图解析，得到相应的视图，例如：ThymeleafView、InternalResourceView、RedirectView

-   View：**视图**
    
作用：将模型数据通过页面展示给用户

## 13.2 DispatcherServlet调用组件处理请求
[SpringMVC 源码解析笔记 - Seazean - 博客园 (cnblogs.com)](https://www.cnblogs.com/seazean/p/15095819.html)
## 13.3 SpringMVC的执行流程

1) 用户向服务器发送请求，请求被SpringMVC 前端控制器 DispatcherServlet捕获。

2) DispatcherServlet对请求URL进行解析，得到请求资源标识符（URI），判断请求URI对应的映射：

a) 不存在

i. 再判断是否配置了mvc:default-servlet-handler

ii. 如果没配置，则控制台报映射查找不到，客户端展示404错误
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301231039.png)


iii. 如果有配置，则访问目标资源（一般为静态资源，如：JS,CSS,HTML），找不到客户端也会展示404错误
![](https://raw.githubusercontent.com/minangong/mng_images/main/images2/20220301231039.png)



b) 存在则执行下面的流程

3) 根据该URI，调用HandlerMapping获得该Handler配置的所有相关的对象（包括Handler对象以及Handler对象对应的拦截器），最后以HandlerExecutionChain执行链对象的形式返回。

4) DispatcherServlet 根据获得的Handler，选择一个合适的HandlerAdapter。

5) 如果成功获得HandlerAdapter，此时将开始执行拦截器的preHandler(…)方法【正向】

6) 提取Request中的模型数据，填充Handler入参，开始执行Handler（Controller)方法，处理请求。在填充Handler的入参过程中，根据你的配置，Spring将帮你做一些额外的工作：

a) HttpMessageConveter： 将请求消息（如Json、xml等数据）转换成一个对象，将对象转换为指定的响应信息

b) 数据转换：对请求消息进行数据转换。如String转换成Integer、Double等

c) 数据格式化：对请求消息进行数据格式化。 如将字符串转换成格式化数字或格式化日期等

d) 数据验证： 验证数据的有效性（长度、格式等），验证结果存储到BindingResult或Error中

7) Handler执行完成后，向DispatcherServlet 返回一个ModelAndView对象。

8) 此时将开始执行拦截器的postHandle(...)方法【逆向】。

9) 根据返回的ModelAndView（此时会判断是否存在异常：如果存在异常，则执行HandlerExceptionResolver进行异常处理）选择一个适合的ViewResolver进行视图解析，根据Model和View，来渲染视图。

10) 渲染视图完毕执行拦截器的afterCompletion(…)方法【逆向】。

11) 将渲染结果返回给客户端。


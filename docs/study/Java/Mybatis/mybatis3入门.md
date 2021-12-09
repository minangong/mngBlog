# mybatis3入门

[MyBatis 3 文档](https://mybatis.org/mybatis-3/zh/configuration.html)

# 一、简介

## 1.mybatis

* MyBatis 是支持定制化 SQL、存储过程以及高级 映射的优秀的持久层框架。 

* MyBatis 避免了几乎所有的 JDBC 代码和手动设 置参数以及获取结果集。
*  MyBatis可以使用简单的XML或注解用于配置和原 始映射，将接口和Java的POJO（Plain Old Java  Objects，普通的Java对象）映射成数据库中的记 录.

## 2.为什么要使用mybatis

* MyBatis是一个**半自动化**的持久化层框架。 

  ![image-20211011001224929](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211011001224929.png)

* JDBC 

  ![image-20211011000630449](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211011000630449.png)

  – SQL夹在Java代码块里，**耦合度高**导致硬编码内伤 

  – 维护不易且实际开发需求中**sql是有变化**，频繁修改的情况多见 

* Hibernate和JPA 

  ![image-20211011001246014](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211011001246014.png)

  – **长难复杂SQL**，对于Hibernate而言处理也不容易 

  – 内部**自动生产**的SQL，不容易做特殊优化。 

  – 基于全映射的**全自动**框架，大量字段的POJO进行**部分映射**时比较**困难**。 导致数据库性能下降。 

* 对开发人员而言，**核心sql**还是需要自己优化,自己**定制**
* **sql和java编码分开**，功能边界清晰，一个专注业务、 一个专注数据。



# 二. mybatis 使用流程

## 1.从 XML 文件中构建 SqlSessionFactory 的实例

​        每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为核心的。SqlSessionFactory 的实例可以通过 SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的 Configuration 实例来构建出 SqlSessionFactory 实例。

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```



## 2. 从 SqlSessionFactory 中获取 SqlSession

​      SqlSession 提供了在数据库执行 SQL 命令所需的所有方法。你可以通过 SqlSession 实例来直接执行已映射的 SQL 语句。一个 SqlSession对象代表和数据库的一次会话。

​      SqlSession非线程安全，所以不能写在共享的成员变量中，应该每次用时在方法里创建对象。

```java
SqlSession sqlSession = sqlSessionFactory.openSession();

/*
  执行SQL语句 方法1
  只需要编写mapper.xml
  第一个参数 SQL statement 唯一标识符
  但缺少参数类型判断
*/
Employee employee = (Employee) sqlSession.selectOne("com.mng.mybatistest.mapper.EmployeeMapper.getEmployeeById", 1L);
System.out.println(employee);
sqlSession.close();
 /*
   执行SQL语句方法二   接口式编程
   mapper：为接口自动创建的动态代理对象。将接口和xml原本就绑定了
   保证类型安全
*/
EmployeeMapper mapper = sqlSession.getMapper(EmployeeMapper.class);
Employee employee = mapper.getEmployeeById(1L);
System.out.println(employee);
sqlSession.close();

```

## 代码附录

### Employee实体类

```java
@Data
public class Employee {
    private Long id;
    String lastname;
    String gender;
    String email;
}
```

### mapper接口（DAO层）

```java
@Mapper
public interface EmployeeMapper {
    public Employee getEmployeeById(Long Id);
}
```

### mybatis-config.xml  mybatis全局配置文件

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
 PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
 "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--default：默认情况下用的数据源-->
	<environments default="development">
        <!--数据源配置，id是标识符-->
		<environment id="development">
            <!--使用JDBC事务管理器-->
			<transactionManager type="JDBC" />
            <!-- 数据库连接池 -->
			<dataSource type="POOLED">
				<property name="driver" value="com.mysql.jdbc.Driver" />
				<property name="url" value="jdbc:mysql://localhost:3306/mybatis" />
				<property name="username" value="root" />
				<property name="password" value="123456" />
			</dataSource>
		</environment>
	</environments>
	<!-- 将我们写好的sql映射文件（EmployeeMapper.xml）一定要注册到全局配置文件（mybatis-config.xml）中 -->
	<mappers>
		<mapper resource="EmployeeMapper.xml" />
	</mappers>
</configuration>
```

### SQL映射文件

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.mng.mybatistest.mapper.EmployeeMapper">
    <resultMap id="ResultMap" type="com.mng.mybatistest.model.Employee">
        <id column="id" jdbcType="BIGINT" property="id"/>
        <result column="last_name" jdbcType="VARCHAR" property="lastname"/>
        <result column="gender" jdbcType="VARCHAR" property="gender"/>
        <result column="email" jdbcType="VARCHAR" property="email"/>
    </resultMap>

    <select id="getEmployeeById" parameterType="Long" resultMap="ResultMap">
        select *
        from tbl_employee
        where id = #{id}
    </select>
</mapper>
```





# 三、程序规范和并发问题

## SqlSessionFactoryBuilder

​        一旦创建了SqlSessionFactory,就不需要了。最佳作用域：方法作用域

## SqlSessionFactory

​        SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，最佳作用域是应用作用域。最简单的就是使用单例模式或者静态单例模式。

## SqlSession

​       不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。不能作为类的对象

​       为了确保每次都能执行关闭操作，你应该把这个关闭操作放到 finally 块中。 下面的示例就是一个确保 SqlSession 关闭的标准模式：

```java
try (SqlSession session = sqlSessionFactory.openSession()) {
  // 你的应用逻辑代码
}finally{
    session.close();
}
```

## 映射器实例（mapper）

最大的作用域应该和SqlSession相同，方法作用域才是映射器实例的最合适的作用域。不需要显式关闭。

```
try (SqlSession session = sqlSessionFactory.openSession()) {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  // 你的应用逻辑代码
}finally{
    session.close();
}
```

# 四、全局配置文件 Configuration

## 1. 属性（properties）

属性可以在外部进行配置，并可以进行动态替换。

```
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
```

设置好的属性可以在整个配置文件中用来替换需要动态配置的属性值

```
<dataSource type="POOLED">
  <property name="driver" value="${driver}"/>
  <property name="url" value="${url}"/>
  <property name="username" value="${username}"/>
  <property name="password" value="${password}"/>
</dataSource>
```

也可以 SqlSessionFactoryBuilder.build() 方法中传入属性值



所以读取顺序：

* 1.properties元素体内指定属性

* 2.然后读取properties中**resource、url**中的配置，并覆盖同名字段（resource引入类路径下的资源，url引入网络路径或磁盘路径下的资源）

* 3.然后传入的属性值，覆盖同名字段

所以mybatis-config.xml的配置可以在.properties文件（或者yaml文件）中配置，spring把这个xml的工作做了。

![image-20211016213747670](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211016213747670.png)

## 2.设置（Settings）

MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为。

具体设置项查看文档。

```xml
<Settings>
    <Setting  name="mapUnderscoreToCamelCase" value="true"/>
</Settings>
```



## 3.类型别名（typeAliases）

```xml
<typeAliases>
  <!--为Java类型设置缩写别名-->
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <!--指定一个包名，MyBatis 会在包名下面搜索需要的 Java Bean-->
  <package name = "domain.blog">
</typeAliases>
```

```
@Alias("author")
//注解起别名
```

## 4.类型处理器（typeHandlers）

MyBatis 在设置预处理语句（PreparedStatement）中的参数或从结果集中取出一个值时， 用类型处理器将获取到的值以合适的方式转换成 Java 类型。



## 5.环境配置（environments） （多数据库 多数据源 但是mapper方法没有区分）

**可以配置多个环境，但每个 SqlSessionFactory 实例只能选择一种环境，即对应一个数据库。**

```java
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment, properties);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, properties);
```

**事务管理器**有两种：JDBC|MANAGED

* JDBC – 这个配置直接使用了 JDBC 的提交和回滚设施，它依赖从数据源获得的连接来管理事务作用域。

* MANAGED – 这个配置几乎没做什么。它从不提交或回滚一个连接，而是让容器来管理事务的整个生命周期（比如 JEE 应用服务器的上下文）。 默认情况下它会关闭连接。然而一些容器并不希望连接被关闭，因此需要将 closeConnection 属性设置为 false 来阻止默认的关闭行为。例如:

  ```
  <transactionManager type="MANAGED">
    <property name="closeConnection" value="false"/>
  </transactionManager>
  ```

**Spring + MyBatis，则没有必要配置事务管理器，因为 Spring 模块会使用自带的管理器来覆盖前面的配置**。

**dataSource type:**UNPOOLED|POOLED|JNDI

* UNPOOLED– 这个数据源的实现会每次请求时打开和关闭连接。
* POOLED– 这种数据源的实现利用“池”的概念将 JDBC 连接对象组织起来，避免了创建新的连接实例时所必需的初始化和认证时间。 这种处理方式很流行，能使并发 Web 应用快速响应请求。
* 

栗子：

首先,mybatis-config.xml中配置了两个数据源, 两个数据源数据不同（development lastname是asdddd  haha lastname 是mng）

![image-20211017000035744](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017000035744.png)



根据mybatis全局配置文件 和  emvironment id 生成SqlSessionFactory

![image-20211017003533099](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017003533099.png)



获得SqlSession、根据接口获得mapper后查表 结果：

![image-20211017010620558](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017010620558.png)

**多数据库 单数据源  成功**





## 6.数据库厂商标识（databaseIdProvider）

\<environment\>中可以配置多个数据源，但是mapper接口在调用方法时，对于不同数据源应该调用不同的SQL语句。根据环境的驱动，采用不同的SQL语句，所以SQL语句要加上数据库厂商的标识

![image-20211017013138499](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017013138499.png)

```xml
    <!--给数据源起别名-->
    <databaseIdProvider type="DB_VENDOR">
        <property name="MySQL" value="mysql"/>
        <property name="Oracle" value="oracle"/>
        <property name="SQL Server" value="sqlserver"/>
    </databaseIdProvider>
```

然后在mapper映射文件里给sql语句加上数据库厂商标识(感觉像多态)，会用到什么数据源，就定义个相应的sql语句，再打上数据库厂商标识。

```
    <select id="getEmployeeById" parameterType="Long" resultMap="ResultMap"
    databaseId="mysql">
        select *
        from tbl_employee
        where id = #{id}
    </select>
```

![image-20211017013613938](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017013613938.png)



过程，根据环境驱动获得数据库厂商标识，例如mysql,然后加载所有带mysql标识、不带标识的sql语句，如果sql语句id同名，那么加载更精确的，例如一个mysql，一个不带，则加载mysql的。



## 7.映射器（mappers）

将写好的sql映射文件注册到全局配置文件

```xml
<mappers>
  <!-- 使用相对于类路径的资源引用 引用类路径下的sql映射文件-->
  <mapper resource="org/mybatis/builder/AuthorMapper.xml"/>
  <mapper resource="mapper/EmployeeMapper.xml"/>
  
  <!-- 使用完全限定资源定位符（URL） 引用网络路径或者磁盘路径下的sql映射文件-->
  <mapper url="file:///var/mappers/AuthorMapper.xml"/>
  <mapper url="file:///var/mappers/BlogMapper.xml"/>
    
  <!-- 使用映射器接口实现类的完全限定类名   
       mapper.xml和接口必须在同一目录里，且同名 不然无法绑定-->
  <mapper class="org.mybatis.builder.AuthorMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  
  <!-- 将包内的映射器接口实现全部注册为映射器 -->
  <package name="org.mybatis.builder"/>
   
</mappers>
```

用class注册时有两种：有映射文件，方便维护，不用改源代码

​                                        没有映射文件，基于注解，开发容易。  

![image-20211017015426310](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017015426310.png)



例：

全局配置文件注册 注解的接口：

![image-20211017020555875](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017020555875.png)



接口定义：

![image-20211017020631004](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017020631004.png)

调用运行及结果：

![image-20211017020701214](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017020701214.png)

![image-20211017020720613](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017020720613.png)





映射文件和接口放在同一目录下，不好看--->方法：

代码最后都会合并到bin (idea 是target）文件目录下，所以可以接口放在src/包名下，映射文件放在conf/相同包名下。

![image-20211017020336230](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211017020336230.png)

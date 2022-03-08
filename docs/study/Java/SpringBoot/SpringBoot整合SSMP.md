# SpringBoot整合SSMP
 # 一、整合Junit

## 1.1 Spring整合Junit
```java
//加载spring整合junit专用的类运行器
@RunWith(SpringJUnit4ClassRunner.class)
//指定对应的配置信息
@ContextConfiguration(classes = SpringConfig.class)
public class AccountServiceTestCase {
    //注入你要测试的对象
    @Autowired
    private AccountService accountService;
    @Test
    public void testGetById(){
        //执行要测试的对象对应的方法
        System.out.println(accountService.findById(2));
    }
}
```
核心代码是前两个注解，第一个注解@RunWith是设置Spring专用于测试的类运行器.
第二个注解@ContextConfiguration是用来设置Spring核心配置文件或配置类。

## 1.2 SpringBoot整合Junit
使用SpringBoot整合JUnit需要保障导入test对应的starter。(初始化项目时默认导入的）
```java
@SpringBootTest
class Springboot04JunitApplicationTests {
    //注入你要测试的对象
    @Autowired
    private BookDao bookDao;
    @Test
    void contextLoads() {
        //执行要测试的对象对应的方法
        bookDao.save();
        System.out.println("two...");
    }
}
```
@SpringBootTest替换了前面两个注解,配置都是默认值。

加载的配置类或者配置文件就是前面启动程序使用的引导类。手动指定配置类：
```java
@SpringBootTest(classes = Springboot04JunitApplication.class)

```

```java
@SpringBootTest
@ContextConfiguration(classes = Springboot04JunitApplication.class)
```


# 二、整合Mybatis
## 2.1 Spring整合Mybatis
* 导入坐标，MyBatis坐标不能少，Spring整合MyBatis还有自己专用的坐标，此外Spring进行数据库操作的jdbc坐标是必须的，剩下还有mysql驱动坐标，本例中使用了Druid数据源，这个倒是可以不要
```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.1.16</version>
    </dependency>
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis</artifactId>
        <version>3.5.6</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <version>5.1.47</version>
    </dependency>
    <!--1.导入mybatis与spring整合的jar包-->
    <dependency>
        <groupId>org.mybatis</groupId>
        <artifactId>mybatis-spring</artifactId>
        <version>1.3.0</version>
    </dependency>
    <!--导入spring操作数据库必选的包-->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-jdbc</artifactId>
        <version>5.2.10.RELEASE</version>
    </dependency>
</dependencies>
```

* Spring核心配置
```java
@Configuration
@ComponentScan("com.itheima")
@PropertySource("jdbc.properties")
public class SpringConfig {
}

```

* MyBatis要交给Spring接管的配置类bean
```java
//定义mybatis专用的配置类
@Configuration
public class MyBatisConfig {
//    定义创建SqlSessionFactory对应的bean
    @Bean
    public SqlSessionFactoryBean sqlSessionFactory(DataSource dataSource){
        //SqlSessionFactoryBean是由mybatis-spring包提供的，专用于整合用的对象
        SqlSessionFactoryBean sfb = new SqlSessionFactoryBean();
        //设置数据源替代原始配置中的environments的配置
        sfb.setDataSource(dataSource);
        //设置类型别名替代原始配置中的typeAliases的配置
        sfb.setTypeAliasesPackage("com.itheima.domain");
        return sfb;
    }
//    定义加载所有的映射配置
    @Bean
    public MapperScannerConfigurer mapperScannerConfigurer(){
        MapperScannerConfigurer msc = new MapperScannerConfigurer();
        msc.setBasePackage("com.itheima.dao");
        return msc;
    }

}
```

* 数据源对应的bean
```java
@Configuration
public class JdbcConfig {
    @Value("${jdbc.driver}")
    private String driver;
    @Value("${jdbc.url}")
    private String url;
    @Value("${jdbc.username}")
    private String userName;
    @Value("${jdbc.password}")
    private String password;

    @Bean("dataSource")
    public DataSource dataSource(){
        DruidDataSource ds = new DruidDataSource();
        ds.setDriverClassName(driver);
        ds.setUrl(url);
        ds.setUsername(userName);
        ds.setPassword(password);
        return ds;
    }
}

```

* 数据库连接信息
```java
jdbc.driver=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/spring_db?useSSL=false
jdbc.username=root
jdbc.password=root
```

## 2.2 SpringBoot整合Mybatis
* 1. 创建模块时勾选要使用的技术，MyBatis，由于要操作数据库，还要勾选对应数据库
![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220306002249.png)

或者手工导入对应技术的starter，和对应数据库的坐标
```xml
<dependencies>
    <!--1.导入对应的starter-->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>2.2.0</version>
    </dependency>

    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

* 2.配置数据源相关信息，没有这个信息你连接哪个数据库都不知道

```yml
#2.配置相关信息  
spring:  
 datasource:  
 driver-class-name: com.mysql.cj.jdbc.Driver  
 url: jdbc:mysql://localhost:3306/ssm_db  
 username: root  
 password: root
```

**实体类**
```java
public class Book {  
 private Integer id;  
 private String type;  
 private String name;  
 private String description;  
}
```


**映射接口（Dao）**
```java
@Mapper  
public interface BookDao {  
 @Select("select * from tbl_book where id = #{id}")  
 public Book getById(Integer id);  
}
```

**测试类**
```java
@SpringBootTest  
class Springboot05MybatisApplicationTests {  
 @Autowired  
 private BookDao bookDao;  
 @Test  
 void contextLoads() {  
 System.out.println(bookDao.getById(1));  
 }  
}
```


1.  MySQL 8.X驱动强制要求设置时区
    -   修改url，添加serverTimezone设定
```
    url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=UTC 
	# (全球标准时间)
    url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=Asia/Shanghai

```
   *   修改MySQL数据库配置
2.  驱动类过时，提醒更换为com.mysql.cj.jdbc.Driver


# 三、整合Mybatis Plus
1. 导入对应的starter
```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version>
</dependency>
```
2. 配置数据源相关信息
```yaml
#2.配置相关信息
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ssm_db
    username: root
    password: root
```

**映射接口（Dao）**
```java
@Mapper  
public interface BookDao extends BaseMapper<Book> {  
}
```

ps: 数据库的表名定义规则是tbl_模块名称，为了能和实体类相对应，需要做一个配置.设置所有表名的通用前缀名.
```yaml
mybatis-plus:
  global-config:
    db-config:
      table-prefix: tbl_		#设置所有表的通用前缀名称为tbl_
```



# 四、整合Druid
## 4.1 切换数据源
1. 导入坐标
```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid</artifactId>
        <version>1.1.16</version>
    </dependency>
</dependencies>
```
2. 修改type配置
```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=UTC
    username: root
    password: root
    type: com.alibaba.druid.pool.DruidDataSource

```
## 4.2 SpringBoot整合Druid
1. 导入starter
```xml
<dependencies>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.2.6</version>
    </dependency>
</dependencies>
```

2. 修改配置
```yaml
spring:
  datasource:
    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=UTC
      username: root
      password: root
```



# 五、SSMP案例
[米南宫/BookManager (gitee.com)](https://gitee.com/minan-palace/BookManager)
## 5.1 实体类
```java
import lombok.Data;  
  
@Data  
public class Book {  
    private Integer id;  
    private String type;  
    private String name;  
    private String description;  
}

```

## 5.2 数据层
1. 首先导入依赖
```xml
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.4.3</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.2.5</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
```

2. 数据源连接的数据源配置
```yaml
spring:  
  datasource:  
    druid:  
      driver-class-name: com.mysql.cj.jdbc.Driver  
      url: jdbc:mysql://localhost:3306/mybatisdb?serverTimezone=UTC  
      username: root  
      password: 

```

3. 使用MP的标准通用接口BaseMapper加速开发
```java
@Mapper  
public interface BookDao extends BaseMapper<Book> {  
  
}
```

4. MP技术默认的主键生成策略为雪花算法，生成的主键ID长度较大，和目前的数据库设定规则不相符，需要配置一下使MP使用数据库的主键生成策略
```yaml
mybatis-plus:  
  global-config:  
    db-config:  
      table-prefix: tbl_  
      id-type: auto

```

5. 查看MP运行日志
```yaml
mybatis-plus:
  global-config:
    db-config:
      table-prefix: tbl_
      id-type: auto
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```


6. 测试Dao
not registered 、not be managered  是没有事务管理（@transactional注解）的原因
```txt
Creating a new SqlSession
SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@35f639fa] was not registered for synchronization because synchronization is not active
JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@3f725306] will not be managed by Spring
==>  Preparing: INSERT INTO tbl_book ( type, name, description ) VALUES ( ?, ?, ? )
==> Parameters: 测试type(String), 测试name(String), 测试description(String)
<==    Updates: 1
Closing non transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@35f639fa]
```



## 5.3 数据层--分页功能
MP将分页操作做成了一个开关，用分页功能要把开关开启。这个开关是通过MP的拦截器的形式存在的。
```java
@Configuration
public class MPConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor(){
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}

```
上述代码第一行是创建MP的拦截器栈，这个时候拦截器栈中没有具体的拦截器，第二行是初始化了分页拦截器，并添加到拦截器栈中。如果后期开发其他功能，需要添加全新的拦截器，按照第二行的格式继续add进去新的拦截器就可以了。

```java
@Test
void testGetPage(){
    IPage page = new Page(2,5);
    bookDao.selectPage(page, null);
    System.out.println(page.getCurrent());		//当前页码值
    System.out.println(page.getSize());			//每页显示数
    System.out.println(page.getTotal());		//数据总量
    System.out.println(page.getPages());		//总页数
    System.out.println(page.getRecords());		//详细数据
}

```


## 5.4 数据层--条件查询
```java
@Test  
public void selectSBooks(){  
    QueryWrapper<Book> queryWrapper = new QueryWrapper<>();  
    queryWrapper.like("name","Spring");  
    List<Book> books = bookDao.selectList(queryWrapper);  
    System.out.println(books);  
}
```
字段是用字符串格式，写错，编译器没有办法发现，只能将问题抛到运行器通过异常堆栈告诉开发者，不太友好。所以MP功能升级，由QueryWrapper对象升级为LambdaQueryWrapper对象,全面支持Lambda表达式。
```java
@Test  
public void selectSBooks2(){  
    LambdaQueryWrapper<Book> bookLambdaQueryWrapper = new LambdaQueryWrapper<>();  
    bookLambdaQueryWrapper.like(Book::getName,"Spring");  
    List<Book> books = bookDao.selectList(bookLambdaQueryWrapper);  
    System.out.println(books);  
}
```
为了防止将null数据作为条件使用，MP还提供了动态拼装SQL的快捷书写方式
```
bookLambdaQueryWrapper.like(name != null,Book::getName,name);  
```


## 5.5 业务层
在开发的时候是可以根据完成的工作不同划分成不同职能的开发团队的。比如制作数据层的人，可以不知道业务是什么样子，拿到的需求文档要求可能是这样的
```tex
接口：传入用户名与密码字段，查询出对应结果，结果是单条数据  
接口：传入ID字段，查询出对应结果，结果是单条数据  
接口：传入离职字段，查询出对应结果，结果是多条数据
```
但是进行业务功能开发的，拿到的需求文档要求差别就很大
```tex
接口：传入用户名与密码字段，对用户名字段做长度校验，4-15位，对密码字段做长度校验，8到24位，对喵喵喵字段做特殊字符校验，不允许存在空格，查询结果为对象。如果为null，返回BusinessException，封装消息码INFO_LOGON_USERNAME_PASSWORD_ERROR
```
ISO标准化文档


业务层接口：
```java
public interface BookService {
    Boolean save(Book book);
    Boolean update(Book book);
    Boolean delete(Integer id);
    Book getById(Integer id);
    List<Book> getAll();
    IPage<Book> getPage(int currentPage,int pageSize);
}
```
业务层实现类：
```java
public class BookServiceImpl implements BookService {  
    @Autowired  
 public BookDao bookDao;  
    @Override  
 public Boolean save(Book book) {  
        return bookDao.insert(book) > 0;  
    }  
  
    @Override  
 public Boolean update(Book book) {  
        return bookDao.updateById(book) > 0;  
    }  
  
    @Override  
 public Boolean delete(Integer id) {  
        return bookDao.deleteById(id) > 0;  
    }  
  
    @Override  
 public Book getById(Integer id) {  
        return bookDao.selectById(id);  
    }  
  
    @Override  
 public List<Book> getAll() {  
        return bookDao.selectList(null);  
    }  
  
    @Override  
 public IPage<Book> getPage(int currentPage, int pageSize) {  
        IPage<Book> page = new Page<>(currentPage,pageSize);  
        bookDao.selectPage(page,null);  
        return page;  
    }  
  
    @Override  
 public IPage<Book> getPage(int currentPage, int pageSize, Book book) {  
        IPage<Book> page = new Page<>(currentPage,pageSize);  
        LambdaQueryWrapper<Book> bookLambdaQueryWrapper = new LambdaQueryWrapper<>();  
        bookLambdaQueryWrapper.eq(!book.getName().isEmpty(),Book::getName,book.getName());  
        bookLambdaQueryWrapper.eq(!book.getType().isEmpty(),Book::getType,book.getType());  
        bookLambdaQueryWrapper.eq(!book.getDescription().isEmpty(),Book::getDescription,book.getDescription());  
        bookDao.selectPage(page,bookLambdaQueryWrapper);  
        return page;  
    }  
}

```

## 5.6 表现层
### 5.6.1 表现层消息一致性处理
不同的操作可能返回 true/false、数据、或者错误消息等等，前端难处理。必须将所有操作的操作结果数据格式统一起来，需要设计表现层返回结果的模型类，用于后端与前端进行数据格式统一，也称为**前后端数据协议**
```java
public class R {  
    private Boolean flag;  
    private Object data;  
    private String msg;
}
```

### 5.6.2 表现层接口
表现层的开发使用基于Restful的表现层接口开发
```java
@RestController  
@RequestMapping("/books")  
public class BookController {  
    @Autowired  
 public BookService bookService;  
  
    @GetMapping  
 public R getAll(){  
        List<Book> bookList = bookService.getAll();  
        return new R(true,bookList);  
    }  
    @PostMapping  
 public R insertBook(@RequestBody Book book) throws IOException{  
        if (book.getName().equals("123") ) throw new IOException();  
        Boolean flag = bookService.save(book);  
        return new R(flag,flag ? "添加成功" : "添加失败");  
    }  
    @PutMapping  
 public R update(@RequestBody Book book) throws IOException {  
        if (book.getName().equals("123") ) throw new IOException();  
        boolean flag = bookService.update(book);  
        return new R(flag, flag ? "修改成功" : "修改失败");  
    }  
    @DeleteMapping("{id}")  
    public R delete(@PathVariable Integer id){  
        return new R(bookService.delete(id));  
    }  
  
    @GetMapping("{id}")  
    public R getById(@PathVariable Integer id){  
        return new R(true, bookService.getById(id));  
    }  
    @GetMapping("{currentPage}/{pageSize}")  
    public R getPage(@PathVariable Integer currentPage, @PathVariable Integer pageSize,Book book){  
        IPage<Book> page = bookService.getPage(currentPage, pageSize,book);  
        //如果当前页码值大于了总页码值，那么重新执行查询操作，使用最大页码值作为当前页码值  
 if(currentPage > page.getPages()){  
            page = bookService.getPage(currentPage, pageSize,book);  
        }  
        return new R(true,page);  
    }  
  
}

```

### 5.6.3 异常处理
```java
@RestControllerAdvice
public class ProjectExceptionAdvice {
    @ExceptionHandler(Exception.class)
    public R doOtherException(Exception ex){
        //记录日志
        //发送消息给运维
        //发送邮件给开发人员,ex对象发送给开发人员
        ex.printStackTrace();
        return new R(false,null,"系统错误，请稍后再试！");
    }
}
```


## 5.6.4 返回页面
## 1. 直接controller
默认/static下
```java
@Controller  
@RequestMapping("/book")  
public class pageController {  
    @RequestMapping  
 public String book(){  
        return "pages/books.html";  
    }  
}
```
## 2. MVC设置
```yaml
spring:  
  mvc:  
    view:  
      prefix: pages/  
      suffix: .html
```

controller:
```java
@Controller  
@RequestMapping("/book")  
public class pageController {  
    @RequestMapping  
 public String book(){  
        return "books";  
    }  
}
```
## 3.thymeleaf
导入依赖
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
```
默认模板目录在/templates
```yaml
spring:  
  thymeleaf:  
    cache: false  # 开发时关闭缓存
	# 默认前后缀
    suffix: .html  
    prefix: classpath:/templates/   # classpath:/static/pages/
	
```
controller
```java
@Controller  
@RequestMapping("/book")  
public class pageController {  
    @RequestMapping  
 public String book(){  
        return "books";  
    }  
}
```

## 5.7 运行
运行启动类后，访问local:8080/pages/books.html
![](https://gitee.com/minan-palace/md_images/raw/master/images2/20220309011347.png)

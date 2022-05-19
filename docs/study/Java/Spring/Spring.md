# Spring

# 一、框架概述

轻量级的JavaEE框架

## 特点

方便解耦，简化开发

* 传统应用程序开发：开发业务逻辑，并且关注这些对象如何协作来实现所需功能，还要低耦合和高聚合。而通过工厂方法模式、建造者模式创建、管理对象及之间的依赖关系，需要管理太多类，增加了负担。
* 所以Spring框架，通过配置创建和管理对象之间的依赖关系。 

AOP编程支持：

* 提供通用日记记录、性能统计、安全控制、异常处理等面向切面的能力。

降低 API 开发难度

* 提供了一套简单的JDBC访问实现

方便和其他框架进行整合

* Hibernate、JPA等
* 提供自己的一套Web框架 SpringMVC

## 设计策略

主要通过 面向Bean(BOP)、依赖注入（DI）、面向切面（AOP）来实现

* 基于POJO的轻量级和最小侵入性编程。
* 通过依赖注入和面向接口实现松耦合。
* 基于切面和惯性进行声明式编程。
* 通过切面和模板减少样板式代码。

**两个核心部分：IOC、AOP**

（1）IOC: 控制反转，把创建对象的过程交给spring进行管理

（2）AOP：面向切面：不修改源代码的情况下，进行功能增强、添加



# 二、入门案例（IOC的实现）

## 1.Maven创建spring项目

(1) add famework 

![image-20211030002027473](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211030002027473.png)

(2) 或者直接加配置类



```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-expression</artifactId>
            <version>5.2.3.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>commons-logging</groupId>
            <artifactId>commons-logging</artifactId>
            <version>1.2</version>
        </dependency>
    </dependencies>
```



## 2.编写POJO类、bean xml配置文件 、测试类

![image-20211029000631408](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211029000631408.png)

```java
//POJO
public class User {
    public void add(){
        System.out.println("addd-----");
    }
}
```

```xml
//bean xml配置文件 
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id ="user" class="User"/>
</beans>
```

```java
//测试类
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;


public class main {
    @Test
    public void test1(){
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("bean1.xml");
        User user = applicationContext.getBean("user", User.class);
        System.out.println(user);
        user.add();

    }
}
```

运行结果：

注意：maven项目是不会自动编译的故没有class文件，会报Class not found: "......" 的错误, 

解决方法：后侧边栏maven compile

![image-20211029000815262](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211029000815262.png)



# 三、IOC容器

[深度理解依赖注入- EagleFish(邢瑜琨)](https://www.cnblogs.com/xingyukun/archive/2007/10/20/931331.html)

[架构师之路(39)---IoC框架 编程宝库](https://blog.csdn.net/wanghao72214/article/details/3969594)

## 概念和原理

IOC，控制反转。用来降低代码之间的耦合度。常见方式为依赖注入。

控制反转：将new 对象的控制权从应用程序的代码本身转移到外部的xml文件

依赖注入：依赖对象通过注入进行创建，由ioc容器负责创建和注入，整个创建和注入过程就叫做依赖注入。

例子：如果实例化一个A类，而A类依赖于B、C、D、E类，B、C、D、E类又依赖于E、F、G,就要new实例化一大堆类了。所以，把有依赖关系的类放到IOC容器中，IOC根据依赖的关系，new对象实例。

## 深度理解

面向对象设计的软件，底层是由N个对象构成的，各个对象之间相互合作，最后实现系统业务逻辑。但是随着工业级应用的规模越来越大，对象之间的依赖关系也越来越复杂，经常出现多重依赖关系。对象之间耦合度过高

![image-20211031110430921](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211031110430921.png)

所以，提出了IOC理论，实现对象之间的解耦。（框架产品：spring等）

IOC理论观点：借助于“第三方”（IOC容器）实现具有依赖关系的对象之间的解耦，如下图

![image-20211031113432409](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211031113432409.png)

​       现在IOC容器成为了系统的关键核心。对象之间没有了耦合关系，实现A时不需要去考虑BCD，对象之间依赖关系降到了最低。

​        同时原本A必须自己主动创建或使用对象B，控制权在自己手上。现在是由IOC容器来主动创建对象B,并注入到A需要的地方。

​         从主动创建，到被动被注入，控制权反转了，称为**控制反转**。而IOc容器运行期间，动态地将某种依赖关系注入到对象中，称为**依赖注入**。

​       **对象A在创建另一个对象实例时，以接口来接收实例，它并不知道具体地实现类是什么。这个创建哪个实现类的控制权在于IOC容器，IOC容器主动帮它创建想要的对象。对象A只能被动的接收。**



## 技术剖析

反射----->工厂模式-------->IOC（工厂模式的升华）

​        可以把IOC容器看作是一个工厂，这个工厂里要生产的对象都在配置文件中给出定义，然后利用编程语言的的反射编程，根据配置文件中给出的类名生成相应的对象。从实现来看，IOC是把以前在工厂方法里写死的对象生成代码，改变为由配置文件来定义，也就是把工厂和对象生成这两者独立分隔开来，目的就是提高灵活性和可维护性。

## IOC优缺点

**优点**

*  当有依赖注入时，对象之间才具有相关性。所以，无论两者中的任何一方出现什么的问题，都不会影响另一方的运行。可维护性比较好，非常便于进行单元测试，便于调试程序和诊断故障。代码中的每一个Class都可以单独测试，彼此之间互不影响，只要保证自身的功能无误即可，这就是组件之间低耦合或者无耦合带来的好处。
*  只需要定义好接口标准。每个开发团队的成员都只需要关心实现自身的业务逻辑，完全不用去关心其它的人工作进展，因为你的任务跟别人没有任何关系，你的任务可以单独测试，你的任务也不用依赖于别人的组件。
*  符合接口标准的实现，都可以插接到支持此标准的模块中。可复用性好。
* 模块具有热插拔特性。IOC生成对象的方式转为外置方式，也就是把对象生成放在配置文件里进行定义，这样，当我们更换一个实现子类将会变得很简单，只要修改配置文件就可以了，完全具有热插拨的特性。

**缺点**

* 引入了第三方IOC容器，生成对象的步骤变得有些复杂。且IOC容器生成对象是通过反射方式，在运行效率上有一定的损耗。特别强调运行效率的项目或者产品，不太适合引入IOC框架产品。
* 会增加团队成员学习和认识的培训成本，并且在以后的运行维护中，还得让新加入者具备同样的知识体系。团队成员的知识能力欠缺，对于IOC框架产品缺乏深入的理解，不要贸然引入
* 第三、具体到IOC框架产品(比如：Spring)来讲，需要进行大量的配制工作，比较繁琐，对于一些小的项目而言，客观上也可能加大一些工作成本。一些工作量不大的项目或者产品，不太适合使用IOC框架产品。
* 第四、IOC框架产品本身的成熟度需要进行评估。







## 实现方式

```
/*服务的接口*/
public interface MovieFinder {
	ArrayList findAll();
}

/*服务的消费者*/
class MovieLister{
	public Movie[] moviesDirectedBy(String arg) {
		List allMovies = finder.findAll();
		for (Iterator it = allMovies.iterator(); it.hasNext();) {
			Movie movie = (Movie) it.next();
			if (!movie.getDirector().equals(arg)) it.remove();
		}
		return (Movie[]) allMovies.toArray(new Movie[allMovies.size()]);
	}

	/*消费者内部包含一个将指向具体服务类型的实体对象*/
	private MovieFinder finder;
	/*消费者需要在某一个时刻去实例化具体的服务。这是我们要解耦的关键所在，
     *因为这样的处理方式造成了服务消费者和服务提供者的强耦合关系（这种耦合是在编译期就确定下来的）。
    **/
	public MovieLister() {
		finder = new ColonDelimitedMovieFinder("movies1.txt");
	}
}
```

![image-20211031224935907](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211031224935907.png)



此时，MovieLister强依赖于MovieFinderImpl,强耦合关系。而MovieLister所需要的电影查找器应该是多种多样的。

必须有一种机制能同时满足下面两个要求：
 （1）解除MovieList对具体MoveFinder类型的强依赖（编译期依赖）。
 （2）在运行的时候为MovieList提供正确的MovieFinder类型的实例。

换句话说，就是在运行的时候才产生MovieList和MovieFinder之间的依赖关系（把这种依赖关系在一个合适的时候“注入”运行时）

![image-20211031232651514](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211031232651514.png)

要提供一个容器，由容器来完成（1）具体ServiceProvider的创建（2）ServiceUser和ServiceProvider的运行时绑定



### 1. 构造器注入

容器的实现

```java
public static class Container{
    /**
     * <Type,null>注册了类，<Type,impl>注册类和实例。
    */
	private static Dictionary<Type, object> _stores = null; 
	private static Dictionary<Type, object> Stores { 
		get { 
			if (_stores == null) 
				_stores = new Dictionary<Type, object>(); 
			return _stores; 
		} 
	} `
    /**
     * @param 需要生成实例的类
     * @return 类构造器需要的<参数名,参数实例>
    */
	private static Dictionary<string, object> CreateConstructorParameter(Type targetType) 
	{ 
		Dictionary<string, object> paramArray = new Dictionary<string, object>(); 
		ConstructorInfo[] cis = targetType.GetConstructors(); 
		if (cis.Length > 1) 
			throw new Exception("target object has more then one constructor,container can't peek one for you."); 
	    /*
	    对于每个 构造器需要的参数，如果容器中有其注册的类，则通过构造器注入生成该实例。
	    */
		foreach (ParameterInfo pi in cis[0].GetParameters()) { 
        	if (Stores.ContainsKey(pi.ParameterType))
            	paramArray.Add(pi.Name, GetInstance(pi.ParameterType)); 
        } 
        return paramArray; 
    } 
    /**
     * @param 需要生成实例的类
     * @return 生成的实例
     * 首先判断 类 是否注册过了。然后有构造器，构造器生成。不然看是否注册了实例，返回实例。不然就false.
    */
	public static object GetInstance(Type t) { 
		if (Stores.ContainsKey(t)) { 
            /*
            构造器生成实例 所以叫做构造器注入
            */
        	ConstructorInfo[] cis = t.GetConstructors(); 
            if (cis.Length != 0){ 
				Dictionary<string, object> paramArray = CreateConstructorParameter(t); 
				List<object> cArray = new List<object>(); 
                /*获得构造器参数实例的列表
                */
				foreach (ParameterInfo pi in cis[0].GetParameters()){ 
					if (paramArray.ContainsKey(pi.Name)) 
						cArray.Add(paramArray[pi.Name]); 
					else 
						cArray.Add(null); 
				} 
			//在这里完成了对构造函数的调用，而构造函数的传入参数是通过在容器中查找匹配类型的实例得到的，
			//所以被称为构造器注入。
                return cis[0].Invoke(cArray.ToArray()); 
            } 
            else if (Stores[t] != null) 
                return Stores[t]; 
            else 
                return Activator.CreateInstance(t, false); 
        } 
        return Activator.CreateInstance(t, false); 
   } 

    //向容器中加入ServiceProvider的实例
    public static void RegisterImplement(Type t, object impl) 
    { 
        if (Stores.ContainsKey(t)) 
            Stores[t] = impl; 
        else 
            Stores.Add(t, impl); 
    } 

    //向容器中加入ServiceUser的类型，类型的构造器将在容器中被调用
    public static void RegisterImplement(Type t) 
    { 
        if (!Stores.ContainsKey(t)) 
            Stores.Add(t, null); 
    } 
} 
```



```
private MutablePicoContainer configureContainer() {
    MutablePicoContainer pico = new DefaultPicoContainer();
    
    //下面就是把ServiceProvider和ServiceUser都放入容器的过程，以后就由容器来提供ServiceUser的已完成依赖注入实例，
    //其中用到的实例参数和类型参数一般是从配置档中读取的，这里是个简单的写法。
    //所有的依赖注入方法都会有类似的容器初始化过程，本文在后面的小节中就不再重复这一段代码了。
    Parameter[] finderParams =  {new ConstantParameter("movies1.txt")};
    pico.registerComponentImplementation(MovieFinder.class, ColonMovieFinder.class, finderParams);
    pico.registerComponentImplementation(MovieLister.class);
    //至此，容器里面装入了两个类型，其中没给出构造参数的那一个（MovieLister）将依靠其在构造器中定义的传入参数类型，在容器中
    //进行查找，找到一个类型匹配项即可进行构造初始化。
    return pico;
}
```

```
public void testWithPico() 
{
    MutablePicoContainer pico = configureContainer();
    MovieLister lister = (MovieLister) pico.getComponentInstance(MovieLister.class);
    Movie[] movies = lister.moviesDirectedBy("Sergio Leone");
    assertEquals("Once Upon a Time in the West", movies[0].getTitle());
}
```



### 2.setter注入

Spring对通过XML进行配置有比较好的支持，也使得Spring中更常使用设值注入的方式

```
public static class Container
{
    private static Dictionary<Type, object> _stores = null;

    private static Dictionary<Type, object> Stores
    {
        get
        {
            if (_stores == null)
                _stores = new Dictionary<Type, object>();
            return _stores;
        }
    }

    public static object GetInstance(Type t)
    {
        if (Stores.ContainsKey(t))
        {
            if (Stores[t] == null)
            {   
                /*target 无参创建的实例
                */
                object target = Activator.CreateInstance(t, false);
                foreach (PropertyDescriptor pd in TypeDescriptor.GetProperties(target))
                {
                    if (Stores.ContainsKey(pd.PropertyType))
                        //在此处为待创建实例设置属性，完成依赖注入过程。属性值是在容器中通过类型匹配的方式查找出来的。
                        pd.SetValue(target, GetInstance(pd.PropertyType));
                }
                return target;
            }
            else
                return Stores[t];
        }
        return Activator.CreateInstance(t, false);
    }

    public static void RegisterImplement(Type t, object impl)
    {
        if (Stores.ContainsKey(t))
            Stores[t] = impl;
        else
            Stores.Add(t, impl);
    }

    public static void RegisterImplement(Type t)
    {
        if (!Stores.ContainsKey(t))
            Stores.Add(t, null);
    }
} 
```



## spring IOC

### IOC接口

（1）BeanFactory：IOC 容器基本实现，是 Spring 内部的使用接口，不提供开发人员进行使用

 **加载配置文件时候不会创建对象，在获取对象（使用）才去创建对象** 

（2）ApplicationContext：BeanFactory 接口的子接口，提供更多更强大的功能，一般由开发人 员进行使用 

**加载配置文件时候就会把在配置文件对象进行创建**

```
//加载xml配置文件
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("bean1.xml");
//获得配置创建的文件
User user = applicationContext.getBean("user1", User.class);
```



### bean管理

bean是由IOC管理的对象

两个操作：spring创建对象，spring注入属性

两种方式：基于xml配置文件方式实现，基于注解方式实现

#### 基于xml配置文件

##### setter注入

要求属性有相对应的setter方法，不然会报错。

```xml
    <!--创建对象，并且setter注入属性-->
    <bean id ="user" class="com.mng.User">
        <property name="name" value="YU"/>
    </bean>
```

##### 构造器注入

要求有相对应的构造器方法 也可以index指定第几个参数，不用name.

```xml
   <!--构造器创建对象-->
    <bean id ="user1" class="com.mng.User">
        <constructor-arg name="name" value="YUU"/>
    </bean>
```

##### p名称空间注入（仅作了解）

（仅作了解，底层还是setter注入）

![image-20211102234447877](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211102234447877.png)

![image-20211103213121908](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103213121908.png)





##### null值

![image-20211103213321086](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103213321086.png)

#####  包含特殊符号的属性值

![image-20211103225248192](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103225248192.png)

![image-20211103225458345](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103225458345.png)



##### 对象属性注入

要求有对应该对象的setter方法。

1.外部bean

![image-20211103232717518](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103232717518.png)

2.内部bean

![image-20211103232811259](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103232811259.png)

3. 直接property注入

   要求有相应对象的getter方法

![image-20211103233536228](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211103233536228.png)



##### xml注入集合属性

![image-20211104001718286](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104001718286.png)

![image-20211104001746780](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104001746780.png)





##### util提取list属性

![image-20211104003249890](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104003249890.png)



![image-20211104003259127](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104003259127.png)



#### FactoryBean

spring有两种bean，一种是普通bean（在配置文件中定义的类型就是返回类型），一种是工厂bean（返回类型由getObject方法定义，配置文件中定义的类型可以和返回类型不同）。

1. 工厂类要继承FactoryBean接口

![image-20211104095745237](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104095745237.png)

2. 配置文件，和普通bean一致

![image-20211104095807104](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104095807104.png)

3.getBean时，解析类和接受类都是工厂生产的类或者接口

![image-20211104095828391](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104095828391.png)

4.输出结果

![image-20211104100434242](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104100434242.png)



#### bean作用域（单/多实例）

spring默认情况下是单实例

![image-20211104100852151](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104100852151.png)



设置单/多实例

spring的配置文件bean中，scope属性。默认值singleton表示单实例，加载配置文件的时候创建单实例对象。prototype表示多实例，在getBean时创建多实例对象。

![image-20211104101001394](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104101001394.png)



#### bean生命周期

（1）通过构造器创建 bean 实例（无参数构造） 

（2）为 bean 的属性设置值和对其他 bean 引用（调用 set 方法） 

（3）把 bean 实例传递 bean 后置处理器的方法 postProcessBeforeInitialization  

（4）调用 bean 的初始化的方法（需要进行配置初始化的方法） 

（5）把 bean 实例传递 bean 后置处理器的方法 postProcessAfterInitialization 

（6）bean 可以使用了（对象获取到了） 

（7）当容器关闭时候，调用 bean 的销毁的方法（需要进行配置销毁的方法）



![image-20211104125826622](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104125826622.png)



#### xml自动装配

指定装配规则（属性名称或者属性类型），spring自动将匹配的属性值进行注入。

自动装配：

* byName:根据属性名称注入，要求bean id 和 属性名相同，不然注入不成功，null值
* byType: 根据属性类型注入，要求相应的bean class 只有一个，不然出现but found 2错误

![image-20211104164118855](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104164118855.png)

#### 外部属性文件

![image-20211104164521603](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104164521603.png)

![image-20211104164533439](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211104164533439.png)



#### 基于注解的方式

注解是代码特殊标记，格式：@注解名称(属性名称=属性值, 属性名称=属性值..)    用于简化xml配置



##### 注解类型:

@Component : 普通组件

@Service : 业务逻辑层，service层

@Controller : web层

@Repository ：dao层，或者持久层

上面四个注解功能是一样的，都可以用来创建bean实例

默认bean id为类名，首字母小写

##### 实现对象的创建

开启组件扫描：

![image-20211105001156674](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105001156674.png)



创建类

![image-20211105001829544](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105001829544.png)







#####  组件扫描的细节配置

![image-20211105001921933](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105001921933.png)



##### 注解方式实现属性注入

* @Autowired 根据属性类型进行自动装配 不需要set方法

* @Qualifier 根据名称进行注入，需要和@Autowired一起使用

  ![image-20211105002217069](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105002217069.png)

* @Resource 可以根据类型或者名称注入，是java扩展包中的注解

  ![image-20211105002330640](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105002330640.png)

* @Value 注入普通属性类型

![image-20211105002414452](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105002414452.png)



##### 完全注解开发

上面的注解开发 还是需要xml配置 扫描组件

完全注解开发不用xml文件

![image-20211105002927581](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105002927581.png)

![image-20211105002911864](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105002911864.png)







# 四、AOP

AOP（Aspect Oriented Programming）称为面向切面编程

运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程

* 在不改变原有的逻辑的基础上，增加一些额外的功能。在程序开发中主要用来解决一些系统层面上的问题，比如日志，事务，权限等待。



面向对象的特点是继承、多态、封装。封装要求功能分散到不同的对象中。软件设计中往往称为职责分配。这样代码**复杂程度降低**了，使类可以**重用**。

![image-20211105221116565](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105221116565.png)

我们可以将这段代码写在一个独立的类独立的方法里，然后再在这两个类中调用。但是，这样一来，这两个类跟我们上面提到的独立的类就有耦合了，它的改变会影响这两个类。

![image-20211105221455765](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105221455765.png)

所以，**在运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程。** 我们管切入到指定类指定方法的代码片段称为切面，而切入到哪些类、哪些方法则叫切入点。有了AOP，我们就可以把几个类共有的代码，抽取到一个切片中，等到需要时再切入对象中去，从而改变其原有的行为。这样，就只要考虑类主要流程，不需要考虑其他流程。

![image-20211105221821624](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105221821624.png)



## 相关概念

* Joint point (连接点)

  类里面哪些方法可以被增强称为连接点。spring只支持方法类型的连接点，所以连接点是被拦截的方法。

* Pointcut (切点)

  实际被增强的部分

  就是带有通知的连接点，在程序中主要体现为书写切入点表达式

* Advice(增强)

  实际增强的逻辑部分，在切入点上执行的增强处理

  类型：前置通知、后置通知、环绕通知、异常通知、最终通知

* Aspect（切面）

  把 增强 应用到 切入点 过程

  通常是一个类，可以定义切入点和通知

* Target(目标对象)

* Weaving(织入)

## 底层原理

AOP使用动态代理

动态代理有两种情况：

* 1.有接口，使用JDK动态管理
* 2.没有接口，使用CGLib动态代理

![image-20211105234345957](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105234345957.png)



![image-20211105235402669](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211105235402669.png)



## 准备工作

1.spring一般是基于AspectJ实现AOP操作

   AspectJ不是Sprin组成部分，独立AOP框架，一般把 AspectJ 和 Spirng   框架一起使 用，进行 AOP 操作

  2.基于AspectJ实现AOP操作

（1）基于xml配置文件实现

（2）基于注解方式实现

4、切入点表达式 

（1）切入点表达式作用：知道对哪个类里面的哪个方法进行增强 

（2）语法结构:execution([权限修饰符] [返回类型] [类全路径] [方法名称]\([参数列表]\) )     

返回类型可以省略。

举例 1：对 com.atguigu.dao.BookDao 类里面的 add 进行增强 execution(* com.atguigu.dao.BookDao.add(..)) 

举例 2：对 com.atguigu.dao.BookDao 类里面的所有的方法进行增强 execution(* com.atguigu.dao.BookDao.* (..)) 

举例 3：对 com.atguigu.dao 包里面所有类，类里面所有方法进行增强 execution(* com.atguigu.dao.*.* (..))



## AspectJ注解

* 1.创建类和增强类（@Component注解为bean）

* 2.spring配置文件开启注解扫描和生成代理对象

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns:context="http://www.springframework.org/schema/context"
         xmlns:aop="http://www.springframework.org/schema/aop"
         xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://www.springframework.org/schema/context  http://www.springframework.org/schema/context/spring-context.xsd
  http://www.springframework.org/schema/aop  http://www.springframework.org/schema/aop/spring-aop.xsd">
  
      <context:component-scan base-package="com.mng.AOPanno"/>
      <aop:aspectj-autoproxy/>
  </beans>
  ```

  

* 3.增强类 @Aspect注解、通知方法用注解方式配置通知

  

  

其他操作：

1.提取相同的切入点

```
@Pointcut注解方法
execution语句直接使用方法
```

2.有多个增强类多同一个方法进行增强，设置增强类优先级 

在增强类上面添加注解 @Order(数字类型值)，数字类型值越小优先级越高

```java
@Component
@Aspect
@Order(1)
public class PersonProxy
```

3.完全使用注解开发

```java
@Configuration
@ComponentScan(basePackages = {"com.atguigu"})
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class ConfigAop {
}
```



## 配置文件

![image-20211110235042569](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211110235042569.png)







# 五、 JdbcTemplate

JdbcTemplate: Spring 框架对 JDBC 进行封装，使用 JdbcTemplate 方便实现对数据库操作

## 准备工作

### 添加依赖

```xml
        <!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid</artifactId>
            <version>1.2.6</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
```

### spring 配置文件

```
//xml实体替换
&-------&amp;      <---------&t;      
>---------&gt;     '-----&apos;         "-----&quot;
```

配置数据连接池、配置JdbcTemplate对象和注入dataSource

```xml
<!--数据库连接池-->
    <bean id = "dataSource" class="com.alibaba.druid.pool.DruidDataSource" destroy-method="close">
        <property name="url" value="jdbc:mysql://localhost:3306/mybatisdb?useSSL=false&amp;serverTimezone=UTC" />
        <property name="username" value="root" />
        <property name="password" value="1234" />
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver" />
    </bean>

<!--    JdbcTemplate对象-->
    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
<!--        注入dataSource-->
        <property name="dataSource" ref="dataSource"/>
    </bean>
```



### 创建Dao类

注入JdbcTemplate (这里不创建service类，直接Dao类操纵数据库) 

```java
@Repository
public class EmployeeDao {
  @Autowired
  JdbcTemplate jdbcTemplate;
```

开启组件扫描

```xml
    <context:component-scan base-package="com.mng.JdbcTemplateTest"/>
```



## JdbcTemplate操作数据库

增删改 都是update、batchUpdate函数

update参数: sql语句   Object[]

batchUpdate参数： sql语句  List\<Object\[\]\> 



查 返回值/object   queryForObject  返回列表 query

参数都是 sql语句、类（数据封装接口实现）、sql参数（Object[]）

### 增加

```java
//Dao类
  public void add(Employee employee){
    String sql = "insert into tbl_employee values (?,?,?,?,?)";
    Object[] args = {employee.getId(), employee.getLastName(),
        employee.getGender(), employee.getEmail(), employee.getDeptId()};
    int update = jdbcTemplate.update(sql,args);
    System.out.println(update);
  }
```

```java
//测试类
  @Test
  public void test1(){
    ApplicationContext context = new ClassPathXmlApplicationContext("JdbcTemplateTest.xml");
    EmployeeDao employeeDao = context.getBean("employeeDao", EmployeeDao.class);
    Employee employee = new Employee(null,"mngg","1","13@qq",1L);
    employeeDao.add(employee);
  }
```

运行结果：

![image-20211112203609894](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211112203609894.png)

### 修改

```java
  public void updateEmployee(Employee employee){
    String sql = "update tbl_employee set email= ? where last_name = ? ";
    Object[] args = {employee.getEmail(),employee.getLastName()};
    int update  = jdbcTemplate.update(sql,args);
    System.out.println(update);
  }
```

### 删除

```java
  public void deleteEmployee(Employee employee){
    String sql = "delete from tbl_employee where last_name = ? ";
    Object[] args = {employee.getLastName()};
    int update  = jdbcTemplate.update(sql,args);
    System.out.println(update);
  }
```

### 查询

1.返回值

queryForObject参数：sql、返回的类型、sql参数

![image-20211112232611047](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211112232611047.png)

```java
  public Integer countEmployee(Department department){
    String sql = "select count(*) from tbl_employee where dept_id = ?";
    Object[] args ={department.getId()};
    Integer count = jdbcTemplate.queryForObject(sql,Integer.class,args);
    return count;
  }
```

2.返回对象

queryForObject参数：sql语句，数据封装接口实现，sql参数

![image-20211112234713272](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211112234713272.png)

```java
  public Employee findEmployee(Long id){
    String sql = "select * from tbl_employee where id = ?";
    Employee employee = jdbcTemplate.queryForObject(sql,
        new BeanPropertyRowMapper<Employee>(Employee.class),id);
    return  employee;
  }
```

查询结果：

![image-20211112234957083](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211112234957083.png)





### 查询返回列表

```java
  public List<Employee> findAllEmployee(Department department){
    String sql = "select * from tbl_employee where dept_id = ?";
    Object[] args  = {department.getId()};
    List<Employee> employeeList = jdbcTemplate.query(sql,
        new BeanPropertyRowMapper<Employee>(Employee.class),args);
    return employeeList;
  }
```





###  批量操作

```java
  //batch update
  public void batchAddEmployee(List<Employee> employeeList){
    String sql = "insert into tbl_employee values (?,?,?,?,?)";
    List<Object[]> argslist = new ArrayList<>();
    for(Employee employee:employeeList){
      argslist.add(new Object[]{employee.getId(), employee.getLastName(),
          employee.getGender(), employee.getEmail(), employee.getDeptId()});
    }
    int[] ints = jdbcTemplate.batchUpdate(sql, argslist);
    System.out.println(ints);
  }
```

删改相同，不做示例了



## 完全注解式

```java
@Configuration //配置类
@ComponentScan(basePackages = "com.atguigu") //组件扫描
@EnableTransactionManagement //开启事务
public class TxConfig {
 //创建数据库连接池
 @Bean
 public DruidDataSource getDruidDataSource() {
 DruidDataSource dataSource = new DruidDataSource();
 dataSource.setDriverClassName("com.mysql.jdbc.Driver");
 dataSource.setUrl("jdbc:mysql:///user_db");
 dataSource.setUsername("root");
 dataSource.setPassword("root");
 return dataSource;
 }
 //创建 JdbcTemplate 对象
 @Bean
 public JdbcTemplate getJdbcTemplate(DataSource dataSource) {
 //到 ioc 容器中根据类型找到 dataSource
 JdbcTemplate jdbcTemplate = new JdbcTemplate();
 //注入 dataSource
 jdbcTemplate.setDataSource(dataSource);
 return jdbcTemplate;
 }
}
```



# 六、事务操作

添加到Service层（业务逻辑层），一般使用声明式事务管理，通过注解方式实现。底层原理是AOP

## 注解声明式事务管理

1、在 spring 配置文件配置事务管理器   

```xml
<!--创建事务管理器-->
<bean id="transactionManager" 
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
 <!--注入数据源-->
 <property name="dataSource" ref="dataSource"></property>
</bean>

```

 2、在 spring 配置文件，开启事务注解 

（1）在 spring 配置文件引入名称空间 tx 

```xml
xmlns:tx="http://www.springframework.org/schema/tx"
```

（2）开启事务注解  

```xml
<!--开启事务注解-->
<tx:annotation-driven transactionmanager="transactionManager"></tx:annotation-driven>
```

3、在 service 类上面（或者 service 类里面方法上面）添加事务注解 

（1）@Transactional，这个注解添加到类上面，也可以添加方法上面 

（2）如果把这个注解添加类上面，这个类里面所有的方法都添加事务 

（3）如果把这个注解添加方法上面，为这个方法添加事务 

```java
@Service
@Transactional
public class UserService {

```



## 注解参数

### 1.propagation：事务传播行为

多事务之间的传播机制 （方法调用有事务的方法）

![image-20211113011314344](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211113011314344.png)



### 2.ioslation：事务隔离级别

![image-20211113011439810](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211113011439810.png)



### 3.其他

![image-20211113011512766](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211113011512766.png)

## xml声明式事务管理

1、在 spring 配置文件中进行配置 第一步 配置事务管理器 第二步 配置通知 第三步 配置切入点和切面                 

```xml
<!--1 创建事务管理器-->
<bean id="transactionManager" 
class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
   <!--注入数据源-->
   <property name="dataSource" ref="dataSource"></property>
</bean>
<!--2 配置通知-->
<tx:advice id="txadvice">
   <!--配置事务参数-->
   <tx:attributes>
       <!--指定哪种规则的方法上面添加事务-->
       <tx:method name="accountMoney" propagation="REQUIRED"/>
       <!--<tx:method name="account*"/>-->
   </tx:attributes>
</tx:advice>
<!--3 配置切入点和切面-->
<aop:config>
   <!--配置切入点-->
   <aop:pointcut id="pt" expression="execution(* 
com.atguigu.spring5.service.UserService.*(..))"/>
   <!--配置切面-->
   <aop:advisor advice-ref="txadvice" pointcut-ref="pt"/>
</aop:config>
```



## 完全注解式

```java
@Configuration //配置类
@ComponentScan(basePackages = "com.atguigu") //组件扫描
@EnableTransactionManagement //开启事务
public class TxConfig {
 //创建数据库连接池
 @Bean
 public DruidDataSource getDruidDataSource() {
 DruidDataSource dataSource = new DruidDataSource();
 dataSource.setDriverClassName("com.mysql.jdbc.Driver");
 dataSource.setUrl("jdbc:mysql:///user_db");
 dataSource.setUsername("root");
 dataSource.setPassword("root");
 return dataSource;
 }
 //创建 JdbcTemplate 对象
 @Bean
 public JdbcTemplate getJdbcTemplate(DataSource dataSource) {
 //到 ioc 容器中根据类型找到 dataSource
 JdbcTemplate jdbcTemplate = new JdbcTemplate();
 //注入 dataSource
 jdbcTemplate.setDataSource(dataSource);
 return jdbcTemplate;
 }
 //创建事务管理器
 @Bean
 public DataSourceTransactionManager 
getDataSourceTransactionManager(DataSource dataSource) {
 DataSourceTransactionManager transactionManager = new 
DataSourceTransactionManager();
 transactionManager.setDataSource(dataSource);
 return transactionManager;
 }
}

```



# 七、spring5特性

## 1. 代码基于JDK8,运行时兼容jdk9

## 2.自带通用的日志封装

## 3.@Nullable

可以使用在方法上面，属性上面，参数上面，表示方法返回可以为空，属性值可以 为空，参数值可以为空

## 4.Spring5核心容器支持函数式风格GenericApplicationContext

```java
//函数式风格创建对象，交给 spring 进行管理
@Test
public void testGenericApplicationContext() {
 //1 创建 GenericApplicationContext 对象
 GenericApplicationContext context = new GenericApplicationContext();
 //2 调用 context 的方法对象注册
 context.refresh();
 context.registerBean("user1",User.class,() -> new User());
 //3 获取在 spring 注册的对象
 // User user = (User)context.getBean("com.atguigu.spring5.test.User");
 User user = (User)context.getBean("user1");
 System.out.println(user);
}

```



## 5.支持整合JUnit5

不用写ApplicationContext context =........

### 1.JUnit4

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:bean1.xml")
public class JUnitTest {
  @Autowired
  User user1;
  @Test
  public  void getuser1(){
    System.out.println(user1);
  }
}
```



出现错误：

```
Could not load TestContextBootstrapper [null]. Specify @BootstrapWith's 'value' attribute or make the default bootstrapper class available.
```

原因：spring-test版本为5.0以上，而JUnit版本为4的原因

降为4.3.30.RELEASE 后成功



### 2.JUnit5

使注解更简单

```
@ExtendWith(SpringExtension.class)
@ContextConfiguration("classpath:bean1.xml")
public class Junit5Test {
  @Autowired
  User user1;
  @Test
  public void test1(){
    System.out.println(user1);
  }
}

```

**会出现下面问题**

```
Test ignored.

java.lang.NoSuchMethodError: org.springframework.util.ReflectionUtils$MethodFilter.and(Lorg/springframework/util/ReflectionUtils$MethodFilter;)Lorg/springframework/util/ReflectionUtils$MethodFilter;
```

**原因：spring-test和spring版本不同**

**解决方法:改版本**



复合注解

代替上面两个注解

```
@SpringJUnitConfig(locations = "classpath:bean1.xml")
```






























































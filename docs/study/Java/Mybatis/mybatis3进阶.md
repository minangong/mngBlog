# mybatis3进阶

# 一、mybatis映射文件

1. mybatis返回sql语句影响的行数，可以在接口处定义返回的类型int,long,boolean(影响行数>0,true).

2. SqlSession需要手动或自动提交sql语句，不然在close时提交

   ```
   //autoCommit 设为 true
   SqlSession sqlSession = sqlSessionFactory.openSession(true);
   
   ```

## select

```
id="selectPerson"
parameterType="int" //可选
resultType=""//返回值类型，全类名或者别名，如果返回的是集合，定义集合中元素的类型。不能和resultMap同时使用。
resultMap=""  //外部resultMap的命名引用。不能和resultType同时使用。
databaseId
```

 **返回List：**

resultType还是写list集合里的类

```xml
//接口类
public List<String> getNameContainsN(String n);

//映射文件   注意字符串拼接的写法！！
    <select id="getNameContainsN" resultType="String">
        select last_name from tbl_employee where last_name like "%"#{n}"%"
    </select>
```



**返回map**：

```xml
//接口类    
    public Map<String,Object> getMapById(Long Id);
    public Map<String,Object> getMapByName(String name);

//映射文件
<!--    public Map<String,Object> getMapById(Long Id);-->
    <select id="getMapById" resultType="map">
        select * from tbl_employee where id = #{id}
    </select>
<!--    public Map<String,Object> getMapByName(String name);-->
    <select id="getMapByName" resultType="map">
        select  * from tbl_employee  where last_name=#{name}
    </select>
```

结果：

![image-20211020233758811](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211020233758811.png)

mybatis会把结果封装成 **列名=列值** 的形式

![image-20211020233441852](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211020233441852.png)

**但结果大于1个时，报错**

```xml
//接口类
@MapKey("id")    //如果没有mapkey，会报com.mng.Employee cannot be cast to java.util.Map错误
public Map<Long, Employee> getByIdReturnMap(Long Id);

    
//映射文件
<!--    Map<Long,Employee> getByIdReturnMap(Long Id);-->
    <select id="getByIdReturnMap" resultType="com.mng.Employee">
        select * from tbl_employee where id = #{id}
    </select>
    
    
```









## insert、update、delete

```java
id="insertAuthor"
parameterType="domain.blog.Author" //可选
 
databaseId
//只有insert、update有这两个属性
keyProperty="id"
useGeneratedKeys="true"


```

1. parameterType可选  MyBatis 可以通过类型处理器（TypeHandler）推断出具体传入语句的参数，默认值为未设置（unset）。
2.  useGeneratedKeys：使用自增主键获取获取主键策略
3.  keyProperty：指定对应的主键属性，即mybatis获取到主键值后（getGeneratedKeys的返回值或者insert的selectKey子元素），将这个值封装给Javabean哪个属性。

**主键生成方式：**

* 数据库支持自动生成主键：设置useGeneratedKeys、keyProperty

* 不支持自动生成主键（如Oracle）：可以使用selectKey从序列中得到下一个主键的值，设置id，然后插入语句调用

  ![image-20211018110442291](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211018110442291.png)

  After版本：（仅作了解，在插入多条记录时，可能出现问题）

  ![image-20211018111038400](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211018111038400.png)



## sql

这个元素可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用。 参数可以静态地（在加载的时候）确定下来，并且可以在不同的 include 元素中定义不同的参数值。

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>
```

这个 SQL 片段可以在其它语句中使用，例如：

```xml
<select id="selectUsers" resultType="map">
  select
    <include refid="userColumns"><property name="alias" value="t1"/></include>,
    <include refid="userColumns"><property name="alias" value="t2"/></include>
  from some_table t1
    cross join some_table t2
</select>
```

也可以在 include 元素的 refid 属性或内部语句中使用属性值，例如：

```xml
<sql id="sometable">
  ${prefix}Table
</sql>

<sql id="someinclude">
  from
    <include refid="${include_target}"/>
</sql>

<select id="select" resultType="map">
  select
    field1, field2, field3
  <include refid="someinclude">
    <property name="prefix" value="Some"/>
    <property name="include_target" value="sometable"/>
  </include>
</select>
```



## 参数处理

**单个参数：**Mybatis不做任何处理

​                   #{参数名}，取出参数值，//命名不同都可以

**多个参数：**mybatis会做特殊处理

​                   多个参数会被封装成1个map

​                           key: param1........paramN,或者参数的索引

​                    #{param1}就是从map里获得指定key的值

![image-20211018205830725](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211018205830725.png)

![image-20211018162127520](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211018162127520.png)



**命名参数：**

为参数使用@Param起一个名字，MyBatis就会将这些参数封装进map中，key就是我们自己指定的名字



**POJO**：

当这些参数属于我们业务POJO时，我们直接传递POJO

#{属性名}，取出传入的POJO的属性值





**map:** 

反正最后封装成map

#{key},取出map中对应的值

而且mapper.xml不用写参数类型。

如果多个参数不是业务模型中的数据，但是经常要使用，推荐编写一个**TO（transfer object），数据传输对象**



**PS：**

![image-20211018165342351](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211018165342351.png)



### 源码分析

```
public Employee getEmp(@Param("id")Integer id,String lastName);
	取值：id==>#{id/param1}   lastName==>#{param2}

public Employee getEmp(Integer id,@Param("e")Employee emp);
	取值：id==>#{param1}    lastName===>#{param2.lastName/e.lastName}

##特别注意：如果是Collection（List、Set）类型或者是数组，
		 也会特殊处理。也是把传入的list或者数组封装在map中。
			key：Collection（collection）,如果是List还可以使用这个key(list)
				数组(array)
public Employee getEmpById(List<Integer> ids);
	取值：取出第一个id的值：   #{list[0]}
	
========================结合源码，mybatis怎么处理参数==========================
总结：参数多时会封装map，为了不混乱，我们可以使用@Param来指定封装时使用的key；
#{key}就可以取出map中的值；

(@Param("id")Integer id,@Param("lastName")String lastName);
ParamNameResolver解析参数封装map的；
//1、names：{0=id, 1=lastName}；构造器的时候就确定好了

	确定流程：
	1.获取每个标了param注解的参数的@Param的值：id，lastName；  赋值给name;
	2.每次解析一个参数给map中保存信息：（key：参数索引，value：name的值）
		name的值：
			标注了param注解：注解的值
			没有标注：
				1.全局配置：useActualParamName（jdk1.8）：name=参数名
				2.name=map.size()；相当于当前元素的索引
	{0=id, 1=lastName,2=2}
				

args【1，"Tom",'hello'】:

public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    //1、参数为null直接返回
    if (args == null || paramCount == 0) {
      return null;
     
    //2、如果只有一个元素，并且没有Param注解；args[0]：单个参数直接返回
    } else if (!hasParamAnnotation && paramCount == 1) {
      return args[names.firstKey()];
      
    //3、多个元素或者有Param标注
    } else {
      final Map<String, Object> param = new ParamMap<Object>();
      int i = 0;
      
      //4、遍历names集合；{0=id, 1=lastName,2=2}
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
      
      	//names集合的value作为key;  names集合的key又作为取值的参考args[0]:args【1，"Tom"】:
      	//eg:{id=args[0]:1,lastName=args[1]:Tom,2=args[2]}
        param.put(entry.getValue(), args[entry.getKey()]);
        
        
        // add generic param names (param1, param2, ...)param
        //额外的将每一个参数也保存到map中，使用新的key：param1...paramN
        //效果：有Param注解可以#{指定的key}，或者#{param1}
        final String genericParamName = GENERIC_NAME_PREFIX + String.valueOf(i + 1);
        // ensure not to overwrite parameter named with @Param
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }
}
```



### 参数值的获取

#{key}获取map或者POJO中的值，\${key}也可以

区别：
	#{}:是以预编译的形式，将参数设置到sql语句中；PreparedStatement；防止sql注入
	${}:取出的值直接拼装在sql语句中；会有安全问题；

```
String sql = "select * from user_table where username=
' "+userName+" ' and password=' "+password+" '";

当输入了上面的用户名和密码，上面的SQL语句变成：
SELECT * FROM user_table WHERE username=
'’or 1 = 1 -- and password='’

条件后面username=”or 1=1 用户名等于 ” 或1=1 那么这个条件一定会成功；

然后后面加两个-，这意味着注释，它将后面的语句注释，让他们不起作用，这样语句永远都能正确执行，用户轻易骗过系统，获取合法身份。

这还是比较温柔的，如果是执行
SELECT * FROM user_table WHERE
username='' ;DROP DATABASE (DB Name) --' and password=''
其后果可想而知…
```

​	大多情况下，我们去参数的值都应该去使用#{}；

![image-20211019095644336](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211019095644336.png)



  \${}用处：原生sql在不支持占位符的地方可以用${}取值

​                  比如分表

![image-20211019101019760](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211019101019760.png)



#{}:更丰富的用法：
	规定参数的一些规则：
	javaType、 jdbcType、 mode（存储过程）、 numericScale、
	resultMap、 typeHandler、 jdbcTypeName、 expression（未来准备支持的功能）；

	jdbcType通常需要在某种特定的条件下被设置：
		在我们数据为null的时候，有些数据库可能不能识别mybatis对null的默认处理。比如Oracle（报错）；
		
		JdbcType OTHER：无效的类型；因为mybatis对所有的null都映射的是原生Jdbc的OTHER类型，oracle不能正确处理;
		
		由于全局配置中：jdbcTypeForNull=OTHER；oracle不支持；两种办法
		1、#{email,jdbcType=NULL};
		2、jdbcTypeForNull=NULL
			<setting name="jdbcTypeForNull" value="NULL"/>

![image-20211019102401610](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211019102401610.png)



## resultMap

自定义封装规则

![image-20211021134006987](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021134006987.png)

![image-20211021133856271](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021133856271.png)

![image-20211021202353145](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021202353145.png)

没有resultMap，也能查到数据，但是lastName会是null，因为没有匹配。



### 关联对象的封装

![image-20211021215021922](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021215021922.png)

![image-20211021214800105](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021214800105.png)

### association（关联对象）

或者使用association定义单个对象封装规则

![image-20211021220653985](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021220653985.png)

```java
//javabean
@Data
public class Employee {
    private Long id;
    private String lastName;
    private String gender;
    private String email;
    private Department department;

}
@Data
public class Department {
    Long id;
    String name;
}
```

```xml
//mapper映射文件
<resultMap id="ResultMap" type="com.mng.Employee">
        <id column="id" property="id"/>
        <result column="last_name" property="lastName"/>
        <result column="gender" property="gender"/>
        <result column="email" property="email"/>
        <association property="department" javaType="com.mng.Department">
            <id column="d_id" property="id"/>
            <result column="d_name" property="name" />
        </association>
    </resultMap>

<!--    public Employee getEmpById(Long id);-->
    <select id="getEmpById" resultMap="ResultMap">
        select * from tbl_employee where id = #{id}
    </select>

<!--       public Employee getEDById(Long id); -->
    <select id="getEDById" resultMap="ResultMap">
        select t1.*,t2.id d_id,t2.name d_name
        from tbl_employee t1 join  department t2
        on t1.dept_id = t2.id
        where t1.id = #{id}
    </select>
```

结果：

方法getEmpById结果：没有匹配的类名所以department为null

![image-20211021234215049](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021234215049.png)

方法getEDById结果，有匹配类名

![image-20211021235242277](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021235242277.png)

**分步查询**

![image-20211021224531032](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021224531032.png)

```
//mapper映射文件
    <resultMap id="ResultMap2" type="com.mng.Employee">
        <id column="id" property="id"/>
        <result column="last_name" property="lastName"/>
        <result column="gender" property="gender"/>
        <result column="email" property="email"/>
        <association property="department" select="com.mng.DepartmentMapper.getDepById"
        column="dept_id">
        </association>

    </resultMap>
    <!--    public Employee getEmpById(Long id);-->
    <select id="getEmpById" resultMap="ResultMap2">
        select * from tbl_employee where id = #{id}
    </select>


```

结果：对上面的getEmpById的resultMap改成了新的，查询到了结果

![image-20211021235438504](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211021235438504.png)



### 延迟加载

查询某个对象时，对于其关联对象，在使用时再去查询。

```
//全局配置
lazyLoadingEnabled 开启！！   ( 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置 fetchType 属性来覆盖该项的开关状态。)

aggressiveLazyLoading 关闭！！ (开启时，任一方法的调用都会加载该对象的所有延迟加载属性。 否则，每个延迟加载属性会按需加载 3.4.1后默认关闭)

```





### 关联集合

```
<mapper namespace="com.mng.DepartmentMapper">
<!--       public List<Department> getDep();-->
    <resultMap id="ListTest" type="com.mng.Department">
        <id column="id" property="id"/>
        <result column="name" property="name"/>
        <collection property="employeeList" ofType="com.mng.Employee" columnPrefix="e_">
            <id column="id" property="id"/>
            <result column="last_name" property="lastName"/>
            <result column="gender" property="gender"/>
            <result column="email" property="email"/>
        </collection>
    </resultMap>
    <select id="getDep" resultMap="ListTest">
        select t1.*,
        t2.id e_id,t2.last_name e_last_name,t2.gender e_gender,t2.email e_email
        from department t1 join tbl_employee t2 on  t1.id = t2.dept_id
    </select>
</mapper>
```

![image-20211026222824283](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026222824283.png)



**分步查询**

```
    //department映射文件
    <resultMap id="ListTest2" type="com.mng.Department">
        <id column="id" property="id"/>
        <result column="name" property="name"/>
        <collection property="employeeList"
                    select="com.mng.EmployeeMapMapper.getELByDeptId"
                    column="id">
        </collection>

    </resultMap>
    
    <select id="getDep2" resultMap="ListTest2">
        select * from department
    </select>
```

```
//employeement映射文件
    <resultMap id="ResultMap2" type="com.mng.Employee">
        <id column="id" property="id"/>
        <result column="last_name" property="lastName"/>
        <result column="gender" property="gender"/>
        <result column="email" property="email"/>
        <association property="department" select="com.mng.DepartmentMapper.getDepById"
        column="dept_id">
        </association>

    </resultMap>
    
<!--       public List<Employee> getELByDeptId(Long deptId); -->
    <select id="getELByDeptId" resultMap="ResultMap2">
        select * from tbl_employee where dept_id = #{deptId}
    </select>
```

![image-20211026230902551](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026230902551.png)





# 二.动态sql

用于拼接sql

## 1. if

```
//接口,多个参数（还是自定义类），所以命名了参数
public interface EmployeeDSQLMapper {
     public List<Employee> getEmpByNameAndDept(@Param("name") String name,@Param("department") Department department);
}
```

```
//映射文件
    <resultMap id="ResultMap" type="com.mng.Employee">
        <id column="id" property="id"/>
        <result column="last_name" property="lastName"/>
        <result column="gender" property="gender"/>
        <result column="email" property="email"/>
        <association property="department" javaType="com.mng.Department">
            <id column="d_id" property="id"/>
            <result column="d_name" property="name" />
        </association>
    </resultMap>
<!--  public List<Employee> getEmpByNameAndDept(String name,Department department); -->
    <select id="getEmpByNameAndDept" resultMap="ResultMap">
         select t1.*,t2.id  d_id,t2.name d_name from
         tbl_employee t1 join department t2
         on t1.dept_id = t2.id
         where t1.last_name = #{name}
         <if test="department!=null and department.name !=null">
             and t2.name = #{department.name}
         </if>
    </select>
```

测试：

1. deparatment.name为null

   ![image-20211026233650086](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026233650086.png)

2. departname.name = “dep1”

   ![image-20211026234131512](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026234131512.png)

## 2. choose [when otherwise]

```
//接口,多个参数（还是自定义类），所以命名了参数
public List<Employee> getEmp(@Param("id")Long id,@Param("name")String name,@Param("department") Department department);
```

```
  //映射文件
  <select id="getEmp" resultMap="ResultMap">
        select t1.*,t2.id  d_id,t2.name d_name from
        tbl_employee t1 join department t2
        on t1.dept_id = t2.id
        where 1=1
        <choose>
            <when test="id != null">
                and t1.id = #{id}
            </when>
            <when test="name != null">
                and t1.name = #{name}
            </when>
            <when test="department!=null and department.name !=null">
                and t2.name = #{department.name}
            </when>
        </choose>
    </select>
```

测试：

1. id为1，但是name，departname不是相应的数据

![image-20211026235429387](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026235429387.png)2. id为null, name为null

![image-20211026235709361](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211026235709361.png)



## 3.trim [where,set,update]

可以看到为了格式的正确，我在choose里用了 where 1=1 的语句，trim就是为了解决这样的问题。

**where:**

相当于

```
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

![image-20211027000736310](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211027000736310.png)

![image-20211027000751728](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211027000751728.png)



**set:**

相当于：

```
<trim prefix="SET" suffixOverrides=",">
  ...
</trim>
```

```
//借用文档例子
<update id="updateAuthorIfNecessary">
  update Author
    <set>
      <if test="username != null">username=#{username},</if>
      <if test="password != null">password=#{password},</if>
      <if test="email != null">email=#{email},</if>
      <if test="bio != null">bio=#{bio}</if>
    </set>
  where id=#{id}
</update>
```

## 4.foreach

![image-20211027001308141](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211027001308141.png)

```
//接口,命名参数
public  List<Employee> getEmpByDeptList(@Param("dapartmentList") List<Department> departmentList);
```

```
  //映射文件
      <select id="getEmpByDeptList" resultMap="ResultMap">
        select t1.*,t2.id  d_id,t2.name d_name from
        tbl_employee t1 join department t2
        on t1.dept_id = t2.id
        where t2.name in
        <foreach collection="dapartmentList" item="item" open="(" close=")" separator=",">
            #{item.name}
        </foreach>
    </select>
```

结果：

![image-20211027002338080](https://gitee.com/minan-palace/md_images/raw/master/images/image-20211027002338080.png)



## 5.script

要在带注解的映射器接口类中使用动态 SQL，可以使用 *script* 元素。比如:

```
    @Update({"<script>",
      "update Author",
      "  <set>",
      "    <if test='username != null'>username=#{username},</if>",
      "    <if test='password != null'>password=#{password},</if>",
      "    <if test='email != null'>email=#{email},</if>",
      "    <if test='bio != null'>bio=#{bio}</if>",
      "  </set>",
      "where id=#{id}",
      "</script>"})
    void updateAuthorValues(Author author);
```

## 6.bind

`bind` 元素允许你在 OGNL 表达式以外创建一个变量，并将其绑定到当前的上下文。比如：

```
<select id="selectBlogsLike" resultType="Blog">
  <bind name="pattern" value="'%' + _parameter.getTitle() + '%'" />
  SELECT * FROM BLOG
  WHERE title LIKE #{pattern}
</select>
```

   

## 7.多数据库支持

如果配置了 databaseIdProvider，你就可以在动态代码中使用名为 “_databaseId” 的变量来为不同的数据库构建特定的语句。比如下面的例子：

```
<insert id="insert">
  <selectKey keyProperty="id" resultType="int" order="BEFORE">
    <if test="_databaseId == 'oracle'">
      select seq_users.nextval from dual
    </if>
    <if test="_databaseId == 'db2'">
      select nextval for seq_users from sysibm.sysdummy1"
    </if>
  </selectKey>
  insert into users values (#{id}, #{name})
</insert>
```

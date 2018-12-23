---
title: mybatis lazyLoading 基本原理
date: 2018-12-17 14:19:31
tags:
    - Mybatis
---

Mybatis的延迟加载（lazyLoading）看起来似乎有丶东西，使用延迟加载，能让内层的查询（嵌套查询）语句在你需要获得内层的数据的时候再执行，默认触发内层查询的方法是resultMap对应的类对象(一般都是List)的四个方法（equals,clone,hashCode,toString）被调用的时候执行内层查询，你可以通过`lazyLoadTriggerMethods`属性设置触发内层查询的方法，[文档](http://www.mybatis.org/mybatis-3/configuration.html#settings)里面初略地提了一下。实际上调用任何以`get`或者`set`开头的方法也会触发延迟加载。

```
public class Article {

    private int id;

    private String title;

    private String content;

    private Timestamp createTime;

    private Timestamp modifyTime;

    private int viewCount;

    private int likeCount;

    private Author author;  // 内层（嵌套）对象
}
```
比如上面的`Article`对象，每一篇文章都有一个作者，因此里面有一个`Author`属性

```
public class Author {

    private int id;

    private String name;

    private String sex;
}
```

## 一个懒加载例子
这个例子使用Intellij IDEA运行，用mybatis-spring和Spring框架进行整合，并且用了Spring的事务管理，数据库是mysql。

### 需要的依赖
```
<!-- https://mvnrepository.com/artifact/org.springframework/spring-core -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-beans -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-beans</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-context -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-tx -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-tx</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.springframework/spring-jdbc -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-jdbc</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>3.4.6</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.mybatis/mybatis-spring -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis-spring</artifactId>
            <version>1.3.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>
```
mytabis-spring用到了spring-jdbc，所以需要引入这个依赖

### 目录结构
![Imgur](https://i.imgur.com/YuOEW8T.png)

`MybatisConfig.java`
```
@Configuration
@MapperScan(basePackages = {"com.pltm.mybatis.mysql.mapper"})
@EnableTransactionManagement
public class MybatisConfig {

    @Bean
    DataSource dataSource(){
        MysqlDataSource dataSource = new MysqlDataSource();
        dataSource.setURL("jdbc:mysql://localhost:3306/test?useSSL=false");
        dataSource.setUser("root");
        dataSource.setPassword("root");
        return dataSource;
    }

    @Bean
    DataSourceTransactionManager transactionManager(){
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    SqlSessionFactoryBean sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlb = new SqlSessionFactoryBean();
        sqlb.setDataSource(dataSource());
        org.apache.ibatis.session.Configuration cf = new org.apache.ibatis.session.Configuration();
        cf.setLazyLoadingEnabled(true);
        cf.setCacheEnabled(true);
        cf.setLogImpl(StdOutImpl.class);
        sqlb.setConfiguration(cf);
        // 上面其实可以用SqlSessionFactoryBean的setConfigLocation方法设置mybatis配置文件地址
        // 然后就可以在xml文件里面设置lazyLoadingEnabled这些属性了
        return sqlb;
    }
}
```

`ArticleMapper.java`
```
public interface ArticleMapper {

    public List<Article> listArticles();

    public List<Article> lazyListArticles();

}
```
`ArticleRepository`
```
@Repository
@Transactional
public class ArticleReopsitory {

    @Autowired
    ArticleMapper articleMapper;

    public List<Article> lazyGetArticles(){
        return articleMapper.lazyListArticles();
    }

}
```
`Bootstrap.java`这个类是用来写个静态方法让应用能跑起来的
```
@ComponentScan
public class Bootstrap {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Bootstrap.class);
        ArticleReopsitory articleReopsitory = ctx.getBean(ArticleReopsitory.class);
        List<Article> articles = articleReopsitory.lazyGetArticles();
        for(Article a:articles){
            System.out.println(articles);
        }
    }
}
```

`ArticleMapper.xml`
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.pltm.mybatis.mysql.mapper.ArticleMapper">
    <resultMap id="lazyArticle" type="com.pltm.mybatis.mysql.entity.Article">
        <id column="id" property="id"></id>
        <result column="title" property="title"></result>
        <result column="content" property="content"></result>
        <result column="view_count" property="viewCount"></result>
        <result column="like_count" property="likeCount"></result>
        <result column="create_time" property="createTime"></result>
        <result column="modify_time" property="modifyTime"></result>
        <association property="author"
                     select="findAuthorById"
                     fetchType="lazy"
                     column="author_id" javaType="com.pltm.mybatis.mysql.entity.Author">
            <id column="author_id" property="id"></id>
            <result column="name" property="name"></result>
            <result column="sex" property="sex"></result>
        </association>
    </resultMap>

    <select id="lazyListArticles" resultMap="lazyArticle">
        select * from article
    </select>

    <select id="findAuthorById" resultType="com.pltm.mybatis.mysql.entity.Author">
        select * from author where id = #{id}
    </select>
</mapper>
```
上面的resultMap返回的对象为类型为List，因此在获取返回结果的任意一个Article对象的时候，会调用到List的get方法然后触发延迟加载

`Article`和`Author`两个类在开头已经贴出来了，这里就不贴了，get和set还有toString方法可以用IDE自动生成，如果是IDEA的话，按一下ALT + Insert再选一下就行

### 表结构
article表
```
create table article
(
	id int auto_increment
		primary key,
	title varchar(50),
	content varchar(10000),
	create_time timestamp,
	modify_time timestamp default '1980-01-01 00:00:01',
	author_id int not null,
	view_count int,
	like_count int
);
```
上面的modify_time的时间戳默认值也是有[丶](https://stackoverflow.com/questions/9192027/invalid-default-value-for-create-date-timestamp-field)东西的，>`TIMESTAMP has a range of '1970-01-01 00:00:01' UTC to '2038-01-19 03:14:07' UTC`，燃鹅我当时设置成1970-01-01 00:00:01也是不行，反复试探发现要设置大一点才ok

author表
```
create table author
(
	id int auto_increment
		primary key,
	name varchar(20) not null,
	sex varchar(8)
);
```

然后手动插入一些数据，登录到数据库，在test数据库（没有的话先创建）执行`insert into author(name, sex) values('tom,'man')`和`insert into article(title,content,author_id) values('titleaaa','content....',1)`，author_id填前面插入到author的id，如果之前没有插入过记录的话就是1

### 运行结果
![Imgur](https://i.imgur.com/BgR1RyE.png)
从输入日志可以看到先后执行了两条sql语句，`Preparing: select * from article `和`Preparing: select * from author where id = ? `，假设在`List<Article> articles = articleReopsitory.lazyGetArticles();`这句的后面设置一个断点，然后debug运行，当跑到断点处的时候，去查看控制台输出，会发现只输出了第一个查询语句的相关信息，如下图：
![Imgur](https://i.imgur.com/fjVjVci.png)

然后在调试窗口展开articles变量的时候，会发现控制台已经输出了第二条sql语句的相关信息，说明调试工具用反射获取这些信息的时候调用了那四个方法(equals,clone,hashCode,toString)的其中一个或几个。

![Imgur](https://i.imgur.com/819OexF.png)
展开articles变量后控制台输出信息如下：
![Imgur](https://i.imgur.com/FJvjKnB.png)
还可以注意到第二条查询语句（嵌套查询语句）是非事务性的（没有被Spring的Transaction管理）
>JDBC Connection [com.mysql.jdbc.JDBC4Connection@5af9926a] will not be managed by Spring

具体原因还不清楚，个人猜测是因为`ResultLoaderMap`的load(final Object userObject) 这个方法，里面有一段注释：
>We are using a new executor because we may be (and likely are) on a new thread and executors aren't thread safe. (Is this sufficient?)A better approach would be making executors thread safe.

![Imgur](https://i.imgur.com/evkQuVk.png)

总结一下因为mybatis`Executor`对象不是线程安全的，所以它们新创建了一个Executor对象来执行第二个嵌套查询语句


### 延迟加载的实现
具体实现就是用Javassist或者cglib来创建一个增强类，然后这个增强类在它的代理对象的相应的方法执行时，会判断执行的方法是不是在lazyLoadTriggerMethods中存在（默认是equals,clone,hashCode,toString这四个方法），下面是用Javassist实现的一小段代码，完整代码可以查看`JavassistProxyFactory`类：
```
if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {
              // 如果执行的方法是lazyLoadTriggerMethods里面的方法，会触发延迟查询语句的执行
              // 如果设置了aggressiveLazyLoading为ture也会触发嵌套查询语句的执行
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                // 执行延迟查询
                lazyLoader.loadAll();
              } else if (PropertyNamer.isSetter(methodName)) {
                // 调用的是set开头的方法，触发嵌套查询语句执行
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
              } else if (PropertyNamer.isGetter(methodName)) {
                  // 调用的是get开头的方法，触发嵌套查询语句执行
                final String property = PropertyNamer.methodToProperty(methodName);
                if (lazyLoader.hasLoader(property)) {
                  lazyLoader.load(property);
                }
              }
            }
```
另一种实现在`CglibProxyFactory`类里面可以看到，和上面的是类似的。两者都实现了`ProxyFactory`接口：
```
public interface ProxyFactory {

  void setProperties(Properties properties);

  // 下面这个方法就是用于创建增强类的，创建增强类之前需要创建一个handler
  // handlr里面要执行逻辑就是前面贴出来的那段代码
  Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs);

}
```

这里介绍得比较粗略，详细的实现在org.apache.ibatis.executor.loader包下可以看到。

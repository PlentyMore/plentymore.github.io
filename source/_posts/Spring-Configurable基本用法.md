---
title: Spring @Configurable基本用法
date: 2018-12-11 20:38:20
tags:
    - Spring
---

{% meting "1316589134" "netease" "song" %}

关于`@Configurable`的用法，[Spring文档](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#aop-atconfigurable)有详细的描述，不过由于看得比较粗略，后面实际使用的时候踩了不少坑。这个注解有以下几个用途：

### 为非Spring管理的对象注入Spring Bean

非Spring管理的对象，这里指的是我们自己new的对象，比如`Dog dog = new Dog()`，这个dog对象的生命周期不是由Spring管理的，而是我们自己创建的对象，根据文档的说法，我们只要在类上面加上`@Configurable`注解，就可以让Spring来配置这些非Spring管理的对象了，即为它们注入需要的依赖（实际上还有很多额外的工作要做）。下面有个例子

```java
// Account类，使用new操作符号手动创建，不交由Spring Container管理
@Configurable(autowire = Autowire.BY_TYPE)
public class Account {

    @Autowired
    Dog dog;

    public void output(){
        System.out.println(dog);
    }

}
```

上面的`Account`类使用的是`@Configurable`，而不是`@Configuration`，`@Configuration`类似于XML配置里面的`<beans></beans>`，我们可以在`<beans></beans>`里面声明要交由Spring Cotainer创建和管理的Bean，因此我们可以在`@Configuration`注解的类里面使用`@Bean`注解达到同样的效果，注意被`@Configuration`标注的类也会被Spring Container创建和管理，因此它也是一个Bean。

被`@Configuration`标注的类，能够被Spring配置，然后当我们手动创建`Account`对象的时候(`Account acc = new Account()`)，Spring将会用创建一个`Account`的Bean，然后这个Bean能被Spring正常地注入需要的属性，接着Spring使用这个Bean来设置我们刚刚创建的
`Account`对象（acc）的属性，最后返回的对象的属性就和Bean的一样了。

```java
// 配置类，使用注解的方式创建Bean
@Configuration
@EnableLoadTimeWeaving
@EnableSpringConfigured
public class Config {

    // 这个Bean将会被注入到Account的属性中
    @Bean
    Dog dog(){
        Dog d = new Dog();
        d.setId(1);
        d.setName("dog");
        return d;
    }
}
```

```java
public class Dog {
    private int id;
    private String name;
    // set，get和toString方法就不贴了
}
```

```java
// 启动类
@ComponentScan
public class App {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(App.class);
        Account account = new Account();
        account.output();
    }
}
```

程序输入结果如下：

![Imgur](https://i.imgur.com/AmyJI2t.png)

可以看到，我们自己创建的`Account`对象，输出了`Dog`对象，而不是null。因为Spring使用AspectJ的LTW(LoadTimeWeaving)技术为我们自己创建的`Account`对象注入了`Dog`对象。

要使上面的例子正常运行，需要满足几个要求：

* spring-core, spring-beans, spring-context, spring-instrument, spring-aspects, aspectjweaver, spring-tx等几个依赖要有，其中spring-tx是可选的，没有的话会输出一些警告信息。

* `@EnableLoadTimeWeaving`和`@EnableSpringConfigured`两个注解必须有，可以注解在任意被`@Configuration`注解的类上面

* 运行前加上`-javaagent:/path/to/spring-instrument.jar`这个jvm参数，`/path/to/spring-instrument.jar`为你的`spring-instrument`jar包的路径

如果用的maven管理依赖，并且IDE为Intellij IDEA，可以打开Project Structure的Libraries选项查看jar包的路径，然后在run/debug里面配置jvm参数。
![Imgur](https://i.imgur.com/OaOBYam.png)
![Imgur](https://i.imgur.com/rotXT5P.png)



#### How it works?

首先，如果仅仅有`@Configuration`注解，是不起任何作用的，因此我们这时候手动创建`Account`对象的话，将会输出null。因为起到关键作用的时候`AnnotationBeanConfigurerAspect `这个类（它是使用aspect定义的，而不是class，因为我还没用过aspectj编程，所以还不太了解），而这个类的实例在添加`@EnableSpringConfigured`注解后将会被Spring创建，然后Spring将会结合LWT技术调用这个对象里面的方法对我们手动创建的`Account`对象的属性进行处理。


```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(SpringConfiguredConfiguration.class) //然后我们去看看SpringConfiguredConfiguration干了什么
public @interface EnableSpringConfigured {

}
```

```java
@Configuration
public class SpringConfiguredConfiguration {

	/**
	 * The bean name used for the configurer aspect.
	 */
	public static final String BEAN_CONFIGURER_ASPECT_BEAN_NAME =
			"org.springframework.context.config.internalBeanConfigurerAspect";

	@Bean(name = BEAN_CONFIGURER_ASPECT_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AnnotationBeanConfigurerAspect beanConfigurerAspect() {
		return AnnotationBeanConfigurerAspect.aspectOf();
	}

}
```

从上可以看到，一个AnnotationBeanConfigurerAspect类型的Bean将会被Spring Container创建和管理，因此它就是让`@Configurable`注解起作用的核心。


```java
public aspect AnnotationBeanConfigurerAspect extends AbstractInterfaceDrivenDependencyInjectionAspect
		implements BeanFactoryAware, InitializingBean, DisposableBean{
      //类的实现就不贴了，从它实现的接口也可以猜到一些东西
    }
```

__文档上面还提到，不仅仅是使用new创建的时候，在反序列化的时候，被`@Configurable`注解的类和使用new创建的时候一样，会被拦截然后注入属性__


### 方便单元测试

当类没有被AspectJ动态织入（LoadTimeWeaving）的时候，`@Configuration`是不起任何作用的，因此对单元测试不会产生影响

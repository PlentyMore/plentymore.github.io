---
title: Spring IoC原理
date: 2018-12-24 19:45:30
tags:
    - Spring
---
Spring的核心是它的IoC（控制反转）容器和AOP（面向切面编程），Spring与其它类库的整合（比如Spring事务，Spring Data JDBC等）大部分都是在建立在这两个功能的基础上的。在使用Spring的IoC容器的时候，在类上面加几个注解就可以了，用起来非常的简单，具体的原理我只知道一点点（比如`BeanFactory`可以访问IoC容器，可以用它来获取创建好的Bean实例，创建好的Bean实例（仅限Scope为Singleton的）放在一个`Map`中，根据`BeanDefinition`创建Bean实例之类的），由于只是一知半解，在读Spring Boot源码的时候显得有点吃力，所以就跑回来研究Spring最基础的IoC容器和AOP了。

在读源码之前，我觉得首先最好要非常熟悉怎样使用Spring，具体就是怎样用它来为我们注入依赖（根据类型注入，根据名称注入，构造器注入等），有哪几种配置方式（xml配置，java注解配置，java注解 + xml配置等），只有非常了解它可以做什么，并能够熟练地使用它来做这些事情，才能更好地理解源码。要熟悉怎样使用，可以直接看Spring的Reference Doc，没有人比它自己更了解自己，看官方文档是最好的选择。每一个Spring项目基本上都配有相应的文档，项目和文档会一起更新。除了文档之外，还会有API Doc，看源码的时候，可以打开相应的API Doc，里面可以很清楚地看到源码的组织结构，每一个package包含哪些类和接口，和它们的主要作用，都可以看的一清二楚，还可以查看类和接口上面的注释（有时候在IDE里面看源码的时候，注释和代码混在一起，想要快速查看某个类有哪些方法会比较麻烦，这时候看API Doc就很方便了，因为API Doc上面只有方法签名和注释，没有具体实现的代码）。

一般读源码的时候，我会DUBUG设置断点根据程序的执行流程跟着去看。
下面有一个很简单的程序，具体功能是使用基于Java注解的方式配置一个Bean，然后用`ApplicationContext`来获取这个Bean的实例
`Bs.java`
```java
@ComponentScan
public class Bs {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Bs.class);
        MyBean mb = ctx.getBean(MyBean.class);
    }
}
```

`Config.java`
```java
@Configuration
public class Config {

    @Bean
    public MyBean myBean(){
        return new MyBean();
    }
}
```

`MyBean.java`
```java
public class MyBean {
}
```

一个有三个类文件，Bs.java，Config.java和MyBean.java
这个程序非常简单，主要是想利用它来DUBUG设置断点，看一下具体的执行流程

在`AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Bs.class);`这行设置断点，跟踪`AnnotationConfigApplicationContext`的创建和初始化过程。首先来看一下类的继承关系
![Imgur](https://i.imgur.com/FJATQ07.png)

图片里面很多类和接口，本来想删掉一些让继承关系看起来更清晰的，但是感觉每一个都很必要，删掉之后总会漏掉什么

## 几个重要的接口
只列举出比较常用和重要的接口
### BeanFactory
[BeanFactory](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/BeanFactory.html)是用来访问Spring IoC容器的顶层接口，可以用它获取容器里面存放的Bean，一般我们通常不使用它来直接获取Bean（`MyBean mb = beanFactory.getBean(MyBean.class)`），而是通过依赖注入的方式（通过构造器，或者setter方法来注入相应的依赖）。实现这个接口的类，里面通常存放有很多的`BeanDefinition`，通常存放在一个`Map`里面，key为`BeanDefinition`对应的类的全限定名，value为`BeanDefinition`实例，`BeanFactory`可以从不同的配置源（比如xml文件，Java注解等）加载这些`BeanDefinition`。`BeanFactory`根据`BeanDefinition`来创建Bean的实例。主要的方法是getBean方法

#### ListableBeanFactory
[ListableBeanFactory](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/ListableBeanFactory.html)拓展了`BeanFactory`接口，它将预先创建好所有的Bean的实例，而不是要使用某个Bean的时候才创建。

#### HierarchicalBeanFactory
[HierarchicalBeanFactory](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/HierarchicalBeanFactory.html)是分级的`BeanFactory`，比如分成父级和子级，调用containsLocalBean方法查找Bean的时候只在当前级别查找，当前级别Bean不存在则返回空，不会向父级查找

#### AutowireCapableBeanFactory
[AutowireCapableBeanFactory](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/config/AutowireCapableBeanFactory.html)提供自动装配能力（自动地根据Bean名称或者类型配置好依赖，无需手动配置）的`BeanFactory`，它是实现依赖注入的关键接口。

### ApplicationContext
[ApplicationContext](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/context/ApplicationContext.html)拥有`BeanFactory`的所有功能（在`ApplicationContext`的实现类里面可以看到有一个`BeanFactory`实现类的成员），还提供了很多额外的功能（它实现了很多其它接口），比如加载文件资源(实现了`ResourceLoader`接口)，发布事件（实现了`ApplicationEventPublisher`接口），国际化（指多语言支持，实现了`MessageSource`接口）等，它同时继承了`ListableBeanFactory`和`HierarchicalBeanFactory`接口，所以它是将所有的Bean的实例预先创建好的，而且还可以分级，比如一个rootApplicationContext，然后多个subApplicationContext。

#### ConfigurableApplicationContext
[ConfigurableApplicationContext](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/context/ConfigurableApplicationContext.html)是可配置的`ApplicationContext`，能够在`ApplicationContext`被创建之后继续配置配置它，比如添加`BeanFactoryPostProcessor`，`ApplicationListener`等

### BeanDefinition
[BeanDefinition](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/config/BeanDefinition.html)用来描述一个Bean，比如这个Bean对应的类名是什么，是不是要延迟加载（要使用的时候才创建Bean的实例），是不是单例，能不能用于自动装配，它的依赖有哪些等

### BeanDefinitionRegistry
[BeanDefinitionRegistry](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/support/BeanDefinitionRegistry.html)相当于一个存放`BeanDefinition`的仓库，它提供了注册、移除、获取`BeanDefinition`的方法（其实就是把`BeanDefinition`放进`Map`，从`Map`移除，从`Map`查找），这个接口一般由bean Definition readers实现，因为bean Definition readers要从配置文件解析`BeanDefinition`然后存起来

### BeanFactoryPostProcessor
[BeanFactoryPostProcessor](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/config/BeanFactoryPostProcessor.html)接口是Spring IoC容器的[拓展点](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-factory-extension-factory-postprocessors)之一，实现了这个接口类配置成Bean后能够被`ApplicationContext`自动检测到（后面会讲到，其实就是用`BeanFactory`获取所有实现了这个接口的Bean）。Spring IoC容器使用`BeanFactoryPostProcessor`来读取Bean配置元数据，也就是`BeanDefinition`，可以在实例化Bean前对`BeanDefinition`作任意的修改。
该接口仅有这样一个方法
`void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException`你可以实现这个接口，然后在这个方法里面修改`BeanFactory`存储的`BeanDefinition`，然后把你实现的`BeanFactoryPostProcessor`注册到IoC容器中，然后就能够实现自定义配置`BeanDefinition`了。可以注册多个`BeanFactoryPostProcessor`到容器，最好实现`Order`接口指定执行的优先级，优先级高的`BeanFactoryPostProcessor`会先执行。

### BeanPostProcessor
[BeanPostProcessor](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/config/BeanPostProcessor.html)接口是Spring IoC容器的[另一个拓展点](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-factory-extension-bpp)，它和`BeanFactoryPostProcessor`类似，不同之处在于，`BeanFactoryPostProcessor`是处理`BeanDefinition`的，而`BeanPostProcessor`是处理实例化的Bean的（实际上`BeanFactoryPostProcessor`不仅仅可以处理BeanDefinition，它甚至可以实例化Bean，因为它的方法参数是`beanFactory`，因此它能做的事情远远不止这些，但是官方不建议用它来做其他事情），实现`BeanPostProcessor`接口，你可以自定义Bean实例化的逻辑，还有Bean的依赖解析逻辑（你可以用自定义的逻辑覆盖默认的逻辑）。该接口的其它方面和`BeanFactoryPostProcessor`类似，比如都要注册到容器，能被`ApplicationContext`自动检测到，可以注册多个实例等


## 创建ApplicationContext
回到`AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(Bs.class)`这行代码，我在这里设置了一个断点，以便观察它的创建流程。`AnnotationConfigApplicationContext`是`ApplicationContext`的一个具体实现类，`ApplicationContext`的具体实现类有如下几个（除了接口其他的类都是具体的实现类）
![Imgur](https://i.imgur.com/TJ2pCSo.png)

从上面的图中可以看到`AnnotationConfigApplicationContext`是`GenericApplicationContext`的子类，`AnnotationConfigApplicationContext`被调用的构造方法如下：

```java
	public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
		this();
		register(annotatedClasses);
		refresh();
	}
```

下面的图主要用来看会调用哪几个父类的构造器和代码块等

![Imgur](https://i.imgur.com/3yvk42M.png)

从图中可以看到，有三个父类（其他的都是接口，没有构造器）

### this.()
将调用这个构造函数创建`ApplicationContext`，首先调用了`AnnotationConfigApplicationContext`类的无参构造器，然后调用register方法，最后调用refresh方法，由于它还有父类，因此父类的构造方法或者父类的静态代码块先执行，它的直接父类`GenericApplicationContext`的构造方法如下：
```java
public GenericApplicationContext() {
		this.beanFactory = new DefaultListableBeanFactory();
	}
```

可以看到它创建了一个`DefaultListableBeanFactory`，这是`BeanFactory`的一种实现，后面将会用到这个`BeanFactory`。
`GenericApplicationContext`的直接父类为`AbstractApplicationContext`，它里面有一段静态代码块：
```java
	static {
		// Eagerly load the ContextClosedEvent class to avoid weird classloader issues
		// on application shutdown in WebLogic 8.1. (Reported by Dustin Woods.)
		ContextClosedEvent.class.getName();
	}
```
这段静态代码块将会最在它的父类的静态代码块执行之后再执行（静态代码块的执行发生在代码块和构造器之前，所以静态代码块会比构造器先执行），作用是把`ContextClosedEvent`类尽快地加载进来避免一些迷之加载器bug

由于`AbstractApplicationContext`还有一个父类`DefaultResourceLoader`，所以它的构造器将会在`DefaultResourceLoader`的构造器执行之后再执行，`DefaultResourceLoader`里面有静态代码块，主要做的事情是将Java基本类型映射到Java包装类（比如int.class -> Integer.class具体可以去看代码）。它的构造器如下：
```java
public DefaultResourceLoader() {
    // 获取默认的类加载器，并设置为DefaultResourceLoader的类加载器
	this.classLoader = ClassUtils.getDefaultClassLoader();
}
```

再回到`AbstractApplicationContext`，它的无参构造器如下：
```java
	/**
	 * Create a new AbstractApplicationContext with no parent.
	 */
	public AbstractApplicationContext() {
		this.resourcePatternResolver = getResourcePatternResolver();
	}
	protected ResourcePatternResolver getResourcePatternResolver() {
		return new PathMatchingResourcePatternResolver(this);
	}
```
创建了一个`PathMatchingResourcePatternResolver`实例，用于根据路径解析出路径里面的资源。



调用`AnnotationConfigApplicationContext`的构造参数的时候，具体的执行顺序为：
1. `AbstractApplicationContext`的静态代码块
2. `AbstractApplicationContext`的无参数构造器（创建了`PathMatchingResourcePatternResolver`的实例）
3. `GenericApplicationContext`的无参构造器（创建了`DefaultListableBeanFactory`的实例）
4. `AnnotationConfigApplicationContext`的参数类型为`Class<?>`的构造器（调用了自身的无参构造器，调用了register和refresh方法）

`AnnotationConfigApplicationContext`的无参构造器如下：
```java
	public AnnotationConfigApplicationContext() {
		this.reader = new AnnotatedBeanDefinitionReader(this);
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```
创建了`AnnotatedBeanDefinitionReader`和`ClassPathBeanDefinitionScanner`的实例，前者用于读取`BeanDefinition`的配置文件信息（这里是基于注解的配置），后者用于扫描`BeanDefinition`的配置文件信息（这里也是基于注解的配置）

重点留意`AnnotatedBeanDefinitionReader`，创建它的实例的时候，会注册很多的`BeanFactoryPostProcessor`和`BeanFactoryPostProcessor`（后面将统称为PostProcessor），用于读取`BeanDefinition`的配置文件信息和创建Bean，它的构造方法如下：
```java
	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry) {
		this(registry, getOrCreateEnvironment(registry));
	}

	public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
		Assert.notNull(environment, "Environment must not be null");
		this.registry = registry;
		this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);

		// 这个方法将会注册需要的PostProcessor
		AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
	}
```

重点看`AnnotationConfigUtils`的registerAnnotationConfigProcessors方法，它将会注册很多的PostProcessor
```java
	public static void registerAnnotationConfigProcessors(BeanDefinitionRegistry registry) {
		registerAnnotationConfigProcessors(registry, null);
	}
```
它调用了另一个registerAnnotationConfigProcessors方法（方法重载）
```java
	/**
	 * Register all relevant annotation post processors in the given registry.
	 * @param registry the registry to operate on
	 * @param source the configuration source element (already extracted)
	 * that this registration was triggered from. May be {@code null}.
	 * @return a Set of BeanDefinitionHolders, containing all bean definitions
	 * that have actually been registered by this call
	 */
	public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);
		if (beanFactory != null) {
			if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
				beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
			}
			if (!(beanFactory.getAutowireCandidateResolver() instanceof ContextAnnotationAutowireCandidateResolver)) {
				beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
			}
		}

        // 用BeanDefinitionHolder把PostProcessor的BeanDefinition存起来
		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			// 将注册ConfigurationClassPostProcessor
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			// 将注册AutowiredAnnotationBeanPostProcessor
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			// 将注册CommonAnnotationBeanPostProcessor
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			// 将注册PersistenceAnnotationBeanPostProcessor
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
			// 将注册EventListenerMethodProcessor
			RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
			// 将注册DefaultEventListenerFactory
			RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
		}

		return beanDefs;
	}
```

registerAnnotationConfigProcessors主要注册了如下的几个PostProcessor

1. `ConfigurationClassPostProcessor`，注册的名称为"org.springframework.context.annotation.internalConfigurationAnnotationProcessor"，用于处理被`@Configuration`注解标注的类，也就是从被`@Configuration`标注的类里面读取`BeanDefinition`配置信息，这个PostProcessor会在以下的情况被注册：xml配置文件里面有<context:annotation-config/>或者<context:component-scan/>，或者类上面有`@ComponentScan`注解

2. `AutowiredAnnotationBeanPostProcessor`，注册的名称为"org.springframework.context.annotation.internalAutowiredAnnotationProcessor"，用于实现依赖属性的自动装配（根据名称或者类型自动注入合适的属性），能够处理`@Autowire`、`@Value`和`@Inject`注解，这个PostProcessor会在以下的情况被注册：和上面的`ConfigurationClassPostProcessor`一样。需要注意的是，xml手动指定的属性会覆盖掉自动装配注入的属性，因为自动装配发生在xml之前

3. `CommonAnnotationBeanPostProcessor`，注册的名称为"org.springframework.context.annotation.internalCommonAnnotationProcessor"，用于支持处理JSR-250的标准注解，比如`@javax.annotation.PostConstruct`，`@javax.annotation.PreDestroy`，`javax.annotation.Resource`。这个PostProcessor会在以下的情况被注册：JSR-250的jar包在classpath中（代码上通过`ClassUtils.isPresent("javax.annotation.Resource", classLoader)`这句代码检测的）

4. `org.springframework.orm.jpa.support.PersistenceAnnotationBeanPostProcessor`（特意用了全限定名，因为它在orm包里面），注册的名称为"org.springframework.context.annotation.internalPersistenceAnnotationProcessor"，主要用于处理 `@PersistenceUnit`和`@PersistenceContext`注解，以便注入相应的`EntityManagerFactory`和`EntityManager`（这些东西要用过Spring集成的Hibernate JPA才懂）。这个PostProcessor会在以下的情况被注册：SpringFramework ORM模块和JPA的API模块在classpath中（代码上通过`ClassUtils.isPresent("javax.persistence.EntityManagerFactory", classLoader) && ClassUtils.isPresent(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, classLoader)`进行检测）

5. `EventListenerMethodProcessor`，注册的名称为"org.springframework.context.event.internalEventListenerProcessor"，主要用于处理`@EventListener`注解。这个PostProcessor会在以下的情况被注册：和`ConfigurationClassPostProcessor`一样。

6. `DefaultEventListenerFactory`，注册的名称为"org.springframework.context.event.internalEventListenerFactory"，主要用于为被`@EventListener`标注的方法创建`ApplicationListener`实例，`DefaultEventListenerFactory`的Bean创建后将会被设置到`EventListenerMethodProcessor`的eventListenerFactories属性中，被`EventListenerMethodProcessor`使用。这个PostProcessor会在以下的情况被注册：和`ConfigurationClassPostProcessor`一样。


### register(Class<?>... annotatedClasses)
上面的PostProcessor注册好后，`AnnotatedBeanDefinitionReader`的实例也将创建完成，然后创建完`ClassPathBeanDefinitionScanner`的实例之后，就要执行`AnnotationConfigApplicationContext`的register方法了
```java
	public void register(Class<?>... annotatedClasses) {
		Assert.notEmpty(annotatedClasses, "At least one annotated class must be specified");
		// 我传入的annotatedClasses为Bs.class
		this.reader.register(annotatedClasses);
	}
```
可以看到实际上调用的是`AnnotatedBeanDefinitionReader`的register方法，这个方法的具体实现如下
```
	public void register(Class<?>... annotatedClasses) {
		for (Class<?> annotatedClass : annotatedClasses) {
			registerBean(annotatedClass);
		}
	}
```
`AnnotatedBeanDefinitionReader`的register方法将遍历传入的annotatedClasses，然后调用registerBean方法，将annotatedClass传入参数（我只传入了一个Bs.class，因此这里实际调用将会只执行一次registerBean(Bs.class)）
```java
	public void registerBean(Class<?> annotatedClass) {
		// 实际调用将会是doRegisterBean(Bs.class)
		doRegisterBean(annotatedClass, null, null, null);
	}
```
可以看到又调用了另一个方法，`AnnotatedBeanDefinitionReader`的doRegisterBean方法
```java
	<T> void doRegisterBean(Class<T> annotatedClass, @Nullable Supplier<T> instanceSupplier, @Nullable String name,
			@Nullable Class<? extends Annotation>[] qualifiers, BeanDefinitionCustomizer... definitionCustomizers) {

		AnnotatedGenericBeanDefinition abd = new AnnotatedGenericBeanDefinition(annotatedClass);
		if (this.conditionEvaluator.shouldSkip(abd.getMetadata())) {
			return;
		}

		abd.setInstanceSupplier(instanceSupplier);
		ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(abd);
		abd.setScope(scopeMetadata.getScopeName());
		String beanName = (name != null ? name : this.beanNameGenerator.generateBeanName(abd, this.registry));

		AnnotationConfigUtils.processCommonDefinitionAnnotations(abd);
		if (qualifiers != null) {
			for (Class<? extends Annotation> qualifier : qualifiers) {
				if (Primary.class == qualifier) {
					abd.setPrimary(true);
				}
				else if (Lazy.class == qualifier) {
					abd.setLazyInit(true);
				}
				else {
					abd.addQualifier(new AutowireCandidateQualifier(qualifier));
				}
			}
		}
		for (BeanDefinitionCustomizer customizer : definitionCustomizers) {
			customizer.customize(abd);
		}

		BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName);
		definitionHolder = AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
		BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry);
	}
```
doRegisterBean方法里面终于是实际的逻辑了，主要步骤如下：
1. 调用`BeanNameGenerator`的generateBeanName方法生成Bean名称（传统是第一个字母小写的驼峰式），由于我传入的类叫做`Bs.class`，所以生成的名称将会是"bs"，具体可以看`BeanNameGenerator`的实现（这里用的具体实现类是`AnnotationBeanNameGenerator`）

2. 调用`AnnotationConfigUtils`的processCommonDefinitionAnnotations方法处理`BeanDefinition`对应的类上面的注解，下面看看它的具体实现
```java
	public static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd) {
		processCommonDefinitionAnnotations(abd, abd.getMetadata());
	}

	static void processCommonDefinitionAnnotations(AnnotatedBeanDefinition abd, AnnotatedTypeMetadata metadata) {
		// 看看有没有@Lazy注解
		AnnotationAttributes lazy = attributesFor(metadata, Lazy.class);
		if (lazy != null) {
			abd.setLazyInit(lazy.getBoolean("value"));
		}
		else if (abd.getMetadata() != metadata) {
			lazy = attributesFor(abd.getMetadata(), Lazy.class);
			if (lazy != null) {
				abd.setLazyInit(lazy.getBoolean("value"));
			}
		}

        // 看看有没有@Primary注解
		if (metadata.isAnnotated(Primary.class.getName())) {
			abd.setPrimary(true);
		}

        // 看看有没有@DependsOn注解
		AnnotationAttributes dependsOn = attributesFor(metadata, DependsOn.class);
		if (dependsOn != null) {
			abd.setDependsOn(dependsOn.getStringArray("value"));
		}
        // 看看有没有@Role注解
		AnnotationAttributes role = attributesFor(metadata, Role.class);
		if (role != null) {
			abd.setRole(role.getNumber("value").intValue());
		}
        // 看看有没有@Description注解
		AnnotationAttributes description = attributesFor(metadata, Description.class);
		if (description != null) {
			abd.setDescription(description.getString("value"));
		}
	}
```
主要检查了类上面有没有`@Lazy`，`@Primary`，`@DependsOn`，`@Role`，`@Description`这几个注解，如果有，则对`BeanDefinition`进行相应的设置，因为我的`Bs.class`类上没有上面的注解，所以就什么也不会发生

3. 看看有没有`BeanDefinitionCustomizer`，有的话调用它的customize方法

4. 调用`AnnotationConfigUtils`的applyScopedProxyMode方法，主要作用是把Bean配置成scoped proxy（如果类上有类似`@Scope(proxyMode = ScopedProxyMode.TARGET_CLASS)`这样的配置），因为我的`Bs.class`类似没有配置，所以默认是不配置成scoped proxy的，所以这个方法将不会产生任何影响。具体可以看[Scoped Beans as Dependencies](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-factory-scopes-other-injection)

5. 调用`BeanDefinitionReaderUtils`的registerBeanDefinition方法，其实就是调用`BeanDefinitionRegistry`的`registerBeanDefinition(String beanName, BeanDefinition beanDefinition)`方法注册`BeanDefinition`，在我的程序中，beanName为"bs"，`BeanDefinition`就是类为`Bs.class`的`BeanDefinition`

到这里doRegisterBean已经执行完成，`AnnotatedBeanDefinitionReader`的register方法也执行完成了，`AnnotatedBeanDefinitionReader`的register方法也执行完成了（因为我只传如了`Bs.class`，所以只执行一次就完成了）

### refresh()
执行`AnnotationConfigApplicationContext`的register方法后，将会执行它的refresh方法。
```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}
			......
			// 省略了后面的代码
		}
	}
```
refresh方法里面的逻辑十分清晰，按顺序调用了12个方法

#### 1. prepareRefresh
prepareRefresh方法在`AbstractApplicationContext`类里面
```java
	protected void prepareRefresh() {
		this.startupDate = System.currentTimeMillis();
		this.closed.set(false);
		this.active.set(true);

		// Initialize any placeholder property sources in the context environment
		initPropertySources();

		// Validate that all properties marked as required are resolvable
		// see ConfigurablePropertyResolver#setRequiredProperties
		getEnvironment().validateRequiredProperties();

		// Allow for the collection of early ApplicationEvents,
		// to be published once the multicaster is available...
		this.earlyApplicationEvents = new LinkedHashSet<>();
	}
```
这个方法主要设置了一些相关的属性，比如startupDate，closed等，然后调用了initPropertySources方法（目前这个方法是什么也不做），接着获取`Environment`实例（如果environment变量为空，则创建一个`StandardEnvironment`实例），调用`Environment`的validateRequiredProperties验证所有需要的属性（其实就是检查一下需要属性是否空），最后给earlyApplicationEvents赋值（值为`LinkedHashSet`实例）。

这里的看不太懂的是获取`Environment`实例这个步骤，因为从前面的步骤可以看到，这个environment变量属于`AbstractApplicationContext`类，但是并没有看到它为这个变量赋值，所以它一定是空的，所以一定会调用createEnvironment方法创建一个`StandardEnvironment`赋值给environment变量。接着看validateRequiredProperties方法，这个方法在`AbstractEnvironment`类里面
```java
	public void validateRequiredProperties() throws MissingRequiredPropertiesException {
		this.propertyResolver.validateRequiredProperties();
	}
```
它调用了propertyResolver的.validateRequiredProperties方法，而propertyResolver是这样赋值的：
```java
	private final ConfigurablePropertyResolver propertyResolver =
			new PropertySourcesPropertyResolver(this.propertySources);
```
所以将会调用`PropertySourcesPropertyResolver`的alidateRequiredProperties方法，而实际上该类并没有这个方法，这个方法是从它的父类`AbstractPropertyResolver`继承的
```java
    private final Set<String> requiredProperties = new LinkedHashSet<>()
	
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
```

在`AbstractPropertyResolver`类里面并没有看到往requiredProperties添加元素的步骤。有一个往requiredProperties添加元素的方法
```java
	public void setRequiredProperties(String... requiredProperties) {
		for (String key : requiredProperties) {
			this.requiredProperties.add(key);
		}
	}
```
在这个方法设置断点，发现这个方法并没有被调用（所以这个requiredProperties究竟是从什么地方放元素进去的？？？）

#### 2. obtainFreshBeanFactory
这个方法在`AbstractApplicationContext`类里面
```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		return getBeanFactory();
	}
```
里面调用了refreshBeanFactory和getBeanFactory方法。
先看方法refreshBeanFactory方法，它在`GenericApplicationContext`类里面（在这个例子中是在这个类里面）
```java
	/**
	 * Do nothing: We hold a single internal BeanFactory and rely on callers
	 * to register beans through our public methods (or the BeanFactory's).
	 * @see #registerBeanDefinition
	 */
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}
```
其实就是调用了`DefaultListableBeanFactory`的setSerializationId方法设置了一个用于序列化的id。
接下来看`AbstractApplicationContext`类的getBeanFactory方法
```java
public abstract ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
```
是个抽象方法，说明是留给子类实现的（模板方法模式），`GenericApplicationContext`类实现了这个方法
```java
	@Override
	public final ConfigurableListableBeanFactory getBeanFactory() {
		return this.beanFactory;
	}
```
直接放回了`GenericApplicationContext`的beanFactory变量，beanFactory变量已经在`GenericApplicationContext`的无参构造器里面赋值了，类型是`DefaultListableBeanFactory`

#### 3. prepareBeanFactory
这个方法在`AbstractApplicationContext`类里面
```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```
这个方法做了很多工作

* 首先它将`BeanFactory`的`BeanClassLoader`设置成和当前`ApplicationContext`相同的的`ClassLoader`，`BeanExpressionResolver`设置成`StandardBeanExpressionResolver`。

* 添加`PropertyEditorRegistrar`（用于注册自定义的`PropertyEditor`），类型为`ResourceEditorRegistrar`。

* 添加`BeanPostProcessor`，具体类型为`ApplicationContextAwareProcessor`，用于将`ApplicationContext`传递到实现了`*Aware`接口的Bean，它主要实现了`BeanPostProcessor`的postProcessBeforeInitialization方法，这个方法主要调用了它里面的一个私有方法invokeAwareInterfaces，这个歌方法会判断Bean实例有没有实现`*Aware`接口，如果有的话则将当前的`ApplicationContext`赋值到Bean实例里面的applicationContext属性，具体如下：
```java
	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```

* 将`*Aware`接口设置为不参与自动装配（意思是实现了`*Aware`接口的类，不会通过自动装配把它们注入到依赖属性里面，但是手动指定依赖关系依然是可以注入的）

* 将当前的`BeanFactory`设置成可以配自动装配到类型为`BeanFactory`的依赖属性中

* 将当前的`AbstractApplicationContext`设置成可以被自动装配到类型为`ResourceLoader`，`ApplicationEventPublisher`和`ApplicationContext`的依赖属性中（具体可以看`ConfigurableListableBeanFactory`的registerResolvableDependency方法上面的注释，这个方法是为了让一些不是Bean的对象也能用于自动装配，比如`BeanFactory`没有被定义成Bean，为了能让它用于自动装配，即我们调用这个方法让它可以用于自动装配），因为`AbstractApplicationContext`间接实现了`ResourceLoader`，`ApplicationEventPublisher`和`ApplicationContext`这几个接口，所以可以这样设置。

* 添加`BeanPostProcessor`，类型为`ApplicationListenerDetector`，它的主要作用是检测实现了`ApplicationListener`接口的Bean，它主要实现了`BeanPostProcessor`接口的postProcessAfterInitialization方法，在这个方法中，它会判断Bean实例是不是`ApplicationListener`类型（即是否实现了`ApplicationListener`接口），如果是，则将其添加到`ApplicationContext`，具体过程如下：
```java

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (bean instanceof ApplicationListener) {
			// potentially not detected as a listener by getBeanNamesForType retrieval
			Boolean flag = this.singletonNames.get(beanName);
			if (Boolean.TRUE.equals(flag)) {
				// singleton bean (top-level or inner): register on the fly
				this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
			}
			else if (Boolean.FALSE.equals(flag)) {
				this.singletonNames.remove(beanName);
			}
		}
		return bean;
	}
```

* 如果检测到名称为"loadTimeWeaver"的Bean，就添加`BeanPostProcessor`，类型为`LoadTimeWeaverAwareProcessor`，用于动态编入，接着在设置一个临时等等类加载器，类型为`ContextTypeMatchClassLoader`（具体作用还不清楚）

* 最后会注册几个默认的`Environment`类型的Bean，名称分别为"environment"，"systemProperties"和"systemEnvironment"

#### 4. postProcessBeanFactory
这个方法在`AbstractApplicationContext`类里面
```java
	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for registering special
	 * BeanPostProcessors etc in certain ApplicationContext implementations.
	 * @param beanFactory the bean factory used by the application context
	 */
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	}
```
是一个空方法，注释上说这个方法可以在`ApplicatinoContext`初始化之后修改它内部的`BeanFactory`，这时候所有Bean都已经被加载（`BeanDefinition`都创建好了），但是还没有创建Bean实例，所以在一些特定的`ApplicationContext`实现中，还能够继续注册特别的`BeanPostProcessor`这类东西（不是很懂它说的什么，感觉只看最前面那句注释就好了）
`AbstractApplicationContext`的子类`GenericApplicationContext`并没有重写这个方法，`AnnotationConfigApplicationContext`也没有，所以这个方法在这里是什么也不做的。

#### 5. invokeBeanFactoryPostProcessors
这个方法在`AbstractApplicationContext`类里面
```java
	/**
	 * Instantiate and invoke all registered BeanFactoryPostProcessor beans,
	 * respecting explicit order if given.
	 * <p>Must be called before singleton instantiation.
	 */
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
```
这个方法的名字非常直观，一眼就可以看出它是用来调用`BeanFactoryPostProcessor`的（注释上说是实例化并调用全部已经注册的`BeanFactoryPostProcessor`类型的Bean，调用的意思是调用`BeanFactoryPostProcessor`接口里面的方法）

它调用了`PostProcessorRegistrationDelegate`的invokeBeanFactoryPostProcessors方法，这个方法有点长，所以就不贴出来了，可以自己去看源码
它的主要逻辑如下：
* 首先调用`BeanDefinitionRegistryPostProcessor`（如果存在的话），
`BeanDefinitionRegistryPostProcessor`肯定是存在的，原因是在前面`AnnotatedBeanDefinitionReader`的doRegisterBean方法执行的时候，注册了几个PostProcessor，其中注册的第一个就是`ConfigurationClassPostProcessor`，它实现了`BeanDefinitionRegistryPostProcessor`接口
```java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware{

		}

public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```
所以首先会调用`ConfigurationClassPostProcessor`实现的的postProcessBeanDefinitionRegistry方法。怎么调用它的postProcessBeanDefinitionRegistry方法呢？前面好像只注册了它的`BeanDefinition`，还没有创建相应的Bean实例。实际上在调用它的方法前会调用`BeanFactory`的getBean方法，这个方法会获取它的Bean实例，因为实例还没创建，所以会先创建Bean实例，具体的逻辑如下：
```java
	String[] postProcessorNames =
			beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			// 下面调用了BeanFactory的getBean方法，这个方法会获取BeanDefinitionRegistryPostProcessor的实例
			// 如果没有获取到（这里还没创建，所以是肯定获取不到的），则会创建它的实例
			currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
			processedBeans.add(ppName);
		}
	}
	// 接着后面就能遍历processedBeans遍历，调用里面的实例的postProcessBeanDefinitionRegistry方法了
```
`ConfigurationClassPostProcessor`实现的的postProcessBeanDefinitionRegistry方法会调用它的processConfigBeanDefinitions方法，这个方法也是非常的长，所以也不贴出来了。但是这个方法非常的重要，因为它包含了解析我们的`@Configguration`和`@Bean`注解的逻辑，实际上它就是用来解析`@Configuration`类的（具体的实现逻辑可以去看源码，在`ConfigurationClassPostProcessor`类的processConfigBeanDefinitions里面）。

-- 先调用实现了`PriorityOrdered`接口的`BeanDefinitionRegistryPostProcessor` （这里其实会调用`ConfigurationClassPostProcessor`类实现的postProcessBeanDefinitionRegistry，后面其实已经没有其他的`BeanDefinitionRegistryPostProcessor`可以调用了）
-- 接着调用实现了`Order`接口的`BeanDefinitionRegistryPostProcessor`
-- 最后调用其它的所有`BeanDefinitionRegistryPostProcessor`

* 调用`BeanFactoryPostProcessor`（实际上到这里只剩下`EventListenerMethodProcessor`了，后面只会调用它，它是用来处理`@EventListener`注解的）
-- 先调用实现了`PriorityOrdered`接口的`BeanFactoryPostProcessor`
-- 接着调用实现了`Order`接口的`BeanFactoryPostProcessor`
-- 最后调用其它的所有`BeanFactoryPostProcessor`

调用完`PostProcessorRegistrationDelegate`会进行一些和动态织入相关的设置

#### 6. registerBeanPostProcessors
这个方法在`AbstractApplicationContext`类里面
```java
	protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
	}
```
它的主要作用是注册`BeanPostProcessor`，`BeanPostProcessor`用于在创建Bean的实例时候对Bean进行一些处理（比如覆盖实例化Bean的默认逻辑，添加自己解析Bean依赖的逻辑），该方法调用了`PostProcessorRegistrationDelegate`的registerBeanPostProcessors方法，逻辑和上面的调用invokeBeanFactoryPostProcessors方法是类似的
```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}
```
* 注册一个`BeanPostProcessor`，具体类型为`BeanPostProcessorChecker`，用于记录Bean被实例化的时候的info级别的日志
* 注册实现了`PriorityOrdered`接口的`BeanPostProcessor`（这里将会注册`AutowiredAnnotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`，之前只是注册它们的`BeanDefinitino`，这里是把它们注册为`BeanPostProcessor`）
* 注册实现了`Order`接口的`BeanPostProcessor`（没有这样的`BeanPostProcessor`）
* 注册常规的，也就是没有实现`PriorityOrdered`和`Order`接口的`BeanPostProcessor`（这里也没有这样的`BeanPostProcessor`）
* 重写注册所有内部的`BeanPostProcessor`（`MergedBeanDefinitionPostProcessor`类型的`BeanPostProcessor`，这里只有`AutowiredAnnotationBeanPostProcessor`和`CommonAnnotationBeanPostProcessor`符合）
* 重新注册一个`BeanPostProcessor`，类型为`ApplicationListenerDetector`（用于自动检测实现了`ApplicationListener`接口的Bean），注释上说这样操作后这个`BeanPostProcessor`就会被移动到processor调用链的后面

#### 7. initMessageSource
这个方法在`AbstractApplicationContext`类里面
```java
	/**
	 * Initialize the MessageSource.
	 * Use parent's if none defined in this context.
	 */
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
		}
	}
```
主要用来初始化`MessageSource`，`MessageSource`是用来支持国际化（多语言）的，里面的getMessage方法可以出解析不同地区支持的语言。但是好像并没有看到在什么地方有注册这种类型的Bean，所以应该是要自己手动添加的。它先在当前`BeanFactory`查找 名称为"messagegSource"的Bean(只在当前的`BeanFactory`查找，找不到也不会往父`BeanFactory`查找)，当然，在这个例子中是找不到的，因为没有配置这样的Bean。

我们假设已经配置了这样的Bean，然后找到了这样的Bean，接着会检查当前的`ApplicatinoContext`是不是有父`ApplicatinoContext`并且找到的`MessaegSource`为`HierarchicalMessageSource`类型，如果是，检查`MessaegSource`的父级`MessaegSource`是不是为空，如果是，则讲个它自身设置为父级`MessaegSource`。

如果没有找到这样的Bean，则会创建一个类型为`DelegatingMessageSource`的`MessaegSource`，然后设置父级`MessaegSource`为getInternalParentMessageSource获取到的`MessaegSource`，在本例中应该是null，接着将刚刚创建的`MessaegSource`注册为单例Bean，名称为"messaggeSource"


#### 8. initApplicationEventMulticaster
这个方法在`AbstractApplicationContext`类里面
```java
	/**
	 * Initialize the ApplicationEventMulticaster.
	 * Uses SimpleApplicationEventMulticaster if none defined in the context.
	 * @see org.springframework.context.event.SimpleApplicationEventMulticaster
	 */
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
						"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
			}
		}
	}
```
主要用来初始化`ApplicationEventMulticaster`，`ApplicationEventMulticaster`是用来管理`ApplicationListener`的（里面有addApplicationListener，removeApplicationListener等方法），能够向`ApplicationListener`发布事件（调用multicastEvent方法），`ApplicationEventPublisher`（一般是`ApplicationContext`，因为它继承了`ApplicationEventPublisher`）通常使用`ApplicationEventMulticaster`作为代理发布事件（意思是直接调用`ApplicationEventMulticaster`的方法发布事件，不自己实现发布事件的逻辑）。

首先该方法在当前的`BeanFactory`中查找名称为"applicationEventMulticaster"的`ApplicationEventMulticaster`（只在当前级别的`BeanFactory`查找，不向父级查找），当然，是本例中，找不到的，因为默认没有配置。

如果找到了，这将这个Bean赋值给applicationEventMulticaster变量。

如果没找到，则创建一个`SimpleApplicationEventMulticaster`类型的`ApplicationEventMulticaster`赋值给applicationEventMulticaster变量，然后注册为单例Bean，名称为"applicationEventMulticaster"

#### 9. onRefresh
这个方法在`AbstractApplicationContext`类里面
```java
	protected void onRefresh() throws BeansException {
		// For subclasses: do nothing by default.
	}
```
目前不进行任何操作

10. registerListeners
这个方法在`AbstractApplicationContext`类里面
```java
	protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```
该方法会注册`ApplicationListener`（包括配置了Bean的和没有配置的，没有配置的就是注释里面说的statically specified listeners）。

* 首先将调用`ApplicationEventMulticaster`（实际的类型为`SimpleApplicationEventMulticaster`，其实就是上面的initApplicationEventMulticaster方法刚刚创建的那个）的addApplicationListener方法（这个方法实际上是在`AbstractApplicationEventMulticaster`实现的）把statically specified listeners（实际上就是`AbstractApplicationContext`的applicationListeners变量的值）添加进去

* 然后和上面类似，把所有`ApplicationListener`类型的Bean添加进去（不一样的是这里是调用addApplicationListenerBean方法）

* 将earlyApplicationEvents变量里面存储的`ApplicationEvent`通过`SimpleApplicationEventMulticaster`的multicastEvent方法发布出去


#### 11. finishBeanFactoryInitialization
这个方法在`AbstractApplicationContext`类里面
```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
	}
```

这个方法会将完成当前`ApplicationContext`的`BeanFactory`的初始化，把所有非延迟加载的单例singleton beans实例化。我觉得这是12个方法里面最复杂的方法了，主要步骤如下：

* 查找名称为"conversionService"且类型为`ConversionService`的Bean，如果有，则调用`BeanFactory`的setConversionService进行设置(`ConversionService`是用来进行类型转换的)

* 如果没有找到默认的embedded value resolver（在本例中，它是用来解析注解的属性值的），则马上添加一个（添加的resolve其实是`PropertySourcesPropertyResolver`）

* 初始化所有实现了`LoadTimeWeaverAware`接口的Bean，注释上说尽快初始化这些Bean，以便他们能够尽快地注册transformers（不是很懂）

* 停用临时的类加载器

* 调用`BeanFactory`的freezeConfiguration来防止配置的变动，使得`BeanDefinition`元数据能够被缓存

* 调用`BeanFactory`的preInstantiateSingletons方法实例化所有的非延迟加载的singleton beans

重点看preInstantiateSingletons方法（在`DefaultListableBeanFactory`里面）
```java
	public void preInstantiateSingletons() throws BeansException {

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```
主要分为两步
* 初始化所有非延迟加载的singleton beans（创建Bean实例）
** 如果这个Bean是`FactoryBean`(特殊类型的Bean，用于创建其它复杂的Bean)，则把beanName加上`&`前缀（具体为什么要这样做后面会讲到），再调用getBean方法获取Bean实例，判断获取到的实例是不是`FactoryBean`类型，如果是，将它转换成`SmartFactoryBean`调用isEagerInit方法，如果返回true，就调用getBean方法（这次传入的参数是beanName，没有加`&`前缀）
** 如果这个Bean不是`FactoryBean`，直接调用getBean方法（如果Bean实例还没有创建的话，getBean方法会创建该实例）

* 调用所有实现了`SmartInitializingSingleton`接口的Bean的afterSingletonsInstantiated方法

接着来看gegtBean方法，getBean方法在`AbstractBeanFactory`里面
```java
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}
```
getBean方法里面调用了doGetBean方法，具体的实现逻辑应该就在doGetBean方法里面了（代码非常多，这里就不贴出来了），主要的逻辑如下：
* 调用transformedBeanName方法把传入的beanName开头的`&`字符全部去除
* 调用getSingleton方法先从手动注册的单例缓存里面获取实例
* 如果前面获取得到了实例，并且没有其他额外的参数传入，则调用getObjectForBeanInstance方法获取Bean实例，否则进行下面的操作
* parentBeanFactory不为空的话，就调用parentBeanFactory的getBean方法，传进去的beanName为原始的beanName，即未调用transformedBeanName方法前的beanName
* 如果parentBeanFactory为空，或者parentBeanFactory调用完getBean方法后依然没有获取到Bean实例，则使用当前的`BeanFactory`获取Bean实例
* 调用getMergedLocalBeanDefinition方法获取`RootBeanDefinition`，并调用checkMergedBeanDefinition方法检验
* 调用`RootBeanDefinition`的getDependsOn方法检测是否依赖于其它的Bean，如果是，则先注册并创建依赖的Bean的实例
* 根据Bean的Scope(作用域)，使用不同的逻辑创建Bean的实例
** 如果Scope为Singleton（Spring的Bean默认的Scope是Singleton），则调用getSingleton方法获取/创建Bean实例（如果Bean实例已经存在了，直接返回，否者就创建一个），获取的Bean实例可能是一个`BeanFactory`，所以还需要调用getObjectForBeanInstance方法，判断是不是`BeanFactory`，如果是，则调用`FactoryBean`的getObject方法获取真正的Bean实例（其实这里面的逻辑非常复杂繁琐，但是最终都会调用`FactoryBean`的getObject方法，当然前提是要获取不是这个`FactoryBean`，而是`FactoryBean`里面创建的Bean实例，即beanName没有`&`前缀）
** 如果Scope为Prototype，则每次都必定会创建一个新的Bean实例，具体是先调用createBean方法，再调用getObjectForBeanInstance方法获取真正要获取的Bean实例
** 如果Scope为其他类型，则调用`Scope`的get方法获取/创建Bean实例（具体取决于`Scope`的类型），再调用getObjectForBeanInstance方法获取真正要获取的Bean实例（创建Scope类型为非Singleton类型的Bean实例的前后会分别调用beforePrototypeCreation和afterPrototypeCreation方法）。
** 最后如果getBean方法指定了Bean实例的类型（参数里面的requiredType），则要进行相应的类型转换
** 返回Bean实例

再来看看createBean方法，可以看到创建Bean实例的时候都是调用的这个方法，这个方法的具体实现在`AbstractAutowireCapableBeanFactory`类中
```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isTraceEnabled()) {
			logger.trace("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			// 如果bean不为null，说明已经可能已经创建了一个代理类实例，不需要往下执行了，直接返回该实例
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			// 如果上面resolveBeforeInstantiation方法返回null，则执行doCreateBean方法创建Bean实例
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```
主要步骤如下：
* 调用resolveBeanClass方法根据`RootBeanDefinition`和beanName解析出要创建的Bean的实例对应的类
* 把`RootBeanDefinition`复制一份，因为后面`RootBeanDefinition`可能会被修改，而后面还有用到原`RootBeanDefinition`的信息
* 调用`RootBeanDefinition`（实际上是`AbstractBeanDefinition`）的prepareMethodOverrides方法，配置好要重写的方法
* 调用resolveBeforeInstantiation方法，这会调用所有实现了`InstantiationAwareBeanPostProcessor`接口（该接口为`BeanPostProcessor`的子接口）的`BeanPostProcessor`，通常用于创建一个代理类实例，而非Bean实例，如果返回`null`，则继续执行下面的doCreateBean方法，否者将直接返回实例
* 如果上面的resolveBeforeInstantiation返回`null`，则调用doCreateBean方法执行具体的创建Bean实例的逻辑

具体的创建Bean实例的逻辑在doCreateBean方法里面
```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
        // 省略了catch块里面的代码
		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
		}

		return exposedObject;
	}
```
主要步骤如下：
* 调用createBeanInstance方法创建`BeanWrapper`实例，具体创建步骤如下：
** 如果Bean配置了使用工厂方法创建Bean实例，则使用工厂方法创建
** 判断构造器或者工厂方法是否已经被解析，如果是，且构造器的参数也已经被解析，调用相应的构造器注入依赖并初始化Bean实例，如果构造器的参数没有被解析，则调用无参构造器实例化Bean
** 这里开始解析构造器或者工厂方法，判断是否有多个构造参数满足条件，这里将调用determineConstructorsFromBeanPostProcessors方法决定使用哪一个构造器
** 如果无法决定使用哪一个构造器，则查看是否设置了要优先使用的构造器，如果有，则使用该构造器创建Bean实例
** 上面的各种条件都不满足的话，就使用无参构造器创建Bean实例，将调用instantiateBean方法
```java
	protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
		try {
			Object beanInstance;
			final BeanFactory parent = this;
			if (System.getSecurityManager() != null) {
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						getInstantiationStrategy().instantiate(mbd, beanName, parent),
						getAccessControlContext());
			}
			else {
				// 这里的getInstantiationStrategy为CglibSubclassingInstantiationStrategy
				// 但是调用的instantiate方法是SimpleInstantiationStrategy的
				// 一般最后会调用BeanUtils的instantiateClass方法，使用JDK的反射机制创建实例
				// 或者调用的CglibSubclassingInstantiationStrategy的instantiateWithMethodInjection方法，
				// 这样会使用Cglib来创建实例
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
			}
			BeanWrapper bw = new BeanWrapperImpl(beanInstance);
			initBeanWrapper(bw);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
		}
	}

```

* 上面创建了Bean实例，而且可能已经通过构造器注入了一些必要的依赖。接下来会调用applyMergedBeanDefinitionPostProcessors方法，这会使得所有的类型为`MergedBeanDefinitionPostProcessor`的postProcessor被调用
* 判断当前创建的Bean实例的能否被提前缓存起来，以解决循环引用的问题
* 调用populateBean方法根据`RootBeanDefinition`填充Bean实例的属性
* 调用initializeBean方法初始化Bean实例，这会调用各种`*Aware`方法（比如BeanNameAware, BeanFactoryAware），`BeanPostProcessor`方法，`InitializingBean`方法等，具体的[顺序](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/BeanFactory.html)如下：
```
1. BeanNameAware's setBeanName
2. BeanClassLoaderAware's setBeanClassLoader
3. BeanFactoryAware's setBeanFactory
4. EnvironmentAware's setEnvironment
5. EmbeddedValueResolverAware's setEmbeddedValueResolver
6. ResourceLoaderAware's setResourceLoader (only applicable when running in an application context)
7. ApplicationEventPublisherAware's setApplicationEventPublisher (only applicable when running in an application context)
8. MessageSourceAware's setMessageSource (only applicable when running in an application context)
9. ApplicationContextAware's setApplicationContext (only applicable when running in an application context)
10. ServletContextAware's setServletContext (only applicable when running in a web application context)
11. postProcessBeforeInitialization methods of BeanPostProcessors
12. InitializingBean's afterPropertiesSet
13. a custom init-method definition
14. postProcessAfterInitialization methods of BeanPostProcessors
```

* `if (earlySingletonExposure)`这一步里面代码的暂时还看不懂
* 调用registerDisposableBeanIfNecessary方法，注册实现了`DestructionAwareBeanPostProcessor`和`DisposableBean`接口或者自定义了destroy方法的Bean
* 返回创建的Bean实例

#### 12. finishRefresh
这个方法在`AbstractApplicationContext`类里面
```java
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```
主要用来发布事件，同时也做了一些其它的事情：
1. 调用clearResourceCaches方法（`this.resourceCaches.clear()`），注释上说是用来清楚资源缓存的，比如ASM扫描产生的元数据

2. 调用initLifecycleProcessor方法，这是用来初始化`LifecycleProcessor`的，`LifecycleProcessor`用来检测所有实现了`LifeCycle`接口的Bean，然后使得`LifeCycle`里面的方法能够在相应的阶段被执行。
```java
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isTraceEnabled()) {
				logger.trace("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isTraceEnabled()) {
				logger.trace("No '" + LIFECYCLE_PROCESSOR_BEAN_NAME + "' bean, using " +
						"[" + this.lifecycleProcessor.getClass().getSimpleName() + "]");
			}
		}
	}
```
这个方法和之前分析的initMessageSource、initApplicationEventMulticaster等方法都是类似的，首先该会在当前的`ApplicationContext`查找名称为lifecycleProcessor的Bean，一般没有手动配置的话是找不到的，所以会创建一个`DefaultLifecycleProcessor`类型的lifecycleProcessor（在spring context包下只找到了这一个具体实现类），这个类实现了`BeanFactoryAware`接口，因此能够被注入`BeanFactory`，然后使用`BeanFactory`（实际上是`ListableBeanFactory`）的getBeanNamesForType方法获取所有实现了`LifeCycle`接口的Bean的名称，然后再用这些名称获取到相应的Bean
```java
	protected Map<String, Lifecycle> getLifecycleBeans() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		Map<String, Lifecycle> beans = new LinkedHashMap<>();
		// 调用ListableBeanFactory的getBeanNamesForType获取实现了LifeCycle接口的所有beanName
		String[] beanNames = beanFactory.getBeanNamesForType(Lifecycle.class, false, false);
		for (String beanName : beanNames) {
			String beanNameToRegister = BeanFactoryUtils.transformedBeanName(beanName);
			// 有可能是一个FactoryBean，如果是，则在beanName前面加上&前缀
			boolean isFactoryBean = beanFactory.isFactoryBean(beanNameToRegister);
			String beanNameToCheck = (isFactoryBean ? BeanFactory.FACTORY_BEAN_PREFIX + beanName : beanName);
			if ((beanFactory.containsSingleton(beanNameToRegister) &&
					(!isFactoryBean || matchesBeanType(Lifecycle.class, beanNameToCheck, beanFactory))) ||
					matchesBeanType(SmartLifecycle.class, beanNameToCheck, beanFactory)) {
				Object bean = beanFactory.getBean(beanNameToCheck);
				if (bean != this && bean instanceof Lifecycle) {
					beans.put(beanNameToRegister, (Lifecycle) bean);
				}
			}
		}
		return beans;
	}
```
为什么如果Bean是`FactoryBean`类型的时候，要在beanName前面加上`&`前缀然后再调用getBean方法呢？
这是因为，当使用getBean(String name)方法去获取Bean实例的时候，如果要获取的Bean是`FactoryBean`类型的，不加`&`前缀调用getBean方法，将得到这个`FactoryBean`的getObject方法返回的实例，如果要获取这个`FactoryBean`的Bean实例，需要在加上`&`前缀才行，[这里](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/org/springframework/beans/factory/BeanFactory.html#FACTORY_BEAN_PREFIX)有讲到

3. 调用`LifecycleProcessor`的onRefresh方法，在这个例子中就是`DefaultLifecycleProcessor`的onRefresh方法
```java
	@Override
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```
* 获取所有实现了`LifeCycle`接口的Bean
* 继续筛选出所有支持自启动的Bean，然后获取它们的阶段值，根据它们的阶段值分组
* 如果上面筛选出的分组不为空，则对分组进行排序，然后按顺序调用每一组（`LifecycleGroup`）的start方法
* 设置running变量为true

4. 发布一个`ContextRefreshedEvent`类型的事件（表示`ApplicationContext`已经被初始化和刷新完成）。实际上会调用`ApplicationEventMulticaster`的multicastEvent方法来发送事件，如果有父`ApplicationContext`的话，还好调用父`ApplicationContext`的publishEvent方法来发送事件。

5. 调用`LiveBeansView`的registerApplicationContext方法，这个方法做的事情有点抽象

* 从`Environment`中获取一个名称为"spring.liveBeansView.mbeanDomain"的属性的值，如果有这个属性，则进行下面的步骤，否则什么也不做
* 判断applicationContexts集合里面是不是有元素，如过有元素，则直接进行下面的步骤，如果没有元素
** 调用`ManagementFactory`的getPlatformMBeanServer方法获取`MBeanServer`实例
** 将`LiveBeansView`的applicationName变量赋值为当前的`ApplicationContext`的applicationName
** 调用`MBeanServer`实例的registerMBean方法，
* 把当前的`ApplicationContext`加到`LiveBeansView`的applicationContexts集合中（`Set<ConfigurableApplicationContext>`类型）
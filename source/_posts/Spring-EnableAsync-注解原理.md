---
title: Spring @EnableAsync 注解原理
date: 2018-12-29 16:31:47
tags:
    - Spring
---

`@EnableAsync`注解是用来开启Spring的异步功能的，一般在方法上加上`@Async`注解，就可以让这个方法变成一个异步方法（其实就是用线程池的其中一个线程来运行这个方法），前提是要使用`@EnableAsync`注解开启Spring的异步功能。Spring的异步功能使用起来非常简单，但是这个`@EnableAsync`究竟是怎样工作的呢？一般情况下都是这些`@EnableXXX`注解里面有一个`@Import`注解，然后这个`@Import`注解会导入一个类，这个被导入的类一般是个配置类（类上面有`@Configuration`注解），然后里面配置了需要用到的类的Bean，比如`@EnableWebMvc`
```java
@Retention(value=RUNTIME)
@Target(value=TYPE)
@Documented
@Import(value=DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc{}
```
`@EnableWebMvc`使用`@Import`导入了`DelegatingWebMvcConfiguration`类，该类是一个配置类，里面配置了需要用到的类的Bean（包括它自身和它的父类`WebMvcConfigurationSupport`配置的），这个配置类能够被`ConfigurationClassPostProcessor`这个`BeanFactoryProcessor`解析处理，然后把解析到Bean注册到Spring IoC容器。
```java
@Configuration
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport{}
```

## 使用例子
首先来看看`@EnableAsync`注解怎样使用，下面将列举一个简单的使用例子。

`MyService.java`
```java
@Service
public class MyService {

    @Async
    public void asyncFunction(){
        for(;;){
            System.out.println("异步方法执行中...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                break;
            }
        }
    }
}
```
在`MyService`类上面，有一个`@Service`注解，你也可以使用其它的注解代替，比如`@Commponent`。该类的方法使用了`@Async`注解标注，表示这是一个异步方法，方法里面做的事情很简单，就是每隔1s输出一些内容，可以看到这是一个没有终止条件的循环，目的是为了观察这个方法是不是真的异步执行了，因为如果该方法是异步执行的话，那程序应该能继续执行后面的代码，如果没有异步执行，而是直接在主线程里面执行的话，程序将会一直在执行这个循环里面的代码，无法执行后面的代码。

`TestAsync.java`
```java
@Configuration
@ComponentScan
@EnableAsync
public class TestAsync {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(TestAsync.class);
        MyService myService = ctx.getBean(MyService.class);
        myService.asyncFunction();
        System.out.println("上面的方法是异步方法吗？");
    }
}
```
`TestAsync`类用来存放main方法，还用来进行一些配置，在这个方法里面创建`ApplicationContext`，并使用它获取`MyService`类的Bean实例，接着使用这个实例调用它的asyncFunction方法，如果这个方法异步执行了，那么`System.out.println("上面的方法是异步方法吗？")`这句代码应该能顺利执行，否则这句代码将用于不会被执行。运行main方法，程序的执行结果如下：
![Imgur](https://i.imgur.com/u0f4l4T.png)

可以看到`System.out.println("上面的方法是异步方法吗？")`这句顺利执行了，说明asyncFunction方法是异步执行的。

## 异步方法主要原理
Spring的异步功能的主要是用Spring AOP和线程池实现的，简单地说，`ctx.getBean(MyService.class)`这里返回实例其实不是`MyService`类的实例，而是一个代理/增强类（`MyService`类的子类）的实例，然后调用asyncFunction方法的时候实际上调用的是这个代理/增强类的实例的asyncFunction方法，代理/增强类的asyncFunction方法会使用把方法交给线程池执行，线程池会使用新的线程来执行这个方法，因此主线程可以继续执行后面的代码，而不会卡在asyncFunction方法的死循环。

在`myService.asyncFunction()`这句上面设置一个断点，执行到这里的时候可以看到myService变量指向的不是`MyService`实例，而是Cglib动态创建的类的实例（这个类是`MyService`的子类，父类引用可以指向子类，因此myService能指向它）
![Imgur](https://i.imgur.com/2cPJ9Fh.png)

具体的就不再介绍了，对Spring AOP和线程池有一定的了解再去看源码大概就能看懂了，具体代码主要在spring-context模块下，还有部分AOP的代码在spring-aop模块下

### Spring线程池
Spring有自己实现的线程池，具体代码在spring-context模块的org.springframework.schedule.concurrent包，还有spring-core模块的or.springframework.core.task包下。当然也可以直接使用JDK实现的线程池。

Spring实现的线程池是基于`TaskExecutor`接口的，这个接口继承了JDK的`Executor`接口，至于为什么要有这个接口，文档上的解释是
>This interface remains separate from the standard Executor interface mainly for backwards compatibility with JDK 1.4 in Spring 2.x.

```java
@FunctionalInterface
public interface TaskExecutor extends Executor {

	/**
	 * Execute the given {@code task}.
	 * <p>The call might return immediately if the implementation uses
	 * an asynchronous execution strategy, or might block in the case
	 * of synchronous execution.
	 * @param task the {@code Runnable} to execute (never {@code null})
	 * @throws TaskRejectedException if the given task was not accepted
	 */
	@Override
	void execute(Runnable task);

}
```

如果使用`@EnableAsync`注解开启了Spring的异步功能，Spring会按照如下的方式查找相应的线程池用于执行异步方法：
1. 查找实现了`TaskExecutor`接口的Bean实例
2. 如果上面没有找到，则查找名称为`taskExecutor`并且实现了`Executor`接口的Bean实例
3. 如果还是没有找到，则使用`SimpleAsyncTaskExecutor`，该实现每次都会创建一个新的线程执行任务

上面的1,2步的Bean实例都是需要自己配置的，可以使用Spring实现的线程池，使用Spring实现的线程池不需要配置Bean的名称，也可以使用JDK实现的线程池，但是需要配置Bean名称为`taskExecutor`

## @EnableAsync注解原理
`@EnableAsync`注解里面使用`@Import`注解导入了`AsyncConfigurationSelector`类，该类实现了`ImportSelector`接口，后面在解析`@Configuration`配置类，处理`@Import`注解那一步的时候，会实例化`AsyncConfigurationSelector`，接着最终会调用到`AsyncConfigurationSelector`实现的selectImports方法，这个方法可能返回`ProxyAsyncConfiguration`类的全限定名或者`org.springframework.scheduling.aspectj.AspectJAsyncConfiguration`或者`null`，如果不是`null`的话，后面会解析这些类上面的注解（两个类都是配置类，即类上面有`@Configuration注解`），然后将读取到的`BeanDefinitino`注册到`BeanFactory`。

### 解析`@Import`注解
在上面给出的例子中，会在调用`AbstractApplicationContext`的invokeBeanFactoryPostProcessors方法这一步的时候解析`@Import`注解，具体是调用`ConfigurationClassPostProcessor`的postProcessBeanDefinitionRegistry方法时候，这个方法会使用`ConfigurationClassParser`的processConfigurationClass方法解析配置类（有`Configuration`注解的类），具体的解析步骤如下：
1. 递归地解析内部类
2. 解析`@PropertySource`注解
3. 解析`@ComponentScan`注解
4. 解析`@Import`注解
5. 解析`@ImportResource`注解
6. 解析`@Bean`注解，这个注解在类方法上
7. 解析类实现的接口上的默认方法（不是很懂这一步，推测是因为接口上的默认方法可能是`@Bean`方法）
8. 如果有父类，则直接返回父类，这样会继续回到第1步，然后开始解析父类。返回`null`，表示解析已经完成

因为`@EnableAsync`里面起到关键作用的是`@Import`注解，所以重点看第4步，解析`@Import`注解。

将会调用`ConfigurationClassParser`的processImports方法解析`@Import注解`
```java
    private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
            Collection<SourceClass> importCandidates, boolean checkForCircularImports) {

        // importCandidates是使用@Import注解导入的类，也就是该注解的值
        // 在本例中，值为AsyncConfigurationSelector
        if (importCandidates.isEmpty()) {
            return;
        }

        if (checkForCircularImports && isChainedImportOnStack(configClass)) {
            this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
        }
        else {
            this.importStack.push(configClass);
            try {
                for (SourceClass candidate : importCandidates) {
                    // 检测有没有实现ImportSelector接口
                    if (candidate.isAssignable(ImportSelector.class)) {
                        // Candidate class is an ImportSelector -> delegate to it to determine imports
                        Class<?> candidateClass = candidate.loadClass();
                        // 实例化candidateClass，即实例化@Import导入的类
                        ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                        // 调用各种Aware接口方法，比如BeanNameAare，BeanFactoryAware
                        ParserStrategyUtils.invokeAwareMethods(
                                selector, this.environment, this.resourceLoader, this.registry);
                        // 如果实现了DeferredImportSelector接口 ，则进行相应的处理
                        if (selector instanceof DeferredImportSelector) {
                            this.deferredImportSelectorHandler.handle(
                                    configClass, (DeferredImportSelector) selector);
                        }
                        else {
                        // 否则直接调用`ImportSelector`的selectImports方法
                            String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                            // 将上面选出的importClassNames用来创建SouceClass实例
                            Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                            // 递归处理创建的SourceClass实例，使得ImportSelector导入的类能够被处理
                            processImports(configClass, currentSourceClass, importSourceClasses, false);
                        }
                    }
                    // 检测有没有实现ImportBeanDefinitionRegistrar接口
                    else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                        // Candidate class is an ImportBeanDefinitionRegistrar ->
                        // delegate to it to register additional bean definitions
                        Class<?> candidateClass = candidate.loadClass();
                        // 实例化candidateClass，它实现了ImportBeanDefinitionRegistrar接口
                        ImportBeanDefinitionRegistrar registrar =
                                BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                        // 调用各种Aware接口方法        
                        ParserStrategyUtils.invokeAwareMethods(
                                registrar, this.environment, this.resourceLoader, this.registry);
                        // 调用ImportBeanDefinitionRegistrar的addImportBeanDefinitionRegistrar方法
                        configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                    }
                    // 如果没有实现上面的两个接口，则当成普通的配置类进行处理
                    else {
                        // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                        // process it as an @Configuration class
                        this.importStack.registerImport(
                                currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                        // 这里使得后面新创建的configClass能够被当成配置类处理
                        processConfigurationClass(candidate.asConfigClass(configClass));
                    }
                }
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to process import candidates for configuration class [" +
                        configClass.getMetadata().getClassName() + "]", ex);
            }
            finally {
                this.importStack.pop();
            }
        }
    }
```
processImports方法的主要步骤如下：
* 检查candidateClass有没有实现`ImportSelector`接口
-- 实例化candidateClass
-- 调用candidateClass实现的各种Aware接口的方法
-- 如果candidateClass实现了`DeferredImportSelector`，则使用deferredImportSelectorHandler进行相应的处理，否则调用candidateClass实现的`ImportSelector`接口的selectImports方法，该方法会返回需要导入的具体类的全限定名称，可能会返回多个类，一般是配置类，然后将这些类封装成SourceClass，然后调用processImports递归地进行处理
* 检查candidateClass有没有实现`ImportBeanDefinitionRegistrar`
-- 实例化candidateClass
-- 调用candidateClass实现的各种Aware接口的方法
-- 将实例化的candidateClass添加到当前的配置类（`ConfigurationClass`）的importBeanDefinitionRegistrars里面
* candidateClass没有实现上面的接口，则把candidateClass当成配置类处理(和处理被`@Configuration`注解标注的类一样的步骤)

`@EnableAsync`注解里面的`@Import`导入的类为`AsyncConfigurationSelector`，在上面的使用案例中，它的selectImports方法最终会返回"org.springframework.scheduling.annotation.ProxyAsyncConfiguration"，调用asSourceClasses能根据类的全限定名创建其对应的SourceClass，得到SourceClass后就能调用processConfigurationClass方法将其当成配置类进行处理，processConfigurationClass的主要步骤在前面有提到。

## ProxyAsyncConfiguration
`ProxyAsyncConfiguration`是`AsyncConfigurationSelector`的selectImports方法返回的配置类，它被一个`@Configuration`注解标注了，因此会被当成一个配置类进行解析。该类配置了一个Bean
```java
    // 名称为"org.springframework.context.annotation.internalAsyncAnnotationProcessor"
    // Bean的类型为AsyncAnnotationBeanPostProcessor
    @Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
        Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
        AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
        bpp.configure(this.executor, this.exceptionHandler);
        Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
        if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
            bpp.setAsyncAnnotationType(customAsyncAnnotation);
        }
        bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
        bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
        return bpp;
    }
```
这个Bean的类形为`AsyncAnnotationBeanPostProcessor`，它实现了`BeanPostProcessor`接口，该接口是Spring的拓展点之一，它提供了一些回调方法，能够让你提供自定义实例化Bean的逻辑还有解析依赖的逻辑，具体介绍可以查看[文档](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-factory-extension-bpp)


## AsyncAnnotationBeanPostProcessor
它实现了`BeanPostProcessor`接口，它实现这个接口的作用是为了让被`@Async`注解标注的方法或者类的方法能够异步执行，通过添加相应的`AsyncAnnotationAdvisor`到代理上（创建的代理类）实现。除了`BeanPostProcessor`接口以外，它还实现了很多其它的接口，比如`BeanFactoryAware`接口，具体如下图：
![Imgur](https://i.imgur.com/S6JLb7U.png)

该类实现`BeanFactory`接口的一个很重要的目的就是设置Advisor，也就是设置`AsyncAnnotationAdvisor`
```java
    @Override
    public void setBeanFactory(BeanFactory beanFactory) {
        super.setBeanFactory(beanFactory);
        // 这里创建了一个Advisor实例，具体类型为AsyncAnnotationAdvisor
        AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
        if (this.asyncAnnotationType != null) {
            advisor.setAsyncAnnotationType(this.asyncAnnotationType);
        }
        advisor.setBeanFactory(beanFactory);
        // 设置Advisor为AsyncAnnotationAdvisor
        this.advisor = advisor;
    }
```

该类还可以配置要使用的`Executor`和`AsyncUncaughtExceptionHandler`，前者为具体要使用的线程池，后者是用来处理没有返回值的异步方法抛出的异常（因为被异步调用的方法抛出的异常不能被调用者捕捉到，所以只能用这个handler处理抛出的异常）
```java
    public void configure(
            @Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

        this.executor = executor;
        this.exceptionHandler = exceptionHandler;
    }
```

### postProcessAfterInitialization方法
`AsyncAnnotationBeanPostProcessor`的父类的父类`AbstractAdvisingBeanPostProcessor`实现了`BeanPostProcessor`的postProcessAfterInitialization方法，通过这个方法就可以在Bean实例化后创建一个该Bean的proxy(所以在上面的例子中，使用beanFactory获取`MyService`的Bean实例后得到的不是`MyService`的Bean实例，而是一个代理类proxy)，然后用这个proxy让相应的方法能够异步执行。
```java
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // 如果advisor为空或者bean为AopInfrastructureBean类型，则直接返回bean，不进行任何处理
        // advisor为空的情况下，没有Advice，也没有Pointcut，所以无法处理bean
        if (this.advisor == null || bean instanceof AopInfrastructureBean) {
            // Ignore AOP infrastructure such as scoped proxies.
            return bean;
        }
        // 如果bean为Advised类型，则往该Advised里面添加Advisor
        // ProxyFactory实现了Advised接口，ProxyFacrory是用来创建proxy的
        if (bean instanceof Advised) {
            Advised advised = (Advised) bean;
            // 因为这是一个Advised实例，所以可以认为ProxyFactory已经被创建好了
            // 接下来只要把Advisor添加到ProxyFactory上就可以了
            if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
                // Add our local Advisor to the existing proxy's Advisor chain...
                if (this.beforeExistingAdvisors) {
                    advised.addAdvisor(0, this.advisor);
                }
                else {
                    advised.addAdvisor(this.advisor);
                }
                return bean;
            }
        }
        // 检验该bean是否能被该processor的Advisor(具体类型为AsyncAnnotationAdvisor)增强
        // 这里实际上会调用子类的isEligible方法，然后子类再调用该类的isEligible方法和和其它的一些逻辑
        if (isEligible(bean, beanName)) {
            // 创建ProxyFactory，将会使用这个factory来创建proxy
            ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
            // 如果proxyTargetClass为false(默认为false)，则使用基于接口的动态代理
            // 如果发现没有相应的接口，则设置proxyTargetClass为true，变成基于类的动态代理(Cglib)
            if (!proxyFactory.isProxyTargetClass()) {
                // 因为proxyTargetClass为false，只能使用基于接口的动态代理，所以这里检查有没有相应的接口
                evaluateProxyInterfaces(bean.getClass(), proxyFactory);
            }
            // 把Advisor(具体类是AsyncAnnotationAdvisor)绑定到factory(创建proxy的时候会绑定到proxy)
            proxyFactory.addAdvisor(this.advisor);
            // 执行自定义的proxyFactory处理逻辑，customizeProxyFactory方法应该由子类重写
            customizeProxyFactory(proxyFactory);
            // 用factory创建proxy
            return proxyFactory.getProxy(getProxyClassLoader());
        }

        // No proxy needed.
        return bean;
    }
```

重点留意isEligible方法，`AsyncAnnotationBeanPostProcessor`就是使用isEligible方法来判断某个Bean实例是否应该被处理的，即是否要创建某个Bean实例的proxy。需要注意的是这里实际调用的是`AbstractBeanFactoryAwareAdvisingPostProcessor`的isEligible方法
```
    protected boolean isEligible(Object bean, String beanName) {
        return (!AutoProxyUtils.isOriginalInstance(beanName, bean.getClass()) &&
                super.isEligible(bean, beanName));
    }
```
它调用了`AutoProxyUtils`的isOriginalInstance方法，这是用来检测Bean的名称是不是以`.ORIGINAL`开头的，如果是的话，将不会为该Bean实例创建代理。它还调用了父类（`AbstractAdvisingBeanPostProcessor`）的isEligible方法：
```java
    protected boolean isEligible(Object bean, String beanName) {
        return isEligible(bean.getClass());
    }

    protected boolean isEligible(Class<?> targetClass) {
        Boolean eligible = this.eligibleBeans.get(targetClass);
        if (eligible != null) {
            return eligible;
        }
        if (this.advisor == null) {
            return false;
        }
        eligible = AopUtils.canApply(this.advisor, targetClass);
        this.eligibleBeans.put(targetClass, eligible);
        return eligible;
    }    
```

具体的判断逻辑就是，调用`AopUtils`的canApply方法，这个方法会用`Advisor`也就是`AsyncAnnotationAdvisor`来进行判断

```java
    public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
        if (advisor instanceof IntroductionAdvisor) {
            return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
        }
        // AsyncAnnotationAdvisor实现了PointcutAdvisor
        else if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pca = (PointcutAdvisor) advisor;
            return canApply(pca.getPointcut(), targetClass, hasIntroductions);
        }
        else {
            // It doesn't have a pointcut so we assume it applies.
            return true;
        }
    }
```
最终会使用`AsyncAnnotationAdvisor`里面的`Pointcut`（具体为多个`AnnotationMatchingPointcut`的并集，`ComposablePointcut`）来判断。

## AsyncAnnotationAdvisor
它是一个`Advisor`（AOP里面的术语），它的作用是让被`@Async`标注的方法或者类的所有方法能够被异步执行，即变成异步方法，它里面有一个`Pointcut`，这个`Pointcut`表示所有需要异步执行的（默认是方法或者类上面有`@Async`注解的方法）。还有一个`Advice`，表示要增强的功能，在这里就是使用`Executor`来执行异步方法。

### Pointcut
`AsyncAnnotationAdvisor`里面的`Pointcut`的构建过程如下：
```java
    protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
        ComposablePointcut result = null;
        // 一开始的asyncAnnotationType只有Async，如果classpath里面有javax.ejb.Asynchronous的话，
        // 也会加上它
        for (Class<? extends Annotation> asyncAnnotationType : asyncAnnotationTypes) {
            Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
            Pointcut mpc = new AnnotationMatchingPointcut(null, asyncAnnotationType, true);
            if (result == null) {
                result = new ComposablePointcut(cpc);
            }
            else {
                result.union(cpc);
            }
            result = result.union(mpc);
        }
        return (result != null ? result : Pointcut.TRUE);
    }
```
返回的`Pointcut`具体类型为`ComposablePointcut`或者`TruePointcut`（这个表示所有的连接点都是切点，即不进行筛选）。默认情况下返回的`Pointcut`能够筛选出方法上或者类上有`@Async`（如果classpath里面有javax.ejb.Asynchronous，也能筛选出有`Asynchronous`注解的）注解的连接点（Joinpoint）

### Advice
`AsyncAnnotationAdvisor`里面的`Advice`的构建过程如下：
```java
    protected Advice buildAdvice(
            @Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

        AnnotationAsyncExecutionInterceptor interceptor = new AnnotationAsyncExecutionInterceptor(null);
        // 把Executor和exceptionHandler绑定到Adcice，使得在Advice里面能够用到
        interceptor.configure(executor, exceptionHandler);
        return interceptor;
    }
```
返回的`Advice`具体类型为`AnnotationAsyncExecutionInterceptor`，它表示具体要增强的功能，所以使得方法能够异步执行的逻辑就在这个类里面了，所以接下来重点看这个类。

![Imgur](https://i.imgur.com/nkVtbl0.png)

`AnnotationAsyncExecutionInterceptor`只有一个getExecutorQualifier方法
```java
    @Override
    @Nullable
    protected String getExecutorQualifier(Method method) {
        // Maintainer's note: changes made here should also be made in
        // AnnotationAsyncExecutionAspect#getExecutorQualifier
        // 获取方法上的@Async注解
        Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
        // 如果方法上没有没有@Async注解，则到方法所属的类上查找该注解
        if (async == null) {
            async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
        }
        // 返回@Async注解上的值，这个值用于指定使用哪个Executor来执行方法
        return (async != null ? async.value() : null);
    }
```
该方法返回`@Async`注解的值，这个值为需要使用的`Executor`的Bean名称。这里并没有看到使用`Executor`执行方法的逻辑，所以在它的父类`AsyncExecutionInterceptor`上面看看能不能找到。

`AsyncExecutionInterceptor`上面有一个invoke方法
```java
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        // 获取要执行的方法所属的类
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
        // 获取要最具体的方法，比如invocation.getMethod返回的可能是接口上面的方法，
        // targetClass是实现了该接口的类，这时候应该返回targetClass上的方法
        Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
        // 查找桥接方法，桥接是为了解决类型擦除问题编译器自动添加的方法
        final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

        // 决定使用哪一个Executor执行方法，
        // 将会用到AnnotationAsyncExecutionInterceptor重写的getExecutorQualifier方法
        AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
        if (executor == null) {
            throw new IllegalStateException(
                    "No executor specified and no default executor set on AsyncExecutionInterceptor either");
        }
        // 创建一个Callable对象
        Callable<Object> task = () -> {
            try {
                // invocation实现了Joinpoint接口，因此它也是一个joinpoint
                // 调用proceed方法表示让该连接点的代码继续向后执行
                Object result = invocation.proceed();
                if (result instanceof Future) {
                    return ((Future<?>) result).get();
                }
            }
            catch (ExecutionException ex) {
                handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
            }
            catch (Throwable ex) {
                handleError(ex, userDeclaredMethod, invocation.getArguments());
            }
            return null;
        };
        // 将上面创建的Callable对象传给doSubmit方法，该方法将调用Executor的execute方法执行任务
        // 也有可能调用submit方法
        return doSubmit(task, executor, invocation.getMethod().getReturnType());
    
```
`AsyncExecutionInterceptor`的invoke方法的主要步骤如下：
1. 查找要异步执行的方法所属的类(targetClass)
2. 查找最具体的要异步执行的方法（specificMethod，即targetClass实现的方法，而不是targetClass实现的接口上面的方法）
3. 查找桥接方法，桥接方法是编译器自动添加的，是为了解决类型擦除导致的问题，如果没有找到桥接方法则直接返回原来的方法
4. 根据`@Async`注解的值决定使用哪一个`Executor`，如果没有值则使用默认的`Executor`，如果没有自己配置任何`Executor`的情况下，将使用`SimpleAsyncTaskExecutor`
5. 创建Callable对象，为匿名内部类，里面封装了方法调用的逻辑
6. 将上面创建的Callable对象传doSumbit方法，该方法会使用第4步获取到的`Executor`执行任务

上面的第4步的逻辑在`AsyncExecutionInterceptor`和其父类`AsyncExecutionAspectSupport`上，`AsyncExecutionInterceptor`重写了父类的getDefaultExecutor方法
```java
    protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
        // 首先调用父类的getDefaultExecutor方法获取Executor
        Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
        // 没有获取到则使用SimpleAsyncTaskExecutor
        return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
    }
```
`AsyncExecutionInterceptor`的getDefaultExecutor方法先调用父类的getDefaultExecutor方法获取`Executor`，父类的getDefaultExecutor方法如下：
```java
    protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
        if (beanFactory != null) {
            try {
                // Search for TaskExecutor bean... not plain Executor since that would
                // match with ScheduledExecutorService as well, which is unusable for
                // our purposes here. TaskExecutor is more clearly designed for it.
                // 从BeanFactory里面查找实现了TaskExecutor的Bean
                return beanFactory.getBean(TaskExecutor.class);
            }
            catch (NoUniqueBeanDefinitionException ex) {
                // 找到的Bean不止一个，则查找名称为taskExecutor且实现了Executor接口的Bean
                logger.debug("Could not find unique TaskExecutor bean", ex);
                try {
                    return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
                }
                catch (NoSuchBeanDefinitionException ex2) {
                    // ...
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
                // 没有找到TaskExecutor类型的Bean，则查找名称为taskExecutor且实现了Executor接口的Bean
                logger.debug("Could not find default TaskExecutor bean", ex);
                try {
                    return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
                }
                catch (NoSuchBeanDefinitionException ex2) {
                    logger.info("No task executor bean found for async processing: " +
                            "no bean of type TaskExecutor and no bean named 'taskExecutor' either");
                }
                // Giving up -> either using local default executor or none at all...
            }
        }
        // 没有找到就返回null，然后子类的getDefaultExecutor方法会返回SimpleAsyncTaskExecutor
        return null;
    }
```
该方法首先查找类型为`TaskExecutor`的Bean，没有找到或者找到了不止一个这样的Bean则查找名称为taskExecutor且类型为`Executor`的Bean，找不到则返回`null`，如果返回null，`AsyncExecutionInterceptor`的getDefaultExecutor方法最终会返回`SimpleAsyncTaskExecutor`。

`SimpleAsyncTaskExecutor`是一个`TaskExecutor`，它的策略是每一个任务都开启一个新的线程执行，所以不能够复用线程，而且创建的线程数量没有限制，容易造成资源枯竭，而且会使CPU频繁地切换线程，所以建议自己配置一个可以复用线程的有界的`Executor`


第7步的逻辑在`AsyncExecutionAspectSupport`的doSubmit方法上
```java
    protected Object doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType) {
        if (CompletableFuture.class.isAssignableFrom(returnType)) {
            return CompletableFuture.supplyAsync(() -> {
                try {
                    return task.call();
                }
                catch (Throwable ex) {
                    throw new CompletionException(ex);
                }
            }, executor);
        }
        else if (ListenableFuture.class.isAssignableFrom(returnType)) {
            return ((AsyncListenableTaskExecutor) executor).submitListenable(task);
        }
        else if (Future.class.isAssignableFrom(returnType)) {
            return executor.submit(task);
        }
        else {
            executor.submit(task);
            return null;
        }
    }
```
该方法根据要调用的方法的返回值类型决定怎样执行任务。
* 当方法的返回值类型为`CompletableFuture`，则调用`CompletableFuture`的supplyAsync方法执行任务，返回值类型为`CompletableFuture`
* 当方法的返回值类型为`ListenableFuture`，则调用`AsyncListenableTaskExecutor`的submitListenable方法执行任务，返回值类型为`ListenableFuture`
* 当方法的返回值类型为`Future`，则调用`AsyncTaskExecutor`的submit方法执行任务，返回值类型为`Future`
* 当方法没有返回值，调用`AsyncTaskExecutor`的submit方法执行任务（其它类型的`Executor`最终也会被封装成`AsyncTaskExecutor`），返回`null`



---
title: SpringMVC DispatcherServlet的创建流程
date: 2018-12-21 10:42:44
tags:
    - SpringMVC
    - Web
---

在spring-mvc里面，一般所有的请求都会被匹配到`DispatcherServlet`，然后由它调用相应的`Interceptor`，`Hnadler`处理请求（`Filter`是在Servlet之前执行的），因此DispatcherServlet可以说是spring-mvc里面很核心的一个东西，下面我们来看看它是怎么被创建和初始化的还有如何注册到Servlet容器中的（因为我一般用纯注解的方式，因此只讲注解相关的）

## 创建和初始化
一般我们使用注解的方式来开发基于spring-mvc的web应用，首先要继承`AbstractAnnotationConfigDispatcherServletInitializer`，重写它的getRootConfigClasses和getServletConfigClasses和它的父类的`getServletMappings`方法，这几个方法的作用在另一篇博客有讲到。

`AbstractAnnotationConfigDispatcherServletInitializer`继承了`AbstractDispatcherServletInitializer`类，`AbstractDispatcherServletInitializer`类继承了`AbstractContextLoaderInitializer`类，`AbstractContextLoaderInitializer`类实现了`WebApplicationInitializer`接口
![Imgur](https://i.imgur.com/88A7itZ.png)


先来看`AbstractDispatcherServletInitializer`类，它重写了父类的onStartup方法
```
	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		super.onStartup(servletContext);
		registerDispatcherServlet(servletContext);
	}
```
在它重写的onStartup方法中，和父类相比，增加了注册DispatcherServlet的逻辑，即增加了registerDispatcherServlet方法
```
	protected void registerDispatcherServlet(ServletContext servletContext) {
		String servletName = getServletName();  // 默认返回dispatcher
		Assert.hasLength(servletName, "getServletName() must not return null or empty");

        // 这里创建servletAppContext，就是每个Servlet私有的WebApplicationContext
        // 可以通过我们重写的getRootConfigClasses方法来配置它
		WebApplicationContext servletAppContext = createServletApplicationContext();
		Assert.notNull(servletAppContext, "createServletApplicationContext() must not return null");

        // 这里创建DispatcherServlet的实例，FrameworkServlet是它的父类
		FrameworkServlet dispatcherServlet = createDispatcherServlet(servletAppContext);
		Assert.notNull(dispatcherServlet, "createDispatcherServlet(WebApplicationContext) must not return null");
		dispatcherServlet.setContextInitializers(getServletApplicationContextInitializers());

        // 动态注册Servlet，这里是注册刚刚创建的DiapatcherServlet到Servlet容器中
		ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
		if (registration == null) {
			throw new IllegalStateException("Failed to register servlet with name '" + servletName + "'. " +
					"Check if there is another servlet registered under the same name.");
		}

		registration.setLoadOnStartup(1);
		// 这里将调用getServletMappings方法设置DispatcherServlet要匹配的url
		// 我们可以在子类重写getServletMappings来自定义要匹配的url
		registration.addMapping(getServletMappings());
		registration.setAsyncSupported(isAsyncSupported());

        // 可以在子类重写getServletFilters方法来添加Filter
		Filter[] filters = getServletFilters();
		if (!ObjectUtils.isEmpty(filters)) {
			for (Filter filter : filters) {
				registerServletFilter(servletContext, filter);
			}
		}

        // 可以在子类重写这个方法来注册自定义的Servlet到Servlet容器中
		customizeRegistration(registration);
	}
```
registerDispatcherServlet方法将调用createServletApplicationContext方法来创建`WebApplicationContext `，这个方法是个抽象方法，它的子类`AbstractAnnotationConfigDispatcherServletInitializer`重写了这个方法（出现了！模板方法模式）
```
	@Override
	protected WebApplicationContext createServletApplicationContext() {
		AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
		// 将调用getServletConfigClasses来获取配置类
		// 这个方法需要子类重写
		Class<?>[] configClasses = getServletConfigClasses();
		if (!ObjectUtils.isEmpty(configClasses)) {
			context.register(configClasses);
		}
		return context;
	}
```
`AbstractAnnotationConfigDispatcherServletInitializer`的createServletApplicationContext方法调用了getServletConfigClasses方法，这个方法也是抽象方法，需要它的子类来实现（就是我们自己创建的类，继承了`AbstractAnnotationConfigDispatcherServletInitializer`，又出现了！模板方法模式）

这样一路看下来发现，getRootConfigClasses方法好像没被调用？这不是配置rootApplicationContext的吗？怎么没有被调用？？？其实已经在`AbstractAnnotationConfigDispatcherServletInitializer`的父类的父类`AbstractContextLoaderInitializer`里面调用了
```
    @Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		// 这个方法会调用到getRootConfigClasses方法获取rootWebApplicationContext的配置类
		registerContextLoaderListener(servletContext);
	}

		protected void registerContextLoaderListener(ServletContext servletContext) {
		// 这个createRootApplicationContext方法，也是抽象方法，它需要子类去实现（又双出现了！模板方法模式）
		// 然后它的子类的子类实现了这个方法，里面真正调用了getRootConfigClasses方法
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}

```

创建完servletAppContext之后（rootApplicationContext在之前已经创建完了，具体看onStartUp方法），就创建DispatcherServlet的实例了，将调用createDispatcherServlet方法进行创建
```
	protected FrameworkServlet createDispatcherServlet(WebApplicationContext servletAppContext) {
		return new DispatcherServlet(servletAppContext);
	}
```
这个方法直接用前面创建的servletAppContext作为构造参数创建`DispatcherServlet`的实例（创建完DispatcherServlet后还有注册Filter和注册自定义Servlet等步骤，这里不细讲），我们先来看看看`DispatcherServlet`的类图
![Imgur](https://i.imgur.com/onJ5QuS.png)
可以看到`DispatcherServlet`继承了`FrameworkServlet`，`FrameworkServlet`继承了`HttpServletBean`，`HttpServletBean`继承了`HttpServlet`，因此DispatcherServlet是一个Servlet。

`DispatcherServlet`的构造函数
```
	public DispatcherServlet() {
		super();
		setDispatchOptionsRequest(true);
	}
	public DispatcherServlet(WebApplicationContext webApplicationContext) {
		super(webApplicationContext);
		setDispatchOptionsRequest(true);
	}
```
上面创建`DispatcherServlet`的实例用的而第二个构造函数，这个构造函数调用了`FrameworkServlet`的构造函数
```
	public FrameworkServlet(WebApplicationContext webApplicationContext) {
		this.webApplicationContext = webApplicationContext;
	}
```
看起来好像就设置了一下webApplicationContext，这就完事了？是的，DispatcherServlet的实例已经创建完成了，不过好像还有很多事情没做，比如默认的视图解析器，主题解析器等都没有设置好，实例都创建完了，怎么还这些属性还没设置呢？其实已经设置好了。

？？？那么究竟是在什么时候设置的呢？一般都会觉得是在构造器里面有一个initXXX的方法会把这些东西设置好，然后这里并没有这样做，而是巧妙地利用了Servlet的生命周期进行这些属性设置（不要忘了DispatcherServlet也是一个Servlet），回想一下Servlet的生命周期，首先，创建Servlet的时候，Servlet容器会调用它的init方法，然后，移除Servlet的时候，会调用它的destroy方法，处理请求的时候，会调用它的service方法，其中init和destroy方法仅仅会被调用最多一次，service则会被调用任意次（每一处理请求都会调用）。

`FrameworkServlet`继承了`HttpServletBean`，`HttpServletBean`继承了`HttpServlet`。然后我们来看`HttpServletBean`的init方法
```
	@Override
	public final void init() throws ServletException {

		// Set bean properties from init parameters.
		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
		if (!pvs.isEmpty()) {
			try {
				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
				initBeanWrapper(bw);
				bw.setPropertyValues(pvs, true);
			}
			catch (BeansException ex) {
				if (logger.isErrorEnabled()) {
					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
				}
				throw ex;
			}
		}

		// Let subclasses do whatever initialization they like.
		initServletBean();
	}
```
这个方法在Servlet被容器创建的之后会（在Servlet开始处理请求之前必须要被执行）被执行。在最后面发现了一个可疑的initServletBean方法
```
	/**
	 * Subclasses may override this to perform custom initialization.
	 * All bean properties of this servlet will have been set before this
	 * method is invoked.
	 * <p>This default implementation is empty.
	 * @throws ServletException if subclass initialization fails
	 */
	protected void initServletBean() throws ServletException {
	}
```
竟然什么也没做，看了下方法上面的注释，发现是留给子类实现的，它子类是`FrameworkServlet`，好像快发现些什么了，赶紧去看`FrameworkServlet`的initServletBean方法
```
	@Override
	protected final void initServletBean() throws ServletException {
		getServletContext().log("Initializing Spring " + getClass().getSimpleName() + " '" + getServletName() + "'");
		if (logger.isInfoEnabled()) {
			logger.info("Initializing Servlet '" + getServletName() + "'");
		}
		long startTime = System.currentTimeMillis();

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException | RuntimeException ex) {
			logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (logger.isDebugEnabled()) {
			String value = this.enableLoggingRequestDetails ?
					"shown which may lead to unsafe logging of potentially sensitive data" :
					"masked to prevent unsafe logging of potentially sensitive data";
			logger.debug("enableLoggingRequestDetails='" + this.enableLoggingRequestDetails +
					"': request parameters and headers will be " + value);
		}

		if (logger.isInfoEnabled()) {
			logger.info("Completed initialization in " + (System.currentTimeMillis() - startTime) + " ms");
		}
	}
```
然后又发现两个可疑的方法，initWebApplicationContext和initFrameworkServlet，先来看initWebApplicationContext，从名字上看，是用来初始化WebApplicationContext的
```
	/**
	 * Initialize and publish the WebApplicationContext for this servlet.
	 * <p>Delegates to {@link #createWebApplicationContext} for actual creation
	 * of the context. Can be overridden in subclasses.
	 * @return the WebApplicationContext instance
	 * @see #FrameworkServlet(WebApplicationContext)
	 * @see #setContextClass
	 * @see #setContextConfigLocation
	 */
	protected WebApplicationContext initWebApplicationContext() {
		// 获取rootWebApplicationContext
		WebApplicationContext rootContext =
				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
		WebApplicationContext wac = null;

		if (this.webApplicationContext != null) {
			// A context instance was injected at construction time -> use it
			wac = this.webApplicationContext;
			if (wac instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent -> set
						// the root application context (if any; may be null) as the parent
						cwac.setParent(rootContext);
					}
					// refresh完成后Spring IOC容器创建的bean就可以使用了
					configureAndRefreshWebApplicationContext(cwac);
				}
			}
		}
		if (wac == null) {
			// No context instance was injected at construction time -> see if one
			// has been registered in the servlet context. If one exists, it is assumed
			// that the parent context (if any) has already been set and that the
			// user has performed any initialization such as setting the context id
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			onRefresh(wac);
		}

		if (this.publishContext) {
			// Publish the context as a servlet context attribute.
			String attrName = getServletContextAttributeName();
			getServletContext().setAttribute(attrName, wac);
		}

		return wac;
	}
```
重点看onRefresh方法
```
	/**
	 * Template method which can be overridden to add servlet-specific refresh work.
	 * Called after successful context refresh.
	 * <p>This implementation is empty.
	 * @param context the current WebApplicationContext
	 * @see #refresh()
	 */
	protected void onRefresh(ApplicationContext context) {
		// For subclasses: do nothing by default.
	}
```
看了一下注释，发现也是让子类实现的，也就是将会由`DispatcherServlet`实现，于是马上去看它的实现
```
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}
```
只调用了一个initStrategies方法，说明设置DispatcherServlet的各种默认视图解析器、主题解析器的逻辑应该就在这里了
```
	protected void initStrategies(ApplicationContext context) {
		initMultipartResolver(context);
		initLocaleResolver(context);
		initThemeResolver(context);
		initHandlerMappings(context);
		initHandlerAdapters(context);
		initHandlerExceptionResolvers(context);
		initRequestToViewNameTranslator(context);
		initViewResolvers(context);
		initFlashMapManager(context);
	}
```
果不其然，9个initXXX方法整齐地被调用了，其实这些initXXX方法方法仅仅是根据名称和类型获取Spring IOC容器里面的Bean，默认的视图解析器、主题解析器等Bean的创建逻辑在`DelegatingWebMvcConfiguration`和它的父类`WebMvcConfigurationSupport`里面，这里篇幅有限，就不再拓展了。


然后回到`FrameworkServlet`的initFrameworkServlet方法
```
	/**
	 * This method will be invoked after any bean properties have been set and
	 * the WebApplicationContext has been loaded. The default implementation is empty;
	 * subclasses may override this method to perform any initialization they require.
	 * @throws ServletException in case of an initialization exception
	 */
	protected void initFrameworkServlet() throws ServletException {
	}
```
又是什么也没有做，但是也没有子类实现这个方法，说明这个方法完全留给我们拓展的，比如我们要魔改`DispatcherServlet`的时候，就可以重写这个方法，然后添加自己的逻辑。

所以DispatcherServlet的创建流程大概就是这样子了。

## 注册到Servlet容器
这里的Servlet容器为Tomcat

用注解的方式注册Servlet到Servlet容器中，需要Servlet3.0+的版本，因为这是3.0版本的新特性，关键是`ServletContainerInitializer`接口，这个接口的文档可以点[这里](https://docs.oracle.com/javaee/7/api/javax/servlet/ServletContainerInitializer.html)查看
```
public interface ServletContainerInitializer {
    public void onStartup(Set<Class<?>> c, ServletContext ctx)
        throws ServletException; 
}

```
这个接口的具体功能是用来在Web应用启动的阶段向Servlet容器动态地注册Servlet, Filter或者listener。接口里面只有一个onStartup方法，里面有两个参数，第一个参数是是Set类型，这个参数可以存放`@HandlesTypes`指定的类的集合，如果没有这样的类，则为空，也可以存放是`ServletContainerInitializer`类型的类的集合，这个参数的具体作用将会在讲spring-web模块中实现了这个接口的类，即`SpringServletContainerInitializer`的时候仔细讲解。第二个参数是ServletContext类型的，用来动态注册Servlet, Filter或者listener。

要实现这个接口，需要在你的类文件根目录下创建一个META-INF/services目录，然后在这个目录下创建一个名为javax.servlet.ServletContainerInitializer的文件，里面的内容为你的实现类的全限定名。在spring-web模块中，具体的实现类为`SpringServletContainerInitializer`
![Imgur](https://i.imgur.com/UAf1oeQ.png)
![Imgur](https://i.imgur.com/u6sVuBe.png)

```
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

        // 用来存放WebApplicationInitializer类型实例
		List<WebApplicationInitializer> initializers = new LinkedList<>();

		if (webAppInitializerClasses != null) {
			// 遍历Servlet容器传过来的webAppInitializerClasses
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				// 筛选出类型为WebApplicationInitializer的具体类
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						// 使用反射动态创建WebApplicationInitializer实例
						initializers.add((WebApplicationInitializer)
								ReflectionUtils.accessibleConstructor(waiClass).newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		servletContext.log(initializers.size() + " Spring WebApplicationInitializers detected on classpath");
		AnnotationAwareOrderComparator.sort(initializers);
		// 调用所有WebApplicationInitializer类型的实例的onStartup方法
		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}

}

```
在`SpringServletContainerInitializer`中`@HandlesTypes`注解的值为WebApplicationInitializer.class，Servlet容器自动检测classpath中所有的类型为`WebApplicationInitializer`或者`ServletContainerInitializer`的类，然后将它们组合起来创建一个Set实例然后传给onStartup方法。

在onStartup方法中，只选择类型为`WebApplicationInitializer`的具体类，然后创建这些类的实例，接着调用这些类的onStartup方法。

`WebApplicationInitializer`在spring-web模块中，它的作用类似与Servlet的`ServletContainerInitializer`接口，都是用来配置`ServletContext`的
```
public interface WebApplicationInitializer {

	/**
	 * Configure the given {@link ServletContext} with any servlets, filters, listeners
	 * context-params and attributes necessary for initializing this web application. See
	 * examples {@linkplain WebApplicationInitializer above}.
	 * @param servletContext the {@code ServletContext} to initialize
	 * @throws ServletException if any call against the given {@code ServletContext}
	 * throws a {@code ServletException}
	 */
	void onStartup(ServletContext servletContext) throws ServletException;

}
```

`AbstractContextLoaderInitializer`类实现了这个接口，它是个抽象类
```
public abstract class AbstractContextLoaderInitializer implements WebApplicationInitializer {

	/** Logger available to subclasses. */
	protected final Log logger = LogFactory.getLog(getClass());

	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		registerContextLoaderListener(servletContext);
	}

	protected void registerContextLoaderListener(ServletContext servletContext) {
		WebApplicationContext rootAppContext = createRootApplicationContext();
		if (rootAppContext != null) {
			ContextLoaderListener listener = new ContextLoaderListener(rootAppContext);
			listener.setContextInitializers(getRootApplicationContextInitializers());
			servletContext.addListener(listener);
		}
		else {
			logger.debug("No ContextLoaderListener registered, as " +
					"createRootApplicationContext() did not return an application context");
		}
	}

	@Nullable
	protected abstract WebApplicationContext createRootApplicationContext();

	@Nullable
	protected ApplicationContextInitializer<?>[] getRootApplicationContextInitializers() {
		return null;
	}

}
```
在onStartup方法里面，调用了registerContextLoaderListener方法，这个方法首先创建了rootAppContext（所有Servlet共享的`WebApplicationContext`），然后这个rootAppContext作为参数用来创建ContextLoaderListener的实例，然后将这个实例添加到servletContext的listeners中。
`ContextLoaderListener`实现了`ServletContextListener`接口
```
public interface ServletContextListener extends EventListener {

    public void contextInitialized(ServletContextEvent sce);

    public void contextDestroyed(ServletContextEvent sce);
}

public class ContextLoaderListener extends ContextLoader implements ServletContextListener {


	public ContextLoaderListener() {
	}


	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}


	/**
	 * Initialize the root web application context.
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}


	/**
	 * Close the root web application context.
	 */
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}

}

```
contextInitialized将在filters 或者 servlets开始被初始化之前（web应用启动的时候）执行，contextDestroyed将在所有servlets 和 filters被销毁之后（web应用关闭的时候）执行。

contextInitialized方法调用了initWebApplicationContext方法，说明在在filters 或者 servlets开始被初始化之前将会初始化WebApplicationContext，因此在初始化filters 或者 servlets的时候WebApplicationContext是可用的。
```
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	    // 这时候rootWebApplicationContext应该还没有被初始化并绑定到servletContext属性中
	    // 因为这个方法是由AbstractContextLoaderInitializer的onStartup方法调用的，因此应该在最前面执行
	    // 而且rootWebApplicationContext只能有一个，不能配置多个
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		servletContext.log("Initializing Spring root WebApplicationContext");
		Log logger = LogFactory.getLog(ContextLoader.class);
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			// 这里在之前的创建ContextLoaderListener的实例的时候已经把rootWebApplicationContext传过来了
			// 所以这里的context是不为空的(如果是用的注解配置，而不是web.xml)
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
			// 这里将初始化rootWebApplicationContext
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			// 初始化之后绑定到servletContext的属性里面
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
			}

			return this.context;
		}
		catch (RuntimeException | Error ex) {
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
	}
```
可以看到这个方法将在创建Servlet和Filters这些东西之前把rootApplicationContext配置好，使得创建这些东西的时候可以使用rootApplicationContext。

## 总结
`SpringServletContainerInitializer`类实现了`ServletContainerInitializer`接口，使得Servlet容器能够使用这个类来自动检测我们的实现了`WebApplicationInitializer`接口的类，然后用反射创建这些类的实例，调用实例的onStartup方法，然后onStartup方法里面会初始化rootWebApplicationContext。

在spring-mvc中，我们通过继承的`AbstractAnnotationConfigDispatcherServletInitializer`类，实际上间接实现了`WebApplicationInitializer`接口（实际上是它的父类的父类实现的），因此能够被Servlet容器检测到，然后就和上面说的一样，Servlet检测到我们的`AbstractAnnotationConfigDispatcherServletInitializer`，然后通过反射创建它的实例，调用实例的onStartup方法。


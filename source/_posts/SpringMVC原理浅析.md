---
title: SpringMVC原理浅析
date: 2018-12-22 14:50:52
tags:
    - SpringMVC
    - Web
---

SpringMVC的原理是把`DispatcherServlet`作为中央Servlet来处理请求（一般情况下所有请求都会被匹配到DispatcherServlet，因为它的servletMapping一般都是设置为`/`，而且不会配置其它的有servletMapping的Servlet，jsp Servlet除外），`DispatcherServlet`根据请求路径找到相应的handler，然后使用HandlerAdapter调用handler进行处理，调用handler之前会先调用拦截器（这里指的是HandlerInterceptor，而不是Filter）的preHandler方法，然后才调用handler，接着解析视图，渲染视图，最后调用拦截器的afterCompletion方法。大概的流程如下图

![Imgur](https://i.imgur.com/6oRxOHv.png)

## DispatcherServlet的匹配规则
一般将`DispatcherServlet`的servletMapping设置成`/`，这会覆盖Tomcat配置`DefaultServlet`的servletMapping，`DefaultServlet`使用来将web根目录下的静态资源返回给客户端的，默认等等servletMapping是`/`，因此我们将`DispatcherServlet`的servletMapping也设置成`/`，会覆盖`DefaultServlet`的servletMapping（实际上就是将`DefaultServlet`的servletMapping移除），然后`DefaultServlet`就不能匹配到任何路径了，所以我们就无法将web根目录下的静态资源返回给客户端了。

如果我们还需要像`DefaultServlet`一样将web根目录下的静态资源返回给客户端，可以启用`DefaultServletHttpRequestHandler`这个handler，它匹配的路径为`/*`，它会把请求获取web目录下的静态资源的请求转发给`DefaultServlet`处理（`DefaultServlet`没有了servletMapping，但是我们依然可以通过`ServletContext`的getNamedDispatcher来根据名称获取到`DefaultServlet`的`RequestDispatcher`，然后将请求转发给它处理）。只需要实现`WebMvcConfigure`接口的configureDefaultServletHandling方法，然后调用方法的参数的enable方法，就可以启用`DefaultServletHttpRequestHandler`了
```java
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
```
效果如下：

![Imgur](https://i.imgur.com/n11g6ae.png)

m.html页面放在web应用根目录下，如果放在WEB-INF目录下，是不能被直接访问的

![Imgur](https://i.imgur.com/FleBKWU.png)

可以看到成功地访问到了m.html页面

## 将各种resolver和属性绑定到request
在`DispatcherServlet`将请求分发给handler处理之前，它会把各种resolver（LocaleResolve, ThemeResolver等）还有一些重要的属性绑定到request，以便在后续处理中(主要是让handler和view对象使用)能使用这些resolver和属性处理请求。具体可以看它的doService方法
```java
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
        // 为了节省空间删除了部分代码，因此这里的方法是不完整的，可以在源码中查看完整的方法
		// Make framework objects available to handlers and view objects.

		// 绑定WebApplicationContext到request
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
        // 绑定LocaleResolver到request
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		// 绑定ThemeResolver到request
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		// 绑定ThemeSource到request
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

		if (this.flashMapManager != null) {
			FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
			if (inputFlashMap != null) {
				request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
			}
			request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
			request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
		}

		try {
			doDispatch(request, response);
		}
        // 删除了后面的部分代码
	}

```

## 解析Multipart
如果这个请求是一个Multipart请求（某个请求头的值含有multipart字样），就使用`MultipartResolver`把请求转换成`MultipartHttpServletRequest`，具体过程可以看`DispatcherServlet`的checkMultipart方法

```java
	/**
	 * Convert the request into a multipart request, and make multipart resolver available.
	 * <p>If no multipart resolver is set, simply use the existing request.
	 * @param request current HTTP request
	 * @return the processed request (multipart wrapper if necessary)
	 * @see MultipartResolver#resolveMultipart
	 */
	protected HttpServletRequest checkMultipart(HttpServletRequest request) throws MultipartException {
		if (this.multipartResolver != null && this.multipartResolver.isMultipart(request)) {
			if (WebUtils.getNativeRequest(request, MultipartHttpServletRequest.class) != null) {
				if (request.getDispatcherType().equals(DispatcherType.REQUEST)) {
					logger.trace("Request already resolved to MultipartHttpServletRequest, e.g. by MultipartFilter");
				}
			}
			else if (hasMultipartException(request) ) {
				logger.debug("Multipart resolution previously failed for current request - " +
						"skipping re-resolution for undisturbed error rendering");
			}
			else {
				try {
					return this.multipartResolver.resolveMultipart(request);
				}
				catch (MultipartException ex) {
					if (request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) != null) {
						logger.debug("Multipart resolution failed for error dispatch", ex);
						// Keep processing error dispatch with regular request handle below
					}
					else {
						throw ex;
					}
				}
			}
		}
		// If not returned before: return original request.
		return request;
	}
```
如果没有配置`MultipartResolver`，就将请求原样返回，不作任何转换，这是因为真正的转换是调用`MultipartResolver`的resolveMultipart进行的，没有`MultipartResolver`，就无法转换了。默认情况下`MultipartResolver`是没有被配置的，因此不能处理Multipart请求，需要我们自己去配置。Spring模块中有两个具体的`MultipartResolver`类，分别为`CommonsMultipartResolver`和`StandardServletMultipartResolver`，一般我们会选择配置`CommonsMultipartResolver`，因为它配置比较简单，只需要在配置类中声明一个id为multipartResolver的Bean就行了（还要添加相应的依赖包，比如Apache Commons IO）。
```java
    @Bean
    MultipartResolver multipartResolver(){
        return new CommonsMultipartResolver();
    }
```

## 根据请求路径找到匹配的handler
将会调用`DispatcherServlet`的getHandler方法来寻找所以匹配的handler
```java
	/**
	 * Return the HandlerExecutionChain for this request.
	 * <p>Tries all handler mappings in order.
	 * @param request current HTTP request
	 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
	 */
	@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```
getHandler方法的返回类型为`HandlerExecutionChain`
```java
public class HandlerExecutionChain {

	private final Object handler;

	@Nullable
	private HandlerInterceptor[] interceptors;

	@Nullable
	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;
}
```
`HandlerExecutionChain`里面存放了一个handler，还有多个`HandlerInterceptor`，说明一个请求路径可能匹配最多一个handler，但是能匹配多个`HandlerInterceptor`。

getHandler方法具体是怎样将请求路径和对应的handler还有`HandlerExecutionChain`匹配起来的，这是一个重点，从`DispatcherServlet`的getHandler方法中可以看到是遍历handlerMappings变量，然后调用`HandlerMapping`实例的getHandler方法。那么handlerMappings变量里面都有哪些`HandlerMapping`实例呢？我们可以去看`DispatcherServlet`的initHandlerMappings方法
```java
	private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;

        // this.detectAllHandlerMappings的值默认为true
		if (this.detectAllHandlerMappings) {
			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
			// 将查找所有实现了HandlerMapping接口的Bean
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

		// Ensure we have at least one HandlerMapping, by registering
		// a default HandlerMapping if no other mappings are found.
		// 如果没有没有在IOC容器里面找到类型为HandlerMapping的Bean，则使用默认的HandlerMapping
		if (this.handlerMappings == null) {
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isTraceEnabled()) {
				logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
						"': using default strategies from DispatcherServlet.properties");
			}
		}
	}
```
需要注意的是，添加了@EnableWebMvc注解后，在`WebMvcConfigurationSupport`类中，会配置5个实现了`HandlerMapping`类型的Bean

第一个Bean为requestMappingHandlerMapping
```java
	/**
	 * Return a {@link RequestMappingHandlerMapping} ordered at 0 for mapping
	 * requests to annotated controllers.
	 */
    @Bean
	public RequestMappingHandlerMapping requestMappingHandlerMapping() {
		RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
		mapping.setOrder(0);
		......// 省略其他代码
	}
```

第二个Bean为viewControllerHandlerMapping
```java
	/**
	 * Return a handler mapping ordered at 1 to map URL paths directly to
	 * view names. To configure view controllers, override
	 * {@link #addViewControllers}.
	 */
    @Bean
	@Nullable
	public HandlerMapping viewControllerHandlerMapping() {
		ViewControllerRegistry registry = new ViewControllerRegistry(this.applicationContext);
		addViewControllers(registry);

		AbstractHandlerMapping handlerMapping = registry.buildHandlerMapping();
		......
	}
```

第三个Bean为beanNameHandlerMapping
```java
	/**
	 * Return a {@link BeanNameUrlHandlerMapping} ordered at 2 to map URL
	 * paths to controller bean names.
	 */
	@Bean
	public BeanNameUrlHandlerMapping beanNameHandlerMapping() {
		BeanNameUrlHandlerMapping mapping = new BeanNameUrlHandlerMapping();
		mapping.setOrder(2);
		mapping.setInterceptors(getInterceptors());
		mapping.setCorsConfigurations(getCorsConfigurations());
		return mapping;
	}
```

第四个Bean为resourceHandlerMapping
```java
	/**
	 * Return a handler mapping ordered at Integer.MAX_VALUE-1 with mapped
	 * resource handlers. To configure resource handling, override
	 * {@link #addResourceHandlers}.
	 */
	@Bean
	@Nullable
	public HandlerMapping resourceHandlerMapping() {
		ResourceHandlerRegistry registry = new ResourceHandlerRegistry(this.applicationContext,
				this.servletContext, mvcContentNegotiationManager(), mvcUrlPathHelper());
		addResourceHandlers(registry);

		AbstractHandlerMapping handlerMapping = registry.getHandlerMapping();
		......
	}
```

第五个Bean为defaultServletHandlerMapping
```java
	 /**
	 * Return a handler mapping ordered at Integer.MAX_VALUE with a mapped
	 * default servlet handler. To configure "default" Servlet handling,
	 * override {@link #configureDefaultServletHandling}.
	 */
	@Bean
	@Nullable
	public HandlerMapping defaultServletHandlerMapping() {
		DefaultServletHandlerConfigurer configurer = new DefaultServletHandlerConfigurer(this.servletContext);
		configureDefaultServletHandling(configurer);
		return configurer.buildHandlerMapping();
	}
```
从上面的代码无法直接看出第2,4,5Bean的具体类型，即viewControllerHandlerMapping，Bean为resourceHandlerMapping，defaultServletHandlerMapping，跟进去对应的方法里面应该可以看出来，但是可以直接DEBUG设置断点快速查看对应的类型

![Imgur](https://i.imgur.com/Uo8T18X.png)

从上面的图中可以看出来，是`SimpleUrlHandlerMapping`类型

其中，第一个Bean的Order的值设置成了0，因此优先级最高，所以默认先调用`RequestMappingHandlerMapping`的getHandler方法获取相应的handler，然而这个类里面并没有这个方法，说明是从父类继承的，最终在`AbstractHandlerMapping`类中发现了这个方法
```java
	public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		// 调用getHandlerInternal方法获取handler，这个方法需要子类实现
		Object handler = getHandlerInternal(request);
		if (handler == null) {
			// 没有获取到handler，则尝试获取默认的handler
			handler = getDefaultHandler();
		}
		if (handler == null) {
			return null;
		}
		// Bean name or resolved handler?
		if (handler instanceof String) {
			String handlerName = (String) handler;
			handler = obtainApplicationContext().getBean(handlerName);
		}
        // 获取HandlerExecutionChain，这里会找到所有匹配的拦截器
		HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
		...... // 删除了部分代码
		return executionChain;
	}
```
可以看到还需要调用getHandlerInternal方法获取handler
```java
protected abstract Object getHandlerInternal(HttpServletRequest request) throws Exception;
```
getHandlerInternal是抽象方法，说明是留给子类实现了，所以这里又使用了模板方法模式，`AbstractHandlerMapping`的直接子类`AbstractHandlerMethodMapping<T>`实现了这个方法
```java
	@Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		this.mappingRegistry.acquireReadLock();
		try {
			// 调用lookupHandlerMethod方法，使用上面的lookupPath找到对应的HandlerMethod
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}
```
getHandlerInternal大概的逻辑是，根据请求的路径，即lookupPath，调用lookupHandlerMethod找到对应的`HandlerMethod`，`HandlerMethod`又是什么东西？？？
```java
/**
 * Encapsulates information about a handler method consisting of a
 * {@linkplain #getMethod() method} and a {@linkplain #getBean() bean}.
 * Provides convenient access to method parameters, the method return value,
 * method annotations, etc.
 *
 * <p>The class may be created with a bean instance or with a bean name
 * (e.g. lazy-init bean, prototype bean). Use {@link #createWithResolvedBean()}
 * to obtain a {@code HandlerMethod} instance with a bean instance resolved
 * through the associated {@link BeanFactory}.
 */
public class HandlerMethod {

	private final Object bean;

	@Nullable
	private final BeanFactory beanFactory;

	private final Class<?> beanType;

	private final Method method;

	private final Method bridgedMethod;

	private final MethodParameter[] parameters;

	@Nullable
	private HttpStatus responseStatus;

	@Nullable
	private String responseStatusReason;

	@Nullable
	private HandlerMethod resolvedFromHandlerMethod;

	@Nullable
	private volatile List<Annotation[][]> interfaceParameterAnnotations;
	...... //主要存放了上面的几个成员变量
```
简单来说就是一个方法 + 方法所属对象的组合，举个例子
```java
@Controller
public class IndexController{
	@GetMaping
	public String index(){
		return "index.html";
	}
}
```
上面的index方法，将会是`HandlerMethod`里面的method变量的值，然后IndexController上面有@Controller注解，所以它是一个Bean，会被容器创建，Bean名成为"indexController"，
所以这个Bean将会是上面的`HandlerMethod`的bean变量的值

然后来看看lookupHandlerMethod方法
```java
	@Nullable
	protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
		List<Match> matches = new ArrayList<>();
		List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
		if (directPathMatches != null) {
			addMatchingMappings(directPathMatches, matches, request);
		}
		if (matches.isEmpty()) {
			// No choice but to go through all mappings...
			addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
		}

		if (!matches.isEmpty()) {
			Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
			matches.sort(comparator);
			Match bestMatch = matches.get(0);
			if (matches.size() > 1) {
				if (logger.isTraceEnabled()) {
					logger.trace(matches.size() + " matching mappings: " + matches);
				}
				if (CorsUtils.isPreFlightRequest(request)) {
					return PREFLIGHT_AMBIGUOUS_MATCH;
				}
				Match secondBestMatch = matches.get(1);
				if (comparator.compare(bestMatch, secondBestMatch) == 0) {
					Method m1 = bestMatch.handlerMethod.getMethod();
					Method m2 = secondBestMatch.handlerMethod.getMethod();
					String uri = request.getRequestURI();
					throw new IllegalStateException(
							"Ambiguous handler methods mapped for '" + uri + "': {" + m1 + ", " + m2 + "}");
				}
			}
			handleMatch(bestMatch.mapping, lookupPath, request);
			return bestMatch.handlerMethod;
		}
		else {
			return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
		}
	}
```
实际上就是把mappingRegistry变量（类型为`MappingRegistry`）里面的urlLookup变量（类型为`MultiValueMap<String, T>`）的值通过lookupPath（即请求路径）取出来，`MultiValueMap`是一个键对应多个值的一种特殊Map，因此能够得到多个值，然后在用这些值创建`Match`的实例，放到`Match`列表中，如果`Match`列表有多个`Match`元素，则从中选出一个best-matching（最匹配）的一个，那`Match`又是什么东西（感觉HandlerMapping应该单独再写一篇博客来总结，内容实在太多了），实际上它就是把上面的`MultiValueMap<String, T>`类型里面的T类型的元素和`HandlerMethod`绑定起来，方便选出最匹配的`Match`，最后返回最匹配的`Match`的`HandlerMethod`变量。所以最终最多只会选出一个`HandlerMethod`，也就是一个请求只能匹配到一个handler

到这里终于匹配完相应的handler了，让我们回到`AbstractHandlerMapping`的getHandler方法，现在它已经匹配到相应的handler了，之后它会调用getHandlerExecutionChain方法获取`HandlerExecutionChain`，下面来看getHandlerExecutionChain方法，这个方法也是在`AbstractHandlerMapping`类里面
```java
	protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
		HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
				(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

		String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
		for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
			if (interceptor instanceof MappedInterceptor) {
				MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
				if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
					chain.addInterceptor(mappedInterceptor.getInterceptor());
				}
			}
			else {
				chain.addInterceptor(interceptor);
			}
		}
		return chain;
	}
```
粗略地讲，这个方法会将请求路径匹配的所有拦截器添加到`HandlerExecutionChain`的实例中，然后返回该实例，最终我们就能得到`HandlerExecutionChain`实例了。

## 获取对应的HandlerAdapter
`HandlerAdapter`是调用getHandlerAdapter方法获取的，这个方法在`DispatcherServlet`里面
```java
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		if (this.handlerAdapters != null) {
			for (HandlerAdapter adapter : this.handlerAdapters) {
				if (adapter.supports(handler)) {
					return adapter;
				}
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```
方法很简单，就是遍历`DispatcherServlet`预先配置好的handlerAdapters，如果遍历到的`HandlerAdapter`支持处理前面匹配到的handler,那么就直接返回这个`HandlerAdapter`。

所以关键要看`DispatcherServlet`是怎样初始化handlerAdapters的。handlerAdapters的初始化是调用`DispatcherServlet`的initHandlerAdapters方法完成的
```java
	private void initHandlerAdapters(ApplicationContext context) {
		this.handlerAdapters = null;

		if (this.detectAllHandlerAdapters) {
			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
		    // 这里的逻辑和初始化HandlerMappingg类似
		    // 找到所有实现了HandlerAdapter接口的Bean
			Map<String, HandlerAdapter> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerAdapters = new ArrayList<>(matchingBeans.values());
				// We keep HandlerAdapters in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
			}
		}
		else {
			try {
				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
				this.handlerAdapters = Collections.singletonList(ha);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerAdapter later.
			}
		}

		// Ensure we have at least some HandlerAdapters, by registering
		// default HandlerAdapters if no other adapters are found.
		// 没有配置HandlerAdapter，就采用默认的配置
		if (this.handlerAdapters == null) {
			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
			if (logger.isTraceEnabled()) {
				logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
						"': using default strategies from DispatcherServlet.properties");
			}
		}
	}
```
可以看到它的逻辑和初始化`HandlerMapping`是类似的，都是找到实现了某个接口等所有Bean，如果这样的Bean存在，则把它们排序后放到列表中，如果不存在，则采用默认的配置(默认的配置在DispatcherServlet.properties文件中，相应的值为org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping)。

在添加`@EnableWebMvc`注解后，在`WebMvcConfigurationSupport`类中，会配置多个实现了`HandlerMapping`接口的Bean
```java
    @Bean
	public RequestMappingHandlerAdapter requestMappingHandlerAdapter() {
		RequestMappingHandlerAdapter adapter = createRequestMappingHandlerAdapter();
		adapter.setContentNegotiationManager(mvcContentNegotiationManager());
		adapter.setMessageConverters(getMessageConverters());
		adapter.setWebBindingInitializer(getConfigurableWebBindingInitializer());
		adapter.setCustomArgumentResolvers(getArgumentResolvers());
		adapter.setCustomReturnValueHandlers(getReturnValueHandlers());

		if (jackson2Present) {
			adapter.setRequestBodyAdvice(Collections.singletonList(new JsonViewRequestBodyAdvice()));
			adapter.setResponseBodyAdvice(Collections.singletonList(new JsonViewResponseBodyAdvice()));
		}

		AsyncSupportConfigurer configurer = new AsyncSupportConfigurer();
		configureAsyncSupport(configurer);
		if (configurer.getTaskExecutor() != null) {
			adapter.setTaskExecutor(configurer.getTaskExecutor());
		}
		if (configurer.getTimeout() != null) {
			adapter.setAsyncRequestTimeout(configurer.getTimeout());
		}
		adapter.setCallableInterceptors(configurer.getCallableInterceptors());
		adapter.setDeferredResultInterceptors(configurer.getDeferredResultInterceptors());

		return adapter;
	}

```
可以看到配置的`HandlerAdapter`具体实现类为`RequestMappingHandlerAdapter`
```java
/**
 * Extension of {@link AbstractHandlerMethodAdapter} that supports
 * {@link RequestMapping} annotated {@code HandlerMethod RequestMapping} annotated {@code HandlerMethods}.
 *
 * <p>Support for custom argument and return value types can be added via
 * {@link #setCustomArgumentResolvers} and {@link #setCustomReturnValueHandlers},
 * or alternatively, to re-configure all argument and return value types,
 * use {@link #setArgumentResolvers} and {@link #setReturnValueHandlers}.
 *
```
看它的注解，发现这个类是标注有`@RequestMapping`的handler（`HandlerMethod`类型，因为被`@RequestMapping`标注的方法会被自动发现然后创建一个相应的`HandlerMethod`）的支撑类，用来执行这类handler。`RequestMappingHandlerAdapter`内部做了很多工作，比如配置HandlerMethodArgumentResolver，配置HttpMessageConverter等（内容有点多了，需要再写一篇博客单独讲解）

在后面DEBUG设置断点查看具体会创建多少个`HandlerAdapter`的时候，发现一个有3个

![Imgur](https://i.imgur.com/79hCOpf.png)

```java
	/**
	 * Returns a {@link HttpRequestHandlerAdapter} for processing requests
	 * with {@link HttpRequestHandler HttpRequestHandlers}.
	 */
    @Bean
	public HttpRequestHandlerAdapter httpRequestHandlerAdapter() {
		return new HttpRequestHandlerAdapter();
	}

	/**
	 * Returns a {@link SimpleControllerHandlerAdapter} for processing requests
	 * with interface-based controllers.
	 */
	@Bean
	public SimpleControllerHandlerAdapter simpleControllerHandlerAdapter() {
		return new SimpleControllerHandlerAdapter();
	}
```
上面的是遗漏的其他两个`HandlerAdapter`

## 调用拦截器的preHandle方法
获取到了`HandlerAdapter`之后，就可以调用它的handle方法调用handler了，不过由于拦截器（`HandlerInterceptor`）的preHandle方法是要在handler调用之前执行的，所以首先要把`HandlerExecutionChain`里面的所有拦截器的preHandle方法执行一遍。

我们回到`DispatcherHandler`的doDispatch方法，可以看到是调用`HandlerExecutionChain`的applyPreHandle方法执行拦截器的preHandle方法的，所以我们来看`HandlerExecutionChain`的applyPreHandle方法
```java
	/**
	 * Apply preHandle methods of registered interceptors.
	 * @return {@code true} if the execution chain should proceed with the
	 * next interceptor or the handler itself. Else, DispatcherServlet assumes
	 * that this interceptor has already dealt with the response itself.
	 */
	boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
		// 获取该实例中所有的HandlerInterceptor
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = 0; i < interceptors.length; i++) {
				遍历素有的HandlerInterceptor，逐个执行perHandle方法
				HandlerInterceptor interceptor = interceptors[i];
				if (!interceptor.preHandle(request, response, this.handler)) {
					// 如果preHandle返回false，则认为它已经自己处理好请求并返回给了客户端
					// 后面的拦截器不需要再执行了，而且认为本次请求已经完成，所以调用拦截器的afterCompletion方法
					triggerAfterCompletion(request, response, null);
					return false;
				}
				this.interceptorIndex = i;
			}
		}
		return true;
	}

	@Nullable
	public HandlerInterceptor[] getInterceptors() {
		if (this.interceptors == null && this.interceptorList != null) {
			this.interceptors = this.interceptorList.toArray(new HandlerInterceptor[0]);
		}
		return this.interceptors;
	}
```
这个方法就是把`HandlerExecutionChain`实例中所有的拦截器的preHandle都执行一遍（假设全部都返回true），如果preHandle方法返回了false，则表示不需要调用后面未执行的拦截器的preHandle方法了，而且认为返回false的preHandle方法已经向客户端返回了数据，本次请求已经完成了，所以后面会直接调用拦截器的afterCompletino方法。

## 调用handler
终于到了真正调用handler这一步了，还是在`DispatcherHandler`的doDispatch方法里面，可以看到是调用`HandlerAdapter`的handle方法来调用handler的，假设我们前面获取到的handler是`HandlerMethod`类型的，则我们的`HandlerAdapter`将会是`RequestMappingHandlerAdapter`，所以我们来看到`RequestMappingHandlerAdapter`的handler方法，然后发现这个类里面没有这个方法，所以这个方法是从它的父类`AbstractHandlerMethodAdapter`继承的，`AbstractHandlerMethodAdapter`的handle方法如下：
```java
	@Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
        // 调用了handleInternal方法，这是一个抽象方法
		return handleInternal(request, response, (HandlerMethod) handler);
	}
	@Nullable
	protected abstract ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception;
```

可以看到具体的实现其实是在handleInternal里面，而这是一个抽象方法，需要子类实现，`RequestMappingHandlerAdapter`实现了这个方法
```java
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		// 检查是否支持处理这个请求和是否需要session
		checkRequest(request);

		// Execute invokeHandlerMethod in synchronized block if required.
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		}
		else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}

		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			}
			else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```
然后重点看invokeHandlerMethod方法，这个方法返回一个`ModelAndView`，以便后面进行视图解析和渲染等操作
```java
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}
```
这个方法做的东西很多而且很核心，内容有点多，所以也将单独写一篇博客进行讲解和总结。可以看到这个方法将会创建一个`ServletInvocableHandlerMethod`对象，然后为它设置好参数解析器等东西，调用`ServletInvocableHandlerMethod`的invokeAndHandle方法，最后根据调用结果创建`ModelAndView`对象并返回

## 调用拦截器的postHandle方法
调用完`HandlerAdapter`的handle方法之后，handle已经执行完成了，并返回了相应的`ModelAndView`对象。因为拦截器的postHandle方法是在handler执行完之后再执行的，因此后面将会执行拦截器的postHandler方法，在`DispatcherServlet`的doDispatch方法中，可以看到将会调用`HandlerExecutionChain`的applyPostHandle方法执行拦截器的postHandler方法（前面还有一个applyDefaultViewName被我跳过了，这个是用来解析默认的视图名称的）
```java
	void applyPostHandle(HttpServletRequest request, HttpServletResponse response, @Nullable ModelAndView mv)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}
```
可以看到和applypreHandle方法差不多，不过applyPostHandle是由最后一个`HandlerInterceptor`开始倒过来执行postHandle方法的，这和applypreHandle方法相反。

## 异常处理
如果上面的步骤抛出了异常，就返回异常相关的`ModelAndView`，具体处理流程如下（在`DispatchreServlet`的processDispatchResult方法中）：
```java
		boolean errorView = false;

		if (exception != null) {
			// 如果异常是ModelAndViewDefiningException类型的，则直接获取里面的ModelAndView
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				// 如果异常是其他类型的，则调用processHandlerException获取与异常相关的ModelAndView
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}
```

## 解析和渲染视图
如果前面返回的`ModelAndView`不为空，则调用`DispatcherServler`的render准备进行视图渲染
```java
	/**
	 * Render the given ModelAndView.
	 * <p>This is the last stage in handling a request. It may involve resolving the view by name.
	 * @param mv the ModelAndView to render
	 * @param request current HTTP servlet request
	 * @param response current HTTP servlet response
	 * @throws ServletException if view is missing or cannot be resolved
	 * @throws Exception if there's a problem rendering the view
	 */
	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale =
				(this.localeResolver != null ? this.localeResolver.resolveLocale(request) : request.getLocale());
		response.setLocale(locale);

		View view;
		String viewName = mv.getViewName();
		if (viewName != null) {
			// We need to resolve the view name.
			view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
			if (view == null) {
				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
						"' in servlet with name '" + getServletName() + "'");
			}
		}
		else {
			// No need to lookup: the ModelAndView object contains the actual View object.
			view = mv.getView();
			if (view == null) {
				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
						"View object in servlet with name '" + getServletName() + "'");
			}
		}

		// Delegate to the View object for rendering.
		if (logger.isTraceEnabled()) {
			logger.trace("Rendering view [" + view + "] ");
		}
		try {
			if (mv.getStatus() != null) {
				response.setStatus(mv.getStatus().value());
			}
			view.render(mv.getModelInternal(), request, response);
		}
		catch (Exception ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Error rendering view [" + view + "]", ex);
			}
			throw ex;
		}
	}
```
首先使用LocaleResolver解析请求所属的地区，接着获取视图默认的视图名（`ModelAndView`里面应该已经有相应的视图名称了），如果不为空则调用resolveViewName方法解析视图

### 解析视图
下面来看resolveViewName方法（在`DispatcherServlet`里面）：
```java
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {

		if (this.viewResolvers != null) {
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					return view;
				}
			}
		}
		return null;
	}
```
如果视图解析器不为空，则使用调用视图解析器的resolveViewName方法解析视图。那么视图解析器是怎么初始化的呢？和前面的`HandlerMapping`还有`HandlerAdapter`一样，也是先查找相应类型的Bean（这里是`ViewResolver`类型），如果找到的话就添加到viewResolvers变量里面，如果没有找到则使用默认的视图解析器（DispatcherServler.properties文件里面定义的）。

再次去看`WebMvcConfigurationSupport`这个类，看它有没有配置`ViewResolver`类型的Bean，发现配置了一个mvcViewResolver，具体类型为`ViewResolverComposite`
```java

	/**
	 * Register a {@link ViewResolverComposite} that contains a chain of view resolvers
	 * to use for view resolution.
	 * By default this resolver is ordered at 0 unless content negotiation view
	 * resolution is used in which case the order is raised to
	 * {@link org.springframework.core.Ordered#HIGHEST_PRECEDENCE
	 * Ordered.HIGHEST_PRECEDENCE}.
	 * <p>If no other resolvers are configured,
	 * {@link ViewResolverComposite#resolveViewName(String, Locale)} returns null in order
	 * to allow other potential {@link ViewResolver} beans to resolve views.
	 * @since 4.1
	 */
	@Bean
	public ViewResolver mvcViewResolver() {
		ViewResolverRegistry registry = new ViewResolverRegistry(
				mvcContentNegotiationManager(), this.applicationContext);
		configureViewResolvers(registry);

		if (registry.getViewResolvers().isEmpty() && this.applicationContext != null) {
			String[] names = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
					this.applicationContext, ViewResolver.class, true, false);
			if (names.length == 1) {
				// 当只有一个元素的时候，说明只有它自身，没有其它的ViewResolver类型的Bean了
				// 这时候添加一个InternalResourceViewResolver
				registry.getViewResolvers().add(new InternalResourceViewResolver());
			}
		}

		ViewResolverComposite composite = new ViewResolverComposite();
		composite.setOrder(registry.getOrder());
		composite.setViewResolvers(registry.getViewResolvers());
		if (this.applicationContext != null) {
			composite.setApplicationContext(this.applicationContext);
		}
		if (this.servletContext != null) {
			composite.setServletContext(this.servletContext);
		}
		return composite;
	}
```

从注释上可以知道，这个视图解析器（类型为`ViewResolverComposite`），是用来存放其它的`ViewResolver`的，然后当调用它的`ViewResolver`的resolveViewName方法时，它会遍历它存放的所有`ViewResolver`然后调用这些`ViewResolver`的resolveViewName方法（其实就是代理模式，它本身的resolveViewName方法其实只是调用其它的`ViewResolver`的resolveViewName方法，如果它里面存放的`ViewResolver`数量为0，这个方法将什么也不做）。

通过DUBUG设置断点查看有实际多少个`ViewResolver`类型的Bean，发现真的只有一个，就是上面的mvcViewResolver，具体类型为`ViewResolverComposite`，而它里面竟然存放一个`ViewResolver`，类型为`InternalResourceViewResolver`。

![Imgur](https://i.imgur.com/CDGQmGy.png)

这个`InternalResourceViewResolver`是什么时候加进去的呢？？？回到配置mvcViewResolver这个Bean的代码，可以发现：
```java
if (names.length == 1) {
    registry.getViewResolver().add(new InternalResourceViewResolver());
}
```
上面创建了一个`InternalResourceViewResolver`实例并添加到了registry的ViewResolver列表中，registry是`ViewResolverRegistry`类型的，它里面有一个viewResolvers变量（类型为`List<ViewResolver>`），所以前面创建的`InternalResourceViewResolver`实例会被添加到viewResolvers变量里面，然后，`composite.setViewResolvers(registry.getViewResolvers());`这句会把这个实例添加到`ViewResolverComposite`实例的视图解析器列表中，也就是把`InternalResourceViewResolver`这个视图解析器存放到`ViewResolverComposite`这个视图解析器里面。

初始化`ViewResolver`的过程大概就是这样了，下面回到`DispatcherServlet`的resolveViewName方法，
```java
for (ViewResolver viewResolver : this.viewResolvers)
```
所以上面的viewResolvers默认情况下应该只有一个元素，也就是`ViewResolverComposite`的实例，后面
```java
View view = viewResolver.resolveViewName(viewName, locale);
```
调用的resolveViewName方法也就是`ViewResolverComposite`的resolveViewName方法，然后代理给`InternalResourceViewResolver`执行，也就是说最终将执行`InternalResourceViewResolver`的resolveViewName方法。


去查看`InternalResourceViewResolver`的resolveViewName方法，发现方法不存在，因此这个方法应该是从父类继承的，这里很容易想到它应该是用了模板方法模式。一般跟踪之后，最终可以发现它会返回一个类型为`InternalResourceView`的视图对象（其实和前面分析过的也运用了模板方法模式的过程是类似的，一路跟踪下去就可以发现它的套路了）
```java
	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		InternalResourceView view = (InternalResourceView) super.buildView(viewName);
		if (this.alwaysInclude != null) {
			view.setAlwaysInclude(this.alwaysInclude);
		}
		view.setPreventDispatchLoop(true);
		return view;
	}
```

所以调用`DispatcherServlet`的resolveViewName方法我们最终可能得到一个类型为`InternalResourceView`得到视图（`View`）对象

### 渲染视图
从上面可以知道我们得到了一个类型为`InternalResourceView`的视图对象，然后我们就可以调用`View`的render渲染视图了。
回到回到`DispatcherServlet`的render方法，在调用完resolveViewName方法后，如果返回的视图不为空，则
```java
view.render(mv.getModelInternal(), request, response);
```
调用返回的视图的render方法渲染视图，因为我们得到的是`InternalResourceView`对象，所以我们可以去看`InternalResourceView`的render方法，然而它并没有这个方法，说明这个方法是从父类继承的，又双叒叕是这个套路（模板方法模式）。经过一番跟踪，可以发现最终的渲染逻辑在`InternalResourceView`类的renderMergedOutputModel方法里面
```java
	/**
	 * Render the internal resource given the specified model.
	 * This includes setting the model as request attributes.
	 */
	@Override
	protected void renderMergedOutputModel(
			Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

		// Expose the model object as request attributes.
		exposeModelAsRequestAttributes(model, request);

		// Expose helpers as request attributes, if any.
		exposeHelpers(request);

		// Determine the path for the request dispatcher.
		String dispatcherPath = prepareForRendering(request, response);

		// Obtain a RequestDispatcher for the target resource (typically a JSP).
		RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
		if (rd == null) {
			throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
					"]: Check that the corresponding file exists within your web application archive!");
		}

		// If already included or response already committed, perform include, else forward.
		if (useInclude(request, response)) {
			response.setContentType(getContentType());
			if (logger.isDebugEnabled()) {
				logger.debug("Including [" + getUrl() + "]");
			}
			rd.include(request, response);
		}

		else {
			// Note: The forwarded resource is supposed to determine the content type itself.
			if (logger.isDebugEnabled()) {
				logger.debug("Forwarding to [" + getUrl() + "]");
			}
			rd.forward(request, response);
		}
	}
```
然而它实际上只能渲染jsp视图，而且具体逻辑不是由它实现的，而是由Servlet容器（比如Tomcat容器默认配置了一个处理请求路径以`.jsp`或者`.jspx`结尾的请求的Servlet，名称叫jsp）实现的，它实际上把请求转发给Servlet容器中的其它Servlet，然后让其他Servlet来处理，而Servlet容器，比如Tomcat，默认只配置了一个处理jsp的Servlet和一个充当静态资源服务器的默认Servlet（这个默认Servlet的servletMapping已经被移除了，因为我们配置了DispatcherServlet的值为`/`的时候会覆盖它的servletMapping)，所以默认情况下只能渲染jsp视图。

## 调用拦截器的afterCompletion方法
渲染完视图之后，还没有结束，还要执行拦截器的afterCompletion方法（所以拦截器的afterCompletion方法是在视图渲染完成之后执行的）。回到`DispatcherServlet`的processDispatchResult方法，可以看到在最后，会调用`HandlerExecutionChain`的triggerAfterCompletion来执行拦截器的afterCompletion方法。查看`HandlerExecutionChain`的triggerAfterCompletion方法
```java
	void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, @Nullable Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			// 这里将从最后一个执行的拦截器的索引开始从后完前执行
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}
```
可以发现，这里将从`this.interceptorIndex`位置开始，从后往前执行相应的拦截器的afterCompletion。为什么要这样执行，而不是从0到最后一个或者直接从最后一个到0？这是因为，执行拦截器的preHandle方法的时候，如果返回了false，后面的拦截器的preHandle是不会执行的，因为后面的拦截器的afterCompletion也不应该最后执行。所以索引是从`this.interceptorIndex`开始的，这个索引的位置为最后一个执行的拦截器的位置。

## 总结
SpringMVC用`DispatcherServlet`来分发请求，大概分为以下几个步骤：
1. 检查request是否包含Multipart，如果是则调用`MultipartResolver`的resolveMultipart方法将请求转换成`MultipartHttpServletRequest`类型

2. 调用`HandlerMapping`的getHandler方法获取`HandlerExecutionChain`，HandlerExecutionChain包含了最匹配的handler和所有匹配的拦截器（`HandlerInterceptor`）

3. 调用`HandlerAdapter`的supports方法获得支持调用本次请求匹配到的handler的`HandlerAdapter`

4. 调用`HandlerExecutionChain`的applyPreHandle方法执行拦截器的preHandle方法

5. 调用`HandlerAdapter`的handle方法调用handler，将返回`ModelAndView`

6. 调用`RequestToViewNameTranslator`的getViewName方法将请求路径映射到视图名称，并设置到`ModelAndView`中

7. 调用`HandlerExecutionChain`的applyPostHandle方法执行拦截器的postHandle方法

8. 调用`LocaleResolver`的resolveLocale方法解析请求对应的地区，并绑定到`HttpServletResponse`的属性中

9. 调用`ViewResolver`的resolveViewName方法解析视图，将返回`View`对象

10. 调用`View`的render方法渲染视图

11. 调用`HandlerExecutionChain`的triggerAfterCompletion方法来执行拦截器的afterCompletion方法

大概就是这样一个过程（异常处理等过程没有写出来）。
还有很多细节需要研究，个人感觉最复杂繁琐的就是上面的第2步了，在获取`HandlerExecutionChain`之前，需要获取请求匹配的handler，而这个handler（假设是我们写的Controller里面的方法）可能有很多个参数，参数可以有很多类型，需要根据`HttpServletRequest`的信息解析出需要的参数的值，而这需要很多`HandlerMethodArgumentResolver`来解析，所以这些参数的解析过程，还是比较复杂的。
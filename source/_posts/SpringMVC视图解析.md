---
title: SpringMVC视图解析
date: 2018-12-19 18:42:09
tags:
    - SpringMVC
    - Web
---

## 默认的视图解析器
spring-mvc默认的视图解析器为`InternalResourceViewResolver`，它是在`DispatcherServlet`里面的某个方法中初始化的
```java
	/**
	 * Initialize the ViewResolvers used by this class.
	 * <p>If no ViewResolver beans are defined in the BeanFactory for this
	 * namespace, we default to InternalResourceViewResolver.
	 */
	private void initViewResolvers(ApplicationContext context) {
		this.viewResolvers = null;

		if (this.detectAllViewResolvers) {
			// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
			Map<String, ViewResolver> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.viewResolvers = new ArrayList<>(matchingBeans.values());
				// We keep ViewResolvers in sorted order.
				AnnotationAwareOrderComparator.sort(this.viewResolvers);
			}
		}
		else {
			try {
				ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
				this.viewResolvers = Collections.singletonList(vr);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default ViewResolver later.
			}
		}

		// Ensure we have at least one ViewResolver, by registering
		// a default ViewResolver if no other resolvers are found.
		// 如果没有自定义视图解析器，就会配置一个默认的InternalResourceViewResolver
		if (this.viewResolvers == null) {
			this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
			if (logger.isTraceEnabled()) {
				logger.trace("No ViewResolvers declared for servlet '" + getServletName() +
						"': using default strategies from DispatcherServlet.properties");
			}
		}
	}
```
可以看到当没有配置自定义的视图解析器的时候，会调用getDefaultStrategies获取默认的视图解析器
```java
	/**
	 * Create a List of default strategy objects for the given strategy interface.
	 * <p>The default implementation uses the "DispatcherServlet.properties" file (in the same
	 * package as the DispatcherServlet class) to determine the class names. It instantiates
	 * the strategy objects through the context's BeanFactory.
	 * @param context the current WebApplicationContext
	 * @param strategyInterface the strategy interface
	 * @return the List of corresponding strategy objects
	 */
	@SuppressWarnings("unchecked")
	protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
		String key = strategyInterface.getName();
		String value = defaultStrategies.getProperty(key);
		if (value != null) {
			String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
			List<T> strategies = new ArrayList<>(classNames.length);
			for (String className : classNames) {
				try {
					Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
					Object strategy = createDefaultStrategy(context, clazz);
					strategies.add((T) strategy);
				}
				catch (ClassNotFoundException ex) {
					throw new BeanInitializationException(
							"Could not find DispatcherServlet's default strategy class [" + className +
							"] for interface [" + key + "]", ex);
				}
				catch (LinkageError err) {
					throw new BeanInitializationException(
							"Unresolvable class definition for DispatcherServlet's default strategy class [" +
							className + "] for interface [" + key + "]", err);
				}
			}
			return strategies;
		}
		else {
			return new LinkedList<>();
		}
	}
```
从注释里面可以看到，这个方法不仅可以获取默认的视图解析器，还可以获取其他默认的地域解析器，主题解析器等

## InternalResourceViewResolver的解析策略
`InternalResourceViewResolver`继承了`UrlBasedViewResolver`，而`AbstractCachingViewResolver`继承了`AbstractCachingViewResolver`，`AbstractCachingViewResolver`实现了`ViewResolver`接口
![Imgur](https://i.imgur.com/hGXx7Lo.png)

`InternalResourceViewResolver`需要根据视图名解析出`View`，而它解析出的`View`的实现类是`InternalResourceView`，它通过buildView方法创建`View`
```java
    @Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		// 将调用父类的buildView方法，也就是AbstractUrlBasedView的buildView方法
		InternalResourceView view = (InternalResourceView) super.buildView(viewName);
		if (this.alwaysInclude != null) {
			view.setAlwaysInclude(this.alwaysInclude);
		}
		view.setPreventDispatchLoop(true);
		return view;
	}
```

`AbstractUrlBaseView`的buildView方法
```java
	/**
	 * Creates a new View instance of the specified view class and configures it.
	 * Does <i>not</i> perform any lookup for pre-defined View instances.
	 * <p>Spring lifecycle methods as defined by the bean container do not have to
	 * be called here; those will be applied by the {@code loadView} method
	 * after this method returns.
	 * <p>Subclasses will typically call {@code super.buildView(viewName)}
	 * first, before setting further properties themselves. {@code loadView}
	 * will then apply Spring lifecycle methods at the end of this process.
	 * @param viewName the name of the view to build
	 * @return the View instance
	 * @throws Exception if the view couldn't be resolved
	 * @see #loadView(String, java.util.Locale)
	 */
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		Class<?> viewClass = getViewClass(); // 这里将返回InternalResourceView.class
		Assert.state(viewClass != null, "No view class");

		AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(viewClass);
		view.setUrl(getPrefix() + viewName + getSuffix());  //设置视图Url为前缀+视图名+后缀
        
		String contentType = getContentType();
		if (contentType != null) {
			view.setContentType(contentType);
		}

		view.setRequestContextAttribute(getRequestContextAttribute());
		view.setAttributesMap(getAttributesMap());

		Boolean exposePathVariables = getExposePathVariables();
		if (exposePathVariables != null) {
			view.setExposePathVariables(exposePathVariables);
		}
		Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
		if (exposeContextBeansAsAttributes != null) {
			view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
		}
		String[] exposedContextBeanNames = getExposedContextBeanNames();
		if (exposedContextBeanNames != null) {
			view.setExposedContextBeanNames(exposedContextBeanNames);
		}

		return view;
	}
```

上面的buildView方法创建了一个`InternalResourceView`实例，并设置了相关的信息。然后接下来看`DispatcherServlet`的processDispatchResult方法
```java
// Did the handler return a view to render?
// mv为ModelAndView实例，完整的方法代码可以查看源码
		if (mv != null && !mv.wasCleared()) {
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
```
可以看到，如果调用了相应的handler之后返回的ModelAndView不为空，则说明要渲染视图，会调用render方法进行渲染，接下来看render方法
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
		response.setLocale(locale);  // 用地域解析器解析地域并设置好

		View view;
		String viewName = mv.getViewName();  // 获取视图名用于稍后的解析
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
可以看到将调用resolveViewName方法根据视图名解析出相应的视图，然后如果解析到有视图，就调用`View`的render方法渲染视图。首先看resolveView方法
```java
	@Nullable
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {
        // 将调用视图解析器解析视图
        // 将使用所有视图解析器进行解析，一但解析到有视图，就马上返回该视图
        // 因此这个过程受视图解析器的优先级影响较大
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
然后看到render方法，这里假设没有配置自定义的视图解析器，因此调用的一定是`InternalResourceView`的render方法，而这个方法是从父类的父类继承的，然后实际上调用的将会是AbstractView的render方法
```java
	@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		if (logger.isDebugEnabled()) {
			logger.debug("View " + formatViewName() +
					", model " + (model != null ? model : Collections.emptyMap()) +
					(this.staticAttributes.isEmpty() ? "" : ", static attributes " + this.staticAttributes));
		}

		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		prepareResponse(request, response);

		// 该方法为抽象方法，需要子类自行实现
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}
```
可以看到该方法用到了模板方法模式，`InternalResourceView`实现了renderMergedOutputModel方法
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
		// 这里将使用Servlet容器实现的RequestDispatcher去获取目标资源
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
可以看到上面的方法使用`RequestDispatcher`处理请求（include或者forward），`RequestDispatcher`由Servlet容器实现，比如Tomcat，可以在Tomcat源码里面查看其具体实现。假设有如下的`Controller`
```java
@Controller
@RequestMapping("/html")
public class HtmlController {

    @GetMapping
    public String simpleHtml(){
        return "/WEB-INF/views/simple.jsp";
    }
}
```
然后在浏览器输入`http://host:port/html`并访问，接着会调用simpleHtml方法，结束调用后会返回`ModelAndView`，`ModelAndView`不为空，调用render方法渲染视图，然后最终会调用`RequestDispatcher`的forward方法转发上面的地址为"/WEB-INF/views/simple.jsp"的请求（可以在IDE里面设断点调试，就会发现流程和这里讲的一样），然后因为Tomcat默认配置了一个用于解析jsp的Servlet，这个Servlet匹配的路径为所有以.jsp/.jspx结尾的路径
```xml
    <servlet>
        <servlet-name>jsp</servlet-name>
        <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
        <init-param>
            <param-name>fork</param-name>
            <param-value>false</param-value>
        </init-param>
        <init-param>
            <param-name>xpoweredBy</param-name>
            <param-value>false</param-value>
        </init-param>
        <load-on-startup>3</load-on-startup>
    </servlet>
    <!-- The mappings for the JSP servlet -->
    <servlet-mapping>
        <servlet-name>jsp</servlet-name>
        <url-pattern>*.jsp</url-pattern>
        <url-pattern>*.jspx</url-pattern>
    </servlet-mapping>

```
可以在Tomcat目录/conf/web.xml这个文件里面看到上面的配置，因此Url为"/WEB-INF/views/simple.jsp"的请求将被上面的名称为jsp的Servlet处理，这个Servlet将把我们的jsp文件经过一般处理之后变成html内容（这个步骤其实就是真正的视图渲染，jsp的视图渲染是代理给Servlet容器的jsp Servlet完成，这个过程，貌似是将jsp文件编译成class文件，然后有一个output方法能输出html内容，过程还是有点复杂的，具体实现可以看Tomcat的tomcat-jasper模块），然后Tomcat容器将处理后得到的内容返回给客户端浏览器。
因为Tomcat默认只配置了处理jsp的Servlet，还有一个当做资源服务器的默认Servlet，这个默认Servlet匹配的路径为/，我们的DiapatcherServlet匹配的路径也是这个，因此这个默认的Servlet将会被我们的DiapatcherServlet覆盖。

所以一般情况下Tomcat只有两个Servlet，一个是我们的DiapatcherServlet，匹配路径为/，一个是处理jsp的Servlet，匹配路径为“*.jsp”，“*.jspx”，所以当我们的请求不以.jsp或者.jspx结尾的时候，都会匹配到DispatcherServlet。所以默认情况下如果你的Controller没有以@RestController标注，而是以@Controller标注，然后在Controller的方法里面return "xxx"的话，只能解析到jsp/jspx文件，如果文件不存在就会返回一个默认的404页面。为什么只能解析到jsp/jspx？因为默认只配置了一个匹配.jsp/.jspx的Servlet，如果是其他的类型，比如html，是不会被这个Servlet处理的，只会又匹配回DiapatcherServlet，然后DiapatcherServlet发现找不到hanlder处理的话最终也是返回404页面。

## 总结
因此，默认的视图解析器`InternalResourceViewResolver`只能解析jsp/jspx（其实也不是它解析的，而是把请求转发，让Servlet容器的jsp Servlet解析），如果需要解析html或者其他的格式，则需要[自行配置视图解析器](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-view)。其实Tomcat配置的默认Servlet是可以把非WEB-INF目录下的所有文件返回给客户端的（也就是我们可以直接输入相应的路径访问对应的文件），但是我们的DispatcherServlet覆盖了这个默认的Servlet，所以就不能直接通过地址访问到服务端的文件了（但是以.jsp/jspx结尾的文件是可以直接被访问的，当然前提是不在WEB-INF目录下，因为以.jsp/jspx结尾的请求是被jsp Servlet处理的，而不是DispatcherServlet）
![Imgur](https://i.imgur.com/dyfQnS9.png)
![Imgur](https://i.imgur.com/gURn7rq.png)
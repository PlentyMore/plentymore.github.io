---
title: SpringMVC纯注解配置
date: 2018-12-17 21:05:41
tags:
    - SpringMVC
    - Web
---


最近复习了一下SpringMVC，相关文档是[Web on Servlet Stack](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html)，发现里面似乎没有一个VAN♂全使用Java注解的使用SpringMVC开发的Web应用的完整例子（例子倒是有，不过太分散了，每隔几个章节贴一点代码），搜索官网里面的guides也没有搜到，有的话大概也是用Spring Boot的，而我想要的是只用Spring Framework的，自己手动配置的例子。于是只好从文档上面把零碎的代码片段搜集起来组成的一个完整例子了。

## 需要的依赖
```
<!-- https://mvnrepository.com/artifact/org.springframework/spring-webmvc -->
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-webmvc</artifactId>
            <version>5.1.2.RELEASE</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/javax.servlet/javax.servlet-api -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>javax.servlet-api</artifactId>
            <version>3.1.0</version>
            <scope>provided</scope>
        </dependency>
```
只需要spring-webmvc和servlet-api就行了，因为spring-mvc用到了其它的spring模块，所以它会把需要的其它spring模块都弄过来，servlet-api也是需要的，编译这个Web项目的时候要用到，没有这个的话会编译不了

## 目录结构
![Imgur](https://i.imgur.com/7a3mM3j.png)

`WebInit.java`
```
public class WebInit extends AbstractAnnotationConfigDispatcherServletInitializer {
    protected Class<?>[] getRootConfigClasses() {
        return new Class[]{WebConfig.class};
    }

    protected Class<?>[] getServletConfigClasses() {
        return null;
    }

    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```
这个类很关键，它利用了Servlet3.0+的一个[新特性](http://docs.jboss.org/jbossas/javadoc/7.1.2.Final/javax/servlet/ServletContainerInitializer.html)，`ServletContainerInitializer`，实现这个接口就可以在应用启动阶段在代码里面注册自己的servlets, filters, 和listeners，利用这个特性就可以连web.xml都省掉了，所以这个web应用一个xml文件都不需要。spring-web对于这个接口实现类是`ServletContainerInitializer`，可以在spring-web模块下面的META-INF/services目录看到一个名为javax.servlet.ServletContainerInitializer的文件，里面的内容为org.springframework.web.SpringServletContainerInitializer

`WebConfig.java`
```
@Configuration
@EnableWebMvc
@ComponentScan(basePackages = "com.pltm.springmvc")
public class WebConfig {

}
```
目前这个类只是用来启用自动扫描然后注入依赖，实际上它可以用来配置SpringMVC，只需要实现WebMvcConfigurer接口（这个接口里面的方法都是默认方法，因此你不需要重写这个接口的每一个方法），然后就可以按照自己的喜好进行配置了，比如配置静态资源路径，配置跨域等。@EnableWebMvc这个注解必须要有，没有这个注解，你的Controller是不会生效的，因为这个注解会导入`DelegatingWebMvcConfiguration`类，这个类上面有@Configuration注解，因此是一个Bean配置类，会被Spring创建，而且它继承了`WebMvcConfigurationSupport`，这个类里面配置了很多默认的@Bean，比如`RequestMappingHandlerMapping`，`PathMatcher`，`UrlPathHelper`，`ContentNegotiationManager`，`HandlerMapping viewControllerHandlerMapping()`，`BeanNameUrlHandlerMapping`，`HandlerMapping resourceHandlerMapping() `，`ResourceUrlProvider`，`HandlerMapping defaultServletHandlerMapping`，
`RequestMappingHandlerAdapter`等一大堆Bean，这些Bean的作用看名字就可以猜到大概了，少了这些Bean，你的Controller里面的handler(就是你用@RequestMapping这样的注解标注了的方法)是不会被解析的，最后会导致返回tomcat的404页面，控制台上会输出`org.springframework.web.servlet.DispatcherServlet.noHandlerFound No mapping for GET /`这样的WARNING级别的日志。

`IndexController.java`
```
@RestController
@RequestMapping("/")
public class IndexController {

    @GetMapping
    public String index(@RequestParam(value = "name", required = false) String name){
        return "hello" + (name == null ? "" : ", " + name);
    }
}
```
随便写的一个控制器，这里用的@RestController，返回字符串，而不是视图文件，因为返回视图要配置视图解析器这些东西，这里为了简单就直接返回字符串了（实际上最终返回格式的取决于你的请求头的Accept的值，内容是返回的字符串）

## Tomcat配置
不用Spring Boot的话只能外面跑个tomcat然后把这个Web项目打包成war或者其它tomcat可以接受的格式放到tomcat里面部署了，Spring Boot内置的tomcat和其它容器还是有丶方便的。

需要注意的就是项目的打包，由于一开始我忘了把依赖也一起打包进去，所以打包之后放到tomcat里面运行之后，一直404，在打包选项界面配置了好久才发现依赖没有被打包进去。假设你用的是Intllij IDEA，你可以直接点右上角那个运行按钮旁边的Edit Configurations按钮，然后添加一个tomcat local配置，选择好你本地的tomcat的目录，然后选择Deployment选项，按一下+
![Imgur](https://i.imgur.com/FdoWCfd.png)
记得要把依赖包都双击一下，然后就会把依赖打包到lib目录下面了，然后配置好就可以运行了，浏览器被自动打开，然后页面输出hello


## WebInit.java
继续讲一下这个类，它继承了`AbstractAnnotationConfigDispatcherServletInitializer`类，实现了三个方法。详情可以查看[api](https://docs.spring.io/spring/docs/5.1.3.RELEASE/javadoc-api/)

### getRootConfigClasses
>for "root" application context (non-web
这里要了解一下WebApplicationContext的分级，详情可以查看[文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-servlet-context-hierarchy)
>The root WebApplicationContext typically contains infrastructure beans, such as data repositories and business services that need to be shared across multiple Servlet instances
这里一般配置一些全局的（在所有Servlet中共享的）基础设施，比如和数据库操作对象Repository，业务逻辑对象Service等

### getServletConfigClasses
这里一般放一些只存在于特定Servlet中的对象（Servlet私有的，不被其他Servlet共享），比如视图解析配置对象，Controller对象，或者其他和Web相关的对象，一般情况下，基于sprinv-mvc的Web应用只需要DispatcherServlet这一个Servlet，因此可以把全部配置放到上面的root WebApplicationContext里面，就是直接让这个方法返回null，只配置上面的getRootConfigClasses方法。root WebApplicationContext和其它的WebApplicationContext的关系如下图
![Imgur](https://i.imgur.com/OcH8ETB.png)

### getServletMappings
这个是配置DispatcherServlet这个核心Servlet匹配的路径的，因为一般我们的应用都只有一个这个Servlet的实例，因此我们会将所有的请求都匹配到这个Servlet，然后由这个Servlet进行各种操作（比如根据请求路径调用我们写的handler，就是我们的controller里面的各个有配置了路径的方法），设置匹配"/"后，所有的请求都会匹配到DispatcherServlet（不明白的话可以先去了解一下servlet容器路径匹配的规则）


## WebMvc配置
前面已经提到了一下，只需要让WebConfig类（任意的类都可以，只要加个@Configuration或者@Compmponent注解，还需要至少有一个@EnableWebMvc注解在有@Configuration注解的类上面）实现`WebMvcConfigurer`接口，然后重写里面的默认方法就可以了。

具体来说，你可以随便创建一个类实现`WebMvcConfigurer`接口，然后在这个类上面加上@Configuration或者@Component注解，最后必须要有一个@EnableWebMvc注解，这个注解可以放在任何一个有@Configurationh或者注解的类上面，也可以放在@Component注解的类上面，@EnableWebMvc里面有个@Import注解，@Import注解的作用相当于XML文件配置里面的<import>标签，就是导入其他的Beans配置文件，@Configuration和@Component注解相当于<Beans>标签，当一个Beans配置文件被导入到另一个标签，会把<Beans>标签里面的所有<Bean>都导入进来，最后还会去除重复的<Bean>标签。那么@Configuratoin和@Component有什么不同？初略地讲，@Configuration里面被@Bean标注的非静态方法是被CGLIB增强过的，而@Component的没有（增强过的意思是在被@Configuration注解类里面，在被@Bean标注的方法里面调用其它同样被@Bean标注的方法，最终将从容器中返回那个方法创建的Bean，而不是直接调用那个方法，而在@Component注解的类里面就真的只是直接调用那个方法），具体可以查看[文档](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/core.html#beans-factorybeans-annotations)。

什么方法配置什么东西就不讲了，文档上面写得很清楚，[点击这里查看](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-config)

## 特定类型的Bean
spring-mvc里面有几种特定类型的Bean，你只需要自定义一个这些特定类型的Bean，就可以自定义spring-mvc的很多配置了，比如路径匹配，视图解析等，DispatcherServlet就会自动检测你有没有自定义这些特定类型Bean，如果没有的话它就使用默认的Bean，基本上用默认的Bean就能满足大部分需求了，如果需要配置的话我一般是像上面说的实现`WebMvcConfigurer`接口然后重写方法进行配置（留下了没技术的泪水.jpg

具体有那些类型的Bean可以[查看文档](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-servlet-special-bean-types)，就里就不再复读了

## DispatcherServlet处理过程
我们把所有请求都匹配到DiapatcherServlet这个Servlet，然后让它把请求进行一番操作后，再根据请求路径调用匹配的handler（我们写的Controller里面有@RequestMapping标注的方法，还有拦截器的方法，过滤器等），然后再经过一番操作后把结果发回给客户端。那么它具体是怎么操作的呢？文档上面有讲到这个[过程](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-servlet-sequence)。

1. 首先找到WebApplicationContext，然后绑定到请求的一个属性里面，这样我们的Controller和其他的拦截器过滤器等就可以获取到WebApplicationContext并使用了，属性的名称可以通过DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE这个静态变量找到，所以我们可以通过request.getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE)获取到绑定的WebApplicationContext
```
@RestController
@RequestMapping("/")
public class IndexController {

    @GetMapping
    public String index(@RequestParam(value = "name", required = false) String name, HttpServletRequest request){
        WebApplicationContext ctx = (WebApplicationContext) request.getAttribute(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);
        System.out.println(ctx);
        System.out.println(DispatcherServlet.WEB_APPLICATION_CONTEXT_ATTRIBUTE);
        return "hello" + (name == null ? "" : ", " + name);
    }
}
```
输出结果为
```
WebApplicationContext for namespace 'dispatcher-servlet', started on Tue Dec 18 19:13:42 CST 2018, parent: Root WebApplicationContext
org.springframework.web.servlet.DispatcherServlet.CONTEXT
```

2. 将locale resolver绑定到请求，然后在后面处理请求的时候（渲染视图，填充数据等）可以根据这个locale resolver解析出请求对应的客户端当前的位置，然后确定了位置信息之后使用相应的语言去继续处理请求（渲染视图，填充数据等），如果你不需要解析位置信息，你可以不使用它。一般网站不是面向全球的话，都不需要用到。

3. 然后轮到theme resolver，这个东西是用来让视图使用不同的主题的，主题（其实是一堆静态资源的集合，各种图片、css等东西组合起来）需要自己进行配置，如果你不需要主题，你可以......（复读暗示）（好像也没见过有人用

4. 如果你配置了multipart文件解析器，将会检查请求是不是带有multipart，如果是，就把请求封装成`MultipartHttpServletRequest`类型，然后继续进行后续的处理。

5. 根据请求路径找到对应的handler（你的Controller里面的被@RequestMapping这类注解标注的方法，还有过滤器，拦截器的方法），如果找到了相应的handler，则执行响应的handler，一般按照过滤器-拦截器前置处理-Controller-拦截器后置处理-拦截器这样的顺序执行对应的handler。如果没有找到就404警告

6. 最后，如果返回的`ModelAndView`不为空，将会渲染视图，否则，不渲染视图，返回的不是model的原因可能是因为拦截器的前置处理和后置处理拦截了请求然后直接返回了。

上面的步骤讲得比较粗糙，很多细节都没有体现，整个流程的逻辑在DiapatcherServlet的doService方法里面可以看到，因此要深入了解可以直接看对应的代码
```
	@Override
	protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
		logRequest(request);

		// Keep a snapshot of the request attributes in case of an include,
		// to be able to restore the original attributes after the include.
		Map<String, Object> attributesSnapshot = null;
		if (WebUtils.isIncludeRequest(request)) {
			attributesSnapshot = new HashMap<>();
			Enumeration<?> attrNames = request.getAttributeNames();
			while (attrNames.hasMoreElements()) {
				String attrName = (String) attrNames.nextElement();
				if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
					attributesSnapshot.put(attrName, request.getAttribute(attrName));
				}
			}
		}

		// Make framework objects available to handlers and view objects.
		request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
		request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
		request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
		request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
        // 上面的request.setAttribute 3连对应上面说的步骤123

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
		finally {
			if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
				// Restore the original attribute snapshot, in case of an include.
				if (attributesSnapshot != null) {
					restoreAttributesAfterInclude(request, attributesSnapshot);
				}
			}
		}
	}
```
其中doDispatch方法的实现如下：
```
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				// 上面检查请求是不是有multipart，对应前面说的步骤4
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				// 后面的一系列对应步骤5
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				// 调用完handle方法后进入步骤6，然后继续看下面的processDispatchResult方法
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			catch (Throwable err) {
				// As of 4.3, we're processing Errors thrown from handler methods as well,
				// making them available for @ExceptionHandler methods and other scenarios.
				dispatchException = new NestedServletException("Handler dispatch failed", err);
			}
			// 这个方法将会检查会决定是否要渲染视图
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Throwable err) {
			triggerAfterCompletion(processedRequest, response, mappedHandler,
					new NestedServletException("Handler processing failed", err));
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				if (mappedHandler != null) {
					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				}
			}
			else {
				// Clean up any resources used by a multipart request.
				if (multipartRequestParsed) {
					cleanupMultipart(processedRequest);
				}
			}
		}
	}
```
```
	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
			@Nullable Exception exception) throws Exception {

		boolean errorView = false;

		if (exception != null) {
			if (exception instanceof ModelAndViewDefiningException) {
				logger.debug("ModelAndViewDefiningException encountered", exception);
				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
			}
			else {
				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
				mv = processHandlerException(request, response, handler, exception);
				errorView = (mv != null);
			}
		}

		// Did the handler return a view to render?
		if (mv != null && !mv.wasCleared()) {
			// 这个方法将会用到视图解析器，然后疯狂操作
			render(mv, request, response);
			if (errorView) {
				WebUtils.clearErrorRequestAttributes(request);
			}
		}
		else {
			if (logger.isTraceEnabled()) {
				logger.trace("No view rendering, null ModelAndView returned.");
			}
		}

		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
			// Concurrent handling started during a forward
			return;
		}

		if (mappedHandler != null) {
			mappedHandler.triggerAfterCompletion(request, response, null);
		}
	}
```
render方法的代码就不贴了，可以在DispatcherServlet里面可以看到具体实现细节。


### 异常处理
`HandlerExceptionResolver`这个类型的Bean可以[自定义异常处理逻辑](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-exceptionhandlers)

### 视图解析
在spring-mvc中，`ViewResolver`负责将视图的名称和实际的视图文件（比如视图名叫index，视图文件是index.html）关联起来，而`View`的则是：
>MVC View for a web interaction. Implementations are responsible for rendering content, and exposing the model. A single view exposes multiple model attributes.


`ViewResolver`的实现有以下几个：`AbstractCachingViewResolver`，`XmlViewResolver`，`ResourceBundleViewResolver`，`UrlBasedViewResolver`，`InternalResourceViewResolver`，`FreeMarkerViewResolver`，`ContentNegotiatingViewResolver`，具体可以查看[文档](https://docs.spring.io/spring/docs/5.1.3.RELEASE/spring-framework-reference/web.html#mvc-viewresolver)



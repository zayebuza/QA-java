# Spring MVC

**Spring MVC启动过程大致分为两个过程**：

1. ContextLoaderListener初始化，实例化IoC容器，并将此容器实例注册到ServletContext中。
2. DispatcherServlet初始化。



## 一，实例化spring IOC容器

想在Web容器中使用Spirng MVC，必须进行四项的配置：

- 修改web.xml，添加servlet定义
- 编写servletname-servlet.xml（ servletname是在web.xm中配置DispactherServlet时使servlet-name的值） 
- 配置contextConfigLocation初始化参数
- 配置ContextLoaderListerner。

在Spring MVC的项目结构中的 web.xml


ContextLoaderListener：Spring MVC在Web容器中的启动类，负责Spring IoC容器在Web上下文中的初始化
contextConfigLocation：指定Spring IoC容器需要读取的定义了非web层的Bean（DAO/Service）的XML文件路径。
    <!-- 配置Spring配置文件路径 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>
            classpath*:applicationContext.xml
            classpath*:applicationContext-shiro.xml
        </param-value>
    </context-param>
    <!-- 配置Spring上下文监听器 -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

ContextLoaderListener是一个监听器,其实现了ServletContextListener接口，其用来监听Servlet，当tomcat启动时会初始化一个Servlet容器，这样ContextLoaderListener会监听到Servlet的初始化，这样在Servlet初始化之后我们就可以在ContextLoaderListener中也进行一些初始化操作。看下面的ServletContextListener的源码也是比较简单的，ContextLoaderListener实现了ServletContextListener接口，所以会有两个方法contextInitialized和contextDestroyed。web容器初始化时会调用方法contextInitialized，web容器销毁时会调用方法contextDestroyed。

ServletContextListener 是ServletContext 的监听者，如果 ServletContext 发生变化，ServletContextListener 就可以监听到，如服务器启动时 ServletContext 被创建，服务器关闭时 ServletContext 将要被销毁

**类：ContextLoaderListener**

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
	public ContextLoaderListener() {}
	public ContextLoaderListener(WebApplicationContext context) {
   	 	super(context);
	}

	/**
	* Spring框架由此启动, contextInitialized也就是监听器类的main入口函数 
	* Web容器调用contextInitialized方法初始化ContextLoaderListener
 	* Initialize the root web application context.
 	* web容器初始化时会调用方法contextInitialized
 	* 这里传入的context是tomcat启动的时候初始化的一个容器
 	*/
	@Override
	public void contextInitialized(ServletContextEvent event) {
		//在父类ContextLoader中实现
		initWebApplicationContext(event.getServletContext());
	}
	/**
 	* Close the root web application context.
 	* web容器销毁时会调用方法contextDestroyed
 	*/
	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```



ContextLoaderListener的方法contextInitialized（）的默认实现是在他的父类ContextLoader的initWebApplicationContext方法中实现的，意思就是初始化web应用上下文。他的主要流程就是创建一个IOC容器，并将创建的IOC容器存到servletContext中，ContextLoader的核心实现如下
**类：    ContextLoader**
**函数：initWebApplicationContext**

```java
 * Initialize Spring's web application context for the given servlet context,通过给定的一个context初始化Spring的web上下文
 * using the application context provided at construction time, or creating a new one
 * according to the "{@link #CONTEXT_CLASS_PARAM contextClass}" and
 * "{@link #CONFIG_LOCATION_PARAM contextConfigLocation}" context-params.
 * @param servletContext current servlet context
 * @return the new WebApplicationContext
 * @see #ContextLoader(WebApplicationContext)
 * @see #CONTEXT_CLASS_PARAM
 * @see #CONFIG_LOCATION_PARAM
 */
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
	//判断web容器中是否已经有WebApplicationContext  
	if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
				"Cannot initialize context because there is already a root application context present - " +
				"check whether you have multiple ContextLoader* definitions in your web.xml!");
	}

	Log logger = LogFactory.getLog(ContextLoader.class);
	servletContext.log("Initializing Spring root WebApplicationContext");
	if (logger.isInfoEnabled()) {
		logger.info("Root WebApplicationContext: initialization started");
	}
	long startTime = System.currentTimeMillis();

	try {
		// Store context in local instance variable, to guarantee that
		// it is available on ServletContext shutdown.
		//创建WebApplicationContext
		if (this.context == null) {
			this.context = createWebApplicationContext(servletContext);
		}
		if (this.context instanceof ConfigurableWebApplicationContext) {
			ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
			//判断应用上下文是否激活 
			if (!cwac.isActive()) {
				// The context has not yet been refreshed -> provide services such as
				// setting the parent context, setting the application context id, etc
				//判断是否已经有父容器
				if (cwac.getParent() == null) {
					// The context instance was injected without an explicit parent ->
					// determine parent for root web application context, if any.
					//获得父容器
					ApplicationContext parent = loadParentContext(servletContext);
					cwac.setParent(parent);
				}
				//设置并刷新WebApplicationContext容器  方法内部会refresh
				configureAndRefreshWebApplicationContext(cwac, servletContext);
			}
		}
	//将初始化的WebApplicationContext设置到servletContext中
	servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
		ClassLoader ccl = Thread.currentThread().getContextClassLoader();
		if (ccl == ContextLoader.class.getClassLoader()) {
			currentContext = this.context;
		}
		else if (ccl != null) {
			currentContextPerThread.put(ccl, this.context);
		}
		if (logger.isDebugEnabled()) {
			logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
					WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
		}
		if (logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
		}
		//初始化 WebApplicationContext完成并返回
		return this.context;
	}
	catch (RuntimeException ex) {
		logger.error("Context initialization failed", ex);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}
	catch (Error err) {
		logger.error("Context initialization failed", err);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
		throw err;
	}
}
```
实例化Spring IoC容器的过程中，最主要的两个方法是
createWebApplicationContext和configureAndRefreshWebApplicationContext方法。

createWebApplicationContext方法用于返回XmlWebApplicationContext实例，即Web环境下的Spring IoC容器。

configureAndRefreshWebApplicationContext用于配置XmlWebApplicationContext，读取web.xml中通过contextConfigLocation标签指定的XML文件，实例化XML文件中配置的bean，并在上一步中实例化的容器中进行注册。

完成以上两步的操作后，Spring MVC会将XmlWebApplicationContext实例以属性的方式注册到ServletContext中，
属性的名称由WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE指定，默认值为：WebApplicationContext.class.getName() + ".ROOT"。
此Spring 容器是ROOT上下文，供所有的Spring MVC Servlcet使用



### 初始化bean    todo





## 二，DispatcherServlet初始化

Spring MVC作为spring项目中的子项目，其可以和spring web容器很好的兼容。其实现机制就是**springMVC也会自己初始化一个IOC容器，然后将spring web的IOC容器作为父容器**，这样就可以使用父容器中注入的bean了，由于是向上继承的，所以父容器无法使用子容器注入的Bean。springMVC的核心DispatcherServlet在web.xml中的配置

   <!-- Spring MVC 核心控制器 DispatcherServlet 配置 -->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath*:spring-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <!-- 拦截所有/rest/* 的请求,交给DispatcherServlet处理,性能最好 -->
        <url-pattern>/rest/*</url-pattern>
    </servlet-mapping>

web容器启动时会首先执行DispatcherServlet，其实质也是一个HttpServlet
DispatcherServlet.uml 如下，![Snipaste_2018-06-05_16-10-08](C:\Users\miaodongbiao\Desktop\Snipaste_2018-06-05_16-10-08.png)
	我们可以看到DispatcherServlet的体系结构，既然其有父类，所以很多初始化的操作就可能放到父类中完成，既然DispatcherServlet是一个Servlet，那么它就会有如下生命周期：

1. **初始化阶段**     调用init()方法。Servlet被装载后，Servlet容器创建一个Servlet实例并且调用Servlet的init()方法进行初始化。在Servlet的整个生命周期内，init()方法只被调用一次。
2. **响应客户请求阶段**　　调用service()方法
3. **终止阶段**　　 调用destroy()方法







###   1,HttpServletBean

​        初始化了一下SpringMVC配置文件的地址contextConfigLocation的配置属性，然后其调用的子类FrameworkServlet的initServletBean方法 

类：HttpServletBean

函数：init()

```java
    /**
	 * Map config parameters onto bean properties of this servlet, and
	 * invoke subclass initialization.
	 * @throws ServletException if bean properties are invalid (or required
	 * properties are missing), or if subclass initialization fails.
	 */
	@Override
	public final void init() throws ServletException {
		if (logger.isDebugEnabled()) {
			logger.debug("Initializing servlet '" + getServletName() + "'");
		}

		// Set bean properties from init parameters.
		try {
            //获得web.xml中的contextConfigLocation配置属性，就是spring MVC的配置文件
			PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
			BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
			ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
			bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            //模板方法，可以在子类中调用，做一些初始化工作，bw代表DispatcherServelt
            //模版设计模式，留给子类覆盖
			initBeanWrapper(bw);
            //将配置的初始化值设置到DispatcherServlet中
			bw.setPropertyValues(pvs, true);
		}
		catch (BeansException ex) {
			logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
			throw ex;
		}

		// Let subclasses do whatever initialization they like.
         //留给子类扩展
		initServletBean();

		if (logger.isDebugEnabled()) {
			logger.debug("Servlet '" + getServletName() + "' configured successfully");
		}
	}
```

类：FrameworkServlet

函数：initServletBean ()

```java
/**
	 * Overridden method of {@link HttpServletBean}, invoked after any bean properties
	 * have been set. Creates this servlet's WebApplicationContext.
	 */
	@Override
	protected final void initServletBean() throws ServletException {
		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
		if (this.logger.isInfoEnabled()) {
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
		}
		long startTime = System.currentTimeMillis();

		try {
			this.webApplicationContext = initWebApplicationContext();
			initFrameworkServlet();
		}
		catch (ServletException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}
		catch (RuntimeException ex) {
			this.logger.error("Context initialization failed", ex);
			throw ex;
		}

		if (this.logger.isInfoEnabled()) {
			long elapsedTime = System.currentTimeMillis() - startTime;
			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
					elapsedTime + " ms");
		}
	}
```

类：FrameworkServlet

函数：initWebApplicationContext

initWebApplicationContext方法的主要工作就是创建或者刷新WebApplicationContext实例并对servlet功能所使用的变量进行初始化。 

```java
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
			if (this.logger.isDebugEnabled()) {
				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
						"' as ServletContext attribute with name [" + attrName + "]");
			}
		}
		return wac;
	}
```

类：FrameworkServlet

函数：configureAndRefreshWebApplicationContext

```java
	protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			if (this.contextId != null) {
				wac.setId(this.contextId);
			}
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(getServletContext().getContextPath()) + "/" + getServletName());
			}
		}

		wac.setServletContext(getServletContext());
		wac.setServletConfig(getServletConfig());
		wac.setNamespace(getNamespace());
		wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
		}

		postProcessWebApplicationContext(wac);
		applyInitializers(wac);
		wac.refresh();
	}

```



类：FrameworkServlet

函数：createWebApplicationContext



```java

	/**
	 * Instantiate the WebApplicationContext for this servlet, either a default
	 * {@link org.springframework.web.context.support.XmlWebApplicationContext}
	 * or a {@link #setContextClass custom context class}, if set.
	 * <p>This implementation expects custom contexts to implement the
	 * {@link org.springframework.web.context.ConfigurableWebApplicationContext}
	 * interface. Can be overridden in subclasses.
	 * <p>Do not forget to register this servlet instance as application listener on the
	 * created context (for triggering its {@link #onRefresh callback}, and to call
	 * {@link org.springframework.context.ConfigurableApplicationContext#refresh()}
	 * before returning the context instance.
	 * @param parent the parent ApplicationContext to use, or {@code null} if none
	 * @return the WebApplicationContext for this servlet
	 * @see org.springframework.web.context.support.XmlWebApplicationContext
	 */
	protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
	     //获取servlet的初始化参数Contextclass，如果没有配置 默认为XmlWebApplicationContext
		Class<?> contextClass = getContextClass();
		if (this.logger.isDebugEnabled()) {
			this.logger.debug("Servlet with name '" + getServletName() +
					"' will try to create custom WebApplicationContext context of class '" +
					contextClass.getName() + "'" + ", using parent context [" + parent + "]");
		}
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException(
					"Fatal initialization error in servlet with name '" + getServletName() +
					"': custom WebApplicationContext class [" + contextClass.getName() +
					"] is not of type ConfigurableWebApplicationContext");
		}
        //通过反射方式实例化contextClass  
		ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

		wac.setEnvironment(getEnvironment());
         //parent为在ContextLoaderListener中创建的WebApplicationContext实例
		wac.setParent(parent);
        //初始化spring环境包括加载配置文件
		wac.setConfigLocation(getContextConfigLocation());

		configureAndRefreshWebApplicationContext(wac);

		return wac;
	}

```

类：AbstractApplicationContext 

函数：refresh



```java
    public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var5) {
                this.destroyBeans();
                this.cancelRefresh(var5);
                throw var5;
            }

        }
    }
```



继续看initWebApplicationContext方法调用DispatcherServlet类 onRefresh(wac)。 

```java
	/**
	 * This implementation calls {@link #initStrategies}.
	 */
	@Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

```

```java
/** //一套完整的springmvc 配置属性
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
	protected void initStrategies(ApplicationContext context) {
         //初始化MultipartResolver
		initMultipartResolver(context);
        //初始化LocaleResolver
		initLocaleResolver(context);
        //初始化ThemeResolver
		initThemeResolver(context);
        //初始化HandlerMappings ：注册了 RequestMappingHandlerMapping和BeanNameUrlHandlerMapping
		initHandlerMappings(context);
         //初始化HandlerAdapters 注册了 RequestMappingHandlerAdapter、HttpRequestHandlerAdapter和SimpleControllerHandlerAdapter
		initHandlerAdapters(context);
        //初始化HandlerExceptionResolvers
		initHandlerExceptionResolvers(context);
        //初始化RequestToViewNameTranslator
		initRequestToViewNameTranslator(context);
        //初始化ViewResolvers
		initViewResolvers(context);
        //初始化FlashMapManager
		initFlashMapManager(context);
	}

```



## 三，请求分发  todo



```java
/**
	 * Process the actual dispatching to the handler.
	 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
	 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
	 * to find the first that supports the handler class.
	 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
	 * themselves to decide which methods are acceptable.
	 * @param request current HTTP request
	 * @param response current HTTP response
	 * @throws Exception in case of any kind of processing failure
	 */
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
				multipartRequestParsed = processedRequest != request;

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
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
					if (logger.isDebugEnabled()) {
						String requestUri = urlPathHelper.getRequestUri(request);
						logger.debug("Last-Modified value for [" + requestUri + "] is: " + lastModified);
					}
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				try {
					// Actually invoke the handler.
					mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
				}
				finally {
					if (asyncManager.isConcurrentHandlingStarted()) {
						return;
					}
				}

				applyDefaultViewName(request, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			catch (Exception ex) {
				dispatchException = ex;
			}
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		catch (Exception ex) {
			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
		}
		catch (Error err) {
			triggerAfterCompletionWithError(processedRequest, response, mappedHandler, err);
		}
		finally {
			if (asyncManager.isConcurrentHandlingStarted()) {
				// Instead of postHandle and afterCompletion
				mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
				return;
			}
			// Clean up any resources used by a multipart request.
			if (multipartRequestParsed) {
				cleanupMultipart(processedRequest);
			}
		}
	}
```

### note

在spring中，实现控制反转的是IOC容器，实现方法是DI（依赖注入），spring先是根据配置文件完成Bean的定义和生成，然后会寻找需要注入的资源，然后注入。spring

1、IoC是什么

Ioc—Inversion of Control，即“控制反转”，不是什么技术，而是一种设计思想。在Java开发中，Ioc意味着将你设计好的对象交给容器控制，而不是传统的在你的对象内部直接控制，控制权从应用程序转移到框架（IOC容器）



##### 2，aop是什么

AOP称为面向切面编程，在程序开发中主要用来解决一些系统层面上的问题，比如日志，事务，权限等 

底层依赖的是 动态代理，参照jdk自带的动态代理。

Spring中的AOP代理还是离不开Spring的IOC容器，代理的生成，管理及其依赖关系都是由IOC容器负责，Spring默认使用JDK动态代理，在需要代理类而不是代理接口的时候，Spring会自动切换为使用CGLIB代理，不过现在的项目都是面向接口编程，所以JDK动态代理相对来说用的还是多一些。 

主要作用就是抽取公共代码，无侵入的增强现有类的功能 

不使用aop连接数据库的操作太繁琐，掺杂太多的try catch，可读性不高。

连接数据库，执行sql，事务，关流等操作，可以直接在aop里搞定。参照示例图

约定优于配置

正常sql的逻辑执行步骤





##### 3,ORM    Object Relation Mapping   对象关系映射

指将java对象状态自动映射到关系数据库中额数据上，从而提供透明化的持久化支持





##### 4,nosql

redis持久化

由于Redis的数据都存放在内存中，如果没有配置持久化，redis重启后数据就全丢失了，于是需要开启redis的持久化功能，将数据保存到磁盘上，当redis重启后，可以从磁盘中恢复数据 

为了能够重用Redis数据，或者防止系统故障，我们需要将Redis中的数据写入到磁盘空间中，即持久化。  Redis提供了两种不同的持久化方法可以将数据存储在磁盘中，一种叫快照（RDB），另一种叫只追加文件（AOF） 

##### 5，消息队列



##### 6，注册中心



##### 7，分布式事务

事务：事务(Transaction)，一般是指要做的或所做的事情。在计算机术语中是指访问并可能更新数据库中各种数据项的一个程序执行单元(unit)。在计算机术语中，事务通常就是指数据库事务 

一个数据库事务通常包含对数据库进行读或写的一个操作序列。它的存在包含有以下两个目的：

> 1、为数据库操作提供了一个从失败中恢复到正常状态的方法，同时提供了数据库即使在异常状态下仍能保持一致性的方法。
> 2、当多个应用程序在并发访问数据库时，可以在这些应用程序之间提供一个隔离方法，以防止彼此的操作互相干扰。

> ## 特性
>
> 并非任意的对数据库的操作序列都是数据库事务。事务应该具有4个属性：原子性、一致性、隔离性、持久性。这四个属性通常称为ACID特性。
>
> > **原子性（Atomicity）**：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行。
> > **一致性（Consistency）**：事务应确保数据库的状态从一个一致状态转变为另一个一致状态。一致状态的含义是数据库中的数据应满足完整性约束。
> > **隔离性（Isolation）**：多个事务并发执行时，一个事务的执行不应影响其他事务的执行。
> > **持久性（Durability）**：一个事务一旦提交，他对数据库的修改应该永久保存在数据库中。
>
> ## 举例
>
> 用一个常用的“A账户向B账号汇钱”的例子来说明如何通过数据库事务保证数据的准确性和完整性。熟悉关系型数据库事务的都知道从帐号A到帐号B需要6个操作：
>
> 1、从A账号中把余额读出来（500）。
> 2、对A账号做减法操作（500-100）。
> 3、把结果写回A账号中（400）。
> 4、从B账号中把余额读出来（500）。
> 5、对B账号做加法操作（500+100）。
> 6、把结果写回B账号中（600）。
>
> ### 原子性：
>
> 保证1-6所有过程要么都执行，要么都不执行。一旦在执行某一步骤的过程中发生问题，就需要执行回滚操作。 假如执行到第五步的时候，B账户突然不可用（比如被注销），那么之前的所有操作都应该回滚到执行事务之前的状态。
>
> ### 一致性
>
> 在转账之前，A和B的账户中共有500+500=1000元钱。在转账之后，A和B的账户中共有400+600=1000元。也就是说，数据的状态在执行该事务操作之后从一个状态改变到了另外一个状态。同时一致性还能保证账户余额不会变成负数等。
>
> ### 隔离性
>
> 在A向B转账的整个过程中，只要事务还没有提交（commit），查询A账户和B账户的时候，两个账户里面的钱的数量都不会有变化。
> 如果在A给B转账的同时，有另外一个事务执行了C给B转账的操作，那么当两个事务都结束的时候，B账户里面的钱应该是A转给B的钱加上C转给B的钱再加上自己原有的钱。
>
> ### 持久性
>
> 一旦转账成功（事务提交），两个账户的里面的钱就会真的发生变化（会把数据写入数据库做持久化保存）！
>
> ## 原子性与隔离行
>
> 一致性与原子性是密切相关的,原子性的破坏可能导致数据库的不一致，数据的一致性问题并不都和原子性有关。
> 比如刚刚的例子，在第五步的时候，对B账户做加法时只加了50元。那么该过程可以符合原子性，但是数据的一致性就出现了问题。
>
> 因此，事务的原子性与一致性缺一不可。

分布式事务

https://mp.weixin.qq.com/s/r8D_PJz6M6C6pD_EdX_Kag

https://mp.weixin.qq.com/s/oKOzvN49zOhl8cwliy3SEg



##### 8，分布式锁

在多线程并发的情况下，如何保证一个代码块在同一时间只能由一个线程访问？

--使用锁，java的 synchronized来保证，不过只能保证同一个JVM进程内的多个线程同步执行

![微信图片_20180619125702](C:\Users\miaodongbiao\Desktop\MD_image\微信图片_20180619125702.png)

如果是在分布式的集群环境中，在分布式系统中，实现不同线程对代码和资源的同步访问。怎么保证？如下

![QA-java/md_image/微信图片_20180619125702.png]



对于单进程的并发场景，我们可以使用语言和类提供的锁。 对于分布式场景，我们可以使用分布式锁

怎么实现分布式锁？ 

1，Memcached分布式锁

2，Redis分布式锁

https://mp.weixin.qq.com/s/8fdBKAyHZrfHmSajXT_dnA

3，Zookeeper分布式锁

https://mp.weixin.qq.com/s/u8QDlrDj3Rl1YjY4TyKMCA



可重入锁： 顾名思义，就是一个锁住的资源（比如代码块）可以被持有锁的线程反复进入 





##### spring事务机制，以及是如何管理的？

https://blog.csdn.net/jie_liang/article/details/77600742

https://blog.csdn.net/trigl/article/details/50968079

Spring的事务机制包括声明式事务和编程式事务。

编程式事务管理：Spring推荐使用TransactionTemplate，实际开发中使用声明式事务较多。

声明式事务管理：将我们从复杂的事务处理中解脱出来，获取连接，关闭连接、事务提交、回滚、异常处理等这些操作都不用我们处理了，Spring都会帮我们处理。

声明式事务管理使用了AOP面向切面编程实现的，本质就是在目标方法执行前后进行拦截。在目标方法执行前加入或创建一个事务，在执行方法执行后，根据实际情况选择提交或是回滚事务。

如何管理的：![Snipaste_2018-06-20_13-14-55](C:\Users\miaodongbiao\Desktop\MD_image\Snipaste_2018-06-20_13-14-55.png)

Spring并不直接管理事务，而是提供了多种事务管理器，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。  Spring事务管理器的接口是org.springframework.transaction.PlatformTransactionManager，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了 

```java
Public interface PlatformTransactionManager()...{  
    // 由TransactionDefinition得到TransactionStatus对象
    TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException; 
    // 提交
    Void commit(TransactionStatus status) throws TransactionException;  
    // 回滚
    Void rollback(TransactionStatus status) throws TransactionException;  
    } 
```

从这里可知具体的具体的事务管理机制对Spring来说是透明的，它并不关心那些，那些是对应各个平台需要关心的，所以Spring事务管理的一个优点就是为不同的事务API提供一致的编程模型，如JTA、JDBC、Hibernate、JPA。 

Spring事务管理主要包括3个接口，Spring的事务主要是由他们三个共同完成的。

1）PlatformTransactionManager：事务管理器--主要用于平台相关事务的管理

 主要有三个方法：commit  事务提交；

​      rollback  事务回滚；

​      getTransaction  获取事务状态。

2）TransactionDefinition：事务定义信息--用来定义事务相关的属性，给事务管理器PlatformTransactionManager使用

这个接口有下面四个主要方法：

getIsolationLevel：获取隔离级别；

getPropagationBehavior：获取传播行为；

getTimeout：获取超时时间；

isReadOnly：是否只读（保存、更新、删除时属性变为false--可读写，查询时为true--只读）

事务管理器能够根据这个返回值进行优化，这些事务的配置信息，都可以通过配置文件进行配置。

3）TransactionStatus：事务具体运行状态--事务管理过程中，每个时间点事务的状态信息。

例如它的几个方法：

hasSavepoint()：返回这个事务内部是否包含一个保存点，

isCompleted()：返回该事务是否已完成，也就是说，是否已经提交或回滚

isNewTransaction()：判断当前事务是否是一个新事务

声明式事务的优缺点：

优点

不需要在业务逻辑代码中编写事务相关代码，只需要在配置文件配置或使用注解（@Transaction），这种方式没有侵入性。

缺点

声明式事务的最细粒度作用于方法上，如果像代码块也有事务需求，只能变通下，将代码块变为方法。

##### springMVC拦截器

我们经常会遇到这些类似的情况，当我们登录到某个网站之后过一段时间再次刷新页面，可能会跳转到登录页面让我们再次登录；在有的网站我们无法查看某些内容，会提示我们权限不足。其实这些都是后台首先对我们的请求进行了拦截，然后决定跳转到哪里，这里我来讲一下我工作中用到的SpringMVC拦截器的用法。 

1，在spring配置文件中添加拦截器

```java
 <!-- 拦截器配置 -->
    <mvc:interceptors>
        <!-- 配置Token拦截器，防止用户重复提交数据 -->
        <mvc:interceptor>
            <mvc:mapping path="/**" /><!--这个地方时你要拦截得路径 我这个意思是拦截所有得URL-->
            <bean class="com.ucredit.mars.common.interceptor.TokenInterceptor"/><!--class文件路径改成你自己写得拦截器路径！！ -->
        </mvc:interceptor>
    </mvc:interceptors>
```

说明几点： 
1、每一个拦截器放在`<mvc:interceptor>`标签下面 
2、`<mvc:mapping>`和`<mvc:exclude-mapping>`故名思义，分别指会被拦截的和不会被拦截的请求 
3、`<bean>`中的类就是具体的自定义的拦截器类

2，自定义拦截器类继 实现HandlerInterceptor



##### 常用注解

 Spring的一个核心功能是IOC，就是将Bean初始化加载到容器中，Bean是如何加载到容器的，可以使用Spring注解方式或者Spring XML配置方式。  Spring注解方式减少了配置文件内容，更加便于管理，并且使用注解可以大大提高了开发效率！  下面按照分类讲解Spring中常用的一些注解。 

​	一，组件类注解

​	思考：spring怎么知道应该哪些Java类当初bean类处理？ 答案：使用配置文件或者注解的方式进行标识需要处理的java类! 

​	1，注解类介绍

​	@Component ：标准一个普通的spring Bean类。  

​	@Repository：标注一个DAO组件类。  

​	@Service：标注一个业务逻辑组件类。

​	 @Controller：标注一个控制器组件类。  

这些都是注解在平时的开发过程中出镜率极高，@Component、@Repository、@Service、@Controller实质上属于同一类注解，用法相同，功能相同，区别在于标识组件的类型。@Component可以代替@Repository、@Service、@Controller，因为这三个注解是被@Component标注的 

​	

​	二，装配bean时常用的注解

​	1，注解介绍





​	三，@Component    @Configuration    @bean

​	



​	四，web模块常用到的注解





​	五，事务常用注解



​	六，缓存注解



​	七，@RestControllerAdvice 异常处理





# SpringBoot2.0









# Dubbo



​		

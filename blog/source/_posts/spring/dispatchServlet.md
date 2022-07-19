---
title: DispatchServlet的源代码分析
date: 2022-07-16 20:57:54
tags: spring
type: "spring"
description: "DispatcherServlet是SpringMVC框架的核心之一，不少傻逼面试官喜欢围绕着这块问东问西。本文着重探讨下Spring的设计"
---

### DispatcherServlet的源代码分析
在web.xml中会有如下配置
```xml
<web-app> 
    <!--定义了WEB应用的名字 -->
    <display-name></display-name>
    <!--声明WEB应用的描述信息 -->
    <description></description> 
    <!--context-param元素声明应用范围内的初始化参数。-->
    <context-param></context-param>  
    <!--过滤器元素将一个名字与一个实现javax.servlet.Filter接口的类相关联。-->
    <filter></filter>  
    <!--一旦命名了一个过滤器，就要利用filter-mapping元素把它与一个或多个servlet或JSP页面相关联。 -->
    <filter-mapping></filter-mapping> 
    <!--servlet API的版本2.3增加了对事件监听程序的支持，事件监听程序在建立、修改和删除会话或servlet环境时得到通知。Listener元素指出事件监听程序类。-->
    <listener>org.springframework.web.context.ContextLoaderListener</listener> 
    <!--在向servlet或JSP页面制定初始化参数或定制URL时，必须首先命名servlet或JSP页面。Servlet元素就是用来完成此项任务的。-->
    <servlet>
        <servlet-name>myDispatchServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    </servlet> 
    <!--服务器一般为servlet提供一个缺省的URL：http://host/webAppPrefix/servlet/ServletName.但是，常常会更改这个URL，以便servlet可以访问初始化参数或更容易地处理相对URL。在更改缺省URL时，使用servlet-mapping元素。 -->
    <servlet-mapping></servlet-mapping> 
    <!--如果某个会话在一定时间内未被访问，服务器可以抛弃它以节省内存。 可通过使用HttpSession的setMaxInactiveInterval方法明确设置单个会话对象的超时值，或者可利用session-config元素制定缺省超时值。 -->
    <session-config></session-config> 
    <!--如果Web应用具有想到特殊的文件，希望能保证给他们分配特定的MIME类型，则mime-mapping元素提供这种保证。 -->
    <mime-mapping></mime-mapping>
    <!--指示服务器在收到引用一个目录名而不是文件名的URL时，使用哪个文件。 -->
    <welcome-file-list></welcome-file-list> 
    <!--在返回特定HTTP状态代码时，或者特定类型的异常被抛出时，能够制定将要显示的页面。 -->
    <error-page></error-page> 
    <!--对标记库描述符文件（Tag Libraryu Descriptor file）指定别名。此功能使你能够更改TLD文件的位置，而不用编辑使用这些文件的JSP页面。 -->
    <taglib></taglib> 
    <!--声明与资源相关的一个管理对象。 -->
    <resource-env-ref></resource-env-ref>
    <!--声明一个资源工厂使用的外部资源。 -->
    <resource-ref></resource-ref> 
    <!--制定应该保护的URL。它与login-config元素联合使用 -->
    <security-constraint></security-constraint> 
    <!--指定服务器应该怎样给试图访问受保护页面的用户授权。它与sercurity-constraint元素联合使用。 -->
    <login-config></login-config> 
    <!--给出安全角色的一个列表，这些角色将出现在servlet元素内的security-role-ref元素的role-name子元素中。分别地声明角色可使高级IDE处理安全信息更为容易。 -->
    <security-role></security-role>
    <!--声明Web应用的环境项。 -->
    <env-entry></env-entry>
    <!--声明一个EJB的主目录的引用。 -->
    <ejb-ref></ejb-ref>
    <!--声明一个EJB的本地主目录的应用。 -->
    <ejb-local-ref></ejb-local-ref>
</web-app>
```

web.xml下的配置
```xml
    <servlet>
        <servlet-name>myDispatchServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    </servlet> 
```



```java

public class DispatcherServlet extends FrameworkServlet {


    static {
        // Load default strategy implementations from properties file.
        // This is currently strictly internal and not meant to be customized
        // by application developers.
        try {
            ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, DispatcherServlet.class);
            defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
        }
        catch (IOException ex) {
            throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH + "': " + ex.getMessage());
        }
    }
    
    
}
```


先看下DispatcherServlet继承关系。
![DispatcherServlet继承关系](././images/dispatcherServletflows.png)
dispatcherServlet继承关系。
从继承关系上看，继承FrameworkServlet，frameworkservlet继承httpServletBean，httpServletBean继承了HttpServlet，
HttpServlet继承GenericServlet。
其中，GenericServlet和HttpServlet是javax.servlet.http的标准API。
先重点看下HttpServletBean

```java
public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {


	/**
	 * Map config parameters onto bean properties of this servlet, and
	 * invoke subclass initialization.
	 * @throws ServletException if bean properties are invalid (or required
	 * properties are missing), or if subclass initialization fails.
	 */
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


	/**
	 * Subclasses may override this to perform custom initialization.
	 * All bean properties of this servlet will have been set before this
	 * method is invoked.
	 * <p>This default implementation is empty.
	 * @throws ServletException if subclass initialization fails
	 */
	protected void initServletBean() throws ServletException {
	}

	

}

```

可以看到

`
initServletBean();
`
的具体实现交给了httpServletBean来进行实现的。

而这个的具体实现交给了FrameworkServlet来实现了
```java
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware{
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
}
```

可以看到在这个initServletBean中，进行了一次webApplication的容器初始化，frameworkServlet的初始化。
重点来看下这段逻辑。

在FrameworkServlet中实现了如下逻辑
```java
    public abstract class FrameworkServlet extends HttpServletBean{
    
    
    //创建webApplicationContext
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
            synchronized (this.onRefreshMonitor) {
                onRefresh(wac);
            }
        }

        if (this.publishContext) {
            // Publish the context as a servlet context attribute.
            String attrName = getServletContextAttributeName();
            getServletContext().setAttribute(attrName, wac);
        }

        return wac;
    }

}


    
```

注意如上的逻辑，通过WebApplicationContextUtils.getWebApplicationContext(),获取了之前ContextLoaderListener创建的ApplicationContext。
这其中，ContextLoader是创建的Spring的Context，而这个是创建了Spring MVC ApplicationContext。
之所以有两个ApplicationContext，是为了职责分开。
第一个是管理Spring应用相关的Bean,第二个是Web应用相关的bean.

在DispatcherServlet中有如下定义：
```java

    public class FrameworkServlet{
        protected WebApplicationContext createWebApplicationContext(@Nullable ApplicationContext parent) {
            Class<?> contextClass = getContextClass();
            if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
                throw new ApplicationContextException(
                        "Fatal initialization error in servlet with name '" + getServletName() +
                                "': custom WebApplicationContext class [" + contextClass.getName() +
                                "] is not of type ConfigurableWebApplicationContext");
            }
            ConfigurableWebApplicationContext wac =
                    (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

            wac.setEnvironment(getEnvironment());
            wac.setParent(parent);
            String configLocation = getContextConfigLocation();
            if (configLocation != null) {
                wac.setConfigLocation(configLocation);
            }
            configureAndRefreshWebApplicationContext(wac);

            return wac;
        }

    } 
```
在创建WebMV ApplicationContext的同时，初始化了环境，
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
						ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
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


在这个过程中会检测容器有无刷新，
```java

if (!this.refreshEventReceived) {
// Either the context is not a ConfigurableApplicationContext with refresh
// support or the context injected at construction time had already been
// refreshed -> trigger initial onRefresh manually here.
           synchronized (this.onRefreshMonitor) {
                 onRefresh(wac);
            }
        }
```

下面着重看下onRefresh方法
```java
    @Override
	protected void onRefresh(ApplicationContext context) {
		initStrategies(context);
	}

	/**
	 * Initialize the strategy objects that this servlet uses.
	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
	 */
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
#### DispatcherServlet的默认策略

dispatchServlet的默认策略，加载了"spring-webmvc"包下的DispatcherServlet.properties
该properties定义了如下配置项：
```properties

org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter,\
	org.springframework.web.servlet.function.support.HandlerFunctionAdapter


org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```


如上配置文件定义了localeResolver,themeResolver,handlerMapping,handlerAdapter,
handlerExpcetionResolver,requestToViewNameTranslator,viewResolver,flashMapManager.
这些接口在org.springframework.web.servlet包下做了定义。
![分发策略](././images/dispatcherStrategies.png)


```java
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
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
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













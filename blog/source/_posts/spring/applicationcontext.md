---
title: ApplicationContext的启动分析
date: 2022-07-16 20:57:54
tags: spring
type: "spring"
---

### ApplicationContext的启动分析

在Spring MVC 中，创建一个Spring MVC的项目，需要在web.xml中作如下配置
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
    <servlet></servlet> 
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

`
org.springframework.web.context.ContextLoaderListener
`
该类实现如下
```java

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
该类的继承关系如下
![ContextLoaderListener](././images/springContextLoader.png)

从如上继承关系，可以看到该类继承了ServletContextListener

ServletContextListener是servlet-api的标准规范接口。
如下是ServletContextListener的定义
```java
public interface ServletContextListener extends EventListener {


    public void contextInitialized(ServletContextEvent sce);


    public void contextDestroyed(ServletContextEvent sce);
}


```

该类继承了
`java.util.EventListener`
定义了contextInitialized,contextDestroyed两个方法，这两个具体的方法由web容器实现方实现。
在tomcat中定义了StandardContext。
StandardContext在org.apache.catalina.core中定义，其定义形式如下：
`


    public boolean listenerStart() {

        if (log.isDebugEnabled())
            log.debug("Configuring application event listeners");

        // Instantiate the required listeners
        String listeners[] = findApplicationListeners();
        Object results[] = new Object[listeners.length];
        boolean ok = true;
        for (int i = 0; i < results.length; i++) {
            if (getLogger().isDebugEnabled())
                getLogger().debug(" Configuring event listener class '" +
                    listeners[i] + "'");
            try {
                String listener = listeners[i];
                results[i] = getInstanceManager().newInstance(listener);
            } catch (Throwable t) {
                t = ExceptionUtils.unwrapInvocationTargetException(t);
                ExceptionUtils.handleThrowable(t);
                getLogger().error(sm.getString(
                        "standardContext.applicationListener", listeners[i]), t);
                ok = false;
            }
        }
        if (!ok) {
            getLogger().error(sm.getString("standardContext.applicationSkipped"));
            return false;
        }

        // Sort listeners in two arrays
        List<Object> eventListeners = new ArrayList<>();
        List<Object> lifecycleListeners = new ArrayList<>();
        for (Object result : results) {
            if ((result instanceof ServletContextAttributeListener)
                    || (result instanceof ServletRequestAttributeListener)
                    || (result instanceof ServletRequestListener)
                    || (result instanceof HttpSessionIdListener)
                    || (result instanceof HttpSessionAttributeListener)) {
                eventListeners.add(result);
            }
            if ((result instanceof ServletContextListener)
                    || (result instanceof HttpSessionListener)) {
                lifecycleListeners.add(result);
            }
        }

        // Listener instances may have been added directly to this Context by
        // ServletContextInitializers and other code via the pluggability APIs.
        // Put them these listeners after the ones defined in web.xml and/or
        // annotations then overwrite the list of instances with the new, full
        // list.
        eventListeners.addAll(Arrays.asList(getApplicationEventListeners()));
        setApplicationEventListeners(eventListeners.toArray());
        for (Object lifecycleListener: getApplicationLifecycleListeners()) {
            lifecycleListeners.add(lifecycleListener);
            if (lifecycleListener instanceof ServletContextListener) {
                noPluggabilityListeners.add(lifecycleListener);
            }
        }
        setApplicationLifecycleListeners(lifecycleListeners.toArray());

        // Send application start events

        if (getLogger().isDebugEnabled())
            getLogger().debug("Sending application start events");

        // Ensure context is not null
        getServletContext();
        context.setNewServletContextListenerAllowed(false);

        Object instances[] = getApplicationLifecycleListeners();
        if (instances == null || instances.length == 0) {
            return ok;
        }

        ServletContextEvent event = new ServletContextEvent(getServletContext());
        ServletContextEvent tldEvent = null;
        if (noPluggabilityListeners.size() > 0) {
            noPluggabilityServletContext = new NoPluggabilityServletContext(getServletContext());
            tldEvent = new ServletContextEvent(noPluggabilityServletContext);
        }
        for (Object instance : instances) {
            if (!(instance instanceof ServletContextListener)) {
                continue;
            }
            ServletContextListener listener = (ServletContextListener) instance;
            try {
                fireContainerEvent("beforeContextInitialized", listener);
                if (noPluggabilityListeners.contains(listener)) {
                    listener.contextInitialized(tldEvent);
                } else {
                    listener.contextInitialized(event);
                }
                fireContainerEvent("afterContextInitialized", listener);
            } catch (Throwable t) {
                ExceptionUtils.handleThrowable(t);
                fireContainerEvent("afterContextInitialized", listener);
                getLogger().error(sm.getString("standardContext.listenerStart",
                        instance.getClass().getName()), t);
                ok = false;
            }
        }
        return ok;
    }


`

在代码中，由如下操作
`
 Object instances[] = getApplicationLifecycleListeners();
`
获取有生命周期的listener实例。
![tomcat 标准容器继承关系](././images/tomcatlifecycleinstances.png)

那么可以得到一个大概的流程关系。
如下图
![Spring创建流程](././images/springcontexcreateflows.png)


### ApplicationContext创建流程分析
    
在ContextLoader有如下API,其他API省略。
```java
public class ContextLoader {

    @Nullable
    private static volatile WebApplicationContext currentContext;
    
    public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
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
            if (this.context == null) {
                this.context = createWebApplicationContext(servletContext);
            }
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
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            } else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            if (logger.isInfoEnabled()) {
                long elapsedTime = System.currentTimeMillis() - startTime;
                logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
            }

            return this.context;
        } catch (RuntimeException | Error ex) {
            logger.error("Context initialization failed", ex);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
            throw ex;
        }
    }
}
```
如上代码大致如下流程
1. createWebApplicationContext
2. 判断当前容器是否有父子关系，如果满足条件，则家在父容器
3. servletContext设置容器
4. 全局共享该容器


在ContextLoader中有如下代码

```java
	protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		Class<?> contextClass = determineContextClass(sc);
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```
可以看到如上的流程，先探测容器类型，然后调用BeanUtils进行实例操作。

```java
	/**
 * Return the WebApplicationContext implementation class to use, either the
 * default XmlWebApplicationContext or a custom context class if specified.
 * @param servletContext current servlet context
 * @return the WebApplicationContext implementation class to use
 * @see #CONTEXT_CLASS_PARAM
 * @see org.springframework.web.context.support.XmlWebApplicationContext
 */
	protected Class<?> determineContextClass(ServletContext servletContext) {
		String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
		if (contextClassName != null) {
			try {
				return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load custom context class [" + contextClassName + "]", ex);
			}
		}
		else {
			contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
			try {
				return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load default context class [" + contextClassName + "]", ex);
			}
		}
	}

```
在如上代码中，获取web.xml Context-param的相关配置，如果contextClass为空，则是默认XmlWebApplicationContext。

#### 默认策略实现
```java
	private static final String DEFAULT_STRATEGIES_PATH = "ContextLoader.properties";

	private static final Properties defaultStrategies;

	static {
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
		}
	}

```

```java
org.springframework.web.context.support.XmlWebApplicationContext
```
如下是XmlWebApplicationContext

```java
public class XmlWebApplicationContext extends AbstractRefreshableWebApplicationContext {

	/** Default config location for the root context. */
	public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";

	/** Default prefix for building a config location for a namespace. */
	public static final String DEFAULT_CONFIG_LOCATION_PREFIX = "/WEB-INF/";

	/** Default suffix for building a config location for a namespace. */
	public static final String DEFAULT_CONFIG_LOCATION_SUFFIX = ".xml";



	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void initBeanDefinitionReader(XmlBeanDefinitionReader beanDefinitionReader) {
	}


	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}


	@Override
	protected String[] getDefaultConfigLocations() {
		if (getNamespace() != null) {
			return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
		}
		else {
			return new String[] {DEFAULT_CONFIG_LOCATION};
		}
	}

}

```
可以看到在
```java
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

        // Allow a subclass to provide custom initialization of the reader,
        // then proceed with actually loading the bean definitions.
        initBeanDefinitionReader(beanDefinitionReader);
        loadBeanDefinitions(beanDefinitionReader);
        }

```
在loadBeanDefinitions中定义了解析bean的相关流程。
那么loadBeanDefinitions在什么地方被引用呢？
下面参考AbstractApplicationContext类的具体实现。

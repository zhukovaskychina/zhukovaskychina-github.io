---
title: Spring的BeanFactory和FactoryBean的关系
date: 2022-07-16 17:14:57
tags:
---

#### Spring的BeanFactory和FactoryBean的关系
之前在脉脉上总是有人抱怨，面试的时候遇到傻逼面试官，喜欢问BeanFactory和FactoryBean之间的关系，如果长时间不关注这块，很容易卡壳，不知道这两者之间的区别。
今天在这里给出这两个之间的差异。
###### BeanFactory的定义
```java
public interface BeanFactory {

    /**
     * Used to dereference a {@link FactoryBean} instance and distinguish it from
     * beans <i>created</i> by the FactoryBean. For example, if the bean named
     * {@code myJndiObject} is a FactoryBean, getting {@code &myJndiObject}
     * will return the factory, not the instance returned by the factory.
     */
    String FACTORY_BEAN_PREFIX = "&";


    /**
     * Return an instance, which may be shared or independent, of the specified bean.
     * <p>This method allows a Spring BeanFactory to be used as a replacement for the
     * Singleton or Prototype design pattern. Callers may retain references to
     * returned objects in the case of Singleton beans.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to retrieve
     * @return an instance of the bean
     * @throws NoSuchBeanDefinitionException if there is no bean with the specified name
     * @throws BeansException if the bean could not be obtained
     */
    Object getBean(String name) throws BeansException;

    /**
     * Return an instance, which may be shared or independent, of the specified bean.
     * <p>Behaves the same as {@link #getBean(String)}, but provides a measure of type
     * safety by throwing a BeanNotOfRequiredTypeException if the bean is not of the
     * required type. This means that ClassCastException can't be thrown on casting
     * the result correctly, as can happen with {@link #getBean(String)}.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to retrieve
     * @param requiredType type the bean must match; can be an interface or superclass
     * @return an instance of the bean
     * @throws NoSuchBeanDefinitionException if there is no such bean definition
     * @throws BeanNotOfRequiredTypeException if the bean is not of the required type
     * @throws BeansException if the bean could not be created
     */
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    /**
     * Return an instance, which may be shared or independent, of the specified bean.
     * <p>Allows for specifying explicit constructor arguments / factory method arguments,
     * overriding the specified default arguments (if any) in the bean definition.
     * @param name the name of the bean to retrieve
     * @param args arguments to use when creating a bean instance using explicit arguments
     * (only applied when creating a new instance as opposed to retrieving an existing one)
     * @return an instance of the bean
     * @throws NoSuchBeanDefinitionException if there is no such bean definition
     * @throws BeanDefinitionStoreException if arguments have been given but
     * the affected bean isn't a prototype
     * @throws BeansException if the bean could not be created
     * @since 2.5
     */
    Object getBean(String name, Object... args) throws BeansException;

    /**
     * Return the bean instance that uniquely matches the given object type, if any.
     * <p>This method goes into {@link ListableBeanFactory} by-type lookup territory
     * but may also be translated into a conventional by-name lookup based on the name
     * of the given type. For more extensive retrieval operations across sets of beans,
     * use {@link ListableBeanFactory} and/or {@link BeanFactoryUtils}.
     * @param requiredType type the bean must match; can be an interface or superclass
     * @return an instance of the single bean matching the required type
     * @throws NoSuchBeanDefinitionException if no bean of the given type was found
     * @throws NoUniqueBeanDefinitionException if more than one bean of the given type was found
     * @throws BeansException if the bean could not be created
     * @since 3.0
     * @see ListableBeanFactory
     */
    <T> T getBean(Class<T> requiredType) throws BeansException;

    /**
     * Return an instance, which may be shared or independent, of the specified bean.
     * <p>Allows for specifying explicit constructor arguments / factory method arguments,
     * overriding the specified default arguments (if any) in the bean definition.
     * <p>This method goes into {@link ListableBeanFactory} by-type lookup territory
     * but may also be translated into a conventional by-name lookup based on the name
     * of the given type. For more extensive retrieval operations across sets of beans,
     * use {@link ListableBeanFactory} and/or {@link BeanFactoryUtils}.
     * @param requiredType type the bean must match; can be an interface or superclass
     * @param args arguments to use when creating a bean instance using explicit arguments
     * (only applied when creating a new instance as opposed to retrieving an existing one)
     * @return an instance of the bean
     * @throws NoSuchBeanDefinitionException if there is no such bean definition
     * @throws BeanDefinitionStoreException if arguments have been given but
     * the affected bean isn't a prototype
     * @throws BeansException if the bean could not be created
     * @since 4.1
     */
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    /**
     * Return a provider for the specified bean, allowing for lazy on-demand retrieval
     * of instances, including availability and uniqueness options.
     * @param requiredType type the bean must match; can be an interface or superclass
     * @return a corresponding provider handle
     * @since 5.1
     * @see #getBeanProvider(ResolvableType)
     */
    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    /**
     * Return a provider for the specified bean, allowing for lazy on-demand retrieval
     * of instances, including availability and uniqueness options.
     * @param requiredType type the bean must match; can be a generic type declaration.
     * Note that collection types are not supported here, in contrast to reflective
     * injection points. For programmatically retrieving a list of beans matching a
     * specific type, specify the actual bean type as an argument here and subsequently
     * use {@link ObjectProvider#orderedStream()} or its lazy streaming/iteration options.
     * @return a corresponding provider handle
     * @since 5.1
     * @see ObjectProvider#iterator()
     * @see ObjectProvider#stream()
     * @see ObjectProvider#orderedStream()
     */
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    /**
     * Does this bean factory contain a bean definition or externally registered singleton
     * instance with the given name?
     * <p>If the given name is an alias, it will be translated back to the corresponding
     * canonical bean name.
     * <p>If this factory is hierarchical, will ask any parent factory if the bean cannot
     * be found in this factory instance.
     * <p>If a bean definition or singleton instance matching the given name is found,
     * this method will return {@code true} whether the named bean definition is concrete
     * or abstract, lazy or eager, in scope or not. Therefore, note that a {@code true}
     * return value from this method does not necessarily indicate that {@link #getBean}
     * will be able to obtain an instance for the same name.
     * @param name the name of the bean to query
     * @return whether a bean with the given name is present
     */
    boolean containsBean(String name);

    /**
     * Is this bean a shared singleton? That is, will {@link #getBean} always
     * return the same instance?
     * <p>Note: This method returning {@code false} does not clearly indicate
     * independent instances. It indicates non-singleton instances, which may correspond
     * to a scoped bean as well. Use the {@link #isPrototype} operation to explicitly
     * check for independent instances.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @return whether this bean corresponds to a singleton instance
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @see #getBean
     * @see #isPrototype
     */
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

    /**
     * Is this bean a prototype? That is, will {@link #getBean} always return
     * independent instances?
     * <p>Note: This method returning {@code false} does not clearly indicate
     * a singleton object. It indicates non-independent instances, which may correspond
     * to a scoped bean as well. Use the {@link #isSingleton} operation to explicitly
     * check for a shared singleton instance.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @return whether this bean will always deliver independent instances
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @since 2.0.3
     * @see #getBean
     * @see #isSingleton
     */
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    /**
     * Check whether the bean with the given name matches the specified type.
     * More specifically, check whether a {@link #getBean} call for the given name
     * would return an object that is assignable to the specified target type.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @param typeToMatch the type to match against (as a {@code ResolvableType})
     * @return {@code true} if the bean type matches,
     * {@code false} if it doesn't match or cannot be determined yet
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @since 4.2
     * @see #getBean
     * @see #getType
     */
    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

    /**
     * Check whether the bean with the given name matches the specified type.
     * More specifically, check whether a {@link #getBean} call for the given name
     * would return an object that is assignable to the specified target type.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @param typeToMatch the type to match against (as a {@code Class})
     * @return {@code true} if the bean type matches,
     * {@code false} if it doesn't match or cannot be determined yet
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @since 2.0.1
     * @see #getBean
     * @see #getType
     */
    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    /**
     * Determine the type of the bean with the given name. More specifically,
     * determine the type of object that {@link #getBean} would return for the given name.
     * <p>For a {@link FactoryBean}, return the type of object that the FactoryBean creates,
     * as exposed by {@link FactoryBean#getObjectType()}. This may lead to the initialization
     * of a previously uninitialized {@code FactoryBean} (see {@link #getType(String, boolean)}).
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @return the type of the bean, or {@code null} if not determinable
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @since 1.1.2
     * @see #getBean
     * @see #isTypeMatch
     */
    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    /**
     * Determine the type of the bean with the given name. More specifically,
     * determine the type of object that {@link #getBean} would return for the given name.
     * <p>For a {@link FactoryBean}, return the type of object that the FactoryBean creates,
     * as exposed by {@link FactoryBean#getObjectType()}. Depending on the
     * {@code allowFactoryBeanInit} flag, this may lead to the initialization of a previously
     * uninitialized {@code FactoryBean} if no early type information is available.
     * <p>Translates aliases back to the corresponding canonical bean name.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the name of the bean to query
     * @param allowFactoryBeanInit whether a {@code FactoryBean} may get initialized
     * just for the purpose of determining its object type
     * @return the type of the bean, or {@code null} if not determinable
     * @throws NoSuchBeanDefinitionException if there is no bean with the given name
     * @since 5.2
     * @see #getBean
     * @see #isTypeMatch
     */
    @Nullable
    Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

    /**
     * Return the aliases for the given bean name, if any.
     * <p>All of those aliases point to the same bean when used in a {@link #getBean} call.
     * <p>If the given name is an alias, the corresponding original bean name
     * and other aliases (if any) will be returned, with the original bean name
     * being the first element in the array.
     * <p>Will ask the parent factory if the bean cannot be found in this factory instance.
     * @param name the bean name to check for aliases
     * @return the aliases, or an empty array if none
     * @see #getBean
     */
    String[] getAliases(String name);

}


```

###### FactoryBean的定义
```java
public interface FactoryBean<T> {

	/**
	 * The name of an attribute that can be
	 * {@link org.springframework.core.AttributeAccessor#setAttribute set} on a
	 * {@link org.springframework.beans.factory.config.BeanDefinition} so that
	 * factory beans can signal their object type when it can't be deduced from
	 * the factory bean class.
	 * @since 5.2
	 */
	String OBJECT_TYPE_ATTRIBUTE = "factoryBeanObjectType";


	/**
	 * Return an instance (possibly shared or independent) of the object
	 * managed by this factory.
	 * <p>As with a {@link BeanFactory}, this allows support for both the
	 * Singleton and Prototype design pattern.
	 * <p>If this FactoryBean is not fully initialized yet at the time of
	 * the call (for example because it is involved in a circular reference),
	 * throw a corresponding {@link FactoryBeanNotInitializedException}.
	 * <p>As of Spring 2.0, FactoryBeans are allowed to return {@code null}
	 * objects. The factory will consider this as normal value to be used; it
	 * will not throw a FactoryBeanNotInitializedException in this case anymore.
	 * FactoryBean implementations are encouraged to throw
	 * FactoryBeanNotInitializedException themselves now, as appropriate.
	 * @return an instance of the bean (can be {@code null})
	 * @throws Exception in case of creation errors
	 * @see FactoryBeanNotInitializedException
	 */
	@Nullable
	T getObject() throws Exception;

	/**
	 * Return the type of object that this FactoryBean creates,
	 * or {@code null} if not known in advance.
	 * <p>This allows one to check for specific types of beans without
	 * instantiating objects, for example on autowiring.
	 * <p>In the case of implementations that are creating a singleton object,
	 * this method should try to avoid singleton creation as far as possible;
	 * it should rather estimate the type in advance.
	 * For prototypes, returning a meaningful type here is advisable too.
	 * <p>This method can be called <i>before</i> this FactoryBean has
	 * been fully initialized. It must not rely on state created during
	 * initialization; of course, it can still use such state if available.
	 * <p><b>NOTE:</b> Autowiring will simply ignore FactoryBeans that return
	 * {@code null} here. Therefore it is highly recommended to implement
	 * this method properly, using the current state of the FactoryBean.
	 * @return the type of object that this FactoryBean creates,
	 * or {@code null} if not known at the time of the call
	 * @see ListableBeanFactory#getBeansOfType
	 */
	@Nullable
	Class<?> getObjectType();

	/**
	 * Is the object managed by this factory a singleton? That is,
	 * will {@link #getObject()} always return the same object
	 * (a reference that can be cached)?
	 * <p><b>NOTE:</b> If a FactoryBean indicates to hold a singleton object,
	 * the object returned from {@code getObject()} might get cached
	 * by the owning BeanFactory. Hence, do not return {@code true}
	 * unless the FactoryBean always exposes the same reference.
	 * <p>The singleton status of the FactoryBean itself will generally
	 * be provided by the owning BeanFactory; usually, it has to be
	 * defined as singleton there.
	 * <p><b>NOTE:</b> This method returning {@code false} does not
	 * necessarily indicate that returned objects are independent instances.
	 * An implementation of the extended {@link SmartFactoryBean} interface
	 * may explicitly indicate independent instances through its
	 * {@link SmartFactoryBean#isPrototype()} method. Plain {@link FactoryBean}
	 * implementations which do not implement this extended interface are
	 * simply assumed to always return independent instances if the
	 * {@code isSingleton()} implementation returns {@code false}.
	 * <p>The default implementation returns {@code true}, since a
	 * {@code FactoryBean} typically manages a singleton instance.
	 * @return whether the exposed object is a singleton
	 * @see #getObject()
	 * @see SmartFactoryBean#isPrototype()
	 */
	default boolean isSingleton() {
		return true;
	}

}



```

如上所示，BeanFactory和FactoryBean在接口的定义上，有很大的差别。BeanFactory定义了Bean容器的获取行为如
</br>
`<T> T getBean(String name, Class<T> requiredType)`</br>
`boolean isSingleton(String name)` </br>
`boolean isPrototype(String name)`</br>
等行为。而FactoryBean则是另外与之不同的行为。如下所示
</br>
```
@Nullable
T getObject() throws Exception;
```
则是返回由该工厂管理的bean实例对象。
由此可见，BeanFactory是定义了容器行为的接口，而FactoryBean则是工厂bean的接口定义。

###### beanFactory和factoryBean的实现类

###### beanFactory类继承关系图

![beanFactory类继承关系图](././images/beanfactory.png)


###### factoryBean类继承关系图
![factoryBean类继承关系图](././images/springfactorybean.png)


##### 总结
在Spring中,所有的Bean都是由BeanFactory（也就是IoC容器）来进行管理的。
但对FactoryBean而言,这个Bean不是简单的Bean,而是一个能产生或者修饰对象生成的工厂Bean,
它的实现与设计模式中的工厂模式和修饰器模式类似。
用户使用容器时,可以使用转义符“＆”来得到FactoryBean本身,用来区分获取FactoryBean产生的对象和获取FactoryBean本身。


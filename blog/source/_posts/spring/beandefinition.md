---
title: beandefinition的分析
date: 2022-07-16 20:57:54
tags: spring
type: "spring"
description: "BeanDefinition是Spring IOC 容器管理Bean对象的元信息，理解BeanDefinition对理解Spring的容器行为有很大的帮助。"
---


### BeanDefinition 的接口定义

在Spring包
`org.springframework.beans.factory.config`
下定义了如下类
```java

public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {


	String SCOPE_SINGLETON = ConfigurableBeanFactory.SCOPE_SINGLETON;


	String SCOPE_PROTOTYPE = ConfigurableBeanFactory.SCOPE_PROTOTYPE;
    
	int ROLE_APPLICATION = 0;


	int ROLE_SUPPORT = 1;

	int ROLE_INFRASTRUCTURE = 2;
    
	void setParentName(@Nullable String parentName);

	@Nullable
	String getParentName();
    
	void setBeanClassName(@Nullable String beanClassName);
    
	void setScope(@Nullable String scope);

	void setLazyInit(boolean lazyInit);
    
	void setDependsOn(@Nullable String... dependsOn);
    
	void setAutowireCandidate(boolean autowireCandidate);
    
	void setPrimary(boolean primary);
    
	void setFactoryBeanName(@Nullable String factoryBeanName);
    
	void setFactoryMethodName(@Nullable String factoryMethodName);
    
    
	ConstructorArgumentValues getConstructorArgumentValues();

	/**
	 * Return if there are constructor argument values defined for this bean.
	 * @since 5.0.2
	 */
	default boolean hasConstructorArgumentValues() {
		return !getConstructorArgumentValues().isEmpty();
	}


	MutablePropertyValues getPropertyValues();


	default boolean hasPropertyValues() {
		return !getPropertyValues().isEmpty();
	}


	void setInitMethodName(@Nullable String initMethodName);



	void setDestroyMethodName(@Nullable String destroyMethodName);
    
	void setRole(int role);
    
	void setDescription(@Nullable String description);
    
	ResolvableType getResolvableType();

	boolean isSingleton();


	boolean isPrototype();


	boolean isAbstract();
	@Nullable
	BeanDefinition getOriginatingBeanDefinition();

}

```


和这个类相关的有
* @see org.springframework.beans.factory.support.RootBeanDefinition
* @see org.springframework.beans.factory.support.ChildBeanDefinition
* @see org.springframework.beans.factory.config.ConfigurableListableBeanFactory

在如上的BeanDefinition中定义了bean的相关行为如
primary,dependsOn,autowiredCandidate,factoryBean,initMethodName,despryMethodName,isSingleton,isPrototype
等方法属性。那么它在什么地方实现了呢？
![beanDefinition的实现类](././images/beandefinitions.png)
参考如上的beanDefinition类继承关系图，我们可以看到


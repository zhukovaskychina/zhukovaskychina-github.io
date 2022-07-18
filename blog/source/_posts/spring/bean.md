---
title: bean的分析
date: 2022-07-16 20:57:54
tags: spring
type: "spring"
---

## bean的分析

```text
IoC 也称为依赖关系注入 （DI）。这是一个过程，其中对象仅通过构造函数参数、工厂方法的参数或在对象实例构造
或从工厂方法返回后在对象实例上设置的属性来定义其依赖项（即它们使用的其他对象）。
然后，容器在创建 Bean 时注入这些依赖项。这个过程基本上是Bean本身的反演（因此得名Inversion of Control），
通过使用类的直接构造或诸如Service Locator模式之类的机制来控制其依赖项的实例化或位置。

和 包是Spring Framework的IoC容器的基础。

在 Spring 中，构成应用程序主干并由 Spring IoC 容器管理的对象称为 Bean。Bean 
是由 Spring IoC 容器实例化、组装和以其他方式管理的对象。否则，Bean 只是应用程序中众多对象之一。
Bean 以及它们之间的依赖关系反映在容器使用的配置元数据中。
```


### bean的代码定义

```java
public @interface Bean {
    
	@AliasFor("name")
	String[] value() default {};


	@AliasFor("value")
	String[] name() default {};

    
	boolean autowireCandidate() default true;


	String initMethod() default "";


	String destroyMethod() default AbstractBeanDefinition.INFER_METHOD;

}

```
在注解@Bean定义了value,name,autowireCandidate(),initMethod,destoryMethod()等方法。

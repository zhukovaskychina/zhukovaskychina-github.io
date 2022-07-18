---
title: transaction的分析
date: 2022-07-16 20:57:54
tags: spring
type: "spring"
---

### Transaction的代码分析

出去面试的时候，总是有一群傻逼面试官喜欢问Spring的事务有没有用过。今天这里对@Transactional进行一次源码分析。

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {


	@AliasFor("transactionManager")
	String value() default "";

	@AliasFor("value")
	String transactionManager() default "";


	Propagation propagation() default Propagation.REQUIRED;


	Isolation isolation() default Isolation.DEFAULT;


	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;


	boolean readOnly() default false;


	Class<? extends Throwable>[] rollbackFor() default {};


	String[] rollbackForClassName() default {};

	Class<? extends Throwable>[] noRollbackFor() default {};


	String[] noRollbackForClassName() default {};

}

```


Transactional注解的实现，是由TransactionAnnotationParser来实现的。

```java
public interface TransactionAnnotationParser {
    default boolean isCandidateClass(Class<?> targetClass) {
        return true;
    }
    
    @Nullable
    TransactionAttribute parseTransactionAnnotation(AnnotatedElement element);
}
```

其调用逻辑
```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
			TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {

		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource);
		advisor.setAdvice(transactionInterceptor);
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		interceptor.setTransactionAttributeSource(transactionAttributeSource);
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}

```
在Spring-tx包下有如上类ProxyTransactionManagementConfiguration，在这里做了事务管理的配置管理。
```java
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}
```
下面来看下AnnotationTransactionAttributeSource的实现。
![事务管理注解的继承](././images/transaction.png)

查看该类的具体实现。
```java
public class AnnotationTransactionAttributeSource extends AbstractFallbackTransactionAttributeSource
		implements Serializable {

	private static final boolean jta12Present;

	private static final boolean ejb3Present;

	static {
		ClassLoader classLoader = AnnotationTransactionAttributeSource.class.getClassLoader();
		jta12Present = ClassUtils.isPresent("javax.transaction.Transactional", classLoader);
		ejb3Present = ClassUtils.isPresent("javax.ejb.TransactionAttribute", classLoader);
	}

	private final boolean publicMethodsOnly;

	private final Set<TransactionAnnotationParser> annotationParsers;




	public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
		this.publicMethodsOnly = publicMethodsOnly;
		if (jta12Present || ejb3Present) {
			this.annotationParsers = new LinkedHashSet<>(4);
			this.annotationParsers.add(new SpringTransactionAnnotationParser());
			if (jta12Present) {
				this.annotationParsers.add(new JtaTransactionAnnotationParser());
			}
			if (ejb3Present) {
				this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
			}
		}
		else {
			this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
		}
	}

	/**
	 * Create a custom AnnotationTransactionAttributeSource.
	 * @param annotationParser the TransactionAnnotationParser to use
	 */
	public AnnotationTransactionAttributeSource(TransactionAnnotationParser annotationParser) {
		this.publicMethodsOnly = true;
		Assert.notNull(annotationParser, "TransactionAnnotationParser must not be null");
		this.annotationParsers = Collections.singleton(annotationParser);
	}

	/**
	 * Create a custom AnnotationTransactionAttributeSource.
	 * @param annotationParsers the TransactionAnnotationParsers to use
	 */
	public AnnotationTransactionAttributeSource(TransactionAnnotationParser... annotationParsers) {
		this.publicMethodsOnly = true;
		Assert.notEmpty(annotationParsers, "At least one TransactionAnnotationParser needs to be specified");
		this.annotationParsers = new LinkedHashSet<>(Arrays.asList(annotationParsers));
	}

	/**
	 * Create a custom AnnotationTransactionAttributeSource.
	 * @param annotationParsers the TransactionAnnotationParsers to use
	 */
	public AnnotationTransactionAttributeSource(Set<TransactionAnnotationParser> annotationParsers) {
		this.publicMethodsOnly = true;
		Assert.notEmpty(annotationParsers, "At least one TransactionAnnotationParser needs to be specified");
		this.annotationParsers = annotationParsers;
	}


	@Override
	public boolean isCandidateClass(Class<?> targetClass) {
		for (TransactionAnnotationParser parser : this.annotationParsers) {
			if (parser.isCandidateClass(targetClass)) {
				return true;
			}
		}
		return false;
	}

	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Class<?> clazz) {
		return determineTransactionAttribute(clazz);
	}

	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Method method) {
		return determineTransactionAttribute(method);
	}


	@Nullable
	protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
		for (TransactionAnnotationParser parser : this.annotationParsers) {
			TransactionAttribute attr = parser.parseTransactionAnnotation(element);
			if (attr != null) {
				return attr;
			}
		}
		return null;
	}

	/**
	 * By default, only public methods can be made transactional.
	 */
	@Override
	protected boolean allowPublicMethodsOnly() {
		return this.publicMethodsOnly;
	}

}

```
看到如上代码，在构造器中有如下代码
```java
	public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
		this.publicMethodsOnly = publicMethodsOnly;
		if (jta12Present || ejb3Present) {
			this.annotationParsers = new LinkedHashSet<>(4);
			this.annotationParsers.add(new SpringTransactionAnnotationParser());
			if (jta12Present) {
				this.annotationParsers.add(new JtaTransactionAnnotationParser());
			}
			if (ejb3Present) {
				this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
			}
		}
		else {
			this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
		}
	}

```

除了jta2和ejb3外，还有Spring事务管理器。
在this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());

看下SpringTransactionAnnotationParser的具体实现类。


```java
public class SpringTransactionAnnotationParser implements TransactionAnnotationParser, Serializable {

	@Override
	public boolean isCandidateClass(Class<?> targetClass) {
		return AnnotationUtils.isCandidateClass(targetClass, Transactional.class);
	}

	@Override
	@Nullable
	public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
				element, Transactional.class, false, false);
		if (attributes != null) {
			return parseTransactionAnnotation(attributes);
		}
		else {
			return null;
		}
	}

	public TransactionAttribute parseTransactionAnnotation(Transactional ann) {
		return parseTransactionAnnotation(AnnotationUtils.getAnnotationAttributes(ann, false, false));
	}

	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());
		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));

		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
}

```

如上图所示，该类实现了TransactionAnnotationParser接口，实现了对@Transacational注解的解析。
那么在哪里调用呢？
在 org.springframework.transaction.interceptor下面定义了如下接口
```java
public interface TransactionAttributeSource {
    
	default boolean isCandidateClass(Class<?> targetClass) {
		return true;
	}
    
	@Nullable
	TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass);

}

```


### spring transaction aspect
在spring-aspect中有如下代码：
```aspectj
public abstract aspect AbstractTransactionAspect extends TransactionAspectSupport implements DisposableBean {

	/**
	 * Construct the aspect using the given transaction metadata retrieval strategy.
	 * @param tas TransactionAttributeSource implementation, retrieving Spring
	 * transaction metadata for each joinpoint. Implement the subclass to pass in
	 * {@code null} if it is intended to be configured through Setter Injection.
	 */
	protected AbstractTransactionAspect(TransactionAttributeSource tas) {
		setTransactionAttributeSource(tas);
	}

	@Override
	public void destroy() {
		// An aspect is basically a singleton -> cleanup on destruction
		clearTransactionManagerCache();
	}

	@SuppressAjWarnings("adviceDidNotMatch")
	Object around(final Object txObject): transactionalMethodExecution(txObject) {
		MethodSignature methodSignature = (MethodSignature) thisJoinPoint.getSignature();
		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		try {
			return invokeWithinTransaction(methodSignature.getMethod(), txObject.getClass(), new InvocationCallback() {
				public Object proceedWithInvocation() throws Throwable {
					return proceed(txObject);
				}
			});
		}
		catch (RuntimeException | Error ex) {
			throw ex;
		}
		catch (Throwable thr) {
			Rethrower.rethrow(thr);
			throw new IllegalStateException("Should never get here", thr);
		}
	}

	/**
	 * Concrete subaspects must implement this pointcut, to identify
	 * transactional methods. For each selected joinpoint, TransactionMetadata
	 * will be retrieved using Spring's TransactionAttributeSource interface.
	 */
	protected abstract pointcut transactionalMethodExecution(Object txObject);


	/**
	 * Ugly but safe workaround: We need to be able to propagate checked exceptions,
	 * despite AspectJ around advice supporting specifically declared exceptions only.
	 */
	private static class Rethrower {

		public static void rethrow(final Throwable exception) {
			class CheckedExceptionRethrower<T extends Throwable> {
				@SuppressWarnings("unchecked")
				private void rethrow(Throwable exception) throws T {
					throw (T) exception;
				}
			}
			new CheckedExceptionRethrower<RuntimeException>().rethrow(exception);
		}
	}

}

```
总是有些傻逼面试官问Spring Tx RuntimeException会不会有异常。
```java
	@SuppressAjWarnings("adviceDidNotMatch")
	Object around(final Object txObject): transactionalMethodExecution(txObject) {
		MethodSignature methodSignature = (MethodSignature) thisJoinPoint.getSignature();
		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		try {
			return invokeWithinTransaction(methodSignature.getMethod(), txObject.getClass(), new InvocationCallback() {
				public Object proceedWithInvocation() throws Throwable {
					return proceed(txObject);
				}
			});
		}
		catch (RuntimeException | Error ex) {
			throw ex;
		}
		catch (Throwable thr) {
			Rethrower.rethrow(thr);
			throw new IllegalStateException("Should never get here", thr);
		}
	}

```

可以看到，当系统抛RuntimeException的时候，事务会失效。

在TransactionAspectSupport中invokeWithinTransaction
做了如下处理
```java

public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean{
    @Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
                                             final InvocationCallback invocation) throws Throwable {

        // If the transaction attribute is null, the method is non-transactional.
        TransactionAttributeSource tas = getTransactionAttributeSource();
        final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
        final TransactionManager tm = determineTransactionManager(txAttr);

        if (this.reactiveAdapterRegistry != null && tm instanceof ReactiveTransactionManager) {
            ReactiveTransactionSupport txSupport = this.transactionSupportCache.computeIfAbsent(method, key -> {
                if (KotlinDetector.isKotlinType(method.getDeclaringClass()) && KotlinDelegate.isSuspend(method)) {
                    throw new TransactionUsageException(
                            "Unsupported annotated transaction on suspending function detected: " + method +
                                    ". Use TransactionalOperator.transactional extensions instead.");
                }
                ReactiveAdapter adapter = this.reactiveAdapterRegistry.getAdapter(method.getReturnType());
                if (adapter == null) {
                    throw new IllegalStateException("Cannot apply reactive transaction to non-reactive return type: " +
                            method.getReturnType());
                }
                return new ReactiveTransactionSupport(adapter);
            });
            return txSupport.invokeWithinTransaction(
                    method, targetClass, invocation, txAttr, (ReactiveTransactionManager) tm);
        }

        PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
        final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

        if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
            // Standard transaction demarcation with getTransaction and commit/rollback calls.
            TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

            Object retVal;
            try {
                // This is an around advice: Invoke the next interceptor in the chain.
                // This will normally result in a target object being invoked.
                retVal = invocation.proceedWithInvocation();
            }
            catch (Throwable ex) {
                // target invocation exception
                completeTransactionAfterThrowing(txInfo, ex);
                throw ex;
            }
            finally {
                cleanupTransactionInfo(txInfo);
            }

            if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                // Set rollback-only in case of Vavr failure matching our rollback rules...
                TransactionStatus status = txInfo.getTransactionStatus();
                if (status != null && txAttr != null) {
                    retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                }
            }

            commitTransactionAfterReturning(txInfo);
            return retVal;
        }

        else {
            Object result;
            final ThrowableHolder throwableHolder = new ThrowableHolder();

            // It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
            try {
                result = ((CallbackPreferringPlatformTransactionManager) ptm).execute(txAttr, status -> {
                    TransactionInfo txInfo = prepareTransactionInfo(ptm, txAttr, joinpointIdentification, status);
                    try {
                        Object retVal = invocation.proceedWithInvocation();
                        if (retVal != null && vavrPresent && VavrDelegate.isVavrTry(retVal)) {
                            // Set rollback-only in case of Vavr failure matching our rollback rules...
                            retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
                        }
                        return retVal;
                    }
                    catch (Throwable ex) {
                        if (txAttr.rollbackOn(ex)) {
                            // A RuntimeException: will lead to a rollback.
                            if (ex instanceof RuntimeException) {
                                throw (RuntimeException) ex;
                            }
                            else {
                                throw new ThrowableHolderException(ex);
                            }
                        }
                        else {
                            // A normal return value: will lead to a commit.
                            throwableHolder.throwable = ex;
                            return null;
                        }
                    }
                    finally {
                        cleanupTransactionInfo(txInfo);
                    }
                });
            }
            catch (ThrowableHolderException ex) {
                throw ex.getCause();
            }
            catch (TransactionSystemException ex2) {
                if (throwableHolder.throwable != null) {
                    logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
                    ex2.initApplicationException(throwableHolder.throwable);
                }
                throw ex2;
            }
            catch (Throwable ex2) {
                if (throwableHolder.throwable != null) {
                    logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
                }
                throw ex2;
            }

            // Check result state: It might indicate a Throwable to rethrow.
            if (throwableHolder.throwable != null) {
                throw throwableHolder.throwable;
            }
            return result;
        }
    }

}




```
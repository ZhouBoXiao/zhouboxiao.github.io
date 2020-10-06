---
layout:     post
title:      "Spring相关知识"
subtitle:   "待进一步完善"
date:       2020-04-06
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - Spring
---

## SpringBoot

### 概念

`Spring Boot`基本上是`Spring`框架的扩展，它消除了设置`Spring`应用程序所需的`XML配置`，为更快，更高效的开发生态系统铺平了道路。

1. 创建独立的`Spring`应用。

2. 嵌入式`Tomcat`、`Jetty`、 `Undertow`容器（无需部署war文件）。

3. 提供的`starters` 简化构建配置

4. 尽可能自动配置`spring`应用。

5. 提供生产指标,例如指标、健壮检查和外部化配置

6. 完全没有代码生成和`XML`配置要求

### 优势

在部署环境中`Spring Boot` 对比`Spring`的一些优点包括：

- 提供嵌入式容器支持
- 使用命令*java -jar*独立运行jar
- 在外部容器中部署时，可以选择排除依赖关系以避免潜在的jar冲突
- 部署时灵活指定配置文件的选项
- 用于集成测试的随机端口生成

### 启动流程

1. 获取并创建SpringApplicationRunListener
2. 创建参数，配置Environment
3. 创建ApplicationContext
4. 初始化ApplicationContext，设置Environment加载相关配置等
5. refresh ApplicationContext
6. 完成最终的程序的启动

## Spring 事务管理实现方式

- **编程式事务管理**
  - 编程式事务管理通过`TransactionTemplate`手动管理事务，调用`beginTransaction()`、`commit()`、`rollback()`等事务管理相关的方法

- **声明式事务管理**
  - 声明式事务管理有三种实现方式：**基于`TransactionProxyFactoryBean`的方式**、**基于AspectJ的XML方式**、**基于@Transactional注解的方式**

## Autowired
### 概念
- 首先介绍依赖注入，就是利用反射机制为类的属性赋值的操作。
- 注入某个对象所需的外部资源（包括对象、资源、常量数据等）。IoC容器注入应用程序的某个对象，应用程序所依赖的对象。
- 在完成对象的创建，为对象变量进行赋值的时候进行注入
### 方式
- no：不进行自动装配，手动设置Bean的依赖关系。 
- byName：根据Bean的名字进行自动装配。 
- byType：根据Bean的类型进行自动装配。 
- constructor：类似于byType，不过是应用于构造器的参数，如果正好有一个Bean与构造器的参数类型相同则可以自动装配，否则会导致错误。 
- autodetect：如果有默认的构造器，则通过constructor的方式进行自动装配，否则使用byType的方式进行自动装配。
### 原理

1. 获取@Autowired数据

   1. 实现主要通过`AutowiredAnnotationBeanPostProcessor`后置处理器，

2. 注入Autowired数据

   - 当beanName为controller时，会进入postProcessProperties方法

   - ```java
         @Override
     	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
     		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
     		try {
     			metadata.inject(bean, beanName, pvs);
     		}
     		catch (BeanCreationException ex) {
     			throw ex;
     		}
     		catch (Throwable ex) {
     			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
     		}
     		return pvs;
     	}
     ```

   - 在这个方法中，第一步还是获取了要注入的元数据，findAutowiringMetadata这个方法，我们在第一步时，已经见过，只是这里获取metadata是从缓存获取的，获取metadata后，便开始反射执行对应的bean和properties了。

   - 进入metadata.inject(bean, beanName, pvs)方法，执行element.inject(target, beanName, pvs)方法，

   - ```java
     public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
     	Collection<InjectedElement> checkedElements = this.checkedElements;
     	Collection<InjectedElement> elementsToIterate =
     			(checkedElements != null ? checkedElements : this.injectedElements);
     	if (!elementsToIterate.isEmpty()) {
     		for (InjectedElement element : elementsToIterate) {
     			if (logger.isTraceEnabled()) {
     				logger.trace("Processing injected element of bean '" + beanName + "': " + element);
     			}
     			// 注入到目标bean
     			element.inject(target, beanName, pvs);
     		}
     	}
     }
     ```

   - 进入element的inject方法（AutowiredAnnotationBeanPostProcessor的inject方法），获取到要注入的数据，最后，ReflectionUtils.makeAccessible(field);field.set(bean, value);进行了真正的反射注入操作。

## Bean的作用域与生命周期

在Spring中，那些组成应用程序的主体及由Spring IOC容器所管理的对象，被称之为Bean。

| 类别          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| singleton     | IOC容器中仅存在一个Bean实例，单例方式存在                    |
| prototype     | 每次从容器调用Bean时，都返回一个新的实例                     |
| request       | 每次HTTP请求都会创建一个新的Bean，该作用域仅适用于WebApplicationContext环境 |
| seesion       | 同一个HTTP Session共享一个Bean，不同Session使用不同Bean，仅适用于WebApplicationContext环境 |
| globalSession | 一般用于Portlet应用环境，仅适用于WebApplicationContext环境   |

### 生命周期

整个执行过程描述如下：

1）扫描类invokeBeanFactoryProcessors，封装BeanDefinition对象各种信息，

​      根据配置情况调用 Bean 构造方法或工厂方法实例化 Bean。

2）利用依赖注入完成 Bean 中所有属性值的配置注入。（注入属性要判断是否有循环依赖）

3）如果 Bean 实现了 `BeanNameAware` 接口，则 Spring 调用 Bean 的 `setBeanName()` 方法传入当前 Bean 的 id 值。

4）如果 Bean 实现了 `BeanFactoryAware` 接口，则 Spring 调用 `setBeanFactory()` 方法传入当前工厂实例的引用。

5）如果 Bean 实现了 `ApplicationContextAware` 接口，则 Spring 调用 `setApplicationContext()` 方法传入当前 `ApplicationContext` 实例的引用。

6）如果 `BeanPostProcessor` 和 Bean 关联，则 Spring 将调用该接口的预初始化方法 `postProcessBeforeInitialzation()` 对 Bean 进行加工操作，此处非常重要，Spring 的 AOP 就是利用它实现的。

7）如果 Bean 实现了 `InitializingBean` 接口，则 Spring 将调用 `afterPropertiesSet()` 方法。

8）如果在配置文件中通过 `init-method` 属性指定了初始化方法，则调用该初始化方法。

9）如果 `BeanPostProcessor` 和 Bean 关联，则 Spring 将调用该接口的初始化方法 `postProcessAfterInitialization()`。此时，Bean 已经可以被应用系统使用了。

10）如果在\<bean> 中指定了该 Bean 的作用范围为 scope="singleton"，则将该 Bean 放入 Spring IoC 的缓存池中，将触发 Spring 对该 Bean 的生命周期管理；如果在 \<bean>中指定了该 Bean 的作用范围为 scope="prototype"，则将该 Bean 交给调用者，调用者管理该 Bean 的生命周期，Spring 不再管理该 Bean。

11）如果 Bean 实现了 `DisposableBean` 接口，则 Spring 会调用 `destory()` 方法将 Spring 中的 Bean 销毁；如果在配置文件中通过 destory-method 属性指定了 Bean 的销毁方法，则 Spring 将调用该方法对 Bean 进行销毁。

## Spring 事务

### 事务管理接口

事务管理相关最重要的 3 个接口如下：

- **`PlatformTransactionManager`**： （平台）事务管理器，Spring 事务策略的核心。

  - ```java
    public interface PlatformTransactionManager {
        //获得事务
        TransactionStatus getTransaction(@Nullable TransactionDefinition var1) throws TransactionException;
        //提交事务
        void commit(TransactionStatus var1) throws TransactionException;
        //回滚事务
        void rollback(TransactionStatus var1) throws TransactionException;
    }
    ```

- **`TransactionDefinition`**： 事务定义信息(事务隔离级别、传播行为、超时、只读、回滚规则)。

  - ```java
    public interface TransactionDefinition {
        int PROPAGATION_REQUIRED = 0;
        int PROPAGATION_SUPPORTS = 1;
        int PROPAGATION_MANDATORY = 2;
        int PROPAGATION_REQUIRES_NEW = 3;
        int PROPAGATION_NOT_SUPPORTED = 4;
        int PROPAGATION_NEVER = 5;
        int PROPAGATION_NESTED = 6;
        int ISOLATION_DEFAULT = -1;
        int ISOLATION_READ_UNCOMMITTED = 1;
        int ISOLATION_READ_COMMITTED = 2;
        int ISOLATION_REPEATABLE_READ = 4;
        int ISOLATION_SERIALIZABLE = 8;
        int TIMEOUT_DEFAULT = -1;
        // 返回事务的传播行为，默认值为 REQUIRED。
        int getPropagationBehavior();
        //返回事务的隔离级别，默认值是 DEFAULT
        int getIsolationLevel();
        // 返回事务的超时时间，默认值为-1。如果超过该时间限制但事务还没有完成，则自动回滚事务。
        int getTimeout();
        // 返回是否为只读事务，默认值为 false
        boolean isReadOnly();
    
        @Nullable
        String getName();
    }
    ```

    

- **`TransactionStatus`**： 事务运行状态。

  - ```java
    public interface TransactionStatus{
        boolean isNewTransaction(); // 是否是新的事物
        boolean hasSavepoint(); // 是否有恢复点 , nested事务里面，回滚的话，会回滚到保存点savePoint，不影响外层事务。而外部事务如果出错，会影响到嵌套事务，事务会回滚到保存点。
        void setRollbackOnly();  // 设置为只回滚
        boolean isRollbackOnly(); // 是否为只回滚
        boolean isCompleted; // 是否已完成
    }
    ```

    

### 实现原理

如果一个类或者一个类中的 public 方法上被标注`@Transactional` 注解的话，Spring 容器就会在启动的时候为其创建一个代理类，在调用被`@Transactional` 注解的 public 方法的时候，实际调用的是，`TransactionInterceptor` 类中的 `invoke()`方法。这个方法的作用就是在目标方法之前开启事务，方法执行过程中如果遇到异常的时候回滚事务，方法调用完成之后提交事务。

使用proxy（动态代理），通过AOP，在进入具体的方法之前，对方法进行增强。

执行invocation的proceed()方法，继续调用invokeWithTransaction()，包裹在try catch中。有了异常会回滚，在catch里面，而且会throw出去。



```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
		final TransactionManager tm = determineTransactionManager(txAttr);
		...
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
			...
			return result;
		}
	}
```

在其中有方法createTransactionIfNecessary()，

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

在tm.getTransaction(txAttr)中的handleExistingTransaction(def, transaction, debugEnabled)，这里就是不同的传播行为会创建不同的事务

判断当前环境是否已经存在事务，有事务走里面方法，没有事务走外面继续走；

继续走，进行很多判断，这里就要说注解@Transactional的信息,重点关注propagation（传播行为）和isolation（隔离级别），会有默认值，不同的值会走不同的方法。

### 事务传播

事务传播行为：当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。

在 TransactionDefinition 定义中包括了如下几个表示传播行为的常量：

支持当前事务的情况：

- TransactionDefinition.PROPAGATION_REQUIRED：当前不存在事务则新建事务，当前存在事务则使用当前事务。
- TransactionDefinition.PROPAGATION_SUPPORTS：当前不存在事务则不使用事务，当前存在事务则使用事务
- TransactionDefinition.PROPAGATION_MANDATORY：当前不存在事务则抛出异常，当前存在事务则使用事务

不支持当前事务

- TransactionDefinition.PROPAGATION_REQUIRES_NEW：当前不存在事务则新建事务，当前存在事务则新建事务
- TransactionDefinition.PROPAGATION_NOT_SUPPORTED：当前不存在事务则不使用事务，当前存在事务则挂起事务
- TransactionDefinition.PROPAGATION_NEVER：当前不存在事务则不使用事务，当前存在事务则抛出异常

其他：

- TransactionDefinition.PROPAGATION_NESTED：当前不存在事务则新建事务，当前存在事务则嵌套事务。

### 隔离级别

TransactionDefinition 接口中定义了五个表示隔离级别的常量：

- **TransactionDefinition.ISOLATION_DEFAULT**
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED**
- **TransactionDefinition.ISOLATION_READ_UNCOMMITTED**
- **TransactionDefinition.ISOLATION_REPEATABLE_READ**
- **TransactionDefinition.ISOLATION_SERIALIZABLE**

## Spring MVC

1. 用户访问浏览器，发送请求
2. DispatcherServlet接收到请求并响应
3. 请求查找handler，找到HandlerMapping解析请求对用的Handler
4. HandlerAdapter 会根据 Handler 来调用真正的Handler处理器（Controller 控制器）处理请求，并处理相应的业务逻辑
5. Handler处理器返回ModelAndView
6.  请求进行View Resolver进行解析，返回视图对象View
7. DispatcherServlet 渲染数据（Model）
8. 将得到视图对象View返回给用户

## Spring AOP

面向方面的编程需要把程序逻辑分解成不同的部分称为所谓的关注点。跨一个应用程序的多个点的功能被称为**横切关注点**，这些横切关注点在概念上独立于应用程序的业务逻辑。有各种各样的常见的很好的方面的例子，如日志记录、审计、声明式事务、安全性和缓存等。

## Spring IOC

将对象交给容器管理，只需要spring配置文件中配置对应的bean以及相关的属性，让spring容器来生成类的实例对象以及管理对象。在spring容器启动的时候，spring会把你在配置文件中配置的bean都初始化好，然后在调用的时候分配给需要的类。

### Spring将配置解析成什么后注册到容器的？

`BeanDefinitionReader`处理配置信息，

```
BeanDefinition接口中包括：
1. BeanClassName
2. Scope
3. LazyInit
4. DependsOn
5. FactoryBeanName
6. InitMethodName
7. Role
8. isSingleton
9. isPrototype
10. isAbstract
```



```
AnnotationConfigApplicationContext ：
	AnnotatedBeanDefinitionReader reader;
	ClassPathBeanDefinitionScanner scanner;
	register(componentClasses);
		reader.register(annotatedClasses);
	refresh();
		
```

### BeanFactoryPostProcessor可以做哪些事情？

OCP

### BeanPostProcessor和BeanFactoryPostProcessor有什么区别？

BeanPostProcessor功能

实现ApplicationContextAware接口，获取ApplicationContext

注册依赖组件

### Spring事件传播器有什么作用呢？监听者如何提供给事件传播器？

### 预先实例化单实例的流程



### 什么是循环依赖呢？

循环依赖其实就是循环引用，也就是两个或则两个以上的bean互相持有对方，最终形成闭环。比如A依赖于B，B依赖于C，C又依赖于A。

### Spring中的循环依赖有几种情况？

1.构造器循环依赖

**2.setter循环依赖**

**3.prototype范围的依赖**

### Spring是如何判定当前发生循环依赖的呢？

prototype判定中有个NamedThreadLocal，保存当前正在创建的bean的name，里面存放的是集合set\<String>

```java
DefaultSingletonBeanRegistry.java
/** Names of beans that are currently in creation. */
private final ThreadLocal<Object> prototypesCurrentlyInCreation =
      new NamedThreadLocal<>("Prototype beans currently in creation");
```
单例判定中有集合set\<String>

```java
DefaultSingletonBeanRegistry.java
/** Names of beans that are currently in creation. */
private final Set<String> singletonsCurrentlyInCreation =
      Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

### Spring setter注入依赖产生的循环依赖是可以解决的，原理是什么

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
  
  // 尝试从缓存中获取成品的目标对象，如果存在，则直接返回
  Object singletonObject = this.singletonObjects.get(beanName);
  
  // 如果缓存中不存在目标对象，则判断当前对象是否已经处于创建过程中，在前面的讲解中，第一次尝试获取A对象
  // 的实例之后，就会将A对象标记为正在创建中，因而最后再尝试获取A对象的时候，这里的if判断就会为true
  if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
    
    synchronized (this.singletonObjects) {
      singletonObject = this.earlySingletonObjects.get(beanName);
      if (singletonObject == null && allowEarlyReference) {
        
        // 这里的singletonFactories是一个Map，其key是bean的名称，而值是一个ObjectFactory类型的
        // 对象，这里对于A和B而言，调用图其getObject()方法返回的就是A和B对象的实例，无论是否是半成品
        ObjectFactory singletonFactory = this.singletonFactories.get(beanName);
        if (singletonFactory != null) {
          
          // 获取目标对象的实例
          singletonObject = singletonFactory.getObject();
          this.earlySingletonObjects.put(beanName, singletonObject);
          this.singletonFactories.remove(beanName);
        }
      }
    }
  }
  return singletonObject;
}
```

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
  throws BeanCreationException {
  // 实例化当前尝试获取的bean对象，比如A对象和B对象都是在这里实例化的
  BeanWrapper instanceWrapper = null;
  if (mbd.isSingleton()) {
    instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
  }
  if (instanceWrapper == null) {
    instanceWrapper = createBeanInstance(beanName, mbd, args);
  }
  // 判断Spring是否配置了支持提前暴露目标bean，也就是是否支持提前暴露半成品的bean
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences 
    && isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    
    // 如果支持，这里就会将当前生成的半成品的bean放到singletonFactories中，这个singletonFactories
    // 就是前面第一个getSingleton()方法中所使用到的singletonFactories属性，也就是说，这里就是
    // 封装半成品的bean的地方。而这里的getEarlyBeanReference()本质上是直接将放入的第三个参数，也就是
    // 目标bean直接返回
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }
  try {
    // 在初始化实例之后，这里就是判断当前bean是否依赖了其他的bean，如果依赖了，
    // 就会递归的调用getBean()方法尝试获取目标bean
    populateBean(beanName, mbd, instanceWrapper);
  } catch (Throwable ex) {
    // 省略...
  }
  return exposedObject;
}
```

doCreateBean()中把singletonFactory放到singletonFactories三级缓存中。

可见[spring-02](./2020-04-08-spring-02.md)

### Spring三级缓存的数据，什么时候升级到二级缓存的呢？

```java
DefaultSingletonBeanRegistry#getSingleton()
```

第一次调用获取第三级缓存中的ObjectFactory

### Spring只要二级缓存不行么，为什么必须要有第三级缓存呢？

当然不行，

第二级缓存earlySingletonObjects中存放的是singletonObject，是由单例工厂调用getObject方法得到的，其中是经过了后置处理器的操作。
singletonObject = singletonFactory.getObject();




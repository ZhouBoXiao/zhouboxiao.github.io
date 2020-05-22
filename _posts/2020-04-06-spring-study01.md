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

还待进一步完善

## Spring 事务管理实现方式

- **编程式事务管理**
  - 编程式事务管理通过TransactionTemplate手动管理事务，调用beginTransaction()、commit()、rollback()等事务管理相关的方法

- **声明式事务管理**
  - 声明式事务管理有三种实现方式：**基于TransactionProxyFactoryBean的方式**、**基于AspectJ的XML方式**、**基于@Transactional注解的方式**

## Spring中自动装配的方式

- no：不进行自动装配，手动设置Bean的依赖关系。 
- byName：根据Bean的名字进行自动装配。 
- byType：根据Bean的类型进行自动装配。 
- constructor：类似于byType，不过是应用于构造器的参数，如果正好有一个Bean与构造器的参数类型相同则可以自动装配，否则会导致错误。 
- autodetect：如果有默认的构造器，则通过constructor的方式进行自动装配，否则使用byType的方式进行自动装配。

## Spring 的 Bean的作用域与生命周期

在Spring中，那些组成应用程序的主体及由Spring IOC容器所管理的对象，被称之为Bean。

| 类别          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| singleton     | IOC容器中仅存在一个Bean实例，单例方式存在                    |
| prototype     | 每次从容器调用Bean时，都返回一个新的实例                     |
| request       | 每次HTTP请求都会创建一个新的Bean，该作用域仅适用于WebApplicationContext环境 |
| seesion       | 同一个HTTP Session共享一个Bean，不同Session使用不同Bean，仅适用于WebApplicationContext环境 |
| globalSession | 一般用于Portlet应用环境，仅适用于WebApplicationContext环境   |

### 生命周期



## Spring 中的事务传播和隔离级别

### 事务传播

事务传播行为：当事务方法被另一个事务方法调用时，必须指定事务应该如何传播。

在 TransactionDefinition 定义中包括了如下几个表示传播行为的常量：

支持当前事务的情况：

- TransactionDefinition.PROPAGATION_REQUIRED
- TransactionDefinition.PROPAGATION_SUPPORTS
- TransactionDefinition.PROPAGATION_MANDATORY

不支持当前事务

- TransactionDefinition.PROPAGATION_REQUIRES_NEW
- TransactionDefinition.PROPAGATION_NOT_SUPPORTED
- TransactionDefinition.PROPAGATION_NEVER

其他：

- TransactionDefinition.PROPAGATION_NESTED

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
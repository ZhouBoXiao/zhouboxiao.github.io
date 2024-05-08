---
layout:     post
title:      "SpringBoot多数据源以及事务处理"
subtitle:   "SpringBoot多数据源以及事务处理"
date:       2024-05-08
author:     "ZBX"
header-img: "img/tag-bg.jpg"
tags:
    - SpringBoot

---



## 背景

操作多个数据库的数据时，需要去解决如何动态管理多个数据源以及切换的问题，并保证多数据源的事务一致性。



## 遇到的问题

### Mybatis-plus—多数据源@DS和@Transactional冲突

当前项目已有自定义的事务配置，如下：

```java
@Aspect
@Configuration
@Slf4j
public class TransactionConfig {

    private static final int TX_METHOD_TIMEOUT = 300;
    private static final String AOP_POINTCUT_EXPRESSION = "execution(*com.example..service..*Service.*(..))";
    
    @Bean(name = "txAdvice")
    public TransactionInterceptor txAdvice() {
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        
        // 只读事务，不做更新操作
        RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();
        readOnlyTx.setReadOnly(true);
        readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

        // 当前存在事务就使用当前事务，当前不存在事务就创建一个新的事务
        RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();
        requiredTx.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(Exception.class)));
        requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        requiredTx.setTimeout(TX_METHOD_TIMEOUT);
        Map<String, TransactionAttribute> txMap = new HashMap<>();
        
        txMap.put("add*", requiredTx);
        txMap.put("save*", requiredTx);
        txMap.put("insert*", requiredTx);
        txMap.put("create*", requiredTx);
        txMap.put("update*", requiredTx);
        txMap.put("batch*", requiredTx);
        txMap.put("modify*", requiredTx);
        txMap.put("delete*", requiredTx);
        txMap.put("remove*", requiredTx);
        txMap.put("exec*", requiredTx);
        txMap.put("set*", requiredTx);
        txMap.put("do*", requiredTx);
        
        txMap.put("get*", readOnlyTx);
        txMap.put("query*", readOnlyTx);
        txMap.put("find*", readOnlyTx);
        txMap.put("*", requiredTx);
        source.setNameMap(txMap);
        TransactionInterceptor txAdvice = new TransactionInterceptor(transactionManager(), source);
        return txAdvice;
    }

    @Bean
    public Advisor txAdviceAdvisor(@Qualifier("txAdvice") TransactionInterceptor txAdvice) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression(AOP_POINTCUT_EXPRESSION);
        return new DefaultPointcutAdvisor(pointcut, txAdvice);
    }
    
    // 自定义事务管理器
    @Bean(name = "transactionManager")
    public DataSourceTransactionManager transactionManager(Datasource datasource) {
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(datasource);
        return transactionManager;
    }
}
```

1. 因为spring在开启事务的同时，会去数据库连接池拿数据库连接。在最外层接口层中开启事务，即TransactionInterceptor 会使用 Spring DataSourceTransactionManager 创建事务，并将事务信息连接信息，通过 ThreadLocal 绑定在当前线程。
2. 在内层使用@DS切换数据源，并没有重新开启新事务，没有改变当前线程事务的连接信息，仅仅是做了一次拦截，改变了DataSourceHolder的栈顶dataSource，对于整个事务的连接是没有影响的，所以会产生数据源没有切换的问题。



## 修复问题

原本的管理单个数据源，现在把动态路由数据源(MyRoutingDataSource)交给事务管理器。

```java

    @Autowired
    private ConfigurableApplicationContext applicationContext;
    
	@Bean(name = "transactionManager")
    public DataSourceTransactionManager transactionManager() {
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(applicationContext.getBean(MyRoutingDataSource.class));
        return transactionManager;
    }
```



![image.png](https://cdn.nlark.com/yuque/0/2023/png/452225/1677295948236-64cf591e-5042-4cd4-8813-3592e9daf39f.png#averageHue=%23f5f5f5&clientId=u3f1dc0bc-19d6-4&from=paste&height=348&id=u11dc3968&name=image.png&originHeight=696&originWidth=1634&originalType=binary&ratio=2&rotation=0&showTitle=false&size=80635&status=done&style=none&taskId=u0459a726-df86-4eb9-94d2-cd1dec32458&title=&width=817)

# EnableTransactionManagement原理

1. 跟spring注解aop类似，先看注解的定义，这里import了一个TransactionManagementConfigurationSelector，这是一个ImportSelector，前面说到ImportSelector的selectImports方法**返回的全限定类名会被加入到容器中**

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Import(TransactionManagementConfigurationSelector.class)
   public @interface EnableTransactionManagement {
   
   	boolean proxyTargetClass() default false;
   
   	AdviceMode mode() default AdviceMode.PROXY;
   
   	int order() default Ordered.LOWEST_PRECEDENCE;
   
   }
   ```

2. selectImports方法中默认情况下返回**AutoProxyRegistrar**、**ProxyTransactionManagementConfiguration**的全限定类名。

3. AutoProxyRegistrar是一个ImportBeanDefinitionRegistrar，它在spring启动的时候可以自定义的往容器中添加bean定义。此处是往容器中加了一个InfrastructureAdvisorAutoProxyCreator，它其实和springaop加的AnnotationAwareAspectJAutoProxyCreator一样，都是BeanPostProcessor，而且**都是由AbstractAutoProxyCreator所派生**（也就意味其他bean在初始化方法调用完毕后会调用org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator#postProcessAfterInitialization，如果需要会被代理）

4. ProxyTransactionManagementConfiguration向容器中添加了三个组件分别是：**transactionAdvisor**、**transactionAttributeSource**、**transactionInterceptor**

   1. transactionAttributeSource中包含了解析声明式事务的@Transactional等注解的解析器，主要用来解析声明式事务的参数
   2. transactionInterceptor它的本质是一个MethodInterceptor（方法拦截器），在spring中，代理对象调用目标方法之前会使用拦截器链进行拦截做一些处理，这个**transactionInterceptor包含了事务的全部操作**（开启事务、执行目标方法、提交或回滚等等）。虽然这些事务操作是在transactionInterceptor中完成，但是却是**委托给事务管理器**做的，也就是前面为什么我们要往容器中添加一个事务管理器
   3. transactionAdvisor事务增强器需要上述的两个组件，同事它也是一个增强器。在容器启动时，容器会为每个bean实例寻找所有可用的增强器（由于transactionAdvisor中有transactionAttributeSource，通过它来匹配所有需要进行事务增强的对象），如果有则通过proxyFactory创建代理对象（springaop那一套）

5. 有了代理对象后续事务方法调用就是springaop那一套了


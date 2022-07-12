# EnableAspectJAutoProxy原理

1. 先看注解的定义，可以看到它使用@Import的方式引入AspectJAutoProxyRegistrar，而AspectJAutoProxyRegistrar是ImportBeanDefinitionRegistrar的实现类（前面第2节中有说到，在容器初始化时会调用它的registerBeanDefinitions方法，可以在该方法中手动注测bean到容器中）

   ```java
   @Target(ElementType.TYPE)
   @Retention(RetentionPolicy.RUNTIME)
   @Documented
   @Import(AspectJAutoProxyRegistrar.class)
   public @interface EnableAspectJAutoProxy {
   
   	boolean proxyTargetClass() default false;
   
   	boolean exposeProxy() default false;
   
   }
   ```

2. 接下来就要看看AspectJAutoProxyRegistrar做了哪些事情。其中关键操作是**为容器中注测了一个AnnotationAwareAspectJAutoProxyCreator这么一个bean的定义**，也就是说只要理解了AnnotationAwareAspectJAutoProxyCreator干了什么就知道了spring的aop是怎么实现的了

   ```java
   class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
   
   	@Override
   	public void registerBeanDefinitions(
   			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
   		// 此处的逻辑是为容器中添加了一个bean定义
           // org.springframework.aop.config.internalAutoProxyCreator = AnnotationAwareAspectJAutoProxyCreator.class
   		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
   
           // 解析EnableAspectJAutoProxy设置的参数
           // 两个参数默认情况下都是false
   		AnnotationAttributes enableAspectJAutoProxy =
   				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
   		if (enableAspectJAutoProxy != null) {
                // 通过该参数的注释可以知道，这个是指定代理方式的，默认情况下使用jdk动态代理，为true时使用cglib代理
   			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
   				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
   			}
                // 是否将代理暴露到ThreadLocal，然后允许通过AopContext进行检索
                // 默认情况下时不暴露的
   			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
   				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
   			}
   		}
   	}
   
   }
   ```

3. 先看看AnnotationAwareAspectJAutoProxyCreator的继承关系，逐步向上寻找最终可以发现它实现了BeanPostProcessor接口，所以**它是一个后置处理器**。这个接口我们很熟悉，它是spring中的后置处理器，在bean初始化前后会通过它来做一些事情

   ```java
   AnnotationAwareAspectJAutoProxyCreator
       ext -> AspectJAwareAdvisorAutoProxyCreator
       	ext -> AbstractAdvisorAutoProxyCreator
       		ext -> AbstractAutoProxyCreator
       			impl -> SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware
   ```

4. AnnotationAwareAspectJAutoProxyCreator到底是怎么注册到容器中的呢？

   1. 由于AnnotationAwareAspectJAutoProxyCreator是一个后置处理器，所以它肯定跟随其他后置处理器一起被注册到容器中
   2. 容器启动时调用refresh()方法中的registerBeanPostProcessors()，向容器中注册定义好的后置处理器
   3. 由于它还实现了Ordered方法，所以在registerBeanPostProcessors方法中的第二步注册（spring先注册实现了PriorityOrdered接口的后置处理器，再注册实现了Ordered接口的后置处理器，然后再注册其他的后置处理器）
   4. 具体注册方式是通过getBean的方式将AnnotationAwareAspectJAutoProxyCreator实例化并且添加到容器中
   5. 需要注意的是由于AnnotationAwareAspectJAutoProxyCreator实现了BeanFactoryAware它在初始化时还会调用自己的setBeanFactory方法（具体调用的是org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator#setBeanFactory）
   6. 在setBeanFactory方法中通过initBeanFactory方法实例化了三个东西：advisorRetrievalHelper、aspectJAdvisorFactory、aspectJAdvisorsBuilder

5. 经过第4步AnnotationAwareAspectJAutoProxyCreator被注册到容器中了，那么在**后续的bean注册**的过程中都会调用AnnotationAwareAspectJAutoProxyCreator来处理bean的注册

6. AnnotationAwareAspectJAutoProxyCreator是怎样实现代理bean的呢？

   1. 首先判断bean是否在advisedBeans中，advisedBeans保存了所有需要增强的bean，value为true表示已经增强过了
   2. 判断当前bean是否是基础设施类型（实现了Advice、Pointcut、Advisor、AopInfrastructureBean中的任何一个都属于基础类型），或者是否需要跳过（beanName表示了一个原始实例），如果满足任何一个，则不增强它（默认情况下是返回false的）
   3. 所以AnnotationAwareAspectJAutoProxyCreator的postProcessBeforeInstantiation方法返回了空，也就意味着doCreateBean会执行，会正常的创建Bean对象
   4. **正常创建bean实例后后会调用AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法返回一个代理后的bean**
   5. **需要注意的是如果是循环依赖情况下返回代理对象的时机AbstractAutoProxyCreateor在getEarlyBeanReference方法调用时**

7. AnnotationAwareAspectJAutoProxyCreator怎样返回一个代理后的bean呢？

   1. warpIfNecessary中获取当前bean的所有增强器（通知方法）
   2. 如果不为空，则将当前bean设置到advisedBeans中，并且标识为已增强(value设置为true)
   3. 通过createProxy创建代理对象
      1. 获取所有增强器，保存到proxyFactory代理工厂中
      2. 通过proxyFactory创建代理对象，如果是接口或者是代理对象或者是Lambda类则使用jdk动态代理，否则使用cglib代理
   4. 返回代理对象

## 总结

1. 使用EnableAspectJAutoProxy注解启动spring的aop代理后会向容器中添加一个AnnotationAwareAspectJAutoProxyCreator的后置处理器
2. AnnotationAwareAspectJAutoProxyCreator跟随其他后置处理器一起注册到容器中
3. spring在代理bean的时候主要通过AnnotationAwareAspectJAutoProxyCreator的postProcessAfterInitialization方法返回代理对象（在bean初始化完成后）
4. 在postProcessAfterInitialization方法中，首先获取当前bean的所有增强器，将其放入proxyFactory中
5. 通过proxyFactory创建代理对象（使用jdk动态代理还是cglib代理由spring自己决定）
6. 将代理对象加入到容器中，后面使用时都使用得这个代理对象




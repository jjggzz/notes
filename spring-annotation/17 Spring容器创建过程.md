# Spring容器创建过程

1. 主要创建逻辑都在org.springframework.context.support.AbstractApplicationContext#refresh()方法中

   ```java
   public void refresh() throws BeansException, IllegalStateException {
   		synchronized (this.startupShutdownMonitor) {
   			StartupStep contextRefresh = this.applicationStartup.start("spring.context.refresh");
   
   			// Prepare this context for refreshing.
   			prepareRefresh();
   
   			// Tell the subclass to refresh the internal bean factory.
   			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
   
   			// Prepare the bean factory for use in this context.
   			prepareBeanFactory(beanFactory);
   
   			try {
   				// Allows post-processing of the bean factory in context subclasses.
   				postProcessBeanFactory(beanFactory);
   
   				StartupStep beanPostProcess = this.applicationStartup.start("spring.context.beans.post-process");
   				// Invoke factory processors registered as beans in the context.
   				invokeBeanFactoryPostProcessors(beanFactory);
   
   				// Register bean processors that intercept bean creation.
   				registerBeanPostProcessors(beanFactory);
   				beanPostProcess.end();
   
   				// Initialize message source for this context.
   				initMessageSource();
   
   				// Initialize event multicaster for this context.
   				initApplicationEventMulticaster();
   
   				// Initialize other special beans in specific context subclasses.
   				onRefresh();
   
   				// Check for listener beans and register them.
   				registerListeners();
   
   				// Instantiate all remaining (non-lazy-init) singletons.
   				finishBeanFactoryInitialization(beanFactory);
   
   				// Last step: publish corresponding event.
   				finishRefresh();
   			}
   
   			catch (BeansException ex) {
   				if (logger.isWarnEnabled()) {
   					logger.warn("Exception encountered during context initialization - " +
   							"cancelling refresh attempt: " + ex);
   				}
   
   				// Destroy already created singletons to avoid dangling resources.
   				destroyBeans();
   
   				// Reset 'active' flag.
   				cancelRefresh(ex);
   
   				// Propagate exception to caller.
   				throw ex;
   			}
   
   			finally {
   				// Reset common introspection caches in Spring's core, since we
   				// might not ever need metadata for singleton beans anymore...
   				resetCommonCaches();
   				contextRefresh.end();
   			}
   		}
   	}
   ```

   

2. 首先是prepareRefresh方法，它在容器刷新之前做一些前置工作：创建环境变量（environment）等等。

3. 接着是通过obtainFreshBeanFactory获取bean工厂（BeanFactory），这个**bean工厂其实是在构造方法执行完的时候就创建了**，此处只是设置序列号而已。BeanFactory的实现类是DefaultListableBeanFactory

4. 接着是调用prepareBeanFactory做一些前置操作，向bean工厂中添加：

   1. ApplicationContextAwareProcessor，此后置处理器用来处理一部分aware接口
   2. 添加一些需要忽略自动装配的接口通过ignoreDependencyInterface
   3. 注册一些可以直接用的依赖：BeanFactory、ResourceLoader、ApplicationEventPublisher、ApplicationContext（**这些对象可以直接@Autowired使用**）
   4. 添加一个后置处理器ApplicationListenerDetector，它用来解析ApplicationListener的所有子实现类并将它们加入到事件多播器中
   5. 注册一些环境相关的bean到容器中：environment、systemProperties、systemEnvironment等等

5. 通过invokeBeanFactoryPostProcessors方法加载所有的bean工厂后置处理器，并按顺序调用其实现方法，此时bean工厂已创建完成，并且bean定义已解析完成

   1. 首先注册并调用实现了优先级接口（PriorityOrdered）的BeanDefinitionRegistryPostProcessor
   2. 再注册并调用实现了排序接口（Ordered）的BeanDefinitionRegistryPostProcessor
   3. 再注册并调用剩下的BeanDefinitionRegistryPostProcessor
   4. 然后注册并调用实现了优先级接口（PriorityOrdered）的BeanFactoryPostProcessor
   5. 再注册并调用实现了排序接口（Ordered）的BeanFactoryPostProcessor
   6. 再注册并调用剩下的BeanFactoryPostProcessor

6. 接着通过registerBeanPostProcessors方法注册bean的后置处理器，spring通过这些后置处理器来实现其强大的拓展功能

   1. 首先注册实现了优先级接口（PriorityOrdered）的BeanPostProcessor
   2. 接着注册实现了排序接口（Ordered）的BeanPostProcessor
   3. 接着注册剩下的BeanPostProcessor
   4. 然后注册MergedBeanDefinitionPostProcessor
   5. 重新注册ApplicationListenerDetector到bean后置处理器的末端（我们知道再4.4的时候也注册过），此处是将其移至末端。为什么两个不同的对象能能够替换？因为ApplicationListenerDetector重写了equals方法，只要内部持有的applicationContext一样则认为是同一个对象

7. 通过initMessageSource初始化messageSource国际化，这一步就是判断用户有没有自定义MessageSource的实现类并将其加入到容器（通过获取bean的定义信息），如果没有则创建一个默认的DelegatingMessageSource并加入到容器（添加到单例池）

8. 通过initApplicationEventMulticaster初始化spring的事件多播器，它用来将用户发布的事件广播到所有符合条件的ApplicationListener中。与7类似，如果用户没有实现该接口则spring自己会创建一个SimpleApplicationEventMulticaster并加入到单例池中

9. 通过registerListeners将所有ApplicationListener的实现类注册到事件多播器中（此处只是从bean定义信息中获取到监听器的名字并加入而已，并没有实例化对象）

10. 通过finishBeanFactoryInitialization方法实例化剩下的bean

    1. 注册一个ConversionService，它是用来做类型转换的，默认情况下不会添加
    2. 实例化所有LoadTimeWeaverAware的实现类（它和aop织入有关），默认情况下没有
    3. 实例化所有非懒加载的bean
       1. 遍历所有非抽象、单例、非懒加载的bean定义，如果是工厂bean（FactoryBean）则调用getBean方法创建的时候加上前缀&，如果是普通bean则直接通过getBean来创建
       2. getBean调用doGetBean
          1. doGetBean中，**首先会尝试从缓存中获取单例对象**（首先从一级缓存（singletonObjects）中获取。如果没有，判断这个bean是否正在创建，如果正在创建则从二级缓存（earlySingletonObjects）中获取，如果还没获取到则到三级缓存（singletonFactories）中获取，如果能够拿到则放入二级缓存，并从三级缓存中移除）
          2. 如果从缓存中获取不到bean则尝试从父容器中获取，如果获取不到则执行创建bean的流程
          3. 在正式创建bean之前要解决先决条件，即实例化所有依赖（此处说的依赖指的并不是需要注入的属性，而是@DependsOn注解所要求的bean）
          4. 创建之前先检查是否该bean正在被创建，如果是则报错
          5. 调用createBean创建bean实例
             1. 首先会调用InstantiationAwareBeanPostProcessor后置处理器的postProcessBeforeInstantiation方法，看能不能获取到对象。如果能，直接对该对象调用所有BeanPostProcessor的postProcessAfterInitialization然后就结束了（短路操作）
             2. 如果不能获取到对象则调用doCreateBean创建对象
                1. 首先是实例化，如果有工厂方法（通过@Bean方法创建实例、FactoryBean等等）则通过工厂方法创建，如果是通过类似@Component的方式实例化，则会使用差异度分析**进行构造器推导**，有点复杂
                2. 如果允许则会对该bean的定义调用所有MergedBeanDefinitionPostProcessor接口的实例进行处理
                3. 将实例化好的对象加入到三级缓存
                4. 通过populateBean方法为bean进行属性填充，主要通过AutowiredAnnotationBeanPostProcessor（@Autowired）和CommonAnnotationBeanPostProcessor（@Resource）实现
                5. 调用initializeBean方法初始化bean。
                   1. 如果该bean实现了BeanNameAware、BeanClassLoaderAware、BeanFactoryAware等接口，则调用这些接口限定的方法
                   2. 使用所有的BeanPostProcessor的postProcessBeforeInitialization处理bean
                   3. 执行init方法，其实就是InitializingBean接口的afterPropertiesSet、被@PostConstruct标注的方法、@Bean注解中指定的initMethod方法
                   4. 使用所有的BeanPostProcessor的postProcessAfterInitialization处理bean
          6. 将得到的bean加入到一级缓存中，并从二三级缓存中移除

11. 通过finishRefresh方法注册一个LifecycleProcessor的实现类（如果没有的话）并调用其onRefresh方法。发布ContextRefreshedEvent事件广播容器已刷新完成










# springboot 启动流程

1. 调用SpringApplication的静态run方法将当前启动类的class对象和命令行参数传入

   ```java
   public static void main(String[] args) {
           SpringApplication.run(Main.class, args);
   }
   ```

2. 最终会以启动类的class对象作为构造参数，创建一个SpringApplication对象，并调用它的run方法（非静态方法）

   ```java
   public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
   		return new SpringApplication(primarySources).run(args);
   }
   ```

3. 创建SpringApplication对象主要做一下几件事

   1. 将主启动类设置到primarySources中保存起来
   2. 判断当前的应用类型是什么并保存下来（reactive、servlet、none）如果有DispatcherHandler而没有DispatcherServlet那么认为是reactive类型的应用，如果没有javax.servlet.Servlet也没有org.springframework.web.context.ConfigurableWebApplicationContext则认为是none，否则认为是servlet
   3. 从spring.factories中获取所有声明的BootstrapRegistryInitializer的实现类并创建实例保存起来（默认情况下一个没有）
   4. 从spring.factories中获取所有声明的ApplicationContextInitializer的实现类并创建实例保存起来
   5. 从spring.factories中获取所有声明的ApplicationListener的实现类并创建实例保存起来（但是这些listener此时并未被容器托管）

4. 调用SpringApplication对象的run方法真正的执行启动操作

   ```java
   public ConfigurableApplicationContext run(String... args) {
   		StopWatch stopWatch = new StopWatch();
   		stopWatch.start();
     	// 1.
   		DefaultBootstrapContext bootstrapContext = createBootstrapContext();
   		ConfigurableApplicationContext context = null;
   		configureHeadlessProperty();
     	// 2.
   		SpringApplicationRunListeners listeners = getRunListeners(args);
     	// 3.
   		listeners.starting(bootstrapContext, this.mainApplicationClass);
   		try {
         // 4.
   			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
         // 5.
   			ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
   			configureIgnoreBeanInfo(environment);
   			Banner printedBanner = printBanner(environment);
         // 6.
   			context = createApplicationContext();
   			context.setApplicationStartup(this.applicationStartup);
         // 7.
   			prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
         // 8.
   			refreshContext(context);
         // 9.
   			afterRefresh(context, applicationArguments);
   			stopWatch.stop();
   			if (this.logStartupInfo) {
   				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
   			}
         // 10.
   			listeners.started(context);
         // 11.
   			callRunners(context, applicationArguments);
   		}
   		catch (Throwable ex) {
         // 12.
   			handleRunFailure(context, ex, listeners);
   			throw new IllegalStateException(ex);
   		}
   
   		try {
         // 13.
   			listeners.running(context);
   		}
   		catch (Throwable ex) {
   			handleRunFailure(context, ex, null);
   			throw new IllegalStateException(ex);
   		}
   		return context;
   	}
   ```

   

   1. 创建引导上下文DefaultBootstrapContext，并对它应用所有的BootstrapRegistryInitializer（3.3创建的）
   2. 从spring.factories中获取所有声明的SpringApplicationRunListener的实现类并创建实例（这些实例就是springboot启动过程的钩子，在启动过程中每到某个阶段就会触发相应的方法）
   3. 调用listeners的所有starting方法，我们可以通过它来为我们的引导上下文设置一些属性
   4. 解析用户提交的命令行参数（就是jar包启动时携带的参数）
   5. 初始化环境变量，并且会执行listeners的所有environmentPrepared方法，我们可以在此处往环境变量中塞一些值（此时我们的properties、yaml文件都还没有解析）
   6. 创建IOC容器，servlet应用创建AnnotationConfigServletWebServerApplicationContext、reacthve应用创建AnnotationConfigReactiveWebServerApplicationContext，none创建AnnotationConfigApplicationContext
   7. 做IOC容器刷新的前置操作（设置环境变量、设置是否允许冲定义bean、类加载器、添加一些转换服务、BeanFactoryPostProcessor等等），还会触发listeners的所有contextPrepared方法、关闭引导上下文（**触发BootstrapContextClosedEvent事件，注意此事件并不会被IOC容器中的applicationListener监听**，因为此时IOC并没有准备好，可以通过3.3的BootstrapRegistryInitializer将listener加入引导容器，从而监听事件触发）、触发listeners的所有contextLoaded方法
   8. 刷新容器，此处就是调用我们熟悉的refresh方法。
   9. afterRefresh空方法，给用户的一个拓展点
   10. 触发所有listeners的started方法
   11. callRunners从容器中获取所有ApplicationRunner实例与CommandLineRunner实例，依次调用（会进行排序）他们的run方法
   12. 容器启动失败会触发所有listeners的failed方法
   13. 如无异常则调用所有listeners的running方法

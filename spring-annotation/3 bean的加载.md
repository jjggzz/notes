# bean的加载

## @Bean的使用

1. 只能标注在方法上或者标注在注解上
2. 标注在注解上则该注解具有@Bean的能力
3. 标注在方法上则该方法返回的对象将被加入到容器中（需要是容器中的对象），该对象的默认name是方法名
4. 可以通过注解的value来指定该对象在容器中的name

```java
@Configuration
public class UserConfig {

    @Bean(name = "user")
    public User user() {
        return new User();
    }

}
```



## @Scope的使用

1. 可以通过@Scope来指定该bean的类型，默认是单例的，ConfigurableBeanFactory.SCOPE_SINGLETON（单例），ConfigurableBeanFactory.SCOPE_PROTOTYPE（原型）
2. 单例的bean在容器初始化时就会执行@Bean的方法
3. 原型bean在获取时才会执行标注了@Bean的方法

```java
@Configuration
public class UserConfig {
	
    @Bean(name = "user")
    // 容器初始化时并不会执行该方法，只有获取bean时才会执行，获取几次执行几次
    @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
    public User user() {
        return new User();
    }

}
```



## @Lazy懒加载

1. 对于**单例**的bean而言一般是在容器初始化时就进行加载了
2. 可以通过它来延迟加载bean，容器初始化时并不会加载该bean，**第一次获取**时才会加载它

```java
@Configuration
public class UserConfig {

    @Bean
    @Lazy
    public User user() {
        return new User();
    }

}
```



## @Conditional条件加载

1. 它是springboot底层大量使用的注解
2. 它可以按照条件进行bean的加载，满足条件则加载bean

```java
@Configuration
public class UserConfig {

    // 如果是linux则zs被加载
    @Bean(value = "zs")
    @Conditional(value = {LinuxCondition.class})
    public User zs() {
        return new User("linux",18);
    }

    // 如果是Windows则ls被加载
    @Bean(value = "ls")
    @Conditional(value = {WindowsCondition.class})
    public User ls() {
        return new User("Windows",20);
    }

}

// 如果系统是Windows则返回true
public class WindowsCondition  implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String property = context.getEnvironment().getProperty("os.name");
        return property.contains("Windows");
    }
}

// 如果系统是linux则返回true
public class LinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String property = context.getEnvironment().getProperty("os.name");
        return property.contains("linux");
    }
}
```



## FactoryBean方式加载bean到容器

1. 这种方式大量用于第三方框架整合到spring中
2. 通过实现FactoryBean接口，其getObject方法的返回值被加载到容器
3. getObjectType返回的是bean的类型
4. isSingleton返回值代表这这个bean是否是单例的
5. 可以通过&加bean名获取到这个bean的工厂bean

```java
@Configuration
public class UserConfig {

    // 这种方式会将UserFactoryBean的返回值加入到容器当中，并且bean的名字为user
    @Bean
    public UserFactoryBean user() {
        return new UserFactoryBean();
    }

}

// 实现FactoryBean接口
public class UserFactoryBean implements FactoryBean<User> {
    
    // 此方法返回的对象将会添加到容器当中
    @Override
    public User getObject() throws Exception {
        return new User();
    }

    // 指定bean的类型
    @Override
    public Class<?> getObjectType() {
        return User.class;
    }
}

public class Main {
    public static void main(String[] args) {
        ApplicationContext  ac = new AnnotationConfigApplicationContext(UserConfig.class);
        // 获取bean，类型为com.jgz.config.User
        Object user = ac.getBean("user");
        // 获取工厂bean，类型为com.jgz.config.UserFactoryBean
        Object userFactory = ac.getBean("&user");
    }
}
```



## @Bean指定初始化和销毁方法

1. 使用initMethod指定初始化方法
2. 使用destroyMethod指定销毁方法
3. 对于单例的bean，initMethod在容器初始化时执行（懒加载在第一次获取时执行），原型的bean在获取时执行
4. 对于单例的bean，destroyMethod在容器销毁时执行，原型的bean不执行销毁方法（因为获取的bean容器并没有管理）

```java
@Configuration
public class UserConfig {

    @Bean(initMethod = "initUser",destroyMethod = "destroyUser")
    public User user() {
        return new User();
    }

}
public class User {

    public void initUser() {
        System.out.println("用户初始化了");
    }

    public void destroyUser() {
        System.out.println("用户销毁了");
    }
}
```

## InitializingBean和DisposableBean指定初始化和销毁方法

1. 实现InitializingBean接口指定初始化方法，该方法在bean实例化完成，并且属性赋值完成后调用
2. 实现DisposableBean指定销毁方法

```java
@Configuration
public class UserConfig {

    @Bean
    public User user() {
        return new User();
    }

}

public class User implements InitializingBean, DisposableBean {

    // 重写InitializingBean的方法
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("用户初始化了");
    }

    // 重写DisposableBean的方法
    @Override
    public void destroy() throws Exception {
        System.out.println("用户销毁了");
    }

}

```

## JSR250中的注解指定初始化和销毁方法

1. 使用@PostConstruct指定初始化方法，该方法在bean实例化完成，并且属性赋值完成后调用
2. 使用@PreDestroy指定销毁方法

```java
@Configuration
public class UserConfig {

    @Bean
    public User user() {
        return new User();
    }

}

public class User {

    @PostConstruct
    public void initUser() {
        System.out.println("用户初始化了");
    }

    @PreDestroy
    public void destroyUser() {
        System.out.println("用户销毁了");
    }

}

```


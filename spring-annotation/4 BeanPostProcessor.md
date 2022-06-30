# BeanPostProcessor

它是**bean的后置处理器**，它可以在bean创建后，bean初始化前后做一些额外的操作，它有两个方法：

1. postProcessBeforeInitialization 这个方法在bean初始化之前调用（即在init-method指定的方法或者afterPropertiesSet方法或者@PostConstruct标注的方法之前调用）
2. postProcessAfterInitialization 这个方法在bean初始化之后调用（即在init-method指定的方法或者afterPropertiesSet方法或者@PostConstruct标注的方法之后调用）
3. 可以通过它来进行增强bean的功能，代理bean，替换bean等等操作
4. spring底层通过它实现赋值，注入其他组件，@Autowired，生命周期注解（@PostConstruct，@PreDestroy）功能，@Async等等

```java
@Configuration
@Import(MyBeanPostProcessor.class)
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

public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        if (beanName.equals("user")) {
            System.out.println("这个会在初始化方法调用之前调用");
        }
        return BeanPostProcessor.super.postProcessBeforeInitialization(bean, beanName);
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (beanName.equals("user")) {
            System.out.println("这个会在初始化方法调用之后调用");
        }
        return BeanPostProcessor.super.postProcessAfterInitialization(bean, beanName);
    }
}
```


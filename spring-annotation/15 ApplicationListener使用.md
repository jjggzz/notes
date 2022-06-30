# ApplicationListener使用

他是spring为用户提供的一个监听时间发布的事件驱动的组件（与guava的eventBus是一样的）

1. spring容器再启动完成后或者关闭时都会触发一些事件

   1. 启动完成会触发ContextRefreshedEvent

   2. 关闭会触发ContextClosedEvent

   3. 由于上述特性，我们就可以再容器启动完成之后做一些事情（比如启动一些内嵌服务或者初始化一些数据等等）

      ```java
      public class MyApplicationListener implements ApplicationListener<ContextRefreshedEvent> {
      
          public void onApplicationEvent(ContextRefreshedEvent event) {
              System.out.println("do something at application refresh after.....");
      
          }
      }
      ```

      

2. 我们也可以定义自己的事件发布并处理

   ```java
   /**
    * 定义一个自己的事件并继承ApplicationEvent
    */
   public class MyEvent extends ApplicationEvent {
   
       public MyEvent(String message) {
           super(message);
       }
   }
   
   /**
    * 定义自己的事件监听器，监听MyEvent
    */
   public class MyApplicationListener implements ApplicationListener<MyEvent> {
   
       public void onApplicationEvent(MyEvent event) {
           // 接收事件并处理
           String source = (String)event.getSource();
           System.out.println("接受到事件：" + source);
       }
   }
   
   /**
    * 将监听器加入容器
    */
   @Configuration
   @Import({MyApplicationListener.class})
   public class Config {
   
   }
   
   public class Main {
       public static void main(String[] args) {
           ApplicationContext applicationContext = new AnnotationConfigApplicationContext(Config.class);
           // 发布事件
           applicationContext.publishEvent(new MyEvent("hello"));
       }
   }
   ```

   
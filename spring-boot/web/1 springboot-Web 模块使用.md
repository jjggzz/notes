# springboot-Web 模块使用

1. 导入web模块的启动器

   ```xml
   				<dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
   ```

   

2. 编写主启动类

   ```java
   @SpringBootApplication
   public class Main {
   
       public static void main(String[] args) {
           SpringApplication.run(Main.class,args);
       }
   }
   ```

   

3. 在主启动类的同包或者子包中编写controller

   ```java
   /**
    * RestController = Controller + ResponseBody
    */
   @Configuration
   @RestController
   public class HelloController {
   
       @GetMapping("hello")
       public String hello(){
           return "world";
       }
   
   }
   ```

4. 现在可以通过浏览器访问http://localhost:8080/hello
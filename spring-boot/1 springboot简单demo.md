# springboot简单demo

## hello world

1. pom文件

   ```xml
   		<!--继承springboot父项目，父项目中定义了各种常用类库的版本号-->
   		<parent>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-parent</artifactId>
           <version>2.5.4</version>
       </parent>
   
       <dependencies>
          <!--倒入springbootweb模块的启动场景-->
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
       </dependencies>
   
   ```

2. 主启动类

   ```java
   @SpringBootApplication
   public class Main {
   
       public static void main(String[] args) {
           SpringApplication.run(Main.class,args);
       }
   }
   ```

3. 在主启动类同包或者子包创建控制器

   ```java
   /**
    * RestController = Controller + ResponseBody
    */
   @RestController
   public class HelloController {
   
       @GetMapping("hello")
       public String hello(){
           return "world";
       }
   
   }
   ```

4. 在浏览器可以通过localhost:8080/hello访问该接口了
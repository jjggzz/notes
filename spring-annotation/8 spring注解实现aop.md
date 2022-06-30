# spring注解实现aop

1. 需要导入配置依赖，确保aop模块和aspects模块都在项目依赖中

   ```xml
   		<!--spring上下文包，该包会自动导入spring-aop依赖-->
   		<dependency>
               <groupId>org.springframework</groupId>
               <artifactId>spring-context</artifactId>
               <version>5.3.20</version>
           </dependency>
   		<!--导入spring-aspects依赖-->
           <dependency>
               <groupId>org.springframework</groupId>
               <artifactId>spring-aspects</artifactId>
               <version>5.3.20</version>
           </dependency>
   ```

2. 定义业务逻辑类

   ```java
   public class MathCalc {
   
       /**
        * 对两数进行除法运算
        */
       public int div(int a,int b){
           return a / b;
       }
   
   }
   ```

3. 定义切面类(注意使用的注解都是aspectj的注解)

   ```java
   import org.aspectj.lang.JoinPoint;
   import org.aspectj.lang.annotation.*;
   
   import java.util.Arrays;
   
   /**
    * Aspect注解告诉spring容器这个类是一个切面类
    */
   @Aspect
   public class AopClass {
   
       /**
        * 定义切入点：对MathCalc的div方法进行切入
        * ..代表不管入参的数量与类型都进行切入
        */
       @Pointcut("execution(public int com.jgz.MathCalc.div(..))")
       public void  point(){};
   
   
       /**
        * 前置通知，被切方法执行前执行
        */
       @Before("point()")
       public void logStart(JoinPoint joinPoint) {
           Object[] args = joinPoint.getArgs();
           System.out.println(joinPoint.getSignature().getName() + "开始执行，参数:" + Arrays.toString(args));
       }
   
       /**
        * spring5
        * 后置通知，被切方法执行完成后（AfterReturning执行完成后或AfterThrowing执行完成后）执行（无论被切方法是否正常结束，该方法都会执行）
        * spring4
        *后置通知，被切方法执行完成后（AfterReturning执行完成前或AfterThrowing执行完成前）执行（无论被切方法是否正常结束，该方法都会执行）
        */
       @After("point()")
       public void logEnd(JoinPoint joinPoint){
           System.out.println(joinPoint.getSignature().getName() + "执行完成");
       }
   
       /**
        * 返回通知，方法执行完，正常返回后执行
        * 注意JoinPoint参数必须为方法的第一个参数
        */
       @AfterReturning(value = "point()",returning = "ret")
       public void logReturn(JoinPoint joinPoint,Object ret) {
           System.out.println(joinPoint.getSignature().getName() + "执行正常返回,返回值" + ret);
       }
   
       /**
        * 异常通知，方法抛出异常后执行
        */
       @AfterThrowing(value = "point()",throwing = "e")
       public void logException(JoinPoint joinPoint,Exception e) {
           System.out.println(joinPoint.getSignature().getName() + "执行异常：" + e);
       }
   
   }
   ```

4. 开启aop功能，并将业务逻辑和切面类加入到容器中

   ```java
   /**
    * EnableAspectJAutoProxy开启spring的aop功能
    */
   @EnableAspectJAutoProxy
   @Configuration
   public class Config {
   
       /**
        * 将业务逻辑类加入到容器中
        * @return
        */
       @Bean
       public MathCalc mathCalc() {
           return new MathCalc();
       }
   
       /**
        * 将切面类加入到容器中
        * @return
        */
       @Bean
       public AopClass aopClass() {
           return new AopClass();
       }
   
   }
   ```

5. 调用

   ```java
   public class Main {
       public static void main(String[] args) {
           ApplicationContext  applicationContext = new AnnotationConfigApplicationContext(Config.class);
           MathCalc bean = applicationContext.getBean(MathCalc.class);
           bean.div(20,10);
       }
   }
   -----------------------------------
   div开始执行，参数:[20, 10]
   com.jgz.MathCalc.div方法执行
   div执行正常返回,返回值2
   div执行完成
   ```

   
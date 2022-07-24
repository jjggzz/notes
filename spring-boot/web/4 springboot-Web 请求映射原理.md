# springboot-Web 请求映射原理

1. 首先要明确的是在springboot是对原生spring的一些封装和自动配置，所以springboot的web请求也是springmvc相关的足迹进行处理的

2. 所有的请求都会由DispatchServlet来处理，而DispatchServlet是一个HttpServlet

   ```java
   DispatcherServlet
     => FrameworkServlet
     		=> HttpServletBean
     				=> HttpServlet
   ```

3. FrameworkServlet重写了HttpServlet的doGet、doPost等等方法，使其都调用本类的processRequest方法，而processRequest方法中主要业务逻辑由doService实现

4. DispatchServlet实现了doService方法，在方法中首先是对请求的参数做了快照，然后设置了一些请求域相关的属性，然后调用了doDispatch方法

5. 在doDispatch方法中就包含了请求处理的所有逻辑，其他的先不看，先看如何获取处理器链的（包含了我们写的标注了@RequestMapping注解的方法和一些拦截器等等）

   1. 方法中主要通过getHandler获取请求的处理器链
   2. getHandler方法做的事是遍历handlerMappings列表（它包含了所有的HandlerMapping），通过HandlerMapping.getHandler方法获取处理器链，如果获取到了则直接return（所以此处隐含优先级）
   3. handlerMappings其他的先不管，我们自己写的controller的标注了@RequestMapping方法的逻辑主要由RequestMappingHandlerMapping来解析
   4. HandlerMapping.getHandler主要通过getHandlerInternal方法来获取请求处理的方法
   5. getHandlerInternal方法会调用lookupHandlerMethod方法
   6. 在lookupHandlerMethod方法中，首先通过url从mappingRegistry中获取能够处理该请求的列表（因为可能存在多个处理器可以处理同一个url，只是能够处理的请求方式不一样，restful）
   7. 最后再度从得到的列表中获取真正完全匹配的处理器
   8. 将处理器与匹配的拦截器（HandlerInterceptor实现类）一起包装成HandlerExecutionChain

6. 最终我们通过请求的url和请求方式拿到了一个处理器的执行链




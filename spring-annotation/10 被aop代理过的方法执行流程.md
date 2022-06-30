# 被aop代理过的方法执行流程

1. 调用被增强的方法首先会进入拦截方法，根据不同的代理类型拦截目标方法的调用
   1. cglib代理主要走到这org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept
   2. jdk动态代理主要走到org.springframework.aop.framework.JdkDynamicAopProxy#invoke
   3. 它们的共同点都是为了获取拦截器链然后去调用
2. 通过proxyFactory获取目标方法的拦截器链，如果没有拦截器链直接调用目标方法
   1. 判断获取到的拦截器是不是MethodInterceptor，如果是，直接加入拦截器链
   2. 如果不是则通过adapter适配成MethodInterceptor，并加入拦截器链
3. 如果有拦截器链则创建一个MethodInvocation对象，把需要执行方法调用的对象、目标方法、拦截器链、目标的类对象等信息封装。并调用proceed方法
   1. cglib创建的对象是CglibMethodInvocation
   2. jdk动态代理创建的是ReflectiveMethodInvocation
   3. CglibMethodInvocation是ReflectiveMethodInvocation的字类，并且在调用proceed方法时是调用父类的（ReflectiveMethodInvocation的）proceed方法
   4. 责任链调用方式为**递归调用**，对于spring5而言
      1. 将本次被调用的MethodInvocation放入ThreadLocal中，并将之前在ThreadLocal中MethodInvocation对象暂存
      2. 先将先将before压入栈中并执行before逻辑，然后再次调用proceed。
      3. 将after压入栈中，然后直接调用proceed，after逻辑包裹再final块中（代表不管执行是否成功都会调用）
      4. 将afterReturning压入栈中，它先调用proceed，再直接调用afterReturning逻辑（代表如果后续调用出错则不会执行该逻辑）
      5. 将afterThrowing压入栈中，它先调用proceed，再在catch块中调用afterThrowing逻辑，随后将异常继续往上层抛出（如果发生异常，它并不处理异常，只是向上抛，这就实现了逻辑4的效果）
      6. 在最终的proceed中，则是调用被增强的方法执行相应逻辑，最后依次弹栈，并将之前暂存的对象重新设置到ThreadLocal中
4. 调用后得到一个返回值，如果返回的是this则返回其代理对象，否则直接将函数调用后产生的结果返回


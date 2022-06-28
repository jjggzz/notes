# 被aop代理过的方法执行流程

1. 调用被增强的方法首先会进入org.springframework.aop.framework.CglibAopProxy.DynamicAdvisedInterceptor#intercept方法，拦截目标方法的调用
2. 通过proxyFactory获取目标方法的拦截器链，如果没有拦截器链直接调用目标方法
   1. 判断获取到的拦截器是不是MethodInterceptor，如果是，直接加入拦截器链
   2. 如果不是则通过adapter适配成MethodInterceptor，并加入拦截器链
3. 如果有拦截器链则创建一个CglibMethodInvocation对象，把需要执行方法调用的对象、目标方法、拦截器链、目标的类对象等信息封装。并调用CglibMethodInvocation的proceed方法
   1. 
4. 调用后得到一个返回值，再通过processReturnType方法处理返回值
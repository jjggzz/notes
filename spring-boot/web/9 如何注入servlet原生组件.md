# 如何注入servlet原生组件

## 注意事项

这些注入的原生组建不会走springmvc的拦截器、参数解析、异常处理等逻辑，因为那些操作在DispatcherServlet中定义，而自定义的servlet与其同级。

## 注解方式

主启动类上标注@ServletComponentScan注解，扫描当前包及其子包的所有标注了@WebFilter、@WebListener、@WebServlet注解的类并将其注册（不需要@Component注解及器派生注解）

```java
// 扫描当前包及其子包
@ServletComponentScan
@SpringBootApplication
public class Main {

    public static void main(String[] args) {
        SpringApplication.run(Main.class,args);
    }
}
```

```java
// 处理/myservlet请求，并向写回hello world字符串
@WebServlet("/myservlet")
public class MyServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("hello world");

    }

```

```java
// 对/myservlet请求拦截并打印myFilter run，然后放行请求给后续拦截器
@WebFilter("/myservlet")
public class MyFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("myFilter run");
        chain.doFilter(request,response);
    }

    @Override
    public void destroy() {

    }
}
```

```java
// 监听请求的创建与销毁，每个请求都会被监听
@WebListener
public class MyListener implements ServletRequestListener {


    @Override
    public void requestDestroyed(ServletRequestEvent sre) {
        System.out.println("request destory");

    }

    @Override
    public void requestInitialized(ServletRequestEvent sre) {
        System.out.println("request init");
    }
}
```



## 使用RegistrationBean

```java
@Configuration
public class ServConfig {

    @Bean
    public ServletRegistrationBean<MyServlet> myServletServletRegistrationBean() {
        MyServlet myServlet = new MyServlet();
        return new ServletRegistrationBean<>(myServlet,"/myservlet");
    }

    @Bean
    public FilterRegistrationBean<MyFilter> myFilterFilterRegistrationBean() {
        MyFilter myFilter = new MyFilter();
        return new FilterRegistrationBean<>(myFilter,myServletServletRegistrationBean());
    }

    @Bean
    public ServletListenerRegistrationBean<MyListener> myListenerServletListenerRegistrationBean() {
        return new ServletListenerRegistrationBean<>(new MyListener());
    }

}
```


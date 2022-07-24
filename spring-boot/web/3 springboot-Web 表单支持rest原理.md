# springboot-Web 表单支持rest原理

1. 首先要明确的是这个的使用场景在于**前后端不分离的情况下表单提交**想要支持restful风格

2. 本身页面表单的提交是不支持DELETE、PUT等请求的

3. 可以通过spring.mvc.hiddenmethod.filter.enabled=true来开启表单restful风格支持

4. 使用方式就是在表单中携带一个隐藏域_method=value，value就是请求方式

5. 处理逻辑封装在filter实现类OrderedHiddenHttpMethodFilter中

   ```java
   	@Bean
   	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
   	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled")
   	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
   		return new OrderedHiddenHttpMethodFilter();
   	}
   ```

   

6. 如果是携带了特殊隐藏域的请求，它会将request域进行包装，并将隐藏域中制定的请求方式作为method返回

   ```java
   @Override
   	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
   			throws ServletException, IOException {
   
   		HttpServletRequest requestToUse = request;
   
   		if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
         // methodParam = _method
   			String paramValue = request.getParameter(this.methodParam);
   			if (StringUtils.hasLength(paramValue)) {
   				String method = paramValue.toUpperCase(Locale.ENGLISH);
           // 如果在允许的请求方式中则进行包装
           // ALLOWED_METHODS => PUT,DELETE,PATCH
   				if (ALLOWED_METHODS.contains(method)) {
             // 包装请求体，HttpMethodRequestWrapper重写了getMethod方法，返回隐藏域中制定的请求方式
   					requestToUse = new HttpMethodRequestWrapper(request, method);
   				}
   			}
   		}
   
   		filterChain.doFilter(requestToUse, response);
   	}
   ```

   

7. 如果是通过curl、postman等客户端或者是前后端分离的情况下前端的http请求框架发起的DELETE或者PUT请求则不再处理之列，因为他们本身就支持发送restful风格的请求

   

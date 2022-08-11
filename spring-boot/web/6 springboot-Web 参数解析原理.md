# springboot-Web 参数解析原理

1. 接上第4篇springboot-Web 请求映射原理，通过RequestMappingHandlerMapping获取到请求的具体HandlerMethod，并和拦截器组成了处理器执行链（HandlerExecutionChain）

2. 通过getHandlerAdapter方法获取能够处理HandlerMethod（注意此处并不是上一步获取到的处理器执行链）的适配器

   1. 其实就是遍历handlerAdapters，找到能够处理（HandlerAdapter.supports返回true）该handler的HandlerAdapter，并将其返回，找到即返回
   2. 此处返回的是RequestMappingHandlerAdapter，它用来处理标注了@RequestMapping注解的方法

3. 调用处理器执行链的前置处理方法（其实就是依次调用拦截器的preHandle方法）

4. 利用找到的RequestMappingHandlerAdapter适配器并调用它的handle方法去执行HandlerMethod

   1. handler => handleInternal => invokeHandlerMethod

   2. 在invokeHandlerMethod方法中先是将HandlerMethod封装成ServletInvocableHandlerMethod，并**设置参数解析器和返回值处理器**

      ```java
      	@Nullable
      	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
      			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
      
      		ServletWebRequest webRequest = new ServletWebRequest(request, response);
      		try {
      			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
      			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
      			// 将HandlerMethod封装为ServletInvocableHandlerMethod
      			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
            // 设置参数解析器，大约27种
      			if (this.argumentResolvers != null) {
      				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
      			}
            // 设置返回值处理器，大约15种
      			if (this.returnValueHandlers != null) {
      				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
      			}
      			
            // ...不关心的代码省略
      			
            // 发起方法调用
      			invocableMethod.invokeAndHandle(webRequest, mavContainer);
      			if (asyncManager.isConcurrentHandlingStarted()) {
      				return null;
      			}
      
      			return getModelAndView(mavContainer, modelFactory, webRequest);
      		}
      		finally {
      			webRequest.requestCompleted();
      		}
      	}
      ```

   3. 用ServletInvocableHandlerMethod的invokeAndHandle方法发起解析并调用

      ```java
      public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
      			Object... providedArgs) throws Exception {
      
        	// 发起调用并得到结果
      		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
      		setResponseStatus(webRequest);
        
      		// ...后续代码省略
      }
      ```

   4. invokeForRequest方法内部

      ```java
      	@Nullable
      	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      			Object... providedArgs) throws Exception {
      		// 解析并获取请求参数，这个是我们本次分析的重点
      		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
      		if (logger.isTraceEnabled()) {
      			logger.trace("Arguments: " + Arrays.toString(args));
      		}
          // 用我们写的controller作为目标对象，发起反射调用
      		return doInvoke(args);
      	}
      ```

   5. getMethodArgumentValues方法

      ```java
      protected Object[] getMethodArgumentValues(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer,
      			Object... providedArgs) throws Exception {
      		// 获取我们写的方法上的所有参数
      		MethodParameter[] parameters = getMethodParameters();
        	// 如果没有参数，自然不用解析什么参数，直接返回空数组就好了
      		if (ObjectUtils.isEmpty(parameters)) {
      			return EMPTY_ARGS;
      		}
      		
        	// 如果有参数列表，那么循环的获取设置参数列表的值，值由解析器解析得到
      		Object[] args = new Object[parameters.length];
      		for (int i = 0; i < parameters.length; i++) {
      			MethodParameter parameter = parameters[i];
      			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
      			args[i] = findProvidedArgument(parameter, providedArgs);
      			if (args[i] != null) {
      				continue;
      			}
            // 判断这些参数解析器中是否有解析器能够解析这个参数
            // 在内部就是循环调用HandlerMethodArgumentResolver.supportsParameter方法依次判断是否有解析器支持解析此参数
            // 内部使用了缓存提升判断速度
      			if (!this.resolvers.supportsParameter(parameter)) {
      				throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
      			}
      			try {
              // 发起值解析，并将其设置到对应的形参位置上
              // 内部实际上是通过上一步设置的缓存来获取值
      				args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, this.dataBinderFactory);
      			}
      			catch (Exception ex) {
      				// Leave stack trace for later, exception may actually be resolved and handled...
      				if (logger.isDebugEnabled()) {
      					String exMsg = ex.getMessage();
      					if (exMsg != null && !exMsg.contains(parameter.getExecutable().toGenericString())) {
      						logger.debug(formatArgumentError(parameter, exMsg));
      					}
      				}
      				throw ex;
      			}
      		}
        	// 最终将解析到的入参返回
      		return args;
      	
      ```

   6. 拿到入参列表，发起反射调用

5. 调用处理器执行链的后置处理方法（其实就是依次调用拦截器的postHandle方法）

6. 调用处理器执行链的完成后处理方法（其实就是依次调用拦截器的afterCompletion方法）



**要注意的是，解析@RequestBody注解的解析器是RequestResponseBodyMethodProcessor，内部使用了HttpMessageConverter的实现类来帮助转换，默认情况下mvc使用MappingJackson2HttpMessageConverter来进行json与java对象进行转换。我们可以通过org.springframework.web.servlet.config.annotation.WebMvcConfigurer#extendMessageConverters来注册我们自己的消息转换器，需要注意的是，消息转换器的遍历匹配是短路的**

**解析通用表单的解析器是ServletModelAttributeMethodProcessor，内部使用的是webDataBinder来进行数据绑定。webDataBinder里面使用GenericConverter的实现类来实现请求参数的转换，我们可以实现Converter<T,S>来自定义转换器（mvc通过适配器将我们写的Converter适配成GenericConverter），通过org.springframework.web.servlet.config.annotation.WebMvcConfigurer#addFormatters注册**


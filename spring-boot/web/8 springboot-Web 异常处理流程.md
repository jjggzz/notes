## springboot-Web 异常处理流程

1. 在Handler抛出异常后会被捕获，并将异常传入并调用processDispatchResult方法

   ```java
   		// ......
   		{
         // ......
   			try{
           // ......
           
   				// Actually invoke the handler.
   				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
   
   				// ......
   			catch (Exception ex) {
   				dispatchException = ex;
   			}
   			catch (Throwable err) {
   				// As of 4.3, we're processing Errors thrown from handler methods as well,
   				// making them available for @ExceptionHandler methods and other scenarios.
   				dispatchException = new NestedServletException("Handler dispatch failed", err);
   			}
         // 捕获异常并调用
   			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   		}
   		// ......
   		
   ```

2. 最终调用processHandlerException方法处理异常

   ```java
   private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
   			@Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
   			@Nullable Exception exception) throws Exception {
   
   		boolean errorView = false;
   
   		if (exception != null) {
   			if (exception instanceof ModelAndViewDefiningException) {
   				logger.debug("ModelAndViewDefiningException encountered", exception);
   				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
   			}
   			else {
   				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
           // 调用此方法处理异常
   				mv = processHandlerException(request, response, handler, exception);
   				errorView = (mv != null);
   			}
   		}
     	// ......
   }
   ```

3. processHandlerException方法中尝试通遍历handlerExceptionResolvers数组，通过HandlerExceptionResolver.resolveException方法来获取一个ModelAndView。但是默认情况下获取不到

   1. DefaultErrorAttributes只是**将异常设置到请求域中**
   2. 由于没有配置任何标注了@ExceptionHandler注解的方法（我们常用的全局异常处理方式最终就会配到这里），所以ExceptionHandlerExceptionResolver返回空
   3. 没有抛出标注了@ResponseStatus注解的异常，ResponseStatusExceptionResolver返回空
   4. 抛出的异常不属于DefaultHandlerExceptionResolver能处理的任何一个，所以也返回空

   ```java
   @Nullable
   	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
   			@Nullable Object handler, Exception ex) throws Exception {
   
   		// Success and error responses may use different content types
   		request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
   
   		// Check registered HandlerExceptionResolvers...
   		ModelAndView exMv = null;
   		if (this.handlerExceptionResolvers != null) {
         // 尝试获取一个ModelAndView，啥都不配的情况下这里获取的是null
   			for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
   				exMv = resolver.resolveException(request, response, handler, ex);
   				if (exMv != null) {
   					break;
   				}
   			}
   		}
   		if (exMv != null) {
   			if (exMv.isEmpty()) {
   				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
   				return null;
   			}
   			// We might still need view name translation for a plain error model...
   			if (!exMv.hasView()) {
   				String defaultViewName = getDefaultViewName(request);
   				if (defaultViewName != null) {
   					exMv.setViewName(defaultViewName);
   				}
   			}
   			if (logger.isTraceEnabled()) {
   				logger.trace("Using resolved error view: " + exMv, ex);
   			}
   			else if (logger.isDebugEnabled()) {
   				logger.debug("Using resolved error view: " + exMv);
   			}
   			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
   			return exMv;
   		}
   		// 异常继续往外抛，抛到了tomcat
   		throw ex;
   	}
   ```

4. 于是就将异常抛到外层了，tomcat拿到这个异常之后会在服务内部进行转发，将请求转发到/error页，而springboot自动配置（ErrorMvcAutoConfiguration）时配置了一个controller来处理这个请求。它返回了一个白页（就是我们经常看见的那个）

   ```java
   @Controller
   @RequestMapping("${server.error.path:${error.path:/error}}")
   public class BasicErrorController extends AbstractErrorController {
     
     // ......
     
     @RequestMapping(produces = MediaType.TEXT_HTML_VALUE)
   	public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {
   		HttpStatus status = getStatus(request);
   		Map<String, Object> model = Collections
   				.unmodifiableMap(getErrorAttributes(request, getErrorAttributeOptions(request, MediaType.TEXT_HTML)));
   		response.setStatus(status.value());
   		ModelAndView modelAndView = resolveErrorView(request, response, status, model);
       // error 视图就是那个白页同样是在ErrorMvcAutoConfiguration中配置
   		return (modelAndView != null) ? modelAndView : new ModelAndView("error", model);
     }
     
     // ......
   }
   ```

   
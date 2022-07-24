# springboot-Web 常用注解使用

1. 路径参数解析@PathVariable，可以解析路径上的某个值作为参数

   ```java
   		// curl -X GET http://localhost:8080/hello/zhangsan
   		// 解析到name的值为zhangsan
   		@GetMapping("hello/{name}")
       public String hello(@PathVariable("name") String name) {
           return name;
       }
   ```

2. 请求头解析@RequestHeader，可以解析请求头中的参数

   ```java
   		// curl -X GET -H "name:zhangsan" http://localhost:8080/hello
   		// 解析到name的值为zhangsan
   		@GetMapping("hello")
       public String hello(@RequestHeader("name") String name) {
           return name;
       }
   ```

3. 请求体解析@RequestParam，解析表单中的请求参数

   ```java
   		// curl -X GET http://localhost:8080/hello?name=zhangsan
   		// 解析到name的值为zhangsan
   		@GetMapping("hello")
       public String hello(@RequestParam("name") String name) {
           return name;
       }
   ```

4. cookie解析@CookieValue，解析cookie中的参数

   ```java
   		// curl -X GET --cookie "name=zhangsan" http://localhost:8080/hello
   		// 解析到name的值为zhangsan
   		@GetMapping("hello")
       public String hello(@CookieValue("name") String name) {
           return name;
       }
   ```

5. 请求域解析@RequestVariable，解析请求域中的参数

   ```java
   // 写一个普通的controller
   @Controller
   public class HelloController {
   
    		// curl -X GET http://localhost:8080/hello
       @GetMapping("hello")
       public String hello(HttpServletRequest request) {
         	// 请求域中设置一个name = zhangsan
           request.setAttribute("name","zhangsan");
   				// 转发到localhost:8080/world
           return "forward:world";
       }
   
     	// 解析到name的值为zhangsan
       @ResponseBody
       @GetMapping("world")
       public String world(@RequestAttribute("name") String name) {
           return name;
       }
   
   }
   ```

   

6. 请求体json解析@RequestBody，这个不用多说，最常用的

7. 矩阵变量解析@MatrixVariable，不常用，而且springboot默认情况下是关闭的，如果想开启需要定制化

   org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter#configurePathMatch

   ```java
   		@Override
   		public void configurePathMatch(PathMatchConfigurer configurer) {
   			if (this.mvcProperties.getPathmatch()
   					.getMatchingStrategy() == WebMvcProperties.MatchingStrategy.PATH_PATTERN_PARSER) {
   				configurer.setPatternParser(new PathPatternParser());
   			}
   			configurer.setUseSuffixPatternMatch(this.mvcProperties.getPathmatch().isUseSuffixPattern());
   			configurer.setUseRegisteredSuffixPatternMatch(
   					this.mvcProperties.getPathmatch().isUseRegisteredSuffixPattern());
   			this.dispatcherServletPath.ifAvailable((dispatcherPath) -> {
   				String servletUrlMapping = dispatcherPath.getServletUrlMapping();
   				if (servletUrlMapping.equals("/") && singleDispatcherServlet()) {
             // UrlPathHelper类是关键，需要将该类中的removeSemicolonContent参数设置为false
             // 可以考虑自己实现一个WebMvcConfigurer，然后重写这个方法，覆盖这一部分的配置
   					UrlPathHelper urlPathHelper = new UrlPathHelper();
   					urlPathHelper.setAlwaysUseFullPath(true);
   					configurer.setUrlPathHelper(urlPathHelper);
   				}
   			});
   		}
   ```

   

   
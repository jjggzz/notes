# springboot-Web 静态资源原理

1. springboot静态资源映射的配置是在autoconfigure模块中进行配置的

2. 主要配置逻辑在org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration.WebMvcAutoConfigurationAdapter类中的addResourceHandlers方法中

   ```java
   @Override
   		public void addResourceHandlers(ResourceHandlerRegistry registry) {
         // 静态资源路径映射开关
   			if (!this.resourceProperties.isAddMappings()) {
   				logger.debug("Default resource handling disabled");
   				return;
   			}
         // 首先将classpath:/META-INF/resources/webjars/路径映射为/webjars/**路径
   			addResourceHandler(registry, "/webjars/**", "classpath:/META-INF/resources/webjars/");
         // 然后将"classpath:/META-INF/resources/","classpath:/resources/", "classpath:/static/", "classpath:/public/"这四个路径映射到/**(可以通过spring.mvc.staticPathPattern来修改这一默认路径)
   			addResourceHandler(registry, this.mvcProperties.getStaticPathPattern(), (registration) -> {
   				registration.addResourceLocations(this.resourceProperties.getStaticLocations());
   				if (this.servletContext != null) {
   					ServletContextResource resource = new ServletContextResource(this.servletContext, SERVLET_LOCATION);
   					registration.addResourceLocations(resource);
   				}
   			});
   		}
   ```

   

3. 可以通过spring.web.resources.addMapping参数来全局控制是否需要开启静态资源路径映射

4. addResourceHandlers方法中首先将classpath:/META-INF/resources/webjars/路径映射为/webjars/**路径

5. 然后将"classpath:/META-INF/resources/","classpath:/resources/", "classpath:/static/", "classpath:/public/"这四个路径映射到/**(可以通过spring.mvc.staticPathPattern来修改这一默认路径)
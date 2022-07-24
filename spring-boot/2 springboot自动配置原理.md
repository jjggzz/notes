# springboot自动配置原理

1. 首先我们看一下@SpringBootApplication注解的签名，它是一个组合注解

   ```java
   @SpringBootConfiguration
   @EnableAutoConfiguration
   @ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
   		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
   public @interface SpringBootApplication {}
   ```

   

2. @SpringBootConfiguration中的@Configuration代表着被这个注解标注的类是一个配置类，具有配置类的功能

   ```java
   @Configuration
   @Indexed
   public @interface SpringBootConfiguration {}
   ```

   

3. @ComponentScan代表着扫描的包的路径为单前主启动类所在的包及其子包

   ```jade
   @ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
   		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
   ```

   

4. @EnableAutoConfiguration是springboot自动配置的关键

   ```java
   @AutoConfigurationPackage
   @Import(AutoConfigurationImportSelector.class)
   public @interface EnableAutoConfiguration {}
   ```

   1. @AutoConfigurationPackage注解其实就是将主启动类所在的包保存到了一个bean定义中（注意，springboot默认扫描主启动类所在的包及其子包并不是这个注解实现的，而是@ComponentScan实现的）
   2. 使用@Import导入了一个AutoConfigurationImportSelector类，它用来扫描MATA-INF下spring.factories文件中org.springframework.boot.autoconfigure.EnableAutoConfiguration这个key所指定的value，并将其解析

5. springboot会按需加载（@Conditional的派生注解来实现相关按需加载的功能）这些解析到的配置类
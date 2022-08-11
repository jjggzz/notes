# springboot自定义starter细节

## 命名

官方定义的starter一般命名为 spring-boot-starter-xxx，官方推荐我们自定义的starter命名为xxx-spring-boot-starter。

比如，spring官方的spring-boot-starter-web，而非官方的mybatis的starter包名为mybatis-spring-boot-starter

## 如何分模块

假设我要做一个名为helloworld的启动器（starter）

1. 首先建一个名为helloworld-spring-boot-autoconfigure的模块，这个模块里面包含了我们写的配置逻辑等等，并且包含我们自己写的xxxAutoConfiguration配置类（可以使用一些条件注解实现可覆盖的配置）
2. helloworld-spring-boot-autoconfigure模块如果有依赖其他第三方的包，最好阻断依赖的传递，这样本模块可以正常用。
3. 在helloworld-spring-boot-autoconfigure的resource/META-INF下的spring.factories中将自己写的xxxAutoConfiguration的全路径配置，key为org.springframework.boot.autoconfigure.EnableAutoConfiguration，这样在使用的时候才会被加载到容器中
4. 新建helloworld-spring-boot-starter，这个模块中什么代码都不写，只是在依赖管理文件中将我们写的helloworld-spring-boot-autoconfigure引入，并将用到的第三方包导入，使该依赖可以传递（因为autoconfigure包中用到的被阻断掉了）
5. 由于helloworld-spring-boot-starter导入了我们的autoconfigure模块，并且将用到的第三方模块导入了，用户在使用时**只导入helloworld-spring-boot-starter模块即可**

## 学习模板

其实很多spring-boot-starter-web和spring-boot-starter-autoconfigure的关系就是helloworld-spring-boot-starter和helloworld-spring-boot-autoconfigure的关系，可以参考其自动配置实现（在autoconfigure中实现自动配置的逻辑，starter-web包只是管理依赖）。可以看看它的源码，不过它是用gradle构建的。maven项目可以参考mybatis-spring-boot-starter的实现
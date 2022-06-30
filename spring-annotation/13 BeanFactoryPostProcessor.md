# BeanFactoryPostProcessor

从名字上可以理解它是bean工厂的后置处理器，应该可以对bean工厂做一些拓展操作的，事实上也确实如此。

1. 它在bean工厂标准初始化之后调用，此时所有的bean的定义已经解析完毕，所有BeanDefinitionRegistryPostProcessor调用完成，但是bean的实例还未创建

2. 由于这种特性所以可以在bean创建之前定义一些或者修改一些跟容器相关、跟bean定义相关的配置

3. 它主要在refresh()方法的invokeBeanFactoryPostProcessors方法这一步进行调用

   1. 先初始化并调用实现了PriorityOrdered接口的
   2. 在初始化并调用实现了Ordered接口的
   3. 在初始化并调用剩下的

4. 需要注意的是此时bean的后置处理器都没有加入到容器中，所以不能使用自动注入功能

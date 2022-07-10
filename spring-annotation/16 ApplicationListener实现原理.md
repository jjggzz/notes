# ApplicationListener实现原理

## 事件发布流程

1. 调用publishEvent方法，内部获取一个事件多播器，由它来实现事件派发
2. 在多播器中，通过事件获取该类型的所有监听器，循环调用其onApplicationEvent方法
3. 在循环中，判断是否有线程池，如果有则使用线程池异步调用，否则使用当前线程循环调用

## 事件多播器的初始化

1. 事件多播器在refresh方法的initApplicationEventMulticaster方法中初始化
2. 如果当前容器中有名为applicationEventMulticaster的事件派发器则使用它
3. 否则初始化一个SimpleApplicationEventMulticaster，并将它注册到容器中

## 怎么注册事件监听器

1. 事件监听器在refresh方法中的registerListeners方法注册的
2. 先从容器中获取所有类型为ApplicationListener的bean的名字（注意此处bean其实还没有实例化，它只是通过bean定义获取名字而已）并注册到多播器中


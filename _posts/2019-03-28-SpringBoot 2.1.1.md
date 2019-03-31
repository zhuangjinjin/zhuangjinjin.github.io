---
title: SpringBoot 2.1.1
tags: [Spring]
---



### SpringBoot事件

`SpringBoot事件其实不是什么新鲜事，是基于SpringFramework的事件机制`

* ApplicationEvent 事件(消息)
  * ApplicationStartingEvent
  * ApplicationEnvironmentPreparedEvent
  * ApplicationContextInitializedEvent
  * ApplicationPreparedEvent
  * ContextRefereshedEvent (SpringFramework)
  * ServletWebServerInitializedEvent
  * ApplicationStartedEvent
  * ApplicationReadyEvent
  * ContextClosedEvent (SpringFramework)
  * ApplicationFailedEvent
* ApplicationListener 事件监听器(消费者/订阅者)
* ApplicationEventMulticaster 事件发布者(生产者/发布者)
  * SimpleApplicationEventMulticaster (默认发布者实现类)

### Spring EventListener机制

* `@EventListener`

  > EventListenerMethodProcessor把EventListener注解的Method方法通过简单工厂方法包装ApplicationListenerMethodAdapter，然后加入到ApplicationContext中

```java
public class EventListenerMethodProcessor {
    
    ...
    
    private void processBean(final String beanName, final Class<?> targetType) {
        ...
        //拿到@EventListener注解的方法
        annotatedMethods = MethodIntrospector.selectMethods(targetType,
                            (MethodIntrospector.MetadataLookup<EventListener>) method ->
                                    AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));	
        ...
        for (Method method : annotatedMethods.keySet()) {
            ...
            //通过简单工厂模式创建出ApplicationListener
            ApplicationListener<?> applicationListener =
                                        factory.createApplicationListener(beanName, targetType, methodToUse);
            ...
            //加入到上下文中
            context.addApplicationListener(applicationListener);
        }
    }
    ...
}
```


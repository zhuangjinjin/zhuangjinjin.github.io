---
layout: page
title: SpringBoot 2.1.2事件
tags: [Spring]	
---

* TOC
{:toc}
-----

### Spring 事件

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

> Spring的事件缺省是同步的机制，但也提供异步的方式。其中异步机制采用线程池的方式来完成的，线程池由用户配置。

### Spring 事件注解

`Spring4.2之后提供了@EventListener，这让代码变得更优雅......`

#### @EventListener

​	上下文在启动的时候。初始化完整个bean工厂，并实例化所有单例bean，会把调用SmartInitializingSingleton#afterSingletonsInstantiated()。

​	EventListenerMethodProcessor把EventListener注解的Method方法通过简单工厂方法包装成ApplicationListenerMethodAdapter，然后加入到AbstractApplicationContext中

![EventListener注册时序图]({{site.url}}/images/EventListener注册时序图.png)

```java
public class EventListenerMethodProcessor implements SmartInitializingSingleton {
    
    ...
    public void afterSingletonsInstantiated() {
    	...
        String beanNames = beanFactory.getBeanNamesForType(Object.class);
      	//遍历处理每个bean里面的@EventListener
      	for (String beanName : beanNames) {
          	...
        	proccessBean(beanName, type);
          	...
      	}
        ...
    }
    
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
AbstractApplicationContext持有一个事件分发器。通过上下文把事件监听器注册到事件分发器上。

```java
public class AbstractApplicationContext {
  	...
	@Nullable
	private ApplicationEventMulticaster applicationEventMulticaster;

	private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();
  	...
    
    @Override
	public void addApplicationListener(ApplicationListener<?> listener) {
		Assert.notNull(listener, "ApplicationListener must not be null");
		if (this.applicationEventMulticaster != null) {
          	//向事件分发器注册
			this.applicationEventMulticaster.addApplicationListener(listener);
		}
		this.applicationListeners.add(listener);
	}
}
```


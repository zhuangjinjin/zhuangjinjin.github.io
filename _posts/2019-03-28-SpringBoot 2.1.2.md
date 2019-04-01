---
layout: post
title: SpringBoot 2.1.2事件
tags: [Spring]	
---

* TOC
{:toc}
-----

### SpringBoot 事件

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

**注意**:

Spring的事件缺省是同步的机制，但也提供异步的方式。其中异步机制采用线程池的方式来完成的，线程池由用户配置。

### Spring 事件注解

> As of Spring 4.2, you can register an event listener on any public method of a managed bean by using the `EventListener`annotation.

`Spring4.2之后`，你可以通过使用`@EventListener`注解在任何被托管的bean的公开方法注册一个事件监听器

### EventListener使用

```java
@EventListener({ContextStartedEvent.class, ContextRefreshedEvent.class})
public void handleContextStart() {
    ...
}
```

通常我们有时候会需要通过条件属性去过滤事件，你只要编写一下[SpEL expression](https://docs.spring.io/spring/docs/5.1.4.RELEASE/spring-framework-reference/core.html#expressions)，它能够让你针对特定的条件去执行特定的处理

```java
@EventListener(condition = "#blEvent.content == 'my-event'")
public void processBlackListEvent(BlackListEvent blEvent) {
    ...
}
```

以下列出Event SpEL的元数据

![Event SpEL]({{site.url}}/images/Event_SpEL.png)

如果你想收到`事件A`后发布`事件B`，你可以修改你的参数签名返回一个事件B

```java
@EventListener
public ListUpdateEvent handleBlackListEvent(BlackListEvent event) {
    // notify appropriate parties via notificationAddress and
    // then publish a ListUpdateEvent...
}
```

> **注意**：这个特性不适合异步事件监听器

如果想返回一系列的事件，你可以在返回Collection<ApplicationEvent>

#### 异步事件监听器

如果你想使用特定的监听器去异步处理事件。你需要使用`@Async`支持，需要开启`@EnableAsync`

```java
@EventListener
@Async
public void processBlackListEvent(BlackListEvent event) {
    //BlackListEvent is processed in a separate thread
}
```

当使用异步化事件需要注意如下限制：

* 如果事件监听器抛出异常，这个异常是不会传播给调用者，也就是调用者不会捕获到这个异常，具体细节可以查看`AsyncUncaughtExceptionHandler`
* 此类事件监听器无法发送回复，如果有这个需求，请注入`ApplicationEventPublisher`手动发送事件

#### 顺序事件监听器

如果你有两个事件监听器同时监听一个事件，并且是有顺序要求，可以使用`@Order`

```java
@EventListener
@Order(42)
public void processBlackListEvent(BlackListEvent event) {
    //Notify appropriate parties via notificationAddress...
}
```

#### 通用事件

通用事件即自定义事件，比如EntityCreatedEvent<T>建议T是一种确切的实体类型。举个例子，你只需要接收EntityCreatedEvent事件且事件源是Person的

```java
@EventListener
public void onPersonCreated(EntityCreatedEvent<Person> event) {
    ...
}
```

在某些情况下，如果所有事件的事件源都是同一种结构体，那操作起来可能会有点繁琐。这种情况下，你可以实现`ResolvableTypeProvider`来引导框架提供超出运行时环境提供的范围。

```java
public class EntityCreatedEvent<T> extends ApplicationEvent implements ResolvableTypeProvider {
    public EntityCreatedEvent(T entity) {
    	super(entity);
    }
  
    @Override
    public ResolvableType getResolvableType() {
        return ResolvableType.forClassWithGenerics(getClass(), ResolvableType.forInstance(getSource()));
    }
}
```

> 这个不仅适用于ApplicationEvent，也适用于事件发送的任意对象

### 	EventListener原理

#### 事件监听器的注册

​	上下文在启动的时候。初始化完整个bean工厂，并实例化所有单例bean，会把调用SmartInitializingSingleton的afterSingletonsInstantiated()方法。

​	EventListenerMethodProcessor把EventListener注解的Method方法通过简单工厂方法包装成ApplicationListenerMethodAdapter，然后加入到AbstractApplicationContext中

![EventListener注册时序图]({{site.baseurl}}/images/EventListener注册时序图.png)

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
			this.applicationEventMulticaster
              .addApplicationListener(listener);
		}
		this.applicationListeners.add(listener);
	}
}
```

#### 异步机制

Spring异步是通过aop实现的，缺省的织入模式是`AdviceMode.PROXY`。会自动导入`AsyncAnnotationBeanPostProcessor`。`AsyncAnnotationBeanPostProcessor`持有`AsyncAnnotationAdvisor`对象。`AsyncAnnotationAdvisor`会创建一个`AnnotationAsyncExecutionInterceptor`拦截器，所有`@Async`的方法，在调用的时候，都会被这个拦截器拦截，并拿到一个`AsyncTaskExecutor`线程池，在线程池中执行原方法。

带`AsyncAnnotationAdvisor`的proxy加载过程

![AsyncAnnotationAdvisor的proxy加载时序图]({{site.baseurl}}/images/Async时序图.png)

`@Order`顺序事件监听器原理

```java
public class ApplicationListenerMethodAdapter implements GenericApplicationListener {
 	...
    public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass, Method method) {
        ...
        //事件监听器如果没有配置@Order，默认是0
        this.order = resolveOrder(method);
    }
    ...
    private int resolveOrder(Method method) {
		Order ann = AnnotatedElementUtils.findMergedAnnotation(method, Order.class);
		return (ann != null ? ann.value() : 0);
	}
}
```

事件分发器在分发事件的时候，会对事件监听器集合进行排序，然后对监听器顺序分发事件。以下是时序图

![OrderEventListener时序图]({{site.baseurl}}/images/OrderEventListener时序图.png)
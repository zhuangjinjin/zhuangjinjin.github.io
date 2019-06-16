---
layout: post
title: OpenFeign之服务调用
tags: [Spring, Spring Cloud] 
---

* TOC
{:toc}
-----

`OpenFeign`是一个声明式的服务调用，通常集成`Ribbon`来实现客户端负载均衡。

### 示例

我们先写个简单的基于`OpenFeign`的客户端的示例：

这是一个典型的SpringBoot启动类

```java
package io.github.ukuz.study.cloud.feign;

@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class FeignClientApplication {

    public static void main(String[] args) {
        new SpringApplicationBuilder(FeignClientApplication.class).web(WebApplicationType.SERVLET).run(args);
    }

    @Autowired
    private TradeRemoteService tradeRemoteService;

    @PutMapping("/pay")
    public TradeInfo pay(@RequestParam String uid, @RequestParam int payMoney) {
        String uuid = UUID.randomUUID().toString().replaceAll("-", "");
        return tradeRemoteService.pay(uuid, uid, payMoney);
    }

    @Bean
    public Retryer retryer() {
        return new Retryer.Default(5000, TimeUnit.SECONDS.toMillis(5), 5);
    }

}
```

接下来再写个简易的远程调用接口：

```java
package io.github.ukuz.study.cloud.feign.remote;

@FeignClient(value = "spring-cloud-server-application", path = "/trade")
public interface TradeRemoteService {

    @PutMapping("/pay")
    TradeInfo pay(@RequestHeader(value = Constant.X_REQ_SEQ_ID) String uuid, @RequestParam(value = "uid") String uid, @RequestParam(value = "payMoney") int payMoney);

}
```

### 原理

通过上面示例，我们可以看出除了加了一个`@EnableFeignClients`和`@FeignClient`注解外，别无其他。那么它是怎么让一个接口在没有定义实现类的情况下，就可以实现远程调用呢。这里首先我们会猜测这应该是用到Java的动态代理机制，而代理对象帮我们来完成远程调用并且返回。那么事实上是不是这样呢，我们来一探究竟。

#### 初始化客户端

在客户端应用程序中声明完`@EnableFeignClients`后，会自动导入FeignClientsRegistrar， 然后FeignClientRegistrar通过`ClassPathScanningCandidateComponetProvider`扫描出所有带`@FeignClient`注解的BeanDefinition ，并且把它重新构造成一个`FeignClientFactoryBean`。如上面当注入TradeRemoteService时，会去调用FeignClientFactoryBean#getObject。

通过阅读FeignClientFactoryBean源码，最终会产生一个代理类`TradeRemoteSerivce$Proxy`，这个代理类里面会有y一个方法映射map，用来每个接口方法映射一个SynchronousMethodHandler ，具体可以参见`ReflectiveFeign.FeignInvocationHandler`。

#### 远程调用

前面我们了解到，实际上我们最终是拿到一个接口的代理类，接下来我们来看看整个调用时序图

![TradeRemoteService调用时序图]({{site.baseUrl}}/images/TradeRemoteService调用时序图.png)

SynchronousMethodHandler是代理接口的方法的具体实现：

```java
package feign;

final class SynchronousMethodHandler implements MethodHandler {
    ...
    @Override
    public Object invoke(Object[] argv) throws Throwable {
        RequestTemplate template = buildTemplateFromArgs.create(argv);
        Retryer retryer = this.retryer.clone();
        while (true) {
            try {
                //执行远程调用
                return executeAndDecode(template);
            } catch (RetryableException e) {
                //如果出现调用异常，执行容错处理，重试
                retryer.continueOrPropagate(e);
                if (logLevel != Logger.Level.NONE) {
                    logger.logRetry(metadata.configKey(), logLevel);
                }
                continue;
            }
        }
    }
    ...
}
```



#### 重要的类

##### DynamicServerListLoadBalancer#updateListOfServers

##### ZookeeperServerIntrospector (Zookeeper的服务自省器)

##### FeignClientFactoryBean

##### ReflectiveFeign

##### LoadBalanceFeignClient

##### NameContextFactory

##### SynchronousMethodHandler

#### 
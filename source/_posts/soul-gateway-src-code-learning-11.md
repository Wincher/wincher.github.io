---
title: soul_gateway_src_code_learning_11
date: 2021-01-26 15:35:36
tags: SOUL

---

### 一个请求是如何被SOUL处理的

至此, soul的基本使用方式已经清楚了, 接下来我们带着标题的疑问来看 一个请求是如何被SOUL处理的

基于以前的结果,我们已知 会被 SoulWebHandler处理, SoulWebHandler实现了WebHandler, 我们来看下WebHandler的作用,

![img](00webhandler.png)

注释写的很清楚, WebHandler用来处理request的, 我们知道  handler()方法处理的ServerWebExchange 是WebFlux 中对request, response的封装,那么我们定义的SoulWebHandler是如何注入到上下文中, 被Spring Flux调用的呢

那WebFlux是什么呢, 看看官方的说明

> Spring WebFlux is a non-blocking web framework built from the ground up to take advantage of multi-core, next-generation processors and handle massive numbers of concurrent connections.

**Spring WebFlux 是一个异步非阻塞式的 Web 框架，它能够充分利用多核 CPU 的硬件资源去处理大量的并发请求**

显然这对比Spring MVC 所构建在servlet api上的阻塞IO模型性能要高

soul-plugin/soul-plugin-api/pom.xml

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-webflux</artifactId>
</dependency>
```

我们封装的核心接口也都是使用ServerWebExchange来作为request, response的载体

soul-web/src/main/java/org/dromara/soul/web/configuration/SoulConfiguration.java

```java
 @Bean("webHandler")
    public SoulWebHandler soulWebHandler(final ObjectProvider<List<SoulPlugin>> plugins) {
        List<SoulPlugin> pluginList = plugins.getIfAvailable(Collections::emptyList);
      //初始化也利用ObjectProvider 讲所有加载进来的插件注入到SoulWebHandler中
        final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
        soulPlugins.forEach(soulPlugin -> log.info("load plugin:[{}] [{}]", soulPlugin.named(), soulPlugin.getClass().getName()));
        return new SoulWebHandler(soulPlugins);
    }
```

系统启动 webHandler会被加载进内存, 那当一个请求调用进来是哪里调用的handle方法呢, 简单的在所有调用SoulWebHandler.handler()的地方打上断点, 发现是org/springframework/web/server/handler/DefaultWebFilterChain.java的

```java
@Override
	public Mono<Void> filter(ServerWebExchange exchange) {
		return Mono.defer(() ->
				this.currentFilter != null && this.chain != null ?
						invokeFilter(this.currentFilter, this.chain, exchange) :
						this.handler.handle(exchange));
	}
```

调用了, 这里的handler正是SoulWebHandler, 追其根本,通过 org/springframework/web/server/adapter/WebHttpHandlerBuilder.java 中

```java
WebHttpHandlerBuilder builder = new WebHttpHandlerBuilder(
//这里获取到的就是我们前面创建的Bean SoulWebHandler
				context.getBean(WEB_HANDLER_BEAN_NAME, WebHandler.class), context);
```

不过这部分已经到了WebFlux初始化的层面了,就此打住, 接下来我们就知道了一个请求来到SOUL, 本质上是SOUL借助WebFlux 高性能 reacot模型的web框架, 扩展了一个WebHandler, 在Handler中对请求做pluginchain的处理,并最终代理访问真实地址, 实现网关功能.
---
title: soul_gateway_src_code_learning_16
date: 2021-02-01 23:50:07
tags: SOUL
---

Hystrix plugin 分析

这是一段对Hystrix的介绍, 目前该项目已经不再开发, 只有维护

>  Hystrix is a latency and fault tolerance library designed to isolate points of access to remote systems, services and 3rd party libraries, stop cascading failure and enable resilience in complex distributed systems where failure is inevitable.
>
> Hystrix是一个延迟和容错库，旨在隔离对远程系统，服务和第三方库的访问点，停止级联故障，并在不可避免发生故障的复杂分布式系统中实现弹性。

一切正常, 启动bootstrap中加入引用

```xml
 <!-- soul hystrix plugin start-->
  <dependency>
      <groupId>org.dromara</groupId>
      <artifactId>soul-spring-boot-starter-plugin-hystrix</artifactId>
       <version>${last.version}</version>
  </dependency>
  <!-- soul hystrix plugin end-->
```



并在管理页面开启  hystrix,为开启, 并配置规则

- 跳闸最小请求数量: 少达到这个最小请求书才会触发熔断
- 错误百分比阀值: 一段短时间内错误响应打到百分比会熔断
- 跳闸休眠时间: 熔断后恢复时间
- 分组Key: 一般为:contextPath,默认为divide的selector名
- 命令Key: 一般为具体的路径接口,默认为divide规则名
- 失败降级URL: 降级调用的的url

HystrixPlugin 核心逻辑

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        // 获取熔断配置
        final HystrixHandle hystrixHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), HystrixHandle.class);
        if (StringUtils.isBlank(hystrixHandle.getGroupKey())) {
            hystrixHandle.setGroupKey(Objects.requireNonNull(soulContext).getModule());
        }
        if (StringUtils.isBlank(hystrixHandle.getCommandKey())) {
            hystrixHandle.setCommandKey(Objects.requireNonNull(soulContext).getMethod());
        }
        Command command = fetchCommand(hystrixHandle, exchange, chain);
        return Mono.create(s -> {
            Subscription sub = command.fetchObservable().subscribe(s::success,
                    s::error, s::success);
            s.onCancel(sub::unsubscribe);
            // 断路器 判断是否熔断
            if (command.isCircuitBreakerOpen()) {
                log.error("hystrix execute have circuitBreaker is Open! groupKey:{},commandKey:{}", hystrixHandle.getGroupKey(), hystrixHandle.getCommandKey());
            }
        }).doOnError(throwable -> {
            log.error("hystrix execute exception:", throwable);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.ERROR.getName());
            chain.execute(exchange);
        }).then();
    }
```

```java
private Command fetchCommand(final HystrixHandle hystrixHandle, final ServerWebExchange exchange, final SoulPluginChain chain) {
        if (hystrixHandle.getExecutionIsolationStrategy() == HystrixIsolationModeEnum.SEMAPHORE.getCode()) {
            return new HystrixCommand(HystrixBuilder.build(hystrixHandle),
                exchange, chain, hystrixHandle.getCallBackUri());
        }
        return new HystrixCommandOnThread(HystrixBuilder.buildForHystrixCommand(hystrixHandle),
            exchange, chain, hystrixHandle.getCallBackUri());
    }
```

TODO: 分析fetchCommand具体实现逻辑
---
title: soul_gateway_src_code_learning_17
date: 2021-02-04 18:49:23
tags: SOUL
---

### SOUL resilience4j plugin

首先启动项目 soul-admin, soul-bootstrap，以`soul-examples`中的`soul-example-http`为例，注册到soul网关上。检查soul-bootstrap的`pom`文件中是否引入相关依赖：

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-resilience4j</artifactId>
            <version>${project.version}</version>
        </dependency>
```

然后再SOUL admin中开启 resilient4j plugin

同时配置selector以及相关rule, 制定相应的熔断策略, 相关 selector 配置

[reference](https://dromara.org/zh/projects/soul/resilience4j-plugin/)

# 探究Resilient4j插件

Resilient4j plugin与其他插件一样继承了AbstractSoulPlugin, 核心逻辑如下

```java
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
      //soul-admin同步来的json串生成Resilience4JHandle, 包含所有在soul-admin中的相关参数信息
        Resilience4JHandle resilience4JHandle = GsonUtils.getGson().fromJson(rule.getHandle(), Resilience4JHandle.class);
      //然后根据getCircuitEnable来确定是否创建熔断和限流的组合控件还是仅创建限流控件
        if (resilience4JHandle.getCircuitEnable() == 1) {
            return combined(exchange, chain, rule);
        }
        return rateLimiter(exchange, chain, rule);
    }
```

 CombinedExecutor和 RateLimiterExecutor
对于combined方法，核心代码如下

```java
private Mono<Void> rateLimiter(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        return ratelimiterExecutor.run(
                chain.execute(exchange), fallback(ratelimiterExecutor, exchange, null), Resilience4JBuilder.build(rule))
                .onErrorResume(throwable -> ratelimiterExecutor.withoutFallback(exchange, throwable));
    }

    private Mono<Void> combined(final ServerWebExchange exchange, final SoulPluginChain chain, final RuleData rule) {
        Resilience4JConf conf = Resilience4JBuilder.build(rule);
        return combinedExecutor.run(
                chain.execute(exchange).doOnSuccess(v -> {
                    HttpStatus status = exchange.getResponse().getStatusCode();
                    if (status == null || !status.is2xxSuccessful()) {
                        exchange.getResponse().setStatusCode(null);
                        throw new CircuitBreakerStatusCodeException(status == null ? HttpStatus.INTERNAL_SERVER_ERROR : status);
                    }
                }), fallback(combinedExecutor, exchange, conf.getFallBackUri()), conf);
    }
```

 combinedExecutor 具体逻辑

```java
public class CombinedExecutor implements Executor {

    @Override
    public <T> Mono<T> run(final Mono<T> run, final Function<Throwable, Mono<T>> fallback, final Resilience4JConf resilience4JConf) {
      //调用 Resilience4J 创建熔断器和限流器
        RateLimiter rateLimiter = Resilience4JRegistryFactory.rateLimiter(resilience4JConf.getId(), resilience4JConf.getRateLimiterConfig());
        CircuitBreaker circuitBreaker = Resilience4JRegistryFactory.circuitBreaker(resilience4JConf.getId(), resilience4JConf.getCircuitBreakerConfig());
        Mono<T> to = run.transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                .transformDeferred(RateLimiterOperator.of(rateLimiter))
                .timeout(resilience4JConf.getTimeLimiterConfig().getTimeoutDuration())
                .doOnError(TimeoutException.class, t -> circuitBreaker.onError(
                        resilience4JConf.getTimeLimiterConfig().getTimeoutDuration().toMillis(),
                        TimeUnit.MILLISECONDS,
                        t));
        if (fallback != null) {
            to = to.onErrorResume(fallback);
        }
        return to;
    }
}
```



上面的 rateLimit和combined 方法都可以看到Resilience4JBuilder来生成Resilience4JConf, 这里通过SOUL的配置构建TimeLimiterConfig, CircuitBreakerConfig,RateLimiterConfig供调用Resilience4J使用

```java
public static Resilience4JConf build(final RuleData ruleData) {
        Resilience4JHandle handle = GsonUtils.getGson().fromJson(ruleData.getHandle(), Resilience4JHandle.class);
        CircuitBreakerConfig circuitBreakerConfig = null;
        if (handle.getCircuitEnable() == 1) {
            circuitBreakerConfig = CircuitBreakerConfig.custom()
                    .recordExceptions(Throwable.class, Exception.class)
                    .failureRateThreshold(handle.getFailureRateThreshold())
                    .automaticTransitionFromOpenToHalfOpenEnabled(handle.getAutomaticTransitionFromOpenToHalfOpenEnabled())
                    .slidingWindow(handle.getSlidingWindowSize(), handle.getMinimumNumberOfCalls(),
                            handle.getSlidingWindowType() == 0
                                    ? CircuitBreakerConfig.SlidingWindowType.COUNT_BASED
                                    : CircuitBreakerConfig.SlidingWindowType.TIME_BASED).waitIntervalFunctionInOpenState(IntervalFunction
                            .of(Duration.ofSeconds(handle.getWaitIntervalFunctionInOpenState() / 1000)))
                    .permittedNumberOfCallsInHalfOpenState(handle.getPermittedNumberOfCallsInHalfOpenState()).build();
        }
        TimeLimiterConfig timeLimiterConfig = TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(handle.getTimeoutDuration() / 1000)).build();
        RateLimiterConfig rateLimiterConfig = RateLimiterConfig.custom()
                .limitForPeriod(handle.getLimitForPeriod())
                .timeoutDuration(Duration.ofSeconds(handle.getTimeoutDurationRate() / 1000))
                .limitRefreshPeriod(Duration.ofNanos(handle.getLimitRefreshPeriod() * 1000000)).build();
        return new Resilience4JConf(Resilience4JHandler.getResourceName(ruleData), handle.getFallbackUri(), rateLimiterConfig, timeLimiterConfig, circuitBreakerConfig);
    }
```




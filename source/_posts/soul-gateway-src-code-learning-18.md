---
title: soul_gateway_src_code_learning_18
date: 2021-02-05 22:12:00
tags: SOUL
---

### Sentinel Plugin

[reference]https://dromara.org/zh/projects/soul/sentinel-plugin/

在bootstrap pom.xml 中添加 sentinel的支持

```XML
        <!-- soul sentinel plugin start-->
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-plugin-sentinel</artifactId>
            <version>${project.version}</version>
        </dependency>
        <!-- soul sentinel plugin end-->
```

然后再SOUL admin中开启 Sentinel plugin

同时配置selector以及相关rule, 制定相应的熔断策略, 相关 selector 配置

Sentinel plugin与其他插件一样继承了AbstractSoulPlugin, 核心逻辑如下

```java

    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        String resourceName = SentinelRuleHandle.getResourceName(rule);
        SentinelHandle sentinelHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SentinelHandle.class);
        return chain.execute(exchange).transform(new SentinelReactorTransformer<>(resourceName)).doOnSuccess(v -> {
            HttpStatus status = exchange.getResponse().getStatusCode();
            if (status == null || !status.is2xxSuccessful()) {
                exchange.getResponse().setStatusCode(null);
                throw new SentinelFallbackException(status == null ? HttpStatus.INTERNAL_SERVER_ERROR : status);
            }
        }).onErrorResume(throwable -> sentinelFallbackHandler.fallback(exchange, UriUtils.createUri(sentinelHandle.getFallbackUri()), throwable));
    }
```

这里使用 transform 加入一个函数作用于所有订阅者

SentinelReactorTransformer 是Sentinel提供的主要的熔断实现类,会在所有插件执行完毕后执行,实现完美的解耦

SentinelReactorTransformer 继承了 publisher, 使用发布订阅的工作模式, 对应订阅类 SentinelReactorSubscriber, 在执行插件链的最后执行 SentinelReactorSubscriber 中的 hookOnSubscribe(),之后完全交给 Sentinal 做处理

```java
   @Override
    protected void hookOnSubscribe(Subscription subscription) {
        doWithContextOrCurrent(() -> currentContext().getOrEmpty(SentinelReactorConstants.SENTINEL_CONTEXT_KEY),
            this::entryWhenSubscribed);
    }
```

这个里面用到大量 webflux 编程，后续搞定出Reactor


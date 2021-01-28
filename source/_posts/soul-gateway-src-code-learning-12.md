---
title: soul_gateway_src_code_learning_12
date: 2021-01-28 00:56:35
tags: SOUL
---

### SOUL: divide plugin

通过上一节的分析, 我们知道了请求被plugin chain处理是通过 SoulWebHandler 中的静态类 DefaultSoulPluginChain 来处理的

```java
@Override
        public Mono<Void> execute(final ServerWebExchange exchange) {
            return Mono.defer(() -> {
                if (this.index < plugins.size()) {
                    SoulPlugin plugin = plugins.get(this.index++);
                    Boolean skip = plugin.skip(exchange);
                    if (skip) {
                        return this.execute(exchange);
                    }
                    return plugin.execute(exchange, this);
                }
                return Mono.empty();
            });
        }
```

soul-web/src/main/java/org/dromara/soul/web/configuration/SoulConfiguration.java 中注入plugin使用order做排序

```java
final List<SoulPlugin> soulPlugins = pluginList.stream()
                .sorted(Comparator.comparingInt(SoulPlugin::getOrder)).collect(Collectors.toList());
```



PluginEnum定义了插件的执行顺序,

```java
public enum PluginEnum {
    GLOBAL(1, 0, "global"),
    SIGN(2, 0, "sign"),
    WAF(10, 0, "waf"),
    RATE_LIMITER(20, 0, "rate_limiter"),
    CONTEXTPATH_MAPPING(25, 0, "context_path"),
    REWRITE(30, 0, "rewrite"),
    REDIRECT(40, 0, "redirect"),
    HYSTRIX(45, 0, "hystrix"),
    SENTINEL(45, 0, "sentinel"),
    RESILIENCE4J(45, 0, "resilience4j"),
    DIVIDE(50, 0, "divide"),
    SPRING_CLOUD(50, 0, "springCloud"),
    WEB_SOCKET(55, 0, "webSocket"),
    DUBBO(60, 0, "dubbo"),
    SOFA(60, 0, "sofa"),
    TARS(60, 0, "tars"),
    MONITOR(80, 0, "monitor"),
    RESPONSE(100, 0, "response");
}
```

这一节来分析divide插件的处理逻辑

```java

protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        final DivideRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), DivideRuleHandle.class);
  //从缓存中拿到后端代理
        final List<DivideUpstream> upstreamList = UpstreamCacheManager.getInstance().findUpstreamListBySelectorId(selector.getId());
        if (CollectionUtils.isEmpty(upstreamList)) {
            log.error("divide upstream configuration error： {}", rule.toString());
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final String ip = Objects.requireNonNull(exchange.getRequest().getRemoteAddress()).getAddress().getHostAddress();
  //通过负载均衡算法拿到upstream
        DivideUpstream divideUpstream = LoadBalanceUtils.selector(upstreamList, ruleHandle.getLoadBalance(), ip);
        if (Objects.isNull(divideUpstream)) {
            log.error("divide has no upstream");
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        // set the http url
        String domain = buildDomain(divideUpstream);
        String realURL = buildRealURL(domain, soulContext, exchange);
  //对请求赋值真实访问的地址
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        // set the http timeout
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        exchange.getAttributes().put(Constants.HTTP_RETRY, ruleHandle.getRetry());
        return chain.execute(exchange);
    }
```

具体的负载均衡算法 实现了 LoadBalance 接口

![img](00loadbalance_impls.png)

可以看到有三种实现, 分别是hash, random和roundrobin, 轮询算法, 通过SPI注入机制, soul自定义了加在spi的ExtensionLoader, 后面会单独分析

另外网关遇到知道upstream的状态, 这里就涉及到了探活机制

具体实现在 UpstreamCheckService

```java
//此注解自动在Bean初始化后执行
@PostConstruct
    public void setup() {
        PluginDO pluginDO = pluginMapper.selectByName(PluginEnum.DIVIDE.getName());
      //初始化divide插件配置的所有upstream添加到UPSTREAM_MAP中
        if (pluginDO != null) {
            List<SelectorDO> selectorDOList = selectorMapper.findByPluginId(pluginDO.getId());
            for (SelectorDO selectorDO : selectorDOList) {
                List<DivideUpstream> divideUpstreams = GsonUtils.getInstance().fromList(selectorDO.getHandle(), DivideUpstream.class);
                if (CollectionUtils.isNotEmpty(divideUpstreams)) {
                    UPSTREAM_MAP.put(selectorDO.getName(), divideUpstreams);
                }
            }
        }
      //check属性使我们在application.yml中定义的soul.upstream.check值
        if (check) {
            new ScheduledThreadPoolExecutor(Runtime.getRuntime().availableProcessors(), SoulThreadFactory.create("scheduled-upstream-task", false))
              //使用线程池对每一个upstream执行定时任务, 下面看下任务的具体逻辑
                    .scheduleWithFixedDelay(this::scheduled, 10, scheduledTime, TimeUnit.SECONDS);
        }
    }
```



```java
private void scheduled() {
        if (UPSTREAM_MAP.size() > 0) {
            UPSTREAM_MAP.forEach(this::check);
        }
}

private void check(final String selectorName, final List<DivideUpstream> upstreamList) {
        List<DivideUpstream> successList = Lists.newArrayListWithCapacity(upstreamList.size());
        for (DivideUpstream divideUpstream : upstreamList) {
          //方法中的核心逻辑使用Socket.connetct方法查看连接是否异常
            final boolean pass = UpstreamCheckUtils.checkUrl(divideUpstream.getUpstreamUrl());
            if (pass) {
              
  //对结果进行处理,更新upstream状态
                if (!divideUpstream.isStatus()) {
                    divideUpstream.setTimestamp(System.currentTimeMillis());
                    divideUpstream.setStatus(true);
                    log.info("UpstreamCacheManager check success the url: {}, host: {} ", divideUpstream.getUpstreamUrl(), divideUpstream.getUpstreamHost());
                }
                successList.add(divideUpstream);
            } else {
                divideUpstream.setStatus(false);
                log.error("check the url={} is fail ", divideUpstream.getUpstreamUrl());
            }
        }
        if (successList.size() == upstreamList.size()) {
            return;
        }
        if (successList.size() > 0) {
          //如果upstream并没有通过check, 更新新的成功连接的upstream list进入缓存
            UPSTREAM_MAP.put(selectorName, successList);
            updateSelectorHandler(selectorName, successList);
        } else {
            UPSTREAM_MAP.remove(selectorName);
            updateSelectorHandler(selectorName, null);
        }
    }

//通知DataChanged
    private void updateSelectorHandler(final String selectorName, final List<DivideUpstream> upstreams) {
        SelectorDO selectorDO = selectorMapper.selectByName(selectorName);
        if (Objects.nonNull(selectorDO)) {
            List<ConditionData> conditionDataList = ConditionTransfer.INSTANCE.mapToSelectorDOS(
                    selectorConditionMapper.selectByQuery(new SelectorConditionQuery(selectorDO.getId())));
            PluginDO pluginDO = pluginMapper.selectById(selectorDO.getPluginId());
            String handler = CollectionUtils.isEmpty(upstreams) ? "" : GsonUtils.getInstance().toJson(upstreams);
            selectorDO.setHandle(handler);
            selectorMapper.updateSelective(selectorDO);
            if (Objects.nonNull(pluginDO)) {
                SelectorData selectorData = SelectorDO.transFrom(selectorDO, pluginDO.getName(), conditionDataList);
                selectorData.setHandle(handler);
                // publish change event.
                eventPublisher.publishEvent(new DataChangedEvent(ConfigGroupEnum.SELECTOR, DataEventTypeEnum.UPDATE,
                                                                 Collections.singletonList(selectorData)));
            }
        }
    }
```

TODO;更细致的分析
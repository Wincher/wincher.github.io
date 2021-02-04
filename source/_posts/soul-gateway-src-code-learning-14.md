---
title: soul_gateway_src_code_learning_14
date: 2021-01-30 02:36:35
tags: SOUL
---

### SOUL: alibaba-dubbo apache-dubbo plugin 分析

dubbo插件主要讲http 请求 转化为dubbo请求, 实现泛化调用 soul-plugin/soul-plugin-apache-dubbo/src/main/java/org/dromara/soul/plugin/apache/dubbo/ApacheDubboPlugin.java

```java
@Override
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        String body = exchange.getAttribute(Constants.DUBBO_PARAMS);
        SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
      //  可以看到与divide插件, SpringCLoud插件的区别是这里用到了Metadata
        MetaData metaData = exchange.getAttribute(Constants.META_DATA);
        if (!checkMetaData(metaData)) {
            assert metaData != null;
            log.error(" path is :{}, meta data have error.... {}", soulContext.getPath(), metaData.toString());
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.META_DATA_ERROR.getCode(), SoulResultEnum.META_DATA_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        if (StringUtils.isNoneBlank(metaData.getParameterTypes()) && StringUtils.isBlank(body)) {
            exchange.getResponse().setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
            Object error = SoulResultWrap.error(SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getCode(), SoulResultEnum.DUBBO_HAVE_BODY_PARAM.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
      //进行泛化调用
        final Mono<Object> result = dubboProxyService.genericInvoker(body, metaData, exchange);
        return result.then(chain.execute(exchange));
    }
```

soul-plugin/soul-plugin-alibaba-dubbo/src/main/java/org/dromara/soul/plugin/alibaba/dubbo/AlibabaDubboPlugin.java 和ApacheDubboPlugin的逻辑基本一致, 只是下面部分有一点差别

```java
......
  //进行泛化调用
        Object result = alibabaDubboProxyService.genericInvoker(body, metaData);
//将结果放到 exchange中
        if (Objects.nonNull(result)) {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, result);
        } else {
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, Constants.DUBBO_RPC_RESULT_EMPTY);
        }
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        return chain.execute(exchange);
    }
```



```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
        // issue(https://github.com/dromara/soul/issues/471), add dubbo tag route
        String dubboTagRouteFromHttpHeaders = exchange.getRequest().getHeaders().getFirst(Constants.DUBBO_TAG_ROUTE);
        if (StringUtils.isNotBlank(dubboTagRouteFromHttpHeaders)) {
            RpcContext.getContext().setAttachment(CommonConstants.TAG_KEY, dubboTagRouteFromHttpHeaders);
        }
        ReferenceConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
        if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterface())) {
            ApplicationConfigCache.getInstance().invalidate(metaData.getPath());
            reference = ApplicationConfigCache.getInstance().initRef(metaData);
        }
        GenericService genericService = reference.get();
        Pair<String[], Object[]> pair;
        if (ParamCheckUtils.dubboBodyIsEmpty(body)) {
            pair = new ImmutablePair<>(new String[]{}, new Object[]{});
        } else {
            pair = dubboParamResolveService.buildParameter(body, metaData.getParameterTypes());
        }
        CompletableFuture<Object> future = genericService.$invokeAsync(metaData.getMethodName(), pair.getLeft(), pair.getRight());
        return Mono.fromFuture(future.thenApply(ret -> {
            if (Objects.isNull(ret)) {
                ret = Constants.DUBBO_RPC_RESULT_EMPTY;
            }
            exchange.getAttributes().put(Constants.DUBBO_RPC_RESULT, ret);
            exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
            return ret;
        })).onErrorMap(exception -> exception instanceof GenericException ? new SoulException(((GenericException) exception).getExceptionMessage()) : new SoulException(exception));
    }
}
```



soul-common/src/main/java/org/dromara/soul/common/dto/MetaData.java

```java
@Data
@ToString
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MetaData implements Serializable {
    private String id;
    private String appName;
    private String contextPath;
    private String path;
    private String rpcType;
    private String serviceName;
    private String methodName;
    private String parameterTypes;
    private String rpcExt;
    private Boolean enabled;
}
```

Medata中保存了dubbo接口的信息

* 每一个dubbo接口方法，都会有一条metadata与之对应，可以在 soul-admin --> Metadata中查看

- path: 即http路径
- rpc扩展参数，对应为dubbo接口的一些配置，调整的化，请在这里修改，支持json格式，以下字段：

```json
{"timeout":10000,"group":"",version":"","loadbalance":"","retries":1}
```

在soul-plugin/soul-plugin-apache-dubbo/src/main/java/org/dromara/soul/plugin/apache/dubbo/response/DubboResponsePlugin.java中返回result, alibaba-dubbo中的DubboResponsePlugin.java 同理

```java
@Override
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
    return chain.execute(exchange).then(Mono.defer(() -> {
        final Object result = exchange.getAttribute(Constants.DUBBO_RPC_RESULT);
        if (Objects.isNull(result)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
        return WebFluxResultUtils.result(exchange, success);
    }));
}
```


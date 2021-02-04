---
title: soul_gateway_src_code_learning_15
date: 2021-01-30 20:21:31
tags: SOUL
---

### SOUL sofa plugin 分析

soul-plugin/soul-plugin-global/src/main/java/org/dromara/soul/plugin/global/GlobalPlugin.java  中使用了builder构建soulContext

```java
@Override
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
    final ServerHttpRequest request = exchange.getRequest();
    final HttpHeaders headers = request.getHeaders();
    final String upgrade = headers.getFirst("Upgrade");
    SoulContext soulContext;
    if (StringUtils.isBlank(upgrade) || !"websocket".equals(upgrade)) {
        soulContext = builder.build(exchange);
    } else {
        final MultiValueMap<String, String> queryParams = request.getQueryParams();
        soulContext = transformMap(queryParams);
    }
    exchange.getAttributes().put(Constants.CONTEXT, soulContext);
    return chain.execute(exchange);
}
```

soul-plugin/soul-plugin-global/src/main/java/org/dromara/soul/plugin/global/DefaultSoulContextBuilder.java

```java
@Override
public SoulContext build(final ServerWebExchange exchange) {
    final ServerHttpRequest request = exchange.getRequest();
    String path = request.getURI().getPath();
  //根据url拿到缓存的Metadata,供后面的插件使用
    MetaData metaData = MetaDataCache.getInstance().obtain(path);
    if (Objects.nonNull(metaData) && metaData.getEnabled()) {
        exchange.getAttributes().put(Constants.META_DATA, metaData);
    }
    return transform(request, metaData);
```

soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/param/BodyParamPlugin.java

```java
@Override
    public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
        final ServerHttpRequest request = exchange.getRequest();
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        if (Objects.nonNull(soulContext) && RpcTypeEnum.SOFA.getName().equals(soulContext.getRpcType())) {
            MediaType mediaType = request.getHeaders().getContentType();
            ServerRequest serverRequest = ServerRequest.create(exchange, messageReaders);
          //根据不同的请求类型做不同处理
            if (MediaType.APPLICATION_JSON.isCompatibleWith(mediaType)) {
                return body(exchange, serverRequest, chain);
            }
            if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(mediaType)) {
                return formData(exchange, serverRequest, chain);
            }
            return query(exchange, serverRequest, chain);
        }
        return chain.execute(exchange);
    }

//可以看到oder小于SOFA plugin, 这样就能保证在sofa plugin之前执行
    @Override
    public int getOrder() {
        return PluginEnum.SOFA.getCode() - 1;
    }
private Mono<Void> formData(final ServerWebExchange exchange, final ServerRequest serverRequest, final SoulPluginChain chain) {
//sofa plugin前置处理, 需要的参数放进exchange上下文中
        return serverRequest.formData()
                .switchIfEmpty(Mono.defer(() -> Mono.just(new LinkedMultiValueMap<>())))
                .flatMap(map -> {
                    exchange.getAttributes().put(Constants.SOFA_PARAMS, HttpParamConverter.toMap(() -> map));
                    return chain.execute(exchange);
                });
    }
//sofa plugin前置处理, 需要的参数放进exchange上下文中
    private Mono<Void> query(final ServerWebExchange exchange, final ServerRequest serverRequest, final SoulPluginChain chain) {
        exchange.getAttributes().put(Constants.SOFA_PARAMS,
                HttpParamConverter.ofString(() -> serverRequest.uri().getQuery()));
        return chain.execute(exchange);
    }
```

soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/SofaPlugin.java

```java
@Override
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
    String body = exchange.getAttribute(Constants.SOFA_PARAMS);
    SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
    assert soulContext != null;
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
        Object error = SoulResultWrap.error(SoulResultEnum.SOFA_HAVE_BODY_PARAM.getCode(), SoulResultEnum.SOFA_HAVE_BODY_PARAM.getMsg(), null);
        return WebFluxResultUtils.result(exchange, error);
    }
  //进行sofa泛化调用
    final Mono<Object> result = sofaProxyService.genericInvoker(body, metaData, exchange);
    return result.then(chain.execute(exchange));
}
```

soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/proxy/SofaProxyService.java 

```java
public Mono<Object> genericInvoker(final String body, final MetaData metaData, final ServerWebExchange exchange) throws SoulException {
    ConsumerConfig<GenericService> reference = ApplicationConfigCache.getInstance().get(metaData.getPath());
    if (Objects.isNull(reference) || StringUtils.isEmpty(reference.getInterfaceId())) {
        ApplicationConfigCache.getInstance().invalidate(metaData.getServiceName());
        reference = ApplicationConfigCache.getInstance().initRef(metaData);
    }
    GenericService genericService = reference.refer();
    Pair<String[], Object[]> pair;
    if (null == body || "".equals(body) || "{}".equals(body) || "null".equals(body)) {
        pair = new ImmutablePair<>(new String[]{}, new Object[]{});
    } else {
        pair = sofaParamResolveService.buildParameter(body, metaData.getParameterTypes());
    }
    CompletableFuture<Object> future = new CompletableFuture<>();
    RpcInvokeContext.getContext().setResponseCallback(new SofaResponseCallback<Object>() {
        @Override
        public void onAppResponse(final Object o, final String s, final RequestBase requestBase) {
            future.complete(o);
        }

        @Override
        public void onAppException(final Throwable throwable, final String s, final RequestBase requestBase) {
            future.completeExceptionally(throwable);
        }

        @Override
        public void onSofaException(final SofaRpcException e, final String s, final RequestBase requestBase) {
            future.completeExceptionally(e);
        }
    });
    genericService.$genericInvoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
    return Mono.fromFuture(future.thenApply(ret -> {
        if (Objects.isNull(ret)) {
            ret = Constants.SOFA_RPC_RESULT_EMPTY;
        }
//讲结果放入exchange如, 供后面的SofaResponsePlugin.java 处理
        GenericObject genericObject = (GenericObject) ret;
        exchange.getAttributes().put(Constants.SOFA_RPC_RESULT, genericObject.getFields());
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        return ret;
    })).onErrorMap(SoulException::new);
}
```

soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/response/SofaResponsePlugin.java

```java
@Override
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
    return chain.execute(exchange).then(Mono.defer(() -> {
      //根据不同结果返回请求调用结果
        final Object result = exchange.getAttribute(Constants.SOFA_RPC_RESULT);
        if (Objects.isNull(result)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SERVICE_RESULT_ERROR.getCode(), SoulResultEnum.SERVICE_RESULT_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        Object success = SoulResultWrap.success(SoulResultEnum.SUCCESS.getCode(), SoulResultEnum.SUCCESS.getMsg(), JsonUtils.removeClass(result));
        return WebFluxResultUtils.result(exchange, success);
    }));
}
```


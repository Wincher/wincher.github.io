---
title: soul_gateway_src_code_learning_04
date: 2021-01-19 00:38:50
tags: SOUL
---

### SOUL sofa demo

和上一节的dubbo一样, 使用sofa插件, 访问 http://127.0.0.1:9095/#/system/plugin 可以看到 sofa 插件默认是点击editor 

开启sofa plugin

启动 zk,我用的docker `docker run -p 2181:2181 --restart unless-stopped --name zk -d zookeeper`

IDEA中运行 soul-examples/soul-examples-sofa/src/main/java/org/dromara/soul/examples/sofa/service/TestSofaApplication.java

```shell
2021-01-19 00:48:51.530  INFO 77721 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/insert","pathDesc":"Insert a row of data","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"insert","ruleName":"/sofa/insert","parameterTypes":"org.dromara.soul.examples.dubbo.api.entity.DubboTest","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
2021-01-19 00:48:51.546  INFO 77721 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/findById","pathDesc":"Find by Id","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findById","ruleName":"/sofa/findById","parameterTypes":"java.lang.String","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
2021-01-19 00:48:51.578  INFO 77721 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : sofa client register success: {"appName":"sofa","contextPath":"/sofa","path":"/sofa/findAll","pathDesc":"Get all data","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findAll","ruleName":"/sofa/findAll","parameterTypes":"","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true} 
```

同样可以看到之前启动 soul-examples-http, soul-examples-dubbo 时的类似log, 显然sofa的服务也注册到了soul

访问 http://127.0.0.1:9095/#/plug/sofa 同样可以看到 刚起的实例服务注册了rule 和selector 到了soul上

![img](00register.png)

尝试通过网关访问 sofa 服务

```bash
$curl -i localhost:9195/sofa/findAll
HTTP/1.1 200 OK
content-length: 0
```

只能得到空200响应, 这显然是不正确的

通过追踪 soul-web/src/main/java/org/dromara/soul/web/handler/SoulWebHandler.java 中静态内部类DefaultSoulPluginChain实现了 soul-plugin/soul-plugin-api/src/main/java/org/dromara/soul/plugin/api/SoulPluginChain.java 接口

```java
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
//              由于需要命中的sofa plugin没有加载, 所以最后返回
                return Mono.empty();
            });
        }
```

所以我们要做的就是在soul-bootstrap/pom.xml文件中添加

```xml
<!-- sofa plugin start -->
<dependency>
  <groupId>com.alipay.sofa</groupId>
  <artifactId>sofa-rpc-all</artifactId>
  <version>5.7.6</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-client</artifactId>
  <version>4.0.1</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-framework</artifactId>
  <version>4.0.1</version>
</dependency>
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>4.0.1</version>
</dependency>
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-plugin-sofa</artifactId>
  <version>${project.version}</version>
</dependency>
<!-- sofa plugin end -->
```

重启bootstrap 可以看到sofa plugin被加载了

![img](01sofa_plugin_log.png)

再次尝试请求 

```bash
curl localhost:9195/sofa/findAll
{"code":200,"message":"Access to success!","data":{"name":"hello world Soul Sofa , findAll","id":"1392929023"}}%
```

类似divide,dubbo 使用了 soul-client/soul-client-sofa/src/main/java/org/dromara/soul/client/sofa/SofaServiceBeanPostProcessor.java 去处理 拥有 @SoulSofaClient 注解的接

接下来在 打断点追踪下 soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/SofaPlugin.java#doExecute

```java
protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
  //这里的获取的值值是在 soul-plugin/soul-plugin-sofa/src/main/java/org/dromara/soul/plugin/sofa/param/BodyParamPlugin.java 中处理的
    String body = exchange.getAttribute(Constants.SOFA_PARAMS);
    SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
    assert soulContext != null;
    MetaData metaData = exchange.getAttribute(Constants.META_DATA);
  //这里的metadata具体内容 eg: metaData -> {MetaData@11407} "MetaData(id=null, appName=sofa, contextPath=/sofa, path=/sofa/findAll, rpcType=sofa, serviceName=org.dromara.soul.examples.dubbo.api.service.DubboTestService, methodName=findAll, parameterTypes=null, rpcExt={"loadbalance":"hash","retries":3,"timeout":-1}, enabled=true)"
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
    final Mono<Object> result = sofaProxyService.genericInvoker(body, metaData, exchange);
    return result.then(chain.execute(exchange));
}
```

然后通过sofaProxyService 去获取真实服务返回值

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
    genericService.$invoke(metaData.getMethodName(), pair.getLeft(), pair.getRight());
    return Mono.fromFuture(future.thenApply(ret -> {
        if (Objects.isNull(ret)) {
            ret = Constants.SOFA_RPC_RESULT_EMPTY;
        }
        exchange.getAttributes().put(Constants.SOFA_RPC_RESULT, ret);
        exchange.getAttributes().put(Constants.CLIENT_RESPONSE_RESULT_TYPE, ResultEnum.SUCCESS.getName());
        return ret;
    })).onErrorMap(SoulException::new);
}
```
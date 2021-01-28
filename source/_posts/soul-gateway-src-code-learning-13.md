---
title: soul_gateway_src_code_learning_13
date: 2021-01-28 22:23:58
tags: SOUL
---

### SOUL: SpringCloud plugin & WebClient plugin

接上一章, 直接看 SpringCloud Plugin的处理逻辑

加在SpringCloudplugin需要在soul-bootstrap/pom.xml加入

```xml
<!--soul springCloud plugin start-->
<dependency>
       <groupId>org.dromara</groupId>
       <artifactId>soul-spring-boot-starter-plugin-springcloud</artifactId>
        <version>${last.version}</version>
  </dependency>

  <dependency>
       <groupId>org.dromara</groupId>
       <artifactId>soul-spring-boot-starter-plugin-httpclient</artifactId>
       <version>${last.version}</version>
   </dependency>
   <!--soul springCloud plugin end-->

   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-commons</artifactId>
        <version>2.2.0.RELEASE</version>
   </dependency>
   <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        <version>2.2.0.RELEASE</version>
   </dependency>
   
   <!--使用 eureka 作为 springCloud的注册中心时 -->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
       <version>2.2.0.RELEASE</version>
   </dependency>  

   <!--使用 nacos 作为 springCloud的注册中心 -->
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
       <version>2.1.0.RELEASE</version>
   </dependency>
```

同样在  soul-bootstrap/src/main/resources/application-local.yml 中

```yaml
 # 使用 eureka
  eureka:
     client:
       serviceUrl:
         defaultZone: http://localhost:8761/eureka/
     instance:
       prefer-ip-address: true
  
  # 使用 nacos   
  spring:
     cloud:
       nacos:
         discovery:
            server-addr: 127.0.0.1:8848
```

看一下SpringCloud plugin的核心逻辑

```java
@Override
 //同样是处理ServerWebExchange
    protected Mono<Void> doExecute(final ServerWebExchange exchange, final SoulPluginChain chain, final SelectorData selector, final RuleData rule) {
        if (Objects.isNull(rule)) {
            return Mono.empty();
        }
        final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
        assert soulContext != null;
        //获取具体的SpringCLoudHandler
        final SpringCloudRuleHandle ruleHandle = GsonUtils.getInstance().fromJson(rule.getHandle(), SpringCloudRuleHandle.class);
        final SpringCloudSelectorHandle selectorHandle = GsonUtils.getInstance().fromJson(selector.getHandle(), SpringCloudSelectorHandle.class);
        if (StringUtils.isBlank(selectorHandle.getServiceId()) || StringUtils.isBlank(ruleHandle.getPath())) {
            Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getCode(), SoulResultEnum.CANNOT_CONFIG_SPRINGCLOUD_SERVICEID.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
//通过LoadBalancerClient获取具体的ILoadBalancer实例
        final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
        final URI uri = loadBalancer.reconstructURI(serviceInstance, URI.create(soulContext.getRealUrl()));
//拼装出真实目标地址url
        String realURL = buildRealURL(uri.toASCIIString(), soulContext.getHttpMethod(), exchange.getRequest().getURI().getQuery());
// set 真实地址到 exchange中
        exchange.getAttributes().put(Constants.HTTP_URL, realURL);
        //set time out.
        exchange.getAttributes().put(Constants.HTTP_TIME_OUT, ruleHandle.getTimeout());
        return chain.execute(exchange);
    }
```

而后 WebClientPlugin会使用刚SpringCloud plugin 处理过后 并set 到SoulContext的真实地址去访问后台服务

```java
public Mono<Void> execute(final ServerWebExchange exchange, final SoulPluginChain chain) {
      final SoulContext soulContext = exchange.getAttribute(Constants.CONTEXT);
      assert soulContext != null;
      String urlPath = exchange.getAttribute(Constants.HTTP_URL);
      if (StringUtils.isEmpty(urlPath)) {
          Object error = SoulResultWrap.error(SoulResultEnum.CANNOT_FIND_URL.getCode(), SoulResultEnum.CANNOT_FIND_URL.getMsg(), null);
          return WebFluxResultUtils.result(exchange, error);
      }
      long timeout = (long) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_TIME_OUT)).orElse(3000L);
      int retryTimes = (int) Optional.ofNullable(exchange.getAttribute(Constants.HTTP_RETRY)).orElse(0);
      log.info("The request urlPath is {}, retryTimes is {}", urlPath, retryTimes);
      HttpMethod method = HttpMethod.valueOf(exchange.getRequest().getMethodValue());
      //WebClient是从Spring WebFlux 5.0版本开始提供基于响应式编程模型的非阻塞的HttpClient
      //拼装请求
      WebClient.RequestBodySpec requestBodySpec = webClient.method(method).uri(urlPath);
      return handleRequestBody(requestBodySpec, exchange, timeout, retryTimes, chain);
  }
private Mono<Void> handleRequestBody(final WebClient.RequestBodySpec requestBodySpec,
                                       final ServerWebExchange exchange,
                                       final long timeout,
                                       final int retryTimes,
                                       final SoulPluginChain chain) {
      return requestBodySpec.headers(httpHeaders -> {
          httpHeaders.addAll(exchange.getRequest().getHeaders());
          httpHeaders.remove(HttpHeaders.HOST);
      })
              .contentType(buildMediaType(exchange))
              .body(BodyInserters.fromDataBuffers(exchange.getRequest().getBody()))
              .exchange()
              .doOnError(e -> log.error(e.getMessage()))
              .timeout(Duration.ofMillis(timeout))
              .retryWhen(Retry.onlyIf(x -> x.exception() instanceof ConnectTimeoutException)
                  .retryMax(retryTimes)
                  .backoff(Backoff.exponential(Duration.ofMillis(200), Duration.ofSeconds(20), 2, true)))
              .flatMap(e -> doNext(e, exchange, chain));
```

至此, 请求被代理到了真实访问的服务.
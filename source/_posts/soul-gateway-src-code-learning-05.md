---
title: soul_gateway_src_code_learning_05
date: 2021-01-20 00:03:36
tags: SOUL
---

### SOUL Spring Cloud demo

拉了下代码,使用了当前使用 249fb1ff8711e7b531f56f89bae9650d7fb4def6 版本

有了上一节的经验

打开bootstrap pom中的 springcloud 和 eureka 依赖

```xml
<!--soul springCloud plugin start-->
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-plugin-springcloud</artifactId>
  <version>${project.version}</version>
</dependency>

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

<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  <version>2.2.0.RELEASE</version>
</dependency>
```

同时打开soul-bootstrap/src/main/resources/application-local.yml, 去掉eureka配置部分的注释 

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

重启bootstrap log中可以看到springcloud plugin被加载了

```bash
2021-01-20 00:20:53.274  INFO 90869 --- [           main] o.d.s.w.configuration.SoulConfiguration  : load plugin:[springCloud] [org.dromara.soul.plugin.springcloud.SpringCloudPlugin]
```

soul-examples-springcloud使用eureka作为服务发现, 我们先启动 soul-examples/soul-examples-eureka/src/main/java/org/dromara/soul/examples/eureka/EurekaServerApplication.java

和上一节的一样, 使用Spring Cloud插件, 访问 http://127.0.0.1:9095/#/system/plugin 可以看到 springCloud plugin 是关闭的, 点击editor 

开启springCloud plugin

同样可以看到之前启动 soul-examples-http, soul-examples-dubbo 时的类似log, 显然spring cloud的服务也注册到了soul

```bash
2021-01-20 00:23:16.972  INFO 90924 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : springCloud client register success: {"appName":"springCloud-test","context":"/springcloud","path":"/springcloud/order/save","pathDesc":"","rpcType":"springCloud","ruleName":"/springcloud/order/save","enabled":true} 
2021-01-20 00:23:16.987  INFO 90924 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : springCloud client register success: {"appName":"springCloud-test","context":"/springcloud","path":"/springcloud/order/findById","pathDesc":"","rpcType":"springCloud","ruleName":"/springcloud/order/findById","enabled":true} 
2021-01-20 00:23:17.003  INFO 90924 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : springCloud client register success: {"appName":"springCloud-test","context":"/springcloud","path":"/springcloud/order/path/**","pathDesc":"","rpcType":"springCloud","ruleName":"/springcloud/order/path/**","enabled":true} 
2021-01-20 00:23:17.017  INFO 90924 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : springCloud client register success: {"appName":"springCloud-test","context":"/springcloud","path":"/springcloud/order/path/**/name","pathDesc":"","rpcType":"springCloud","ruleName":"/springcloud/order/path/**/name","enabled":true} 
```



访问 http://127.0.0.1:9095/#/plug/springCloud 同样可以看到 刚起的实例服务注册了rule 和selector 到了soul上

![img](00springcloud_plugin_log.png)

 直接访问 spring cloud 服务

```bash
$curl http://localhost:8884/order/findById\?id\=3
{"id":"3","name":"hello world spring cloud findById"}%
```

正常

尝试通过网关访问 spring cloud 服务

```bash
$curl http://localhost:9195/springcloud/order/findById\?id=3
{"id":"3","name":"hello world spring cloud findById"}%
```

转发请求的核心逻辑在 soul-plugin/soul-plugin-springcloud/src/main/java/org/dromara/soul/plugin/springcloud/SpringCloudPlugin.java#doExecute

```java
// 这里通过封装的serviceId 通过 loadBalancer 来找到服务的实例
final ServiceInstance serviceInstance = loadBalancer.choose(selectorHandle.getServiceId());
//serviceInstance的值得如下 RibbonServer{serviceId='springCloud-test', server=ip:8884, secure=false, metadata={management.port=8884}},后面就根据真实地址去请求了
        if (Objects.isNull(serviceInstance)) {
            Object error = SoulResultWrap.error(SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getCode(), SoulResultEnum.SPRINGCLOUD_SERVICEID_IS_ERROR.getMsg(), null);
            return WebFluxResultUtils.result(exchange, error);
        }
```


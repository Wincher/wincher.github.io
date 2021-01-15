---
title: soul_gateway_src_code_learning_02
date: 2021-01-16 00:15:01
tags: SOUL
---

### SOUL divide plugin

上篇我们使用divide插件配置selector和rule成功将访问网关的请求转发到了baidu.com

今天对divide plugin进一步的使用

我们启动SOUL项目准备好的示例 soul-examples/soul-examples-http/src/main/java/org/dromara/soul/examples/http/SoulTestHttpApplication.java

稍微留意一下log, 字面意思就是将接口注册到了网关上

```bash
o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/test/**","pathDesc":"","rpcType":"http","host":"127.0.0.1","port":8188,"ruleName":"/http/test/**","enabled":true,"registerMetaData":false} 
2021-01-16 00:22:49.555  INFO 7520 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/save","pathDesc":"Save order","rpcType":"http","host":"127.0.0.1","port":8188,"ruleName":"/http/order/save","enabled":true,"registerMetaData":false} 
2021-01-16 00:22:49.569  INFO 7520 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/path/**","pathDesc":"","rpcType":"http","host":"127.0.0.1","port":8188,"ruleName":"/http/order/path/**","enabled":true,"registerMetaData":false} 
2021-01-16 00:22:49.585  INFO 7520 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/path/**/name","pathDesc":"","rpcType":"http","host":"127.0.0.1","port":8188,"ruleName":"/http/order/path/**/name","enabled":true,"registerMetaData":false} 
2021-01-16 00:22:49.601  INFO 7520 --- [pool-1-thread-1] o.d.s.client.common.utils.RegisterUtils  : http client register success: {"appName":"http","context":"/http","path":"/http/order/findById","pathDesc":"Find by id","rpcType":"http","host":"127.0.0.1","port":8188,"ruleName":"/http/order/findById","enabled":true,"registerMetaData":false} 
```

启动后打开 http://127.0.0.1:9095/#/plug/divide admin divide plugin页面

![pic](00soul_auto_register_divide.png)

我们发现在我们刚刚启动的示例项目中定义的接口都已经注册到了divide plugin上

```bash
# 访问example项目获得结果
$curl http://localhost:8188/test/findByUserId\?userId\=132
{"userId":"132","userName":"hello world"}%
由于被gateway divide plugin 配置了代理, 访问gateway http path也代理到了刚启动的example下
$curl http://localhost:9195/http/test/findByUserId\?userId\=132
{"userId":"132","userName":"hello world"}%
```

自动注册还是很方便控制接口访问的,今天就看看是如何实现的

```xml
<dependency>
  <groupId>org.dromara</groupId>
  <artifactId>soul-spring-boot-starter-client-springmvc</artifactId>
  <version>${soul.version}</version>
</dependency>
```

 该项目引入了 soul-spring-boot-starter-client-springmvc

```xml
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>soul-client-springmvc</artifactId>
    <version>${project.version}</version>
</dependency>
```

soul-examples-http 项目中application.yml

```yaml
soul:
  http:
    adminUrl: http://localhost:9095
    port: 8188
    contextPath: /http
    appName: http
    full: false
```

配置信息用来注册到网关, 配置定义在 soul-client/soul-client-http/soul-client-springmvc/src/main/java/org/dromara/soul/client/springmvc/config/SoulSpringMvcConfig.java 中

请求的发起来自 soul-client/soul-client-http/soul-client-springmvc/src/main/java/org/dromara/soul/client/springmvc/init/SpringMvcClientBeanPostProcessor.java

```java
@Override
    public Object postProcessAfterInitialization(@NonNull final Object bean, @NonNull final String beanName) throws BeansException {
        if (soulSpringMvcConfig.isFull()) {
            return bean;
        }
        Controller controller = AnnotationUtils.findAnnotation(bean.getClass(), Controller.class);
        RestController restController = AnnotationUtils.findAnnotation(bean.getClass(), RestController.class);
        RequestMapping requestMapping = AnnotationUtils.findAnnotation(bean.getClass(), RequestMapping.class);
        if (controller != null || restController != null || requestMapping != null) {
            SoulSpringMvcClient clazzAnnotation = AnnotationUtils.findAnnotation(bean.getClass(), SoulSpringMvcClient.class);
            String prePath = "";
            if (Objects.nonNull(clazzAnnotation)) {
                if (clazzAnnotation.path().indexOf("*") > 1) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(clazzAnnotation, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                    return bean;
                }
                prePath = clazzAnnotation.path();
            }
            final Method[] methods = ReflectionUtils.getUniqueDeclaredMethods(bean.getClass());
            for (Method method : methods) {
                SoulSpringMvcClient soulSpringMvcClient = AnnotationUtils.findAnnotation(method, SoulSpringMvcClient.class);
                if (Objects.nonNull(soulSpringMvcClient)) {
                    String finalPrePath = prePath;
                    executorService.execute(() -> RegisterUtils.doRegister(buildJsonParams(soulSpringMvcClient, finalPrePath), url,
                            RpcTypeEnum.HTTP));
                }
            }
        }
        return bean;
    }
```

这段逻辑判断当我们处理一个接口的时候 ,如果配置了@SoulSpringMvcClient 注解, 那么就会 调用 RegisterUtils.doRegister 将接口信息注册到admin

```java
public static void doRegister(final String json, final String url, final RpcTypeEnum rpcTypeEnum) {
        try {
            String result = OkHttpTools.getInstance().post(url, json);
            if (AdminConstants.SUCCESS.equals(result)) {
                log.info("{} client register success: {} ", rpcTypeEnum.getName(), json);
            } else {
                log.error("{} client register error: {} ", rpcTypeEnum.getName(), json);
            }
        } catch (IOException e) {
            log.error("cannot register soul admin param, url: {}, request body: {}", url, json, e);
        }
    }
```

我们看到这个log是否很熟悉呢,和最开始soul-examples-http 启动 时的log对应上了

至此我们了解了SOUL admin的一个优势就是低侵入自动注册接口到gateway
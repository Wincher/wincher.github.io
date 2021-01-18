---
title: soul_gateway_src_code_learning_03
date: 2021-01-17 02:26:10
tags: SOUL
---

### SOUL dubbo demo

使用dubbo插件, 访问 http://127.0.0.1:9095/#/system/plugin 可以看到 dubbo 插件默认是

![img](01turn_on.jpg)

点击editor 

![img](00dubbo_plugin.png)

开启dubbo plugin

启动 zk,我用的docker `docker run -p 2181:2181 --restart unless-stopped --name zk -d zookeeper`

IDEA中运行 soul-examples/soul-examples-dubbo/soul-examples-apache-dubbo-service/src/main/java/org/dromara/soul/examples/apache/dubbo/service/TestApacheDubboApplication.java

```shell
2021-01-17 02:55:52.742  INFO 38578 --- [pool-2-thread-1] o.d.s.client.common.utils.RegisterUtils  : dubbo client register success: {"appName":"dubbo","contextPath":"/dubbo","path":"/dubbo/findAll","pathDesc":"Get all data","rpcType":"dubbo","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findAll","ruleName":"/dubbo/findAll","parameterTypes":"","rpcExt":"{\"group\":\"\",\"version\":\"\",\"loadbalance\":\"random\",\"retries\":2,\"timeout\":10000,\"url\":\"\"}","enabled":true} 
```

同样可以看到之前启动 soul-examples-http时的类似log, 显然dubbo的服务也注册到了soul

访问 http://127.0.0.1:9095/#/plug/dubbo 同样可以看到 刚起的实例服务注册了rule 和selector 到了soul上

尝试通过网关访问dubbo服务, 果然成功

```bash
curl  localhost:9195/dubbo/findAll
{"code":200,"message":"Access to success!","data":{"name":"hello world Soul Apache, findAll","id":"-1845873449"}}%
```

dubbo注册到soul同http一样需要以下配置

```yaml
soul:
  dubbo:
    adminUrl: http://localhost:9095
    contextPath: /dubbo
    appName: dubbo
```

流程与divide完全一致

同divide, 使用了 AlibabaDubboServiceBeanPostProcessor 去处理 拥有 @SoulDubboClient 注解的接口



通过SoulWebHandler#execute遍历注插件后执行 AbstractSoulPlugin具体实现类的execute方法, 这里留个todo:接下来详细分析
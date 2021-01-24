---
title: soul_gateway_src_code_learning_01
date: 2021-01-15 00:07:03
tags: SOUL
---


## Soul

这是一个异步的,高性能的,跨语言的,响应式的API网关。我希望能够有一样东西像灵魂一样，保护您的微服务。参考了Kong，Spring-Cloud-Gateway等优秀的网关后，站在巨人的肩膀上，Soul由此诞生！

## Features

- 支持各种语言(http协议)，支持 dubbo，springcloud协议。
- 插件化设计思想，插件热插拔,易扩展。
- 灵活的流量筛选，能满足各种流量控制。
- 内置丰富的插件支持，鉴权，限流，熔断，防火墙等等。
- 流量配置动态化，性能极高，网关消耗在 1~2ms。
- 支持集群部署，支持 A/B Test, 蓝绿发布。

[官方文档](https://dromara.org/zh-cn/docs/soul/soul.html)

[Github repo](https://github.com/dromara/soul)

本人使用的是 master分支 b1ed3daa87a1633cc6c5cc131baa0cdd78df643c 版本, 未来可能会有略微不同

``` shell
# clone repo 到本地
git clone https://github.com/dromara/soul
# 然后进行编译项目:
mvn clean package install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true

# 编译中会遇到有些包找不到的问题, 主动地去编译相应的模块就好, 重复此步骤
# eg:
mvn clean package install -Dmaven.test.skip=true -Dmaven.javadoc.skip=true -Drat.skip=true -Dcheckstyle.skip=true -f soul-plugin/soul-plugin-monitor/pom.xml

```

编辑 soul-admin的配置文件 soul-admin/src/main/resources/application.yml

```yaml
# 去掉h2配置文件的注释
spring:
  profiles:
    active: h2
# 或者配置mysql数据库
datasource:
    url: jdbc:mysql://localhost:3306/soul?useUnicode=true&characterEncoding=utf-8
    username: root
    password:
    driver-class-name: com.mysql.jdbc.Driver
```

在IDEA中运行 soul-admin 的入口函数 soul-admin/src/main/java/org/dromara/soul/admin/SoulAdminBootstrap.java

访问 <http://127.0.0.1:9095> 我们就看到首页了

![pic](00login.png)

默认的 账号: **admin** 密码:**123456**

![pic](01index.png)

随后在IDEA中运行 soul-bootstrap 的入口函数 soul-bootstrap/src/main/java/org/dromara/soul/bootstrap/SoulBootstrapApplication.java

![pic](02start_bootstrap_log.jpg)

注意这个log是soul-admin产生的,也就是soul-bootstrap启动后与admin建立了连接

这个时候访问 <http://localhost:9195/> 也就是soul-bootstrap会返回

```json
{
 code: -107,
 message: "Can not find selector, please check your configuration!",
 data: null
}
```

bootstrap产生log

```bash
2021-01-15 00:20:00.542 ERROR 93827 --- [-work-threads-2] o.d.soul.plugin.base.utils.CheckUtils    : can not match selector data: divide
```

根据提示我们打开divide页面, <http://localhost:9095/#/plug/divide>

添加selector

![pic](03selector.png)

![pic](030_opt.jpg)

> 添加selector中我发现 protocol 和 ip:port 会拼接成我们的代理目标地址, 也就是说https**://** 这部分也是要手动写的,这是可以优化的点, 做成可选项的 schema, ip, 和port可选缺省 80, 这样可以更规范配置,另外如上图所示这样的写法也是可以正常被代理到 <http://baidu.com> 的

添加rule

![pic](04rule.png)

```bash
$curl -i localhost:9195
HTTP/1.1 302 Found
Content-Length: 161
Connection: keep-alive
Content-Type: text/html
Date: Thu, 14 Jan 2021 16:29:38 GMT
Keep-Alive: timeout=4
Location: http://www.baidu.com/
Proxy-Connection: keep-alive
Server: bfe/1.0.8.18

<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>bfe/1.0.8.18</center>
</body>
</html>
```

可以看到我们访问刚配置好的根路径 / path 已经得到重定向返回结果, 浏览器会重定向到https的百度

至此, 最基本的soul gateway已经跑起来了

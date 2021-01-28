---
title: soul_gateway_src_code_learning_10
date: 2021-01-26 00:31:42
tags: SOUL
---

SOUL admin/bootstrap cluster

模拟soul 网关集群部署, 启动2个admin, 2个bootstrap

![img](00parallel_run.jpg)

如上开启配置可以启动多个实例,也可以打包后通过java -jar启动

在`soul-bootstrap`的`application.yml`配置刚才启动的`soul-admin`的 url

```yaml
soul:
    sync:
        websocket :
             urls: ws://localhost:9095/websocket,ws://localhost:9096/websocket
```

启动后分别打开 <http://localhost:9095/#/plug/divide>, <http://localhost:9096/#/plug/divide>,可以看到目前都没有数据

![img](01no_data_admins.png)

启动一个soul-examples-http, 具体可参考前面的文章

```bash
$curl 127.0.0.1:9195/http/order/findById\?id=1
{"id":"1","name":"hello world findById"}%
$curl 127.0.0.1:9196/http/order/findById\?id=1
{"id":"1","name":"hello world findById"}%
```

这时候, 我们看到9195, 9196都可以访问代理到,实际上我们只是因为刚启动的配置注册到了admin:9196, 而后被同步到了配置了两个admin的websocket同步的9195和9196 bootstrap, 那么我好奇的是如果这个时候在启动一个example, 修改接口路径, 同步到9096,那么我们bootstrap得到的是合集还是会被覆盖呢, 接下来操作

修改 soul-examples/soul-examples-http/src/main/resources/application.yml

```yaml
server:
  port: 8189
  address: 0.0.0.0
soul:
  http:
    adminUrl: http://localhost:9096
```

soul-examples/soul-examples-http/src/main/java/org/dromara/soul/examples/http/controller/HttpTestController.java

将路径改为

```java
@GetMapping("/findById")
@SoulSpringMvcClient(path = "/9016FindById", desc = "Find by id")
public OrderDTO findById(@RequestParam("id") final String id) {}
```

启动后做下

![img](02diff_admin_selector.jpg)

可以看到 8188 和 8189 examples-http分别注册到了 9095和9096admin, 接下来测试下

```bash
$url 127.0.0.1:9196/http/order/findById\?id=1
{"id":"1","name":"hello world findById"}%
$curl 127.0.0.1:9195/http/order/findById\?id=1
{"id":"1","name":"hello world findById"}%
$curl 127.0.0.1:9195/http/order/9016FindById\?id=1
{"code":-102,"message":"Rule not found!","data":null}%
$curl 127.0.0.1:9196/http/order/9016FindById\?id=1
{"code":-102,"message":"Rule not found!","data":null}%
```

可以看到, 我们新启动的8189 examples-http并没有对任意一个bootstrap生效, 可以猜测只有第一个配置的url生效了

而实际上我们看到 soul-sync-data-center/soul-sync-data-websocket/src/main/java/org/dromara/soul/plugin/sync/data/websocket/WebsocketSyncDataService.java

```java
for (String url : urls) {
            try {
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
            for (WebSocketClient client : clients) {
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
                executor.scheduleAtFixedRate(() -> {
                    try {
                        if (client.isClosed()) {
                            boolean reconnectSuccess = client.reconnectBlocking();
                            if (reconnectSuccess) {
                                log.info("websocket reconnect is successful.....");
                            } else {
                                log.error("websocket reconnection is error.....");
                            }
                        }
                    } catch (InterruptedException e) {
                        log.error("websocket connect is error :{}", e.getMessage());
                    }
                }, 10, 30, TimeUnit.SECONDS);
            }
```

明显所有的client都会同步数据,  这里留下一个TODO:看看为什么会产生这种现象




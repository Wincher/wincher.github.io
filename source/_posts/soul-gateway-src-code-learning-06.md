---
title: soul_gateway_src_code_learning_06
date: 2021-01-20 23:57:43
tags: SOUL
---

## SOUL Admin & 网关 Websocket 数据同步

启动后 admin 和 bootstrap 后, admin log中可以看到

```java
2021-01-21 00:09:46.004  INFO 11893 --- [0.0-9095-exec-8] o.d.s.a.l.websocket.WebsocketCollector   : websocket on open successful....
```

根据log找到 soul-admin/src/main/java/org/dromara/soul/admin/listener/websocket/WebsocketCollector.java ,

```java
@ServerEndpoint("/websocket")
public class WebsocketCollector {
......
```

可见admin是一个server端, 那么调用方就在bootstrap中了, bootstrap中有这样一条log

```java
2021-01-20 23:16:02.285  INFO 21427 --- [           main] b.s.s.d.w.WebsocketSyncDataConfiguration : you use websocket sync soul data.......
```

找到 soul-spring-boot-starter/soul-spring-boot-starter-sync-data-center/soul-spring-boot-starter-sync-data-websocket/src/main/java/org/dromara/soul/spring/boot/starter/sync/data/websocket/WebsocketSyncDataConfiguration.java

```java
@Bean
    public SyncDataService websocketSyncDataService(final ObjectProvider<WebsocketConfig> websocketConfig, final ObjectProvider<PluginDataSubscriber> pluginSubscriber,
                                           final ObjectProvider<List<MetaDataSubscriber>> metaSubscribers, final ObjectProvider<List<AuthDataSubscriber>> authSubscribers) {
        log.info("you use websocket sync soul data.......");
        return new WebsocketSyncDataService(websocketConfig.getIfAvailable(WebsocketConfig::new), pluginSubscriber.getIfAvailable(),
                metaSubscribers.getIfAvailable(Collections::emptyList), authSubscribers.getIfAvailable(Collections::emptyList));
    }
```

  其中 websocketCondfig 来自

```java
@Bean
    @ConfigurationProperties(prefix = "soul.sync.websocket")
    public WebsocketConfig websocketConfig() {
        return new WebsocketConfig();
    }
```

properties 来自如下配置:

soul-bootstrap/src/main/resources/application-local.yml 

```yaml
soul :
    sync:
        websocket :
             urls: ws://localhost:9095/websocket
```

WebsocketSyncDataService 构造方法

```java
    public WebsocketSyncDataService(final WebsocketConfig websocketConfig,
                                    final PluginDataSubscriber pluginDataSubscriber,
                                    final List<MetaDataSubscriber> metaDataSubscribers,
                                    final List<AuthDataSubscriber> authDataSubscribers) {
      //可配置多个admin地址
        String[] urls = StringUtils.split(websocketConfig.getUrls(), ",");
      //构建线程池
        executor = new ScheduledThreadPoolExecutor(urls.length, SoulThreadFactory.create("websocket-connect", true));
        for (String url : urls) {
            try {
              //这个clients 是定义的 private final List<WebSocketClient> clients = new ArrayList<>();
              //下面会分析SoulWebSocketClient
                clients.add(new SoulWebsocketClient(new URI(url), Objects.requireNonNull(pluginDataSubscriber), metaDataSubscribers, authDataSubscribers));
            } catch (URISyntaxException e) {
                log.error("websocket url({}) is error", url, e);
            }
        }
        try {
          //初始化对所有的WebSocketClient建立连接
            for (WebSocketClient client : clients) {
                boolean success = client.connectBlocking(3000, TimeUnit.MILLISECONDS);
                if (success) {
                    log.info("websocket connection is successful.....");
                } else {
                    log.error("websocket connection is error.....");
                }
              //每30秒去重连断开连接的WebSocket连接
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
            /* client.setProxy(new Proxy(Proxy.Type.HTTP, new InetSocketAddress("proxyaddress", 80)));*/
        } catch (InterruptedException e) {
            log.info("websocket connection...exception....", e);
        }

    }
```

这样和Admin之间的通信就建立了, 那接下来就是数据的同步了, 上面代码中 我们使用了自己封装的 SoulWebSocketClient

```java
//这个标记了此client是否已经发送了握手请求到admin, 详情看下面onOpen方法
private volatile boolean alreadySync = Boolean.FALSE;

public SoulWebsocketClient(final URI serverUri, final PluginDataSubscriber pluginDataSubscriber,
                           final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {
    super(serverUri);
  //初始化的时候我们传入了websocketDataHandler, 顾名思义是要处理通信的数据的
    this.websocketDataHandler = new WebsocketDataHandler(pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
}

@Override
    public void onOpen(final ServerHandshake serverHandshake) {
      //client第一次onOpen, 建立连接后会发送DataEventTypeEnum.MYSELF.name()消息到admin, 这里有个印象, 后面会讲道它的用途
        if (!alreadySync) {
            send(DataEventTypeEnum.MYSELF.name());
            alreadySync = true;
        }
    }

@Override
public void onMessage(final String result) {
  handleResult(result);
}


@SuppressWarnings("ALL")
private void handleResult(final String result) {
  WebsocketData websocketData = GsonUtils.getInstance().fromJson(result, WebsocketData.class);
  ConfigGroupEnum groupEnum = ConfigGroupEnum.acquireByName(websocketData.getGroupType());
  String eventType = websocketData.getEventType();
  String json = GsonUtils.getInstance().toJson(websocketData.getData());
  //果不其然, onMessage使用了 websocketDataHandler 来处理数据
  websocketDataHandler.executor(groupEnum, json, eventType);
}
```



```java
public WebsocketDataHandler(final PluginDataSubscriber pluginDataSubscriber,
                            final List<MetaDataSubscriber> metaDataSubscribers,
                            final List<AuthDataSubscriber> authDataSubscribers) {
    ENUM_MAP.put(ConfigGroupEnum.PLUGIN, new PluginDataHandler(pluginDataSubscriber));
    ENUM_MAP.put(ConfigGroupEnum.SELECTOR, new SelectorDataHandler(pluginDataSubscriber));
    ENUM_MAP.put(ConfigGroupEnum.RULE, new RuleDataHandler(pluginDataSubscriber));
    ENUM_MAP.put(ConfigGroupEnum.APP_AUTH, new AuthDataHandler(authDataSubscribers));
    ENUM_MAP.put(ConfigGroupEnum.META_DATA, new MetaDataHandler(metaDataSubscribers));
}

public void executor(final ConfigGroupEnum type, final String json, final String eventType) {
  //WebsocketDataHandler初始化的时候定义了几种Config类型为key, 讲handler作为value存入map中
  //使用对应的handler处理json
        ENUM_MAP.get(type).handle(json, eventType);
}
```

上面的DataHandler都继承自 soul-sync-data-center/soul-sync-data-websocket/src/main/java/org/dromara/soul/plugin/sync/data/websocket/handler/AbstractDataHandler.java,  会根据不同的eventType来对数据进行不同的操作

```java
@Override
    public void handle(final String json, final String eventType) {
        List<T> dataList = convert(json);
        if (CollectionUtils.isNotEmpty(dataList)) {
            DataEventTypeEnum eventTypeEnum = DataEventTypeEnum.acquireByName(eventType);
            switch (eventTypeEnum) {
                case REFRESH:
                case MYSELF:
                    doRefresh(dataList);
                    break;
                case UPDATE:
                case CREATE:
                    doUpdate(dataList);
                    break;
                case DELETE:
                    doDelete(dataList);
                    break;
                default:
                    break;
            }
        }
```

然后回到最开始的 soul-admin/src/main/java/org/dromara/soul/admin/listener/websocket/WebsocketCollector.java 

```java
@OnMessage
    public void onMessage(final String message, final Session session) {
      //admin如果收到DataEventTypeEnum.MYSELF.name(), 还记得上面bootstrap中每个client只有第一次onOpen才会发送的消息吗
        if (message.equals(DataEventTypeEnum.MYSELF.name())) {
            try {
                ThreadLocalUtil.put(SESSION_KEY, session);
                SpringBeanUtils.getInstance().getBean(SyncDataService.class).syncAll(DataEventTypeEnum.MYSELF);
            } finally {
                ThreadLocalUtil.clear();
            }
        }
    }
```

TODO:  这块MYSELF的含义具体分析下


---
title: soul_gateway_src_code_learning_07
date: 2021-01-22 01:03:03
tags:
---

## SOUL Admin & 网关 Zookeeper 数据同步

![img](00sync_starters.png)

目前SOUL提供了这几种数据同步方式, 上一节我们使用了WebSocket, 这一节使用Zookeerper来做数据同步, 代码实现结构与WebSocket如出一辙, 只不过具体的同步过程使用了Zookeeper自身的特性

启动 zk,我用的docker `docker run -p 2181:2181 --restart unless-stopped --name zk -d zookeeper`

保证 soul-bootstrap pom.xml 引入

```xml
        <dependency>
            <groupId>org.dromara</groupId>
            <artifactId>soul-spring-boot-starter-sync-data-zookeeper</artifactId>
            <version>${project.version}</version>
        </dependency>
```

soul-bootstrap/src/main/resources/application-local.yml 打开zk的配置,注释掉websocket(不注释掉的话就两种同步方式都开启,没有必要)

```yaml
soul:
    sync:
#        websocket :
#             urls: ws://localhost:9095/websocket

        zookeeper:
             url: localhost:2181
             sessionTimeout: 5000
             connectionTimeout: 2000
```

启动 bootstrap

```bash
2021-01-22 01:22:42.073  INFO 48117 --- [           main] org.I0Itec.zkclient.ZkClient             : Waiting for keeper state SyncConnected
2021-01-22 01:22:42.077  INFO 48117 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Opening socket connection to server localhost/127.0.0.1:2181. Will not attempt to authenticate using SASL (unknown error)
2021-01-22 01:22:42.093  INFO 48117 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Socket connection established, initiating session, client: /127.0.0.1:61106, server: localhost/127.0.0.1:2181
2021-01-22 01:22:42.105  INFO 48117 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Session establishment complete on server localhost/127.0.0.1:2181, sessionid = 0x100000074b10006, negotiated timeout = 5000
2021-01-22 01:22:42.107  INFO 48117 --- [ain-EventThread] org.I0Itec.zkclient.ZkClient             : zookeeper state changed (SyncConnected)
```

查看zk, 我用是 zkCli, 可以看到在 / 路径下有了soul

```
[zk: localhost:2181(CONNECTED) 2] ls /soul
[auth, metaData, plugin]
ls /soul/auth
[]
[zk: localhost:2181(CONNECTED) 4] ls /soul/metaData
[]
[zk: localhost:2181(CONNECTED) 5] ls /soul/plugin
[]
```

由于admin还没启动 目前auth, metaData, plugin都是空的

同样修改soul-admin/src/main/resources/application.yml 打开zookeeper配置

```yaml
  sync:
#    websocket:
#      enabled: true
    zookeeper:
        url: localhost:2181
        sessionTimeout: 5000
        connectionTimeout: 2000
```

启动 soul-admin/src/main/java/org/dromara/soul/admin/SoulAdminBootstrap.java

```bash
2021-01-22 01:30:59.421  INFO 48224 --- [           main] org.I0Itec.zkclient.ZkClient             : Waiting for keeper state SyncConnected
2021-01-22 01:30:59.426  INFO 48224 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181. Will not attempt to authenticate using SASL (unknown error)
2021-01-22 01:30:59.443  INFO 48224 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Socket connection established, initiating session, client: /0:0:0:0:0:0:0:1:61453, server: localhost/0:0:0:0:0:0:0:1:2181
2021-01-22 01:30:59.453  INFO 48224 --- [localhost:2181)] org.apache.zookeeper.ClientCnxn          : Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, sessionid = 0x100000074b10007, negotiated timeout = 5000
2021-01-22 01:30:59.454  INFO 48224 --- [ain-EventThread] org.I0Itec.zkclient.ZkClient             : zookeeper state changed (SyncConnected)
```

看log像是已经建立连接, 但并未找到同步数据的log, 查看zk, 

```
[zk: localhost:2181(CONNECTED) 2] ls /soul
[auth, metaData, plugin]
ls /soul/auth
[]
[zk: localhost:2181(CONNECTED) 4] ls /soul/metaData
[]
[zk: localhost:2181(CONNECTED) 5] ls /soul/plugin
[]
```

的确没有同步, 我们先分析一下zk同步的代码逻辑

上一节分析 WebSocket数据同步的时候, WebsocketSyncDataService实现了, WebSocketsoul-sync-data-center/soul-sync-data-api/src/main/java/org/dromara/soul/sync/data/api/SyncDataService.java, 这个接口就是为多种数据同步方式定义的标准, 同样可以找到soul-sync-data-center/soul-sync-data-zookeeper/src/main/java/org/dromara/soul/sync/data/zookeeper/ZookeeperSyncDataService.java

```java
				//构造方法中就会使用zookeeper的监听三组数据
				watcherData();
        watchAppAuth();
        watchMetaData();
       //都是相同逻辑,只看一个就好 
        private void watcherData() {
        final String pluginParent = ZkPathConstants.PLUGIN_PARENT;
        List<String> pluginZKs = zkClientGetChildren(pluginParent);
        //这里分别再去监听所有插件的内容
        for (String pluginName : pluginZKs) {
            watcherAll(pluginName);
        }
        //就是各种递归一层一层的找到所有可以被监听的节点都给与监听
        zkClient.subscribeChildChanges(pluginParent, (parentPath, currentChildren) -> {
            if (CollectionUtils.isNotEmpty(currentChildren)) {
                for (String pluginName : currentChildren) {
                    watcherAll(pluginName);
                }
            }
        });
    }

```

大部分最后都会来到下面的代码, 可以看到这个代码被rule, selector, auth,metadata的监听最后调用 

![img](01.png)

```java
//最后上面的, 可以看到根局不同的group key,如果还有监听,如果路径下面有还有paths, 调用队一行的subscribe**datachagnes方法
private void subscribeChildChanges(final ConfigGroupEnum groupKey, final String groupParentPath, final List<String> childrenList) {
        switch (groupKey) {
            case SELECTOR:
                zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                    if (CollectionUtils.isNotEmpty(currentChildren)) {
                        List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                        addSubscribePath.stream().map(addPath -> {
                            String realPath = buildRealPath(parentPath, addPath);
                            cacheSelectorData(zkClient.readData(realPath));
                            return realPath;
                        }).forEach(this::subscribeSelectorDataChanges);

                    }
                });
                break;
            case RULE:
                zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                    if (CollectionUtils.isNotEmpty(currentChildren)) {
                        List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                        // Get the newly added node data and subscribe to that node
                        addSubscribePath.stream().map(addPath -> {
                            String realPath = buildRealPath(parentPath, addPath);
                            cacheRuleData(zkClient.readData(realPath));
                            return realPath;
                        }).forEach(this::subscribeRuleDataChanges);
                    }
                });
                break;
            case APP_AUTH:
                zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                    if (CollectionUtils.isNotEmpty(currentChildren)) {
                        final List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                        addSubscribePath.stream().map(children -> {
                            final String realPath = buildRealPath(parentPath, children);
                            cacheAuthData(zkClient.readData(realPath));
                            return realPath;
                        }).forEach(this::subscribeAppAuthDataChanges);
                    }
                });
                break;
            case META_DATA:
                zkClient.subscribeChildChanges(groupParentPath, (parentPath, currentChildren) -> {
                    if (CollectionUtils.isNotEmpty(currentChildren)) {
                        final List<String> addSubscribePath = addSubscribePath(childrenList, currentChildren);
                        addSubscribePath.stream().map(children -> {
                            final String realPath = buildRealPath(parentPath, children);
                            cacheMetaData(zkClient.readData(realPath));
                            return realPath;
                        }).forEach(this::subscribeMetaDataChanges);
                    }
                });
                break;
            default:
                throw new IllegalStateException("Unexpected groupKey: " + groupKey);
        }
    }
```

下面代码是核心逻辑, 当监听的节点发生变化, 同步数据 TODO: 至于 pluginDataSubscriber 等一系列的DataSubscriber后面分析

```java
private void subscribePluginDataChanges(final String pluginPath, final String pluginName) {
        zkClient.subscribeDataChanges(pluginPath, new IZkDataListener() {

            @Override
         
            public void handleDataChange(final String dataPath, final Object data) {
                Optional.ofNullable(data)
                        .ifPresent(d -> Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.onSubscribe((PluginData) d)));
            }

            @Override
            public void handleDataDeleted(final String dataPath) {
                final PluginData data = new PluginData();
                data.setName(pluginName);
                Optional.ofNullable(pluginDataSubscriber).ifPresent(e -> e.unSubscribe(data));
            }
        });
    }
```

打开 http://localhost:9095/#/plug/divide, 点击

![img](02sync_button.jpg)

在查看zk, 发现有值了

```bash
[zk: localhost:2181(CONNECTED) 19] ls /soul/plugin
[divide]
```

点一个插件就有一个插件的值, 这也未免太...我又找了一找, 果然 打开 http://localhost:9095/#/system/plugin

点击 ![img](03sync_all_button.jpg) 

再次查看全部同步了

```
[zk: localhost:2181(CONNECTED) 22] ls /soul/plugin
[context_path, divide, dubbo, hystrix, monitor, rate_limiter, resilience4j, rewrite, sentinel, sign, sofa, springCloud, tars, waf]
```

我们使用浏览器的network追踪了下

```bash
Request URL: http://localhost:9095/plugin/syncPluginAll
```

去看相关代码

```java
@PostMapping("/syncPluginAll")
    public SoulAdminResult syncPluginAll() {
        boolean success = syncDataService.syncAll(DataEventTypeEnum.REFRESH);
        if (success) {
            return SoulAdminResult.success(SoulResultMessage.SYNC_SUCCESS);
        } else {
            return SoulAdminResult.error(SoulResultMessage.SYNC_FAIL);
        }
    }
```

后面章节会以此为线索, 分析SyncDataService的工作机制
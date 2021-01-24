---
title: soul_gateway_src_code_learning_09
date: 2021-01-23 23:44:35
tags: SOUL
---

### SOUL Admin & 网关 nacos 数据同步

首先准备好nacos server, 这里不详细叙述

```bash
sh startup.sh -m standalone
.......省略
nacos is starting with standalone
nacos is starting，you can check the /Users/wincher/Documents/learning/service/nacos/logs/start.out
```

http://127.0.0.1:8848/nacos/index.html, 默认账号:密码都是nacos

![img](00nacos_config.png)

可以看到启动了, 配置项目前也都是空的

记下来配置soul

类似之前的数据同步, admin和bootstrap项目的配置文件分别打开nacos的注释

```yaml
soul:
	sync:
		nacos:
      url: localhost:8848
      namespace: 1c10d748-af86-43b9-8265-75f487d20c6c
```

soul-admin/pom.xml 已经添加了nacos-client

```xml
<dependency>
  <groupId>com.alibaba.nacos</groupId>
  <artifactId>nacos-client</artifactId>
  <version>${nacos-client.version}</version>
</dependency>
```

soul-bootstrap/pom.xml 打开nacos的注释

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2.1.0.RELEASE</version>
</dependency>
```



根据log查看HttpSyncDataService逻辑, 同样实现了SyncDataService接口

```java
public NacosSyncDataService(final ConfigService configService, final PluginDataSubscriber 	pluginDataSubscriber,
                                final List<MetaDataSubscriber> metaDataSubscribers, final List<AuthDataSubscriber> authDataSubscribers) {

        super(configService, pluginDataSubscriber, metaDataSubscribers, authDataSubscribers);
  //构造方法执行后就开始工作了
        start();
    }


    public void start() {
      //顾名思义观察不同类型的数据变化
      //NacosSyncDataService 继承自 NacosCacheHandler.java, , updatePluginMap
        watcherData(PLUGIN_DATA_ID, this::updatePluginMap);
        watcherData(SELECTOR_DATA_ID, this::updateSelectorMap);
        watcherData(RULE_DATA_ID, this::updateRuleMap);
        watcherData(META_DATA_ID, this::updateMetaDataMap);
        watcherData(AUTH_DATA_ID, this::updateAuthMap);
    }

//举一个 PLUGIN 的例子看看, 其他的也类似, oc是传进来的onchange时间
//OnChange是定义在 NacosCacheHandler的内部函数式接口, 所以可以接收一个方法作为参数, 如上面的 this::updatePluginMap
 protected void watcherData(final String dataId, final OnChange oc) {
        Listener listener = new Listener() {
            @Override
            public void receiveConfigInfo(final String configInfo) {
                oc.change(configInfo);
            }

            @Override
            public Executor getExecutor() {
                return null;
            }
        };
        oc.change(getConfigAndSignListener(dataId, listener));
        LISTENERS.getOrDefault(dataId, new ArrayList<>()).add(listener);
    }


protected void updatePluginMap(final String configInfo) {
        try {
            List<PluginData> pluginDataList = new ArrayList<>(GsonUtils.getInstance().toObjectMap(configInfo, PluginData.class).values());
          // 同步更新的配置到本地, 之后会分析pluginDataSubscriber运行的机制, 以及配置是如何缓存在本地的
            pluginDataList.forEach(pluginData -> Optional.ofNullable(pluginDataSubscriber).ifPresent(subscriber -> {
                subscriber.unSubscribe(pluginData);
                subscriber.onSubscribe(pluginData);
            }));
        } catch (JsonParseException e) {
            log.error("sync plugin data have error:", e);
        }
    }
```

admin中实现了一组DataChangedListener, 其中NacosDataChangedListener就是帮助admin同步配置到nacos的

```java
@Override
public void onPluginChanged(final List<PluginData> changed, final DataEventTypeEnum eventType) {
    updatePluginMap(getConfig(PLUGIN_DATA_ID));
    switch (eventType) {
        case DELETE:
            changed.forEach(plugin -> PLUGIN_MAP.remove(plugin.getName()));
            break;
        case REFRESH:
        case MYSELF:
            Set<String> set = new HashSet<>(PLUGIN_MAP.keySet());
            changed.forEach(plugin -> {
                set.remove(plugin.getName());
                PLUGIN_MAP.put(plugin.getName(), plugin);
            });
            PLUGIN_MAP.keySet().removeAll(set);
            break;
        default:
            changed.forEach(plugin -> PLUGIN_MAP.put(plugin.getName(), plugin));
            break;
    }
    publishConfig(PLUGIN_DATA_ID, PLUGIN_MAP);
}
```



TODO: nacos start failed
---
title: soul_gateway_src_code_learning_08
date: 2021-01-23 00:13:04
tags:
---

### SOUL Admin & 网关 Http长轮询 数据同步

书接上回, 使用Http长轮询, 数据同步,

soul-admin/src/main/resources/application.yml

```yaml
soul:
  sync:
#    websocket:
#      enabled: true
#    zookeeper:
#        url: localhost:2181
#        sessionTimeout: 5000
#        connectionTimeout: 2000
      http:
        enabled: true
```

soul-bootstrap/src/main/resources/application-local.yml, 同理

```yaml
soul :
    sync:
#        websocket :
#             urls: ws://localhost:9095/websocket
#        zookeeper:
#             url: localhost:2181
#             sessionTimeout: 5000
#             connectionTimeout: 2000
        http:
             url : http://localhost:9095
```

soul-bootstrap/pom.xml  已经添加了

```xml
<!--soul data sync start use http-->
<dependency>
    <groupId>org.dromara</groupId>
    <artifactId>soul-spring-boot-starter-sync-data-http</artifactId>
    <version>${project.version}</version>
</dependency>

```

分别启动 soul-admin, soul-bootstrap, 注意目前使用http长连接,  bootstrap启动前需要soul-admin已经启动完成, 不然会由于无法通过http连接admin而无法启动,

接下来正常启动我们看到soul-bootstrap log

```bash
2021-01-23 00:36:08.279  INFO 62417 --- [           main] o.d.s.s.data.http.HttpSyncDataService    : request configs: [http://localhost:9095/configs/fetch?groupKeys=APP_AUTH&groupKeys=PLUGIN&groupKeys=RULE&groupKeys=SELECTOR&groupKeys=META_DATA]
2021-01-23 00:36:08.562  INFO 62417 --- [onPool-worker-3] o.d.s.s.d.h.refresh.AppAuthDataRefresh   : clear all appAuth data cache
2021-01-23 00:36:08.570  INFO 62417 --- [           main] o.d.s.s.data.http.HttpSyncDataService    : get latest configs: [{"code":200,"message":"success","data":{"META_DATA":{"md5":"f96c44af66298be2e76a6b0a55a7b629","lastModifyTime":1611333355313,"data":[{"id":"1351785376513667072","appName":"springCloud-test","contextPath":null,"path":"/springcloud/**","rpcType":"springCloud","serviceName":"springCloud-test","methodName":"/springcloud","parameterTypes":null,"rpcExt":null,"enabled":true},{"id":"1351837754670669824","appName":"sofa","contextPath":null,"path":"/sofa/insert","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"insert","parameterTypes":"org.dromara.soul.examples.dubbo.api.entity.DubboTest","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true},{"id":"1351837763067666432","appName":"sofa","contextPath":null,"path":"/sofa/findAll","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findAll","parameterTypes":null,"rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true},{"id":"1351837768541233152","appName":"sofa","contextPath":null,"path":"/sofa/findById","rpcType":"sofa","serviceName":"org.dromara.soul.examples.dubbo.api.service.DubboTestService","methodName":"findById","parameterTypes":"java.lang.String","rpcExt":"{\"loadbalance\":\"hash\",\"retries\":3,\"timeout\":-1}","enabled":true}]},"SELECTOR":{"md5":"55e041ea36b518946ef09f9ed5d16ffe","lastModifyTime":1611333355190,"data":[{"id":"1351785378313023488","pluginId":"8","pluginName":"springCloud","name":"/springcloud","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"{\"serviceId\":\"springCloud-test\"}","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/**"}]},{"id":"1351837756520357888","pluginId":"11","pluginName":"sofa","name":"/sofa","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"sofa","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/sofa/**"}]}]},"PLUGIN":{"md5":"5e9a7365763848a4498b445a5a35c0b4","lastModifyTime":1611333351898,"data":[{"id":"1","name":"sign","config":null,"role":1,"enabled":false},{"id":"10","name":"sentinel","config":null,"role":1,"enabled":false},{"id":"11","name":"sofa","config":"{\"protocol\":\"zookeeper\",\"register\":\"127.0.0.1:2181\"}","role":0,"enabled":true},{"id":"12","name":"resilience4j","config":null,"role":1,"enabled":false},{"id":"13","name":"tars","config":null,"role":1,"enabled":false},{"id":"14","name":"context_path","config":null,"role":1,"enabled":false},{"id":"2","name":"waf","config":"{\"model\":\"black\"}","role":1,"enabled":false},{"id":"3","name":"rewrite","config":null,"role":1,"enabled":false},{"id":"4","name":"rate_limiter","config":"{\"master\":\"mymaster\",\"mode\":\"standalone\",\"url\":\"192.168.1.1:6379\",\"password\":\"abc\"}","role":1,"enabled":false},{"id":"5","name":"divide","config":null,"role":0,"enabled":true},{"id":"6","name":"dubbo","config":"{\"register\":\"zookeeper://localhost:2181\"}","role":1,"enabled":false},{"id":"7","name":"monitor","config":"{\"metricsName\":\"prometheus\",\"host\":\"localhost\",\"port\":\"9190\",\"async\":\"true\"}","role":1,"enabled":false},{"id":"8","name":"springCloud","config":null,"role":1,"enabled":true},{"id":"9","name":"hystrix","config":null,"role":0,"enabled":false}]},"APP_AUTH":{"md5":"d751713988987e9331980363e24189ce","lastModifyTime":1611333351778,"data":[]},"RULE":{"md5":"79088c310f2d3ec32a2c366957fc7165","lastModifyTime":1611333354636,"data":[{"id":"1351785381014155264","name":"/springcloud/order/findById","pluginName":"springCloud","selectorId":"1351785378313023488","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"path\":\"/springcloud/order/findById\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/springcloud/order/findById"}]},{"id":"1351785386563219456","name":"/springcloud/order/path/**","pluginName":"springCloud","selectorId":"1351785378313023488","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"path\":\"/springcloud/order/path/**\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/order/path/**"}]},{"id":"1351785391931928576","name":"/springcloud/order/path/**/name","pluginName":"springCloud","selectorId":"1351785378313023488","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"path\":\"/springcloud/order/path/**/name\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/order/path/**/name"}]},{"id":"1351785398240161792","name":"/springcloud/order/save","pluginName":"springCloud","selectorId":"1351785378313023488","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"path\":\"/springcloud/order/save\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/springcloud/order/save"}]},{"id":"1351785403906666496","name":"/springcloud/test/**","pluginName":"springCloud","selectorId":"1351785378313023488","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"path\":\"/springcloud/test/**\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/test/**"}]},{"id":"1351837759351513088","name":"/sofa/insert","pluginName":"sofa","selectorId":"1351837756520357888","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"retries\":0,\"loadBalance\":\"random\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/sofa/insert"}]},{"id":"1351837764875411456","name":"/sofa/findAll","pluginName":"sofa","selectorId":"1351837756520357888","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"retries\":0,\"loadBalance\":\"random\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/sofa/findAll"}]},{"id":"1351837770344783872","name":"/sofa/findById","pluginName":"sofa","selectorId":"1351837756520357888","matchMode":0,"sort":1,"enabled":true,"loged":true,"handle":"{\"retries\":0,\"loadBalance\":\"random\",\"timeout\":3000}","conditionDataList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/sofa/findById"}]}]}}}]
```

显然我们通过 http://localhost:9095/configs/fetch 同步了配置

接下来修改一个配置,可以看到

![img](00change_config.png)

admin log

```bash
2021-01-23 00:42:55.071  INFO 62391 --- [0.0-9095-exec-5] o.d.s.a.l.AbstractDataChangedListener    : update config cache[SELECTOR], old: {group='SELECTOR', md5='55e041ea36b518946ef09f9ed5d16ffe', lastModifyTime=1611333660488}, updated: {group='SELECTOR', md5='c8fa3ad3868f3ea1947f3641c753e888', lastModifyTime=1611333775071}
2021-01-23 00:42:55.077  INFO 62391 --- [-long-polling-2] a.l.h.HttpLongPollingDataChangedListener : send response with the changed group,ip=127.0.0.1, group=SELECTOR, changeTime=1611333775074
```

bootstrap log, 看上去像是受到通知,然后去拉的配置

```bash
2021-01-23 00:42:55.079  INFO 62417 --- [-long-polling-1] o.d.s.s.data.http.HttpSyncDataService    : Group config changed: [SELECTOR]
2021-01-23 00:42:55.081  INFO 62417 --- [-long-polling-1] o.d.s.s.data.http.HttpSyncDataService    : request configs: [http://localhost:9095/configs/fetch?groupKeys=SELECTOR]
2021-01-23 00:42:55.127  INFO 62417 --- [-long-polling-1] o.d.s.s.d.h.refresh.AbstractDataRefresh  : update SELECTOR config: {"md5":"c8fa3ad3868f3ea1947f3641c753e888","lastModifyTime":1611333775071,"data":[{"id":"1351785378313023488","pluginId":"8","pluginName":"springCloud","name":"/springcloud","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"{\"serviceId\":\"springCloud-test\"}","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/**"}]},{"id":"1351837756520357888","pluginId":"11","pluginName":"sofa","name":"/sofa","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"sofa","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/sofa/**"}]},{"id":"1352658003622137856","pluginId":"5","pluginName":"divide","name":"mo","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"[{\"upstreamHost\":\"baidu.com\",\"protocol\":\"http://\",\"upstreamUrl\":\"baidu.com\",\"weight\":\"50\",\"status\":true,\"timestamp\":\"0\",\"warmup\":\"0\"}]","conditionList":[{"paramType":"uri","operator":"\u003d","paramName":"/","paramValue":"/"}]}]}
2021-01-23 00:42:55.135  INFO 62417 --- [-long-polling-1] o.d.s.s.data.http.HttpSyncDataService    : get latest configs: [{"code":200,"message":"success","data":{"SELECTOR":{"md5":"c8fa3ad3868f3ea1947f3641c753e888","lastModifyTime":1611333775071,"data":[{"id":"1351785378313023488","pluginId":"8","pluginName":"springCloud","name":"/springcloud","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"{\"serviceId\":\"springCloud-test\"}","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/springcloud/**"}]},{"id":"1351837756520357888","pluginId":"11","pluginName":"sofa","name":"/sofa","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"sofa","conditionList":[{"paramType":"uri","operator":"match","paramName":"/","paramValue":"/sofa/**"}]},{"id":"1352658003622137856","pluginId":"5","pluginName":"divide","name":"mo","matchMode":0,"type":1,"sort":1,"enabled":true,"loged":true,"continued":true,"handle":"[{\"upstreamHost\":\"baidu.com\",\"protocol\":\"http://\",\"upstreamUrl\":\"baidu.com\",\"weight\":\"50\",\"status\":true,\"timestamp\":\"0\",\"warmup\":\"0\"}]","conditionList":[{"paramType":"uri","operator":"=","paramName":"/","paramValue":"/"}]}]}}}]
```

根据log查看HttpSyncDataService逻辑, 同样实现了SyncDataService接口

```java
//构造方法调用start方法
private void start() {
        // It could be initialized multiple times, so you need to control that.
        if (RUNNING.compareAndSet(false, true)) {
            // fetch all group configs.
            this.fetchGroupConfig(ConfigGroupEnum.values());
            int threadSize = serverList.size();
            this.executor = new ThreadPoolExecutor(threadSize, threadSize, 60L, TimeUnit.SECONDS,
                    new LinkedBlockingQueue<>(),
                    SoulThreadFactory.create("http-long-polling", true));
            // start long polling, each server creates a thread to listen for changes.
            this.serverList.forEach(server -> this.executor.execute(new HttpLongPollingTask(server)));
        } else {
            log.info("soul http long polling was started, executor=[{}]", executor);
        }
    }
```



```java
class HttpLongPollingTask implements Runnable {

        private String server;

        private final int retryTimes = 3;

        HttpLongPollingTask(final String server) {
            this.server = server;
        }

        @Override
        public void run() {
            while (RUNNING.get()) {
                for (int time = 1; time <= retryTimes; time++) {
                    try {
                      //核心长轮询, 看下面
                        doLongPolling(server);
                    } catch (Exception e) {
                        // print warnning log.
                        if (time < retryTimes) {
                            log.warn("Long polling failed, tried {} times, {} times left, will be suspended for a while! {}",
                                    time, retryTimes - time, e.getMessage());
                            ThreadUtils.sleep(TimeUnit.SECONDS, 5);
                            continue;
                        }
                        // print error, then suspended for a while.
                        log.error("Long polling failed, try again after 5 minutes!", e);
                        ThreadUtils.sleep(TimeUnit.MINUTES, 5);
                    }
                }
            }
            log.warn("Stop http long polling.");
        }
    }
```



```java
private void doLongPolling(final String server) {
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>(8);
        for (ConfigGroupEnum group : ConfigGroupEnum.values()) {
            ConfigData<?> cacheConfig = factory.cacheConfigData(group);
            String value = String.join(",", cacheConfig.getMd5(), String.valueOf(cacheConfig.getLastModifyTime()));
            params.put(group.name(), Lists.newArrayList(value));
        }
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_FORM_URLENCODED);
        HttpEntity httpEntity = new HttpEntity(params, headers);
        String listenerUrl = server + "/configs/listener";
        log.debug("request listener configs: [{}]", listenerUrl);
        JsonArray groupJson = null;
        try {
          //请求listener查看是否有变坏
            String json = this.httpClient.postForEntity(listenerUrl, httpEntity, String.class).getBody();
            log.debug("listener result: [{}]", json);
            groupJson = GSON.fromJson(json, JsonObject.class).getAsJsonArray("data");
        } catch (RestClientException e) {
            String message = String.format("listener configs fail, server:[%s], %s", server, e.getMessage());
            throw new SoulException(message, e);
        }
 				
        if (groupJson != null) {
            // fetch group configuration async.
            ConfigGroupEnum[] changedGroups = GSON.fromJson(groupJson, ConfigGroupEnum[].class);
            if (ArrayUtils.isNotEmpty(changedGroups)) {
                log.info("Group config changed: {}", Arrays.toString(changedGroups));
              //如果有变化,发送请求获取配置
                this.doFetchGroupConfig(server, changedGroups);
            }
        }
    }
```

CongfigController

```java
//依赖于HttpLongPollingDataChangedListener, HttpLongPollingDataChangedListener依赖于配置的soul.sync.http.enable
@ConditionalOnBean(HttpLongPollingDataChangedListener.class)
@RestController
@RequestMapping("/configs")
@Slf4j
public class ConfigController {
    @PostMapping(value = "/listener")
    public void listener(final HttpServletRequest request, final HttpServletResponse response) {
        longPollingListener.doLongPolling(request, response);
    }
}
```



```java
public void doLongPolling(final HttpServletRequest request, final HttpServletResponse response) {

        //对比请求携带的配置md5, 如果与当前不一致证明已经更新, 这里面细节就先不详看了
        List<ConfigGroupEnum> changedGroup = compareChangedGroup(request);
        String clientIp = getRemoteIp(request);

        // 如果有变化直接就返回 变化的配置group
        if (CollectionUtils.isNotEmpty(changedGroup)) {
            this.generateResponse(response, changedGroup);
            log.info("send response with the changed group, ip={}, group={}", clientIp, changedGroup);
            return;
        }

        // listen for configuration changed.
        final AsyncContext asyncContext = request.startAsync();

        // AsyncContext.settimeout() does not timeout properly, so you have to control it yourself
        asyncContext.setTimeout(0L);

        // block client's thread.
        scheduler.execute(new LongPollingClient(asyncContext, clientIp, HttpConstants.SERVER_MAX_HOLD_TIMEOUT));
    }
```


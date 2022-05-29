# Spring Cloud Netflix

### Eureka

#### 客戶端服务注册

Spring Cloud提供服务注册公共接口`ServiceRegistry`

![image-20220430223454612](http://cdn.blocketh.top/img/image-20220430223454612.png)

**EurekaClientAutoConfiguration（自动装配入口）**

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
```

**InstanceInfo（注册到EurekaServer的客户端实例信息）**

> 在Eurek Server的注册表中代表一个服务实例,其他服务实例可以通过 Instancelnfo了解该服务的相关信息从而发起服务请求。在AbstractJerseyEurekaHttpClient中调用register(InstanceInfo info)进行客户端的注册。

```java
public ApplicationInfoManager eurekaApplicationInfoManager(
      EurekaInstanceConfig config) {
   InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
   return new ApplicationInfoManager(config, instanceInfo);
}
```

**EurekaAutoServiceRegistration**

1. `EurekaAutoServiceRegistration`实例化，`EurekaAutoServiceRegistration implements SmartLifecycle`会在spring初始化完成后调用`start()` ->
2. `reg.getApplicationInfoManager().setInstanceStatus(reg.getInstanceConfig().getInitialStatus())`
3. `listener.notify(new StatusChangeEvent(prev, next))`回调到`DiscoveryClient`构造中注入的`new ApplicationInfoManager.StatusChangeListener(){ public void notify() }`
4. `notify()`中调用`instanceInfoReplicator.onDemandUpdate()`
5.  `onDemandUpdate()`

    ```java
    scheduler.submit(new Runnable() {
        @Override
        public void run() {
            InstanceInfoReplicator.this.run();
        }
    });
    ```
6.  `run()`

    ```java
    httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    ```
7. `AbstractJerseyEurekaHttpClient.register()`服务注册

**CloudEurekaClient**

Spring Cloud中定义用来服务发现的客户端接口，实现netflix的`DiscoveryClient`

```java
CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager,
      config, this.optionalArgs, this.context);
cloudEurekaClient.registerHealthCheck(healthCheckHandler);
```

![image-20220501160251574](http://cdn.blocketh.top/img/image-20220501160251574.png)

1. 在`DiscoveryClient`构造函数中初始化\`\`（心跳检测线程池）、`cacheRefreshExecutor`（缓存刷新线程池）
2.  `initScheduledTasks()`

    初始化调度任务（例如集群解析器、心跳、instanceInfo 复制器、获取

    1.  初始化监听器内部类

        ```java
        statusChangeListener = new ApplicationInfoManager.StatusChangeListener() {
            @Override
            public String getId() {
                return "statusChangeListener";
            }

            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                        InstanceStatus.DOWN == statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}", statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}", statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };
        ```

#### 发送心跳

`DiscoveryClient`中

1.  构建心跳检测线程池

    ```java
    heartbeatExecutor = new ThreadPoolExecutor(
            1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(),
            new ThreadFactoryBuilder()
                    .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                    .setDaemon(true)
                    .build()
    ```
2.  心跳检测定时器

    ```
    heartbeatTask = new TimedSupervisorTask(
            "heartbeat",
            scheduler,
            heartbeatExecutor,
            renewalIntervalInSecs, //30s
            TimeUnit.SECONDS,
            expBackOffBound,
            new HeartbeatThread()
    );
    ```
3.  执行`TimedSupervisorTask.run()` -> `HeartbeatThread.run()`

    ```java
    private class HeartbeatThread implements Runnable {

        public void run() {
            if (renew()) {
                lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
            }
        }
    }
    ```

#### 从EurekaServer获取服务列表

`DiscoveryClient`构造函数中进行判断是否从EurekaServer获取注册表信息

```java
if (clientConfig.shouldFetchRegistry()) {
    boolean primaryFetchRegistryResult = fetchRegistry(false);
}
```

1.  `clientConfig.shouldDisableDelta()`判断进行服务更新

    `getAndStoreFullRegistry()`全量更新 || `getAndUpdateDelta()`增量更新

    存储服务列表缓存

    ```java
    private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();
    ```
2. `initScheduledTasks()`
   1.  注册服务刷新定时任务

       ```java
           int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
           int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
           cacheRefreshTask = new TimedSupervisorTask(
                   "cacheRefresh",
                   scheduler,
                   cacheRefreshExecutor,
                   registryFetchIntervalSeconds,
                   TimeUnit.SECONDS,
                   expBackOffBound,
                   new CacheRefreshThread()
           );
           scheduler.schedule(
                   cacheRefreshTask,
               	//间隔30s
                   registryFetchIntervalSeconds, TimeUnit.SECONDS);
       }
       ```

       ```java
       class CacheRefreshThread implements Runnable {
           public void run() {
               refreshRegistry();
           }
       }
       ```

#### 服务端

**注册客户端**

1.  `ApplicationResource.addInstance()` -> `registry.register()`

    <img src="http://cdn.blocketh.top/img/image-20220501195613935.png" alt="image-20220501195613935" data-size="original">
2. `PeerAwareInstanceRegistryImpl.register()`
   1.  `super.register(info, leaseDuration, isReplication)`

       <img src="http://cdn.blocketh.top/img/image-20220429172725489.png" alt="image-20220429172725489" data-size="original">

       ```java
       private final ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>> registry
               = new ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>();
       ```

       > 存储客户端实例信息，第一个key存储客户端服务名称，第二个key：服务名称+端口
   2.  添加进最新变更队列

       ```java
       recentlyChangedQueue.add(new RecentlyChangedItem(lease));
       ```

       > 读写缓存load时从队列中获取
   3. `replicateToPeers(）`集群下复制客户端信息

**多级缓存**

![image-20220429193619110](http://cdn.blocketh.top/img/image-20220429193619110.png)

**读写缓存**

```java
private final LoadingCache<Key, Value> readWriteCacheMap;
```

**定时任务**

在`ResponseCacheImpl`构造函数中进行读写缓存的定时任务初始化

```java
private final java.util.Timer timer = new java.util.Timer("Eureka-CacheFillTimer", true);
```

```java
if (shouldUseReadOnlyResponseCache) {
    timer.schedule(getCacheUpdateTask(),
            new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                    + responseCacheUpdateIntervalMs),
            responseCacheUpdateIntervalMs //30s执行一次);
}
```

```java
private TimerTask getCacheUpdateTask() {
    //get -> Guava中localCache.getOrLoad(key) -> readWriteCacheMap.load() 下面的load方法
     Value cacheValue = readWriteCacheMap.get(key);
    //从只读缓存中获取
     Value currentCacheValue = readOnlyCacheMap.get(key);
     if (cacheValue != currentCacheValue) {
         //不同则加入只读缓存
     	readOnlyCacheMap.put(key, cacheValue);
     }
}
```

```java
@Override
public Value load(Key key) throws Exception {
    if (key.hasRegions()) {
        Key cloneWithNoRegions = key.cloneWithoutRegions();
        regionSpecificKeys.put(cloneWithNoRegions, key);
    }
    Value value = generatePayload(key);
    return value;
}
```

> 从读写缓存（二级缓存）获取，获取不到从三级缓存的变更队列获取，结果同一级缓存比对，不同则加入只读缓存（一级缓存）

**过期时间**

```java
serverConfig.getResponseCacheAutoExpirationInSeconds()//默认180s
```

**收到客户端心跳进行续约**

`InstanceResource.renewLease()`

更新服务最后一次收到请求的过期时间。

**自我保护机制**

**开启**

1. EurekaServerAutoConfiguration

```java
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
      EurekaServerContext serverContext) {
   return new EurekaServerBootstrap(this.applicationInfoManager,
         this.eurekaClientConfig, this.eurekaServerConfig, registry,
         serverContext);
}
```

1. EurekaServerInitializerConfiguration

```java
eurekaServerBootstrap.contextInitialized(
      EurekaServerInitializerConfiguration.this.servletContext);
```

1. `serverContext.initialize()`
   1.  每隔15min更新自我保护阈值

       ```java
        private void scheduleRenewalThresholdUpdateTask() {
               timer.schedule(new TimerTask() {
                               @Override
                                  public void run() {
                                      updateRenewalThreshold();
                                  }
                              }, serverConfig.getRenewalThresholdUpdateIntervalMs() //延期15分钟,
                       serverConfig.getRenewalThresholdUpdateIntervalMs() //每隔15分钟执行);
        }
       ```
2. `initEurekaServerContext()`
   1. `this.registry.openForTraffic(this.applicationInfoManager, registryCount)`
   2.  更新每分钟续约阈值

       ```java
       protected void updateRenewsPerMinThreshold() {
           this.numberOfRenewsPerMinThreshold = (int) 		(this.expectedNumberOfClientsSendingRenews //预期收到客户端心跳数量，每个客户端1分钟2次 * 客户端数量
                   * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds() // 30)
                   * serverConfig.getRenewalPercentThreshold() //0.85 自我保护因子);
       }
       ```

       **Renews threshold** ：Eureka Server 期望每分钟收到客户端实例续约的总数

       $$
       Renews threshold = 服务总数 * 每分钟续约数量（60s/客户端的续约间隔）* 自我保护续约百分比阈值因子
       $$

       > 客户端的注册、下线都会触发阈值重算
   3. `super.postInit();`
      1.  每隔60s进行

          `lastBucket` 把当前分钟收集到的心跳数存储上一分钟这个计数器 lastBucket

          `currentBucket`当前心跳续约数量清零

          ```java
          renewsLastMin.start();
          ```

          ```java
          public synchronized void start() {
              if (!isActive) {
                  timer.schedule(new TimerTask() {

                      @Override
                      public void run() {
                          try {
                              lastBucket.set(currentBucket.getAndSet(0));
                          } catch (Throwable e) {
                              logger.error("Cannot reset the Measured Rate", e);
                          }
                      }
                  }, sampleInterval, sampleInterval);

                  isActive = true;
              }
          }
          ```

          1.  每隔60s进行过期实例检查

              `evict(compensationTimeMs);`

              ```java
              //未开启自我保护会直接返回True
                  public boolean isLeaseExpirationEnabled() {
                      if (!isSelfPreservationModeEnabled()) {
                          return true;
                      }
                      开启自我保护进行判断，如果最后一分钟心跳续约数小于期望值 进入自我保护
                      return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
                  }
              ```

Eureka服务端为了防止Eureka客户端本身是可以正常访问的，但是由于网路通信故障等原因，造成Eureka服务端失去于客户端的连接，从而形成的不可用。

```yaml
eureka:
  server:
    enable-self-preservation: true
```

**触发**

**自我保护因子**

```yaml
eureka:
  server:
    renewal-percent-threshold: 0.85
```

**Renews (last min)** ：最后一分钟收到的客户端实际的续约总数

![image-20220429170329563](http://cdn.blocketh.top/img/image-20220429170329563.png)

**数据同步**

![image-20220502194253490](http://cdn.blocketh.top/img/image-20220502194253490.png)

**高可用集群**

```yaml
server:
  port: 8089
spring:
  application:
    name: eureka-2
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8088/eureka/
```

```yaml
server:
  port: 8088
spring:
  application:
    name: eureka-1
eureka:
  instance:
    hostname: localhost
  client:
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:8089/eureka/
```

![image-20220429162625279](http://cdn.blocketh.top/img/image-20220429162625279.png)

\+++

### Hystrix

* 服务隔离
* 服务熔断：当请求持续失败的时候，熔断5s
* 服务降级：当请求失败的时候，返回降级结果

#### 熔断

10s内，超过20次请求，并且失败率超过50%（默认）->触发熔断5s（5s内请求拒绝）

```java
@HystrixProperty(name="circuitBreaker.enable",value="true")//开启状态
@HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value="5")//最小请求数
@HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value="5000")//5s熔断时间
@HystrixProperty(name="circuitBreaker.errorThresholdPercentage",value="50")//百分比50
```

![image-20220430121133195](http://cdn.blocketh.top/img/image-20220430121133195.png)

**工作原理**

* 滑动窗口实现数据统计

![image-20220430122534505](http://cdn.blocketh.top/img/image-20220430122534505.png)

#### 服务隔离

避免因某一节点故障导致请求堆积，占用线程池资源。

*   线程池隔离

    > 针对不同的业务分配不同的线程池

    <img src="http://cdn.blocketh.top/img/image-20220430123405073.png" alt="image-20220430123405073" data-size="original">
*   计数器隔离

    <img src="http://cdn.blocketh.top/img/image-20220430123519239.png" alt="image-20220430123519239" data-size="original">

#### 集成Openfeign

![image-20220430115059114](http://cdn.blocketh.top/img/image-20220430115059114.png)

***

### Ribbon

客户端的负载均衡

#### 自动装配

**RibbonAutoConfiguration**

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration
```

```java
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
		AsyncLoadBalancerAutoConfiguration.class })
public class RibbonAutoConfiguration {
}
```

> RibbonAutoConfiguration在LoadBalancerAutoConfiguration之前进行自动装配

注入`RibbonLoadBalancerClient`

```java
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
   return new RibbonLoadBalancerClient(springClientFactory());
}
```

**LoadBalancerAutoConfiguration**

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.client.loadbalancer.LoadBalancerAutoConfiguration
```

启动时注入`RestTemplate`实例

```java
public class LoadBalancerAutoConfiguration {
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
}
```

针对`RestTemplate`增加拦截器

```java
static class LoadBalancerInterceptorConfig {

   @Bean
   public LoadBalancerInterceptor loadBalancerInterceptor(LoadBalancerClient loadBalancerClient,
         LoadBalancerRequestFactory requestFactory) {
      return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
   }

   @Bean
   @ConditionalOnMissingBean
   public RestTemplateCustomizer restTemplateCustomizer(final LoadBalancerInterceptor loadBalancerInterceptor) {
      return restTemplate -> {
         List<ClientHttpRequestInterceptor> list = new ArrayList<>(restTemplate.getInterceptors());
         list.add(loadBalancerInterceptor);
         restTemplate.setInterceptors(list);
      };
   }
}
```

> `RestTemplate`执行`execute()`时被`LoadBalancerInterceptor`拦截器进行拦截，执行`LoadBalancerInterceptor.intercept()` -> `this.loadBalancer.execute()` -> `RibbonLoadBalancerClient`执行`execute` -> `getServer()` -> `loadBalancer.chooseServer`负载均衡

**LoadBalancer**

![image-20220430194940319](http://cdn.blocketh.top/img/image-20220430194940319.png)

**IRule（负载均衡策略）**

![image-20220430195429676](http://cdn.blocketh.top/img/image-20220430195429676.png)

* RandomRule：随机选取服务实例
* RoundRobinRule：轮询
* Retry：轮询基础上增加重试
* WeightedResponseTimeRule：权重

#### 静态服务地址获取

```yaml
<clientName>.ribbon.listOfServers=hostname:port
```

```java
@Override
public List<Server> getUpdatedListOfServers() {
       String listOfServers = clientConfig.get(CommonClientConfigKey.ListOfServers);
       return derive(listOfServers);
}
```

#### IPing

```java
public class RibbonClientConfiguration{
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
		……
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
}
```

`new ZoneAwareLoadBalancer()){super()}` -> `new DynamicServerListLoadBalancer(){super()}` -> `new BaseLoadBalancer(){ initWithConfig() }`

```java
public void setPing(IPing ping) {
    if (ping != null) {
        if (!ping.equals(this.ping)) {
            this.ping = ping;
            setupPingTask(); // since ping data changed
        }
    } else {
        this.ping = null;
        // cancel the timer task
        lbTimer.cancel();
    }
}
```

$$
pingIntervalSeconds = 30s
$$

```java
void setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
        lbTimer.cancel();
    }
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
            true);
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    forceQuickPing();
}
```

启动IPing定时器，每隔30s去访问目标服务

```java
class PingTask extends TimerTask {
    public void run() {
        try {
           new Pinger(pingStrategy).runPinger();
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Error pinging", name, e);
        }
    }
}
```

#### 更新服务列表（集成Eureka）

```java
public class RibbonClientConfiguration{
	@Bean
    @ConditionalOnMissingBean
    public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
       return new PollingServerListUpdater(config);
    }
}
```

```java
public PollingServerListUpdater(IClientConfig clientConfig) {
    this(LISTOFSERVERS_CACHE_UPDATE_DELAY,//1000
         getRefreshIntervalMs(clientConfig)//30 * 1000);
}
```

```java
public class RibbonClientConfiguration{
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
		……
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
}
```

`new ZoneAwareLoadBalancer()){super()}` -> `new DynamicServerListLoadBalancer(){super()}`

```java
public class DynamicServerListLoadBalancer{
	public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
       ……
        this.serverListUpdater = serverListUpdater;
        if (filter instanceof AbstractServerListFilter) {
            ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
        }
        restOfInit(clientConfig);
    }
}
```

```java
void restOfInit(IClientConfig clientConfig) {
 	……
    enableAndInitLearnNewServersFeature();

    updateListOfServers();
    ……
}
```

```java
public void enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
    serverListUpdater.start(updateAction);
}
```

启动更是服务定时器，间隔30s执行

```java
public synchronized void start(final UpdateAction updateAction) {
    if (isActive.compareAndSet(false, true)) {
        final Runnable wrapperRunnable = new Runnable() {
            @Override
            public void run() {
                if (!isActive.get()) {
                    if (scheduledFuture != null) {
                        scheduledFuture.cancel(true);
                    }
                    return;
                }
                try {
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };

        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                wrapperRunnable,
                initialDelayMs, //延迟1秒执行
                refreshIntervalMs,//间隔30s执行
                TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
```

更新服务列表

```java
void restOfInit(IClientConfig clientConfig) {
 	……
    updateListOfServers();
    ……
}
```

```java
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
```

```java
public class DiscoveryEnabledNIWSServerList{
  public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
        return obtainServersViaDiscovery();
    }
    
    private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
        List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

        EurekaClient eurekaClient = eurekaClientProvider.get();
        if (vipAddresses!=null){
            for (String vipAddress : vipAddresses.split(",")) {
                // if targetRegion is null, it will be interpreted as the same region of client
                List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
                for (InstanceInfo ii : listOfInstanceInfo) {
                    if (ii.getStatus().equals(InstanceStatus.UP)) {
					  ……
                        DiscoveryEnabledServer des = createServer(ii, isSecure, shouldUseIpAddr);
                        serverList.add(des);
                    }
                }
                if (serverList.size()>0 && prioritizeVipAddressBasedServers){
                    break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
                }
            }
        }
        return serverList;
    }
}
```

```java
List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
```

从EurekaClient从服务端拉取保存的本地缓存中获取

```java
public class DiscoveryClient implements EurekaClient {
	applications = this.localRegionApps.get();
}
```

更新服务列表到Ribbon中`BaseLoadBalancer`类的`protected volatile List<Server> allServerList = Collections.synchronizedList(new ArrayList<Server>());`中

```
public void setServersList(List lsrv) {
	
	allServerList = allServers;
}
```

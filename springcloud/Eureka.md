### Eureka

#### Eureka 基本架构

三种角色：

- 服务注册中心：提供服务注册和发现功能（EurekaServer）
- 服务提供者：提供服务（EurekaClient）
- 服务消费者：消费服务（EurekaClient）

![image-20210413101615861](/Users/liqin/Library/Application Support/typora-user-images/image-20210413101615861.png)

服务消费的基本过程：

首先需要服务注册中心，服务提供者向服务注册中心注册，将自己的信息（服务名和服务IP地址等）通过 REST API 的形式提交给服务注册中心。消费者也向服务注册中心注册，同时服务消费者获取一份服务注册列表信息，该列表包含了所有向服务注册中心注册的服务信息。获取服务注册列表信息后，服务消费者就知道服务提供者的 IP 地址，就可以通过 Http 远程调用来消费

#### Eureka 的基本概念

- 服务注册（Register）：当 Eureka 客户端向 Eureka 服务器注册时，Eureka 客户端提供自身的元数据，比如 IP 地址、端口等信息
- 服务续约（Renew）：Eureka 客户端默认情况下每隔 30 秒发送一次心跳进行服务续约。通过服务续约来告知 Eureka 服务器该客户端可用。若注册中心在 90 秒没收到 Eureka 客户端心跳，Eureka 服务器会将 Eureka 客户端实例从注册列表删除
- 获取服务注册列表信息（Fetch Registeres）：Eureka  客户端从服务器获取服务注册表信息，并将其缓存在本地。客户端会使用服务注册列表信息查找其他服务，从而进行远程调用。该注册列表信息定时（每30秒钟）更新一次。每次返回注册列表信息可能与  Eureka  客户端的缓存信息不同, Eureka 客户端自动处理。如果由于某种原因导致注册列表信息不能及时匹配，Eureka  客户端则会重新获取整个注册表信息。 Eureka 服务器缓存注册列表信息，并将整个注册表以及每个应用程序的信息进行了压缩，压缩内容和没有压缩的内容完全相同。Eureka  客户端和 Eureka  服务器可以使用JSON / XML格式进行通讯。在默认的情况下 Eureka  客户端使用压缩 JSON 格式来获取注册列表的信息。
- 服务下线（Cancel）：Eureka 客户端在程序关闭时向 Eureka 服务器发送下线请求。 发送请求后，该客户端实例信息将从服务器的实例注册表中删除。该下线请求不会自动完成，需要在程序关闭时调用一下代码：

```java
DiscoveryManager.getInstance().shutdownComponent();
```

- 服务剔除（Eviction）：在默认的情况下，当 Eureka 客户端连续90秒（3个续约周期）没有向Eureka服务器发送服务续约，即心跳，Eureka 服务器会将该服务实例从服务注册列表删除，即服务剔除

#### 服务注册

服务注册主要涉及类：DiscoveryClient、InstanceInfoReplicator、AbstractJerseyEurekaHttpCilent、AppicationResource、PeerAwareInstanceRegistryImpl

首先先看 DiscoverCilent 的构造方法：

```java
@Inject
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
   	//在之前有很多参数校验和初始化工作
    //重点是这行代码，主要工作时初始化调度任务
    initScheduledTasks();
    //后面也是一些日志打印等工作
}
```

由于主要方法是 `initScheduledTasks`，所以我们跟进去看一看

```java

/**
 * Initializes all scheduled tasks.
 */
private void initScheduledTasks() {
    //任务调度获取注册列表
    if (clientConfig.shouldFetchRegistry()) {
      	//从eureka服务端获取注册列表时间默认30秒(实现类中DefaultEurekaClientConfig)
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
      	//定义任务调度
        cacheRefreshTask = new TimedSupervisorTask(
                "cacheRefresh",
                scheduler,
                cacheRefreshExecutor,
                registryFetchIntervalSeconds,
                TimeUnit.SECONDS,
                expBackOffBound,
                new CacheRefreshThread()
        );
      	//执行
        scheduler.schedule(
                cacheRefreshTask,
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }
		//如果向注册中心注册自己，则执行如下代码
    if (clientConfig.shouldRegisterWithEureka()) {
      	//服务续约的时间 30s
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

        // Heartbeat timer
      	//发送心跳
        heartbeatTask = new TimedSupervisorTask(
                "heartbeat",
                scheduler,
                heartbeatExecutor,
                renewalIntervalInSecs,
                TimeUnit.SECONDS,
                expBackOffBound,
                new HeartbeatThread()
        );
      	//执行心跳的任务调度
        scheduler.schedule(
                heartbeatTask,
                renewalIntervalInSecs, TimeUnit.SECONDS);

      	//重要：更新和复制本地实例信息到远程服务器的任务(具体作用可以看类上的注释）
        instanceInfoReplicator = new InstanceInfoReplicator(
                this,
                instanceInfo,
                clientConfig.getInstanceInfoReplicationIntervalSeconds(),
                2); // burstSize:可以限制任务处理速率

      	//这是一个状态改变的监听器
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

        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }
				//启动线程
        instanceInfoReplicator.
      			start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

由于我们需要将实例注册到 Eureka 服务器上，接下来去看看 InstanceInfoReplicator 类，该类发现实现了 Runable 接口，所以先聚焦于 run 方法上

```java
public void run() {
    try {
      	//刷新实例信息
        discoveryClient.refreshInstanceInfo();
        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
          	//这个方法比较重要，真正注册的方法
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```

在上述中我们看到会调用 discoveryCilent 中的 register 方法

```java
/**
 * Register with the eureka service by making the appropriate REST call.
 */
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
      	//这里又调用了一个注册方法（主要是调用 AbstractJerseyEurekaHttpCilent）
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier, e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier, httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```

接下来看看 AbstractJerseyEurekaHttpCilent 中的 register 方法

```java
@Override
public EurekaHttpResponse<Void> register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    ClientResponse response = null;
    try {
      	//实际上就是这个调用服务端的API进行注册的
        Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
        addExtraHeaders(resourceBuilder);
        response = resourceBuilder
                .header("Accept-Encoding", "gzip")
                .type(MediaType.APPLICATION_JSON_TYPE)
                .accept(MediaType.APPLICATION_JSON)
                .post(ClientResponse.class, info);
        return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
    } finally {
        if (logger.isDebugEnabled()) {
            logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                    response == null ? "N/A" : response.getStatus());
        }
        if (response != null) {
            response.close();
        }
    }
}
```

这里会有个问题，调用的 API 是在哪儿呢？主要是调用了 AppicationResource 中的接口

```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    //大量的参数校验
		//...
    //处理客户端可能注册坏的DataCenterInfo并丢失数据的情况
		//...
  	
  	//下面这个是重点：会调用 PeerAwareInstanceRegistryImpl 的 register
    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

进入 PeerAwareInstanceRegistryImpl 的 register 看一下

```java
public void register(InstanceInfo info, boolean isReplication) {
  	int leaseDuration = 90;
  	if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
  			leaseDuration = info.getLeaseInfo().getDurationInSecs();
  	}
  	//核心方法，主要逻辑
  	super.register(info, leaseDuration, isReplication);
  	this.replicateToPeers(PeerAwareInstanceRegistryImpl.Action.Register, info.getAppName(), 		info.getId(), info, (InstanceStatus)null, isReplication);
}
```

接下来调用了父类的 register 方法，这是核心方法

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        read.lock();
      	//registry 是一个全局的 map
      	//根据appName 获取map，若不存在则创建一个 CurrentHashMap()
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
      	//根据这是来自其他eureka服务器的复制，还是eureka客户端发起的操作，递增给定统计数据的计数器
        REGISTER.increment(isReplication);
        if (gMap == null) {
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
      	//看容器是否存在当前注册实例
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        //保留最后一个脏时间戳而不覆盖它(如果已经有租约的话)
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                        " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            // The lease does not exist and hence it is a new registration
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
      	//设置租约
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
      	//set进map中
        gMap.put(registrant.getId(), lease);
      	//加入注册队列
        recentRegisteredQueue.add(new Pair<Long, String>(
                System.currentTimeMillis(),
                registrant.getAppName() + "(" + registrant.getId() + ")"));
        //后面主要是状态覆盖
    } finally {
        read.unlock();
    }
}
```

#### 服务续约

由于代码流程差不多，所以不再附上代码，只列出关键调用点即可：

- 首先是在 DiscoverClient 中 initScheduledTasks 方法中开启了线程 HeartbeatThread（主要是在心跳任务创建时生成）

- 再线程 HeartThread 中调用了 DiscoveryClient 中的 renew 方法

- 再调用了 AbstractJerseyEurekaHttpClient 中的 sendHeartBeat 方法

- 在 sendHeartBeat 方法中远程调用 InstanceResource 中的 renewLease 接口

- 在接口中调用 InstanceRegistry 的 renew 方法

- 再调用 InstanceRegistry 父类 PeerAwareInstanceRegistryImpl 的 renew 方法

- 再调用 PeerAwareInstanceRegistryImpl 父类的 renew 方法

#### Eureka注册为什么慢

- Eureka 客户端延迟注册，默认的延迟注册时间为 40s

```java
//类名：InstanceInfoReplicator

//调用处(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds()默认为40s)
instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
//具体方法
public void start(int initialDelayMs) {
    if (this.started.compareAndSet(false, true)) {
        this.instanceInfo.setIsDirty();
        Future next = this.scheduler.schedule(this, (long)initialDelayMs, TimeUnit.SECONDS);
        this.scheduledPeriodicRef.set(next);
    }
}
```

- Eureka 服务端响应缓存：每 30 秒更新一次响应缓存，可以通过更改配置如下配置修改。所以即使是刚刚注册的实例，也不会立即出现在服务注册列表中

```yaml
eureka:
  server:
    response-cache-update-interval-ms: 
```

- Eureka 客户端缓存：也是 30 秒更新一次缓存，所以刷新本地缓存和发现其他新注册的实例可能需要 30 秒
- LoadBalancer 缓存：Ribbon 的负载均衡会从本地的 Eureka 客户端获取注册列表信息。才缓存每 30 秒刷新一次，可以通过如下配置修改。所以说至少需要 30 秒时间才能使用新注册的实例

```yaml
ribbon: 
  ServerListRefreshInterval: 
```

#### 自我保护机制

自我保护机制的工作机制是：如果在15分钟内超过85%的客户端节点都没有正常的心跳，那么 Eureka 就认为客户端与注册中心出现了网络故障，Eureka Server 自动进入自我保护机制，进入自我保护及之后就不会再剔除注册列表信息

关闭自我保护机制（默认是开启的）:

```yaml
eureka:
  server:
    enable-self-preservation: false
```


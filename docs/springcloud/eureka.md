# eureka

针对于  [spring cloud openfeign](springcloud/openfeign.md) 中的使用示例，完成了面向接口的远程通信功能，但是还有一个问题，我们的服务地址信息是写入到配置文件中，正式环境中，这些服务肯定是动态的，基于 openfeign 我们是实现不了服务的动态感知的。所以引入了 eureka 组件实现注册中心的功能。

# 使用实例

**启动一个 eureka 服务**

新增spring boot 项目，在启动类上添加 @EnableEurekaServer 注解

pom.xml

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

application.yml

```yaml
server:
  port: 9090
eureka:
  instance:
    hostname: localhost   #eureka 服务端实例名称
  client:
    register-with-eureka: false   #false 表示不向注册中心中注册自己
    fetch-registry: false   #false 表示自己端就是注册中心,我的职责就是维护服务实例，并不需要去检索服务
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/   #设置与 Eureka Server 交互的地址查询服务和注册服务都需要依赖这个地址(服务暴露的地址)
```



**其他服务与注册中心交互**

引入 jar

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

application.yml

```properties
server:
  port: 8080
eureka:
  client:
    service-url:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/   #设置与 Eureka Server 交互的地址查询服务和注册服务都需要依赖这个地址(服务暴露的地址)
```

# 服务注册流程图



![eureka-2](image\eureka-2.png ':size=50%')



# 服务注册源码分析

要找到 Eureka 的服务注册入口，首先要知道 Spring 中的 SmartLifecycle 接口，当 Spring 容器加载完所有的 Bean 并且初始化之后，会继续回调实现了 SmartLifecycle 接口对应的方法（start方法）。

Spring boot 执行 SmartLifecycle 的调用链路如下：

```java
SpringApplication.run() -> 
    this.refreshContext(context); -> 
    this.refresh(context); -> 
    ServletWebServerApplicationContext.refresh(); ->
    this.finishRefresh(); ->
    AbstractApplicationContext.finishRefresh() ->
    DefaultLifecycleProcessor.onRefresh() -> 
    this.startBeans(); ->
    this.start(); ->
    this.doStart()->
    SmartLifecycle.start()
```



找到 DefaultLifecycleProcessor.start() 方法，在 `Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();` 这一行加上 Debug，然后启动 Spring boot 项目，可以看到 lifecycleBeans 里面已经包含了 eurekaAutoServiceRegistration，从而我们知道 eureka 的服务注册肯定是在 `EurekaAutoServiceRegistration.start()` 方法内执行的。

![eureka](image/eureka-1.png ":size=40%")

## EurekaAutoServiceRegistration.start()

```java
public void start() {
    // only set the port if the nonSecurePort or securePort is 0 and this.port != 0
    // 安全端口处理
    if (this.port.get() != 0) {
        if (this.registration.getNonSecurePort() == 0) {
            this.registration.setNonSecurePort(this.port.get());
        }

        if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
            this.registration.setSecurePort(this.port.get());
        }
    }

    // only initialize if nonSecurePort is greater than 0 and it isn't already running
    // because of containerPortInitializer below
    if (!this.running.get() && this.registration.getNonSecurePort() > 0) {
		// 注册服务
        // this.serviceRegistry = EurekaClientAutoConfiguration
        // 在EurekaClientAutoConfiguration 中初始化
        this.serviceRegistry.register(this.registration);
		// 发布事件
        this.context.publishEvent(new InstanceRegisteredEvent<>(this,
                                                                this.registration.getInstanceConfig()));
        this.running.set(true);
    }
}
```

## EurekaServiceRegistry.register()

```java
public void register(EurekaRegistration reg) {
    maybeInitializeClient(reg);

    if (log.isInfoEnabled()) {
        log.info("Registering application "
                 + reg.getApplicationInfoManager().getInfo().getAppName()
                 + " with eureka with status "
                 + reg.getInstanceConfig().getInitialStatus());
    }
	// 设置当前实例的状态，一旦这个实例的状态发生变化，只要状态不是 DOWN，那么就会被监听器监听并执行服务注册
    reg.getApplicationInfoManager()
        .setInstanceStatus(reg.getInstanceConfig().getInitialStatus());
	// 设置监控检测的处理
    reg.getHealthCheckHandler().ifAvailable(
        healthCheckHandler ->reg.getEurekaClient().registerHealthCheck(healthCheckHandler));
}
```

## ApplicationInfoManager.serInstancesStatus()

```java
public synchronized void setInstanceStatus(InstanceStatus status) {
    InstanceStatus next = instanceStatusMapper.map(status);
    if (next == null) {
        return;
    }
	// 设置状态 = UP
    InstanceStatus prev = instanceInfo.setStatus(next);
    if (prev != null) {
        for (StatusChangeListener listener : listeners.values()) {
            try {
                // 通知监听器处理
                listener.notify(new StatusChangeEvent(prev, next));
            } catch (Exception e) {
                logger.warn("failed to notify listener: {}", listener.getId(), e);
            }
        }
    }
}
```

从代码中看，并没有去注册服务实例，而是仅仅设置了一个状态以及监听一个状态变更的事件。

监听器的实例是 StatusChangeListener，发现只有一个监听方法，那么肯定有地方接受这个事件的通知。那就要找到这个事件是在哪里初始化的？

这个事件是 ApplicationInfoManager 类中一个属性并看到这个属性是在构造方法中初始化值。因为 ApplicationInfoManager 是

EurekaRegistration 中的一个属性，而 EurekaRegistration 是在 EurekaClientAutoConfiguration 中初始化的。

所以我们从 EurekaClientAutoConfiguration 查看，看到 ApplicationInfoManager、EurekaClient、EurekaServiceRegistry等等的初始化。

```java
public class EurekaClientAutoConfiguration {
    @Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingRefreshScope
	protected static class EurekaClientConfiguration {

		@Autowired
		private ApplicationContext context;

		@Autowired
		private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;

		@Bean(destroyMethod = "shutdown")
		@ConditionalOnMissingBean(value = EurekaClient.class,
				search = SearchStrategy.CURRENT)
		public EurekaClient eurekaClient(ApplicationInfoManager manager,
				EurekaClientConfig config) {
			return new CloudEurekaClient(manager, config, this.optionalArgs,
					this.context);
		}

		@Bean
		@ConditionalOnMissingBean(value = ApplicationInfoManager.class,
				search = SearchStrategy.CURRENT)
		public ApplicationInfoManager eurekaApplicationInfoManager(
				EurekaInstanceConfig config) {
			InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
			return new ApplicationInfoManager(config, instanceInfo);
		}

		@Bean
		@ConditionalOnBean(AutoServiceRegistrationProperties.class)
		@ConditionalOnProperty(
				value = "spring.cloud.service-registry.auto-registration.enabled",
				matchIfMissing = true)
		public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient,
				CloudEurekaInstanceConfig instanceConfig,
				ApplicationInfoManager applicationInfoManager, @Autowired(
						required = false) ObjectProvider<HealthCheckHandler> healthCheckHandler) {
			return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager)
					.with(eurekaClient).with(healthCheckHandler).build();
		}

	}
}
```

看到代码，eureka 在启动的时候做了自动装配，对 EurekaClient 进行初始化，从名字可以猜测这是一个用来实现和服务端交互的处理类。具体看这个初始化过程做了什么？

## CloudEurekaClient 构造

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
                         EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args,
                         ApplicationEventPublisher publisher) {
    // 调用父类构造 父类 = DiscoveryClient
    super(applicationInfoManager, config, args);
    this.applicationInfoManager = applicationInfoManager;
    this.publisher = publisher;
    this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,
                                                          "eurekaTransport");
    ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```

## DiscoveryClient 构造

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args, EndpointRandomizer randomizer) {
    this(applicationInfoManager, config, args, new Provider<BackupRegistry>() {
        // 省略部分代码
    }, randomizer);
}
```

构造调用

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
    // 省略部分代码 主要是参数赋值逻辑
    
    // 是否要从eureka server上获取服务地址信息
    if (config.shouldFetchRegistry()) {
        this.registryStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }
	// 是否要注册到eureka server上
    if (config.shouldRegisterWithEureka()) {
        this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

    logger.info("Initializing Eureka in region {}", clientConfig.getRegion());
	
    // 如果不需要注册并且不需要更新服务地址
    if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
        logger.info("Client configured to neither register nor query for data.");
        scheduler = null;
        heartbeatExecutor = null;
        cacheRefreshExecutor = null;
        eurekaTransport = null;
        instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        initRegistrySize = this.getApplications().size();
        registrySize = initRegistrySize;
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                    initTimestampMs, initRegistrySize);

        return;  // no need to setup up an network tasks and we are done
    }
	
    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        scheduler = Executors.newScheduledThreadPool(2,
                                                     new ThreadFactoryBuilder()
                                                     .setNameFormat("DiscoveryClient-%d")
                                                     .setDaemon(true)
                                                     .build());
		// 心跳定时任务
        heartbeatExecutor = new ThreadPoolExecutor(
            1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(),
            new ThreadFactoryBuilder()
            .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
            .setDaemon(true)
            .build()
        );  // use direct handoff
		// 心跳定时任务
        cacheRefreshExecutor = new ThreadPoolExecutor(
            1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
            new SynchronousQueue<Runnable>(),
            new ThreadFactoryBuilder()
            .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
            .setDaemon(true)
            .build()
        );  // use direct handoff

        eurekaTransport = new EurekaTransport();
        scheduleServerEndpointTask(eurekaTransport, args);

        AzToRegionMapper azToRegionMapper;
        if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
            azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
        } else {
            azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
        }
        if (null != remoteRegionsToFetch.get()) {
            azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
        }
        instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
    } catch (Throwable e) {
        throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
    }

    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }

    // call and execute the pre registration handler before all background tasks (inc registration) is started
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }
	// 如果需要注册到Eureka server并且是开启了初始化的时候强制注册，则调用register()发起服务注册
    if (clientConfig.shouldRegisterWithEureka() && clientConfig.shouldEnforceRegistrationAtInit()) {
        try {
            if (!register() ) {
                throw new IllegalStateException("Registration error at startup. Invalid server response.");
            }
        } catch (Throwable th) {
            logger.error("Registration error at startup: {}", th.getMessage());
            throw new IllegalStateException(th);
        }
    }

    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
    // 启动定时任务
    initScheduledTasks();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register timers", e);
    }

    // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
    // to work with DI'd DiscoveryClient
    DiscoveryManager.getInstance().setDiscoveryClient(this);
    DiscoveryManager.getInstance().setEurekaClientConfig(config);

    initTimestampMs = System.currentTimeMillis();
    initRegistrySize = this.getApplications().size();
    registrySize = initRegistrySize;
    logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, initRegistrySize);
}
```

## DiscoveryClient.initScheduledTasks()

这个方法主要是启动一个定时任务：

- 如果开起了从注册中心刷新服务列表，则会开启 cacheRefreshExecutor 定时任务
- 如果开启了服务注册到 Eureka，会进行一下事情
  - 建立心跳检测
  - 实例化 StatusChangeListener 来监听状态变化

```java
private void initScheduledTasks() {
    // 如果开起了从注册中心刷新服务列表，则会开启 cacheRefreshExecutor 定时任务
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
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
            registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }
	// 开启了服务注册到Eureka
    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

        // Heartbeat timer
        // 创建心跳检测任务
        heartbeatTask = new TimedSupervisorTask(
            "heartbeat",
            scheduler,
            heartbeatExecutor,
            renewalIntervalInSecs,
            TimeUnit.SECONDS,
            expBackOffBound,
            new HeartbeatThread()
        );
        scheduler.schedule(
            heartbeatTask,
            renewalIntervalInSecs, TimeUnit.SECONDS);

        // InstanceInfo replicator
        // 状态变更处理监听
        instanceInfoReplicator = new InstanceInfoReplicator(
            this,
            instanceInfo,
            clientConfig.getInstanceInfoReplicationIntervalSeconds(),
            2); // burstSize

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
                // 根据实例数据是否发生变化，来触发服务注册中心的数据
                instanceInfoReplicator.onDemandUpdate();
            }
        };
		// 注册实例状态变化的监听
        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }

		// 启动一个实例信息复制器，主要就是为了开启一个定时线程，
        // 每40秒判断实例信息是否变更，如果变更了则重新注册
		instanceInfoReplicator.start(
			clientConfig.getInitialInstanceInfoReplicationIntervalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

到这里我们就找到了在 ApplicationInfoManager.serInstancesStatus() 中创建的 StatusChangeListener.notify() 接受处理的地方。

这里状态变更了执行 InstanceInfoReplicator.onDemandUpdate() 进行处理。

## InstanceInfoReplicator.onDemandUpdate()

```java
public boolean onDemandUpdate() {
    // 限流判断
    if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
        if (!scheduler.isShutdown()) {
            // 提交一个任务
            scheduler.submit(new Runnable() {
                @Override
                public void run() {
                    logger.debug("Executing on-demand update of local InstanceInfo");
					// 取出来之前已经提交的任务，也就是 start() 中提交的更新任务
                    // 如果任务还没有执行完成，则取消之前的任务
                    Future latestPeriodic = scheduledPeriodicRef.get();
                    if (latestPeriodic != null && !latestPeriodic.isDone()) {
                        logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                        latestPeriodic.cancel(false);
                    }
					// 通过调用 run() 令任务在延时后执行，相当于周期性任务中的一次
                    InstanceInfoReplicator.this.run();
                }
            });
            return true;
        } else {
            logger.warn("Ignoring onDemand update due to stopped scheduler");
            return false;
        }
    } else {
        logger.warn("Ignoring onDemand update due to rate limiter");
        return false;
    }
}
```

## InstanceInfoReplicator.this.run()

实际上和前面自动装配所执行的服务注册方法是一样的，也就是调用 register() 方法进行服务注册，并在 finally 中，每 30s 会定时执行以下当前的 run() 方法进行检查。

```java
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();

        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
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

## DiscoveryClient.register()

```java
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        // 在DiscoveryClient构造中
        // scheduleServerEndpointTask(eurekaTransport, args) 方法中初始化
        // eurekaTransport.registrationClient == SessionedEurekaHttpClient
        // 所以调用父类的方法 AbstractJerseyEurekaHttpClient
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

## AbstractJerseyEurekaHttpClient.register()

这里就是发起一次 http 请求，访问 Eureka-Server 服务中的 apps/${app_name} 接口，将当前服务实例的信息发送到 Eureka_Server 进行保存。

```java
public EurekaHttpResponse<Void> register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    ClientResponse response = null;
    try {
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

## 服务注册总结

从上面分析可知，在服务启动发起服务注册时，有两个地方执行服务注册的任务

1. 在 Spring boot 启动时，由于自动装配机制将 CloudEurekaClient 注入到了容器，并且执行了构造方法，而在构造方法中有一个定时任务每 40s 会执行一次判断，判断实例信息是否发生了变化，如果是则发起服务注册的流程
2. 在 Spring boot 启动时，通过 refresh() 方法，最终调用 StatusChangeListener.notify() 进行服务状态变更的监听，而这个监听的方法收到事件通知之后会去执行服务注册

# Eureka Server 接受请求

在服务注册时候，我们通过 http 请求通知服务端进行服务节点信息的维护存储。

请求入口在：`com.netflix.eureka.resources.ApplicationResource#addInstance()`

## ApplicationResource.addInstance()

```java
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info,
                            @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(), isReplication);
    // validate that the instanceinfo contains all the necessary required fields
    // 省略部分代码 主要是参数校验

    // handle cases where clients may be registering with bad DataCenterInfo with missing data
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId = ((UniqueIdentifier) dataCenterInfo).getId();
        if (isBlank(dataCenterInfoId)) {
            boolean experimental = "true".equalsIgnoreCase(serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " + dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            } else if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo) dataCenterInfo;
                String effectiveId = amazonInfo.get(AmazonInfo.MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(AmazonInfo.MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }
	// 注册服务
    // registry == PeerAwareInstanceRegistry
    registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();  // 204 to be backwards compatible
}
```

## PeerAwareInstanceRegistry.register()

主要代码逻辑

- leaseDuration 是续约过期时间，默认 90s，当服务端超过 90s 没有收到客户端心跳，则主动剔除节点
- 调用 super.register() 发起服务注册
- 将数据同步到集群中的其他节点上（实现简单，获取集群中的所有信息，逐个发起注册）

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    // 服务续约过期时间
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        // 自定义的心跳时间
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    // 服务注册
    super.register(info, leaseDuration, isReplication);
    // 数据同步到集群中的其他节点
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

## AbstractInstanceRegistry.register()

简单来说，Eureka-Server 的服务注册，实际上是将客户端传递过来的实例数据保存到 Eureka-Server 中 的 ConcurrentHashMap 中。

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        read.lock();
        // 从 registry 中获取当前实例信息
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        // 增加注册次数到监控信息中
        REGISTER.increment(isReplication);
        // 如果当前服务是第一次注册，则初始化一个 ConcurrentHashMap
        if (gMap == null) {
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        // 从 registry 中查询已经存在的 Lease 信息
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        // 当instance已经存在，那就和客户端的instance的信息做比较，时间最新的为有效的instance信息
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
            // 当lease不存在，是未注册过的服务
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        // 构建一个 Lease 
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            // 当原来存在 lease 的信息时，设置 serviceUpTimestamp
            // 保证服务启动的时间一直是第一次注册的那个时间
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        gMap.put(registrant.getId(), lease);
        // 添加到队列内
        recentRegisteredQueue.add(new Pair<Long, String>(
            System.currentTimeMillis(),
            registrant.getAppName() + "(" + registrant.getId() + ")"));
        // This is where the initial state transfer of overridden status happens
        // 检查实例状态是否发生变化，如果是并且存在，则覆盖原来的状态
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                         + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        // 得到instanceStatus，判断是否为up状态
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        // 设置注册类型为 ADDED
        registrant.setActionType(ActionType.ADDED);
        // 续约变更记录队列，记录了实时的每次变化，用于注册信息的增量获取
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        // 让缓存失效
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```

## 总结

至此，我们服务端接受处理客户端的注册请求做了一个简单的分析。实际上 Eureka Server 会把客户端的地址信息保存到 ConcurrentHashMap 中存储，并且服务提供者与注册中心直接会建立一个心跳检测机制，用于监控服务提供者的健康状态。

# Eureka Server 三级缓存

在注册中心设计的架构中，我们在服务端肯定是存储有关客户端信息。通常的注册中心会存磁盘(Zookeeper)、数据库(Nacos)等，而 Eureka 则使用的是存储在内存中。

Eureka Server 中注册服务存在三个变量：（registry、readWriteCacheMap、readOnlyCacheMap）保存服务注册信息，默认情况下定时任务会每隔 30s 将 readWriteCacheMap 同步到 readOnlyCacheMap，每隔 60s 清理超过 90s 未续约的节点， Eureka Client 每隔 30s 从 readOnlyCacheMap 拉取服务注册信息，而客户端服务的注册则推送到 registry 中。

为什么要设计多级缓存呢？

当大规模的服务注册和更新时，如果只是修改一个 ConcurrentHashMap 数据，那么会因为锁存在竞争，从而影响性能。而 Eureka 又是 AP 模型，只需要满足最终可用就可以了。所以在这里用到多级缓存来实现读写分离。

## 服务注册会使得读写缓存失效



```java
public void invalidate(Key... keys) {
    for (Key key : keys) {
        logger.debug("Invalidating the response cache key : {} {} {} {}, {}",
                     key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept());

        readWriteCacheMap.invalidate(key);
        Collection<Key> keysWithRegions = regionSpecificKeys.get(key);
        if (null != keysWithRegions && !keysWithRegions.isEmpty()) {
            for (Key keysWithRegion : keysWithRegions) {
                logger.debug("Invalidating the response cache key : {} {} {} {} {}",
                             key.getEntityType(), key.getName(), key.getVersion(), key.getType(), key.getEurekaAccept());
                readWriteCacheMap.invalidate(keysWithRegion);
            }
        }
    }
}
```

## 缓存同步

ResponseCacheImpl 的构造方法中，会启动一个定时任务，这个定时任务会定时检查写缓存中的数据变化，进行更新和同步。

```java
private TimerTask getCacheUpdateTask() {
    return new TimerTask() {
        @Override
        public void run() {
            logger.debug("Updating the client cache from response cache");
            for (Key key : readOnlyCacheMap.keySet()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Updating the client cache from response cache for key : {} {} {} {}",
                                 key.getEntityType(), key.getName(), key.getVersion(), key.getType());
                }
                try {
                    CurrentRequestVersion.set(key.getVersion());
                    Value cacheValue = readWriteCacheMap.get(key);
                    Value currentCacheValue = readOnlyCacheMap.get(key);
                    if (cacheValue != currentCacheValue) {
                        readOnlyCacheMap.put(key, cacheValue);
                    }
                } catch (Throwable th) {
                    logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), th);
                } finally {
                    CurrentRequestVersion.remove();
                }
            }
        }
    };
}
```



# 服务续约

所谓的服务续约，其实就是一种心跳检测机制，客户端会定期发送心跳来续约。

在客户端注册服务的时候，初始化了 `DiscoveryClient`，通过构造方法中的 initScheduledTasks() 方法创建心跳检测的定时任务。

## DiscoveryClient.initScheduledTasks()

```java
private void initScheduledTasks() {
    // 省略代码

    if (clientConfig.shouldRegisterWithEureka()) {

        // Heartbeat timer
        // 心跳检测
        heartbeatTask = new TimedSupervisorTask(
            "heartbeat",
            scheduler,
            heartbeatExecutor,
            renewalIntervalInSecs,
            TimeUnit.SECONDS,
            expBackOffBound,
            new HeartbeatThread()
        );
        scheduler.schedule(
            heartbeatTask,
            renewalIntervalInSecs, TimeUnit.SECONDS);

        // 省略代码
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```

## HeartbeatThread.run()

```java
private class HeartbeatThread implements Runnable {

    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```

## HeartbeatThread.renew()

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        // 请求服务端接口进行服务续约
        // eurekaTransport.registrationClient = SessionedEurekaHttpClient
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to send heartbeat!", appPathIdentifier, e);
        return false;
    }
}
```

## AbstractJerseyEurekaHttpClient.sendHeartBeat()

向服务端发起请求，进行服务续约操作。

```java
public EurekaHttpResponse<InstanceInfo> sendHeartBeat(String appName, String id, InstanceInfo info, InstanceStatus overriddenStatus) {
    String urlPath = "apps/" + appName + '/' + id;
    ClientResponse response = null;
    try {
        WebResource webResource = jerseyClient.resource(serviceUrl)
            .path(urlPath)
            .queryParam("status", info.getStatus().toString())
            .queryParam("lastDirtyTimestamp", info.getLastDirtyTimestamp().toString());
        if (overriddenStatus != null) {
            webResource = webResource.queryParam("overriddenstatus", overriddenStatus.name());
        }
        Builder requestBuilder = webResource.getRequestBuilder();
        addExtraHeaders(requestBuilder);
        response = requestBuilder.put(ClientResponse.class);
        EurekaHttpResponseBuilder<InstanceInfo> eurekaResponseBuilder = anEurekaHttpResponse(response.getStatus(), InstanceInfo.class).headers(headersOf(response));
        if (response.hasEntity() &&
            !HTML.equals(response.getType().getSubtype())) { //don't try and deserialize random html errors from the server
            eurekaResponseBuilder.entity(response.getEntity(InstanceInfo.class));
        }
        return eurekaResponseBuilder.build();
    } finally {
        if (logger.isDebugEnabled()) {
            logger.debug("Jersey HTTP PUT {}/{}; statusCode={}", serviceUrl, urlPath, response == null ? "N/A" : response.getStatus());
        }
        if (response != null) {
            response.close();
        }
    }
}
```



## InstanceResource.renewLease()

服务端主要就是修改

```java
@PUT
public Response renewLease(
    @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
    @QueryParam("overriddenstatus") String overriddenStatus,
    @QueryParam("status") String status,
    @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
    // 进行服务续约操作，就是更新 lastUpdateTimestamp
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

    // Not found in the registry, immediately ask for a register
    if (!isSuccess) {
        logger.warn("Not Found (Renew): {} - {}", app.getName(), id);
        return Response.status(Status.NOT_FOUND).build();
    }
    // Check if we need to sync based on dirty time stamp, the client
    // instance might have changed some value
    Response response;
    if (lastDirtyTimestamp != null && serverConfig.shouldSyncWhenTimestampDiffers()) {
        response = this.validateDirtyTimestamp(Long.valueOf(lastDirtyTimestamp), isFromReplicaNode);
        // Store the overridden status since the validation found out the node that replicates wins
        if (response.getStatus() == Response.Status.NOT_FOUND.getStatusCode()
            && (overriddenStatus != null)
            && !(InstanceStatus.UNKNOWN.name().equals(overriddenStatus))
            && isFromReplicaNode) {
            registry.storeOverriddenStatusIfRequired(app.getAppName(), id, InstanceStatus.valueOf(overriddenStatus));
        }
    } else {
        response = Response.ok().build();
    }
    logger.debug("Found (Renew): {} - {}; reply status={}", app.getName(), id, response.getStatus());
    return response;
}
```



# 服务发现

在客户端注册服务的时候，初始化了 `DiscoveryClient`，通过构造方法中的 fetchRegistry() 方法获得服务端上的地址列表。

## DiscoveryClient 构造方法

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                    Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
    // 省略代码
	if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }
    // 省略代码
}
```

## DiscoveryClient.fetchRegistry()

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();

    try {
        // If the delta is disabled or if it is the first time, get all
        // applications
        Applications applications = getApplications();
		// 判断全量拉取或者增量拉取
        if (clientConfig.shouldDisableDelta()
            || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
            || forceFullRegistryFetch
            || (applications == null)
            || (applications.getRegisteredApplications().size() == 0)
            || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        {
            logger.info("Disable delta property : {}", clientConfig.shouldDisableDelta());
            logger.info("Single vip registry refresh property : {}", clientConfig.getRegistryRefreshSingleVipAddress());
            logger.info("Force full registry fetch : {}", forceFullRegistryFetch);
            logger.info("Application is null : {}", (applications == null));
            logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
            logger.info("Application version is -1: {}", (applications.getVersion() == -1));
            // 全量拉取
            getAndStoreFullRegistry();
        } else {
            // 增量拉取
            getAndUpdateDelta(applications);
        }
        applications.setAppsHashCode(applications.getReconcileHashCode());
        logTotalInstances();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
        return false;
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }

    // Notify about cache refresh before updating the instance remote status
    //将本地缓存更新的事件广播给所有已注册的监听器，注意该方法已被CloudEurekaClient类重写
    onCacheRefreshed();
	//检查刚刚更新的缓存中，有来自Eureka server的服务列表，其中包含了当前应用的状态， 
    //当前实例的成员变量lastRemoteInstanceStatus，记录的是最后一次更新的当前应用状态， 
    //上述两种状态在updateInstanceRemoteStatus方法中作比较 ，如果不一致，就更新 lastRemoteInstanceStatus，并且广播对应的事件
    // Update remote status based on refreshed data held in the cache
    updateInstanceRemoteStatus();

    // registry was fetched successfully, so return true
    return true;
}
```



## DiscoveryClient.getAndStoreFullRegistry()

从 eureka server 端获取服务注册中心的地址信息，然后更新并设置到本地缓存 localRegionApps。

```java
private void getAndStoreFullRegistry() throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    logger.info("Getting all instance registry info from the eureka server");

    Applications apps = null;
    EurekaHttpResponse<Applications> httpResponse = clientConfig.getRegistryRefreshSingleVipAddress() == null
        ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
        : eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddress(), remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }
    logger.info("The response status is {}", httpResponse.getStatusCode());

    if (apps == null) {
        logger.error("The application is null for some reason. Not storing this information");
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(apps));
        logger.debug("Got full registry with apps hashcode {}", apps.getAppsHashCode());
    } else {
        logger.warn("Not updating applications as another thread is updating it already");
    }
}
```



## 服务端接受客户端查询请求

客户端拉取服务端信息有两种方式：全量、增量。

对于全量会调用 Eureka-Server 的 ApplicationsResource.getContainers() 方法，而增量会调用 ApplicationsResource.getContainerDifferential() 方法。



## ApplicationsResource.getContainers()

```java
@GET
public Response getContainers(@PathParam("version") String version,
                              @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                              @HeaderParam(HEADER_ACCEPT_ENCODING) String acceptEncoding,
                              @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT) String eurekaAccept,
                              @Context UriInfo uriInfo,
                              @Nullable @QueryParam("regions") String regionsStr) {

    boolean isRemoteRegionRequested = null != regionsStr && !regionsStr.isEmpty();
    String[] regions = null;
    if (!isRemoteRegionRequested) {
        EurekaMonitors.GET_ALL.increment();
    } else {
        regions = regionsStr.toLowerCase().split(",");
        Arrays.sort(regions); // So we don't have different caches for same regions queried in different order.
        EurekaMonitors.GET_ALL_WITH_REMOTE_REGIONS.increment();
    }

    // Check if the server allows the access to the registry. The server can
    // restrict access if it is not
    // ready to serve traffic depending on various reasons.
    // EurekaServer无法提供服务，返回403
    if (!registry.shouldAllowAccess(isRemoteRegionRequested)) {
        return Response.status(Status.FORBIDDEN).build();
    }
    CurrentRequestVersion.set(Version.toEnum(version));
    KeyType keyType = Key.KeyType.JSON;
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }

    Key cacheKey = new Key(Key.EntityType.Application,
                           ResponseCacheImpl.ALL_APPS,
                           keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
                          );

    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        // 去缓存中去数据
        response = Response.ok(responseCache.getGZIP(cacheKey))
            .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
            .header(HEADER_CONTENT_TYPE, returnMediaType)
            .build();
    } else {
        response = Response.ok(responseCache.get(cacheKey))
            .build();
    }
    CurrentRequestVersion.remove();
    return response;
}
```

## ResponseCacheImpl.getGZIP()

去缓存中读取数据

```java
public byte[] getGZIP(Key key) {
    Value payload = getValue(key, shouldUseReadOnlyResponseCache);
    if (payload == null) {
        return null;
    }
    return payload.getGzipped();
}
```

## ResponseCacheImpl.getValue()

```java
Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    try {
        if (useReadOnlyCache) {
            // 读缓存中获取
            final Value currentPayload = readOnlyCacheMap.get(key);
            if (currentPayload != null) {
                payload = currentPayload;
            } else {
                payload = readWriteCacheMap.get(key);
                readOnlyCacheMap.put(key, payload);
            }
        } else {
            payload = readWriteCacheMap.get(key);
        }
    } catch (Throwable t) {
        logger.error("Cannot get value for key : {}", key, t);
    }
    return payload;
}
```



# 服务下线

在 DiscoveryClient 中有个 shutdown() 方法上有 @PreDestroy 注解。在 springboot 关闭之前，会调用这个方法。

## DiscoveryClient.shutdown()

```java
@PreDestroy
@Override
public synchronized void shutdown() {
    if (isShutdown.compareAndSet(false, true)) {
        logger.info("Shutting down DiscoveryClient ...");

        if (statusChangeListener != null && applicationInfoManager != null) {
            applicationInfoManager.unregisterStatusChangeListener(
                statusChangeListener.getId());
        }

        cancelScheduledTasks();

        // If APPINFO was registered
        if (applicationInfoManager != null
            && clientConfig.shouldRegisterWithEureka()
            && clientConfig.shouldUnregisterOnShutdown()) {
            applicationInfoManager.setInstanceStatus(InstanceStatus.DOWN);
            // 取消注册
            unregister();
        }

        if (eurekaTransport != null) {
            eurekaTransport.shutdown();
        }

        heartbeatStalenessMonitor.shutdown();
        registryStalenessMonitor.shutdown();

        Monitors.unregisterObject(this);

        logger.info("Completed shut down of DiscoveryClient");
    }
}
```

## DiscoveryClient.unregister()

```java
void unregister() {
    // It can be null if shouldRegisterWithEureka == false
    if(eurekaTransport != null && eurekaTransport.registrationClient != null) {
        try {
            logger.info("Unregistering ...");
            // 向服务端发起下线通知
            EurekaHttpResponse<Void> httpResponse = eurekaTransport.registrationClient.cancel(instanceInfo.getAppName(), instanceInfo.getId());
            logger.info(PREFIX + "{} - deregister  status: {}", appPathIdentifier, httpResponse.getStatusCode());
        } catch (Exception e) {
            logger.error(PREFIX + "{} - de-registration failed{}", appPathIdentifier, e.getMessage(), e);
        }
    }
}
```



## InstanceResource.cancelLease()

服务下线，主要就是删除节点信息，并失效对应节点的缓存

```java
@DELETE
public Response cancelLease(
    @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication) {
    try {
        boolean isSuccess = registry.cancel(app.getName(), id,
                                            "true".equals(isReplication));

        if (isSuccess) {
            logger.debug("Found (Cancel): {} - {}", app.getName(), id);
            return Response.ok().build();
        } else {
            logger.info("Not Found (Cancel): {} - {}", app.getName(), id);
            return Response.status(Status.NOT_FOUND).build();
        }
    } catch (Throwable e) {
        logger.error("Error (cancel): {} - {}", app.getName(), id, e);
        return Response.serverError().build();
    }

}
```



# Eureka 自我保护机制

Eureka Server 在运行期间会去统计心跳失败比例在 15 分钟之内是否低于 85%，如果低于 85%，Euerka Server 会认为当前实例的客户端与自己的心跳连接出现了网络故障，那么 Eureka Server 会把这些实例保护起来，让这些实例不会过期导致删除。

这样做的目的是为了减少网络不稳定或者网络分区情况下，Eureka Server 将健康服务剔除下线的问题。

从这个机制中，最重要的如何计算在规定时间内的心跳比例，这里的心跳比例是指所有服务的心跳总比例，而不是单指某一服务。

核心的计算规则在 AbstractInstanceRegistry.updateRenewsPerMinThreshold() 方法中。

## AbstractInstanceRegistry.updateRenewsPerMinThreshold()

```java
// 每分钟最小续约数
protected volatile int numberOfRenewsPerMinThreshold;
// 预期每分钟收到续约的客户端数量，取决于注册到 eureka-server 的服务数量
protected volatile int expectedNumberOfClientsSendingRenews;

protected void updateRenewsPerMinThreshold() {
    // 自我保护的阈值 = 服务总数 * (60 / 客户端续约次数) * 自我保护续约百分比阀值因子
    // 自我保护的阈值 = serverCount * （60/30）* 0.85
    this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
                                                * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
                                                * serverConfig.getRenewalPercentThreshold());
}
```

根据自我保护机制的功能，想一想会有哪几个过程会对这个值进行更新操作

- eureka-server 启动会初始化这个值
- eureka-client 的注册、下线都会更新这个值
- 定时任务监听

## eureka-server 服务启动

EurekaServerInitializerConfiguration 实现了 SmartLifecycle 接口，Springboot 启动会执行 EurekaServerInitializerConfiguration.run()。具体过程参照 服务注册分析。 

### EurekaServerInitializerConfiguration.run()

```java
@Configuration(proxyBeanMethods = false)
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration implements WebMvcConfigurer {
    	@Override
	public void start() {
		new Thread(() -> {
			try {
				// TODO: is this class even needed now?
				eurekaServerBootstrap.contextInitialized(
						EurekaServerInitializerConfiguration.this.servletContext);
				log.info("Started Eureka Server");

				publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
				EurekaServerInitializerConfiguration.this.running = true;
				publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
			}
			catch (Exception ex) {
				// Help!
				log.error("Could not initialize Eureka servlet context", ex);
			}
		}).start();
	}
}
```



### EurekaServerBootstrap.contextInitialized()

```java
public void contextInitialized(ServletContext context) {
    try {
        initEurekaEnvironment();
        initEurekaServerContext();

        context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
    }
    catch (Throwable e) {
        log.error("Cannot bootstrap eureka server :", e);
        throw new RuntimeException("Cannot bootstrap eureka server :", e);
    }
}
```

### EurekaServerBootstrap.initEurekaServerContext()

```java
protected void initEurekaServerContext() throws Exception {
    // For backward compatibility
    JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                                                XStream.PRIORITY_VERY_HIGH);
    XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
                                               XStream.PRIORITY_VERY_HIGH);

    if (isAws(this.applicationInfoManager.getInfo())) {
        this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
                                               this.eurekaClientConfig, this.registry, this.applicationInfoManager);
        this.awsBinder.start();
    }

    EurekaServerContextHolder.initialize(this.serverContext);

    log.info("Initialized server context");

    // Copy registry from neighboring eureka node
    int registryCount = this.registry.syncUp();
    this.registry.openForTraffic(this.applicationInfoManager, registryCount);

    // Register all monitoring statistics.
    EurekaMonitors.registerAllStats();
}
```

### PeerAwareInstanceRegistryImpl.openForTraffic()

```java
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    this.expectedNumberOfClientsSendingRenews = count;
    // 更新自我保护机制的阈值
    updateRenewsPerMinThreshold();
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    super.postInit();
}
```



## eureka-client 服务注册、下线

### PeerAwareInstanceRegistryImpl.cancel()

当服务提供者主动下线时，表示这个时候Eureka-Server要剔除这个服务提供者的地址，同时也代表这 这个心跳续约的阈值要发生变化。所以在 PeerAwareInstanceRegistryImpl.cancel 中可以看到数据的更新

```java
protected boolean internalCancel(String appName, String id, boolean isReplication) {
    
	// 省略代码
    synchronized (lock) {
        if (this.expectedNumberOfClientsSendingRenews > 0) {
            // Since the client wants to cancel it, reduce the number of clients to send renews.
            this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews - 1;
            updateRenewsPerMinThreshold();
        }
    }

    return true;
}
```



### PeerAwareInstanceRegistryImpl.register()

当有新的服务提供者注册到 eureka-server上时，需要增加续约的客户端数量，所以在register方法中会进行处理

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    // 省略代码
    // The lease does not exist and hence it is a new registration
    synchronized (lock) {
        if (this.expectedNumberOfClientsSendingRenews > 0) {
            // Since the client wants to register it, increase the number of clients sending renews
            this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
            updateRenewsPerMinThreshold();
        }
    }
    // 省略代码
}
```





## 定时任务监听阈值变化

### EurekaServerAutoConfiguration.eurekaServerContext()

```java
@Bean
@ConditionalOnMissingBean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
                                               PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
    return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
                                          registry, peerEurekaNodes, this.applicationInfoManager);
}
```



### DefaultEurekaServerContext.initialize()

构造 DefaultEurekaServerContext 之后会执行 PostConstruct 注解内的方法

```java
@Inject
public DefaultEurekaServerContext(EurekaServerConfig serverConfig,
                                  ServerCodecs serverCodecs,
                                  PeerAwareInstanceRegistry registry,
                                  PeerEurekaNodes peerEurekaNodes,
                                  ApplicationInfoManager applicationInfoManager) {
    this.serverConfig = serverConfig;
    this.serverCodecs = serverCodecs;
    this.registry = registry;
    this.peerEurekaNodes = peerEurekaNodes;
    this.applicationInfoManager = applicationInfoManager;
}

@PostConstruct
@Override
public void initialize() {
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}
```



### PeerAwareInstanceRegistryImpl.init()

```java
public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    initializedResponseCache();
    scheduleRenewalThresholdUpdateTask();
    initRemoteRegionRegistry();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", e);
    }
}
```



### PeerAwareInstanceRegistryImpl.scheduleRenewalThresholdUpdateTask()

启动一个定时任务，15分钟定时轮循一次

```java
private void scheduleRenewalThresholdUpdateTask() {
    timer.schedule(new TimerTask() {
        @Override
        public void run() {
            updateRenewalThreshold();
        }
    }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
                   serverConfig.getRenewalThresholdUpdateIntervalMs());
}
```



### PeerAwareInstanceRegistryImpl.updateRenewalThreshold()

```java
private void updateRenewalThreshold() {
    try {
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        synchronized (lock) {
            // Update threshold only if the threshold is greater than the
            // current expected threshold or if self preservation is disabled.
            if ((count) > (serverConfig.getRenewalPercentThreshold() * expectedNumberOfClientsSendingRenews)
                || (!this.isSelfPreservationModeEnabled())) {
                this.expectedNumberOfClientsSendingRenews = count;
                updateRenewsPerMinThreshold();
            }
        }
        logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
    } catch (Throwable e) {
        logger.error("Cannot update renewal threshold", e);
    }
}
```



## 自我保护机制的触发任务

上面都是在更新自我保护机制的阈值，那么哪里会去检查这个阈值来触发自我保护机制呢？

在 eureka-server 启动的时候，会调用 PeerAwareInstanceRegistryImpl.openForTraffic() 方法，而这个方法里面调用 super.postInit() 方法来创建自我保护机制的定时任务。

### PeerAwareInstanceRegistryImpl.openForTraffic() 

```java
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    this.expectedNumberOfClientsSendingRenews = count;
    updateRenewsPerMinThreshold();
    logger.info("Got {} instances from neighboring DS node", count);
    logger.info("Renew threshold is: {}", numberOfRenewsPerMinThreshold);
    this.startupTime = System.currentTimeMillis();
    if (count > 0) {
        this.peerInstancesTransferEmptyOnStartup = false;
    }
    DataCenterInfo.Name selfName = applicationInfoManager.getInfo().getDataCenterInfo().getName();
    boolean isAws = Name.Amazon == selfName;
    if (isAws && serverConfig.shouldPrimeAwsReplicaConnections()) {
        logger.info("Priming AWS connections for all replicas..");
        primeAwsReplicas(applicationInfoManager);
    }
    logger.info("Changing status to UP");
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    // 创建自我保护机制的定时任务
    super.postInit();
}
```

### AbstractInstanceRegistry.postInit()

```java
protected void postInit() {
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    evictionTaskRef.set(new EvictionTask());
    evictionTimer.schedule(evictionTaskRef.get(),
                           serverConfig.getEvictionIntervalTimerInMs(),
                           serverConfig.getEvictionIntervalTimerInMs());
}
```

EvictionTask.run()

```java
public void run() {
    try {
        long compensationTimeMs = getCompensationTimeMs();
        logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
        evict(compensationTimeMs);
    } catch (Throwable e) {
        logger.error("Could not run the evict task", e);
    }
}
```



### AbstractInstanceRegistry.evict()

```java
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");
	// 是否需要开启自我保护机制，如果需要，那么直接RETURE， 不需要继续往下执行了
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }
    
	//这下面主要是做服务自动下线的操作的
    // We collect first all expired items, to evict them in random order. For large eviction sets,
    // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
    // the impact should be evenly distributed across all applications.
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }

    // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
    // triggering self-preservation. Without that we would wipe out full registry.
    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;

    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    if (toEvict > 0) {
        logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

        Random random = new Random(System.currentTimeMillis());
        for (int i = 0; i < toEvict; i++) {
            // Pick a random item (Knuth shuffle algorithm)
            int next = i + random.nextInt(expiredLeases.size() - i);
            Collections.swap(expiredLeases, i, next);
            Lease<InstanceInfo> lease = expiredLeases.get(i);

            String appName = lease.getHolder().getAppName();
            String id = lease.getHolder().getId();
            EXPIRED.increment();
            logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
            internalCancel(appName, id, false);
        }
    }
}
```

### PeerAwareInstanceRegistryImpl.isLeaseExpirationEnabled()

```java
public boolean isLeaseExpirationEnabled() {
    //是否开启了自我保护机制，如果没有，则跳过，默认是开启 
    // 计算是否需要开启自我保护，判断最后一分钟收到的续约数量是否大于 numberOfRenewsPerMinThreshold
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```












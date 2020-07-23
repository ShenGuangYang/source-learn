# ribbon

在分布式的架构下，一个服务调用其他服务集群节点，那实现请求转发、负载均衡呢？

**Spring Cloud Ribbon 就是实现负载均衡的组件。核心思想是通过对 RestTemplate 进行拦截，然后选择配置中的路由地址，url 重构之后进行服务调用。**

# 使用 demo 实例

比如电商系统中，用户要查询自己的订单。业务逻辑拆分成用户系统与订单系统，用户系统要远程调用订单系统的接口。

## order-service

新建 spring boot 项目

**pom.xml**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-web</artifactId>
</dependency>
```

**properties**

```properties
spring.application.name=order-service
```

**代码**

```java
@RestController
public class OrderController {
    @GetMapping("orders")
    public String orders() {
        System.out.println("port: " + port);
        return "All orders";
    }
}
```

## user-service

新建 spring boot 项目

**pom.xml**

```xml
 <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
    <version>2.2.3.RELEASE</version>
</dependency>
```

**properties**

```properties
server.port=8088
spring.application.name=user-service
# order-service 启动的服务地址
order-service.ribbon.listOfServers=\
  localhost:8080,localhost:8081,localhost:8082
```

**代码**

```java
@RestController
public class UserController {

    @Autowired
    private RestTemplate restTemplate;

    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @GetMapping("user")
    public String user() {
        return restTemplate.getForObject("http://order-service/orders", String.class);
    }

}
```

# 源码分析

从使用 Ribbon 组件来看，主要使用 spring boot AutoConfiguration 功能，扫描 `@LoadBalanced` 注解，并对添加注解的 RestTemplate 设置拦截器，调用订单接口时通过拦截器对请求 url 进行重构。



## AutoConfiguration 初始化  Bean

```java
public class LoadBalancerAutoConfiguration {
    // 注入所有带有 LoadBalanced 的 RestTemplate
	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();
	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();
    // 对 restTemplate 进行包装
	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
			for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
				for (RestTemplateCustomizer customizer : customizers) {
					customizer.customize(restTemplate);
				}
			}
		});
	}
	// 初始化 LoadBalancerRequestFactory
	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
	}
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		// 初始化拦截器
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}
        // 对 RestTemplate 添加拦截器
		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
				List<ClientHttpRequestInterceptor> list = new ArrayList<>(
						restTemplate.getInterceptors());
				list.add(loadBalancerInterceptor);
				restTemplate.setInterceptors(list);
			};
		}

	}
	// 省略部分代码
}
```

## restTemplate 请求对 url 进行重构

`RestTemplate.getForObject()` 内部链路调用（不细致介绍了），最后肯定会走到我们设置的`LoadBalancerInterceptor.intercept()`  的方法逻辑中

### LoadBalancerInterceptor.intercept()

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {

	// 省略部分代码

	@Override
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
        // originalUri = http://order-service/orders
		final URI originalUri = request.getURI();
        // serviceName = order-service
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null,
				"Request URI does not contain a valid hostname: " + originalUri);
        
        // loadBalancer就是初始bean的设置 LoadBalancerClient
		return this.loadBalancer.execute(serviceName,
				this.requestFactory.createRequest(request, body, execution));
	}

}
```

### LoadBalancerClient.execute()

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
    throws IOException {
    return execute(serviceId, request, null);
}

public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
    throws IOException {
    // 获取 负载均衡处理器 下面会重点分析
    // TODO 下面会重点分析
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    // 通过负载均衡处理器获取 Server 节点
    // TODO 下面会重点分析
    Server server = getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    // 封装成 ribbon server
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
                                                 isSecure(server, serviceId),
                                                 serverIntrospector(serviceId).getMetadata(server));
    return execute(serviceId, ribbonServer, request);
}

public <T> T execute(String serviceId, ServiceInstance serviceInstance,
                     LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if (serviceInstance instanceof RibbonServer) {
        server = ((RibbonServer) serviceInstance).getServer();
    }
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }

    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);
    // 统计相关
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

    try {
        // 执行请求
        // 这里的request 是通过 LoadBalancerRequestFactory.createRequest() 创建出来的
        // 所以执行的 LoadBalancerRequestFactory.createRequest()
        T returnVal = request.apply(serviceInstance);
        statsRecorder.recordStats(returnVal);
        return returnVal;
    }
    // catch IOException and rethrow so RestTemplate behaves correctly
    catch (IOException ex) {
        statsRecorder.recordStats(ex);
        throw ex;
    }
    catch (Exception ex) {
        statsRecorder.recordStats(ex);
        ReflectionUtils.rethrowRuntimeException(ex);
    }
    return null;
}
```



### LoadBalancerRequestFactory.createRequest()

```java
public class LoadBalancerRequestFactory {

	// 省略部分代码

	public LoadBalancerRequest<ClientHttpResponse> createRequest(
			final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) {
		return instance -> {
            // 将请求包装成了 ServiceRequestWrapper
			HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance,
					this.loadBalancer);
            // autoconfiguration 初始化的时候 transformers 是一个空集合
			if (this.transformers != null) {
				for (LoadBalancerRequestTransformer transformer : this.transformers) {
					serviceRequest = transformer.transformRequest(serviceRequest,
							instance);
				}
			}
            // execution 传递的 InterceptingRequestExecution
			return execution.execute(serviceRequest, body);
		};
	}

}
```

### InterceptingRequestExecution.execute()

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
    if (this.iterator.hasNext()) {
        ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
        return nextInterceptor.intercept(request, body, this);
    }
    else {
        HttpMethod method = request.getMethod();
        Assert.state(method != null, "No standard HTTP method");
        // 重构 url
        // 这里的 request 是包装的 ServiceRequestWrapper
        // 这里的 requestFactory 是RestTemplate.doExecute() 里面的 
        // ClientHttpRequest request = createRequest(url, method) 创建的
        // 默认 delegate = SimpleClientHttpRequestFactory
        ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
        request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
        if (body.length > 0) {
            if (delegate instanceof StreamingHttpOutputMessage) {
                StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
            }
            else {
                StreamUtils.copy(body, delegate.getBody());
            }
        }
        // SimpleClientHttpRequestFactory.execute()
        return delegate.execute();
    }
}
```

### ServiceRequestWrapper.getURI()

```java
public URI getURI() {
    // 重构Url 
    URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
    return uri;
}
```

### RibbonLoadBalancerClient.reconstructURI()

```java
public URI reconstructURI(ServiceInstance instance, URI original) {
    Assert.notNull(instance, "instance can not be null");
    String serviceId = instance.getServiceId();
    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);

    URI uri;
    Server server;
    if (instance instanceof RibbonServer) {
        RibbonServer ribbonServer = (RibbonServer) instance;
        // localhost:8080
        server = ribbonServer.getServer();
        // http://order-service/orders
        uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
    }
    else {
        server = new Server(instance.getScheme(), instance.getHost(),
                            instance.getPort());
        IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
        ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
        uri = updateToSecureConnectionIfNeeded(original, clientConfig,
                                               serverIntrospector, server);
    }
    // 替换成正常的能访问的 URI: http://localhost:8080/orders
    return context.reconstructURIWithServer(server, uri);
}
```

### 选择对应的 http 客户端来执行请求

我们回到 `InterceptingRequestExecution.execute()` 的逻辑中，我们通过 `ServiceRequestWrapper.getURI` 获得重构后的 url，然后 `delegate.execute();` 来请求对应的接口。

这里的 `delegate` 是 RestTemplate 默认使用的 `SimpleClientHttpRequestFactory`。这里的 `execute()` 是模板方法的实现。所以执行父类的逻辑 `AbstractClientHttpRequest.execute()`。

### AbstractClientHttpRequest.execute()

```java
public final ClientHttpResponse execute() throws IOException {
    assertNotExecuted();
    // 没有子类实现，调用父类实现
    ClientHttpResponse result = executeInternal(this.headers);
    this.executed = true;
    return result;
}
```

### AbstractBufferingClientHttpRequest.executeInternal()

```java
protected ClientHttpResponse executeInternal(HttpHeaders headers) throws IOException {
    byte[] bytes = this.bufferedOutput.toByteArray();
    if (headers.getContentLength() < 0) {
        headers.setContentLength(bytes.length);
    }
    // 调用子类实现
    ClientHttpResponse result = executeInternal(headers, bytes);
    this.bufferedOutput = new ByteArrayOutputStream(0);
    return result;
}
```

### SimpleBufferingClientHttpRequest.executeInternal()

执行到这里就是真正去发起接口请求了。大功告成。 :tada::tada::tada::tada::tada::tada::tada::tada:

```java
protected ClientHttpResponse executeInternal(HttpHeaders headers, byte[] bufferedOutput) throws IOException {
    addHeaders(this.connection, headers);
    // JDK <1.8 doesn't support getOutputStream with HTTP DELETE
    if (getMethod() == HttpMethod.DELETE && bufferedOutput.length == 0) {
        this.connection.setDoOutput(false);
    }
    if (this.connection.getDoOutput() && this.outputStreaming) {
        this.connection.setFixedLengthStreamingMode(bufferedOutput.length);
    }
    this.connection.connect();
    if (this.connection.getDoOutput()) {
        FileCopyUtils.copy(bufferedOutput, this.connection.getOutputStream());
    }
    else {
        // Immediately trigger the request in a no-output scenario as well
        this.connection.getResponseCode();
    }
    return new SimpleClientHttpResponse(this.connection);
}
```

## 负载均衡算法

在上面执行代码中，在 `LoadBalancerClinet.execute()` 方法中有如下代码：

```java
ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
Server server = getServer(loadBalancer, hint);
```

是用来选择负载均衡器来选择对应的服务节点信息。那来看看是如何服务节点的。

### RibbonLoadBalancerClient.getLoadBalancer()

```java
protected ILoadBalancer getLoadBalancer(String serviceId) {
    // clientFactory 是 SpringClientFactory
    return this.clientFactory.getLoadBalancer(serviceId);
}
```

### SpringClientFactory.getLoadBalancer()

```java
public ILoadBalancer getLoadBalancer(String name) {
    return getInstance(name, ILoadBalancer.class);
}

public <C> C getInstance(String name, Class<C> type) {
    // 通过 spring ioc 的模式创建对象
    // 加载 RibbonClientConfiguration 的配置内容 LoadBalancer = ZoneAwareLoadBalancer
    // 因为 ZoneAwareLoadBalancer 调用父类的 DynamicServerListLoadBalancer 构造
    C instance = super.getInstance(name, type);
    // 所以获取到实例类型为 ZoneAwareLoadBalancer  
    if (instance != null) {
        // 返回 ZoneAwareLoadBalancer
        return instance;
    }
    // 这里不执行
    IClientConfig config = getInstance(name, IClientConfig.class);
    return instantiateWithConfig(getContext(name), type, config);
}
```

### ZoneAwareLoadBalancer 构造

```java
public ZoneAwareLoadBalancer(IClientConfig clientConfig, IRule rule,
                             IPing ping, ServerList<T> serverList, ServerListFilter<T> filter,
                             ServerListUpdater serverListUpdater) {
    // 调用父类的构造方法
    super(clientConfig, rule, ping, serverList, filter, serverListUpdater);
}
```

### DynamicServerListLoadBalancer 构造

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                     ServerList<T> serverList, ServerListFilter<T> filter,
                                     ServerListUpdater serverListUpdater) {
    // 调用父类构造
    super(clientConfig, rule, ping);
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    // TODO 下面重点会分析
    restOfInit(clientConfig);
}
```

### BaseLoadBalancer 构造

```java
public BaseLoadBalancer(IClientConfig config, IRule rule, IPing ping) {
    initWithConfig(config, rule, ping, createLoadBalancerStatsFromConfig(config));
}

void initWithConfig(IClientConfig clientConfig, IRule rule, IPing ping, LoadBalancerStats stats) {
    this.config = clientConfig;
    String clientName = clientConfig.getClientName();
    this.name = clientName;
    int pingIntervalTime = Integer.parseInt(""
                                            + clientConfig.getProperty(
                                                CommonClientConfigKey.NFLoadBalancerPingInterval,
                                                Integer.parseInt("30")));
    int maxTotalPingTime = Integer.parseInt(""
                                            + clientConfig.getProperty(
                                                CommonClientConfigKey.NFLoadBalancerMaxTotalPingTime,
                                                Integer.parseInt("2")));

    // 设置心跳检查定时任务 
    // TODO 下面会介绍
    setPingInterval(pingIntervalTime);
    setMaxTotalPingTime(maxTotalPingTime);
    // cross associate with each other
    // i.e. Rule,Ping meet your container LB
    // LB, these are your Ping and Rule guys ...
    // 上面注释说的很清楚
    // 使得 LoadBalancer、Rule、Ping 相互关联
    setRule(rule);
    setPing(ping);
    setLoadBalancerStats(stats);
    // 下面 ZoneAvoidanceRule.choose() 会使用到
    rule.setLoadBalancer(this);
    
    // 省略部分代码

}
```

### NamedContextFactory.getInstance()

```java
public <T> T getInstance(String name, Class<T> type) {
    AnnotationConfigApplicationContext context = getContext(name);
    if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
                                                            type).length > 0) {
        return context.getBean(type);
    }
    return null;
}
// 根据名称获取对象，不存在就创建
protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) {
        synchronized (this.contexts) {
            if (!this.contexts.containsKey(name)) {
                this.contexts.put(name, createContext(name));
            }
        }
    }
    return this.contexts.get(name);
}

// spring ioc 的模式创建容器
protected AnnotationConfigApplicationContext createContext(String name) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
             .getConfiguration()) {
            context.register(configuration);
        }
    }
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    context.register(PropertyPlaceholderAutoConfiguration.class,
                     this.defaultConfigType);
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
        this.propertySourceName,
        Collections.<String, Object>singletonMap(this.propertyName, name)));
    if (this.parent != null) {
        // Uses Environment from parent as well as beans
        context.setParent(this.parent);
        // jdk11 issue
        // https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
        context.setClassLoader(this.parent.getClassLoader());
    }
    context.setDisplayName(generateDisplayName(name));
    context.refresh();
    return context;
}
```

### RibbonLoadBalancerClient.getServer()

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // Use 'default' on a null hint, or just pass it on?
    // 这里的 loadBalancer 是 ZoneAwareLoadBalancer
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

### ZoneAwareLoadBalancer.chooseServer()

```java
public Server chooseServer(Object key) {
    // demo 只有一个区域，执行这个if内的内容，正常多区域部署会执行下面内容
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        }
        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        }
        // 根据 triggeringLoad、triggeringBlackoutPercentage 计算出可用区
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        // 从可用区里随机选择一个区域（zone里面机器越多，被选中概率越大）
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                server = zoneLoadBalancer.chooseServer(key);
            }
        }
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    }
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    }
}
```


### BaseLoadBalancer.chooseServer()

```java
public Server chooseServer(Object key) {
    // 应该是统计数量相关的
    if (counter == null) {
        counter = createCounter();
    }
    // 访问次数 + 1
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            // 选择对应的负载均衡规则，默认的是 RoundRobinRule
            // 但是在 RibbonClientConfiguration 配置中改成了 ZoneAvoidanceRule
            // 另外这个 ZoneAvoidanceRule.choose() 没有相关实现
            // 所以调用父类的 PredicateBasedRule.choose()
            return rule.choose(key);
        } catch (Exception e) {
            logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
            return null;
        }
    }
}
```

### PredicateBasedRule.choose()

```java
public Server choose(Object key) {
    ILoadBalancer lb = getLoadBalancer();
    // RibbonClientConfiguration 中初始化 ribbonRule 中设置
    // getPredicate() == CompositePredicate 
    // 照理说会执行 CompositePredicate.chooseRoundRobinAfterFiltering()
    // 但是 CompositePredicate 中没有实现，调用父类中的 chooseRoundRobinAfterFiltering()
    Optional<Server> server = getPredicate().chooseRoundRobinAfterFiltering(lb.getAllServers(), key);
    if (server.isPresent()) {
        return server.get();
    } else {
        return null;
    }       
}
```

### AbstractServerPredicate.chooseRoundRobinAfterFiltering()

```java
public Optional<Server> chooseRoundRobinAfterFiltering(List<Server> servers, Object loadBalancerKey) {
    // 执行子类中的 CompositePredicate.getEligibleServers()
    List<Server> eligible = getEligibleServers(servers, loadBalancerKey);
    if (eligible.size() == 0) {
        return Optional.absent();
    }
    // 从返回的服务节点中循环获得一个节点
    return Optional.of(eligible.get(incrementAndGetModulo(eligible.size())));
}
```

### CompositePredicate.getEligibleServers()

```java
public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
    // 又调用父类中的 getEligibleServers
    // 主要是获取到正常的服务节点
    List<Server> result = super.getEligibleServers(servers, loadBalancerKey);
    Iterator<AbstractServerPredicate> i = fallbacks.iterator();
    // 不执行
    while (!(result.size() >= minimalFilteredServers && result.size() > (int) (servers.size() * minimalFilteredPercentage))
           && i.hasNext()) {
        AbstractServerPredicate predicate = i.next();
        result = predicate.getEligibleServers(servers, loadBalancerKey);
    }
    return result;
}
```

### AbstractServerPredicate.getEligibleServers()

```java
public List<Server> getEligibleServers(List<Server> servers, Object loadBalancerKey) {
    if (loadBalancerKey == null) {
        return ImmutableList.copyOf(Iterables.filter(servers, this.getServerOnlyPredicate()));            
    } else {
        List<Server> results = Lists.newArrayList();
        for (Server server: servers) {
            // 调用 ZoneAvoidancePredicate.apply()
            // 添加存活的服务节点
            if (this.apply(new PredicateKey(loadBalancerKey, server))) {
                results.add(server);
            }
        }
        return results;            
    }
}
```

### ZoneAvoidancePredicate.apply()

```java
public boolean apply(@Nullable PredicateKey input) {
    if (!ENABLED.get()) {
        return true;
    }
    String serverZone = input.getServer().getZone();
    if (serverZone == null) {
        // there is no zone information from the server, we do not want to filter
        // out this server
        return true;
    }
    LoadBalancerStats lbStats = getLBStats();
    if (lbStats == null) {
        // no stats available, do not filter
        return true;
    }
    // 有服务存活
    if (lbStats.getAvailableZones().size() <= 1) {
        // only one zone is available, do not filter
        return true;
    }
    // 省略部分代码
} 
```

### AbstractServerPredicate.incrementAndGetModulo()

从 `AbstractServerPredicate.getEligibleServers(servers, loadBalancerKey);` 获取到所有的服务节点之后，那就应该选择其中的一个节点来执行了。选择代码如下：

```java
 /**
  * Referenced from RoundRobinRule
  * Inspired by the implementation of {@link AtomicInteger#incrementAndGet()}.
  *
  */
// 注释写的很明白，引用了循环规则中
private int incrementAndGetModulo(int modulo) {
    for (;;) {
        int current = nextIndex.get();
        int next = (current + 1) % modulo;
        if (nextIndex.compareAndSet(current, next) && current < modulo)
            return current;
    }
}
```

### 总结

Ribbon 默认使用循环负载均衡算法。

Ribbon 默认的负载均衡器是ZoneAwareLoadBalancer（区域亲和负载均衡器），通过配置计算可用区的机器属性，来选择同区域的路由。

可以使用自定义配置属性来更改或者启动的时候注入 Bean

```yaml
order-service:
  ribbon:
  	NFLoadBalancerClassName: com.study.MyLoadBalancer
    NFLoadBalancerRuleClassName: com.study.MyRule
```

配置可参照 spring cloud 官网 中的 [7.2 Customizing the Ribbon Client](https://cloud.spring.io/spring-cloud-static/spring-cloud-netflix/2.2.0.RELEASE/reference/html/#customizing-the-ribbon-client) 章说明

## 心跳检测

服务节点是我们配置在配置文件中的信息，那有可能服务会下线，Ribbon 要是没有心跳检测的话，那就会访问到下线的路由上，导致程序出错。那么什么时候进行心跳检测的呢？

Ribbon 默认配置的心跳检测是 `DummyPing`。是一个假的心跳，默认不检测。

在上面 `BaseLoadBalancer 构造方法` 中有一行代码，是设置心跳检测定时任务的。

### BaseLoadBalancer 构造

```java
public BaseLoadBalancer(IClientConfig config, IRule rule, IPing ping) {
    initWithConfig(config, rule, ping, createLoadBalancerStatsFromConfig(config));
}

void initWithConfig(IClientConfig clientConfig, IRule rule, IPing ping, LoadBalancerStats stats) {
	// 省略部分代码
    // 设置 Ping 定时任务
    setPingInterval(pingIntervalTime);
	// 省略部分代码
}
```

### BaseLoadBalancer.setPingInterval()

```java
public void setPingInterval(int pingIntervalSeconds) {
    if (pingIntervalSeconds < 1) {
        return;
    }

    this.pingIntervalSeconds = pingIntervalSeconds;
    if (logger.isDebugEnabled()) {
        logger.debug("LoadBalancer [{}]:  pingIntervalSeconds set to {}",
                     name, this.pingIntervalSeconds);
    }
    setupPingTask(); // since ping data changed
}
```

### BaseLoadBalancer.setupPingTask()

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
    // 启动一个PingTask， 每个 30s 执行一次
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000);
    forceQuickPing();
}
```

### PingTask.run()

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



### Pinger.runPinger()

```java
public void runPinger() throws Exception {
    if (!pingInProgress.compareAndSet(false, true)) { 
        return; // Ping in progress - nothing to do
    }

    // we are "in" - we get to Ping

    Server[] allServers = null;
    boolean[] results = null;

    Lock allLock = null;
    Lock upLock = null;

    try {
        /*
         * The readLock should be free unless an addServer operation is
         * going on...
          */
        allLock = allServerLock.readLock();
        allLock.lock();
        allServers = allServerList.toArray(new Server[allServerList.size()]);
        allLock.unlock();

        int numCandidates = allServers.length;
        // 调用设置的心跳策略来进行检测
        // 获得存活的节点
        // pingerStrategy = SerialPingStrategy
        results = pingerStrategy.pingServers(ping, allServers);

        final List<Server> newUpList = new ArrayList<Server>();
        final List<Server> changedServers = new ArrayList<Server>();

        // 循环所有失效节点
        for (int i = 0; i < numCandidates; i++) {
            boolean isAlive = results[i];
            Server svr = allServers[i];
            boolean oldIsAlive = svr.isAlive();

            svr.setAlive(isAlive);

            if (oldIsAlive != isAlive) {
                changedServers.add(svr);
                logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
                             name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
            }

            if (isAlive) {
                newUpList.add(svr);
            }
        }
        upLock = upServerLock.writeLock();
        upLock.lock();
        // 重新设置更新后的节点
        upServerList = newUpList;
        upLock.unlock();
		// 通知服务节点监听器
        notifyServerStatusChangeListener(changedServers);
    } finally {
        pingInProgress.set(false);
    }
}
```

### SerialPingStrategy.pingServers()

```java
public boolean[] pingServers(IPing ping, Server[] servers) {
    int numCandidates = servers.length;
    boolean[] results = new boolean[numCandidates];
    logger.debug("LoadBalancer:  PingTask executing [{}] servers configured", numCandidates);
    for (int i = 0; i < numCandidates; i++) {
        results[i] = false; /* Default answer is DEAD. */
        try {
            if (ping != null) {
                // 调用具体的心跳检测逻辑
                results[i] = ping.isAlive(servers[i]);
            }
        } catch (Exception e) {
            logger.error("Exception while pinging Server: '{}'", servers[i], e);
        }
    }
    return results;
}
```

## 服务节点列表

我们从服务列表节中选择对应的节点来负载均衡，通过心跳来剔除下线节点，那么服务列表是怎么维护的呢？

上述分析中，在 `ZoneAwareLoadBalancer` 构造的时候调用了父类 `DynamicServerListBalancer` 构造，这个 `DynamicServerListBalancer` 中执行了一个 `restOfInit()` 方法来初始化 serviceList 信息。

### DynamicServerListLoadBalancer 构造

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                     ServerList<T> serverList, ServerListFilter<T> filter,
                                     ServerListUpdater serverListUpdater) {
    super(clientConfig, rule, ping);
    this.serverListImpl = serverList;
    this.filter = filter;
    this.serverListUpdater = serverListUpdater;
    if (filter instanceof AbstractServerListFilter) {
        ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
    }
    restOfInit(clientConfig);
}
```

### DynamicServerListLoadBalancer.restOfInit()

```java
void restOfInit(IClientConfig clientConfig) {
    boolean primeConnection = this.isEnablePrimingConnections();
    // turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
    this.setEnablePrimingConnections(false);
    // 启动监听节点
    enableAndInitLearnNewServersFeature();
	// 更新服务节点
    updateListOfServers();
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections()
            .primeConnections(getReachableServers());
    }
    this.setEnablePrimingConnections(primeConnection);
    LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
}
```

### DynamicServerListLoadBalancer.enableAndInitLearnNewServersFeature()

```java
public void enableAndInitLearnNewServersFeature() {
    LOGGER.info("Using serverListUpdater {}", serverListUpdater.getClass().getSimpleName());
    // serverListUpdater = PollingServerListUpdater
    serverListUpdater.start(updateAction);
}
```

### PollingServerListUpdater.start()

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
                    // 执行的 updateAction 中的方法
                    // updateAction是一个匿名内部类
                    // 主要执行 DynamicServerListLoadBalancer.updateListOfServers() 方法
                    updateAction.doUpdate();
                    lastUpdated = System.currentTimeMillis();
                } catch (Exception e) {
                    logger.warn("Failed one update cycle", e);
                }
            }
        };
		// 启动定时任务
        scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
            wrapperRunnable,
            initialDelayMs,
            refreshIntervalMs,
            TimeUnit.MILLISECONDS
        );
    } else {
        logger.info("Already active, no-op");
    }
}
```

### DynamicServerListLoadBalancer.updateListOfServers()

```java
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        // 获得更新的服务节点列表
        // serverListImpl 是 RibbonClientConfiguration 中默认配置
        // serverListImpl = ConfigurationBasedServerList
        // 也可以配置成其他的，比如 eureka 的实现
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                     getIdentifier(), servers);

        if (filter != null) {
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                         getIdentifier(), servers);
        }
    }
    // 更新所有的服务节点列表
    updateAllServerList(servers);
}
```

### ConfigurationBasedServerList.getUpdatedListOfServers()

```java
public List<Server> getUpdatedListOfServers() {
    // 获得配置文件中的内容
    String listOfServers = clientConfig.get(CommonClientConfigKey.ListOfServers);
    // 返回服务节点集合
    return derive(listOfServers);
}

protected List<Server> derive(String value) {
    List<Server> list = Lists.newArrayList();
    if (!Strings.isNullOrEmpty(value)) {
        for (String s: value.split(",")) {
            list.add(new Server(s.trim()));
        }
    }
    return list;
}
```

### DynamicServerListLoadBalancer.updateAllServerList()

```java
protected void updateAllServerList(List<T> ls) {
    // other threads might be doing this - in which case, we pass
    // CAS 防止其他线程竞争
    if (serverListUpdateInProgress.compareAndSet(false, true)) {
        try {
            for (T s : ls) {
                s.setAlive(true); // set so that clients can start using these
                // servers right away instead
                // of having to wait out the ping cycle.
            }
            // 设置服务节点
            setServersList(ls);
            super.forceQuickPing();
        } finally {
            serverListUpdateInProgress.set(false);
        }
    }
}
```

### DynamicServerListLoadBalancer.setServersList()

```java
public void setServersList(List lsrv) {
    // 执行父类 BaseLoadBalancer 中的方法
    // 把父类中的 allServerList、upServerList 更新
    super.setServersList(lsrv);
    List<T> serverList = (List<T>) lsrv;
    Map<String, List<Server>> serversInZones = new HashMap<String, List<Server>>();
    for (Server server : serverList) {
        // make sure ServerStats is created to avoid creating them on hot
        // path
        getLoadBalancerStats().getSingleServerStat(server);
        String zone = server.getZone();
        if (zone != null) {
            zone = zone.toLowerCase();
            List<Server> servers = serversInZones.get(zone);
            if (servers == null) {
                servers = new ArrayList<Server>();
                serversInZones.put(zone, servers);
            }
            servers.add(server);
        }
    }
    // 更新 ZoneServer 节点内容
    setServerListForZones(serversInZones);
}
```

### LoadBalancerStats.updateZoneServerMapping()

````java
public void updateZoneServerMapping(Map<String, List<Server>> map) {
    upServerListZoneMap = new ConcurrentHashMap<String, List<? extends Server>>(map);
    // make sure ZoneStats object exist for available zones for monitoring purpose
    for (String zone: map.keySet()) {
        getZoneStats(zone);
    }
}
````


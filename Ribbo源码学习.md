Ribbon 是 Netflix 公司开源的一个负载均衡项目。可以在 Zuul 中使用 Ribbon 做负载均衡，也可以和 Feign 结合使用。在 Spring Cloud 开发中使用的最多的可能就是 RestTemplate 和 Ribbon。代码可能如下：

```java
@Configuration
public class RibbonConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

使用 RestTemplate 消费服务接口的代码可能是这样的：

```java
@Service
public class RibbonService {
    @Autowired
    private RestTemplate restTemplate;
    
    public String hi(String name) {
        return restTemplate.getForObject("http://eureka-client/hi?name="+name,String.class);
    }
}
```

RestTemplate 在 Spring 中就已经存在了，查看以上的代码可以发现 RestTemplate Bean 上有一个 @LoadBalanced 注解，这个注解标记在 RestTemplate 上，让负载均衡客户端 LoadBalancerClient 来配置它。

## LoadBalancerClient

spring-cloud-common 包中提供了 LoadBalancerClient 接口，它是 Ribbon 中一个非常重要的组件。继承结构如下：

![LoadBalancerClient 继承结构](C:\Users\user1\Desktop\笔记\LoadBalancerClient 继承结构.png)

而在 spring-cloud-commons 中相同的包下面，可以看到 LoadBalancerAutoConfiguration，看类名就能看出来这是一个自动配置类：

![LoadBalancerAutoConfiguration](C:\Users\user1\Desktop\笔记\LoadBalancerAutoConfiguration.jpg)

```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {
    // 省略代码。。。主要是对 LoadBalancerInterceptor 和 RetryLoadBalancerInterCeptor 的等进行配置，这里我们看类上的注解@ConditionalOnBean和@ConditionalOnClass
}
```

可以看到该自动配置类上有注解 `@ConditionalOnBean(LoadBalancerClient.class)` 和 `@ConditionalOnClass(RestTemplate.class)`，也就是说此类的生效条件是：

- 当前工程中要有 RestTemplate 类
- 在 Spring 的 IOC 容器中必须要有 LoadBalancerClient 的实现 Bean

而在 org.springframework.cloud.netflix.ribbon 包中，存在 RibbonAutoConfiguration.java 配置类：

```java
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
@AutoConfigureAfter(
		name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
		AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
		ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration {
    // 省略。。。
    @Bean
	@ConditionalOnMissingBean(LoadBalancerClient.class)
	public LoadBalancerClient loadBalancerClient() {
		return new RibbonLoadBalancerClient(springClientFactory());
	}
    // 省略。。。
}
```

该配置类中配置了 RibbonLoadBalancerClient 的 Bean，而且 RibbonLoadBalancerClient 继承 LoadBalancerClient，所以在启动服务的时候会初始化一个 LoadBalancerClient Bean，并加载各种负载均衡的拦截器配置。

LoadBalancerClient 接口中有三个方法：

```java
public interface LoadBalancerClient extends ServiceInstanceChooser {
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
    // 使用 LoadBalancer 的 ServiceInstance，对其执行请求，返回结果
	<T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException;
    // 构造一个包含主机和端口的真正的 url
    // http://serviceId/path/... --> http://host:port/path/...
	URI reconstructURI(ServiceInstance instance, URI original);
}
```

ServiceInstanceChooser 中的方法：

```java
public interface ServiceInstanceChooser {
    ServiceInstance choose(String serviceId); // 通过 serviceId 选择服务
}
```

在 RibbonLoadBalancerClient 中，主要有以下几个方法：

```java
@Override
public ServiceInstance choose(String serviceId) {
    Server server = getServer(serviceId);
    if (server == null) {
        return null;
    }
    return new RibbonServer(serviceid, server, isSecure(server, serviceId), serverInstrospector(serviceId).getMetaData(server));
}
public Server getServer(String serviceId) {
    return getServer(getLoadBalancer(serviceId));
}
public Server getServer(ILoadBalancer loadBalancer) {
    // ... 省略代码
    return loadBalancer.chooseServer("default"); // TODO: better handling of key
}
```

RibbonLoadBalancerClient 通过 `choose` 方法获取服务实例，其中又调用 `getServer` 方法来获取 Server 对象，跟踪源码可以看到，最终是通过 ILoadBalancer 的 `chooseServer` 去选择服务实例。

```java
public interface ILoadBalancer {
    // 添加一个 Server 集合
    public void addServers(List<Server> newServers);
    // 根据 key 获取 Server
    public Server chooseServer(Object key);
    // 标记某个服务下线
    public void markServerDown(Server server);
    // 获取可用的 Server 集合
    public List<Server> getReachableServers();
    // 获取所有的 Server集合
    public List<Server> getAllServers();
}
```

## DynamicServerListLoadBalancer

跟踪源码后，我们可以找到 ILoadBalancer 的继承结构如下，DynamicServerListLoadBalancer 继承了 ILoadBalancer，也就是说我们可以通过跟踪这个类来搞清楚 Ribbon 是如何实现负载均衡的。

![ILoadBalancer 继承结构](C:\Users\user1\Desktop\笔记\ILoadBalancer 继承结构.png)

查看 DynamicServerListLoadBalancer，BaseLoadBalancer，ILoadBalancer 这三个类，配置了 IClientConfig，IRule，IPing，ServerList，ServerListFilter 和 ILoadBalancer，在 BaseLoadBalancer 中，默认进行了以下配置：

- IClientConfig ribbonClientConfig：DefaultClientConfigImpl 配置（用于客户端的负载均衡配置）
- IRule ribbonRule：默认路由策略为 RoundRobinRule
- IPing ribbonPing：DummyPing
- ServerList ribbonServerList：ConfigurationBasedServerList
- ServerListFilter ribbonServerListFilter：ZonePreferenceServerListFilter
- ILoadBalancer ribbonLoadBalancer：ZoneAwareLoadBalancer

## IRule

IRule 有很多默认的实现类，都通过不同的算法来处理负载均衡，Ribbon 中实现的 IRule 又以下几种：、

![IRule 实现类](C:\Users\user1\Desktop\笔记\IRule 实现类.png)

- BestAvailableRule：选择最小请求数
- ClientConfigEnabledRoundRobinRule：轮询
- RandomRule：随机选择 Server
- RoundRobinRule：轮询
- WeightedResponseTimeRule：根据响应时间分配一个权重 weight，weight越低，被选择的可能性就越低
- ZoneAvoidanceRule：根据 Server 的 Zone 区域和可用性轮询选择

## IPing

IPing 的实现类又 PingUrl，PingConstant，NoOpPing，DummyPing 和 NIWSDiscoveryPing。IPing 接口中有一个 `isAlive` 方法。

```java
public boolean isAlive(Server Server);
```

通过向 Server 发送 ping 信号，来判断 Server 是否可用

![IPing 实现类](C:\Users\user1\Desktop\笔记\IPing 实现类.png)

- PingUrl 真实的去ping 某个url，判断其是否alive
- PingConstant 固定返回某服务是否可用，默认返回true，即可用
- NoOpPing 不去ping,直接返回true,即可用。
- DummyPing 直接返回true，并实现了initWithNiwsConfig方法。
- NIWSDiscoveryPing，根据DiscoveryEnabledServer的InstanceInfo的InstanceStatus去判断，如果为InstanceStatus.UP，则为可用，否则不可用。

## ServerList

ServerList 是定义了获取所有的 Server 的注册列表信息的接口。

```java
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers();
	public List<T> getUpdatedListOfServers();
}
```

## ServerListFilter

ServerListFilter 可根据配置过滤或者根据特性动态获取符合条件的 Server 列表。该类也是一个接口

```java
public interface ServerListFilter<T extends Server> {
    public List<T> getFilteredListOfServers(List<T> servers);
}
```

## 源码分析

### DynamicServerListLoadBalancer 构造方法

DynamicServerListLoadBalancer 中的构造方法中调用了一个方法 - `initWithNiwsConfig()`。

```java
public DynamicServerListLoadBalancer(IClientConfig clientConfig) {
    initWithNiwsConfig(clientConfig);
}
public initWithNiwsConfig(IClientConfig clientConfig) {
    try {
        super.initWithNiwsConfig(clientConfig);
        // 获取 ServerList 的 classname
        String niwsServerListClassname = clientConfig.getPropertyAsString(CommonClientConfigKey.NIWSServerListClassName, DefaultClientConfigImpl.DEFAULT_SERVER_LIST_CLASS);
        // 构造 ServerList
        ServerList<T> niwsServerListImpl = (ServerList<T>) ClientFactory.instantiateInstanceWithClientConfig(niwsServerListClassName, clientConfig);
        // 如果该 ServerList 是 AbstractServerList 的子类，则获取并设置过滤器
        if (niwsServerListImpl instanceof AbstractServerList) {
            AbstractServerListFilter<T> niwsFilter = ((AbstractServerList) niwsServerListImpl).getFilterImpl(clientConfig);
            niwsFilter.setLoadBalancerStats(getLoadBalancerStats());
            this.filter = niwsFilter;
        }
        // 获取 ServerListUpdater 的 classname
        String serverListUpdaterClassName = clientConfig.getPropertyAsString(
        	CommonClientConfigKey.ServerListUpdaterClassname,
            DefaultClientConfigImpl.DEFAULT_SERVER_LIST_UPDATER_CLASS
        );
        
        // 通过 classname 构造 ServerListUpdater 设置到 serverListUpdater 成员属性中
        this.serverListUpdater = (ServerListUpdater) ClientFactory
            .instantiateInstanceWithClientConfig(serverListUpdaterClassName, clientConfig);
        // 执行剩下的初始化操作
        restOfInit(clientConfig);
    }
}
```

### restOfInit

```java
void restOfInit(IClientConfig clientConfig) {
    boolean primeConnection = this.isEnablePrimingConnections();
    // 将这个关闭来避免 BaseLoadBalancer.setServerList() 中重复的异步启动
    this.setEnablePrimingConnections(false);
    enableAndInitLearnNewServersFeature();
    
    updateListOfServers(); // 用来获取所有的 Server 
    if (primeConnection && this.getPrimeConnections() != null) {
        this.getPrimeConnections().primeConnections(getReachableServers());
    }
    this.setEnablePrimingConnections(primeConnection);
    LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
}
```

上面源码中的 `updateListOfServers()` 最终是通过 serverListImpl.getUpdatedListOfServers() 来获取所有的服务列表的：

```java
@VisibleForTesting
public void updateListOfServers() {
    List<T> servers = new ArrayList<T>();
    if (serverListImpl != null) {
        servers = serverListImpl.getUpdatedListOfServers();
        LOGGER.debug("List of Servers for {} obtained from Discovery client: {}", 
                     getIdentifier(), servers);
        if (filter != null) {
            // 如果配置了过滤器，则将符合条件的 server 筛选出来
            servers = filter.getFilteredListOfServers(servers);
            LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}", getIdentifier(), servers);
        }
    }
    updateAllServerList(servers);
}
```

其中 serverListImpl 是 ServerList 的实现类 - DiscoveryEnabledNIWSServerList。而 `getUpdatedListOfServers()` 的具体实现为：

```java
@Override
public List<DiscoveryEnabledServer> getInitialListOfServers() {
    return obtainServersViaDiscovery();
}
public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
    return obtainServersViaDiscovery();
}
```

### obtainServersViaDiscovery

```java
private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList<DiscoveryEnabledServer>();

    if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList<DiscoveryEnabledServer>();
    }

    EurekaClient eurekaClient = eurekaClientProvider.get();
    if (vipAddresses!=null){
        for (String vipAddress : vipAddresses.split(",")) {
            // if targetRegion is null, it will be interpreted as the same region of client
            List<InstanceInfo> listOfInstanceInfo = eurekaClient.getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
            for (InstanceInfo ii : listOfInstanceInfo) {
                if (ii.getStatus().equals(InstanceStatus.UP)) {

                    if(shouldUseOverridePort){
                        if(logger.isDebugEnabled()){
                            logger.debug("Overriding port on client name: " + clientName + " to " + overridePort);
                        }

                        // copy is necessary since the InstanceInfo builder just uses the original reference,
                        // and we don't want to corrupt the global eureka copy of the object which may be
                        // used by other clients in our system
                        InstanceInfo copy = new InstanceInfo(ii);

                        if(isSecure){
                            ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                        }else{
                            ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                        }
                    }

                    DiscoveryEnabledServer des = new DiscoveryEnabledServer(ii, isSecure, shouldUseIpAddr);
                    des.setZone(DiscoveryClient.getZone(ii));
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
```

看到这里，可以知道负载均衡器 Ribbon 是通过 Eureka Client 来获取注册列表信息，然后通过配置的路由规则 IRule 来路由。但是它从 Eureka Client 获取注册信息的时间间隔是多久呢？

### 定时任务更新服务器列表和状态

我们可以看到 BaseLoadBalancer 的构造方法中，开启了一个 PingTask 任务，PingTask 是 BaseLoadBalancer 的内部类，根据 IPingStrategy 策略来发送 ping 请求获取和更新服务器列表，默认策略是 SerialPingStrategy。

```java
// BaseLoadBalancer.java
// Constructor
public BaseLoadBalancer(String name, IRule rule, LoadBalancerStats stats,
        IPing ping, IPingStrategy pingStrategy) {

    logger.debug("LoadBalancer [{}]:  initialized", name);
    
    this.name = name;
    this.ping = ping;
    this.pingStrategy = pingStrategy;
    setRule(rule);
    setupPingTask();
    lbStats = stats;
    init();
}
// 定时任务
void setupPingTask() {
    if (canSkipPing()) {
        return;
    }
    if (lbTimer != null) {
        lbTimer.cancel();
    }
    lbTimer = new ShutdownEnabledTimer("NFLoadBalancer-PingTimer-" + name,
            true);
    lbTimer.schedule(new PingTask(), 0, pingIntervalSeconds * 1000); // 默认10秒执行一次
    forceQuickPing();
}
// 内部类 PingTask
class PingTask extends TimerTask {
    public void run() {
        try {
        	new Pinger(pingStrategy).runPinger();
        } catch (Exception e) {
            logger.error("LoadBalancer [{}]: Error pinging", name, e);
        }
    }
}
// 内部类 Pinger
class Pinger {

    private final IPingStrategy pingerStrategy;

    public Pinger(IPingStrategy pingerStrategy) {
        this.pingerStrategy = pingerStrategy;
    }

    public void runPinger() throws Exception {
        // 用 CAS 设置 pingInProgress 为 true，代表正在执行 Ping 任务。
        // 如果设置失败，则表示有线程正在执行 Ping 任务，这里就不再执行
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
            // 读锁应该是空闲状态，除了 addServer 操作正在执行。
            allLock = allServerLock.readLock();
            // 加读锁，避免其他线程修改 serverList
            allLock.lock();
            allServers = allServerList.toArray(new Server[allServerList.size()]);
            // 解锁
            allLock.unlock();

            int numCandidates = allServers.length;
            // 向每个服务器发送 ping 请求，得到一个布尔值的结果集（服务器是否存活 - 能否请求成功）
            results = pingerStrategy.pingServers(ping, allServers);

            final List<Server> newUpList = new ArrayList<Server>();
            final List<Server> changedServers = new ArrayList<Server>();

            // 遍历当前所有Server
            for (int i = 0; i < numCandidates; i++) {
                // 获取第 i 个 Server 是当前否为存活状态（UP）
                boolean isAlive = results[i];
                Server svr = allServers[i];
                // 获取 ping 之前的服务器状态
                boolean oldIsAlive = svr.isAlive();

                // 将该服务器状态改为当前获取到的状态
                svr.setAlive(isAlive);

                if (oldIsAlive != isAlive) { // 如果之前状态与当前获取的状态不一致
                    // 加入到状态更改过的服务器列表中
                    changedServers.add(svr);
                    // 输出日志：当前服务器状态修改
                    logger.debug("LoadBalancer [{}]:  Server [{}] status changed to {}", 
                		name, svr.getId(), (isAlive ? "ALIVE" : "DEAD"));
                }

                if (isAlive) {
                    // 如果获取到的当前状态为 true（存活UP状态）
                    // 则将该服务器加入到 newUpList 中，用于后面更新至存活服务器列表
                    newUpList.add(svr);
                }
            }
            // 更新存活服务器列表，需要加写锁，避免并发问题
            upLock = upServerLock.writeLock();
            upLock.lock();
            upServerList = newUpList;
            upLock.unlock();

            // 通知服务器状态修改
            notifyServerStatusChangeListener(changedServers);
        } finally {
            pingInProgress.set(false);
        }
    }
}
```

setupPingTask 方法中初始化了一个定时任务，默认每隔10秒执行一次 PingTask。PingTask 继承了 TimerTask，执行其中的 run 方法，最终调用了 Pinger 的 runPinger 方法，构造方法中传入了 Ping 的策略。首先将用 CAS 来修改 pingInProgress （AtomicBoolean 对象），如果修改不成功，则表示当前有其他线程正在发送 ping 请求，并且还没有执行完毕，所以当前可以不再执行。

然后通过加读锁获取 allServerList（避免获取数据时有其他线程修改，保证当前数据一致）。获取完成后解锁，在将 allServerList 传入 pingStrategy 的 pingServers 方法获取每个服务器的可用性（存活状态）。遍历当前服务器列表，判断每个服务器之前的状态与刚刚获取到的状态是否一致，不一致则加入到 changedServers 列表中。最后更新存活服务器列表，执行服务器状态修改事件。

全部执行完成后，将 pingInProgress 改为 false。

### 总结

由此可见，Ribbon的负载均衡，主要是通过 LoadBalancerClient 来实现的，而 Load'BalancerClient 又将具体实现交给了 ILoadBalancer 来处理，ILoadBalancer 通过配置 IRule、IPing 等信息，向 Eureka 获取服务注册列表，并且 10s 一次向 EurekaClient 发送 ping 请求，来判断服务的可用性，如果服务的可用性发生改变或者服务数量与之前的不一致，则更新当前服务器列表或重新拉取。最后 ILoadBalancer 获取到这些服务列表之后，便可以根据 IRule 来进行负载均衡。

而 RestTemplate 被 @LoadBalanced 注解后，能够实现负载均衡，主要是通过给 RestTemplate 添加拦截器，较给负载均衡器去处理




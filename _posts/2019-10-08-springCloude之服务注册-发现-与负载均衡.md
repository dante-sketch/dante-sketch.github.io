本文主要内容: 探索服务是如何注册到注册中心、消费者(客户端client)如何发现服务，然后使用负载均衡进行消费的。

在云应用中，服务提供者的位置(IP端口等)可能是动态的，所以需要一个服务的注册和发现机制,让服务提供者注册自己的服务位置，消费者找到对应的服务进行消费。
本文主要探讨服务注册、发现机制以及基于发现机制的负载均衡.

### 服务注册
服务注册，用于服务实例(ServiceInstance)向注册中心提供自己的位置和附加的数据(比如服务版本号等)。

在```spring-cloud-commons```模块里，定义了服务注册和发现的基本协议,提供了注册服务和发现服务的规范.
具体的服务注册实现比如Zookeeper、Eureka均需要提供这些规范协议的实现。

我们现在看看服务发现包```org.springframework.cloud.client.serviceregistry```里的核心类:
- ServiceRegistry (服务注册类，用于注册服务)
- Registration 注册信息(包含服务提供者的位置、服务等)

```java
// ServiceRegistry用于注册、注销服务提供者的信息
public interface ServiceRegistry<R extends Registration> {
	/**
	 * 注册服务信息，一般是服务器实例的hostname(IP域名等)和对应的端口
	 */
	void register(R registration);

	/**
	 * 注销服务信息
	 */
	void deregister(R registration);

	/**
    	 * 关闭注册中心
	 */
	void close();

	/**
	 * 设置服务的状态
	 */
	void setStatus(R registration, String status);

	/**
	 * 获取服务状态
	 */
	<T> T getStatus(R registration);
}
// Registration 包含了 服务实例信息
public interface Registration extends ServiceInstance {
}
// 服务实例信息类
public interface ServiceInstance {
	/**
	 * 返回服务实例的ID 
	 */
	default String getInstanceId() {
		return null;
	}
	/**
	 * 返回服务ID，一个服务ID可能对应多个实例ID 
	 */
	String getServiceId();

	String getHost();

	int getPort();

	boolean isSecure();

	URI getUri();

	Map<String, String> getMetadata();

	default String getScheme() {
		return null;
	}
}
```

### 服务发现
服务发现一般依赖于服务的注册，但是在spring-cloud-commons 模块中，是将服务注册和服务发现分开的.
估计部分原因是因为服务的发现可以独立于服务的注册。比如，服务发现可以通过配置文件等形式实现，而不需要依赖于服务注册中心。  
服务发现的核心类位于```org.springframework.cloud.client.discovery```包里面，核心类有:
- DiscoveryClient 

DiscoveryClient 用于获取服务提供者的信息，比如有哪些服务，对应的服务实例的信息.
```java
public interface DiscoveryClient extends Ordered {
	/**
	 * 用于健康检查
	 */
	String description();

	/**
	 * 返回所有的服务的ID
	 */
	List<String> getServices();

	/**
	 * 获取指定服务的所有实例
	 */
	List<ServiceInstance> getInstances(String serviceId);
}
```

### 基于zookeeper的服务注册、发现与负载均衡
上面探索了spring-cloud 关于 服务注册、发现的接口约定，这里探索基于 Zookeeper的实现。
基于zookeeper的服务注册发现，需要引入依赖:org.springframework.cloud:spring-cloud-starter-zookeeper-discovery.
在查阅源代码的时候，先大致了解某个模块的依赖树，每个模块的职责，心里有个大体的轮廓，对于理解源码是有好处的。

```
 \- org.springframework.cloud:spring-cloud-starter-zookeeper-discovery:jar:2.1.1.RELEASE:compile
    +- org.springframework.cloud:spring-cloud-starter-zookeeper:jar:2.1.1.RELEASE:compile (引入spring-cloud-commons该模块包含了服务注册发现、负载均衡的协议接口)
    |  \- org.springframework.cloud:spring-cloud-zookeeper-core:jar:2.1.1.RELEASE:compile (完成zookeeper客户端实例的启动)
    +- org.springframework.cloud:spring-cloud-zookeeper-discovery:jar:2.1.1.RELEASE:compile (完成注册中心的创建、服务自动注册等功能)
    |  \- commons-configuration:commons-configuration:jar:1.8:compile
    +- org.apache.curator:curator-x-discovery:jar:4.0.1:compile
```

#### Zookeeper 服务注册
zookeeper 服务注册的启动顺序大致如下:  
- 启动 Zookeeper 客户端实例
- 创建注册中心
- 将服务实例自动化注册到注册中心

> Notes :  
> 通过依赖树，每个jar包里面的spring.factories文件、以及AutoConfigureBefore、AutoConfigureAfter以及各种Conditional，我们可以拼凑出各个自动配置的顺序，进而知道
> 各个bean的加载顺序。

**启动 Zookeeper 客户端实例**  
在ZookeeperAutoConfiguration中，根据配置完成 Zookeeper客户端CuratorFramework的初始化。

**创建注册中心**  
在ZookeeperServiceRegistryAutoConfiguration中，完成了ZookeeperServiceRegistry bean的加载。 ZookeeperServiceRegistry 比较重要,因为这是spring cloud 服务注册 ServiceRegistry 接口的实现,用于服务实例注册.
ZookeeperServiceRegistry bean的加载，依赖于zookeeper的扩展:服务发现模块curator-x-discovery(具体依赖于CuratorServiceDiscoveryAutoConfiguration自动配置提供).

**将服务实例自动化注册到注册中心**  
在 ZookeeperAutoServiceRegistrationAutoConfiguration 中,完成了服务实例信息(hostname和port等)的获取和自动注册到注册中心。
```java
public class ZookeeperAutoServiceRegistrationAutoConfiguration {
    public ZookeeperAutoServiceRegistrationAutoConfiguration() {
    }

    /**
     *	ZookeeperAutoServiceRegistration 继承了 spring 服务注册的模板类 AbstractAutoServiceRegistration
     *  当接收到 web server 的初始化事件时开始注册服务实例
     */
    @Bean
    public ZookeeperAutoServiceRegistration zookeeperAutoServiceRegistration(ZookeeperServiceRegistry registry, ZookeeperRegistration registration, ZookeeperDiscoveryProperties properties) {
        return new ZookeeperAutoServiceRegistration(registry, registration, properties);
    }
    
    /**
     *
     * 获取服务实例的位置信息
     */
    @Bean
    @ConditionalOnMissingBean({ZookeeperRegistration.class})
    public ServiceInstanceRegistration serviceInstanceRegistration(ApplicationContext context, ZookeeperDiscoveryProperties properties) {
        String appName = context.getEnvironment().getProperty("spring.application.name", "application");
        String host = properties.getInstanceHost();
        if (!StringUtils.hasText(host)) {
            throw new IllegalStateException("instanceHost must not be empty");
        } else {
            ZookeeperInstance zookeeperInstance = new ZookeeperInstance(context.getId(), appName, properties.getMetadata());
            RegistrationBuilder builder = ServiceInstanceRegistration.builder().address(host).name(appName).payload(zookeeperInstance).uriSpec(properties.getUriSpec());
            if (properties.getInstanceSslPort() != null) {
                builder.sslPort(properties.getInstanceSslPort());
            }

            if (properties.getInstanceId() != null) {
                builder.id(properties.getInstanceId());
            }

            return builder.build();
        }
    }
}
```

#### Zookeeper 服务发现
ZookeeperDiscoveryClient 实现了 spring-cloud 约定的 DiscoveryClient接口。
ZookeeperDiscoveryClient 的bean注册，是在自动配置类ZookeeperDiscoveryAutoConfiguration 中实现的。

```java
@ConditionalOnBean({Marker.class})
@ConditionalOnZookeeperDiscoveryEnabled
@AutoConfigureBefore({CommonsClientAutoConfiguration.class, NoopDiscoveryClientAutoConfiguration.class})
@AutoConfigureAfter({ZookeeperDiscoveryClientConfiguration.class})
public class ZookeeperDiscoveryAutoConfiguration {
    ... ... 
    /**
     * 服务发现依赖于curator-x-discovery 模块
     */
    @Bean
    @ConditionalOnMissingBean
    public ZookeeperDiscoveryClient zookeeperDiscoveryClient(ServiceDiscovery<ZookeeperInstance> serviceDiscovery, ZookeeperDiscoveryProperties zookeeperDiscoveryProperties) {
        return new ZookeeperDiscoveryClient(serviceDiscovery, this.zookeeperDependencies, zookeeperDiscoveryProperties);
    }
    ... ... 
}
```

大致理清了基于Zookeeper的服务注册和发现以后，接下来就是探索基于服务发现的负载均衡如何实现了。

#### 基于Zookeeper服务发现的 Ribbon负载均衡
先浏览一下依赖树，看看 spring-cloud-starter-zookeeper-discovery 模块都提供了什么功能 :  
```
\- org.springframework.cloud:spring-cloud-starter-zookeeper-discovery:jar:2.1.1.RELEASE:compile
   +- org.springframework.cloud:spring-cloud-starter-zookeeper:jar:2.1.1.RELEASE:compile
   |  \- org.springframework.cloud:spring-cloud-zookeeper-core:jar:2.1.1.RELEASE:compile
   +- org.springframework.cloud:spring-cloud-zookeeper-discovery:jar:2.1.1.RELEASE:compile
   |  \- commons-configuration:commons-configuration:jar:1.8:compile
   +- org.apache.curator:curator-x-discovery:jar:4.0.1:compile
   +- org.springframework.cloud:spring-cloud-netflix-core:jar:2.1.1.RELEASE:compile
   |  \- org.springframework.cloud:spring-cloud-netflix-hystrix:jar:2.1.1.RELEASE:compile
   +- org.springframework.cloud:spring-cloud-starter-netflix-archaius:jar:2.1.1.RELEASE:compile
   |  \- org.springframework.cloud:spring-cloud-netflix-archaius:jar:2.1.1.RELEASE:compile
   \- org.springframework.cloud:spring-cloud-starter-netflix-ribbon:jar:2.1.1.RELEASE:compile
      +- com.netflix.ribbon:ribbon:jar:2.3.0:compile
      |  +- com.netflix.ribbon:ribbon-transport:jar:2.3.0:runtime
      |  |  +- io.reactivex:rxnetty-contexts:jar:0.4.9:runtime
      |  |  \- io.reactivex:rxnetty-servo:jar:0.4.9:runtime
      |  \- io.reactivex:rxnetty:jar:0.4.9:runtime
      +- com.netflix.ribbon:ribbon-core:jar:2.3.0:compile
      +- com.netflix.ribbon:ribbon-httpclient:jar:2.3.0:compile
      |  +- commons-collections:commons-collections:jar:3.2.2:runtime
      |  +- org.apache.httpcomponents:httpclient:jar:4.5.8:runtime
      |  |  \- org.apache.httpcomponents:httpcore:jar:4.4.11:runtime
      |  +- com.sun.jersey:jersey-client:jar:1.19.1:runtime
      |  |  \- com.sun.jersey:jersey-core:jar:1.19.1:runtime
      |  |     \- javax.ws.rs:jsr311-api:jar:1.1.1:runtime
      |  +- com.sun.jersey.contribs:jersey-apache-client4:jar:1.19.1:runtime
      |  +- com.netflix.servo:servo-core:jar:0.12.21:runtime
      |  \- com.netflix.netflix-commons:netflix-commons-util:jar:0.3.0:runtime
      +- com.netflix.ribbon:ribbon-loadbalancer:jar:2.3.0:compile --------------------------- 负载均衡实现
      |  \- com.netflix.netflix-commons:netflix-statistics:jar:0.1.1:runtime  
      \- io.reactivex:rxjava:jar:1.3.8:compile
```

从依赖树里面，我们可以发现提供了 robbin 的负载均衡和 hystrix的熔断 等功能。
现在先看看负载均衡是如何实现的,从依赖树上大致可以猜测 ```com.netflix.ribbon:ribbon-loadbalancer``` 包是其中的关键. 
打开这个包，发现没有自动配置的文件spring.factories 。 看来找错地方了.
下一个应该试着从 ribbon 和 spring-cloud 关联的jar 包```spring-cloud-starter-netflix-ribbon```中看看.
打开jar包后，果然在spring.factories 文件里面包含了负载均衡的自动配置类: ```RibbonAutoConfiguration```

```java
// 摘取的部分代码
// a) 在 spring的负载均衡自动配置完成前，先注册LoadBalancerClient的具体实现
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
		AsyncLoadBalancerAutoConfiguration.class })
public class RibbonAutoConfiguration {
	// 实现了 spring-cloud 负载均衡的LoadBalanceClient，配合LoadBalanceInterceptor实现负载均衡
	@Bean
	@ConditionalOnMissingBean(LoadBalancerClient.class)
	public LoadBalancerClient loadBalancerClient() {
		return new RibbonLoadBalancerClient(springClientFactory());
	}
	@Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}
}
```

到目前为止， 我们大体可以了解到 RestTemplate 在责任链模式下，如何使用负载均衡拦截器实现负载均衡功能了。
唯一还存在疑惑的是，RibbonLoadBalancerClient 是如何检查到 在Zookeeper 注册的服务实例，从而进行具体的负载均衡的? 即RibbonLoadBalancerClient 是 如何 和 ZookeeperDiscoveryClient 勾搭起来的?
要知道这个秘密，我得试着查阅RibbonLoadBalancerClient的具体实现,看看能否找到线索。
```java
/**
 * spring-cloud的负载均衡拦截器，调用的是execute方法。
 * 通过execute方法，我们可以看到:用于获取ILoadBalancer的clientFactory是关键。
 *
 */
public class RibbonLoadBalancerClient implements LoadBalancerClient {

	private SpringClientFactory clientFactory;

	@Override
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request)
			throws IOException {
		return execute(serviceId, request, null);
	}
	public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
			throws IOException {
		ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
		Server server = getServer(loadBalancer, hint);
		if (server == null) {
			throw new IllegalStateException("No instances available for " + serviceId);
		}
		RibbonServer ribbonServer = new RibbonServer(serviceId, server,
				isSecure(server, serviceId),
				serverIntrospector(serviceId).getMetadata(server));

		return execute(serviceId, ribbonServer, request);
	}

	protected ILoadBalancer getLoadBalancer(String serviceId) {
		/**
                 * 这个ILoadBalancer 是否和Zookeeper的有关?
                 */
		return this.clientFactory.getLoadBalancer(serviceId);
	}

	/**
         * 从ILoadBalancer选出服务器
         */
	protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
		if (loadBalancer == null) {
			return null;
		}
		// Use 'default' on a null hint, or just pass it on?
		return loadBalancer.chooseServer(hint != null ? hint : "default");
	}
}
```
既然觉得 SpringClientFactory 是关键线索，那么我们得看看这个 SpringClientFactory是如何注册和实现的。
```
public class RibbonAutoConfiguration {
	@Autowired(required = false)
	private List<RibbonClientSpecification> configurations = new ArrayList<>();

	@Bean
	public SpringClientFactory springClientFactory() {
		SpringClientFactory factory = new SpringClientFactory();
		factory.setConfigurations(this.configurations);
		return factory;
	}
}


public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification> {
	// 获取 ILoadBalancer
	public ILoadBalancer getLoadBalancer(String name) {
		return getInstance(name, ILoadBalancer.class);
	}
	//获取ILoadBalancer类型的实例
	public <C> C getInstance(String name, Class<C> type) {
		C instance = super.getInstance(name, type);
		if (instance != null) {
			return instance;
		}
		IClientConfig config = getInstance(name, IClientConfig.class);
		return instantiateWithConfig(getContext(name), type, config);
	}
}
```

通过查阅SpringClientFactory的代码，我发现还是无法知道ILoadBalance是在什么时候注册的。貌似线索断掉了。
后来，突然想到之前好像在哪个jar的spring.factories文件中看到ribbon与zookeeper的关联自动配置类。是哪一个jar呢?大概率是zookeeper的discover相关的jar包。
通过查找```spring-cloud-starter-zookeeper-discovery``` 和 ```spring-cloud-zookeeper-discovery```jar 包，终于在spring-cloud-zookeeper-discovery 的 spring.factories
文件中发现了关键的一行:

```java
# Auto Configuration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.zookeeper.discovery.RibbonZookeeperAutoConfiguration,\
```
RibbonZookeeperAutoConfiguration 是将 robbin 和 zookeeper 关联起来的自动配置类!

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnZookeeperEnabled
@ConditionalOnBean({SpringClientFactory.class})
@ConditionalOnRibbonZookeeper
@AutoConfigureAfter({RibbonAutoConfiguration.class}) // 在RibbonAutoConfiguration之后配置
@RibbonClients(
    defaultConfiguration = {ZookeeperRibbonClientConfiguration.class} //间接引入的这个ZookeeperRibbonClientConfiguration是关键
)
public class RibbonZookeeperAutoConfiguration {
    public RibbonZookeeperAutoConfiguration() {
    }
}

@Configuration
public class ZookeeperRibbonClientConfiguration {
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnDependenciesPassed
    public ServerList<?> ribbonServerListFromDependencies(IClientConfig config, ZookeeperDependencies zookeeperDependencies, ServiceDiscovery<ZookeeperInstance> serviceDiscovery) {
        ZookeeperServerList serverList = new ZookeeperServerList(serviceDiscovery);
        serverList.initFromDependencies(config, zookeeperDependencies);
        log.debug(String.format("Server list for Ribbon's dependencies based load balancing is [%s]", serverList));
        return serverList;
    }
	
    /**
     * 终于找到了ILoadBalancer 的实现类进行bean注册的地方
     * DependenciesBasedLoadBalancer 通过ZookeeperServerList 间接连接了 zookeeper的发现类ServiceDiscovery
     * Robbin 负载均衡终于和 Zookeeper 关联起来了。
     */ 
    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnDependenciesPassed
    @ConditionalOnProperty(
        value = {"spring.cloud.zookeeper.dependency.ribbon.loadbalancer"},
        matchIfMissing = true
    )
    public ILoadBalancer dependenciesBasedLoadBalancer(ZookeeperDependencies zookeeperDependencies, ServerList<?> serverList, IClientConfig config, IPing iPing) {
        return new DependenciesBasedLoadBalancer(zookeeperDependencies, serverList, config, iPing);
    }
}
```

到目前为止，本文探索了 spring-cloud 的 服务注册、发现、以及基于发现的负载均衡 协议，以及一个具体的实现: Zookeeper & Robbin.
探索了这个完整的流程以后，真心觉得 spring ioc 容器真是伟大，基于spring可以轻松连接各个框架，像极了胶水框架。
另外，从基于 RestTemplate 的负载均衡实现，深感 设计模式的强大。基于责任链，不单单代码清晰了，而且增删功能都极其方便，是开闭原则的经典例子。

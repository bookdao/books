##微服务实现方实时流量监控

### 项目介绍
`lorik-hystrix-metric-monitor`是一个依赖于断路器Hystrix、支持对微服务Server进行实时流量监控。
  
### 使用场景

本组件主要适用于微服务实现方（Server)

### Maven依赖

#### hystrix-core 1.5.3及以上
SpringCloud版本号：1.1.1.RELEASE及以上
```xml  
	<dependency>
		<groupId>com.dooioo.se.lorik</groupId>
		<artifactId>lorik-hystrix-metric-monitor</artifactId>
		<version>0.0.3</version>
	</dependency>
```
####  hystrix-core 1.5.2及以下
SpringCloud版本号：1.1.0.RELEASE及以下
```xml  
	<dependency>
		<groupId>com.dooioo.se.lorik</groupId>
		<artifactId>lorik-hystrix-metric-monitor</artifactId>
		<version>0.0.1.1</version>
	</dependency>
```

### 注意事项
除了依赖以上组件外，还应添加以下依赖：

```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-hystrix</artifactId>
		</dependency>
```

同时代码要开启断路器特性： `@EnableCircuitBreaker`

推荐使用注解：`@SpringCloudApplication` 代替：`@SpringBootApplication`

开启断路器后，可通过: `feign.hystrix.enabled: false` 禁用SPI调用时开启断路（如果需要的话）

### 流量查看
 
 URL: `http://turbine.se.dooioo.cn/monitor?serviceId=`
 
 serviceId值为你项目的spring.application.name；
 
 正式环境顶级域名改成com;
 
 
### 断路器的一些配置属性说明
```json
#断路器命令相关
hystrix.command.default.execution:
          timeout.enabled: false #是否允许线程执行超时，默认true；如果你希望以HttpClient的超时时间为准，则可以设置为false
          isolation.thread.timeoutInMilliseconds: 6000 #timeout.enabled=true时，线程中执行方法时的超时时间，默认1秒，调整为6秒才中断（内网调用已经很长了）
          isolation.strategy: THREAD #默认隔离级别为线程，写出来仅为了让开发知道；还有可选值：SEMAPHORE，信号量

#断路器隔离级别为线程时线程池的配置（isolation.strategy: THREAD)
hystrix.threadpool.default:
          maxQueueSize: 80 #线程池的队列大小，-1=SynchronousQueue，>0则使用有界队列，池初始化之后不能变更
          coreSize: 10  #线程池默认core=10，默认值为10，一般保持默认值即可
          maximumSize: 10 #线程池最大线程数，默认值为10
          queueSizeRejectionThreshold: 50 #默认值为5，由于线程池创建之后，maxQueueSize不能动态调整，因此使用此参数方便线上动态调整队列允许的任务数
        
#断路器隔离级别为信号量时（isolation.strategy: SEMAPHORE)
hystrix.threadpool.default:
          execution.isolation.semaphore.maxConcurrentRequests: 80 #信号量的大小，代表并发数多少
 
##断路逻辑的配置
hystrix.command.default.circuitBreaker:
                requestVolumeThreshold: 30 ###10秒内请求数超过30次才应用断路器逻辑，默认20次
                errorThresholdPercentage: 65 ##10秒内requestVolumeThreshold超过%65的请求失败，则开启断路器，默认%50
                sleepWindowInMilliseconds: 10000 ##断路后多久不向后端发送请求，默认5秒；
         
#断路器其他默认配置           
hystrix.command.default:
         fallback.enabled: false #默认true，我们禁止使用fallback
         requestCache.enabled: false  #默认true，我们暂不涉及客户端缓存
         requestLog.enabled: false #默认true，我们暂不关心执行过程的日志

```

<!-- toc -->
### 微服务
关于微服务的讨论，可以参考： [Martin Fowler - MircroServices](http://martinfowler.com/articles/microservices.html "martinfowler microservices")。

站在工程师的角度看，微服务涉及三种角色：

* Service SPI [^1]   服务契约，一般称之为：SPI
* Service Provider 服务实现方，一般称之为Server
* Client 服务调用方，即 SPI 依赖方，一般称之为UI/Client

服务提供方公布一系列接口（```SPI```），使用Maven构建成jar包，同时，另起一个项目实现SPI（```Service Provider```)。

对于依赖某个服务的工程师来说，TA只需引入特定版本的SPI jar，比如: ```loupan-spi-1.1.3.jar```，代码中引入Service 和调用方式并没有什么变化，Spring项目代码如下：

```java
  @Service
  public class MyService{
  	// CitySpi 为SPI
    @Autowired
    CitySpi citySpi;
    
    public List<MyObject> findById(int id){
       City city= citySpi.findById(id);
       if(city == null){
        return Collections.emptyList();
       }
       // other logic
     }
  }
```
但是在底层，Service调用已由 ```同一进程内调用``` 透明的转换为 ```跨JVM进程间调用```。

之前，当我们某个项目线提供一组公共服务时，通常将服务封装成Jar包发布，但随之而来的，如果Jar包出现Bug、配置文件变更、功能调整、或者安全性问题时，所有依赖方（客户端）必须强制升级，真可谓，牵一发而动全身，我们称这种服务依赖方式为：“**硬引用**“。

微服务是如何避免这种尴尬的服务维护呢？

对客户端来说，如果我要萝卜，服务提供方给它萝卜即可，至于萝卜是从何而来，客户端是不关心的。反观我们采用“**硬引用**”的方式暴露服务时，将实现方式直接驻留在了客户端，从而变相地和客户端耦合在了一起。

基于微服务的架构开发时，项目线（```Service Provider```）只向客户端(```Client```)提供 ```Service SPI```。SPI是一系列声明（接口和Model），没有任何实现细节，驻留在客户端的只是一个空壳，真身仍在项目线。

如果某服务声明说我有（返回）萝卜，那客户端就能获取萝卜，绝无可能冒出白菜（所谓的契约）。至于萝卜的制造工艺，各项目线可随时变更，只要不违背契约即可（通常服务都有版本号）。

我们称这种服务依赖方式为: ”**软引用**“，类似文件的快捷方式。

微服务还有其他诸多特点或挑战，但在此书中不会过多讨论。

### 微服务、Spring Cloud、Netflix OSS[^2]

上文说到，微服务的调用是一种跨JVM进程的调用。也就意味着，微服务离不开分布式系统之间的交互。

在一个分布式的环境中，我们通常会部署多个节点（实例），而且可能动态地增加一些节点来应对紧急网络流量。传统的基于```Nginx```的静态路由（由人工手动配置），显然是不合适的。

一次典型的微服务调用，其背后的基本步骤如下：  

	 1. 服务发现 - 根据所调用的服务查找Service Provider的所有可用节点；
	 2. 服务路由 - 从众多节点以某种规则选择一个节点。
	 3. RPC请求 - 向选定的节点发起远程网络调用(RPC)，解析返回数据。
 
由于网络调用的复杂性，我们还必须处理RPC[^3]请求异常时可能导致的雪崩效应[^4]，所以网络流量整形[^5]和监控是必要的。

另外，由于部署多个节点，如何管理应用程序的运行时配置也是必须考虑的。

针对以上需求，Spring团队整合了一种解决方案：Spring Cloud + Netflix OSS。

#### 服务发现 - Eureka
[Netflix Eureka 1.X.X](https://github.com/Netflix/eureka/wiki "Eureka Wiki") 使用了一种简单机制实现了节点的自动注册和发现。

Eureka 有两种角色：```Eureka Client``` 和 ```Eureka Server```。通常我们的Spring Cloud 程序都是作为```Eureka Client```启动的。 

Eureka Client启动时，会根据配置的Eureka Server的url，主动向Eureka Server汇报自身以下信息：

*  ```虚拟主机地址（VIPAddress)```  
	服务的标识，极其重要，只有根据VIPAddress才能在Eureka Server中查找服务的节点。
*  ```主机名或IP地址```  
默认是主机名，但通常都配置成节点的IP地址。
*  ```端口```  
节点对外提供服务的端口
*  ```节点实例ID```  
全局唯一，如果重复，会覆盖之前注册相同实例ID的节点信息。
*  ```status url ```  
程序运行状态的监控url.
*  ```health url ```  
程序健康状况的监控url
*  ```metadata 和其他 ```  
自定义的元数据信息，可被其他Client获取。

与此同时，```Eureka Client```会开启一些定时任务，按固定频率向```Eureka Server```轮询其他客户端的注册信息（注册信息会缓存在本地一段时间），当然也少不了发送心跳包。

```Eureka Server(1.X.X)```目前不支持向所有节点广播节点变更，节点的自动发现主要依赖Eureka Client定时Poll数据。

默认情况下，```Eureka Client```的实例ID为主机名，这样同一个台机器只能运行一个Client。通过Spring，我们可以让Eureka Client启动时生成随机的Instance Id。

SE团队目前部署了两个Eureka Server节点（集成环境）：  
[http://discovery1.se.dooioo.cn](http://discovery1.se.dooioo.cn) 和[http://discovery2.se.dooioo.cn](http://discovery2.se.dooioo.cn)，正式环境将顶级域名.cn 改为.com，访问即可查看所有已注册的```Eureka Client```。

#### 服务路由 - Ribbon
当Eureka Client通过待调用服务的```VIPAddress```获得该服务的多个节点时，如何路由到一个合适的节点？这时轮到Netflix的开源组件 [Ribbon](https://github.com/Netflix/ribbon/wiki "Netflix Ribbon Wiki") 出场了。

Ribbon 有以下特性：

* 插件式的路由规则，内置```Round Robin``` 和 ```Response time weighted ```。既可以向节点随机分发请求，也支持根据响应时间来分发。另外，我们也可以方便地扩展一些符合自己应用场景的路由规则。
* 集成Eureka服务发现（```Ribbon-Eureka```模块）。
* 弹性容错。Ribbon通过```IPing```接口可以动态感知节点是否存活，以过滤一些失去响应的节点。它可以进一步基于断路器[^6]模式过滤节点。关于断路器模式，请参考 [Circuit Breaker](http://martinfowler.com/bliki/CircuitBreaker.html)。
* 支持分布式云环境。假如，我们将服务部署到阿里云不同的数据中心，比如，北京，杭州，上海，节点路由时，可以优先选择位于同一数据中心的节点，也可以主动避开网络拥堵的数据中心。

默认情况下，```Ribbon``` 随机转发请求到各个节点。

#### RPC调用 - Feign
当Ribbon选定节点后，接下来客户端便要发起RPC调用了。

常见的RPC调用，有二进制流序列化+TCP协议 或 二进制流序列化+Http协议(比如Hessian)，序列化/反序列化协议既可以是原生JAVA，也可以是其他序列化协议（比如，[Kryo](https://github.com/EsotericSoftware/kryo/wiki) 和 [Fst](https://github.com/RuedigerMoeller/fast-serialization/wiki) ,Thrift,ProtoBuf)；有文本序列化+ Http协议，使用JSON或XML反序列化。

最终，我们的```RPC调用```决定采用```Http协议+JSON```文本序列化的方式，这样即可以直接将```服务作为REST接口暴露```出去，也更容易对众多服务进行自动化测试。

而完成这一RPC调用的组件，就是 [Netflix Feign](https://github.com/Netflix/feign)。

Feign 根据 ```Service SPI``` 接口的标注信息，构造符合Http协议的请求参数。你可以把Feign看成一个HttpClient，只不过它构造Http请求是通过接口的元数据-标注。比如，我们以后的```Service SPI```类似以下代码：

``` java
@FeignClient("city")
//@FeignClient(name="city",url="http://localhost:8080")
public interface CitySpi{
    @RequestMapping(value="/v1/citys/{id}",method=RequestMethod.GET)
    City findByIdV1(@PathVariable(value="id")int id );
 }
```

Feign 有一个接口 `feign.Contract`，用于完成Http请求的构造。Spring提供了 `SpringMvcContract`以支持Spring MVC标注的解析，解析在程序启动时就完成了。

```Service SPI```的每个接口都将被Feign代理（JAVA 动态代理），当方法调用时，模板参数将被动态地替换为方法实参。

上文中CityService的`findById`将会被Feign理解为：向 URL = http://city/v1/citys/{id}的主机发起一个Http **GET**请求，id为模板参数，运行时替换。

大家也看到了，URL的主机为```city```，也就是`@FeignClient("city")`中的```city```,```city```被称为**虚拟主机地址**，是服务的标识。一个服务通常是一个```Eureka Client```，服务会部署多个节点。

此时，发起RPC请求之前需向```Eureka Server```查询该虚拟主机的所有节点，再由```Ribbon```选定一个节点。

但我们也可以直接指定请求的url：`@FeignClient(name="city",url="http://localhost:8080")`，此时会被Feign理解为：向 URL = http://localhost:8080/v1/citys/{id}的主机发起一个Http **GET**请求。此时已无需Ribbon路由，通常在测试时才指定节点。

Url的主机地址被动态解析和替换之后，Feign开始发送请求，最后对响应信息反序列化。

#### 网络流量整形以及断路器
正常情况下，一次完整的RPC请求如上文所言。

俗话说，风平浪静时，人人都可以是舵手。

但当 Service Provider 无法及时响应时？有多种原因会导致此类情况。

* 客户端业务逻辑调整，导致某个时间点访问量暴增，大量请求被分发至Service Provider，服务提供方过载，数据库服务器CPU 100%，导致其他服务访问数据库时也超时。
* 服务提供方业务调整，原先一个50ms就能响应的请求，现在需要300ms，从而导致客户端大量线程阻塞，进而失去响应。
* …..

通常服务不可用时，终端用户可能不停地刷新页面，从而加剧问题。

以上情况我们统称为：```服务过载```。

为了应对分布式环境中服务过载可能引起的"雪崩反应”，我们启用了Netflix公司的开源组件 [Hystrix](https://github.com/Netflix/Hystrix/wiki)。

下面我简要介绍一下Hystrix的工作流程：

1,将所有对外部系统的调用封装成`HystrixCommand` 或 `HystrixObservableCommand` ，并运行在一个隔离的线程中。  
比如，当我们发起远程调用时，如果启用Hystrix，则实际的调用如下：

```java
  new HystrixCommand(){
      Object run(){
        //我们的RPC调用
      }
  }.execute();
```
2, 每个执行的`HystrixCommand`都有一个超时时间，默认是一秒。客户端可以配置。
如果 `HystrixCommand`执行超时，那么将抛出类型为超时的```HystrixRuntimeException```。（资源隔离）

3，为每个外部依赖维护一个小型的线程池或信号量，如果线程池满了，新的请求会立即被拒绝。线程池或信号量的大小即为转发到后端请求的最大并发数。（流量整形）

4，收集请求成功、失败、超时以及池满被拒绝的请求数。

5，根据收集到的统计数据，周期性地触发断路器。当断路器开启时，接下来的所有请求都会被直接拒绝。断路器可以手动触发或者当请求的错误率（errorThresholdPercentage，可配置，默认%50）达到一定阈值时自动触发。
断路器开启一段时间之后（sleepWindowInMilliseconds,可配置，默认5秒），会自动关闭。也就是说如果断路器开启，那么默认5秒内不会向外部依赖发送请求。

6，当请求失败、池满被拒绝、超时以及短路（断路器开启）时，`HystrixCommand`将执行一个默认逻辑。```HystrixCommand```有个`getFallback`方法，当失败时，你可以返回默认数据。


对服务过载的预防，```Hystrix```只是做了它能做的，但更重要的是，服务实现方、调用方必须透彻地理解自己的业务逻辑，明白自己服务的边界（QPS,TPS，依赖，服务可用率），从而调整好相关参数。

无论是前端Web或后台系统，请求发起方有义务为预防服务过载做一些应对措施。

如何保障服务的可用性，我们以后再详加讨论。

#### API网关：智能代理 Zuul
对于企业内部的服务间调用来说，以上的 ```Eureka & Ribbon & Feign & Hystrix ```基本就满足应用了。

但如果前端Web页面想Ajax访问内部服务呢？或者我们要暴露一些基础服务给移动端呢？

之前，各个项目线有单独的API模块。部署之后，```Nginx```配好指向，Web/App就可以通过域名访问接口了。

当改为微服务架构后，节点只能通过```Eureka Server```来发现了。而且，服务提供方就是标准的REST实现，只需要暴露出去即可，已无需单独的API模块了。

也就是说，我们需要个智能代理，它既可以动态发现新注册的Eureka节点，并把请求分发到合适的节点；又可以做一些权限控制，安全校验等通用的逻辑。

[Netflix Zuul](https://github.com/Netflix/zuul/wiki) 正是解决这一类问题的不二之选。

Zuul是由一系列过滤器组成的，过滤器内置四种类型：

* “**pre**” 请求预处理，可以做权限校验、CORS支持、OAuth认证等。
* “**route**” 将前端请求路由到后端节点。
* "**post**"  将后端节点的响应信息写入客户端。
* "**error**" 代理执行过程中，如果发生异常，则会调用此种类型的Filter处理异常。

我们知道，虚拟主机地址（```VIPAddress```)代表了一个服务，如果要从```Eureka Server```中获取特定服务的节点，则必须告诉```Eureka Server``` 虚拟主机地址。

那Zuul是如何知道 ```Request Path``` 关联的虚拟主机地址呢？

我们的Zuul代理项目(以下统称为**API网关**)，采用了一种命名约定，比如对以下的请求url:

```GET http://aroute.dooioo.cn/loupan/v1/building/3```

API网关默认会将”loupan“作为虚拟主机地址，并从请求路径中移除，此时，转发到后端节点的请求路径变为：  

``` GET http://node1:port/v1/building/3```

也就是说，访问API网关的request path必须以```虚拟主机地址开始```。

对于以下SPI代码：

``` java
@FeignClient("loupan-server")
public interface CitySpi{
    @RequestMapping(value="/v1/citys/{id}",method=RequestMethod.GET)
    City findByIdV1(@PathVariable(value="id")int id );
  }
```
如果通过API网关访问，则request path必须为：/loupan-server/v1/citys/{id}。

我们的API网关会将以```-```连起来的虚拟主机地址转为```/```分隔的路径，对于：
/loupan-server/v1/citys/{id}，客户端也可以访问：/loupan/server/v1/citys/{id}。

**我们称“loupan/server”为虚拟主机地址 “loupan-server”的别名**。  
  
强烈推荐前端Web以别名Path请求API网关。

SE团队的API网关域名为（集成环境）：[http://aroute.dooioo.cn](http://aroute.dooioo.cn),正式环境，请将.cn改为.com。  

最后让我们粗略过一下前端Web对示例SPI的请求流程：

1， Web 发起Ajax请求：  

```GET http://aroute.dooioo.cn/loupan/server/v1/citys/32```

2，请求到达API网关，首先类型为 “**pre**” 的Zuul Filter找到别名"/loupan/server"对应的虚拟主机地址："loupan-server"，并标识请求为Eureka，将虚拟主机地址放在 ```RequestContext```中。

3，接下来类型为 "**route**" 的Zuul Filter发现```RequestContext```有虚拟主机地址，开始使用Ribbon以某种路由规则选定服务的一个节点，移除虚拟主机地址或别名后发送请求。此时后端接受到的请求如下：  

```GET http://selected-node:port/v1/citys/32```

4，后端节点响应后，类型为 “**post**”的Zuul Filter将响应写入客户端。

5，以上任何一步发生异常，都会调用类型为 “**error**” 的Zuul Filter。

附带说明一点：我们的API网关已支持```CORS```，Web Ajax请求已无需考虑跨域的问题了。

#### 配置中心 Spring Cloud Config
对应用配置的管理，Spring Cloud目前默认的实现是使用`Git`集中存储，这样多节点也只有一份副本，管理和维护起来比较容易，相关模块为：[Spring-Cloud-Config](http://cloud.spring.io/spring-cloud-config/)。

当Spring Cloud项目启动时，首先会使用```PropertySourceLocator```自动加载远程Git仓库里的配置文件。

对开发人员来说，原先放在```/src/main/resources```目录下的配置文件，放在了Git里维护了而已。

具体细节，在此不多说。
 
<br>
<br>
 
[^1]: SPI = Service Provider Interface，服务提供方接口
[^2]: OSS = Open Source Software 开源软件
[^3]: RPC = Remote Procedure Call 远程过程调用
[^4]: “雪崩效应”是指信息在沿供应链传递中其波动会被依次放大的现象，这种现象导致信息在传递过程之中有如滚雪球一般越滚越大。
[^5]: 流量整形(traffic shaping)典型作用是限制流出某一网络的某一连接的流量与突发，使这类报文以比较均匀的速度向外发送。
[^6]: 断路器（Circuit Breaker）是指能够关合、承载和开断正常回路条件下的电流并能关合、在规定的时间内承载和开断异常回路条件下的电流的开关装置。

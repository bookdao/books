<!-- toc -->
## MicroService，What the hell?
今天我来和大家来分享我们微服务的编码规范，以及在微服务落地过程中的代码演化。

2015年底，我们决定将业务架构迁移到微服务的架构上。

Why? 因为听起来很美好。 O(∩_∩)O~

单体应用架构越来越乏味：项目臃肿、应用或模块耦合度过高、对新的团队成员不友好、文档长期缺乏、不方便敏捷部署 ==> 腐化。
### 技术选型
我们团队主流开发语言是Java，大多数项目是Web项目，采用的框架是Spring MVC Framework。很自然地，我们决定采用Spring团队整合的微服务解决方案：Spring-Cloud-Netflix，新项目可以基于Spring Framework平滑迁移到微服务架构。


### 组件依赖
最终，我们优先采用了以下组件以支持最基本的分布式架构：
* 服务注册与发现 - Eureka，极简服务发现
* 服务负载均衡 - Ribbon，跨数据中心路由和灵活的负载均衡策略
* RPC - 基于Http协议的JSON文本序列化和反序列化
* 服务调用 - Feign，声明式HttpClient
* 服务弹性容错 - Hystrix，资源隔离
* 服务配置 - 基于Git的Spring Config Server

### Spring Cloud版本
#### Angel  
  
第一版的微服务项目依赖的Spring Cloud版本是A系列：`Angel.SR4`，使用下来发现，这根本不是一个Angel，最多算一个原型项目，不像一个框架，扩展性比较差，功能比较简陋，仅能让你的项目运行起来而已。

#### Brixton  
  
很快，我们将Spring Cloud版本升级到B系列：`Brixton.M4`。   
  
从B系列开始，Spring Cloud逐渐演变为一个整合微服务组件的框架，功能更完备，增加了很多扩展点。 2016年5月，Spring团队发布了`Brixton.RELEASE`，不过没多久，就发现一些Bug，SpringCloud官方文档也取消了`Brixton.RELEASE`，目前GA版本为：`Brixton SR6`。  
  
Brixton发布详情，请参考： [Spring Cloud Brixton Release Notes
](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Brixton-Release-Notes)
#### Camden  
  
最新的Spring Cloud GA版本：`Camden SR1`，除了升级组件版本、Bug修复之外，将Netflix Feign切换到 Open Feign，增加了新项目Spring Cloud Contract。  
    
更多发布详情，请参考: [Spring Cloud Camden Release Notes
](https://github.com/spring-projects/spring-cloud/wiki/Spring-Cloud-Camden-Release-Notes)

#### 我们的选择  
  
目前，我们采用的Spring Cloud版本：`Brixton.RELEASE`，这是最早可用的稳定版。  
  
当时我们急需一个稳定版本以便在此基础上推广微服务，保障线上服务的稳定。此版本虽有部分Bug，但不影响使用。

## 编码规范：微服务，how？
确定技术选型之后，接下来的问题是如何实施微服务，包括编码约定、服务划分等。
### 旧的代码结构
传统的单体Web应用，通常由以下几部分构成：
* Model
* View
* Controller
* Service
* Dao
* xml和properties配置文件
* 其他组件

我们可以换种思路看待这种结构：
 ```
 数据 => Dao[数据加载]  
	   => Service [数据转换] 
	     => Model [转换结果] 
	   	=> Controller [路由] 
	   	  => View [页面呈现结果]
 ```
单体应用将`数据的加载、操作、路由、呈现全打包`在一起，而微服务则希望通过`打包数据的加载和操作以实现业务的复用`，为各种客户端（Web、Native App、Wap、Service RPC)提供更灵活的支持。
### 服务划分
所以相对于单体应用，微服务可视为分布式部署的Dao和Service。
在将楼盘字典改造成微服务的过程中，我们积累了以下服务划分的经验：  

#### 按数据库拆分微服务    

  读写同一个数据库的业务可打包到一个微服务里，这里划分的粒度比较粗，相当于将单体应用的Dao和Service直接搬到新项目了。
  
#### 按业务模块或者业务模块的重要性来做隔离   
 
  比如楼盘字典，按核心业务、搜索、统计Job拆分成了三个微服务，搜索是提供全站支持的，即使搜索遇到故障，也不会影响核心业务；各种统计任务相对独立，属于CPU密集型任务，代码变动较少，部署频率小。  
#### 查询类接口只提供原子服务，禁止数据库跨库、甚至交叉访问  
所谓原子服务，也称数据接口，可理解为针对单表的查询，不会跨表关联。
`原子服务是面向数据、针对单表的，不涉及复杂业务逻辑`，因此可保持简单、稳定、高度优化。    
比如，商圈的Model - BizCircle:
```java
  //商圈ID
  private long id;
  //商圈名
  private String name;
  //商圈创建人
  private int cuser;
  //创建人姓名，model里不应该出现此字段
  //private int cuserName
```
我们的业务表不会冗余创建人的名称，即使UI需要呈现创建人名称，原子服务也不应返回创建人的姓名。    

如果客户端需要呈现创建人名称呢？  
为了支撑业务，存在另一种角色：`组合原子接口的服务`。  
  
目前大多数UI项目属于此类角色，它们`偏向业务`，有自己的业务Model，有些业务可能要调用几个不同的原子服务拼接业务数据，大家`可类比成老MVC项目中的业务API`。 
 
比如，为了显示商圈创建人的姓名，组合原子接口的服务的代码可能如下  
``` java
  //原子接口
  @Autowired
  private BizCircleSpi bizCircleSpi;
  //原子接口
  @Autowired
  private UserSpi userSpi;
  //组合原子接口的服务
  public ClientBizCircle findById(long id){
        //原子接口
        BizCircle zc= bizCircleSpi.findByIdV1(id);
        Assert.found(zc!=null,”商圈不存在”);
        //原子服务的Model没有cuserName字段，偏业务接口的Model有此字段
        ClientBizCircle cbc=BeanUtils.copy(new ClientBizCircle(),zc);
        cbc.setCuserName(userSpi.findByUserCode(zc.getCuser()).getUserName());
        return cbc;
  }
```
  
####  数据增删改的接口最好面向业务逻辑  
  针对多张表的数据操作最好放在同一个接口，这样可保证只有一个事务，避免分布式事务，而跨服务的数据交互可使用分布式消息队列。
  
#### 必须跨库关联的业务，可通过数据仓库查询  
各种维度的业务指标统计，可利用数据仓库统一查询。
数据仓库，通俗地讲，订阅了大部分数据库的表，我们的跨库查询SQL仅限数据仓库的查询。

### Coding演化
#### 服务暴露
理论上来说，开发人员可以将微服务当做分布式部署的Dao和Service。  
  
而如何将Service暴露给客户端？ 

Spring Cloud默认以Http协议暴露服务，具体到编码层面，开发人员需编写Rest风格的SpringMVC代码。  
  
`City服务`，如下所示：
``` java
  @RestController
  public class CityController{
     //需要暴露出去的服务
     @Autowired
     private CityService cityService;
     /**
     * 通过Spring MVC将服务CityService暴露给客户端
     */
     @RequestMapping(value=“/city/{id}”,method=RequestMethod.Get)
     public City findById(@PathVariable(value=“id”)int id){
      return cityService.findById(id);
     }
  }
```

服务实现方的Spring MVC Controller在架构上仅充当服务暴露的角色，未来版本可能更换成Netty。

#### 客户端：谁将调用微服务？
编写微服务的另一个要点就是：我们的微服务支持哪些客户端？
目前，公司的项目主要面向以下客户端：
* Web浏览器、Wap
* Apps
* 微服务  

因为我们的微服务已通过Rest风格暴露出来了，不管那种客户端，通过标准的Http协议访问即可，除了一种客户端：微服务。
  
如果一个微服务有5个微服务在调用，那么这5个微服务必须各自写一套类似的HttpClient调用。
  
为了方便微服务之间的调用，我们决定给微服务增加一层：`SDK`。  
  
每个微服务都有自己的SDK，其他微服务可直接依赖特定版本的SDK，客户端无需编写代码，就像`调用本地的Service一样`。

SDK层在我们的微服务实践中被称为：`SPI（Service Provider Interface)`。

SDK层不包含任何业务逻辑，只是服务接口的声明。  

更抽象地说，一个完整的微服务包含两部分：服务声明（SPI)和服务实现（Dao&Service)。  

引入SDK层之后，我们的`City`服务变成了两个Maven模块：
* SDK , city-spi模块
``` java
  /**
   * 只是通过Rest暴露出来的服务接口的声明，不包含任何业务逻辑
   */
   @FeignClient(“city-server”)
   public interface CitySpi{
          @RequestMapping(value=“/city/{id}”,method=RequestMethod.Get)
     public City findById(int id);
   }
```
* 服务，city-server模块
``` java
  @RestController
  public class CityController implements CitySpi{
     //需要暴露出去的服务
     @Autowired
     private CityService cityService;
     /**
     * 通过Spring MVC将服务CityService暴露给客户端;
     * 但是注意，注解都继承自接口了（这是我们提供的扩展功能）
     */
     public City findById(int id){
      return cityService.findById(id);
     }
  }
```  
      
客户端示例（依赖City服务的微服务） 
POM
``` xml 
	   <dependency>
			<groupId>com.lianjia.sh</groupId>
			<artifactId>city-spi</artifactId>
			<version>1.0.0</version>
		</dependency>
```
Java 代码
``` java
  @Service
  public class OMGService{
     //依赖特定版本的city服务
     @Autowired
     private CitySpi citySpi;
     /**
     * 其他微服务
     */
     public Model doSomeStuff(int id){
      Model model=xxxDao.findById(id);
      //就像调用本地的service一样
      City city=citySpi.findById(model.getCityId());
      //其他业务逻辑
      return model;
     }
  }
```

#### SPI编码规范的演进  
在微服务落地过程中，我们的SPI逐渐形成了一套约定的代码规范：[接口和方法规范](spi-interface.html)。

第一版

``` java
  @FeignClient(“loupan-server”)
  public interface IBizCircleService{
       @RequestMapping(value=“/bizcircle/{id}”,method=RequestMethod.GET)
       public BizCircle findById(@PathVariable(value=“id”)int id);
  }
```
最早版本的SPI接口命名，遵守的是Java接口的命名约定：首字母大写I+业务类+Service，比如：`IDistrictService`。
但是实践下来，容易和服务实现方的Service命名混淆。
比如City服务，服务声明为：
``` java
    public interface ICityService{
    }
```
服务实现方的代码可能如下：
``` java
  public class CityController implements ICityService{
     //这个Service和ICityService是没有关系的。
     @Autowired 
     private CityService citySerivce;
     
     public City findById(int id){
      return cityService.findById(id);
     }
  }
  
  @Service 
  public class CityService {
  	public City findById(int id){
      return cityDao.findById(id);
     }
  }
```
而且客户端注入Service时生成的变量名，首字母’I’看起来有点怪（单薄？)。  
实际上这种类名生成的变量名：`Introspector.decapitalize("ICityService”)`，输出依旧是：ICityService， 或者你改成：icityService? iCityService?
``` java
   @Service
   public class ClientService{
     @Autowired
     ICityService iCityService;
   }
```

为了增加SDK层的辨识度，我们决定更改接口命名约定：`所有接口名使用其他后缀（不采用Service作为后缀），取消首字母‘I’`。  
  
当时有三个候选后缀：Api 、Client、Spi，我们的接口可能变为：CityApi、CityClient、CitySpi，最终选择了后缀：`Spi`，据说是因为`Spi`听起来逼格高。

除了SDK层的辨识度之外，另一个最大的问题，就是接口方法的版本隔离。  

第二版  
``` java
@FeignClient(“loupan-server”)
public interface BizCircleSpi{
       @RequestMapping(value=“/v1/bizcircle/{id}”,method=RequestMethod.GET)
       @LoginNeedless
       public BizCircle findByIdV1(@PathVariable(value=“id”)int id);
  }
```
第二版主要解决接口的版本控制、是否登录、SDK层的辨识度。  
  
Rest接口的版本控制，大家可参考文档：[REST API版本控制](/parts/chapter1/restful-api-versioncontrol.html)。 
  
为了实现接口方法的版本控制，我们约定：Url路径以版本号为前缀、方法名以版本号为后缀。  
  
举个例子， Url路径 => 接口方法名 ：  
`/v1/bizcircle/{id} => findByIdV1`，`/v2/bizcircle/{id} => findByIdV2`。

同时，为了保证接口请求的安全性，我们引入注解：`@LoginNeedless`。

至于为啥不叫`@Logined` （登录），而是叫`@LoginNeedless`（无需登录），传说是为了责任认定：  
	默认所有接口都是要登录的，不需要登录的肯定是某个程序猿`手动`添加的，由此可认定是该程序猿的有意行为。  
  
第三版
``` java

@FeignClient(“loupan-server”)
public interface BizCircleSpi{
       @RequestMapping(value=“/v1/bizcircle/{id}”,method=RequestMethod.GET)
       @LoginNeedless
       @LorikRest(value = {Feature.NullTo404},codes={21000})
       public BizCircle findByIdV1(@PathVariable(value=“id”)int id);
  }
```
第三版主要解决不同客户端调用时Rest语义上的兼容以及引入业务码。  
  
Rest语义兼容，目前只有：`Feature.NullTo404`，即Rest请求接口返回Null时，服务提供方应该响应`404`。  
  
SPI调用
``` java
   public class ClientService{
     @Autowired
     private BizCircleSpi bizCircleSpi;
     
     public void doSomeStuff(int id){
            //SPI访问微服务，如果ID对应的商圈不存在，返回的是null
            BizCircle bc=bizCircleSpi.findByIdV1(2000);
            //null 对于java代码来说就是不存在
     }
   }
```  

但是通过HttpClient调用    

``` json
 //如果商圈ID=2000不存在，接口会返回Http Status=200，
 //只不过Response Body为空字符串，服务提供方不会序列化null。
 //按Rest语义来说，此处应该返回404：没有找到该资源，
 //而不是status=200，表示资源存在。
 $.get( 
	”http://api.route.dooioo.com/loupan/server/v1/bizcircle/2000”  
	);
```

业务码（`BizCode`）主要用于业务流程异常时对客户端的反馈，一般配合断言(`com.dooioo.se.lorik.core.web.result.Assert`)使用。  
  
业务码主要由两个字段：
* code  业务码值，整型。
* message 业务码值的说明。

举个业务码的例子：
``` java
 public interface LoupanBizCode{
   BizCode NO_PRIVLEGE=new BizCode(20000,”您没有操作权限”);
 }
```
服务实现方  
  
``` java
 public class BizCircleController implements BizCircleSpi{
   @Autowired
   private BizCircleService bizCircleService;  

   public BizCircle findByIdV1(@RequestHeader(int loginUserCode,int id){
    return bizCircleSerivce.findById(loginUserCode,id);
   }
 }
 
 @Service
 public class BizCircleService{
  //服务
  public BizCircle findById(int loginUserCode,int id){
    //配合com.dooioo.se.lorik.core.web.result.Assert
    //抛出业务码：没有权限，其实是运行时异常，中断业务流程
   Assert.authorized(userSpi.hasPrivileV1(loginUserCode,  
“can_view_bizcircle”,LoupanBizCode.NO_PRIVLEGE);
   //正常业务流程
   return bizCircleDao.findById(id);
   }
 }
```

客户端Spi调用
``` java
@Service
public class ClientAnyService{
   @Autowired
   private BizCircleSpi bizCircleSpi;
   
   public void doSomeStuff(int id,int userCode){
    try{
    	 //lorik-core版本：1.1.7.1，lorik-spi-view版本：2.1.7
         //Spi接口，如果想获取业务码，就得使用Try - Catch
    	 BizCircle bz= bizCircleSpi.findByIdV1(userCode,id);
    }catch(Exception e){
      //获取业务码
      RestResult bz= BizCodes.from(e);
      //bz.getCode() = 20000; //业务码
      //bz.getMessage() = “您没有操作权限”;//业务码说明
    }
    
   }
}
```

Rest方式请求接口，如果没权限，业务码输出：
``` json
{
  “code”: 20000,
  “message”: “您没有操作权限”,
  “path”:”/v1/bizcircle/20”,
  “timestamp”:1342324324364
}
```


## 配置
## 性能
## 高可用
## 监控：多维度Metric 
## 测试：单元测试
## 文档：接口文档自动化
## 代码：自动生成、编码理念、响应式编程
## 调试：基于Http RPC
目前版本的微服务仅支持基于Http协议、文本序列化（JSON)的RPC调用。
### 调用方（客户端）
服务调用方最常用的调试，是查看RPC调用时构造的Http请求以及服务提供方的响应。  
  
#### 核心类：SynchronousMethodHandler
最核心的代码，在这个类：`feign.SynchronousMethodHandler`。

以下是RPC请求的核心代码，IDE Debug模式可查看方法调用时的实参：
```java
 @Override
  public Object invoke(Object[] argv) throws Throwable {
    // argv 是方法调用的实际参数，Feign根据实际参数替换Http模板请求的占位符
    // 占位符包括：@RequestParam、@RequestHeader注解的参数
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    //请求重试？
     Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        //发起Http请求，同时解码（反序列化）响应
        return executeAndDecode(template);
      } catch (RetryableException e) {
        //是否需要重试（重新请求） 或者抛出错误（结束方法调用）？
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```  
 
#### Http请求和反序列化
Http请求和反序列化的实现，忽略部分代码：
``` java
Object executeAndDecode(RequestTemplate template) throws Throwable {
 	//构造Http请求，这里可以查看构造好的Http请求
 	//Uri、Http Request Header、Request Body等
    Request request = targetRequest(template);
    Response response;
    try {
      //可以查看使用了那种HttpClient，默认基于JDK的URLConnection
      //新版本的lorik-spi-view使用Apache Http Client。
      //options是请求的一些选项，比如请求超时时间、读取超时时间
      response = client.execute(request, options);
    } catch (IOException e) {
       //忽略代码：请求出错，直接抛出异常
    }
   try {
      //服务端响应2xx
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
         //解码（JSON反序列化）Response Body，这里可查看服务端的响应的信息
          return decode(response);
        }
      } else if (decode404 && response.status() == 404) {
        //如果客户端配置了解码404，
        //则解码响应信息；否则使用错误解码器统一处理。
        return decoder.decode(response, metadata.returnType());
      } else {
       //如果服务端响应3xx以上状态码，
       //使用错误解码器解码服务端响应的Response Body
       //我们lorik-spi-view就是通过自定义的SpiErrorDecoder
       //将服务提供方的业务码传播到客户端（调用方）
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      //忽略异常处理和流关闭
    }
  }
```

### 服务提供方
服务提供方是基于SpringMVC实现的Rest服务，最常见的调试就是获取服务接口的请求参数。

#### 核心类：ServletInvocableHandlerMethod
最核心的代码，在这个类：`ServletInvocableHandlerMethod`。

以下是完成Request => Response转换的核心代码：

``` java
public void invokeAndHandle(ServletWebRequest webRequest,
			ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
		//完成方法调用：request参数转换为Java Method的参数
	    //通过Java反射method.invoke(obj,args)获取返回值
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		//设置响应的http状态码
		//这个类构造函数会尝试根据@ResponseStatus初始化http状态码
		setResponseStatus(webRequest);

		//如果方法调用返回null，则标识请求已处理。
		//其实是为了标识这个请求已不需要序列化returnValue了。
		if (returnValue == null) {
		  //忽略相关代码
			mavContainer.setRequestHandled(true);
			return;
		}
		//方法返回值了，进一步处理，因为方法可能返回的各种类型的返回值
		//比如ResponseEntity、Callable、HystrixCommand、Model
		mavContainer.setRequestHandled(false);
		try {
		   //HandlerMethodReturnValueHandler专门用来序列化方法的返回值   
		   //有兴趣的同学可以查看下所有子类，包括如何处理异步。
		   this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
		  //接口返回值序列化出错了，抛出异常
		  throw ex;
		}
	}
```

#### 接口请求参数 => 方法参数转换
实现接口请求参数 => 方法参数转换的代码：
``` java
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//method的所有参数
		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			//设置参数名，根据参数名去Request中查找参数值。
parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			//解析参数类型
			GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
//所有Http参数 => 方法参数的转换都是通过HandlerMethodArgumentResolver
//有兴趣的同学可查看所有子类，了解下@RequestParam、@RequestHeader等所有
//SpringMVC相关的Http注解如何解析和转换为方法实参的
						if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
				   //找到合适的Resolver用来解析方法参数。
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
				throw new IllegalStateException(msg);
			}
		}
		return args;
	}
```

#### 在线诊断：Trace Endpoint
SpringBoot Actuator默认提供了调试Endpoint: http://host:port/admin/trace。

接口响应如下：

``` json
[
	{
		timestamp: 1477298109627,
		info: {
				method: "GET",
				path: "/v1/company/3/gbCode",
				headers: {
					 request: {
						x-client-id: "10.22.16.35:9500",
						x-client-name: "loupan-search",
						x-client-security-token: "66625864-cae3-4492-bfa6-085de365abf8",
						accept: "*/*",
						user-agent: "Java/1.8.0_40",
						host: "10.22.16.36:9600",
						connection: "keep-alive"
				      },
					 response: {
					  	 X-Instance-Id: "b7b0fc1f593866afe230f4af6117b5ee",
						 X-Application-Context: "loupan-server:production:9600",
						 Content-Type: "application/json;charset=UTF-8",
						 Transfer-Encoding: "chunked",
						 Vary: "Accept-Encoding",
						 Date: "Mon, 24 Oct 2016 08:35:09 GMT",
						 status: "200"
			}
		}
	}
}
]

```

Trace Endpoint只记录最近的100条接口请求，根据请求的时间戳倒排，

由于此接口会泄露敏感信息，尤其是UI项目，生产环境建议禁用此功能（在配置文件中增加property:`endpoints.trace.enabled:false`)。


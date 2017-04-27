<!-- toc -->
## Since 2010
典型项目结构：
* Model
* View(jsp/freemarker/html)
* Controller
* Service
* Dao
* xml和properties配置文件

我们可以换种思路看待这种结构：

 ```
 数据 => Dao[数据加载]  
	   => Service [数据转换] 
	     => Model [转换结果] 
	   	=> Controller [路由] 
	   	  => View [页面呈现结果]
 ```

单体应用将`数据的加载、操作、路由、呈现全打包`在一起，而微服务则希望通过`打包数据的加载和转换以实现业务的复用`，为各种客户端（Web、Native App、Wap、Service RPC)提供更灵活的支持。

## 微服务？

微服务，简而言之，就是可分布式部署的Dao和Service;

你可以花费2分钟写一个OMS微服务：

``` java
  @Service
  public class EmployeeService{
    @Autowired
    private EmployeeDao employeeDao;
  	
  	public Employee findByUserCode(int userCode){
  	  return this.employeeDao.findByUserCode(userCode);
  	}
  }
```
### 服务暴露
现在，你将面临一个问题，客户端该怎么访问OMS服务呢？

我们称之为如何将`微服务暴露`出去，最终我们选择基于SpringMVC、以Rest风格对外暴露服务。

微服务将面向：IOS/Andriod App、Wap、Web、Node.js等客户端，基于Http Rest的json payload更方便不同平台的接入和测试（包括自动化测试）。

``` java
@RestController
public class EmployeeController{
	 @Autowired
	 private EmployeeService employeeService;
	
	 @RequestMapping(value="/employee/{userCode}",method=RequestMethod.GET)
	 //JAX-RS： @GET @Path("/employee/{userCode}") @PathParam("userCode")
	 //Spring Http Remoting  Or Netty : any non-invasive solution?
	 public Employee findByUserCode(@PathVariable("userCode")int userCode){
	 	return this.employeeService.findByUserCode(userCode);
	 }
} 
```

这里强调一点： RestController不应涉及Java Servlet API，比如 `HttpServletRequest/Response`、`HttpSession`。
 
至此，一个简单的微服务编写完毕，于是你找运维申请了服务器和域名，开始上线：
 `http://oms.dooioo.com => 10.22.16.201:8080,10.22.16.202:8080`

此时，应用程序架构如下图所示：
![应用架构1]({{book.imagePath}}/parts/chapter5/images/arch-level-0.png)

### 客户端接入
 听说OMS服务已上线，于是你的钉钉开始收到一系列轰炸：
 
   * 那个xx接口Url Path是什么？
   * 那个xx接口的参数都有啥？什么类型？各个参数什么意思？
   * 接口返回值是什么意思？接口返回那些字段？
   * 。。。。。。。。。。。。。。。。。。。。。。。。。。。
 

你已经不堪其扰，于是思考能不能做到接口文档自动化？
 
#### 微服务文档自动化 
 
 或许你会去了解Swagger之类的API文档生成组件。。。
 
 最终，我们找到了微服务文档自动化的方式：基于Java Doc注释自动生成Rest接口文档。
 
 我们只需稍微调整服务暴露层的代码注释就可以实现：
 
 ``` java
 /**
 * @summary 员工
 */
@RestController
public class EmployeeController {

  @Autowired
  private EmployeeService employeeService;

  /**
   * 查询在职的正式、试用期的员工
   * @param userCode 员工工号
   * @summary 根据工号查询在职员工
   * @author Huisman
   * @version v1
   * @since 2017年3月23日 上午11:34:30
   */
  @RequestMapping(value = "/employee/{userCode}", method = RequestMethod.GET)
  // JAX-RS： @GET @Path("/employee/{userCode}") @PathParam("userCode")
  // Spring Http Remoting Or Netty : any non-invasive solution?
  public Employee findByUserCode(@PathVariable("userCode") int userCode) {
    return this.employeeService.findByUserCode(userCode);
  }
}

```

自动生成的文档如下：

![自动生成接口文档]({{book.imagePath}}/parts/chapter5/images/api_doc_demo.png)

 
你的世界又恢复了平静，for now。

#### SDK层
暴露出来的OMS微服务可使用任何HttpClient来请求，比如Jersey：
``` java
 javax.ws.rs.client.target("http://oms.dooioo.com")
	 .path("/employee/{userCode}")
	 .resolveTemplate("userCode",80001)
	 .request(MediaType.APPLICATION_JSON_TYPE)
	 .get(Employee.class); 
```
由于OMS是基础微服务，不久，很多Java项目开始大量使用，另外，有些项目会同时依赖几个微服务，各个团队不断重复类似的请求代码。

对客户端来说，如果OMS微服务和本地Service使用方式一致，那会极大简化Java客户端接入的成本；

这对这种情况，我们决定在微服务上增加一层：SDK，这层代码必须保持极简、纯粹、不包含业务逻辑。

SDK层在内部，我们称之为`SPI`（Service Provider Interface)，具体编码层面，使用Netflix Feign来实现。

``` java
@FeignClient(url="http//oms.dooioo.com")
public interface EmployeeSpi {

 /**
  * 根据工号查询在职员工
 * @param userCode 工号
 */
 @RequestMapping(value = "/employee/{userCode}", method = RequestMethod.GET)
 public Employee findByUserCode(@PathVariable("userCode") int userCode);
}
```
SDK层是单独的一个Maven模块，只包括model和微服务接口的声明，不存在业务逻辑，最终打包成一个独立的jar。

比如，OMS微服务，它对外提供的SDK层jar如下：

``` xml
 	<dependency>
			<groupId>com.lianjia.sh.se</groupId>
			<artifactId>oms-spi</artifactId>
			<version>1.0.15</version>
	</dependency>
```

任何Java客户端，想接入OMS微服务，只需要依赖此oms-spi-1.0.15.jar，就可以像注入本地服务一样引用OMS微服务：

``` java
   //客户端访问oms 微服务
   @Service
   public class XXService{
    @Autowired
    private EmployeeSpi employeeSpi;
   public Model findById(int bizId){
       Model bizBean=this.xxxLocalService.findById(bizId);
       Employee createUser=this.employeeSpi.findByUserCode(bizBean.getCreateUser());
	//....其他
     }
  }
```

此时，应用程序的架构如下：
![应用架构2]({{book.imagePath}}/parts/chapter5/images/arch-level-1.png)
  

通常来说，我们的微服务都是Maven多模块项目。
#### 服务降级与限流
Fail Fast： Hystrix/RateLimiter/always Timeout...Timeout...Timeout.....

#### Boundary
![API网关Family]({{book.imagePath}}/parts/chapter1/images/api-route-deploy-abstract.png)

接入层： API 网关 ， ORoute OAuth Server

跨域？json callback?
  
### 服务划分
 在将楼盘字典改造成微服务的过程中，我们积累了以下服务划分的经验：  
#### 微服务 ≠  API
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

 
### 扩展：服务发现
 引入SDK层之后，客户端接入更方便了，不久之后，OMS微服务经常在高峰时段收到响应延时的报警短信。
 查看Zabbix tomcat线程数，线程数最近几周一直在增加的趋势，你决定横向扩展OMS微服务，再增加两个节点。
 
 经过填写扩容申请单、运维增加节点并调整Nginx域名指向，OMS微服务又恢复了生机。
  
 随着越来越多的微服务被创建出来，域名的申请、调整nginx配置也成了一个负担。
 
 另外，微服务大多按请求高峰期来申请节点，会造成很多资源浪费。
 
 * 如果我们想实现按需服务、节点弹性伸缩？
 * 如果我们想实现灰度发布？
 * 如果我们想监控分散在各个角落的节点，比如实时流量查看？
 * 如果我们想将微服务通过TCP协议暴露出去？

为了增强对分布式服务的治理能力，我们引入服务注册与发现组件： Eureka。 
OMS微服务只需要在配置文件里添加以下property，就可以自动注册：

``` json
eureka:
    instance:
        appGroupName: oms
        instanceId: ${spring.cloud.client.ipAddress}:${server.port}
```

### 实时流量

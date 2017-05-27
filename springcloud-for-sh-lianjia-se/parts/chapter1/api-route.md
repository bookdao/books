<!-- toc -->

### 认证
所有接口文档（[http://api.doc.dooioo.cn/doc](http://api.doc.dooioo.cn/doc)）中出现***需要授权***的Label时，接口请求都必须传递Http Request Header: `X-Token`。

![Login Label]({{book.imagePath}}/parts/chapter1/images/login_label.png)


#### 申请X-Token
API网关负责统一认证，所谓认证，是指调用SSO登录服务验证`X-Token`，认证通过后，API网关自动添加Http Request Header： `X-Login-UserCode`和`X-Login-Company-Id`，将员工登录信息传递给后台服务。

`X-Token`可通过工号和密码调用SSO登录接口获得，接口响应的`loginTicket`即`X-Token`的值，具体接口文档请参考：
[SSO登录服务 - 员工登录 (员工登录)
](http://api.doc.dooioo.cn/v1/doc/3606983534/3255124964/1626426016)

注意事项：
* 多次调用SSO登录接口时，如果`Source`相同，则旧的loginTicket会失效，`30分钟`内`旧loginTicket`会提示重复登录或者登录状态异常。
* `loginTicket`目前所有平台（Source)的有效期是两小时，携带`X-Token`访问任何接口都会`自动顺延`loginTicket的过期时间。
* 通常情况下，只需调用一次登录SSO接口获取loginTicket，之后可通过刷新Ticket的方式延长loginTicket的过期时间。

#### Web页面跨域访问授权接口
众所周知，授权接口都需要传递Http Request Header ：`X-Token`，为了支持Web页面跨域访问授权接口，
Ajax请求必须携带登录cookie，代码示例(集成环境)如下：
``` javascript
//需要授权的，JQuery1.5.1+，低版本有bug
$.ajax({
   type:'GET',
    xhrFields: {
      withCredentials: true
   },
   url:'http://aroute.dooioo.cn/loupan/management/server/v1/organization/coordinates/paginate?orgId=&pageNo=1&pageSize=20',
   dataType:'json'
   })
   .done(function( data ) {
    if ( console && console.log ) {
      console.log( "Sample of data:", data );
    }
  });
```
Web页面请求授权接口的前提条件：
1. 员工登录了当前页面（生成了登录SSO cookie: `login_ticket` );
2. Ajax请求设置了请求属性： `withCredentials: true`;

注意事项：`无须授权的接口（可公开访问的接口），请不要设置Ajax请求属性withCredentials`。

#### 网关负责认证，业务方负责授权
API网关只负责认证，授权则交给每个服务提供者。    
  
也就是说，后端服务拿到的当前登录人（`X-Login-UserCode`和`X-Login-Company-Id`）API网关可以保证是合法的、不会被客户端伪造，至于员工能否查看某个数据，则交给业务方来处理。  
  
如果业务方编码不当，则可能出现越权访问。

#### API网关常见业务码
通过API网关访问授权接口，`X-Token`非法时，会响应以下业务码：  
![api route bizcode]({{book.imagePath}}/parts/chapter1/images/apiroute_bizcode.png)

### 路由
API网关是访问机房服务的桥梁，而能否访问某个服务，则取决于网关是否有此服务的路由信息。 

路由，可理解为虚拟路径和主机IP的映射规则：
```
   # 如果请求路径前缀是：/ky/old/server ，则路由请求到主机：http://10.8.1.204:1080
  /ky/old/server =>  http://10.8.1.204:1080
  
  # 如果请求路径前缀是：/loupan/server，则路由请求到主机：http://loupan.dooioo.com
  /loupan/server => http://loupan.dooioo.com
```


路由目前分为静态路由和动态路由。

#### 动态路由
所谓动态路由，就是通过服务发现，自动建立服务名到IP的映射规则，无需人为处理。  
  
目前，我们使用Eureka来实现服务的自动发现，大家可以访问：[http://discorvery1.se.dooioo.com](http://discovery1.se.dooioo.com)查看所有注册的服务。
  
动态路由将虚拟路径映射为虚拟主机名，请求路由时，根据**虚拟主机名**去Eureka里查询所有节点IP：端口，再负载均衡到某个节点上。

而虚拟主机名通常为 FeignClient的value，比如： `@FeignClient(“loupan-management-server”)`;

动态路由的映射规则示例：
```
   #如果请求路径前缀是：/loupan/management/server ，则路由请求到虚拟主机：loupan-management-server，
   #实际路由时，组件根据虚拟主机名去Eureka里查询所有节点的IP:端口，负载均衡到某一个IP上。
  /loupan/management/server =>  loupan-management-server
```
#### 静态路由
静态路由，是指需要管理员手动维护的映射规则。  
通常是为了方便测试或者将老系统接入API网关。

静态路由虚拟路径的命名规范： /产品线/应用/server，比如：/ky/tel/server，表示客源产品线的电话服务。

静态路由的映射规则（path=> location)示例：
```
   #如果请求路径前缀是：/ky/tel/server ，则路由请求到主机：http://tel.ky.dooioo.com:80
  /ky/tel/server =>  http://tel.ky.dooioo.com
```

##### 添加静态路由
测试和集成环境，开发和测试人员可自由添加静态路由，方便调试。  
接口说明文档请参考：[API网关 - 管理支持 (新增或更新静态路由)
](http://api.doc.dooioo.cn/v1/doc/3100800167/194699906/292285745)

此接口需要Header参数：`X-Role-Id`，测试和集成环境有效值为：admin。
`path`参数即虚拟路径，`location`为服务访问的域名。

生产环境的静态路由，请在上线之前钉钉发给我。

#### 查询可用路由
开发和测试人员可通过接口： [http://aroute.dooioo.com/admin/routes.json](http://api.aroute.dooioo.com/admin/routes.json) ，查询当前可用路由信息。

集成环境将.com调整为.cn，测试环境将.com调整为.net。

### SpringMVC&SpringBoot项目接入API网关

#### 接口请求隔离
目前我们正在限制API接口请求方的IP - 接口请求隔离，下面我以房源接口为例，介绍如何隔离接口请求：
1. 假设客户端（IP：10.8.1.22）请求了房源接口：`http://fang.dooioo.com/api/v2/house/1234`，房源部署在IP为`192.168.0.22`的主机上。  

2. 请求到达Nginx后，Nginx判断Request Path是否以`/api` 开头，如果是，则继续判断客户端IP是否属于`机房网段`（IP段白名单），如果客户端IP不属于IP白名单，Nginx直接响应Http状态码：`403`，否则请求转发至192.168.0.22的房源节点。  

3. 网段隔离后，所有应用的接口（http：//domain.dooioo.com/api/**)`仅限服务端通过HttpClient调用`，门店和总部员工通过浏览器、Postman、编写代码均无法请求接口。  

4. Web页面或测试人员访问应用接口，统一通过API网关代理。  
 在此之前，应用须向API网关添加自己的静态路由信息。  
 比如房源的静态路由：  
	`/fy/old/server` => `http://fang.dooioo.com` ，即 path => location 的映射。  
 之前Ajax或Postman里的接口请求地址：  
`http://fang.dooioo.com/api/v2/house/1234`  
通过API网关访问时调整成：  
`http://aroute.dooioo.com/fy/old/server/api/v2/house/1234`     

5. 服务端程序调用房源接口时，通过应用的域名访问：`http://fang.dooioo.com/api/v2/house/1234`。  

6. 简而言之，所有应用的接口（http ://domain.dooioo.com/api/**)，服务端可通过应用域名访问，非服务端的接口调用只能通过API网关代理。

##### 注意事项
1. 如果接口路径不以”/api”开头，则不会限制调用方的IP，比如：`/user/api/**`、`/v1/api/**`，局域网内的任何设备、任何人都可以访问。  
   出于数据安全性的考虑，请尽量将接口改造成`/api/**`，新增接口的路径前缀请设置为`/api/`。  
	注意，此处所说的接口：
	`仅限提供给其他系统调用的数据接口，自己项目使用的ajax接口可通过登录SSO拦截器拦截`。  
   
2. 接口调用方的IP限制策略默认不启用，必须向`运维报备`后才启用。   
 
3. 接口网段隔离的策略，请务必在测试、集成环境调试通过以后，再上线生产环境。
4. 老项目不以`/api`为前缀的接口路径重构为`/api/**`，服务端调用仅需调整接口路径，域名保持不变，非服务端（页面或测试）则通过API网关访问。  

5. 老项目的改造流程示例:  
	* 原接口： `http://xiaoqu.dooioo.com/house/api/v2/nearby`。  
	内网的员工都能访问，出于安全性考虑，我们将接口按约定前缀`/api`重构为：`http://xiaoqu.dooioo.com/api/house/v2/nearby`。   
   
    * 项目新增接口拦截器：[AuthorizedRequestInterceptor](authorize-interceptor.html)，拦截应用接口的所有调用，但是注意，登录SSO拦截器不能拦截接口。  

	* 钉钉联系运维部的小伙伴，申请开启接口调用的网段隔离，先调整测试和集成环境的Nginx配置，测试通过之后，上线时通知运维部调整你的应用生产环境的Nginx配置。  

	* 发邮件（或钉钉）给使用此接口的团队，告知新的接口路径，说明启用接口请求隔离，老接口会保留一段时间（提供明确的deadline）。  
	  如果不知道谁在调用你的接口，可协助运维部小伙伴调整Nginx、记录接口的访问日志。  

	* 如果有Web页面访问此接口，找API网关的项目负责人，添加应用的静态路由，并告知Web页面调用方新的接口路径。  
	  比如你应用的静态路由：/loupan/xiaoqu/server => http://xiaoqu.dooioo.com，
	   则页面请求路径调整为：http://aroute.dooioo.com/loupan/xiaoqu/server/api/house/v2/nearby 。 
	
	

#### API网关接口路径的组成
通过API网关访问接口时路径由三部分组成：`API网关域名`+`应用注册的虚拟路径`+`原始接口路径`  
  API网关域名：`http://aroute.dooioo.com`  
  应用注册的虚拟路径： `/fy/old/server`  
  原始接口的路径： `/api/v2/house/1234`  
  完整的请求路径：`http://aroute.dooioo.com/fy/old/server/api/v2/house/1234`

#### 接口数据的安全：登录 OR 无须登录
通常情况下，登录SSO拦截器（基于工号和密码）是不会拦截数据接口的。  
因为调用方通常是服务端程序，比如定时任务，基于工号和密码的权限验证体系无法适用。

这会导致一个问题：任何知道接口地址的内网员工都能请求该接口，尤其是涉及到利益分配或者员工隐私的接口。
##### 限制调用方IP

为了增强接口调用的安全性，我们决定限制接口（`/api/**`）调用的IP来源：
`机房网段的IP地址都属于白名单，非机房网段的IP禁止通过应用域名访问接口，只能通过API网关访问原始接口`。

##### 引入接口拦截器，敏感接口进行登录认证
同时，配合API网关，接口提供方也增加了类似登录SSO拦截器的[接口拦截器](authorize-interceptor.html)，请自己Copy源码，并配置拦截所有接口。  
  
如果你的接口无须登录，可公开访问，请在@Controller的方法上添加注解：`@LoginNeedless`；而无此注解的接口默认都会校验请求是否是登录员工发出的。

要求登录的接口，可通过Http Header: `StandardHttpHeaders.X_Login_UserCode` 和 `StandardHttpHeaders.X_Login_CompanyId`获取当前登录人的工号和公司ID。

 以下是老项目的编码示例，仅供大家参考和理解：
 ``` java
 /**
 * 无需登录身份认证的接口请手动添加注解：@LoginNeedless，默认接口都需要登录认证的<br>
 * 请确保你的接口都是以‘/api/’为前缀的，另外记得找运维启用接口调用的IP来源限制。
 */
@RestController
@RequestMapping(value = "/api/salary")
public class SalaryApiController {

  /**
   * 此处的安全要点是，此接口仅能查看接口请求者自己的工资，不能随便传一个工号就能访问别人的。<br>
   * 这个接口没有添加注解：@LoginNeedless，因此可以保证接口请求方肯定是已登录的员工。
   * 
   * @author huisman
   * @version v1
   * @param loginUserCode 身份认证信息：接口请求方的工号（登录的工号），同时也是业务参数
   * @param loginCompanyId 身份认证信息：接口请求方登录时所选公司ID（存在多公司）
   * @since 2016年10月20日
   * @summary 查询自己的工资
   */
  @RequestMapping(value = "/v1/mySalary", method = RequestMethod.GET)
  public Salary mySalary(
      @RequestHeader(StandardHttpHeaders.X_Login_UserCode) int loginUserCode,
      @RequestHeader(StandardHttpHeaders.X_Login_CompanyId) int loginCompanyId) {

    // 业务授权： oms jar去根据loginUserCode和公司ID去判断是否有权限查看userCode的工资
    // Employee emp= employeeService.findUserPrivilege(loginUserCode);
    // ! emp.hasPrivilege("view_my_salary_only")

    // 业务逻辑，当前登录工号本身是业务逻辑的一部分
    // return salaryService.findByUserCode(loginUserCode);
    return new Salary();
  }

  /**
   * 这个接口和上一个接口最大的不同是存在两个工号：loginUserCode和userCode。<br>
   * 为啥会有两个工号呢，userCode是实际业务中需要的参数，代表想查看谁的工资，<br>
   * 而loginUserCode代表接口请求方身份，是一个合法的登录员工，不是一个离职、账号被屏蔽等员工状态异常的账号。<br>
   * 这个loginUserCode业务接口通常用来检查接口请求者的权限，判断loginUserCode是否有权限查看userCode的工资。<br>
   * 比如有的接口的创建人、修改人都可以是loginUserCode，接口提供者可判断loginUserCode是否可以操作数据。
   * 
   * @author huisman
   * @version v1
   * @param loginUserCode 身份认证信息：接口请求方的工号（登录的工号）
   * @param loginCompanyId 身份认证信息：接口请求方登录时所选公司ID（存在多公司）
   * @param userCode 业务参数，想查询谁的工资信息，工号
   * @since 2016年10月20日
   * @summary 查看任何人的工资
   */
  @RequestMapping(value = "/v1/anyoneSalary", method = RequestMethod.GET)
  public Salary anyoneSalary(
      @RequestHeader(StandardHttpHeaders.X_Login_UserCode) int loginUserCode,
      @RequestHeader(StandardHttpHeaders.X_Login_CompanyId) int loginCompanyId,
      @RequestParam(value = "userCode") int userCode) {
    // 业务授权： oms jar去根据loginUserCode和公司ID去判断是否有权限查看userCode的工资
    // Employee emp= employeeService.findUserPrivilege(loginUserCode);
    // ! emp.hasPrivilege("view_anyone_salary")

    // 业务逻辑，当前登录人的工号不会用于业务，只为了判断请求方是否有权限操作数据
    // return salaryService.findByUserCode(userCode);
    return new Salary();
  }


  /**
   * 这个接口没有用Header来接收登录人的工号，没用使用loginUserCode来进行业务授权。<br>
   * 它的应用场景：业务方只是想确保接口是被一个正常登录的员工请求的，<br>
   * 也就是说唯一能访问此接口的人肯定是用工号和密码登录过系统的。
   * @author huisman
   * @version v1
   * @param userCode 业务参数，工号
   * @since 2016年10月20日
   * @summary omg
   */
  @RequestMapping(value = "/v1/omg", method = RequestMethod.GET)
  public int omg(@RequestParam(value = "userCode") int userCode) {
    // 业务逻辑
    // 我只想确保我的请求方都是正派人，实名认证过的。
    return userCode;
  }
  
   /**
    * 无需登录的接口示例，任何人都可访问，请注意：接口任何参数名不能出现X-Login-UserCode和X-Login-CompanyId。<br>
    * 必须手动添加@LoginNeedless注解，敏感接口请不要添加此注解。
    * @author huisman
    * @version v1
    * @param userCode 业务参数，工号
    * @since 2016年10月20日
    * @summary oreally
    */
  @LoginNeedless
  @RequestMapping(value = "/v1/oreally", method = RequestMethod.GET)
  public int oreally(@RequestParam(value = "userCode") int userCode) {
    // 业务逻：我是公开数据，谁都能查看，有求必应。
    return userCode;
  }

  public static class Salary {
  }
}
 ```

 假设以上代码所在应用的访问域名：http: //salary.dooioo.com，我们在API网关登记的静态路由是： /oa/salary/server => http://salary.dooioo.com。    

我们对比下服务端调用和非服务端调用，请求构造上的区别：   
  
授权接口（需要登录的接口）：查看任何人的工资
 ``` java
   # 通过API网关访问
  requst url: 
     http://aroute.dooioo.com/oa/salary/server/api/salary/v1/anyoneSalary?userCode=8989989
  Request Header: 
  		 X-Token:c71edad37eb20f300639496ba7b28d20
  		 
  #服务端调用
  requst url: 
     http://salary.dooioo.com/api/salary/v1/anyoneSalary?userCode=8989989
  Request Header: 
  		 X-Login-UserCode:8781213412
  		 X-Login-CompanyId:1
  		 
   
 ```  

需要登录的接口，两种请求方式的区别：
   * API网关访问接口时，无需传递`X-Login-UserCode`和`X-Login-CompanyId`，只需要指定Header: `X-Token`，API网关根据`X-Token`自动转换为：`X-Login-UserCode`和`X-Login-CompanyId`，放在Request Header里传递给后台服务；
   * 服务端调用时，通过应用域名请求原始接口，原始接口需要什么参数，都必须提供。比如“查看任何人的工资”，这个接口需要参数：`X-Login-UserCode`、`X-Login-CompanyId`、`userCode`，构造HttpClient请求时必须提供所有参数，服务端程序之间默认是信任的，可查看任意数据。  
  
公开接口（不需要登录的接口）：oreally
``` java
   # 通过API网关访问
  requst url: 
     http://aroute.dooioo.com/oa/salary/server/api/salary/v1/oreally?userCode=8989989
  		 
  #服务端调用
  requst url: 
     http://salary.dooioo.com/api/salary/v1/oreally?userCode=8989989
 ```  
不需要登录的接口，两者构造Http请求时的区别：
   * API网关访问和服务端调用唯一的不同是接口请求路径不同。  
   * 两种调用方式都必须提供原始接口所需的参数。

##### 接口仅限服务端调用
 如果应用有此需求，需修改接口拦截器，增加逻辑：
 	`当发现Request有“X-Route-By”请求头时，直接响应403`。  
注意，有些接口可能支持两种访问方式（API网关和服务端），你可能要检查Controller的方法上是否有特定注解，暗示了这个逻辑。
另外，接口需要登录和接口仅限服务端调用不能并存。
  

### 跨域支持
API网关支持CORS规范，IE10、Chrome、FireFox、Safari浏览器可直接跨域访问接口，不需要通过Jsonp获取数据。

#### Origin的限制
跨域请求都会设置Http请求头：`Origin`，有效值为发起跨域请求的主机域名。
目前我们支持的域名后缀：测试环境=dooioo.net，集成环境=dooioo.cn，生产环境=dooioo.com。

### 超时、重试、断路器机制
#### 超时
通过网关路由的Http请求（静态路由和动态路由）读取超时：`7000ms`，连接超时：`5000ms`。
#### 重试
动态路由的Http请求（访问微服务接口），如果连接超时，默认重试一次；  
出现读取超时，如果请求方法为`GET/DELETE/HEAD`等幂等性方法，会重试一次，其他`POST/PUT`不会重试。

静态路由的Http请求不重试。

#### 断路器
动态路由的Http请求（访问微服务接口）才会适用断路器逻辑。

断路器开启条件： `10秒`内，请求次数超过`30`次，请求失败比例超过：`%65`。
  
粗略地说，如果某个服务10秒内有超过30个请求，但是失败了：30*0.65=18个请求，则会开启断路。

断路器开启后，`5秒`（时间窗口）内不转发请求给后台服务。

### 测试
测试人员可通过API文档中心（[http://api.doc.dooioo.cn/doc](http://api.doc.dooioo.cn/doc)： 服务- 导入到Postman - API网关简单测试，将服务接口导入Postman Chrome App，请不要手动输入接口。

下面我演示如何将楼盘字典核心接口导入Postman：  
#### 访问API文档中心
 访问API文档中心（[http://api.doc.dooioo.cn/doc](http://api.doc.dooioo.cn/doc)，找到楼盘字典核心服务，点击展开 ，选择导出到Postman。  
![apidoc loupan]({{book.imagePath}}/parts/chapter1/images/api_doc_loupan_core.png)

#### 复制API网关简单测试里的链接 
![apidoc import postman]({{book.imagePath}}/parts/chapter1/images/api_doc_import_postman.png)
#### 导入Postman Chrome App
打开Postman Chrome App，找到左上角的导入，在弹出的对话框中选择”From Link”，将刚才的连接粘贴到输入框，点击”Import“，关闭对话框，可在左边的集合目录里找到导入的微服务。  
![import postman]({{book.imagePath}}/parts/chapter1/images/postman_import.png)


### 实时流量监控

#### API网关实时流量监控
API网关实时流量查看：[http://turbine.se.dooioo.com/monitor](http://turbine.se.dooioo.com/monitor)。  
  
集成环境将com更换为cn，测试环境暂不提供此功能。

监控指标说明：  

![Hystrix Dashboard intro]({{book.imagePath}}/parts/chapter1/images/turbine_dashboard_introduction.png)

#### 其他微服务实时流量监控
如果后端服务或者UI项目启用了断路器，也可通过turbine监控实时流量。  
  
访问地址： [http://turbine.se.dooioo.com/monitor?serviceId=你的spring.application.name](http://turbine.se.dooioo.com/monitor?serviceId=spring.applicaiton.name)  

serviceId默认为API网关。  

示例：   
小区UI开启了断路器，它的`spring.application.name`为“loupan-web”，
我们可通过以下连接查看实时流量: [http://turbine.se.dooioo.com/monitor?serviceId=loupan-web](http://turbine.se.dooioo.com/monitor?serviceId=loupan-web)

#### 实时流量监控的前提
1. 增加Hystrix依赖，开启断路器。 
```xml
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-hystrix</artifactId>
		</dependency>
```
2. 依赖spring-boot-actuator。Spring Boot默认有此依赖，可忽略。
3. feign.hystrix.enabled=true，默认true，除非你配置成false。
4. 没有禁用Management Endpoint，默认不禁用，可忽略。
  
### 调试指南
当某个API网关的请求异常时，如何定位问题？  
  
默认情况下，所有的微服务接口都会自动添加Response Header: `X-Instance-Id`。  
而经过API网关路由的请求会自动添加Response Header: `X-Route-By`。

这两个Header的值即节点IP的加密值，开发人员可通过接口解密：[API网关 - 管理支持 (查询X-Instance-Id对应的IP)
](http://api.doc.dooioo.cn/v1/doc/3100800167/194699906/226772003)

通过`X-Instance-Id`、`X-Route-By`开发人员可以判断请求是否到达API网关、是否到达服务接口以及那个节点响应了客户端请求。


下图是使用Postman向我们的API网关`aroute.dooioo.com` 发起的一个请求 

```http
  $.get("http://aroute.dooioo.com/loupan/server/v1/citys")  
```

API网关响应如下：  

![API网关请求头]({{book.imagePath}}/parts/chapter1/images/api-route-header.png)  
  

我们通过接口：[http://aroute.dooioo.com/instance/b7b0fc1f593866af3ac8e2526dd0a880](http://aroute.dooioo.com/instance/b7b0fc1f593866af3ac8e2526dd0a880)，即可查询是那个楼盘服务的节点响应了客户端。

### 注意事项
#### 静态路由可能导致Nginx IP Hash流量到单节点
原因是经过API网关的静态路由请求，请求最终到达Nginx后，Nginx IP Hash默认是以客户端IP为准，此时客户端IP已变更为网关的服务器IP，由于网关服务器IP都是连续的，导致IP Hash到原始服务器的单个节点上。


#### 静态路由可能会添加多次’X-Forwarded-For’ Header。
主要原因是，静态路由通过域名访问后端服务，会多次经过Nginx，这样获取客户端IP时，如果通过 ‘X-Forwarded-For’获取客户端IP时会取到多个IP值。
#### 字符编码
所有接口URI的编码字符集为：`UTF-8`，请求参数的编码字符集为：`UTF-8`。
#### 跨域限制
Web页面跨域访问API网关时，Http请求Header ’Origin‘的值必须是以`dooioo.com`(正式环境)为后缀的域名，否则API网关直接响应403。

#### 并发数
动态路由的每个微服务都有专属的HttpClient，断路器的并发数为：100，HttpClient单节点最大并发数：150，HttpClient配置如下：
```
spi.httpclient:
    keepAliveInSeconds: 40 
    maxRoutePerHost: 150  #单节点最大并发150个连接
    maxTotal: 600   #单个服务的所有节点最大并发600个连接
```

所有静态路由共享一个HttpClient，单个域名最大并发数：300， HttpClient配置如下：
```
##连接池总连接数1200  
zuul.host.maxTotalConnections: 1200
#每个域名并发连接300
zuul.host.maxPerRouteConnections: 300 
```




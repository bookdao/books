<!-- toc -->
### 引言
外网产品线（比如移动工作站、链家网），部署在公网环境（Overseas Zone)，客户端使用OAuth协议完成身份认证，以访问部署在机房（Trusted Zone）的接口（服务）。

支持OAuth协议的服务端，称为 OAuth Server。

以下章节我们以移动工作站为例，简要介绍OAuth Server的工作流程和注意细节。

### OAuth Server - 部署在外网的API网关
我们的OAuth Server目前仅支持OAuth2协议，访问域名：
* 正式环境：https://oroute.dooioo.com  
* 集成环境：https://oroute.dooioo.net  
* 暂不支持测试环境。

OAuth Server的功能类似API网关，不过它认证的是客户端的身份，客户端不登记不申请签证（`access_token`）就不能入境（访问内网接口），而API网关认证的是请求接口的员工（`x-token`)，它要确保授权接口的确是该员工访问的，无法抵赖（因为x-token是根据工号和密码申请的）。

客户端身份通过认证之后，OAuth Server将接口请求转发到API网关。

### 颁发护照 - 客户端登记
移动工作站欲访问部署在内网机房的接口，第一步先向OAuth Server申请护照 - 登记 。

您可以发送邮件或者钉钉联络OAuth Server项目的负责人，沟通相关事宜。

登记成功之后，客户端会拿到自己的护照：  
![护照]({{book.imagePath}}/parts/chapter1/images/passport_O.png)

护照中最关键的字段是：
* ClientId，客户端ID，不能重复。
* ClientSecret，客户端密钥，不能泄露。

这两个字段用来标识客户端身份，合法的clientId和clientSecret才能申请Access Token。  

### 申请签证 - 申请Access Token
调用任何接口（入境）之前，必须通过ClientId和ClientSercret(护照)申请签证 - Access Token。

下面我演示集成环境如何通过OAuth Server访问楼盘接口：
 `https://oroute.dooioo.net/loupan/server/v1/citys`
#### 1. 通过护照（clientId和clientSecret）申请access_token  
详细的接口文档，请参考：[OAuth2认证服务 - token (申请Token)
](http://api.doc.dooioo.org/v1/doc/212240510/2696765699/2081483726)。  
注意事项：  
1. `接口参数必须放在Request Body里，以Form表单的方式提交,Url参数会泄露clientId和clientSecret` 
2. 请尽量将接口升级到签名校验的版本，签名计算规则，请参考章节：[注意事项](#%E6%89%80%E6%9C%89%E6%8E%A5%E5%8F%A3%E5%B0%86%E5%BC%BA%E5%88%B6%E6%A3%80%E6%9F%A5%E7%AD%BE%E5%90%8D%E5%8F%82%E6%95%B0signature)。
  
代码示例：
``` http
# Http Header
Request URL:https://oroute.dooioo.net/oauth/token
Request Method:POST
Content-Type:application/x-www-form-urlencoded

#Request Body - Form 表单
grant_type:client_credentials
client_id: andriod_app
client_secret:f60zc3ndf1f80ac3e8a4fcavbaacn91vmf4dad7a7c5

#需要校验签名的版本，不需要传递clientSecret
#grant_type:client_credentials
#client_id:andriod_app
#timestamp:134523332432544
#signature:dsfs3sdsfsdsqsdsfdf3sdwerferdfdsfwfw
```
接口响应数据如下：
``` json
{
  "token_type": "Bearer",
  "access_token": "40d77f7f5bc6152092e49464294077bc",
  "expires_in": 3600,
  "refresh_token": "708ccc3a1b5ee331ef12d36cf925ecee",
  "scope": null
}
```
`access_token`: 即Access Token的值，示例为：40d77f7f5bc6152092e49464294077bc。  
`expireds_in`:  access_token的有效期，单位秒，目前集成和正式环境Access Token的有效期皆为3600秒（1小时）。  
`refresh_token`: 用于刷新的access_token的过期时间。

#### 2. 接口请求增加Request Header：Authorization
获取Access Token之后，请求接口时新增Http Request Header:
 `Authorization: Bearer 40d77f7f5bc6152092e49464294077bc`，将access_token传递给OAuth Server，进行身份校验。
 
代码示例:  
``` http
#Http Header
Request URL:https://oroute.dooioo.net/loupan/server/v1/citys
Request Method:GET
#每次接口请求必须设置Authorization请求头
Authorization: Bearer 40d77f7f5bc6152092e49464294077bc

#X-Token请求头仅限访问授权接口，其值为SSO登录接口响应的loginTicket。
#X-Token: 2cgd77f7f5bc152092e494232xsa1vmd
```
如果`access_token`非法，则OAuth Server会响应以下业务码:  
![oauth bizcode]({{book.imagePath}}/parts/chapter1/images/oauth_bizcode.png)


性能优化：  
客户端可在Access Token有效期内共用一个access_token，并定时调用OAuth Server刷新access_token的接口延长其有效期，这样可避免每次接口请求时重新申请access_token。


### 续签 -  刷新Access Token
出于安全性考虑，access_token的有效期通常为1小时。但OAuth Server提供了刷新access_token的接口，客户端可根据`refresh_token`刷新`access_token`的过期时间，接口参数必须以form表单的方式提交。

代码示例：
``` http
# Http Header
Request URL:https://oroute.dooioo.net/oauth/token
Request Method:POST
Content-Type:application/x-www-form-urlencoded

#Request Body - Form 表单
grant_type:refresh_token
refresh_token:708ccc3a1b5ee331ef12d36cf925ecee

#签名校验版本
#grant_type:refresh_token
#referesh_token:708ccc3a1b5ee331ef12d36cf925ecee
#timestamp:134532324359023
#signature:xsdf23daqazdfdfree2sfsfefeferewr
```

接口会返回新的access_token：
``` json
{
  "token_type": "Bearer",
  "access_token": "90ae85a983c612cd1d912a4a06061431",
  "expires_in": 3600,
  "refresh_token": "708ccc3a1b5ee331ef12d36cf925ecee",
  "scope": null
}
```

### 超时、重试、断路器机制
#### 超时
经过OAuth Server路由的Http请求读取超时：`8000ms`，连接超时：`5000ms`。
#### 重试
经过OAuth Server路由的Http请求不重试，由API网关负责重试。

#### 断路器
断路器开启条件： `10秒`内，请求次数超过`30`次，请求失败比例超过：`%65`。
  
粗略地说，如果某个服务10秒内有超过30个请求，但是失败了：30*0.65=18个请求，则会开启断路。

断路器开启后，`5秒`（时间窗口）内不转发请求给后台服务。

### OAuth Server实时流量监控

打开Turbine监控页面：[http://turbine.se.dooioo.com](http://turbine.se.dooioo.com)，如下图所示：  

![Turbine Dashboard]({{book.imagePath}}/parts/chapter1/images/turbine_monitor.png)

将以下连接复制到上图所示的输入框：
``` http
 http://oroute.dooioo.com:16099/admin/hystrix.stream
```  
  
点击按钮：monitor stream，即可查看实时流量，如下图所示：
 ![ORoute Turbine monitor]({{book.imagePath}}/parts/chapter1/images/oroute_turbine_monitor.png)

监控指标说明：  

![Hystrix Dashboard intro]({{book.imagePath}}/parts/chapter1/images/turbine_dashboard_introduction.png)

#### 注意事项
通过以上方法只能查看OAuth Server单个节点的实时流量，新版本会废弃。

另外，实时流量监控仅限内网访问。

### 调试指南
当某个接口请求异常时，如何定位问题？  

经过OAuth Server路由的请求都会自动添加Response Header: `X-Authorize-By`；  
  
经过API网关路由的请求会自动添加Response Header: `X-Route-By`；  

默认情况下，所有的微服务接口都会自动添加Response Header: `X-Instance-Id`；  

这些Header的值即节点IP的加密值，开发人员可通过接口解密：[API网关 - 管理支持 (查询X-Instance-Id对应的IP)
](http://api.doc.dooioo.org/v1/doc/3100800167/194699906/427734797)

通过`X-Authorize-By`、`X-Route-By`、`X-Instance-Id`，开发人员可以判断请求是否到达OAuth Server、API网关、原始服务接口以及那些节点响应了客户端请求。


下图是使用Postman向我们的OAuth Server`https://oroute.dooioo.net` 发起的一个请求: 

![OAuth Server请求头]({{book.imagePath}}/parts/chapter1/images/oroute_header.png)  
  

我们通过接口：`http://api.route.dooioo.com/instance/{instanceId}`，即可查询是那些节点响应了客户端。

### 注意事项
#### 集成环境OAuth Server的Https证书无效  

由于我们集成环境OAuth Server域名：[https://oroute.dooioo.net](https://oroute.dooioo.net)没有申请Https证书，Chrome Postman App首次访问接口时无法建立https连接，需要复制域名，在浏览器里手动确认信任此Https链接，如下图：  

![Invalid Https CA]({{book.imagePath}}/parts/chapter1/images/invalid_https_ca.png) 

之后Chrome Postman App就可以正常请求接口了。
#### 字符编码
所有接口URI的编码字符集为：`UTF-8`，请求参数的编码字符集为：`UTF-8`。

#### 并发数
每个服务（路由）有自己的专属HttpClient，断路器的并发数为：100，HttpClient单节点最大并发数：200，HttpClient配置如下：
```
spi.httpclient:
    keepAliveInSeconds: 40
    maxRoutePerHost: 200 ###目前只有API网关一个host
    retryCount: 0 ## 禁止重试
    maxTotal: 200 ## 最大连接数，我们单个服务共用一个HttpClient
```

#### 所有接口将强制检查签名参数signature
未来版本的OAuth Server将强制检查请求参数：`signature`。  
signature的计算规则如下：
* 所有客户端请求自动增加参数： `timestamp`，其有效值为请求发出时的时间戳。  
  OAuth Server会比较服务器时间和请求时间戳，如果两者`相距15分钟`以上，就直接响应`403`，并提示请求已过期，因此客户端必须`定时同步时钟`。
* 所有参数按照参数名升序排列，然后按照 `参数名=参数值` 格式以&拼接；如果参数值是多个，则参数值以英文逗号（,)隔开追加在一起。
* 将客户端分配到的clientSecret追加在最后，参数名为client_secret。
* 对以上步骤生成的字符串进行MD5。
    
#### Signature计算示例  
接口：https://oroute.dooioo.com/loupan/server/v1/reblock  
  
参数：userCode=892312302&companyId=12&city=230&zone=test&zone=dev&timestamp=1477621956723&p1=&p1=2&p1=null  
  
clientSecret：`f6848af3e8a4fcaba2b3911f4da7a7c5` 
  
下面开始构造Signature签名：  
    
第1步：将所有参数名按字典顺序升序排列，排序结果为：city,companyId,p1,timestamp,userCode,zone。   

第2步：将第1步排序后的参数以`参数名=参数值`(key=value)的格式、以`&`拼接起来。多个参数值，参数值之间以逗号隔开。  

``` json
city=230&companyId=12&p1=,2,null&timestamp=1477621956723&userCode=892312302&zone=test,dev
```
参数p1有三个值：””、”2”、”null”，拼接起来后：”,2,null”。  

第3步：将clientSecret的值追加在在第二步生成的字符串后，参数名为client_secret:
``` json
city=230&companyId=12&p1=,2,null&timestamp=1477621956723&userCode=892312302&zone=test,dev&client_secret=f6848af3e8a4fcaba2b3911f4da7a7c5
```
第4步：对第3步生成的字符串进行MD5加密，即为signature的值。
``` json
MD5(city=230&companyId=12&p1=,2,null&timestamp=1477621956723&userCode=892312302&zone=test,dev&client_secret=f6848af3e8a4fcaba2b3911f4da7a7c5) = 2946ad157e140bfb134197403b42ca42
```
  
第5步：发送实际请求
``` json
https://oroute.dooioo.com/loupan/server/v1/resblock?  
userCode=892312302&companyId=12&city=230&zone=test&zone=dev&timestamp=1477621956723  
&p1=&p1=2&p1=null&signature=2946ad157e140bfb134197403b42ca42
  
Authorization: Bearer xdfw3dv3sdsdfawqozfjefqweqwr
```  
  







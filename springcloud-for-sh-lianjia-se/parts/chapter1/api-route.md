<!-- toc -->

### 认证
所有接口文档（http://api.doc.dooioo.org）中出现**需要授权**的Label，都需要传输 ”X-Token“。

API网关负责统一认证，所谓认证，是通过工号和密码调用SSO登录服务登录，成功登录后（认证通过） 的信息通过 ‘X-Login-UserCode’、‘’-Login-Company-Id’传递给后台服务。

### 路由
API是访问机房服务的桥梁，而能否访问到某个服务，则取决于是否有此服务的路由信息。 

路由，可理解为虚拟路径和主机IP的映射规则：
```
   ###  如果请求路径前缀是：/ky/old/server ，则路由请求到主机：http://10.8.1.204:1080
  /ky/old/server =>  http://10.8.1.204:1080
  
  ### 如果请求路径前缀是：/loupan/server，则路由请求到主机：http://loupan.dooioo.com
  /loupan/server => http://loupan.dooioo.com
```


路由目前分为静态路由和动态路由。

#### 动态路由
所谓动态路由，就是服务会自动发现，并自动建立服务名到IP的映射规则，无需人为处理。  
  
目前，我们使用Eureka来实现服务的自动发现，大家可以访问：[http://discorvery1.se.dooioo.com](http://discovery1.se.dooioo.com)查看所有注册的服务。
  
动态路由将服务名映射为虚拟主机名，请求路由时，根据**虚拟主机名**去Eureka里查询所有节点IP：端口，再负载均衡到某个节点上。

动态路由的映射规则示例：
```
   ###如果访问的是服务：/loupan/server ，则路由请求到虚拟主机：ky-old-server，
   ###实际路由时，组件根据虚拟主机名去Eureka里查询所有节点的IP:端口，负载均衡到某一个IP上。
  /loupan/server =>  ky-old-server
```
#### 静态路由
静态路由，是指需要管理员手动维护的映射规则。
通常是为了老系统可以通过API网关访问。

静态路由的虚拟路径推荐的命名规范： /产品线/产品/server，比如：/ky/tel/server，表示客源产品线的电话服务。

静态路由的映射规则（path=> location)示例：
```
   ###如果访问的是服务：/ky/tel/server ，则路由请求到主机：http://tel.ky.dooioo.com:80
  /ky/tel/server =>  http://tel.ky.dooioo.com
```

##### 添加静态路由
测试和集成环境，开发和测试人员可自由添加静态路由，方便调试。  
接口说明文档请参考：[API网关 - 管理支持 (新增或更新静态路由)
](http://api.doc.dooioo.org/v1/doc/3100800167/194699906/2606921125)

此接口需要Header参数：X-Role-Id，测试和集成环境有效值为：admin。
path参数即虚拟路径，location为服务访问的域名。

生产环境的静态路由，请在上线之前钉钉发给我。

#### 查询可用路由
开发和测试人员可通过接口： [http://api.route.dooioo.com/admin/routes.json](http://api.route.dooioo.com/admin/routes.json) ，查询当前可用路由信息。

### 实时流量监控
API网关的实时流量在线监控：[http://turbine.se.dooioo.com/monitor](http://turbine.se.dooioo.com/monitor)。  
  
集成环境将com更换为org，目前测试环境暂不提供此功能。

监控指标说明：  

![Hystrix Dashboard intro]({{book.imagePath}}/parts/chapter1/images/turbine_dashboard_introduction.png)

### 调试指南
当某个API网关的请求异常时，如何定位问题？
默认情况下，所有的微服务接口都会自动添加Header: X-Instance-Id。  
而经过API网关路由的请求会自动添加Header: X-Route-By。

这两个Header的值即节点IP的加密值，开发人员可通过接口解密：[API网关 - 管理支持 (查询X-Instance-Id对应的IP)
](http://api.doc.dooioo.org/v1/doc/3100800167/194699906/427734797)

通过X-Instance-Id、X-Route-By开发人员可以判断请求是否到达API网关、是否到达服务接口了以及那个节点响应了客户端请求。


下图是使用Postman向我们的API网关`api.route.dooioo.com` 发起的一个请求 

```http
  $.get("http://api.route.dooioo.com/loupan/server/v1/citys")  
```

API网关响应如下：  

![API网关请求头]({{book.imagePath}}/parts/chapter1/images/api-route-header.png)  
  

我们通过接口：[http://api.route.dooioo.com/instance/b7b0fc1f593866af3ac8e2526dd0a880](http://api.route.dooioo.com/instance/b7b0fc1f593866af3ac8e2526dd0a880)，即可查询是那个楼盘服务的节点响应客户端

### 超时、重试、断路器机制
#### 超时
API网关静态路由和动态路由的Http请求读取超时时间：7000ms，连接超时：5000ms。
#### 重试
通过动态路由的Http请求，如果连接超时，默认重试一次；  
遇到读取超时，如果请求方法为GET/DELETE/HEAD等幂等性方法，会重试一次，其他POST/PUT不会重试。

通过静态路由的Http请求不重试。

#### 断路器
通过动态路由的Http请求才会应用断路器逻辑。

断路器开启条件： 10秒内，请求次数超过30次，请求失败次数超过：%65。
  
粗略来说，如果某个服务10秒内有超过30个请求，但是失败了：30*0.65=18个请求，则会开启断路。

断路器开启后，5秒（时间窗口）内不转发请求给后台服务。

### 测试
测试人员可以通过API文档中心（[http://api.doc.dooioo.org](http://api.doc.dooioo.org)： 服务- 文档及其他 - API网关简单测试，将服务接口导入到Postman，请不要手动输入接口。

### 注意事项
#### 静态路由可能会添加多次’X-Forwarded-For’ Header。
主要原因是，静态路由通过域名访问后端服务，会多次经过Nginx，这样获取客户端IP时，如果通过 ‘X-Forwarded-For’获取客户端IP时会取到多个IP值。

#### 并发数
动态路由的每个服务对应一个HttpClient，断路器的并发数为：100，HttpClient单节点最大并发数：150，HttpClient配置如下：
```
spi.httpclient:
    keepAliveInSeconds: 40
    maxRoutePerHost: 150
    maxTotal: 600
```

所有静态路由共用一个HttpClient，单个域名最大并发数：300， HttpClient配置如下：
```
##连接池总连接数1200  
zuul.host.maxTotalConnections: 1200
#每个域名并发连接300
zuul.host.maxPerRouteConnections: 300 
```



## 服务发现客户端
Hi，boys，为了简化非微服务应用接入微服务体系，我们开发了此组件。

### 限制条件
* 组件仅限服务端应用使用
* 组件使用JDK8编译
* 组件需依赖Spring Framework（管理生命周期），但是可方便改为独立使用（有需要的私聊）

### changelog
 0.8.6：依旧采用动态token的安全方案，但是延长了动态token变更时间（推荐此版本）
 
 ====== 以下为历史版本 =====================
 
 0.8.3: 修复Tomcat端口变更未及时更新健康检查url
 
 0.8.4: 客户端采用动态token生成方案，提供更高的安全性
 
 0.8.5: 简化安全Header方案

### 使用指南
#### 第一步 pom.xml文件添加依赖

``` xml
 		<dependency>
			<groupId>com.lianjia.sh.se</groupId>
			<artifactId>dummy-eureka-client</artifactId>
			<version>0.8.6</version>
		</dependency>
 ```

 
#### 第二步 配置Spring Bean

 ``` xml
<!-- 老项目都会配置个placeholder，自动将环境保存成env变量 或者SpringBoot项目使用spring.profiles.active -->
<bean
		class="com.lianjia.sh.se.dummy.eureka.client.discovery.DiscoveryClient">
		<property name="env" value="${env}" />
		<!-- spring boot也可以使用spring.profiles.active
		<property name="env" value="${spring.profiles.active}"/>
		-->
		<!-- 当前应用的标识，注意，不同应用应该使用不同的app，要保持全局唯一，只允许字母数字和连字符 -->
		<property name="app" value="jiaoyi-old-server"/>
	</bean>
	
``` 
 
SpringBoot项目可以使用@Bean，直接实例化并设置属性env和app;

#### 第三步 使用
``` java
@Component
public class DiscoveryDemo {
  @Autowired
  private DiscoveryClient discoveryClient;
  
  @Scheduled(fixedDelay=30000)
  public void doJob(){
    try {
      //根据微服务名来查询当前可用节点，采用轮询的方式从多个节点里选取一个
      Server availableServer=this.discoveryClient.findNextServer("se-phonebook");
      //任何一次接口调用都必须传递整个securityHeaders
      SecurityHttpHeaders securityHeaders=this.discoveryClient.getSecurityHttpHeaders();
      
      httpClient.doGet(availableServer.getUrl()+"/v1/telcom/subscriber/5430112544727/number")
      for (HttpHeader header : securityHeaders.getHeaders()) {
        httpClient.addHeader(header.getName(),header.getValue());
      }
      //发送请求
      httpClient.send();
    } catch (NoAvailableServerException e) {
      //ops...没有可用节点.
    }
  }
}
```
### 注意事项
#### IP和端口
组件会自动向服务发现中心注册Tomcat实例的IP和端口。

IP地址，默认为当前机器启用的网卡的IP，如果有多个网卡，则默认为最后一个网卡的IP（网卡默认会有顺序）；

端口，组件默认从Spring `Enviroment`接口里获取配置属性：`server.port`，如果获取不到则会尝试自动解析Tomcat端口号。


SpringBoot和SpringCloud项目都会有此配置，老Tomcat项目通常来说，自动解析端口就足够了，如果无法解析成正确的Tomcat端口，则可以通过：`server.port=8999`显式指定。

如果Tomcat有多个可用端口，也可以使用以上属性显式指定。

#### 负载均衡
组件从多个节点中选择服务节点的策略是：轮询。

#### app 和 env
 DiscoveryClient的属性app推荐命名规范为：小写字母，多个单词之间'-'连接。
 例如： `fy-old-server`，`old-login-ui`，`ky-old-server`，`shouhou-serve`
 
 app作为作为服务的唯一标识，必须保证全局唯一，并且不得变更。
 
 默认情况下，所有app都会自动追加前缀：`overseas-`。
 
 假设你配置的app为`fy-old-server`，实际上注册到服务发现的服务名为：`overseas-fy-old-server`。
 
 因此老项目之间调用时应该为：`this.discoveryClient.findNextServer("overseas-fy-old-server");`
 
 DiscoveryClient的属性env目前仅限：development，test，integration，production，大小写敏感。
 
#### 缺少X-Login-*相关参数错误
使用DiscoveryClient访问接口时，请以API文档-头部参数-内部调用Tab为准。


如果出现缺少`X-Login-UserCode`、`X-Login-CompanyId`、`X-Login-Token`等参数的错误，请提供对应参数。


相应参数说明如下：
``` java
	"X-Login-UserCode" ： 员工工号，如果从API网关过来的请求，则是登录员工的工号，内部服务可直接传业务相关工号
	"X-Login-CompanyId" ：公司ID，如果从API网关过来的请求，则是登录时该员工所选择公司ID，内部服务可直接传业务相关公司
	"X-Login-Token" ： 员工登录后分配的登录ticket，如果接口需要此参数，则使用方必须提供登录ticket，也就是说此接口只能用在Web。
```

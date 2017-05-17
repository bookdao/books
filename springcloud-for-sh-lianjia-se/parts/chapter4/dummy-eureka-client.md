## 服务发现客户端
Hi，boys，为了简化非微服务应用接入微服务体系，我们开发了此组件。

### 注意事项
* 组件仅限服务端应用使用
* 组件使用JDK8编译
* 组件需依赖Spring Framework（管理生命周期），但是可方便改为独立使用（有需要的私聊）

### 使用指南
#### 第一步 pom.xml文件添加依赖

``` xml
 		<dependency>
			<groupId>com.lianjia.sh.se</groupId>
			<artifactId>dummy-eureka-client</artifactId>
			<version>0.8.2</version>
		</dependency>
 ```

 
#### 第二步 配置Spring Bean

 ``` xml
<!-- 老项目都会配置个placeholder，自动将环境保存成env变量 或者SpringBoot项目使用spring.profiles.active -->
<bean
		class="com.lianjia.sh.se.dummy.eureka.client.discovery.DiscoveryClient">
		<property name="env" value="${env}"></property>
		<!-- spring boot也可以使用spring.profiles.active
		<property name="env" value=“${spring.profiles.active}”></property>
		-->
		<!-- 当前应用的标识，注意，不同应用应该使用不同的app，要保持全局唯一，只允许字母数字和连字符 -->
		<property name="app" value=“jiaoyi-old-server”></property>
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

 
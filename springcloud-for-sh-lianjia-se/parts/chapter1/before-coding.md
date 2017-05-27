<!-- toc -->
在编码之前，有几个编码理念，需要强调一下：

#### 面向契约  
自古以来，Java中就流传着一种传说：面向接口、面向契约编程。但在快速迭代中，接口反而成了累赘。这次微服务实践，Service SPI将全部面向契约，但服务实现方不做约束。

#### 应用程序无状态  
编写分布式程序的最佳实践就是节点无状态，我们可以简单地将无状态性理解为Http请求的幂等性：  
一次和多次请求任何一个节点都有相同的副作用。  
  
简单来说，应用程序节点之间无依赖，我们不需要同一个用户的请求只能落在一个节点上（IP Hash)。那么，遇到共享资源（数据）时，我们如何解决？    

我之前做的一个项目，电话转接号的申请接口就是一个共享资源的例子：

*  电话转接有两到三个节点，尚未分配的所有转接号就是一个共享资源池（同一个数据库）。
*  一个转接号只能分配给一个人。
*  请求可能同时重复发送（可能落在同一节点，也可能落在任一节点），无论请求发送多少次，此人已分配的转接号应该不变（不能重复申请,但每隔三天此人的转接号就要过期）。

当时设计的目标TPS=300/s,平均响应时间应在300ms以下,具体实现分以下几步：
1. 节点一启动，首先主动向数据库申请4000个转接号（更新转接号状态为已占用）放在程序内存中，比如 currentPool=`ConcurrentLinkedQueue`，有一个`AtomicInteger`类型的计数器，统计集合中已分配了多少转接号，如果达到某个阈值，再异步向数据库批量申请4000个放在另一个 preparedPool=` ConcurrentLinkedQueue`中，当currentPool分配完或不足时，此时再加锁，执行逻辑：`currentPool=preparedPool,preparedPool=null;`。  
   这样便将数据库事务转换成JAVA并发了，高并发时也能稳定服务。  

2. 节点收到员工申请转接号的请求时，先向memcache中添加一个标识位（`memcache#add(key,value,10s)`),类似互斥锁。操作成功之后，开始向currentPool申请转接号，然后缓存到memcache中。  

3. 如果某个节点向memcache添加互斥锁时失败，那说明有节点已在处理了，那它应该等待200毫秒，直接从缓存中获取已分配的转接号。一般可以重试两三次，我们允许此种情况下可能出现的失败。  
  基本这种情况是极其少见的，一般是压力测试或被攻击时才频繁。
  
 可以看到，我使用了memcache来承担全局变量的角色，也存在某种程度的风险。
 在平时的业务实现中，应该尽量在设计上避免此类共享资源的场景。
 
除了上述的共享资源，还有一种场景我们也应该避免：分布式事务。  
  
跨系统的业务交互，尽量走异步队列，如果某次业务逻辑处理失败，尽量有重试机制，以保证业务数据最终一致性。
 
#### 代码的自我表达能力  
代码的自我表达能力是一种最佳编码实践。  
  
有时候，仅仅是实现方式的稍微调整，比如以下代码是向远程服务器发起Http调用：
 
``` java
	RemoteRequest  
	.to(“http://aroute.dooioo.cn/loupan/server/v1/city/{id}”)
    .withParam("id",3).get(City.class);
```
比如，有时候使用静态内部类（接口），也可使代码更可读：

```java
  public interace InvocationHandler{
       Object invoke(some param);
       
       static final class Factory{
         InvocationHandler   create(some param){
         }
       }
  }
  //调用时
   new InvocationHandler.Factory();
  
```
代码编写既要追求代码简洁，又要追求表达力。

在这方面，我们还有许多改进和学习的地方，欢迎讨论。

#### 完善的注释以及版本意识  
本次微服务实践，Service SPI的编写者必须提供完善的注释。  

除了方便客户端更友好的接入外，我们会自动根据java doc 生成 API 网关的文档。
java doc的说明文档请[参考这里](http://www.oracle.com/technetwork/java/javase/documentation/index-137868.html#tag)，大家可以了解下tags的顺序及其他说明。

下面我演示一下标准SPI接口注释的写法：
``` java
/**
 * 区域标准服务，区域指行政区划，比如上海市的静安区、黄埔区等。
 * @summary 区域
 * @Copyright (c) 2016, Lianjia Group All Rights Reserved.
 */
@FeignClient("loupan-server")
public interface DistrictSpi{
	/**
	 * 根据区域ID查找区域，方法的详细说明
	 * 说明，说明。
	 * @author huisman
	 * @version v1
	 * @param  districtId 区域ID
	 * @param  userCode 工号
	 * @since 2016-01-01
	 * @summary 根据ID查找区域 
	 */
	@LoginNeedless
	@LorikRest(value={Feature.NullTo404},codes={20010,20020})
	@RequestMapping(value = "/v1/districts/{districtId}", method = RequestMethod.GET)
	District  findDistrictV1(
	        @PathVariable(value = "districtId") long districtId,          
	        @RequestParam(value=“userCode”,required=false,defaultValue=“8080”)  Integer userCode);
}

```

上述java doc在API Gateway 将被展示为：

|  字段  | 说明|
| :------------ | :-----------| 
| Request Path  | GET /v1/district/{districtId}?userCode={userCode}  |
| 接口功能  | 根据ID查找区域 (since 2016-01-01)         |
| 接口说明  | 根据区域ID查找区域，方法的详细说明<br>说明，说明。      |
| 安全性  | 无需登录即可访问 or 登录后才可访问|
| 版本号  | v1          |
| 路径参数 | @PathVariable districtId，区域ID, 类型:Long，必填       |
| 请求参数 | @RequestParam userCode，工号，类型:Integer，可选，默认值8080      |
| 业务码|20010：区域已删除；20020：区域不存在   |
| 返回值| model District json输出 |
| author|huisman|

新增了两个用在方法上的注解：

*    ```@LoginNeedless```   

	指明此接口无需登录，即可访问。如果无此标注，则登录后才能访问，否则提示无权限。该标注对应接口文档的安全性说明。  

*   ```@LorikRest```     

	可以配置此方法通过API网关（aroute.dooioo.cn)访问时的一些REST特性和业务码。  
比如：```@LorikRest(value={Feature.NullTo404},codes={20010,20020})```，可理解为通过API网关（aroute.dooioo.cn)访问时，如果此方法返回结果为null，则统一响应为404；codes指明此方法可能抛出的业务错误码。  
非必须，如果方法实现会抛出业务错误码，则需显式在方法上声明。业务码对应接口文档的业务码说明。

新增了个doc tag: @summary。

*  ```@summary``` 即可用做接口方法的功能简介，也可用做SPI 类的功能简介（一般不超过20字）。


其他doc tag说明如下：<br>
```@author``` tag既可以是负责人名，也可以是邮件地址或者产品线团队。<br>
```@param``` 我们用于显示方法参数的说明。<br>
Spring MVC的标注将会被正确解释为符合Http 语义的说明，比如路径参数，请求参数。<br>
```@version``` REST接口的版本号 。<br>
```@since``` 接口开发或发布日期，比如：2016-01-01，追加在接口功能里。<br>

方法的描述（doc 注释的第一行）则显示为接口说明。
各个doc tags 的顺序建议和示例保持一致。


 
 
       
##分布式任务组件接入教程

### 项目介绍
`se.tools.quartz-toc`是一个依赖于 quartz 与 sqlserver的分布式调度组件。
  
主要特点是 **易用**，**易用**，**易用**！

用户可在无需改变现有代码的情况下，使得通过`@Scheduled`和`xml配置`的Spring Task支持`多节点分布式调度`。

### 使用场景
* 本组件仅支持基于Spring Framework （3.x及以上版本）的Java项目
* 透明地接管基于`xml配置`、`@Scheduled`注解实现的Spring定时任务。
* 目前不支持使用原生`Quartz`实现的定时任务。

### Maven依赖(支持cn环境)
```xml  
	<dependency>
			<groupId>com.lianjia.sh.se.tools</groupId>
			<artifactId>se-quartz-toc</artifactId>
			<version>0.3.3</version>
	</dependency>
```

### 项目配置
#### SpringBoot & Spring Cloud
在 `start-class` 中使用注解 `@EnableQuartzToC`开启托管，代码示例：
``` java
@EnableEurekaClient
@SpringBootApplication
@EnableBuiltinRestSupport
@EnableQuartzToC
public class ApplicationStat {

  public static void main(String[] args) throws Exception {
    SpringApplication.run(ApplicationStat.class, args);
  }
}
	```
#### 老SpringMVC项目
  ##### 环境配置
  在Spring的xml配置文件中实例化一个`EnvironmentConfiguration` Bean，用于指定组件运行的环境，以下为代码示例：
  
 ``` xml
 <bean class="com.lianjia.sh.se.tools.quartz.toc.config.EnvironmentConfiguration"> 
        <!--当前启动环境，支持 development、test(与development统一管理)、integration、production-->
        <property name="env" value=“${env}” /> 
        <!--系统名，请修改为你的应用有关的名称（英文字符），用于Job分组，方便Job监控和查看 -->
        <property name="applicationName" value=“YouApp”/>
    </bean>
 ```
  ##### 添加注解 @EnableQuartzToC
  在任意一个Spring Bean的类上添加注解 `@EnableQuartzToC`以开启托管。
  ``` java
    //@Service
    //Configuration
    @Component
    @EnableQuartzToC
    public class AnySpringComponentBean{
    }
  
  ```


### 监控：任务查看
大家可访问：[分布式任务监控 - 简陋版](http://job.dooioo.com/triggers "简陋版") 查看所有系统被托管的定时任务的运行状况、运行历史等，目前不提供变更操作。  
  
![简陋版 任务查看]({{book.imagePath}}/parts/chapter4/images/triggers_page.png)



<!-- toc -->
## SPI自动生成API文档
### 插件配置
API文档是根据Java源代码的注释自动生成，Maven项目（SPI模块）仅需配置apidoc插件：
```xml
		<plugin>
				<groupId>com.dooioo.se.lorik</groupId>
				<artifactId>maven-apidoc-plugin</artifactId>
				<version>1.0.5.3</version>
				<extensions>true</extensions>
				<configuration>
					<options>-appName "${project.name}"</options>
				</configuration>
		</plugin>
```

该插件需配置参数：
  -appName “微服务的中文名称”
   
然后运行Maven 命令： `mvn apidoc` 即可。

源代码的解析是由lorik-apidoclet-1.0.4.jar负责的，解析后的数据自动导入到：[http://api.doc.dooioo.cn](http://api.doc.dooioo.cn)。

### 插件支持的选项
 -packages.exclude   
		声明那些java package（包括所有sub package) 不生成API文档。
  多个值以:分隔，示例：  
  `-packages.exclude com.lianjia.sh.xx.spi:com.lianjia.sh.xx.search`
 
 -packages.include  
	声明仅限那些package（包括sub package)生成API文档。
  多个值以:分隔，示例：
  `-packages.include com.lianjia.sh.xx.spi:com.lianjia.sh.xx.search`  
  
-app  
	服务或应用的标识，一般为API网关的虚拟路径，同一个app生成的文档会覆盖之前生成的，如果某个应用生成了多份文档，请检查应用app的值是否变更。   
	 微服务项目无需指定这个选项，系统会自动根据 @FeignClient(“service-name”)提取app的值，但是老的SpringMVC项目或者Spring Boot项目，需要手动指定app的名称，此名称即应用在API网关中的虚拟路径。  

  示例： -app “fy-old-server”  

 -appName   
	服务或应用的标识app在API文档中的displayName，主要方便人类识记。  

   示例： -appName “楼盘字典”  
  
-print  
	是否打印解析后的数据，示例： -print true  
  
-version   
	默认版本号，内置默认版本号：v0，示例： -version v1  
	
-forceLogin  
  开启强制登录，默认情况下，微服务所有接口都是需要登录的，只有使用```@LoginNeedless```注解的方法才无需登录，而普通SpringMVC项目的RestController则默认是不需要登录的，这对于开放给其他项目的对外接口来说，存在安全隐患；  
  
生成文档时，指定此选项，普通Spring MVC的接口也会提示需要登录，无需登录的接口，请设置-loginNeedless选项；  
  
示例：  -forceLogin  
  
适用v1.0.5及以上；
  
 -loginNeedless  
  
生成文档时，对于接口是否登录的判断，默认是检查方法是否包含类名为：```com.dooioo.se.lorik.spi.view.authorize.LoginNeedless```的注解，对于非微服务的其他项目来说，标识无需登录的注解可能是其他类名，客户端可使用此选项覆盖默认行为；  
  
示例：  -loginNeedless com.lianjia.sh.jiaoyi.interceptor.LoginNeedless  
  
适用v1.0.5及以上；
 
-strictMode  
  
开启严格模式，默认情况下，所有没使用@nodoc注释的类都会被导出，当开启严格模式时，仅限使用@apidoc注释的类才会生成文档；   
  
注意，@apidoc仅适用于类级别的注释   
  
示例： -strictMode    
  
适用v1.0.5及以上  
  
-exportTo  
	解析后的数据导出到那个接口，默认值为：http://api.doc.dooioo.cn/v1/restapps/import/binary。  
	  示例：-exportTo http://balabala.domain.com/v1/binary  
  
-ignoreVirtualPath  
	此选项会影响API文档展示的请求路径，默认值：false。  
	虚拟路径一般为-app选项指定的值，默认情况下，-app选项的值会追加到接口RequestPath之前。  
	比如：原始接口= /v1/user/findById ，-app选项（虚拟路径）: fy-old-server。  
	如果ignoreVirtualPath为true，则最终生成的接口路径为：/v1/user/findById。  
	如果ignoreVirtualPath为false，则最终生成的接口路径为：/fy/old/server/v1/user/findById。  
	所有通过API网关访问的应用，此选项必须为false。如果通过自己应用域名访问，则可以设置此选项值为true。
  
  示例： -ignoreVirtualPath true
  
  完整示例：
  ```xml
		<plugin>
				<groupId>com.dooioo.se.lorik</groupId>
				<artifactId>maven-apidoc-plugin</artifactId>
				<version>1.0.5.3</version>
				<extensions>true</extensions>
				<configuration>
					<options>
					   -appName "${project.name}" 
					   -packages.include com.lianjia.sh.se.loupan.spi 
					   -packages.exclude com.lianjia.sh.se.loupan.spi.core:com.lianjia.sh.se.loupan.spi.search
					   -print true  -version v1 
					   -ignoreVirtualPath true
					   -strictMode
					   -forceLogin 
					   -loginNeedless com.lianjia.sh.jiaoyi.interceptor.LoginNeedless
					</options>
				</configuration>
		</plugin>
```


### 注意事项
如果遇到Maven Apidoc插件无法下载时，请修改你本地的${user.home}/.m2/setting.xml，添加插件仓库：

``` xml
<pluginRepositories>
            <pluginRepository>
              <id>central</id>
              <name>Maven2</name>
                <!— 公司私服地址 —>
              <url>http://nexus.dooioo.cn/repository/public</url>
              <layout>default</layout>
              <snapshots>
                <enabled>true</enabled>
              </snapshots>
              <releases>
                <updatePolicy>never</updatePolicy>
              </releases>
            </pluginRepository>
     </pluginRepositories>
```

### 注释要求
源代码的注释必须符合我们的要求：
``` java
/**
 * 客户端调用的房屋登盘申请SPI
 * @nodoc  //此类不生成文档
 * @apidoc //当开启stricMode时，使用apidoc注释的类才会生成文档；
 * @summary 房屋登盘申请
 * @Copyright (c) 2016, Lianjia Group All Rights Reserved.
 */
@FeignClient(name = "loupan-management-server")
public interface HouseRegisterApplySpi {
  /**
   * 经纪人申请房屋登盘，必须指定楼盘名称、栋坐名称、房屋名称。如果是自动提
   * 示所选择的，则可提供楼盘ID、栋坐ID
   * @nodoc    //此方法无需生成文档
   * @author huisman
   * @version v1
   * @param resblockId 楼盘ID
   * @param unitId 单元ID
   * @param resblockName 楼盘名称
   * @param unitName 单元名称
   * @param houseName 房屋名称
   * @param houseAddress 房屋地址，如果有楼盘ID ，那么可能为空
   * @param applyComment 申请备注
   * @param attachPaths 附件路径，多个路径以逗号隔开
   * @param attachFileNames 附件名称，多个路径以逗号隔开，先后顺序和attachPaths相同
   * @param cuser 创建人，当前登录用户
   * @return 申请成功，返回申请结构，包含ID
   * @summary 经纪人申请房屋登盘
   */
  @LorikRest(codes = {20111})
  @RequestMapping(value = "/v1/houses/register/apply", method = RequestMethod.POST)
  HouseRegisterApply applyV1(@RequestParam(value = "resblockId", required = false) Long resblockId,
      @RequestParam(value = "unitId", required = false) Long unitId,
      @RequestParam(value = "resblockName") String resblockName,
      @RequestParam(value = "unitName") String unitName,
      @RequestParam(value = "houseName") String houseName,
      @RequestParam(value = "houseAddress", required = false) String houseAddress,
      @RequestParam(value = "applyComment", required = false) String applyComment,
      @RequestParam(value = "attachPaths", required = false) String attachPaths,
      @RequestParam(value = "attachFileNames", required = false) String attachFileNames,
      @RequestHeader(StandardHttpHeaders.X_Login_UserCode) int cuser);
}
```

### 文档截图
上文示例生成的文档如下图所示：
#### 目录导航
![文档截图-目录]({{book.imagePath}}/parts/chapter2/images/spi-summary-page-left.png)

#### 方法展示
![文档截图-方法展示]({{book.imagePath}}/parts/chapter2/images/spi-method-page-header.png)


#### 参数部分
![文档截图-方法参数]({{book.imagePath}}/parts/chapter2/images/spi-method-page-body.png)


## 普通SpringMVC项目（非Spring Cloud）
我们的文档自动生成也支持普通的SpringMVC项目，所有@Controller和@RestController标注的类的源代码都会被解析。

### 额外配置
由于普通的SpringMVC项目没有虚拟主机地址（一般为英文，可以使用’-’分隔），所以插件需额外配置一个参数：-app “you-app”，这个值一般是配置在API网关的静态路由的path。
``` xml

	<plugin>
		 <groupId>com.dooioo.se.lorik</groupId>
		 <artifactId>maven-apidoc-plugin</artifactId>
		 <version>1.0.5</version>
		 <extensions>true</extensions>
		 <configuration>
			<options>-app "you-app（API网关的虚拟路径）” -appName "${project.name}"</options>
		</configuration>
	</plugin>
```


### 高级功能
#### 覆盖默认的访问域名以及自定义每个方法输出的示例JSON串
在项目根路径（跟pom.xml平级）提供apidoclet.json文件，格式如下：
``` json
{
 "env":{
   "headers":["环境","内网访问域名","外网访问域名"],
   "rows":[["正式环境","http://aroute.dooioo.com","https://oroute.dooioo.com"],
   ["集成环境	","http://aroute.dooioo.cn","https://oroute.dooioo.cn"],
   ["测试环境	","http://aroute.dooioo.net"]]
 },
 "com.lianjia.sh.se.login.spi.core.EmployeeLoginSpi.loginV1.returnJson":  {
   "loginStatus":0,
   "userCode":0,
   "userName":"用户名",
   "companyId":0,
   "createAt":0,
   "expiredIn":0,
   "loginTicket":"客户端持有的ticket,sso验证凭证",
   "lastAccessAt":0
  },
  "com.lianjia.sh.se.login.spi.core.EmployeeLoginSpi.findLoginedEmployeeAndRefreshTicketV1.returnJson":  {
   "loginStatus":0,
   "userCode":0,
   "userName":"用户名",
   "companyId":0,
   "createAt":0,
   "expiredIn":0,
   "loginTicket":"客户端持有的ticket,sso验证凭证",
   "lastAccessAt":0
  },
  "com.lianjia.sh.se.login.spi.core.EmployeeLoginSpi.checkAndRefreshLoginTicketV1.returnJson":  {
   "status":0,
   "message":"登录状态的描述信息",
  }


}
```
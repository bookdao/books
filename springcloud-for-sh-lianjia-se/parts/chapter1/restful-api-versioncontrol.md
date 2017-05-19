<!-- toc -->
### REST API 版本控制策略
 
##### 策略一： 无版本号 (Knot)
平台API永远只有一个版本，所有的用户都必须使用最新的API，API的任何变动都会影响到所有调用方（升级新版本+完整功能测试）。

![无版本号]({{book.imagePath}}/parts/chapter1/images/knot.png)

##### 策略二：以版本号隔离功能 (Point-to-Point)
平台API自带版本号，用户根据自己的需求选择使用对应的API，需要使用新的API特性，用户必须自己升级。

![自带版本号]({{book.imagePath}}/parts/chapter1/images/p2point.png)

##### 策略三: 版本兼容（Compatible）
同策略一：平台API只有一个版本，但是最新版本需要兼容历史版本API的行为。

![版本兼容]({{book.imagePath}}/parts/chapter1/images/comp-vers.png)


API升级时三种策略开销对比图（X轴表示API有多少个版本，Y轴表示API升级的开销）：

![相对开销对比]({{book.imagePath}}/parts/chapter1/images/results.png)

可以看到策略一（knot)最不可行，当API迭代4到5个版本后，这时所有客户端升级新版本并完整测试所有功能的开销已经无法接受了。

策略二（Point-to-Point)的API升级开销相对稳定，需要新功能的客户端升级新版本即可，其他API调用方可以保持不变。

最完美的版本控制策略是策略三（Compatible），但实际项目中，不可避免的会出现无法兼容老版本的情况。

综上，首先，API一定得有版本号，否则升级对于调用方来说将是噩梦。

其次，考虑到实际项目的情况，版本控制需结合策略二和策略三：优先提供统一的兼容版本，如果新版本无法兼容历史版本，则升级版本号。

而对于历史版本的支持最好有时间和用户限制，即老版本API支持到一定时间后就删除，新用户必须使用新版API，否则10来个API版本共存对API的维护人员来说就是一场噩梦了。

以上内容参考自：[understanding-the-costs-of-versioning](http://www.ebpml.org/blog2/index.php/2013/11/25/understanding-the-costs-of-versioning)


### REST API 版本控制实现

#### 利用URL
这种实现方式有两个变种：

##### 版本号是请求路径的一部分  

```GET  http://api.route.dooioo.cn/v1/client/token```
##### 版本号是查询参数的一部分    

```GET  http://api.route.dooioo.cn/client/token?version=1```

这两种方式的优点是：很容易在浏览器里查看不同版本API的输出。
不足之处是：版本号在URI中破坏了REST的HATEOAS（hypermedia as the engine of application state）规则，版本号和资源之间并无直接关系。

#### 自定义Request Header
客户端在发送请求的时候，需要一起发送自定义的Request Header，比如： X-Api-Version或Accept-Version，来指明请求那个版本的接口。

``` http
GET http://api.route.dooioo.cn/client/token
X-Api-Version: 1
```
#### 利用ContentType

ContentType组成：```Content-Type := type "/" subtype *[";" parameter] ```

这种方式利用HTTP标准Request Header： Accept，以表明自己想接收的是什么样的数据，也有两个变种：

#####  方式一    
版本号是ContentType sub type的一部分。

``` http
 GET http://api.route.dooioo.cn/client/token  
 Accept: application/vnd.dooioo.v2+json    
```  
vnd = Vendor (The "vnd" prefix means that the MIME value is vendor specific.)

##### 方式二  
将版本号从MIME的sub type里分离出来，当成MIME的一个参数。
  
``` http
  GET http://api.route.dooioo.cn/client/token  
  Accept: application/vnd.dooioo+json;version=2  
```
更多Accept例子：  

``` Accept: application/json;version=2 ```  

``` Accept: application/vnd.dooioo+xml;version=2 ```  

```Accept: application/xml;version=2```



<!-- toc -->
### Ajax跨域
Web浏览器对跨域（Cross Domain)请求有着严格的安全限制（same-origin security policy 同源安全策略）。  

下图是从网站`stackoverflow.com` 向我们的API网关`aroute.dooioo.cn` 发起的一个跨域Ajax请求，直接被浏览器拒绝：  

```http
  $.get("http://aroute.dooioo.cn/loupan/server/v1/citys")  
```

<br>


![跨域请求拒绝]({{book.imagePath}}/parts/chapter1/images/crossdomain-denied.png)

跨域请求发送时浏览器会添加Request Header: `Origin`，如果服务端检测到有此Header，就可以断定此请求是跨域的。  
  
Origin为请求发起方的主机名。  

![跨域请求头部]({{book.imagePath}}/parts/chapter1/images/crossdomian-headers.png)

Ajax跨域有两种解决方案：`JSONP(JSON with Padding)` 、 `CORS(Cross-origin resource sharing 跨域资源共享)`。

### JSONP
JSONP 是 **JSON** with **P**adding的简称，JSONP被Web开发者用来克服浏览器的同源策略、以获取其他域名提供的数据。

#### JSONP请求流程及原理
假设以下请求：   

```
GET  http://aroute.dooioo.cn/loupan/server/v1/citys/1
```

响应数据为：
```json
 {
   "id": 1,
   "gbCode": 310000,
   "name" : "上海市"
 }
```
如果将此URL赋给HTML标签 &lt;script&gt; 的src：
``` javascript
<script type=“text/javascript"
        src="http://aroute.dooioo.cn/loupan/server/v1/citys/1">
</script>
```
那么浏览器会自动下载脚本文件，解释并执行脚本内容。而此时的JSON响应数据会被当做`JavaScript`代码块，并且抛出js语法错误的异常。

即使JSON数据被浏览器解释为`JavaScript Object` ，但由于缺少变量赋值，比如
 `val={Response Object}`，JS也无法访问响应数据。
 
 当使用JSONP模式时，&lt;script&gt; 属性src的响应数据必须使用JS代码包装（通常是函数调用），比如以下脚本引用：  

 ``` javascript
<script type=“text/javascript"
        src="http://aroute.dooioo.cn/loupan/server/v1/citys/1">
</script>
```
如果：`aroute.dooioo.cn/loupan/server/v1/citys/1`响应数据为：
```json
responseObj= {
   "id": 1,
   "gbCode": 310000,
   "name" : "上海市"
 }
```
那么浏览器下载脚本文件、解释执行之后，全局变量`responseObj`就可以被其他js代码访问了。

也就是说，JSONP能起作用，服务端必须用JS代码包装原始JSON数据后响应。这里包装用的JS代码便被称为：**Padding**。

在实际使用中，客户端通常会预定义一个函数，也叫回调函数，比如:

``` javascript
<script type=“text/javascript">
  function parseResponse(data){
    //操作json数据
  }
</script>
```
函数名通过一个约定的查询参数callback 或者jsonp发送给服务端：

``` javascript
<script type=“text/javascript"
        src="http://aroute.dooioo.cn/loupan/server/v1/citys/1?callback=parseResponse">
</script>
```
服务端使用客户端提供的方法名包装原始数据并响应：
``` json
parseResponse({
   "id": 1,
   "gbCode": 310000,
   "name" : "上海市"
 });
```

浏览器接收到响应数据后，将脚本内容解释为：执行函数调用`parseResponse(data)`。

至此，一次完整的JSONP请求执行完毕。

#### JSONP应用限制
综上，JSONP必须满足以下条件：
* 必须依赖HTML标签 &lt;script&gt;的src属性来发起请求。
* JSONP响应数据本质上是js脚本。
* 客户端和服务端必须相互配合。  
客户端预定义回调函数，将方法名通过约定查询参数callback或jsop传给服务端，服务端使用方法名包装原始响应数据。

如果是动态Ajax请求，客户端必须动态预定义函数，动态创建&lt;script&gt;标签。  

通常使用JavaScript库处理通用逻辑，比如`JQuery`，我们只需关注业务回调函数即可。
以下代码为JQuery JSONP示例：

```javascript
$.ajax({
    // script src
    url: "http://aroute.dooioo.cn/loupan/server/v1/citys/1",
    // 传到服务端的查询参数-函数名
    jsonp: "callback",
    // Tell jQuery we're expecting JSONP
    dataType: "jsonp",
    // Work with the response
    success: function( response ) {
        console.log( response ); // server response
    }
});

```

#### JSONP不足之处
 1. 仅支持Http GET，不支持其他Http Method（`POST/DELETE/PATCH/PUT`)。
 2. 需要服务端配合，服务端需要理解`JSONP语义`并包装原始数据。

#### JSONP优点
 兼容所有版本的浏览器，只要能正确解释执行JavaScript。

### 跨域资源共享（CORS)
 CORS是**C**ross-**O**rigin **R**esource **S**haring的简称。CORS规范定义了一系列 Request Header和Response Header，浏览器和服务端程序根据Request/Response Header做出不同的行为。
 
#### CORS Request Header
* Origin  
   客户端域名
* Access-Control-Request-Method  
   客户端发起跨域请求所用的Http Method
* Access-Control-Request-Headers   
   客户端发起跨域请求时携带的Header

#### CORS Response Header
* Access-Control-Allow-Origin  
   资源提供方允许跨域访问的主机域名，如果不限，则返回*，否则返回具体的域名。
* Access-Control-Allow-Credentials  
   是否允许客户端携带cookie、发送SSL握手以及Http认证数据等。
* Access-Control-Expose-Headers  
   暴露给客户端、客户端跨域可读取的Response Header
* Access-Control-Max-Age  
    预校验请求的缓存时间，单位为妙。
* Access-Control-Allow-Methods  
    允许客户端使用那些Http Method跨域访问当前资源，换句话说，资源支持那些Http Method，多个以逗号隔开，比如： GET,POST,OPTION。
* Access-Control-Allow-Headers  
   资源提供方允许客户端携带那些请求头访问资源。

#### CORS 请求流程

1， 从网站```stackoverflow.com``` 向我们的API网关```aroute.dooioo.cn``` 发起Ajax请求： 
```
$.get("http://aroute.dooioo.cn/loupan/server/v1/citys") 
``` 
 
2， 浏览器发现当前主机域名为：```stackoverflow.com```，但请求的主机域名为：```aroute.dooioo.cn```，断定请求为跨域请求，主动添加Header`Origin`:
```javascript
 Request URL: http://aroute.dooioo.cn/loupan/server/v1/citys
 Request Method: GET
```  
```http  
 Origin: stackoverflow.com
```
3， 浏览器根据CORS规范，判断请求是否为**简单跨域请求** [^1]（Simple Cross-Origin Request)。  

4， 如果不是简单跨域请求，浏览器将发起一个预校验（Preflight）的```OPTION``` 请求。  
**Preflighted请求** [^2]会发送 ```Access-Control-Request-Method```(客户端发起Http请求的Method) 、```Access-Control-Request-Headers```（客户端自定义的Request Header）,服务端根据Header判断跨域请求是否安全：
 ```javascript
 Request URL: http://aroute.dooioo.cn/loupan/server/v1/citys
 Request Method: OPTION
 ```
 ```http  
 Origin: stackoverflow.com 
 Access-Control-Request-Method: GET
 Access-Control-Request-Headers: X-Api-Version  
 ```
 服务端校验不通过，可直接响应```401/403```，跨域请求被拒绝。
 
 校验通过之后，可响应以下Response Header： 
 ```http 
 Access-Control-Allow-Credentials: false  
 Access-Control-Allow-Origin: *  
 Access-Control-Allow-Methods: GET,POST,OPTION 
 Access-Control-Allow-Headers: X-Api-Version  
 Access-Control-Max-Age: 3600 
```   

非简单跨域请求每次都会发起Preflight请求,```Access-Control-Max-Age```响应头指示客户端可以将校验请求缓存多久，单位为秒。

5，浏览器发起实际Ajax请求，服务端必须添加响应头：  
```http
Access-Control-Allow-Origin: stackoverflow.com
```  
Access-Control-Allow-Origin值可为*（允许所有网站访问）或明确指定的域名。  

如果服务端允许客户端发送Http Authentication 数据、客户端SSL、以及Cookies数据，则可以响应：
```http  
Access-Control-Allow-Credentials: true  
``` 
默认情况下，Ajax请求仅能读取标准HTTP 响应头，服务端也可以限定暴露给客户端访问的Response Header：
```http
Access-Control-Expose-Headers: X-Intance-Id,X-Login-Token  
```

6，浏览器判断服务端响应的```Access-Control-Allow-Origin```的值是否包含当前Host：```stackoverflow.com```，包含则意味着可以跨域访问资源。

另外，* 意味着所有主机都可以访问此域名。

**如果此响应头不存在或响应头不包括当前主机，浏览器直接丢弃数据，拒绝跨域请求**。 

下图为简单跨域请求：  

![CORS简单跨域请求]({{book.imagePath}}/parts/chapter1/images/cors-simple-request.png)

#### CORS请求流程图（节选自Wiki）
![CORS请求流程图]({{book.imagePath}}/parts/chapter1/images/cors.png)
 
#### CORS 应用限制
 可以看到，使用CORS实现跨域时，浏览器必须支持CORS规范，服务端也必须支持CORS规范。
 
 目前以下版本的浏览器都支持CORS规范：
*  基于Chromium的浏览器（Chrome 28+，Opera 15+，Android's 4.4+ WebView）。
* Internet Explorer 10内置支持。Internet Explorer 8 & 9通过XDomainRequest提供部分支持。
* 基于WebKit的浏览器（Safari 4 及以上）

服务端（资源方）通常使用`Filter`拦截请求，在过滤器里按照CORS规范解析并响应请求。Tomcat 内置了```org.apache.catalina.filters.CorsFilter```，另外，lorik-core也提供了实现：[lorik-core: CORSFilter](https://github.com/huisman6/lorik/blob/master/lorik-core/src/main/java/com/dooioo/se/lorik/core/web/filter/CORSFilter.java)。

如果你想实现自己的CORSFilter，请参考 [W3c CORS 规范](http://www.w3.org/TR/cors/)。另外，请不要在Spring拦截器里解析CORS请求，因为Tomcat会自动响应```OPTION```请求。

#### CORS 优点和不足
##### CORS的优点
  1. 支持所有Http Method( ```GET/POST/HEAD/PUT/DELETE/PATCH/OPTION```)。
  2. 仅配置CORS过滤器即可，业务接口无需关心跨域问题、不用改变响应数据，这一切对业务接口来说是透明的。

##### CORS的不足
   低版本的浏览器不支持，比如IE6/IE7，IE8/IE9仅提供部分支持（仅支持GET/POST)。  
   IE10已全部支持CORS规范。
   
   因此，如果客户端浏览器是Chrome/Safari/FireFox/IE10，**使用CORS解决跨域请求是首选**。 
   
####  CORS 术语（Terminology）
 
##### 简单方法 （Simple Method)  
HTTP METHOD为： ```GET/HEAD/POST``` ，大小写敏感。
##### 简单头部 （Simple Header)  
Request Header为```Accept,Accept-Language,Content-Language```，以及Content-Type MIME类型为 ```application/x-www-form-urlencoded, multipart/form-data,  text/plain```的Header，大小写不敏感。
##### 简单跨域请求（Simple Cross-origin Request)  
请求方法为**简单方法**并且请求头为**简单头部**的请求。也就是说简单跨域请求至少满足两个条件： Http Method 必须为```GET、HEAD、POST```之一，可被手动设置的Request Header为```Accept、Accept-Language/Content-Language以及Content-Type```（Content-Type可选值为```application/x-www-form-urlencoded、 multipart/form-data、text/plain```)。

##### 简单响应头(Simple Response Header)  
Response Header为```Cache-Contro,Content-Language,Content-Type,Expires,Last-Modified,Pragma```，大小写不敏感。

##### 预校验请求（Preflighted Request)  
所有非简单请求，必须先发送HTTP METHOD 为 ```OPTION```的Preflighted 请求，以决定是否可以安全的发送实际请求。  
符合以下任一条件都会发送预校验请求：
*   Http METHOD不是```GET、HEAD、POST```，比如```PUT/DELETE/PATCH```；
* 如果使用POST发送请求，但Cotent-Type不是  
```application/x-www-form-urlencoded、multipart/form-data、text/plain```，比如```application/xml、text/xml```；
* 设置自定义请求头，比如 ```X-Api-Version```。  

下面代码浏览器就会发起预校验请求：  

``` javascript
     function callOtherDomain(){
     if(invocation)
     {
      invocation.open('POST', url, true);
      invocation.setRequestHeader('X-Api-Version', '1');
      invocation.setRequestHeader('Content-Type', 'application/xml');
      invocation.onreadystatechange = handler;
      invocation.send(body); 
    }
}  
```
<br/>  

[^1]: 简单跨域请求，指请求方法为**简单方法**并且请求头为**简单头部**的请求。也就是说简单跨域请求至少满足两个条件： Http Method 必须为GET、HEAD、POST之一，可被手动设置的Request Header为Accept、Accept-Language/Content-Language以及Content-Type（Content-Type可选值为application/x-www-form-urlencoded、 multipart/form-data、text/plain)。

[^2]: 预校验请求（Preflighted Request)指所有非简单请求，符合以下任一条件都会发送预校验请求： Http METHOD不是GET、HEAD 或者POST，比如PUT/DELETE/PUT/PATCH；如果使用POST发送请求，但Cotent-Type不是application/x-www-form-urlencoded、multipart/form-data 或text/plain，比如application/xml、text/xml；设置自定义请求头，比如 X-Api-Version。
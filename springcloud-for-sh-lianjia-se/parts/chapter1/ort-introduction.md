<!-- toc -->
### 苍梧国
苍梧国居民总体分为两类：天龙人和普通市民。  
  
黄色区域，统称为区域T，学术名称： Trusted Zone，居住着特权阶级：天龙人，他们垄断着宇宙的知识，个体可浏览任何知识，信息对其是透明、自由的。
  

蓝色区域，统称为区域R，学术名称：Restricted Zone，居住着普通市民，市民有自己的身份证，受法律约束。  
  
市民只能通过红色的门，俗称龙门，去区域T查询公共资料（可公开访问的接口），但是某些会危及统治的资料（授权接口），市民必须实名浏览，以便监控。

市民的身份证件如下所示：
  ![市民身份证]({{book.imagePath}}/parts/chapter1/images/identity-card-R.png)

  

白色区域，统称为区域O，学术名称： Overseas Zone，为他国的领土。  
区域O的他国公民，必须先申请护照，然后办理签证，才能访问苍梧国。

颁发护照的机构，称为出入境管理中心，即地图中区域O和R交界处的房型Icon。


苍梧国的护照如下所示：
![护照]({{book.imagePath}}/parts/chapter1/images/passport_O.png)


###  Trusted Zone
我们的机房局域网，机房内的所有服务默认是受信任的，服务之间可以任意访问彼此的数据。
通常情况下，服务之间通过SPI互相访问。

机房服务器所在网段和其他网络隔离。

### Restricted Zone
受限制区域，包括门店电脑，总部电脑、员工等，是除了机房服务器之外的整个局域网设备。

### 龙门
API网关（[http://aroute.dooioo.com](http://aroute.dooioo.com))，Restricted Zone的客户端必须通过API网关访问机房的服务。

API网关也可以称作接入层。

### Overseas Zone
外国领土，海外华人华侨，包括Andriod App、IOS App、定制机、外网（公网）产品比如链家网、境外恶势力。

### 出入境管理中心
OAuth服务（[https://oroute.dooioo.com](https://oroute.dooioo.com))，Overseas Zone的客户端
必须通过OAuth服务申请护照（登记），办理签证（申请AccessToken)，之后才能访问苍梧国。














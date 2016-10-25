<!-- toc -->
##微服务落地：规范、外部配置、调试、性能、高可用
### MicroService，What the hell?
今天我来和大家来分享我们微服务的编码规范，以及在微服务落地过程中的代码演化。


### 调试：基于Http RPC
目前版本的微服务仅支持基于Http协议、文本序列化（JSON)的RPC调用。
#### 调用方（客户端）
服务调用方最常用的调试，是查看RPC调用时构造的Http请求以及服务提供方的响应。  
  
##### 核心类：SynchronousMethodHandler
最核心的代码，在这个类：`feign.SynchronousMethodHandler`。

以下是RPC请求的核心代码，IDE Debug模式可查看方法调用时的实参：
```java
 @Override
  public Object invoke(Object[] argv) throws Throwable {
    // argv 是方法调用的实际参数，Feign根据实际参数替换Http模板请求的占位符
    // 占位符包括：@RequestParam、@RequestHeader注解的参数
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    //请求重试？
     Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        //发起Http请求，同时解码（反序列化）响应
        return executeAndDecode(template);
      } catch (RetryableException e) {
        //是否需要重试（重新请求） 或者抛出错误（结束方法调用）？
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```  
 
##### Http请求和反序列化
Http请求和反序列化的实现，忽略部分代码：
``` java
Object executeAndDecode(RequestTemplate template) throws Throwable {
 	//构造Http请求，这里可以查看构造好的Http请求
 	//Uri、Http Request Header、Request Body等
    Request request = targetRequest(template);
    Response response;
    try {
      //可以查看使用了那种HttpClient，默认基于JDK的URLConnection
      //新版本的lorik-spi-view使用Apache Http Client。
      //options是请求的一些选项，比如请求超时时间、读取超时时间
      response = client.execute(request, options);
    } catch (IOException e) {
       //忽略代码：请求出错，直接抛出异常
    }
   try {
      //服务端响应2xx
      if (response.status() >= 200 && response.status() < 300) {
        if (void.class == metadata.returnType()) {
          return null;
        } else {
         //解码（JSON反序列化）Response Body，这里可查看服务端的响应的信息
          return decode(response);
        }
      } else if (decode404 && response.status() == 404) {
        //如果客户端配置了解码404，
        //则解码响应信息；否则使用错误解码器统一处理。
        return decoder.decode(response, metadata.returnType());
      } else {
       //如果服务端响应3xx以上状态码，
       //使用错误解码器解码服务端响应的Response Body
       //我们lorik-spi-view就是通过自定义的SpiErrorDecoder
       //将服务提供方的业务码传播到客户端（调用方）
        throw errorDecoder.decode(metadata.configKey(), response);
      }
    } catch (IOException e) {
      //忽略异常处理和流关闭
    }
  }
```

#### 服务提供方
服务提供方是基于SpringMVC实现的Rest服务，最常见的调试就是获取服务接口的请求参数。

##### 核心类：ServletInvocableHandlerMethod
最核心的代码，在这个类：`ServletInvocableHandlerMethod`。

以下是完成Request => Response转换的核心代码：

``` java
public void invokeAndHandle(ServletWebRequest webRequest,
			ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
		//完成方法调用：request参数转换为Java Method的参数
	    //通过Java反射method.invoke(obj,args)获取返回值
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		//设置响应的http状态码
		//这个类构造函数会尝试根据@ResponseStatus初始化http状态码
		setResponseStatus(webRequest);

		//如果方法调用返回null，则标识请求已处理。
		//其实是为了标识这个请求已不需要序列化returnValue了。
		if (returnValue == null) {
		  //忽略相关代码
			mavContainer.setRequestHandled(true);
			return;
		}
		//方法返回值了，进一步处理，因为方法可能返回的各种类型的返回值
		//比如ResponseEntity、Callable、HystrixCommand、Model
		mavContainer.setRequestHandled(false);
		try {
		   //HandlerMethodReturnValueHandler专门用来序列化方法的返回值   
		   //有兴趣的同学可以查看下所有子类，包括如何处理异步。
		   this.returnValueHandlers.handleReturnValue(
					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		}
		catch (Exception ex) {
		  //接口返回值序列化出错了，抛出异常
		  throw ex;
		}
	}
```

##### 接口请求参数 => 方法参数转换
实现接口请求参数 => 方法参数转换的代码：
``` java
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {
		//method的所有参数
		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			//设置参数名，根据参数名去Request中查找参数值。
parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			//解析参数类型
			GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
//所有Http参数 => 方法参数的转换都是通过HandlerMethodArgumentResolver
//有兴趣的同学可查看所有子类，了解下@RequestParam、@RequestHeader等所有
//SpringMVC相关的Http注解如何解析和转换为方法实参的
						if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
				   //找到合适的Resolver用来解析方法参数。
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
				throw new IllegalStateException(msg);
			}
		}
		return args;
	}
```

##### 在线诊断：Trace Endpoint
SpringBoot Actuator默认提供了调试Endpoint: http://host:port/admin/trace。

接口响应如下：

``` json
[
	{
		timestamp: 1477298109627,
		info: {
				method: "GET",
				path: "/v1/company/3/gbCode",
				headers: {
					 request: {
						x-client-id: "10.22.16.35:9500",
						x-client-name: "loupan-search",
						x-client-security-token: "66625864-cae3-4492-bfa6-085de365abf8",
						accept: "*/*",
						user-agent: "Java/1.8.0_40",
						host: "10.22.16.36:9600",
						connection: "keep-alive"
				      },
					 response: {
					  	 X-Instance-Id: "b7b0fc1f593866afe230f4af6117b5ee",
						 X-Application-Context: "loupan-server:production:9600",
						 Content-Type: "application/json;charset=UTF-8",
						 Transfer-Encoding: "chunked",
						 Vary: "Accept-Encoding",
						 Date: "Mon, 24 Oct 2016 08:35:09 GMT",
						 status: "200"
			}
		}
	}
}
]

```

Trace Endpoint只记录最近的100条接口请求，根据请求的时间戳倒排，

由于此接口会泄露敏感信息，尤其是UI项目，生产环境建议禁用此功能（在配置文件中增加property:`endpoints.trace.enabled:false`)。


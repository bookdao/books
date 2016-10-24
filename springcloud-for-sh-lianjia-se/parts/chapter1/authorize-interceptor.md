### 老项目接口拦截器的实现
``` java
/**
 * 所有被调用的SPI接口方法，我们会检查是否登录过（登录后有token）。<br>
 * 目前仅检查API网关路由过来的请求，不拦截Feign RPC调用。
 * 
 * @author huisman
 * @since 1.0.10
 * @Copyright (c) 2015, Lianjia Group All Rights Reserved.
 */
public class AuthorizedRequestInterceptor extends HandlerInterceptorAdapter {
  private static final Log logger =
      LogFactory.getLog(MethodHandles.lookup().lookupClass());
  private final Set<DispatcherType> supportDispatchTypes = new HashSet<>(
      Arrays.asList(DispatcherType.ASYNC, DispatcherType.REQUEST));


  public AuthorizedRequestInterceptor() {
    super();

    logger.info(
        "============**********************************************************************  ================");
    logger.info(
        "============  com.dooioo.se.lorik.spi.view.authorize.LoginNeedless  ================");
    logger.info(
        "============  启用对API网关路由过来的请求的登录检查                 ================");
    logger.info(
        "============  无需登录校验请在SPI接口方法中添加：@LoginNeedless           ================");
    logger.info(
        "============                                                                                         ================");
    logger.info(
        "============***********************************************************************================");

  }

  @Override
  public boolean preHandle(HttpServletRequest request,
      HttpServletResponse response, Object handler) throws Exception {
    // 仅支持Request请求，内部Include,Forward,error都将被忽略
    // 如果不是DispatcherType不是正常的Request/ASYNC，则pass
    if (!this.supportDispatchTypes.contains(request.getDispatcherType())) {
      return true;
    }
    // 路由节点的ID
    String routeHeader = request.getHeader(StandardHttpHeaders.X_Route_By);
    // 即使 传过来的值为“”也要校验登录token
    if (routeHeader == null) {
      // 如果不是api 路由过来的，则直接pass
      return true;
    }
    // 判断是否需要授权，仅处理HandlerMethod
    if (handler instanceof HandlerMethod) {
      HandlerMethod hm = (HandlerMethod) handler;
      // 检查method 是否有@LoginNeedless
      if (AnnotationUtils.findAnnotation(hm.getMethod(),
          LoginNeedless.class) != null) {
        return true;
      }

      // 检查标准请求头，我们需要校验员工是否登录，登录后会有token
      String loginToken = request.getHeader(StandardHttpHeaders.X_Login_Token);

      // 如果需要登录，并且请求还未提交
      if (!StringUtils.hasText(loginToken) && !response.isCommitted()) {
        // first 检查头部是否有工号？
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.setContentType(
            "application/json;charset=" + StandardCharsets.UTF_8.name());

        StringBuilder message = new StringBuilder(130);
        message.append("{\"code\":")
            .append(SpiBuiltinBizCode.API_GATEWAY_UNLOGIN.getCode())
            .append(",\"message\":\""
                + SpiBuiltinBizCode.API_GATEWAY_UNLOGIN.getMessage() + "\"")
            .append(",\"timestamp\":").append(System.currentTimeMillis())
            .append(",\"path\":\"" + request.getRequestURI() + "\"}");

        response.getWriter().write(message.toString());
        response.flushBuffer();
        return false;
      }
    }
    return true;
  }

  /**
   * 标准请求或响应头
   */
  public interface StandardHttpHeaders {
    /**
     * 员工登录后请求头会自动添加 此人登录时分配的token
     */
    String X_Login_Token = "X-Login-Token";
    /**
     * 员工登录后请求头会自动添加 登录人员的工号，工号为上海链家员工工号（5位或6位），如果升级8位或其他位数会特别说明
     */
    String X_Login_UserCode = "X-Login-UserCode";
    /**
     * 员工登录后请求头会自动添加 登录工号关联的公司ID，比如苏州链家、上海链家公司的ID
     */
    String X_Login_CompanyId = "X-Login-CompanyId";

    /**
     * 如果一个接口是通过API网关路由过来的，则API网关会自动在请求头、响应头里添加此HttpHeader X-Route-By是API
     * 网关节点IP地址的加密，解密请访问：http://api.route.dooioo.org(com)/instance/{instanceId}
     */
    String X_Route_By = "X-Route-By";
  }

  /**
   * 无需登录的方法，请在Controller上添加此标识
   */
  @Documented
  @Retention(RetentionPolicy.RUNTIME)
  @Target(value = {ElementType.METHOD})
  public @interface LoginNeedless {
  }

  /**
   * 内置业务码
   */
  public interface SpiBuiltinBizCode {

    /**
     * API网关未登录，当报此错误时，需要先向API网关申请Token
     */
    BizCode API_GATEWAY_UNLOGIN = new BizCode(900001, "API网关未登录");
  }

  /**
   * 标准业务码
   * 
   * @author huisman
   */
  public static final class BizCode {
    private int code;
    private String message;

    public BizCode(int code, String message) {
      super();
      this.code = code;
      if (message == null) {
        message = "";
      }
      this.message = message;
    }

    public int getCode() {
      return this.code;
    }

    public String getMessage() {
      return this.message;
    }

    /**
     * 将多个业务码组成字符串输出，可以用在 {@link LorikRest#codes()}
     * 
     * @param results
     * @return 业务码字符串，比如：“21000,21020"
     */
    public static String toCodes(BizCode... results) {
      if (results == null || results.length == 0) {
        return "";
      }
      StringBuilder message = new StringBuilder();
      for (BizCode restResult : results) {
        message.append("," + restResult.getCode());
      }
      return message.deleteCharAt(0).toString();
    }
  }
}
```
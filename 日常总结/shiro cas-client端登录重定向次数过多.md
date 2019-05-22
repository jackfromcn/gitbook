# Shiro Cas-client端登录重定向次数过多

> 背景：最快上线前，接到反馈说 `shrio cas` 登录验证成功后，回调出现重定向次数过多的错误提示。这个问题之前出现过，当时搭建项目框架中，使用 `springboot` 同时需要兼顾之前项目需要的 `jar` 包依赖，对 `shiro` 框架也不熟，只是根据之前项目配置，改成了 `springboot` 的 `@Bean` 方式实现了。 

## 之前出现重定向次数过多的问题，怎么解决的

最开始出现这个问题时，也找了好久，后来试了一下将 `chrome` 浏览器开启无痕模式后就解决了。于是也没有继续往下深入了。



## 这次出现这个问题时，怎么定位的

出现这个问题后，和上次出现这个问题时一样，第一时间将 `chrome` 浏览器调到无痕模式，发现还是同样的问题。

于是通过远端 `debug` 发现自定义的 `CasRealm` 的获取**身份验证信息**方法 `doGetAuthenticationInfo` 内部调用 `rpc` 服务时，发现业务不满足，自己抛出了 `AuthenticationException`（其他异常也会包装成 `AuthenticationException` 异常抛出）。

但是也没有觉得奇怪。因为为了内部调试方便，自己也写了一个 `mock` 登录的方法，通过用户名密码方式登录，如果发生异常，会将异常信息包装成 `json` 体返回。

于是试探性的将 `rpc` 接口的返回值通过 `mock` 数据的方式返回，发现这个问题果然解决了。



## 疑问？为什么 `CasRealm#doGetAuthenticationInfo` 方法中抛出 `AuthenticationException` 异常会导致重复重定向的问题呢？

之前的老项目中，其项目使用 `tomcat` 方式部署，虽然现在项目改成了 `springboot`，但是 `shiro` 依赖以及其配置都是和原来一样，为什么之前项目就可以，在 `CasRealm#doGetAuthenticationInfo` 方法中抛出 `AuthenticationException` 异常，而且没有见过类似重复重定向的问题。

即使在 `CasRealm#doGetAuthenticationInfo` 方法中抛出了 `AuthenticationException` 异常，有什么方法可以像自己 `mock` 登录一样返回异常 `json` 体，而不是一直重复重定向。

`mock` 登录的代码是在 `controller` 中定义 `@RequestMapping`  实现的，其**获取身份信息**方法即使抛出异常，其异常栈也是 `controller` 层，而项目中定义了其异常处理的 `@ControllerAdvice`，因此可以包装成提前定义的 `json` 体。

但是 `cas` 登录，则是通过 `cas server` 端通过回调的形式来触发的，项目配置了 `CasRealm`，拦截了回调 `url`，其实现由框架来控制。于是考虑在最外层的 `filter` 中对 `filterChain.doFilter(servletRequest, servletResponse);` 进行 `try catch`，并且进行包装。

但是 `catch` 中无论是 `Exception` 还是 `Throwable` 都不见其被触发。于是猜想是框架内部进行了遗产处理。

跟踪进入 `org.apache.shiro.cas.CasFilter#onAccessDenied`方法。方法如下：<br/>

```java
@Override
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    return executeLogin(request, response);
}

protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
    AuthenticationToken token = createToken(request, response);
    if (token == null) {
        String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                "must be created in order to execute a login attempt.";
        throw new IllegalStateException(msg);
    }
    try {
        Subject subject = getSubject(request, response);
        subject.login(token);
        return onLoginSuccess(token, subject, request, response);
    } catch (AuthenticationException e) {
        return onLoginFailure(token, e, request, response);
    }
}
```

 `shiro` 框架内部对 `AuthenticationException` 异常进行了处理，要想自定义异常返回，需要继承 `CasFilter`，并重写 `executeLogin` 方法。实现如下：<br/>

```java
@Override
protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
    AuthenticationToken token = createToken(request, response);
    if (token == null) {
        String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                "must be created in order to execute a login attempt.";
        throw new IllegalStateException(msg);
    }
    try {
        Subject subject = getSubject(request, response);
        subject.login(token);
        return onLoginSuccess(token, subject, request, response);
    } catch (AuthenticationException exception) {
        HttpServletRequest req = (HttpServletRequest) request;
        HttpServletResponse resp = (HttpServletResponse) response;
        log.error("CORSFilter request AuthenticationException, uri={}, args={}, remoteIP={}",
                req.getRequestURI(),
                JsonUtil.toJson(request.getParameterMap()),
                IPUtils.getClientIpAddr(req),
                exception);
        Message message;
        if (StringUtils.isBlank(exception.getMessage())) {
            message = Message.fail(ErrorCode.DATA_PERMISSION_INSUFFICIENT);
        } else {
            message = Message.fail(ErrorCode.DATA_PERMISSION_INSUFFICIENT.getCode(), exception.getMessage());
        }
        resp.setStatus(HttpStatus.UNAUTHORIZED.value());
        resp.setContentType("application/json; charset=utf-8");
        resp.getWriter().write(JsonUtil.toJson(message));
//            return onLoginFailure(token, exception, request, response);
        return false;
    }
}
```



## 在 `CasRealm#doGetAuthenticationInfo` 定义逻辑上的无权限空对象，避免抛出异常，框架重新想 `cas server` 端重新发起校验

除了调用使用 `cas server`  返回的 `ticket` 凭证获取用户信息，并且解析。即，`org.jasig.cas.client.validation.TicketValidator#validate` 方法调用时抛出的 `TicketValidationException`，其他自定义异常全部 `try catch` 后忽略，返回默认空权限对象。



## 解决一个重复重定向问题，又来一个 302 Found 问题

重复重定向的问题，主要是在**获取用户身份信息**的方法中，抛出了异常。网上一些 `shiro` 的 `demo` 大多是通过用户名和密码在本系统通过调用 `mysql` 服务校验的，即使发生异常，异常也会体现在调用的 `controller` 方法中，用户能即使的到反馈。不会有大问题。

但是接入 `cas server` 后，默认框架在抛出 `AuthenticationException` 异常后，会默认重定向到登录失败页面，而本项目中，配置的登录失败的页面是登录页。因此会出现这个重复重定向的问题。



后面出现的 `302` 问题在下面介绍。
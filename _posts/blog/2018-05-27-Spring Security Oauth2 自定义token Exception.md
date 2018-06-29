---
layout: post
title: Spring Security Oauth2 自定义 OAuth2 Exception
categories: Security
description: Security
keywords: Security
---

> 付出就要得到回报，这种想法是错的。

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/spring-security-OAuth208.png](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth208.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth208.png")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth208.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-OAuth208.png")

## 前言 ##

在使用`Spring Security Oauth2`登录和鉴权失败时，默认返回的异常信息如下
```java
{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}
```
。它与我们自定义返回信息不一致，并且描述信息较少。那么如何自定义`Spring Security Oauth2`异常信息呢，下面我们简单实现以下。格式如下：
```java

{
"error": "400",
"message": "坏的凭证",
"path": "/oauth/token",
"timestamp": "1527432468717"
}
```

### 自定义登录失败异常信息 ##

#### 新增CustomOauthException

- 添加自定义异常类，指定`json`序列化方式

```java
@JsonSerialize(using = CustomOauthExceptionSerializer.class)
public class CustomOauthException extends OAuth2Exception {
    public CustomOauthException(String msg) {
        super(msg);
    }
}
```
#### 新增CustomOauthExceptionSerializer

- 添加`CustomOauthException`的序列化实现

```java
public class CustomOauthExceptionSerializer extends StdSerializer<CustomOauthException> {
    public CustomOauthExceptionSerializer() {
        super(CustomOauthException.class);
    }

    @Override
    public void serialize(CustomOauthException value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();

        gen.writeStartObject();
        gen.writeStringField("error", String.valueOf(value.getHttpErrorCode()));
        gen.writeStringField("message", value.getMessage());
//        gen.writeStringField("message", "用户名或密码错误");
        gen.writeStringField("path", request.getServletPath());
        gen.writeStringField("timestamp", String.valueOf(new Date().getTime()));
        if (value.getAdditionalInformation()!=null) {
            for (Map.Entry<String, String> entry : value.getAdditionalInformation().entrySet()) {
                String key = entry.getKey();
                String add = entry.getValue();
                gen.writeStringField(key, add);
            }
        }
        gen.writeEndObject();
    }
}

```

#### 添加CustomWebResponseExceptionTranslator

- 添加`CustomWebResponseExceptionTranslator`，登录发生异常时指定`exceptionTranslator`

```java
public class CustomOauthExceptionSerializer extends StdSerializer<CustomOauthException> {
    public CustomOauthExceptionSerializer() {
        super(CustomOauthException.class);
    }

    @Override
    public void serialize(CustomOauthException value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();

        gen.writeStartObject();
        gen.writeStringField("error", String.valueOf(value.getHttpErrorCode()));
        gen.writeStringField("message", value.getMessage());
//        gen.writeStringField("message", "用户名或密码错误");
        gen.writeStringField("path", request.getServletPath());
        gen.writeStringField("timestamp", String.valueOf(new Date().getTime()));
        if (value.getAdditionalInformation()!=null) {
            for (Map.Entry<String, String> entry : value.getAdditionalInformation().entrySet()) {
                String key = entry.getKey();
                String add = entry.getValue();
                gen.writeStringField(key, add);
            }
        }
        gen.writeEndObject();
    }
}

```

#### 修改MerryyouAuthorizationServerConfig

- 指定自定义`customWebResponseExceptionTranslator`

```java
@Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(tokenStore)
                .authenticationManager(authenticationManager)
                .userDetailsService(userDetailsService);
        //扩展token返回结果
        if (jwtAccessTokenConverter != null && jwtTokenEnhancer != null) {
            TokenEnhancerChain tokenEnhancerChain = new TokenEnhancerChain();
            List<TokenEnhancer> enhancerList = new ArrayList();
            enhancerList.add(jwtTokenEnhancer);
            enhancerList.add(jwtAccessTokenConverter);
            tokenEnhancerChain.setTokenEnhancers(enhancerList);
            //jwt
            endpoints.tokenEnhancer(tokenEnhancerChain)
                    .accessTokenConverter(jwtAccessTokenConverter);
        }
        endpoints.exceptionTranslator(customWebResponseExceptionTranslator);
    }

```

### 自定义Token异常信息 ##

#### 添加AuthExceptionEntryPoint
-  自定义`AuthExceptionEntryPoint`用于`tokan`校验失败返回信息

```java
public class AuthExceptionEntryPoint implements AuthenticationEntryPoint {


    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException)
            throws  ServletException {

        Map map = new HashMap();
        map.put("error", "401");
        map.put("message", authException.getMessage());
        map.put("path", request.getServletPath());
        map.put("timestamp", String.valueOf(new Date().getTime()));
        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        try {
            ObjectMapper mapper = new ObjectMapper();
            mapper.writeValue(response.getOutputStream(), map);
        } catch (Exception e) {
            throw new ServletException();
        }
    }
}
```

#### 添加CustomAccessDeniedHandler
- 授权失败(forbidden)时返回信息

```java
@Slf4j
@Component("customAccessDeniedHandler")
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Autowired
    private ObjectMapper objectMapper;

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        response.setContentType("application/json;charset=UTF-8");
            Map map = new HashMap();
            map.put("error", "400");
            map.put("message", accessDeniedException.getMessage());
            map.put("path", request.getServletPath());
            map.put("timestamp", String.valueOf(new Date().getTime()));
            response.setContentType("application/json");
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write(objectMapper.writeValueAsString(map));
    }
}
```
#### 修改MerryyouResourceServerConfig

```java
    @Override
    public void configure(ResourceServerSecurityConfigurer resources) throws Exception {
        resources.authenticationEntryPoint(new AuthExceptionEntryPoint())
        .accessDeniedHandler(CustomAccessDeniedHandler);
    }
```

### 效果如下

#### 登录异常
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth209.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth209.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth209.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth209.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth209.gif")

#### token异常
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth210.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth210.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth210.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth210.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth210.gif")

#### 禁止访问
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth211.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth211.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth211.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth211.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth211.gif")

#### token失效
[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth212.gif](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth212.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth212.gif")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth212.gif "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/security/spring-security-oauth212.gif")


## 代码下载 ##

- github:[https://github.com/longfeizheng/security-oauth2](https://github.com/longfeizheng/security-oauth2)
- gitee:[https://gitee.com/merryyou/security-oauth2](https://gitee.com/merryyou/security-oauth2)

## 推荐文章
1. [Java创建区块链系列](https://longfeizheng.github.io/categories/#%E5%8C%BA%E5%9D%97%E9%93%BE)
2. [Spring Security源码分析系列](https://longfeizheng.github.io/categories/#Security)
3. [Spring Data Jpa 系列](https://longfeizheng.github.io/categories/#JPA)
4. [【译】数据结构中关于树的一切（java版）](https://longfeizheng.github.io/2018/04/16/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E4%B8%AD%E4%BD%A0%E9%9C%80%E8%A6%81%E7%9F%A5%E9%81%93%E7%9A%84%E5%85%B3%E4%BA%8E%E6%A0%91%E7%9A%84%E4%B8%80%E5%88%87/)
5. [SpringBoot+Docker+Git+Jenkins实现简易的持续集成和持续部署](https://longfeizheng.github.io/2018/04/22/SpringBoot+Docker+Git+Jenkins%E5%AE%9E%E7%8E%B0%E7%AE%80%E6%98%93%E7%9A%84%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90%E5%92%8C%E9%83%A8%E7%BD%B2/)

---

[![https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/wechat/xiaochengxu.png](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/wechat/xiaochengxu.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/wechat/xiaochengxu.png")](https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/wechat/xiaochengxu.png "https://raw.githubusercontent.com/longfeizheng/longfeizheng.github.io/master/images/wechat/xiaochengxu.png")

> 🙂🙂🙂关注微信小程序**java架构师历程**
上下班的路上无聊吗？还在看小说、新闻吗？不知道怎样提高自己的技术吗？来吧这里有你需要的java架构文章，1.5w+的java工程师都在看，你还在等什么？
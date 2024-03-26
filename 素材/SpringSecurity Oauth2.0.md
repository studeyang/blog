# 登入功能

WebFluxSecurityConfiguration

![img](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17005328718969.png)

### jwt在哪里生成？

```
1. /login/oauth2/code/{id}
接口代码在：AuthenticationWebFilter

2. code换token代码追踪：
org.springframework.security.authentication.DelegatingReactiveAuthenticationManager#authenticate
代理了(OidcAuthorizationCodeReactiveAuthenticationManager, OAuth2LoginReactiveAuthenticationManager)
org.springframework.security.oauth2.client.authentication.OAuth2LoginReactiveAuthenticationManager#authenticate (1)
org.springframework.security.oauth2.client.authentication.OAuth2AuthorizationCodeReactiveAuthenticationManager#authenticate

org.springframework.security.oauth2.client.endpoint.ReactiveOAuth2AccessTokenResponseClient#getTokenResponse
org.springframework.security.oauth2.client.endpoint.AbstractWebClientReactiveOAuth2AccessTokenResponseClient#getTokenResponse

3. 获取用户信息：
回到上面标识(1)处，会调用 org.springframework.security.oauth2.client.userinfo.ReactiveOAuth2UserService#loadUser

4、将token放到Cookie(security_context)中：
com.casstime.rc.cloud.webagent.security.context.CookieSecurityContextRepository#save
```

### 在哪里进行jwt失效校验的？

# 登出 token 失效

![image2023-12-19_13-46-24](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image2023-12-19_13-46-24.png)

![image20](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/image20.png)
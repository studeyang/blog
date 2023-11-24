WebFluxSecurityConfiguration

![img](https://technotes.oss-cn-shenzhen.aliyuncs.com/2023/%E4%BC%81%E4%B8%9A%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_17005328718969.png)

```
1. /login/oauth2/code/{id}
接口代码在：AuthenticationWebFilter

2. code换token代码追踪：
org.springframework.security.authentication.ReactiveAuthenticationManager#authenticate
org.springframework.security.oauth2.client.authentication.OAuth2LoginReactiveAuthenticationManager#authenticate

org.springframework.security.authentication.ReactiveAuthenticationManager#authenticate
org.springframework.security.oauth2.client.authentication.OAuth2AuthorizationCodeReactiveAuthenticationManager#authenticate

org.springframework.security.oauth2.client.endpoint.ReactiveOAuth2AccessTokenResponseClient#getTokenResponse
org.springframework.security.oauth2.client.endpoint.AbstractWebClientReactiveOAuth2AccessTokenResponseClient#getTokenResponse

3. 将token放到Cookie(security_context)中：
com.casstime.rc.cloud.webagent.security.context.CookieSecurityContextRepository#save
```
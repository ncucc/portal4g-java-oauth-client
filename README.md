# OAuth2 for NCU Portal

中央大學第四代的 Portal 支援 OAuth2 的協定，這個是一個 demo
程式展示如何很簡單的在 Spring Boot 的框架結合 Portal 的登入。

使用的方式其實很簡單，只要在專案加入 OAuth2 Connection client，
再加上簡單的設定即可。

首先，必需教職員帳號登入 portal，然後新加一個應用系統，並且選擇使用 OAuth2 認證。系統可以取得 Client ID 與
Client Secret，之後會用到。

其次，在 pom.xml 裡加入 spring security 及 oauth2-client 的支援，然後 import 即可。
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

接下來，在 application.properties 加上必要的設定，把 Client ID 及 Client Secret 填上。

*註: 第四代 portal 上線後，請把所有的 portal-preview.cc.ncu.edu.tw 改成 portal.ncu.edu.tw 即可。

```properties
spring.security.oauth2.client.registration.bael.client-name=NCU Portal OAuth2 Demo
spring.security.oauth2.client.registration.bael.client-id=CLIENT_ID
spring.security.oauth2.client.registration.bael.client-secret=CLIENT_SECRET
spring.security.oauth2.client.registration.bael.authorization-grant-type=authorization_code
spring.security.oauth2.client.registration.bael.redirect-uri=http://localhost:8080/login/oauth2/code/client
spring.security.oauth2.client.registration.bael.scope=identifier,chinese-name
# SCOPES:identifier,chinese-nam,aculty-records,english-name,gender,birthday,personal-id,student-id,academy-records,email,mobile-phone

spring.security.oauth2.client.provider.bael.token-uri=https://portal-preview.cc.ncu.edu.tw/oauth2/token
spring.security.oauth2.client.provider.bael.authorization-uri=https://portal-preview.cc.ncu.edu.tw/oauth2/authorization
spring.security.oauth2.client.provider.bael.user-info-uri=https://portal-preview.cc.ncu.edu.tw/apis/oauth/v1/info
spring.security.oauth2.client.provider.bael.user-name-attribute=identifier
```

接下來呢? ... 接下來幾乎就完成了：設定好 web security 那些 Route 不需要認證，其餘使用 oauth2Login：

```java
package ncu.cc.oauthclient.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers("/", "/about", "/css/**", "/webjars/**").permitAll()
                .anyRequest().authenticated()
                .and().oauth2Login();
    }
}

```

真得，這樣就寫完了，那如何得到登入者及授權的資料:

```java
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

if (authentication instanceof OAuth2AuthenticationToken) {
    final var token = (OAuth2AuthenticationToken) authentication;

    token.getPrincipal().getAttributes().forEach((k, v) -> {
        System.out.printf("%s : %s\n", k, v);
    });
}
```

對了，Portal 使用者其實可以不想該應用系統得到帳號資訊，這適合於無記名投票，或無記名的意見反應系統。如果系統這樣設計的話，
把 application.properties 的一個參數修改即可。

```properties
spring.security.oauth2.client.provider.bael.user-name-attribute=id
```

那這個 id 是什麼? 這是一個 64 位元的長整數，是使用者的代碼。每次同一個使用者登入到同一個應用系統時，這個代碼是一樣的。
但是同一個使用者，在 A 應用系統與 B 應用系統是不一樣的。

如果 user-name-attribute 設為 identifier, 會得到使用者的帳號，但使用者可以拒絕授權帳號，這樣情形會產生登入失敗。

不建議設定 user-name-attribute 為 id 及 identifier 以外的值。

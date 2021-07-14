---
layout: post
title: "Spring OAuth wiht Stocktwits"
date: 2021-07-13 21:20:15 -0700
categories: guide
tags: spinrg security oauth stocktwits
---

The other day, I was working on a project that consumes
[StockTwits API](https://api.stocktwits.com/developers/docs), so I knew that I have to handle OAuth
right away. Since I've never done OAuth integration in spring before, I did what every good engineer
would do, I googled how to handle OAuth in spring. Boom, right away, I found this official tutorial
[Spring Boot and OAuth2](https://spring.io/guides/tutorials/spring-boot-oauth2/)

After quickly skimmed through the tutorial, I found that the security library does all the
heavy-lifting stuff for me. I just needed to provide the OAuth client configuration (secrets,
keys...etc). I told myself that this should not take more than 5 minutes, I should be able to move
to the next tasks easily.

So I made the following change in the `application.yaml`

```yml
spring:
  security:
    oauth2:
      client:
        registration:
          stocktwits:
            client-id: <app-client-id>
            client-secret: <app-client-secret>
```

Oh Holy-Java god, was I wrong!!

I clicked on the sign-in button on the tutorial `index.html` page. I was successfully brought to the
StockTwits app authorization consent page. I saw the redirect url contains `code` param. Everything
seemed just working smoothly like butter. Suddenly, I saw a lot of error messages started piling up
in the dev server terminal console, and I saw error in the browser as well.

It started with complain a few missing configuration items, such as: `*-uri` fields are missing;
provider cannot be found...etc. Then I realized that there were more work to be done. After series
of debug efforts and googling. I was finally content with my configuration. Everything need to be in
the client config should already be in there.

```yml
spring:
  security:
    oauth2:
      client:
        registration:
          # see: https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/security/oauth2/client/OAuth2ClientProperties.Registration.html
          stocktwits:
            client-id: <insert-your-app-client-id>
            client-secret: <insert-your-app-client-secret>
            client-authentication-method: post
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope:
              - read
              - watch_lists
              - follow_stocks
              - follow_users
        provider:
          # see: https://docs.spring.io/spring-boot/docs/current/api/org/springframework/boot/autoconfigure/security/oauth2/client/OAuth2ClientProperties.Provider.html
          stocktwits:
            token-uri: https://api.stocktwits.com/api/2/oauth/token
            authorization-uri: https://api.stocktwits.com/api/2/oauth/authorize
            user-name-attribute: user
            user-info-uri: https://api.stocktwits.com/api/2/account/verify.json
```

Alright, nothing should stop us from getting _accessToken_ to retrieve information from the
StockTwits Api.

![error right away](/assets/img/error_right_away.jpg)

After calming my inner screams, I decided to look into error messages....as I should. Between the
log lines, I finally found that the token I need in the spring security framework couldn't be
constructed properly. Turns out, the response from stocktwits API is slightly different from what
the framework expects. Luckily, this tutorial
[Spring Security Custom OAuth Requests](https://www.baeldung.com/spring-security-custom-oauth-requests)
pointed me to the right direction. I was able to add the missing properties into the token via a
custom token converter.

**Customization in
[StockTwitsAccessTokenConverter.java](https://github.com/enid0227/spring-boot-stocktwits-starter/blob/main/src/main/java/com/example/stocktwitsdemo/security/StockTwitsAccessTokenConverter.java)**

```java
public class StockTwitsAccessTokenConverter
    implements Converter<Map<String, String>, OAuth2AccessTokenResponse> {
  private static final FluentLogger logger = FluentLogger.forEnclosingClass();

  private static final Set<String> TOKEN_RESPONSE_PARAMETER_NAMES =
      ImmutableSet.of(
          OAuth2ParameterNames.ACCESS_TOKEN,
          OAuth2ParameterNames.TOKEN_TYPE,
          OAuth2ParameterNames.EXPIRES_IN,
          OAuth2ParameterNames.REFRESH_TOKEN,
          OAuth2ParameterNames.SCOPE);

  @Override
  public OAuth2AccessTokenResponse convert(Map<String, String> tokenResponseParameters) {
    logger.atInfo().log("entering convert method");
    String accessToken = tokenResponseParameters.get(OAuth2ParameterNames.ACCESS_TOKEN);
    String refreshToken = tokenResponseParameters.get(OAuth2ParameterNames.REFRESH_TOKEN);

    TOKEN_RESPONSE_PARAMETER_NAMES.forEach(
        k -> {
          logger.atInfo().log("token[%s] = %s", k, tokenResponseParameters.get(k));
        });

    return OAuth2AccessTokenResponse.withToken(accessToken)
        .tokenType(TokenType.BEARER)
        .expiresIn(0L)
        .scopes(ImmutableSet.of("read"))
        .refreshToken(refreshToken)
        .additionalParameters(ImmutableMap.of("username", "username"))
        .build();
  }
}
```

**Customization in
[WebSecurityConfiguration.java](https://github.com/enid0227/spring-boot-stocktwits-starter/blob/main/src/main/java/com/example/stocktwitsdemo/security/WebSecurityConfiguration.java#L46-L62)**

```java
  private OAuth2AccessTokenResponseClient<OAuth2AuthorizationCodeGrantRequest>
      stockTwitsTokenResponseClient() {
    logger.atInfo().log("entering security converter dispatch");

    OAuth2AccessTokenResponseHttpMessageConverter tokenCoverter =
        new OAuth2AccessTokenResponseHttpMessageConverter();
    tokenCoverter.setTokenResponseConverter(new StockTwitsAccessTokenConverter());
    // No public API to set converter directly. Therfore, set RestOperations instead
    // see default implementation at:
    // https://github.com/spring-projects/spring-security/blob/main/oauth2/oauth2-client/src/main/java/org/springframework/security/oauth2/client/endpoint/DefaultAuthorizationCodeTokenResponseClient.java
    DefaultAuthorizationCodeTokenResponseClient accessTokenResponseClient =
        new DefaultAuthorizationCodeTokenResponseClient();
    RestTemplate restTemplate =
        new RestTemplate(Arrays.asList(new FormHttpMessageConverter(), tokenCoverter));
    restTemplate.setErrorHandler(new OAuth2ErrorResponseErrorHandler());
    accessTokenResponseClient.setRestOperations(restTemplate);
    return accessTokenResponseClient;
  }
```

Finally, I was able to move further to see accessToken in the log. Now, I follow the other guide to
creating a user entry when the user login for the first time. (see
[Spring Boot OAuth2 Social Login...](https://www.callicoder.com/spring-boot-security-oauth2-social-login-part-1/))

Sure, the user should be created in the database at first login....Well, it didn't. With the
experience in the previous section, I was able to figure out that the response from the `user-info`
api is also slightly different than the expected one. Because the real user attributes are buried in
inside the extra `user` attribute at the JSON response root level.

**Example Response**

from [api/2/account/verify.json](https://api.stocktwits.com/developers/docs/api#account-verify-docs)

```json
{
  "response": { "status": 200 },
  "user": {
    "id": 176389,
    "username": "jimmychanos",
    "name": "Jim Chanos",
    "avatar_url": "http://avatars.stocktwits.com/images/default_avatar_thumb.jpg",
    "avatar_url_ssl": "https://s3.amazonaws.com/st-avatars/images/default_avatar_thumb.jpg",
    "identity": "User",
    "classification": []
  }
}
```

Therefore, I need to make sure the `OAuthUser` implementation handles that too.

[StockTwitsOAuth2User.java](https://github.com/enid0227/spring-boot-stocktwits-starter/blob/main/src/main/java/com/example/stocktwitsdemo/security/StockTwitsOAuth2User.java)

```java
public class StockTwitsOAuth2User implements OAuth2User {

  private OAuth2User user;
  private Map<String, Object> userAttributes;

  public StockTwitsOAuth2User(OAuth2User user) {
    this.user = user;
    this.userAttributes = user.getAttribute("user");
  }

  @Override
  public Map<String, Object> getAttributes() {
    return userAttributes;
  }

  @Override
  public Collection<? extends GrantedAuthority> getAuthorities() {
    return user.getAuthorities();
  }

  @Override
  public String getName() {
    return (String) userAttributes.get("username");
  }

  public String getUsername() {
    return (String) userAttributes.get("username");
  }

  public String getDisplayName() {
    return (String) userAttributes.get("name");
  }

  public Long getId() {
    return Long.valueOf((Integer) userAttributes.get("id"));
  }

  public String getEmail() {
    return (String) userAttributes.get("email");
  }
}
```

Finally, I was able to login using my own StockTwits credentials and seeing my username being
display on the homepage in tears (debugging dry eyes...). I hope this article helps anyone coming
across similar situations. Or I hope it at least provides some comfort that you're not alone.

## Source Code in This Blog

For the complete source code and the usage guide mentioned in this blog, please see
[enid0227/spring-boot-stocktwits-starter](https://github.com/enid0227/spring-boot-stocktwits-starter)

## Related Links

- [Spring Boot and OAuth2](https://spring.io/guides/tutorials/spring-boot-oauth2/)
- [Spring Security Custom OAuth Requests](https://www.baeldung.com/spring-security-custom-oauth-requests)
- [Spring Boot OAuth2 Social Login Part 1](https://www.callicoder.com/spring-boot-security-oauth2-social-login-part-1/)

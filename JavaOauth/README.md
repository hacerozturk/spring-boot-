This is aimed at implementing the capability for authenticating an end-user against a standard OAuth 2.0 Provider or an OpenID Connect 1.0 Provider. The feature essentially implements the use-case for, “Login with Google” or “Login with Facebook” or any other similar auth service.

First, we go to the OAuth2 Authentication service provider’s developer website (e.g. Google or Twitter or Facebook etc. Dev APIs console), register and obtain a new OAuth2 client credential, which we will use in our app.

Next, we create a Spring Web MVC project with Spring Boot 2.0 or higher (e.g. using Spring Initialzr or Spring Boot CLI or an IDE with Spring support etc.)
In the pom.xml file, include the following:
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.obinna.spring.demos</groupId>
  <artifactId>springsecurity-demo</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>jar</packaging>

  <name>springsecurity-demo</name>
  <description>Demo project for Spring Boot</description>

  <parent>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-parent</artifactId>
     <version>2.0.3.RELEASE</version>
     <relativePath/> <!-- lookup parent from repository -->
  </parent>

  <properties>
     <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
     <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
     <java.version>1.8</java.version>
  </properties>

  <dependencies>
     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
     </dependency>
     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-thymeleaf</artifactId>
     </dependency>
     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
     </dependency>
     <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-client</artifactId>
     </dependency>
     <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-oauth2-jose</artifactId>
     </dependency>
     <dependency>
        <groupId>org.thymeleaf.extras</groupId>
        <artifactId>thymeleaf-extras-springsecurity4</artifactId>
     </dependency>
     <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webflux</artifactId>
     </dependency>
     <dependency>
        <groupId>io.projectreactor.ipc</groupId>
        <artifactId>reactor-netty</artifactId>
     </dependency>

     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-devtools</artifactId>
        <scope>runtime</scope>
     </dependency>
     <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
     </dependency>
     <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
     </dependency>
  </dependencies>

  <build>
     <plugins>
        <plugin>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
     </plugins>
  </build>

</project>

In the src\main\resources\application.yml or properties file, enter the following minimal settings:
spring:
 thymeleaf:
   cache: false
 security:
   oauth2:
     client:
       registration:
         google:
           client-id: [enter-your-client-id-here]
           client-secret: [enter-your-client-secret-here]
#            client-authentication-method: basic
#            authorization-grant-type: authorization_code
#            redirect-uri-template: "{baseUrl}/login/oauth2/code/{registrationId}"
           scope: profile, email, address, phone
           client-name: Google Account
#        provider:
#          google:
#            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
#            token-uri: https://www.googleapis.com/oauth2/v4/token
#            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
#            user-name-attribute: sub
#            jwk-set-uri: https://www.googleapis.com/oauth2/v3/certs

In the src\main\java\app-package-name package, create a new package named, controller; and add a new Controller class (named e.g. MainController.java) and in it code the following:
@Controller
public class MainController {
   @Autowired
   private OAuth2AuthorizedClientService authorizedClientService;

   @GetMapping("/")
   public String index(Model model, OAuth2AuthenticationToken authentication) {
       OAuth2AuthorizedClient authorizedClient =
               this.authorizedClientService.loadAuthorizedClient(
                       authentication.getAuthorizedClientRegistrationId(),
                       authentication.getName());
       model.addAttribute("userName", authentication.getName());
       model.addAttribute("clientName", authorizedClient.getClientRegistration().getClientName());
       return "index";
   }

   @GetMapping("/userinfo")
   public String userinfo(Model model, OAuth2AuthenticationToken authentication) {
       OAuth2AuthorizedClient authorizedClient =
               this.authorizedClientService.loadAuthorizedClient(
                       authentication.getAuthorizedClientRegistrationId(),
                       authentication.getName());
       Map userAttributes = Collections.emptyMap();
       String userInfoEndpointUri = authorizedClient.getClientRegistration()
               .getProviderDetails().getUserInfoEndpoint().getUri();
       if (!StringUtils.isEmpty(userInfoEndpointUri)) {   // userInfoEndpointUri is optional for OIDC Clients
           userAttributes = WebClient.builder()
                   .filter(oauth2Credentials(authorizedClient))
                   .build()
                   .get()
                   .uri(userInfoEndpointUri)
                   .retrieve()
                   .bodyToMono(Map.class)
                   .block();
       }
       model.addAttribute("userAttributes", userAttributes);
       return "userinfo";
   }

   private ExchangeFilterFunction oauth2Credentials(OAuth2AuthorizedClient authorizedClient) {
       return ExchangeFilterFunction.ofRequestProcessor(
               clientRequest -> {
                   ClientRequest authorizedRequest = ClientRequest.from(clientRequest)
                           .header(HttpHeaders.AUTHORIZATION, "Bearer " + authorizedClient.getAccessToken().getTokenValue())
                           .build();
                   return Mono.just(authorizedRequest);
               });
   }
}

In the src\main\resources\templates folder, add the following thymeleaf pages: 

index.html:

<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity4">
<head>
   <title>Spring Security - OAuth 2.0 Login</title>
   <meta charset="utf-8" />
</head>
<body>
<div style="float: right" th:fragment="logout" sec:authorize="isAuthenticated()">
   <div style="float:left">
       <span style="font-weight:bold">User: </span><span sec:authentication="name"></span>
   </div>
   <div style="float:none">&nbsp;</div>
   <div style="float:right">
       <form action="#" th:action="@{/logout}" method="post">
           <input type="submit" value="Logout" />
       </form>
   </div>
</div>
<h1>OAuth 2.0 Login with Spring Security</h1>
<div>
   You are successfully logged in <span style="font-weight:bold" th:text="${userName}"></span>
   via the OAuth 2.0 Client <span style="font-weight:bold" th:text="${clientName}"></span>
</div>
<div>&nbsp;</div>
<div>
   <a href="/userinfo" th:href="@{/userinfo}">Display User Info</a>
</div>
</body>
</html>

userinfo.html:

<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml" xmlns:th="http://www.thymeleaf.org" xmlns:sec="http://www.thymeleaf.org/thymeleaf-extras-springsecurity4">
<head>
   <title>Spring Security - OAuth 2.0 User Info</title>
   <meta charset="utf-8" />
</head>
<body>
<div th:replace="index::logout"></div>
<h1>OAuth 2.0 User Info</h1>
<div>
   <span style="font-weight:bold">User Attributes:</span>
   <ul>
       <li th:each="userAttribute : ${userAttributes}">
           <span style="font-weight:bold" th:text="${userAttribute.key}"></span>: <span th:text="${userAttribute.value}"></span>
       </li>
   </ul>
</div>
</body>
</html>

Start the application and go to http://localhost:8080/. Click on the link titled Google Account and it will redirect to the Google Account login page. After authentication, the app will be redirected to the http://localhost:8080/index page. Also, on clicking the User Info link on the index page, the app will display the Google user account details for the authenticated principal.
Et viola! OAuth2 in action!!!
Note: Further steps that can be done are: Setting the app to use a custom landing-page, a custom redirect page etc, by simply overriding the defaults. For this and much more, see the Spring Security Reference Guide, online.
Implementing the above OAuth2 configuration but in a programmatic way (say, for a classic Springframework app that is not using the autoconfig-steroids from Spring Boot). To accomplish this, we do the following - see https://docs.spring.io/spring-security/site/docs/current/reference/html5/#jc-oauth2login:
In the src\main\java\app-package-name package, add the following custom config code in this src file named, OAuth2LoginConfig (note: this class is annotated with the @Configuration annotation, so this will override the Spring Boot defaults).

OAuth2LoginConfig.java

@Configuration
public class OAuth2LoginConfig {

   @EnableWebSecurity
   public static class OAuth2LoginSecurityConfig extends WebSecurityConfigurerAdapter {

       @Override
       protected void configure(HttpSecurity http) throws Exception {
           http
               .authorizeRequests()
               .anyRequest().authenticated()
               .and()
               .oauth2Login();
       }
   }

   @Bean
   public ClientRegistrationRepository clientRegistrationRepository() {
       return new InMemoryClientRegistrationRepository(this.googleClientRegistration());
   }

   private ClientRegistration googleClientRegistration() {
       return ClientRegistration.withRegistrationId("google")
               .clientId("enter-client-id-here")
               .clientSecret("enter-client-secret-here")
               .clientAuthenticationMethod(ClientAuthenticationMethod.BASIC)
               .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
               .redirectUriTemplate("{baseUrl}/login/oauth2/code/{registrationId}")
               .scope("profile", "email", "address", "phone")
               .authorizationUri("https://accounts.google.com/o/oauth2/v2/auth")
               .tokenUri("https://www.googleapis.com/oauth2/v4/token")
               .userInfoUri("https://www.googleapis.com/oauth2/v3/userinfo")
               .userNameAttributeName(IdTokenClaimNames.SUB)
               .jwkSetUri("https://www.googleapis.com/oauth2/v3/certs")
               .clientName("Google")
               .build();
   }
}

---
title: Spring Boot + JWT + Redis + OAuth2를 이용한 회원가입, 로그인 구현 (2) - Spring Security 및 기본 설정
layout: post
categories: coding
tags: spring
---

### 프로젝트 세팅
이 프로젝트에 사용될 것들을 gradle에 설정하고 application.yml에도 server와 db 설정을 해주었다.
mysql, postgresql은 사용해봤는데 mariadb를 사용해본 적이 없어서 이번에 사용해보기로 했다.    
jsp로 페이지를 만들거라 따로 jsp 설정이 들어갔다.    

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    compileOnly 'org.projectlombok:lombok'
    runtimeOnly 'org.mariadb.jdbc:mariadb-java-client'
    annotationProcessor 'org.projectlombok:lombok'
    developmentOnly 'org.springframework.boot:spring-boot-devtools'

    // auth
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'io.jsonwebtoken:jjwt-api:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-impl:0.11.5'
    implementation 'io.jsonwebtoken:jjwt-jackson:0.11.5'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'

    // jsp
    implementation 'jakarta.servlet:jakarta.servlet-api' //스프링부트 3.0 이상
    implementation 'jakarta.servlet.jsp.jstl:jakarta.servlet.jsp.jstl-api' //스프링부트 3.0 이상
    implementation 'org.glassfish.web:jakarta.servlet.jsp.jstl' //스프링부트 3.0 이상
    implementation "org.apache.tomcat.embed:tomcat-embed-jasper"
    implementation "org.springframework.security:spring-security-taglibs"

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.springframework.security:spring-security-test'
}
```

```yaml
server:
  servlet:
    context-path: /
    encoding:
      charset: utf-8
      enabled: true
    jsp:
      init-parameters:
        development: true

spring:
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/~
    username: root
    password: ~

  mvc:
    view:
      prefix: /WEB-INF/views/
      suffix: .jsp

  jpa:
    hibernate:
      ddl-auto: update
      naming:
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  security:
    user:
      name: test
      password: 1111

  devtools:
    restart:
      enabled: true

  redis:
    host: localhost
    port: 6379

jwt:
  secret: ~
```

### SecurityConfig
```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {

    }
}
```
__@Confituration__: 설정파일을 만들기 위한 어노테이션. 스프링 컨테이너는 @Configuration이 붙어있는 클래스를 자동으로 빈으로 등록해두고, 해당 클래스를 파싱해서 @Bean이 있는 메소드를 찾아서 빈을 생성해준다.    
__@EnablWebSecurity__: 이게 있어야 security가 활성화된다.

auth 관련과 login은 인증이 없어도 접근이 가능해야 하므로 이것들은 허용하고 나머지는 인증 절차를 거쳐야한다는 설정을 해주겠다.    

- deprecated 이전    

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    return http
            .authorizeRequests() // 요청에 의한 보안검사 시작
            .antMatchers("/auth/**", "/login").permitAll()
            .anyRequest().authenticated();
}
```

- deprecated 이후    

```java
private static final String[] PERMIT_ALL_PATTERNS = new String[]{
        "/auth/sign-up",
        "/auth/login"
        };

@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    return httpSecurity
            .authorizeHttpRequests(requests -> requests
                    .requestMatchers(Stream
                            .of(PERMIT_ALL_PATTERNS)
                            .map(AntPathRequestMatcher::antMatcher)
                            .toArray(AntPathRequestMatcher[]::new)).permitAll()
                    .anyRequest().authenticated())
}
```    

JWT를 사용할것이므로 이와 관련된 http 설정을 해준다.    

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    return httpSecurity.csrf(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(requests -> requests
                .requestMatchers(Stream
                    .of(PERMIT_ALL_PATTERNS)
                    .map(AntPathRequestMatcher::antMatcher)
                    .toArray(AntPathRequestMatcher[]::new)).permitAll()
                .anyRequest().authenticated())
            .exceptionHandling(e -> { e.authenticationEntryPoint(authenticationEntryPoint); e.accessDeniedHandler(jwtAccessDeniedHandler);})
            .formLogin(formLogin -> formLogin.loginPage("/auth/login").defaultSuccessUrl("/"))
            .addFilterBefore(new JwtFilter(jwtProvider, redisTemplate), UsernamePasswordAuthenticationFilter.class)
            .build();

}
```    

* csrf(AbstractHttpConfigurer::disable): csrf는 html tag를 통한 공격으로 일반적으로 사용자가 요청하는 모든 것에 csrf 보호를 사용하는 것이 좋지만 
현재는 클라이언트 서비스만을 만들기때문에 비활성화 시켜야된다. API를 작성하는데 프런트가 정해져있지 않기 때문
* httpBasic(AbstractHttpConfigurer::disable): 사용자 인증방법으로 HTTP Basic Authentication을 사용하지 않을 것이다. (JWT는 Bearer 방식을 사용하기 때문)
(* HTTP Basic Authentication: HTTP user 가 username, password로 요청을 보내고 헤더의 Authorization에 Basic <credentials> 형태로 보내진다.)    
* sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS)): JWT를 사용하므로 세션을 사용하지 않아야한다. STATELESS는 인증 정보를 서버에 담아두지 않는다는 것이다.
* formLogin().disable(): 기본 로그인 페이지가 불필요하다.    
* exceptionHandling(e -> { e.authenticationEntryPoint(); e.accessDeniedHandler();}): 예외 설정
* formLogin(formLogin -> formLogin.loginPage("/auth/login").defaultSuccessUrl("/")): 기본 로그인 페이지로 jsp를 이용해 만든 페이지로 넘어가도록 설정
* addFilterBefore(): 특정 필터를 등록하는 역할. 첫 번째 인자로는 등록 필터 전달, 두 번째 인자로는 등록할 위치 전달. UsernamePasswordAuthenticationFilter는 Spring Security에서 기본적으로 제공하는 폼 인증 처리 필터. 
이를 기준으로 JwtFilter가 등록되므로 사용자 인증 전 JWT 토큰 검사하게 되는 것이다.    

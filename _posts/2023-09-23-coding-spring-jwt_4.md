---
title: Spring Boot + JWT + Redis + OAuth2를 이용한 회원가입, 로그인 구현 (4) - filter
layout: post
categories: coding
tags: spring
---

## OncePerRequestFilter
토큰을 사용하려면 토큰에 대한 검증이 필요하다. 이를 위해 인증이 필요한 요청이 거칠 필터를 만들어줘야 한다.
JwtFilter는 내가 만든 필터로 OncePerRequestFilter를 상속한다.    
이런 필터는 api가 실행될때마다 토큰인증을 해주려면 컨트롤러 메서드 첫 부분마다 인증 코드를 작성해줘야 되는 거를 해결해준다.    
스프링의 서블릿 필터는 디스패처 서블릿 실행 전에 실행되는 클래스이다.
#### [참고]
<https://velog.io/@zueon/BE-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0>


```java
@Slf4j
@RequiredArgsConstructor
public class JwtFilter extends OncePerRequestFilter {

    private final JwtProvider jwtProvider;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String[] SHOULD_NOT_FILTER_URI_LIST = new String[]{
            "/auth/sign-up",
            "/auth/login"
    };

    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) throws ServletException {
        return Stream.of(SHOULD_NOT_FILTER_URI_LIST).anyMatch(request.getRequestURI()::startsWith);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        String token = resolveToken(request);

        if (StringUtils.hasText(token)) {
            JwtCode jwtCode = jwtProvider.validateToken(token);
            if (jwtCode.equals(JwtCode.ACCESS) && SecurityContextHolder.getContext().getAuthentication() == null) {
                Authentication authentication = jwtProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } else if (jwtCode.equals(JwtCode.EXPIRED)) {
                ...
            }
        }

    }


    private String resolveToken(HttpServletRequest request) {
        return request.getHeader(HttpHeaders.AUTHORIZATION);
    }
}
```    

이런 필터는 보통 GenericFilterBean을 상속받거나 내가 한 것 처럼 **OncePerRequestFilter** 를 상속받는다.    
인증, 접근 제어는 RequestDispatcher 클래스에 의해 다른 서블릿으로 dispatch된다. 
이때, 이동할 서블릿에 도착하기 전에 다시 한 번 filter chain을 거치며 필터가 두 번 실행되는 현상이 발생할 수 있다. 
같은 클라이언트 안에서는 같은 서블릿이 서빙되어야 이상적이나 다른 서블릿 객체가 생성되는 경우가 있는데 이 때 GenericFilterBean은 
새 필터 로직을 수행하므로 OncePerRequestFilter를 이용해서 동일 클라이언트는 동일 서블릿을 서빙할 수 있도록 구현하는 게 이상적이라고 한다.    

#### [참고]
<https://sunghs.tistory.com/151>

<hr>

위에서 작성한 코드를 보자면 모든 요청이 들어오면 doFilterInternal을 거친다. 그러나 우리는 인증이 필요없는 작업도 있을거고, 
인증이 없어야 되는 작업도 있을거다(회원가입, 로그인). 그런 작업은 securityConfig에서 이미 permitAll 작업을 해줬는데 
분명 예전에는 그걸로 됐던 것 같은데 왜인지 이번에는 작동이 되지 않아 찾아보던 중 OncePerReuqestFilter에서 shouldNotFilter 설정을 
해줘야된다는 걸 알았다.     

예전 프로젝트는 스프링 부트 3.0.5 버전이고 이거는 3.1버전이어서 그 사이에 spring security에서 뭐가 달라진건지 httpSecurity 설정도 원래는 deprecated 됐다는 얘기도 안 나왔오고 예전 방식을 
잘 썼고 저것도 shouldNot이런 거 안 했었는데 갑자기 안 된다고 떠서 당황스러웠다.    

#### doFilterInternal 순서
1. 토큰이 있다면 토큰의 유효성 겁사
2. 유효한 토큰인데 SecurityContextHolder에 인증객체가 저장되어 있지 않다면 설정
3. 만료된 토큰이라면 리프레시 토큰 검사를 통해 새 토큰 발급 or 리프레시 토큰 만료로 로그인 화면 이동
4. 필터 진행


위의 필터를 거치기 위해서는 SecurityConfig에서 수정이 필요하다.    

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
    return httpSecurity
            .csrf(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(requests -> requests
                    .requestMatchers(
                        Arrays.stream(PERMIT_ALL_PATTERNS).map(AntPathRequestMatcher::antMatcher).toArray(AntPathRequestMatcher[]::new)
                    ).permitAll()
                    .anyRequest().authenticated()
            )
            .exceptionHandling(e -> { e.authenticationEntryPoint(authenticationEntryPoint); e.accessDeniedHandler(jwtAccessDeniedHandler);})
            .addFilterBefore(new JwtFilter(jwtProvider), UsernamePasswordAuthenticationFilter.class)
            .build();
}
```    

* addFilterBefore(): 특정 필터를 등록하는 역할. 첫 번째 인자로는 등록 필터 전달, 두 번째 인자로는 등록할 위치 전달. UsernamePasswordAuthenticationFilter는 Spring Security에서 기본적으로 제공하는 폼 인증 처리 필터. 
이를 기준으로 JwtFilter가 등록되므로 사용자 인증 전 JWT 토큰 검사하게 되는 것이다.    
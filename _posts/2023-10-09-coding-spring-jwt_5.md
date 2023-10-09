---
title: Spring Boot + JWT + Redis + OAuth2를 이용한 회원가입, 로그인 구현 (5) - 토큰 만료, 로그아웃
layout: post
categories: coding
tags: spring
---
JWT를 사용한다면 토큰 만료에 대해 어떻게 다룰 것인지 생각해봐야한다.    
예전 프로젝트에서는 토큰 만료일 경우 따로 백에서 처리를 해주는게 아니고, 토큰이 만료되기 직전에 프론트에서 /reissue 요청을 발생시키고 refreshToken의 유효성을 검사한 후 재발급해주는 형식으로 진행됐다.    

여기서 내가 생각이 많아진 이유는 한 가지 방식이 정해진게 아니고 누구는 프론트에서 reissue를 발생시켜 토큰이 만료되기 전에 새 토큰을 발생시키고, 어디에서는 프론트가 아닌 백에서 토큰 만료를 체크해 새 토큰을 발생시켜 준다는 점에서 고민이 많았다.    

그러나 내가 작업할때는 프론트가 따로 없고 프론트에서 신경쓸 필요가 없다고 느껴져 백에서 만료를 체크해 새로 발급해주는 방식을 선택했다.    

### JwtFilter
요청이 들어오면 검사하는 필터에서 토큰의 만료를 확인해주면 된다.    

내가 생각한 흐름은
1. AccessToken 유효 && 사용 불가능한 토큰으로 설정되어 있지 않음    
    1. 인증 객체로 설정    
    2. 필터 진행
2. AccessToken 만료    
    1. RefreshToken 검사    
        1. RefreshToken 만료    
            1. CustomException 던짐    
                1. JwtFilter에서 catch해서 에러 내용 보내줌    
        2. RefreshToken 유효    
            1. RefreshToken으로 인증 객체 가져옴    
            2. 새 AccessToken 발급    
            3. 인증 객체 설정 후 response의 헤더에 발급한 AccessToken 설정    
            4. 필터 진행    
3. AccessToken 문제 발생    
    1. 문제가 발생한 토큰 사용하지 못하도록 설정 후 exception 던짐    

```java
@Override
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
    log.info("[doFilterInternal]: " + request.getRequestURI());
    String token = resolveToken(request);
    Authentication authentication = null;
    try {
        if (StringUtils.hasText(token)) {
            // accessToken의 유효기간 검사
            JwtCode jwtCode = jwtProvider.validateToken(token);
            // 토큰의 유효기간이 남아있고 사용가능한 토큰
            if (jwtCode.equals(JwtCode.ACCESS) && !jwtProvider.isLogout(token)) {
                authentication = jwtProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            } else if (jwtCode.equals(JwtCode.EXPIRED)) {
                // 만료된 accessToken이라면 refreshToken 검사 후 재발급
                String refreshToken = jwtProvider.getRefreshToken(request.getHeader("Auth"));
                if (StringUtils.hasText(refreshToken)) {
                    authentication = jwtProvider.getAuthentication(refreshToken);
                    String accessToken = jwtProvider.createAccessToken(authentication);
                    SecurityContextHolder.getContext().setAuthentication(authentication);
                    response.setHeader(HttpHeaders.AUTHORIZATION, accessToken);
                    log.info("새 accessToken 발급");
                }
            } else {
                // accessToken 검사에서 JwtCode가 DENIED가 나올 떄
                log.info("jwtFilter 기타 에러 발생");
                // 문제가 생긴 accessToken은 더이상 사용 불가능하도록
                jwtProvider.setLogout(token);
                throw new CustomException(ErrorCode.INTERNAL_SERVER_ERROR, "jwt 에러로 재로그인 필요");
            }
        }
        filterChain.doFilter(request, response);
    } catch (CustomException c) {
        response.setStatus(c.getErrorCode().getStatus());
        response.setContentType(String.valueOf(MediaType.APPLICATION_JSON));
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(c.getMessage());
    } catch (ExpiredJwtException e) {
        log.info("refresh token 만료, 재로그인 필요");
        jwtProvider.setLogout(token);
        throw new CustomException(ErrorCode.INTERNAL_SERVER_ERROR, "jwt 에러로 재로그인 필요");
    }
}
```

여기서 고민한 것 중 하나는 refreshToken을 가져오기위해서는 redis에 접근해야 하는데 프론트에서 요청해서 토큰 재발급을 하는 게 아니기때문에 refreshToken을 가져오기위한 userName을 알기가 힘들다는 것이었다.    
accessToken으로 가져오려 해도 이미 만료됐으므로 에러가 발생할 수 밖에 없고, 결국 header에 userName을 항상 보내줘야 된다는 단점이 있다.    
그래서 프론트에서 reissue와 같은 요청으로 보내주는 건가?    

<img src="https://github.com/team-web-development-projects/developer-talks-backend/assets/68698007/786dfc6f-e9dc-4e12-9127-d832aac8bfbd">


### JwtProvider
RefreshToken에 대한 코드와 토큰 사용 불가를 위한 코드가 추가되었다.    

```java
public JwtCode validateToken(String token) {
    try {
        Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token);
        return JwtCode.ACCESS;
    } catch (ExpiredJwtException e) {
        return JwtCode.EXPIRED;
    } catch (Exception e) {
        return JwtCode.DENIED;
    }
}

public Authentication getAuthentication(String token) {
    Claims claims = Jwts.parserBuilder().setSigningKey(key).build()
            .parseClaimsJws(token)
            .getBody();
    Collection<? extends GrantedAuthority> authorities =
            Arrays.stream(claims.get("AUTHORITIES").toString().split(","))
                    .map(SimpleGrantedAuthority::new)
                    .toList();
    UserDetails principal = new User(claims.getSubject(), "", authorities);
    return new UsernamePasswordAuthenticationToken(principal, "", authorities);
}

public String getRefreshToken(String userName) {
    return refreshTokenRepository.findByUserName(userName).orElseThrow(() ->
            new CustomException(ErrorCode.NEED_LOGIN, "refresh token 만료, 재로그인 필요")).getRefreshToken();
}

public Long getAccessTokenExpiration(String accessToken) {
    try {
        Claims claims = Jwts.parserBuilder().setSigningKey(key).build()
                .parseClaimsJws(accessToken)
                .getBody();
        return claims.getExpiration().getTime() - System.currentTimeMillis();
    } catch (JwtException e) {
        throw new CustomException(ErrorCode.INTERNAL_SERVER_ERROR, "Token expiration error");
    }
}

public boolean isLogout(String accessToken) {
    if (accessTokenRepository.findByAccessToken(accessToken).isPresent()) {
        throw new CustomException(ErrorCode.NEED_LOGIN, "로그인이 필요합니다.");
    }
    return false;
}

public void setLogout(String accessToken) {
    Long accessTokenExpiration = getAccessTokenExpiration(accessToken);
    accessTokenRepository.save(AccessTokenDto.builder()
            .accessToken(accessToken)
            .value("logout")
            .expiration(accessTokenExpiration * 60 * 60)
            .build());
}
```

logout이라는 설정은 사용자가 문제가 생긴 accessToken에 접근하려고 할때나 해당 토큰에 다른 사용자가 접근하려고 할 떄 logout 설정된 토큰이면 문제가 발생하도록 하기 위해 만들어진것이다.    

#### AccessTokenDto    
```java
@Getter
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@RedisHash(value = "access")
public class AccessTokenDto {
    @Id
    private String id;

    @Indexed
    String accessToken;

    String value;

    @TimeToLive
    private Long expiration;
}
```    

redis에 setLogout이 불려지면 해당 토큰의 값이 logout으로 저장되고 만약 여기에 저장된 토큰으로 접근한다면 에러가 발생되도록 처리했다.    

<br>

이러면 이제 로그인을 했을 때 redis에 refreshToken이 저장되어 있는 걸 확인할 수 있고 그 값을 jwtProvider에서 가져와 확인하고 진행한다.    

<hr>

### 로그아웃

```java
@PutMapping("/logout")
public ResponseEntity<Void> logout(HttpServletRequest request) {
    authService.logout(request.getHeader(HttpHeaders.AUTHORIZATION));
    return ResponseEntity.ok().build();
}
```    
위에 설정한 setLogout으로 로그아웃도 만들 수 있다.    

```java
@Override
public void logout(String accessToken) {
    if (!jwtProvider.validateToken(accessToken).equals(JwtCode.ACCESS)) {
        throw new CustomException(ErrorCode.INTERNAL_SERVER_ERROR, "유효하지 않은 토큰");
    }

    Authentication authentication = jwtProvider.getAuthentication(accessToken);
    String userName = authentication.getName();
    Optional<RefreshTokenDto> optionalRefreshToken = refreshTokenRepository.findByUserName(userName);
    optionalRefreshToken.ifPresent(refreshTokenRepository::delete);

    jwtProvider.setLogout(accessToken);
}
```
refreshToken을 사용 불가능하게 설정하고 accessToken또한 삭제하면 된다.    

<img src="https://github.com/team-web-development-projects/developer-talks-backend/assets/68698007/3465442e-1778-4a86-87e2-f6eb4552ee71">
로그아웃하면 위와 같이 redis에 accessToken이 생기는 걸 확인할 수 있다.    

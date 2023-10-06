---
title: Spring Boot + JWT + Redis + OAuth2를 이용한 회원가입, 로그인 구현 (2) - 회원가입, 로그인
layout: post
categories: coding
tags: spring
---

인스타그램 클론코딩 생각하면서 한거라 User는 userName으로 로그인할거고 이메일, 프로필 이미지, 소개글, 비공개 여부를 가질거다.    


### User

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class User implements UserDetails {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    private String userName;

    @NotNull
    private String email;

    @NotNull
    private String password;

    private String profileImgUrl;

    @Column(columnDefinition = "TEXT", length = 100)
    private String description;

    @NotNull
    @ColumnDefault("false")
    private boolean nonPublic;

    @ElementCollection(fetch = FetchType.LAZY)
    private List<String> roles;

    @Builder
    public User(String userName, String email, String password, String profileImgUrl, String description, boolean nonPublic) {
        this.userName = userName;
        this.email = email;
        this.password = password;
        this.profileImgUrl = profileImgUrl;
        this.description = "";
        this.nonPublic = false;
        this.roles = Collections.singletonList("ROLE_USER");
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.roles.stream().map(SimpleGrantedAuthority::new).collect(Collectors.toList());
    }

    @Override
    public String getUsername() {
        return userName;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
```

그렇게 나온 게 위의 User 엔티티.    
- implements UserDetails: 인증 절차에서 authenticationManager의 authenticate를 통해 UserDetailService의 loadBy~를 거칠 때 
 userName을 통해 로그인하므로 loadUserByUsername이 필요하다. 이걸 위해 CustomUserDetailService 클래스를 작성하고 userRepository에서 userName으로 사용자를 
찾는데, 이때 반환해야 되는 값이 User가 아닌 UserDetails이어야하므로 User가 UserDetails를 구현하도록 하는 것이다.    

  
### 회원가입

```java
@Override
@Transactional
public void signUp(SignUpDto dto) {
    if (userRepository.existsByUserName(dto.getUserName())) {
        throw new CustomException(ErrorCode.ACCEPTABLE_BUT_EXISTS, "이미 존재하는 아이디입니다.");
    }
    User user = User.builder()
        .userName(dto.getUserName())
        .email(dto.getEmail())
        .password(passwordEncoder.encode(dto.getPassword()))
        .build();
    userRepository.save(user);
}
```    

<hr>

### 토큰 생성과 로그인
로그인을 하면 사용자의 정보를 가지고 있는 토큰 정보를 생성하고 저장해야 한다.
리프레시 토큰을 redis에 저장하기 위해 RedisConfig를 작성해 빈 등록을 하고 RedisTemplate과 repositorty중 repositorty 사용을 하기로 해서 RefreshTokenRepository도 등록해준다.    

#### RedisConfig

```java
@Configuration
@EnableRedisRepositories
@RequiredArgsConstructor
public class RedisConfig {
    private final RedisProperties redisProperties;
    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory(redisProperties.getHost(), redisProperties.getPort());
    }
}
```    

#### RefreshTokenDto

userName으로 리프레시 토큰을 찾도록 할거라 userName에 @Indexed를 붙여줬다.    

```java
@Getter
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
@RedisHash(value = "refresh", timeToLive = 60 * 2)
public class RefreshTokenDto {
    @Id
    private String id;

    private String refreshToken;

    private long refreshTokenExpiration;

    @Indexed
    private String userName;
}
```    

#### RefreshTokenRepository

```java
@Repository
public interface RefreshTokenRepository extends CrudRepository<RefreshTokenDto, Long> {
    Optional<RefreshTokenDto> findByUserName(String userName);
}
```

이렇게 리프레시 토큰 준비는 끝났고 로그인을 하면 accessToken과 refreshToken 정보를 반환해야된다.    


#### TokenDto

```java
@Getter
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Builder
public class TokenDto {
    private String accessToken;
    private RefreshTokenDto refreshTokenDto;
}
```    


#### JwtProvider.class    

토큰 생성과 검사를 위한 클래스    

```java
@Component
@RequiredArgsConstructor
public class JwtProvider {

    @Value("${jwt.secret}")
    private String secretKey;
    private Key key;

    private static final long ACCESS_TOKEN_EXPIRE_TIME = 60 * 1000L;
    private static final long REFRESH_TOKEN_EXPIRE_TIME = 2 * 60 * 1000L;   // 테스트를 위해 짧게 설정

    private final RefreshTokenRepository refreshTokenRepository;

    @PostConstruct
    protected  void init() {
        secretKey = Base64.getEncoder().encodeToString(secretKey.getBytes());
        key = Keys.hmacShaKeyFor(secretKey.getBytes());
    }

    public String createAccessToken(Authentication authentication) {
        return createToken(authentication, ACCESS_TOKEN_EXPIRE_TIME);
    }

    @Transactional
    public RefreshTokenDto createRefreshToken(Authentication authentication) {
        String refreshToken = createToken(authentication, REFRESH_TOKEN_EXPIRE_TIME);

        return refreshTokenRepository.save(RefreshTokenDto.builder()
                    .userName(authentication.getName())
                    .refreshToken(refreshToken)
                    .refreshTokenExpiration(REFRESH_TOKEN_EXPIRE_TIME)
                    .build());
    }

    private String createToken(Authentication authentication, long tokenValidTime) {
        String authorities = authentication.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority).collect(Collectors.joining(","));
        Date now = new Date();
        Date expiration = new Date(now.getTime() + tokenValidTime);

        Claims claims = Jwts.claims()
                .setSubject(authentication.getName());
        claims.put("AUTHORITIES", authorities);

        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(expiration)
                .signWith(key, SignatureAlgorithm.HS256)
                .compact();
    }
}
```    
access token과 refresh token의 유효시간은 테스트를 위해 짧게 잡았다.    
key값은 암호화한 값을 application.yml에 가지고있지 않아서 문자열의 값을 @PostConstruct로 암호화하는걸 먼저 실행하도록 했다.    

- createToken
  - 설정한 정보로 토큰을 만들어서 반환
  - Authentication을 받고 거기서 권한들을 가지고 와 Claims에 넣어줬다. 
  - Claims는 Jwt의 페이로드 부분으로 다른 포스트에서 다룬 적 있고, subject로는 authentication.getName()을 가지고 온다. usernamePassword~를 만들때 썼던걸로 나는 userName을 설정했으므로 User의 userName 값이 나온다.    
  - expiration: 토큰의 유효시간으로 현재 날짜에 내가 설정한 유효시간을 더해준다.    


- createRefreshToken
  - 리프레시 토큰은 발급된 후 설정한 시간이 지나면 만료되어야 한다. redis를 사용하면 설정한 유효시간이 지났는지 확인하기가 편하다.    


#### AuthServiceImpl.class

```java
@Override
@Transactional(readOnly = true)
public TokenDto login(LoginInfoDto dto) {
    Authentication authentication = authenticationManagerBuilder.getObject().authenticate(
            new UsernamePasswordAuthenticationToken(dto.getUserName(), dto.getPassword())
    );
    String accessToken = jwtProvider.createAccessToken(authentication);
    RefreshTokenDto refreshTokenDto = jwtProvider.createRefreshToken(authentication);

    return TokenDto.builder()
            .accessToken(accessToken)
            .refreshTokenDto(refreshTokenDto)
            .build();
}
```    

1. 사용자가 입력한 아이디(userName)와 비밀번호로 UsernamePasswordAuthenticationToken 객체 생성    
2. AuthenticationManager의 authenticate 메소드를 호출 (DaoAuthenticationProvider가 authenticate하는 것)
   * DaoAuthenticationProvder가 PasswordEncoder를 주입 받아 갖고 있으며, authenticate 메소드 내의 additionalAuthenticationChecks 메소드에서 비밀번호 확인    
3. 인증 객체로 access token과 refresh token 생성 (리프레시 생성 때 redis에 리프레시 정보도 저장됨)
4. 생성된 정보 반환

#### AuthController.class    
원래는 토큰 정보를 헤더에 담아서 보내줬었다. 그런데 여기서 jwt를 공부하고 나니 헤더에 담아서 보내줘야 되는 거는 accessToken밖에 없고 리프레시도 그런 방식으로 보내주면 
보안의 문제가 있고 쿠키에 담아서 보내는게 좋다는 걸 알게됐다.    
쿠키는 한 번도 다뤄본적이 없어서 많이 헤맸고 찾아 본 결과 ResponseCookie라는 거를 사용한다는 걸 알게됐다.    

```java
@PostMapping("/login")
public ResponseEntity<Void> login(LoginInfoDto dto) {
    TokenDto tokenDto = authService.login(dto);
    HttpHeaders httpHeaders = new HttpHeaders();
    httpHeaders.add("Authorization", tokenDto.getAccessToken());

    ResponseCookie cookie = ResponseCookie.from("refreshToken", tokenDto.getRefreshTokenDto().getRefreshToken())
            .maxAge(tokenDto.getRefreshTokenDto().getRefreshTokenExpiration())
            .path("/")
            // https 환경에서만 쿠키가 발동
            .secure(true)
            // 동일 사이트과 크로스 사이트에 모두 쿠키 전송이 가능
            .sameSite("None")
            // 브라우저에서 쿠키에 접근할 수 없도록 제한
            .httpOnly(true)
            .build();
    httpHeaders.add(HttpHeaders.SET_COOKIE, cookie.toString());
    return ResponseEntity.ok().headers(httpHeaders).build();
}
```    

<hr>
모든 작업을 마치고 나면 아래와 같은 결과를 얻는다.     


<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/22894c5e-faac-4f02-9afb-b5a55db7e658">

<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/4298f20b-44c5-423a-a6c3-bdc43b4b2637">    


<hr>

<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/7ff592ef-5484-4cce-9167-0c4b381f4ba4" width="60%">

<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/8515fed8-5547-4e56-816f-808d69faaefc">

헤더와 쿠키에도 잘 들어가 있는 걸 확인할 수 있다.    

그러나 그 전에 위와 같은 작업을 아무런 설정 없이 실행하면 401 에러가 발생한다. 이를 해결하기 위해 SecurityConfig 설정을 해줘야 된다.    
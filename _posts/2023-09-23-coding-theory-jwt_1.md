## Spring Boot + JWT + Redis + OAuth2를 이용한 회원가입, 로그인 구현 (1) - Spring Security

스프링에서는 가장 중요한 보안을 Spring Security를 이용해 처리한다.

### Spring Security
Spring 기반의 애플리케이션의 보안을 담당하는 하위 프레임워크로 '인증', '권한' 부분을 Filter 흐름에 따라 처리한다.    

<img src="https://user-images.githubusercontent.com/68698007/229714785-ac2e79f8-fab0-4fc7-8d8f-bd9b343a7969.png">

principal을 아이디로, credential을 비밀번호로 사용하는 Credential 기반의 인증 방식을 사용    

(_principal (접근 주체): 보호받는 Resource에 접근하는 대상_    
_credential (비밀번호) : Resource에 접근하는 대상의 비밀번호_)

#### [UsernamePasswordAuthenticationToken]
Authenticaion을 implements하는 클래스의 하위클래스로 User ID가 principal, password가 credential이 된다.    

#### [AuthenticationManager]
실질적 인증은 AuthenticationManager에 등록된 Authentication provider에서 처리되고 인증이 성공하면 인증이 성공한 객체를 생성하여 Security context에 저장한다.    

#### [AuthenticationProvider]
실제 인증에 대한 부분을 처리하고 인증 전의 Authentication 객체를 받아서 인증이 완료된 객체를 반환한다.    

#### [UserDetailsService]
UserDetails 객체를 반환하는 loadUserByUsername 하나만 가지고 있고 일반적으로 이를 구현한 클래스의 내부에 repository를 주입받아 DB와 연결하여 처리한다.    

#### [UserDetails]
인증에 성공하여 생성된 객체로 Authentication 객체를 구현한 UsernamePasswordToken을 생성하기 위해 사용된다.    
UserVO모델에 implements하여 처리 가능하다.    

#### [SecurityContextHolder]
보안 주체의 세부 정보를 포함하여 응용 프로그램의 현재 보안 컨텍스트에 대한 세부 정보를 저장한다.    

#### [SecurityContext]
Authentication 객체를 보관하고 꺼내올 수 있다. ThreadLocal에 저장되어 아무 곳에서나 참조가 가능하도록 설계되었고 인증이 완료되면 HttpSession에 저장되어 어플리케이션 전반에 걸쳐 전역적인 참조가 가능하다.

#### [Authentication]
현재 접근하는 주체의 정보와 권한을 담는 인터페이스이다.

### 처리 흐름
1. 사용자가 로그인 정보와 함께 인증을 요청
2. AuthenticationFilter가 UsernamePasswordAuthenticationToken의 인증용 객체를 생성한다.    
3. AuthenticationManger의 구현체인 ProviderManger에게 생성한 UsernamePasswordToken 객체를 전달한다.
4. AuthenticationProvier는 UserDetails를 넘겨받고 사용자 정보를 비교한다.
5. 실제 DB에서 사용자 인증정보를 가져오는 UserDetailsService에 사용자 정보를 넘겨준다.
6. 넘겨받은 사용자 정보를 통해 DB에서 찾은 사용자 정보인 UserDetails 객체를 만든다.
7. AuthenticaionProvider는 UserDetails를 넘겨받고 사용자 정보를 비교한다.
8. 인증이 완료되면 권한 등의 사용자 정보를 담은 Authenticaion 객체를 반환한다.
9. AuthenticationFilter에 Authentication 객체가 반환된다.
10. Authenticaion 객체를 SecurityContext에 저장한다.

<br>
Spring Security는 이전까지 WebSecurityConfigurerAdpater를 상속받아 configure 메소드를 오버라이드 하며 사용했었다.    
그렇지만 5.7로 들어가며 이를 더이상 사용하지 않기로 결정했고 SecurityFilterChain을 직접 빈으로 등록하며 체인 등록 작업을 해줘야 한다.    

```java
@Configuration
public class SecurityConfiguration {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests((authz) -> authz
                .anyRequest().authenticated()
            )
            .httpBasic(withDefaults());
        return http.build();
    }

}
```

<br>    
<hr>    

이렇게 알아보며 나는 토큰을 만드는 방식에서 한 가지 궁금한 점이 생겼다. 프로젝트를 진행할 때 사용한 방식과 위에서 내가 서술한 방식의 차이점이었다.    
- AuthService에서 UserRepository에 바로 접근해 사용자가 존재한다면 필요한 정보를 가져오고 그 값을 토큰 생성 메소드로 넘겨줘서 토큰을 만드는 방법
- AuthService에서 id,pw 기반으로 UsernamePasswordAuthenticationToken 객체 생성, 
authenticationManagerBuilder.getObject().authentication(authenticationToken) 실행으로 
직접 만든 CustomUserDetailsService의 loadBy~에서 repository에 접근해 검증을 마치고 
여기서 나온 authentication 값을 토큰 생성 메소드에 넘겨줘서 만드는 방법    

결국 토큰을 생성한다는 것은 같지만 두 번쨰처럼 복잡해 보이는 방식을 사용해야 되는 이유를 알고싶었고 커뮤니티에 질문한 결과    

첫 번째 방식은 **빠른 구현, 중복 호출 회피** 로 성능이 좋아지고    
두 번째 방식은 **보안 관련 작업을 더 쉽게 구현이 가능하고 유지보수와 로직분리** 가 된다는 장점이 있다는 답변을 받았다.    

보안 관련 작업이 뭘 얘기하는지 물어봤더니    
1. "필터"로 외부 공격을 방지 할 수 있다.
2. "세션관리" 사용자의 타임 아웃, 로그인 제한 등을 설정이 가능하다.
3. "프로바이더" 사용자의 DB, OAuth등 암호화 기능과 다른 API와의 연결이 가능하다.
4. "자원 보호" 애플리케이션 안에 URL, Method를 보호하고 권한이 없는 자는 엑세스를 거부한다.

같은 보안관련 추가적인 기능을 쉽게 활용하여 이용 할 수 있습니다.    
이라는 답변을 받았다. 사실 이해하지 못 했다ㅎㅎㅎ    

어쨌든 분리하는게 좋다는 것 같기는한데... 확실하게 이해하기에는 내 지식이 너무 부족하다    


#### 참고
<https://dev-coco.tistory.com/174>
<https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter>
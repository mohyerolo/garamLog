---
title: JWT 이론
layout: post
categories: coding
tags: theory
---
진행하고 있던 프로젝트에서 뒤늦게 jwt 관리와 사용자 정보가 변경 될 때마다 토큰을 재발급 받게 되어있는거에 대해 알게 되어 jwt를 다시 공부하기 시작했다. 리프레시 토큰을 검증하지 않는 것 같아서 그것도 신경쓰이지만 redis 사용하실 것 같아서 아마 검증도 하게 될 것 같다!    

jwt에 대해서 백엔드 코드를 공부하는 것이 아닌 이론을 다시 잡고자 해서 코드는 없다.    

#### [JWT와 AccessToken, RefreshToken 기본 이론]
<https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-JWTjson-web-token-%EB%9E%80-%F0%9F%92%AF-%EC%A0%95%EB%A6%AC>
<https://inpa.tistory.com/entry/WEB-%F0%9F%93%9A-Access-Token-Refresh-Token-%EC%9B%90%EB%A6%AC-feat-JWT>


#### [JWT 보안, 탈취]   
<https://blogeon.tistory.com/entry/JWT%EC%9D%98-Refresh-Token%EA%B3%BC-Access-Token%EC%9D%80-%EC%96%B4%EB%94%94%EC%97%90-%EC%A0%80%EC%9E%A5%ED%95%B4%EC%95%BC-%ED%95%A0%EA%B9%8C>
<https://alkhwa-113.tistory.com/entry/TIL-JWT-%EC%99%80-%EB%B3%B4%EC%95%88-CORS>
<https://dkswnkk.tistory.com/684>

위의 블로그의 글들을 읽고 정리한 내용이다.    
<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/fa72be6c-56e1-4ebe-856b-a676c0043c84
">

밑에가 잘렸는데 key를 refresh token 값이 아닌 사용자를 판별할 수 있는 정보로 두면 정상 사용자도 재발급이 가능하다는 얘기이다.    

access token에 대한 탈취 예방 방법은 refresh token과 유효기간 말고는 찾아보기가 힘든데 뺏기면 어쩔 수 없는건지..

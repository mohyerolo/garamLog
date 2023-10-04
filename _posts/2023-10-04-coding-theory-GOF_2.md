---
title: "[자바 디자인 패턴의 이해] 2강 - Adapter Pattern"
layout: post
categories: coding
tags: theory
---
## 인프런 - 자바 디자인 패턴의 이해 2강
<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/beb989c4-a37f-47c3-89f1-9772e49ea7ed">

#### 예시

```java
public class Math {
    // 두배
    public static double twoTime(double num) {
        return num * 2;
    }

    // 절반
    public static double half(double num) {
        return num / 2;
    }

    // 강화된 알고리즘
    public static Double doubled(Double d) {
        return d * 2;
    }
}
```    
double 타입으로 선언된 Math 클래스의 함수들.    
Float으로 매개변수를 받고 반환하는 요구사항이 들어옴    

```java
public class AdapterImpl implements Adapter {

    @Override
    public Float twiceOf(Float f) {
        // return (float) Math.twoTime(f.doubleValue());
        return Math.doubled(f.doubleValue()).floatValue();
    }

    @Override
    public Float halfOf(Float f) {
        System.out.println("더 기능이 필요할 떄는 구현하는 곳에서 작성");
        return (float) Math.half(f.doubleValue());;
    }
}
```

그러면 우리는 위와 같이 반환 값을 변경해주는 작업이 필요.    
만약 인자가 더 많아지고 복잡해진다면 한 줄로 끝나지 않을 수 있음.    

이미 만들어진 알고리즘을 가져다가 쓸 때 인터페이스를 통해 구현된 클래스 안에서 사용하고 수정하는게 어댑터 패턴?    


<hr>

## 따로 공부
- 어댑터 패턴: 호환되지 않는 인터페이스들을 연결하는 디자인 패턴. 기존의 클래스를 수정하지 않고도 특정 인터페이스를 필요로 하는 코드에서 사용할 수 있게 해줌. 클래스의 인터페이스를 다른 인터페이스로 변환할 수 있게 해줘서 서로 다른 인터페이스를 가진 클래스들이 상호 작용할 수 있도록 해 코드의 재사용성을 증해시킨다.    

- 주요 구성 요소    
1) 타겟    
2) 어댑티    
3) 어댑터    
4) 클라이언트    

클라이언트가 어댑티 클래스를 호출하지 않고도 자신이 원하는 인터페이스를 통해 어댑티 클래스의 기능을 이용할 수 있음.    

- 단점: 어댑터 클래스 추가 작성으로 소스 코드 증가 -> 코드의 복잡성 증가 -> 유지 보수 어려움 증가 가능성 + 데이터 변환 과정에서 추가적인 처리 시간과 오버헤드 발생 가능성


### 결론
코드 재사용이라는 장점이 있으나 오버헤드가 발생할 수 있고 추가 코드 작성도 있으므로 호환되지 않는 인터페이스를 가진 클래스들이 함께 쓰여야되거나 요구 사항이 맞지 않을 경우에만 고려. 수정 가능한 소스 코드일 경우에는 수정하는 게 더 간단하다. 객체 지향 원칙에 가깝게 클래스를 나눌 수 있지만 어떤게 더 실용적인지.    
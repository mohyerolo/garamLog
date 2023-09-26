---
title: [자바 디자인 패턴의 이해] 1강 - Strategy Pattern
layout: post
categories: coding
tags: theory
---

## 인프런 - 자바 디자인 패턴의 이해 1강

인터페이스: 기능에 대한 선언과 구현이 분리된 것

```java
public interface AInterface {
    // 기능 선언
    public void funcA();
}
```

```java
// 기능 구현 클래스
public class AInterfaceImpl implements AInterface {
    // 기능 구현
    @Override
    public void funcA() {

    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        AInterfaceImpl aInterface = new AInterfaceImpl();
        aInterface.funcA();
    }

}
```

<hr>

델리케이트: 위임하다.    
복잡한 기능을 다른 곳에서 개발하고 가져다가 쓸 수 있도록 하는 것.    

```java
public class AObj {

    AInterface aInterface;

    public AObj() {
        aInterface = new AInterfaceImpl();
    }

    public void funcAA() {
//        System.out.println("AA");
        // 어떤 기능을 개발할때 그 책임을 다른 객체에 부여하는 것 -> 위임
        aInterface.funcA();
    }
}
```

```java
public class Main {
    public static void main(String[] args) {
        AInterfaceImpl aInterface = new AInterfaceImpl();
        aInterface.funcA();

        AObj aObj = new AObj();
        aObj.funcAA();
    }
}
```

-> 델리게이트: 특정 객체의 기능을 사용하기위해 다른 객체의 기능을 호출하는 것.

<hr>

### 스트레티지 패턴 (Strategy Pattern)
여러 알고리즘을 하나의 **추상적인 접근점** 을 만들어 접근 점에서 서로 **교환 가능** 하도록 하는 패턴    

추상적인 접근 -> 인터페이스    

<img src="https://github.com/mohyerolo/mohyerolo.github.io/assets/68698007/e794fda7-65da-44b5-8394-35365a6a3d04">

<hr>

### 예제
```java
public interface Weapon {
    void attack();
}
```    

```java
public class Knife implements Weapon {
    @Override
    public void attack() {
        System.out.println("Knife attack");
    }
}
```    

```java
public class Sword implements Weapon {
    @Override
    public void attack() {
        System.out.println("Sword attack");
    }
}
```    

```java
public class Character {
    // 접근점
    private Weapon weapon;

    // 교환 가능
    public void setWeapon(Weapon weapon) {
        this.weapon = weapon;
    }

    public void attack() {
        if (weapon == null) {
            System.out.println("no weapon");
        } else {
            // 델리게이트: 공격이라는 기능이 어떤 걸 들고있는지에 따라 형태가 달라지기 때문
            weapon.attack();
        }
    }
}
```    

```java

public class Main {
    public static void main(String[] args) {
        Character character = new Character();
        character.attack();

        character.setWeapon(new Knife());
        character.attack();

        character.setWeapon(new Sword());
        character.attack();
    }
}
```    

[실행결과]    
no weapon    
Knife attack    
Sword attack    


if) 새로운 무기가 생긴다면

```java
public class Ax implements Weapon {
    @Override
    public void attack() {
        System.out.println("Ax attack");
    }
}
```    

```java
public class Main {
    public static void main(String[] args) {
        Character character = new Character();
        character.attack();

        character.setWeapon(new Knife());
        character.attack();

        character.setWeapon(new Sword());
        character.attack();

        character.setWeapon(new Ax());
        character.attack();
    }
}
```    


[실행결과]    
no weapon    
Knife attack    
Sword attack    
Ax attack    

그냥 추가만 해주면 됨    

interface를 사용하면 다 스트레티지 패턴인것?   

<br>


## 따로 공부
스트레티지 패턴은 각각을 **캡슐화** 하여 교환해서 이용할 수 있도록 만들어 클라이언트의 영향 없이 독립적으로 알고리즘 변경이 가능하다.    

상속으로 위와 같은 기능들을 구현 했을 때의 문제점    
- 요구사항 변경이 생겼을 때, 서브 클래스 중 상속 클래스의 기능 중 사용하지 않는 게 있거나 다른 행동을 해야 한다면 재정의 필요    

### 상속이 갖는 단점
- 강한 결합이 강제된다.    
    - 부모 클래스의 변화가 모든 자식 클래스에 영향을 미침   
    - 부모 클래스의 기능이 하나라도 변경되면 자식 클래스가 제대로 돌아가는지 많은 테스트 필요    
    - 복잡해지면 코드의 유지보수와 수정을 어렵게 만듬
        - 상속을 받고 준 클래스들이 많을 수록 클래스간의 구조가 복잡해져 나중에는 오히려 보수가 힘들어지고 어떤 결과를 실행할 지에 대한 예측이 힘들어진다    
- 접근자에 대한 제약이 강하게 걸린다.    
- 정적인 코드가 된다.
- 캡슐화와 은닉을 깨뜨릴 수 있다.    
- 상속은 한 번만 가능하다.    

이런 단점을 해결하기 위해 인터페이스로 구현을 한다면 구현 기능이 필요없는 클래스는 해당 기능을 구현하지 않으면 된다.    

그러나 **여전히 하나의 기능을 고치려고 하면 해당 기능을 구현한 모든 클래스들을 고쳐야되고 재사용할 수 없다는 단점** 도 있다.    

결국은 최대한 **유연하게 만드는 것** 이 중요.    
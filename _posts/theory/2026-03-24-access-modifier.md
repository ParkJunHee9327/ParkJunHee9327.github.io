---
layout: single
title: 접근 제어자 종류, 사용시 주의사항
categories:
  - theory
---

## 개요

- 클래스, 변수, 메서드 등의 접근성을 설정하는 키워드이다.
- OOP의 4가지 원칙 중 캡슐화와 밀접한 관련이 있다.

## private

- 자신이 선언된 클래스 내에서만 접근 가능하도록 설정한다.
- 자식 클래스를 포함한 모든 다른 클래스들이 접근할 수 없다.
- 인스턴스 변수들, 내부적으로 사용되는 로직에 쓰인다.

```java
public class BankAccount {
    private double balance; // Only accessible inside BankAccount

    private void logTransaction() { // A helper method only for this class
        // ...
    }
}
```

## default

- 동일한 패키지 내에서만 접근 가능하게 설정한다.
- 아무런 접근 제어자도 사용하지 않았을 때의 기본적인 값이다.
- 자식 클래스라고 해도 패키지 외부에 있으면 접근 불가능하다.
- 패키지 내에 서로 관련된 클래스들을 구성하는 경우, 외부에 드러내지 않도록 사용한다.

```java
class Logger { // No modifier = default access
    String format; // Accessible within the 'com.utilities' package

    void logMessage(String msg) { // Accessible within the package
        // ...
    }
}
```

## protected

- 동일한 패키지 내부 또는 자식 클래스가 접근 할 수 있게 한다.
- default와 비슷하지만 패키지 외부에 있는 자식 클래스도 접근할 수 있다는 점에서 차이가 있다.
- 상속을 사용할 때 유용한 접근 제어자이다.

```java
public class Vehicle {
    protected String engineType; // Accessible in package and to subclasses

    protected void startEngine() {
        // Subclasses like Car or Bike can use or override this.
    }
}
```

## public

- 프로그램 내의 어디서든지간에 접근 가능하다.
- 접근하기 쉬우므로 신중하게 사용해야 한다.

```java
public class Calculator {
    public int add(int a, int b) { // Accessible from anywhere
        return a + b;
    }
}
```

## 유의사항

- 최고 레벨의 클래스 또는 인터페이스는 public이거나 default이어야 한다.
- 생성자는 모든 접근 제어자를 사용할 수 있다. Singleton 패턴을 사용할 때는 private 생성자를 사용한다.
- 메서드를 오버라이딩 할 때 접근성을 축소할 수 없다. 다만, 접근성을 늘릴 수는 있다.
    - ❌ public 메서드 → private 메서드로 오버라이딩
    - ✅ private 메서드 → public 메서드로 오버라이딩
- 자식 클래스와 부모 클래스가 서로 다른 패키지에 위치할 때, 부모 클래스의 요소가 protected 혹은 public이어야만 자식 클래스가 접근 가능함을 유의한다.

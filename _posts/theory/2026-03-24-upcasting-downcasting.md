---
layout: single
title: 업캐스팅과 다운캐스팅
categories:
  - theory
---

## 업캐스팅

- 자식 클래스의 타입을 부모 클래스의 타입으로 사용하는 방식이다.
- 암묵적이든 명시적이든 언제나 사용하기 안전하다.
- 컴파일러가 자동적으로 허용한다.
- 런타임 타입이 변하지 않고 유지된다. (런타임 타입 = 실제로 사용되는 자식 클래스 타입)
- 추상화를 기반으로 다형성의 이점을 활용하기 위해 많이 쓰인다.

```java
class Animal {}
class Dog extends Animal {}

Dog dog = new Dog();
Animal animal = dog;  // Upcasting (implicit)
// OR
Animal animal = (Animal) dog;  // Upcasting (explicit)
```

## 다운캐스팅

- 부모 클래스의 타입을 자식 클래스의 타입으로 사용하는 방식이다.
- 안전하는 보장이 없어서 명시적으로 캐스팅 해야 한다.
    - 캐스팅이 성립되지 않으면 ClassCastException이 발생한다.
    - instanceof으로 다운캐스팅이 되는지 확인하여 예외를 방지하는 게 좋다.
- 런타임에 타입을 확인하는 과정이 필요하다.
- equals() 메서드 등 일반적인 클래스를 구체화된 클래스로 변환하는 경우가 아니면 흔하게 사용되지 않는다.
    - equals()는 파라미터로 Object를 받는다.
    - 클래스 내에서 equals()를 오버라이딩할 때 반드시 다운캐스팅을 해야 한다.
    
    ```java
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof User)) return false;
        
        // Downcasting is required here to access .username
        User other = (User) obj; 
        return this.username.equals(other.username);
    }
    ```
    

```java
Animal animal = new Dog();
Dog dog = (Dog) animal;  // Downcasting (explicit, safe here)

Animal animal2 = new Animal();
Dog dog2 = (Dog) animal2;  // Runtime error: ClassCastException
```

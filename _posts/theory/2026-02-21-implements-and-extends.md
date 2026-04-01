---
layout: single
title: extends와 implements
---

## 개요

- extends와 implements는 둘 다 상속에 사용된다는 공통점이 있으나, 상속하는 구조에 있어 차이가 있다.
- 간단하게 말해 존재하는 행동을 물려주느냐(class-to-class), 새로운 행동을 만드는 데 동의하느냐(class-to-inteface)의 차이다.

## extends

- 클래스를 상속하는 데 사용된다. (Implementation Inheritance)
- 자식 클래스와 부모 클래스가 “is a” 관계를 맺는다. (예시: a Dog is an Animal)
- 하나의 클래스만 상속받을 수 있다.
- 일반적으로 필드, 바디가 있는 메서드를 상속받는다. (abstract 메서드는 예외)
- 대부분 메서드 오버라이딩이 선택이다. (abstract 메서드는 예외)

## implements

- 인터페이스를 구현한다. (Interface Inheritance)
- 클래스와 인터페이스가 “can do” 관계를 맺는다. (예시: a Dog can Run)
- 여러가지 인터페이스들을 구현할 수 있다. (구체적 로직이 없어서 다이아몬드 문제 X)
- 일반적으로 메서드 시그니처만 상속받는다. (default 메서드는 예외)
- 대부분 메서드 오버라이딩이 필수다. (default 메서드의 경우, 서로 관련 없는 인터페이스들이 동일한 deafult 메서드를 지닌 경우 필수)

➕ extends와 implements가 물려주는 요소의 차이

- 질문 배경
    - extends와 implements 둘 다 클래스에게 무언가를 물려준는 점에서 동일하지 않나?
    - 왜 extends에서는 “is a” 관계를 요구하고 implements는 “can do” 관계를 요구하는가?
- extends는 자식에게 필수적인 요소들을 물려준다.
    - 그렇기에 부모 클래스의 필드(state)와 구체적인 로직(메서드)를 물려주는 것이다.
    - 자식 클래스가 코드를 재작성할 필요를 없애려는 목적이 있다.
    - 예시: Animal 클래스가 Dog, Cat 클래스에게 heartRate 변수와 eat() 메서드를 물려준다. (모든 동물들은 심박수가 있고, 먹을 수 있음)
- implements는 선택적인 요소들을 물려준다.
    - 구현하는 클래스가 수행할 수 있는 행동들을 정의하는 목적이 있다.
    - 동일한 작업을 수행할 수 있는 여러 클래스들을 갈아끼울 수 있는 구조(OOP의 다형성)를 만드는 목적이 있다.
    - 예시: Dog 클래스가 Soundable 인터페이스에서 makeSound() 메서드를 받아 구현한다. (Dog는 소리를 낼 수 있음)
    - 예시: Fish 클래스가 Swimmable 인터페이스에서 swim() 메서드를 받아 구현한다. (Fish는 수영할 수 있음)
- 코드 예시
    - ✅ 클래스가 필수적 요소를, 인터페이스가 선택적 요소를 물려주는 모습
        
        ```java
        // 필수적인 요소를 물려주는 클래스
        class Animal {
            void eat() { System.out.println("Eating..."); }
        }
        
        // 선택적인 요소를 물려주는 인터페이스
        interface Climbable {
            void climb();
        }
        
        // Cat is an Animal이고 Cat can do Climbing임
        class Cat extends Animal implements Climbable {
            public void climb() { System.out.println("Climbing high!"); }
        }
        
        // Dog is an Animal임. 그러나 Climbing은 못 함
        class Dog extends Animal {
            void bark() { System.out.println("Woof!"); }
        }
        ```
        
    - ❌ 클래스가 선택적인 요소를 물려주는 모습
        
        ```java
        class Animal {
        		// 필수적 요소
            void eat() { System.out.println("Eating..."); }
            
            // 선택적 요소
            void makeSound() { System.out.println("I'm making sound!"); }
        }
        
        // Cat은 eat과 makeSound 모두 할 수 있음
        class Cat extends Animal {
            public void eat() { System.out.println("Eating meat!"); }
            public void makeSound() { System.out.println("Meow!"); }
        }
        
        // Fish is an Animal은 맞음
        // 그러나 Fish는 소리를 내지 못함
        class Fish extends Animal {
            public void eat() { System.out.println("Eating seaweed!"); }
            public void makeSound() { System.out.println("??"); }
        }
        ```
        
    - ❌ 인터페이스가 필수적인 요소를 물려주는 모습
        - 인터페이스는 인스턴스 필드를 생성할 수 없으므로 클래스에게 변수를 물려줄 수 없다.
            - static 필드는 상속될 수 있으나, 객체마다의 고유한 값을 설정할 수도 없고 상태가 있지도 않아서 상속인가에 대해 논란이 있다.
        - default 메서드로 바디가 있는 메서드를 클래스에 줄 수 있지만 extends로 상속받는 메서드와 목적이 다르다.
            - extends로 상속받는 메서드의 “로직과 상태를 공유한다”는 목적이 아닌 “이전 버전과의 호환성을 유지한다”는 목적이 있다.
            - 상속받아 봤자 클래스의 인스턴스 변수에 접근하지도 못한다.

➕ extends는 implements보다 덜 쓰인다?

- 질문 배경
    - 내가 개발을 하면서 extends를 쓸 일은 사실상 없었다. 주로 implements를 썼지.
    - 근데 생각해보니 Spring Security, Java의 기본적 클래스들(JDK에 있는 클래스들)에는 extends가 많이 쓰이더라.
    - 현대에 와서 extends가 구식이 되어버린 것일까?
- extends가 덜 쓰인다 보다는 implements와 역할이 달라졌다고 보는 게 적절하다.
- implements는 구현체가 무엇을 할 수 있는지를 나타내는 계약 사항이다.
    - 클래스가 구현체가 아닌 추상화에 의존하도록 하여 느슨한 커플링을 형성한다.
    - 추상화에 상황에 맞는 구현체를 갈아끼울 수 있어 테스트가 수월해진다.
    - 클라이언트의 요구에 따라 코드가 유연하게 변경되어야 하는 비즈니스 로직 개발에 주로 쓰인다.
- extends는 클래스의 신원 또는 상태를 나타낸다.
    - 자식 클래스가 부모 클래스의 필드와 메서드를 물려받아 클래스들의 위계 구조를 확립한다.
    - Generic에 범위를 지정하는 등 타입 안전성을 유지하는 역할을 한다.
        
        ```java
        // This means "T must be a type that implements List"
        public <T extends List> void processList(T list) { ... }
        ```
        
    - 잠재 고객들인 개발자에게 코드의 일관되고 엄격한 구조를 보장해주어야 하는 라이브러리 또는 프레임워크 개발에 주로 쓰인다.

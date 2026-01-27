---
layout: single
title: 메서드 Overriding / Overloading
---
## 메서드 오버로딩(컴파일 다형성)

<aside>
💡

하나의 클래스 내에서 같은 이름의 다른 파라미터 구조를 지닌 메서드들을 구현함.

</aside>

## 특징

- 하나의 클래스 내에 있는 같은 이름의 다른 파라미터 종류, 순서, 수량을 지닌 메서드들을 구현함을 의미한다.
- 메서드 시그니처에 속하지 않는 요소들은 상관하지 않는다. (예시: 반환값, 접근 제어자, 메서드 바디 등)
- 다양한 입력값(자료형 또는 수량)에 대해 비슷한 작업을 하려는 목적이다.

## 컴파일 타임 다형성의 의미

- 정적 바인딩(binding), 얼리 바인딩으로 불리기도 한다.
- 메서드의 시그니처, 메서드의 이름과 파라미터가 **컴파일 타임에 확인(컴파일러가 확인함)**되므로 컴파일 다형성이라고 불린다.

➕다른 이름의 같은 파라미터의 메서드도 메서드 오버로딩일까?

- 메서드 시그니처는 메서드 이름과 파라미터(순서, 수량, 타입)로 구성된다. 그럼 “같은 이름의 다른 파라미터” 말고 “다른 이름의 같은 파라미터”는 메서드 오버로딩이 될 수 없을까?
- 메서드 오버로딩은 **개념적으로 동일한 작업**을 여러가지 방식으로 수행하는 목적이 있다.
- 개념적으로 동일한 작업을 나타내는 수단이 **메서드의 이름**이다. 따라서 다른 이름의 같은 파라미터를 지닌 메서드들은 오버로딩 되었다고 할 수 없다.
- 코드 예시
    
    ```java
    // 올바른 오버로딩
    class ValidOverloader {
        // Different parameter types
        void process(int x) { }
        void process(String s) { }
        
        // Different number of parameters  
        void process(int a, int b) { }
        
        // Different parameter order
        void process(String s, int i) { }
        void process(int i, String s) { }
    }
    
    // 메서드 오버로딩이 아님
    class NotOverloader {
        void process(String data) { }
        void handle(String data) { }     // Different name = different method
        void transform(String data) { }  // Different name = different method
    }
    ```
    

➕연관되는 클래스가 하나인데 왜 컴파일 “다형성”이지?

- OOP의 다형성은 “여러 자식 클래스들이 하나의 부모 클래스로 취급될 수 있다”는 원칙이다. → “메서드 오버로딩은 컴파일 다형성이라는데, 그럼 메서드 오버로딩도 여러 클래스들이 수반되는 개념일까?”
- ❗메서드 오버로딩은 “컴파일 다형성”이라 불리지만 **하나의 클래스만 수반**된다.
- ❗메서드 오버로딩의 다형성은 **OOP의 다형성이 아니라 애드혹 다형성(ad-hoc polymorphism)**에 포함되기 때문이다.
- 애드혹 다형성은 “컴파일 타임에 인자에 따라 호출될 메서드를 결정함”을 의미한다. OOP의 다형성과 다르다.
- 덧붙임. 다형성(Polymorphism)은 “다양한 형태”를 의미한다. 프로그래밍에서는 함수 호출, 다양한 데이터 타입을 다루는 인터페이스 등이 “형태”에 해당된다.
- 덧붙임. Java가 OOP 언어이기는 하지만 Java에 속한 모든 개념이 OOP임은 아니라는 점을 시사한다. (애드혹 다형성은 OOP의 개념이 아니다)

➕기술적이기보단 인간적인 개념이다

- 컴파일러는 메서드를 시그니처(이름 + 파라미터)로 식별한다. 즉 오버로딩 된 메서드들은 컴파일러에게는 각기 다른 메서드들일 뿐이며, **특별하게 오버로딩 된 메서드라고 인식하지 않는다**.
- 메서드 오버로딩은 같은 개념의 작업을 다양한 방식으로 처리한다는 **개발자들의 편의**로 만들어진 개념이다.

**코드 예시**

- 수학적 작업
    
    ```java
    class MathOperations {
    
        // Overloaded method to add two integers
        int add(int a, int b) {
            return a + b;
        }
    
        // Overloaded method to add three integers (different number of parameters)
        int add(int a, int b, int c) {
            return a + b + c;
        }
    
        // Overloaded method to add two doubles (different parameter types)
        double add(double a, double b) {
            return a + b;
        }
    }
    ```
    

## 메서드 오버라이딩(런타임 다형성)

<aside>
💡

자식 클래스에서 부모 클래스가 정의한 메서드를 구현하는 것.

</aside>

### 특징

- 위계 관계가 있는, 적어도 2개의 클래스들이 관련된다.
- 자식 클래스의 구현체는 부모 클래스의 메서드와 시그니처가 동일해야 한다. (같은 이름, 같은 파라미터 구조)
- 부모 클래스에 정의된 바를 따르면서도 자식 클래스에게 맞춤형인 작업을 수행하려는 목적이다.

### 런타임 다형성의 의미

- 동적 바인딩, 느린 바인딩으로도 불린다.
- 호출되는 실제 객체의 타입은 **런타임 때 확인(JVM이 확인)**되므로 런타임 다형성이라고 불린다.
- 코드 예시
    
    ```java
    // 참조값은 Animal, 실제 객체는 Dog
    // 컴파일러는 참조값(Animal)만 봄
    // JVM에 런타임에 Dog 객체를 확인하고
    // Dog의 메서드를 호출함
    Animal myPet = new Dog();
    myPet.makeSound();
    ```
    

➕메서드 오버라이딩의 규칙

- 자식 클래스의 구현체의 접근 제어자는 부모 클래스가 정의한 메서드의 **접근 제어자와 동일하거나 범주가 더 넓어야** 한다.
    - ❌ 부모는 public → 자식은 private
- 자식 클래스의 구현체는 부모 클래스가 정의한 메서드에서 던지는 예외보다 **더 넓은 범주의 확인된 예외를 던질 수 없다**.
    - ❌(부모 클래스가 IOException을 던진다면 자식은 Exception을 던질 수 없음)
- **확인되지 않은 예외**의 경우 부모 클래스가 던지는 예외와 무관하게 **아무 예외나 던질 수 있다**.
- 자식 클래스가 **static 메서드는 구현할 수 없다**. (Method Hiding)
    - static 메서드는 클래스 자체게 속하여 컴파일 때 인지되기 때문이다.
        - 실제 객체의 타입을 확인하는 다형성은 런타임에 적용되는 원리이다.
    - 자식 클래스가 부모 클래스의 메서드의 시그니처와 동일한 static 메서드를 만들면 Method Hiding이 발생한다.
    
    ```java
    // static 메서드는 컴파일 때 인식된다.
    // 컴파일러가 정적 메서드를 호출하는 대상은
    // 참조되는 타입이다. 즉 Parent다.
    // 따라서 Parent의 static 메서드가 실행된다.
    // 자식의 static 메서드가 "숨겨진다".
    Parent p = new Child();
    p.staticMethod();
    ```
    
- **private 메서드, final 메서드도 구현할 수 없다**.
    - private 메서드는 자식의 시점에서 인지될 수 없다.
    - final로 선언된 대상은 불변하기 때문에 구현할 수 없다.
- 자식 클래스의 구현체는 **부모 클래스의 메서드가 반환하는 타입** 또는 **그 반환 타입의 서브 타입**을 반환할 수 있다.
- **@Override 어노테이션은 거의 필수**다. 오타 확인, 의도치 않은 오버라이딩, 가독성 증진 등의 이점이 있다.

➕뭐가 메서드 시그니처가 아닌 것들에 대한 제약 사항을 강제하는가?

- 메서드 오버라이딩을 할 때 동일한 메서드 시그니처를 따라야 하는 것은 이해할 수 있다. 그래야 컴파일러가 동일한 메서드라고 인식할 테니. 그럼 “뭐가 메서드 시그니처가 아닌 것들(반환값, 접근 제어자 등)에 대해 제약을 거는 걸까?”
- 메서드 오버로딩에 대한 규칙은 **Java Language Specification, 즉 JLS가 정의**한다.
- JSL에 메서드 시그니처(이름 + 파라미터), 반환값, 접근 제어자, 예외 타입, final 메서드에 대해 명시되어 있다.
- 단! **static 메서드를 구현할 수 없다는 것은 명시되어 있지 않다**. 본질적으로 “static 메서드는 컴파일 때 참조되는 클래스에 따라 호출되기 때문에” 구현이 불가능 한 것이다.
- 코드 예시
    
    ```java
    class Parent {
        public static void staticMethod() {
            System.out.println("Parent static");
        }
        
        public void instanceMethod() {
            System.out.println("Parent instance");
        }
    }
    
    class Child extends Parent {
        // This is METHOD HIDING, not overriding
        public static void staticMethod() {
            System.out.println("Child static");
        }
        
        // This is METHOD OVERRIDING
        @Override
        public void instanceMethod() {
            System.out.println("Child instance");
        }
    }
    
    // Test the behavior
    Parent obj = new Child();
    
    obj.staticMethod();    // Output: "Parent static" (reference type)
    obj.instanceMethod();  // Output: "Child instance" (object type)
    ```
    

**코드 예시**

- 동물 부모 클래스
    
    ```java
    class Animal {
        // Method to be overridden
        public void makeSound() {
            System.out.println("The animal makes a sound.");
        }
    }
    
    class Dog extends Animal {
        // Overriding the parent class method
        @Override
        public void makeSound() {
            System.out.println("The dog barks: Woof Woof!");
        }
    }
    
    class Cat extends Animal {
        // Overriding the parent class method
        @Override
        public void makeSound() {
            System.out.println("The cat meows: Meow Meow!");
        }
    }
    ```

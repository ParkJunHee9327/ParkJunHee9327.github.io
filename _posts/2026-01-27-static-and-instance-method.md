---
layout: single
title: 인스턴스 메서드와 static 메서드
---
## 인스턴스 메서드

<aside>
💡

클래스의 단일한 인스턴스가 실행할 수 있는 메서드.

</aside>

### 특징

- 클래스의 인스턴스로만 호출될 수 있다.
- 클래스의 정적 필드와 인스턴스 필드에 모두 접근할 수 있다.
- 메서드를 호출하는 인스턴스를 가리키는 this 키워드를 사용할 수 있다.
- 새로운 인스턴스를 생성할 때 마다 메모리에 로딩된다.

### 유스 케이스

- 각각의 인스턴스들에 특정된 작업을 수행할 목적으로 정의된다.
- 메서드의 작업이 객체의 상태에 따라 좌우될 때
- 객체의 필드를 수정할 때
- 비즈니스 로직 등 애플리케이션의 핵심 로직을 수행할 때

➕인스턴스 메서드는 인스턴스에 속하고, static 메서드는 클래스 자체에 속한다?

- 위의 문구는 인스턴스 메서드와 static 메서드를 설명할 때 흔하게 접할 수 있는 문구다. 그러나 인스턴스 메서드도 클래스에 정의되어 있다. 내 눈에는 사실상 “인스턴스 메서드든 static 메서드든 클래스에 속했다”고 보인다.
- 인스턴스 메서드와 static 메서드는 모두 클래스의 정보로서 JVM의 메서드 영역에 보관된다. 즉 **둘 다 클래스에 속한다**고 할 수 있다.
- 위 말의 요지는 “인스턴스 메서드는 반드시 인스턴스가 필요하지만, static 메서드는 필요하지 않다”이다.
- 인스턴스 메서드는 실행될 때 실행의 대상이 되는 객체를 참조하기 위해 **암묵적으로 this 키워드를 사용**한다. 따라서 인스턴스가 없으면 안 된다.
- static 메서드는 클래스 그 자체에 속하므로 인스턴스가 없어도 실행된다.
- 인스턴스 메서드의 this 사용 코드 예시
    
    ```java
    public class Calculator {
        private int value;  // Instance field
        
        // Instance method
        public void add(int number) {
            this.value += number;
        }
        
        // Static method  
        public static int multiply(int a, int b) {
            return a * b;
        }
    }
    
    // JVM이 보는 인스턴스 메서드의 모습.
    // 개발자가 명시적으로 this 파라미터를 넣지 않으면 
    // 컴파일러가 넣는다.
    public void add(Calculator this, int number) {
        this.value += number;
    }
    
    // JVM이 보는 static 메서드의 모습.
    // static 메서드는 숨겨진 this 파라미터가 없다.
    public static int multiply(int a, int b) {
        return a * b;
    }
    ```
    

**코드 예시**

- 자동차 클래스
    
    ```java
    public class Car {
        // Instance variable (each Car has its own)
        private String model;
        private int speed;
    
        // Static variable (shared by all Cars)
        private static int numberOfCars = 0;
    
        // Constructor
        public Car(String model) {
            this.model = model;
            this.speed = 0;
            numberOfCars++; // Increment the shared static variable
        }
    
        // --- INSTANCE METHODS ---
        // Actions that a specific Car can do.
    
        public void accelerate(int increment) {
            this.speed += increment; // 'this' refers to this specific object
            System.out.println(model + " is accelerating. Speed: " + speed + " mph");
        }
    
        public void displayInfo() {
            // Can access BOTH instance AND static members
            System.out.println("Model: " + this.model + ", Speed: " + speed);
            System.out.println("Total cars created: " + numberOfCars); // Accessing static variable
        }
    }
    ```
    

## static 메서드

<aside>
💡

클래스의 인스턴스들에 대해 독립적으로 실행될 수 있는 메서드.

</aside>

### 특징

- 클래스 그 자체와 클래스의 인스턴스를 통해서 호출될 수 있다.
- 클래스의 정적 필드에만 접근할 수 있다.
- 인스턴스를 참조하지 않으므로 this 키워드를 사용할 수 없다.
- 클래스가 처음으로 메모리에 로딩될 때 한 번 로딩된다.
- 객체의 상태와 무관한 유틸리티 기능을 공유할 목적으로 정의된다.

### 유스 케이스

- 유틸리티 또는 헬퍼 메서드가 필요할 때
- 애플리케이션의 설정에 따라 인스턴스를 생성하는 팩토리 메서드를 구성할 때
- 입력값에 따라서만 동작하는 기능성 메서드가 필요할 때

➕왜 인스턴스에 static 메서드를 호출해도 this 키워드를 사용할 수 없을까?

- static 메서드는 기술적으로 인스턴스를 통해서도 호출될 수 있다. 그런데 왜 이런 경우에도 this 키워드를 사용할 수 없을까?
- **static 메서드는 컴파일 때 선언된 타입을 기반으로 작동**한다. 즉 런타임 때 결정되는 실제 객체 타입인 인스턴스와 무관하다.
- 따라서 인스턴스를 통해 static 메서드를 호출한다 한들 this 키워드는 사용할 수 없다.
- 코드로 보는 예시
    - 흔히 하는 예상: static 메서드가 인스턴스(Child)를 통해 실행됐으니까, static 메서드가 Child를 참조하려고 this를 쓸 수 있지 않을까?
    - 사실: 정적 메서드인 staticMethod는 컴파일 때 처리되며, 실행될 때 선언된 타입, 즉 Parent만 필요하다. 런타임 때 사용되는 실제 타입인 Child는 전혀 고려하지 않는다. 즉 Child를 참조할 수 있는 this를 사용할 수 없다.
    
    ```java
    Parent pa = new Child();
    pa.staticMethod();
    ```
    

➕자바에서 가장 유명한 static 메서드인 main 메서드를 알아보자

- main 메서드의 정확한 생김새는 “public static void main(String[] args)”이다.
- public: 메서드가 어디서든지 접근 가능하여 프로그램의 외부에 있는 **JVM이 접근하도록 허용**한다.
- static: 메서드를 인스턴스가 아닌 클래스 그 자체에 속하게 만든다. 즉 **인스턴스가 없어도 애플리케이션이 실행될 수 있게** 만든다.
- void: main 메서드의 목적인 애플리케이션을 실행하는 것이다. 무언가를 반환하기 위함이 아니다.
- main: **애플리케이션을 실행하는 메서드의 이름**으로, JLS에 명시되어 있다. 즉 이름을 begin, start 등으로 바꾸면 애플리케이션이 실행되지 않는다.
- String[] args: 프로그램이 들여오는 **커맨드 라인 아규먼트**들이다.

**코드 예시**

- 자동차 클래스
    
    ```java
    public class Car {
        // Instance variable (each Car has its own)
        private String model;
        private int speed;
    
        // Static variable (shared by all Cars)
        private static int numberOfCars = 0;
    
        // Constructor
        public Car(String model) {
            this.model = model;
            this.speed = 0;
            numberOfCars++; // Increment the shared static variable
        }
    
        // --- STATIC METHODS ---
        // Utility functions related to Cars, but not to any single Car.
    
        public static void displayTotalCars() {
            // Can only access STATIC members
            System.out.println("Total number of cars: " + numberOfCars);
    
            // COMPILE-TIME ERROR: Cannot access instance members
            // System.out.println(model); // What model? There's no object!
            // System.out.println(this.speed); // No 'this' in a static context!
        }
    
        // A common utility static method
        public static int convertMphToKph(int mph) {
            return (int)(mph * 1.60934);
        }
    }
    ```

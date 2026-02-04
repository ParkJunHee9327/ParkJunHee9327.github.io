---
layout: single
title: 왜 static 메서드는 오버라이딩 되지 않고, 인스턴스 메서드는 오버라이딩이 될까?
---

### 결론

- static 메서드는 오버라이딩이 불가능하고, 인스턴스 메서드는 오버라이딩이 가능한 이유는 “**메서드가 실행되는 방식이 다르기 때문**”이다.

### static 메서드는 JVM의 메서드 영역에 보관된다

- 클래스가 로딩되면 static 메서드가 JVM의 메서드 영역에 보관된다.
- static 메서드는 컴파일 시간에 확인되는 메모리 주소를 지닌다.
- JVM의 메모리 구조도
    
    ```java
    JVM Memory (Simplified)
    ┌─────────────────┐
    │    Method Area  │ ← Class metadata, static variables, static methods
    │  (Class Area)   │
    ├─────────────────┤
    │      Heap       │ ← Object instances, instance variables
    ├─────────────────┤
    │   Stack         │ ← Method calls, local variables, object references
    └─────────────────┘
    ```
    

### static 메서드의 실행 과정

- 컴파일 시간에 컴파일러가 메서드에 있는 static 키워드를 확인하고, 변수를 선언한 타입의 클래스가 정의한 static 메서드를 실행한다.
- 코드 예시
    
    ```java
    class Animal {
        public static void staticMethod() {
            System.out.println("Animal static method");
        }
    }
    
    class Dog extends Animal {
        public static void staticMethod() {
            System.out.println("Dog static method");
        }
    }
    
    // Animal이 참조하는 타입이다.
    Animal myDog = new Dog();
    myDog.staticMethod(); // Outputs "Animal static method" - NO polymorphism!
    ```
    

### 왜 static 메서드는 오버라이딩 될 수 없는가

- 메서드 오버라이딩은 런타임에 실제 객체를 확인하는 과정에 의해 이뤄진다.
- static 메서드를 호출할 때는 **실제로 사용되는 객체의 클래스 자체를 확인하지 않으므로** 오버라이딩될 수 없다.
- static 메서드는 클래스 그 자체에 속하기 때문에, 자식 클래스가 같은 시그니처의 static 메서드를 정의해도 부모의 static 메서드와 개별적인 메서드가 된다.

➕메서드 하이딩(Method Hiding)

- 기술적으로 자식 클래스는 부모 클래스의 시그니처와 동일한 static 메서드를 정의할 수 있다.
- 이는 메서드 오버라이딩이 아닌 메서드 하이딩이라 불린다.
- 코드로 보는 예시
    
    ```java
    class Animal {
        public static void describe() {
            System.out.println("This is a generic Animal.");
        }
        
        public void makeSound() {
            System.out.println("The animal makes a sound.");
        }
    }
    
    class Dog extends Animal {
        // This HIDES the parent's static method. It does NOT override it.
        public static void describe() {
            System.out.println("This is a specific Dog.");
        }
        
        // This OVERRIDES the parent's instance method.
        @Override
        public void makeSound() {
            System.out.println("The dog barks.");
        }
    }
    
    public class Test {
        public static void main(String[] args) {
            Animal myAnimal = new Dog(); // The reference is Animal, the object is Dog.
    
            // --- Static Method Call (Method Hiding) ---
            myAnimal.describe(); // Output: "This is a generic Animal."
            // ^^^ COMPILE-TIME RESOLUTION ^^^
            // The compiler sees `myAnimal` is of reference type `Animal`.
            // It directly links this call to `Animal.describe()`.
    
            // --- Instance Method Call (Overriding) ---
            myAnimal.makeSound(); // Output: "The dog barks."
            // ^^^ RUNTIME RESOLUTION (Dynamic Dispatch) ^^^
            // The JVM sees the actual object is a `Dog` at runtime.
            // It calls `Dog.makeSound()`, the overridden version.
            
            // The correct way to call static methods:
            Animal.describe(); // "This is a generic Animal."
            Dog.describe();    // "This is a specific Dog."
        }
    }
    ```
    

➕메서드 하이딩에서 “자식 클래스의 static 메서드가 부모 클래스의 static 메서드를 숨긴다”고 하는 이유

- 메서드 하이딩의 예시 코드는 대개 부모 클래스의 static 메서드가 실행되는 모습을 보여준다.
    - 따라서 되려 “부모의 static 클래스가 자식 클래스의 static 메서드를 숨기는 거 아님”이라고 할 수 있다.
- “자식 클래스의 static 메스드가 부모 클래스의 static 메서드를 숨긴다”는 말의 의미는 “자식 클래스만 봐서는 부모에게 같은 시그니처의 static 메서드가 있는지 알아챌 수 없다”는 의미이다.
- static 메서드는 오버라이딩 되지 않는다. 따라서 자식이 부모 클래스의 static 메서드와 시그니처가 동일한 메서드를 만들더라도 자식 클래스만 봐서는 메서드 하이딩을 하고 있는지 알 수 없다.

### 인스턴스 메서드는 가상 테이블을 사용한다

- 인스턴스 메서드 또한 메서드 영역에 저장된다. 그러나 접근되는 방식이 static 메서드와 다르다.
- 인스턴스 메서드들은 **가상 테이블(vtable)**을 통해 접근된다.
- 생성된 객체들은 자신의 클래스가 사용하는 vtable로의 참조값을 지닌다.
- JVM의 메모리 구조도
    
    ```java
    JVM Memory (Simplified)
    ┌─────────────────┐
    │    Method Area  │ ← Class metadata, static variables, static methods
    │  (Class Area)   │
    ├─────────────────┤
    │      Heap       │ ← Object instances, instance variables
    ├─────────────────┤
    │   Stack         │ ← Method calls, local variables, object references
    └─────────────────┘
    ```
    

### vtable(Dispatch Table)에 대하여

- 객체 지향형 언어들이 다형성을 지원하기 위해 사용하는 컴파일러 수준의 데이터 구조이다.
- 본질적으로 메서드들의 메모리 주소로 구성된 배열이다.
- static / private / final로 선언되지 않은 메서드, 즉 virtual 메서드들을 대상으로 한다.
    - 다른 말로 하면 “상속 가능한 메서드”들이다.
- 모든 객체들은 자신이 생성된 클래스의 vtable을 가리키는 포인터를 지닌다.

### 오버라이딩 된 인스턴스 메서드의 실행 과정

- 컴파일 시간의 과정
    - 컴파일러가 변수를 선언한 타입의 **클래스에 호출하려는 메서드가 있음을 확증**한다. (메서드에 대한 기본적인 유효성 검사)
        - “메서드가 존재한다”는 것만 확인하지 “그 메서드의 구현체가 무엇인가”는 모름
    - 컴파일러가 vtable 내에 있는 **메서드의 인덱스**를 확인한다.
        - vtable에 있는 메서드의 인덱스는 해당 메서드를 오버라이딩하는 모든 자식 클래스들의 vtable 내에서도 동일하기 때문에 중요하다.
- 런타임의 과정
    - JVM이 heap 영역에 있는 객체를 확인한 뒤, 객체의 메타 데이터에 있는 **해당 객체의 클래스가 사용하는 vtable을 조회**한다.
    - vtable에서 컴파일러가 찾은 **인덱스에 해당하는 메서드 구현체**를 찾은 뒤 실행한다.

### 오버라이딩되지 않은 인스턴스 메서드의 실행 과정

- 컴파일 시간의 과정: 오버라이딩 된 인스턴스 메서드의 실행 과정과 동일하다.
- 런타임의 과정
    - JVM이 heap 영역에 있는 실제 객체를 확인한다.
    - JVM이 변수를 선언한 클래스의 vtable을 조회한다. (실제 객체를 생성한 클래스를 추적하는 과정이 없다)
    - vtable에서 컴파일러가 찾은 인덱스에 해당하는 메서드 구현체를 찾은 뒤 실행한다.

### 왜 인스턴스 메서드는 오버라이딩 될 수 있는가

- 인스턴스 메서드는 실행될 때 **실제로 사용되는 객체의 타입을 확인**한다.
- 실행하는 과정에서 이뤄지는 **vtable의 조회 과정**을 통해 런타임 다형성인 메서드 오버라이딩이 가능해진다.

### 코드로 보는 예시

- Animal과 Dog 클래스
    
    ```java
    class Animal {
        public static void staticMethod() { /* in Method Area: Animal.staticMethod@0x100 */ }
        public void instanceMethod() { /* in Animal's vtable */ }
    }
    
    class Dog extends Animal {
        public static void staticMethod() { /* in Method Area: Dog.staticMethod@0x200 */ }
        @Override
        public void instanceMethod() { /* in Dog's vtable */ }
    }
    ```
    
- 메모리 위치
    
    ```java
    Method Area:
    ┌──────────────────────┐
    │ Animal Class Data    │
    │ ├─ staticMethod: 0x100  ← Fixed address!
    │ └─ vtable[0]: Animal.instanceMethod@0x300
    ├──────────────────────┤
    │ Dog Class Data       │
    │ ├─ staticMethod: 0x200  ← Fixed address!
    │ └─ vtable[0]: Dog.instanceMethod@0x400    ← Overridden!
    └──────────────────────┘
    
    Heap:
    ┌──────────────────────┐
    │ Animal animalRef = new Dog() │
    │ ├─ object header     │
    │ ├─ _vptr → Dog.vtable  ← Points to Dog's vtable!
    │ └─ instance vars     │
    └──────────────────────┘
    ```

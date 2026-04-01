---
layout: single
title: Java의 4가지 변수들
---

## 인스턴스 변수 (non-static 필드)

<aside>
💡

클래스 내부에 선언된 변수들 중 static 키워드 없이 선언된 변수.

</aside>

### 특징

- 클래스의 인스턴스마다 할당된 값이 다르다.
- 메서드, 생성자, 코드 블록 바깥에서 선언된다.
- 객체의 상태를 정의한다.

### JVM에서 할당되는 위치

- Heap 영역에 할당된다.
- 객체가 new 키워드로 인스턴스화 될 때 생성된다.
- 객체가 참조되거나 메모리 상 존재하는 경우에 유효하다.
- 객체가 garbage collect되면 제거된다.

**코드 예시**

```java
public class Student {
    // Instance Variable
    private String name;
    private int age;

    // ... rest of the class
}
```

## 클래스 변수 (static 필드)

<aside>
💡

클래스 내에서 static 키워드로 선언된 변수.

</aside>

### 특징

- 클래스로부터 생성되는 인스턴스의 개수에 상관 없이 하나만 존재한다.
- 인스턴스가 아닌 클래스 그 자체에 속한다.

### JVM에서 할당되는 위치

- Method 영역에 할당된다.
- 클래스가 JVM에 처음 로딩될 때 생성된다.
- 프로그램이 종료되거나 JVM이 셧다운되기 전까지 유효하다.

**코드 예시**

```java
public class Student {
    // Class Variable (Static Field)
    public static String schoolName = "ABC High School";

    private String name; // instance variable
}
```

- ➕ 인스턴스에서 가져다 쓸 수 있지만, 쓰지 마세요
    - 기술적으로 인스턴스에서 클래스 필드를 사용하지 못하는 건 아니다.
    - 인스턴스에서 정적 필드를 쓰면 “이거 인스턴스 필드인가?”하고 헷갈릴 위험이 있다.
    - final을 쓰지 않으면 인스턴스가 정적 변수를 수정해버릴 위험도 있다!
    
    ```java
    class MyClass {
        static int count = 0;
    }
    
    public class Main {
        public static void main(String[] args) {
            MyClass obj = new MyClass();
            
            // These all work technically:
            obj.count = 5;           // Through instance
            MyClass.count = 5;       // Through class name (recommended)
        }
    }
    ```
    
- ➕ JVM에서 Method 영역의 위치는?
    - 전통적인 JVM에서 Method 영역은 Heap과 구분되어 있었다.
    - 그러나 HotSpot JVM같은 현대 JVM에서 Method 영역은 Heap의 일부로 구현되고는 한다.
    - 개념적으로 이해할 때는 “Method 영역에 있다”고 하는 게 맞으나, 기술적으로 이해할 때는 “JVM의 garbage collector와 Heap이 관리하는 메모리 공간”이라고 하는 게 맞다.

## 지역 변수

<aside>
💡

코드 블록 내에서 선언되는 변수.

</aside>

### 특징

- 메서드, 생성자, if문의 블록 등에서 선언된다.
- 자신이 선언된 코드 블록 내에서만 사용될 수 있으며, 인식될 수 있다.
    
    ```java
    if (condition) {
    	int myLocalVal = 1;
    }
    
    // 인식 못 함
    int whatIsIt = myLocalVal;
    ```
    

### JVM에서 할당되는 위치

- Stack 영역에 할당된다.
- 지역 변수가 선언된 코드 블록이 실행될 때 생성된다.
- 지역 변수가 선언된 코드 블록을 빠져 나오는 즉시 제거된다.
- 다른 변수들에 비해 생애주기가 유달리 짧은 편이다.

**코드 예시**

```java
public void calculateTotal() {
    // Local Variable
    int total = 0;
    for (int i = 0; i < 10; i++) { // 'i' is also a local variable
        total += i;
    }
} // 'total' and 'i' are destroyed here as the stack frame is popped.
```

- ➕ if문의 코드 블록도 중괄호로 이뤄져 있으니까 메서드라고 볼 수 있는 것 아닌가요?
    - 결론: **둘이 다르다**.
    - 정의와 목적
        - 메서드: 행동을 정의한다.
        - if 구문: 조건에 따라 호출된다.
        
        ```java
        // METHOD - defines behavior, can be called
        public void myMethod() {
            System.out.println("I'm a method");
        }
        
        // IF BLOCK - conditional execution, cannot be called
        if (condition) {
            System.out.println("I'm just a code block");
        }
        ```
        
    - 호출
        - 메서드는 다른 코드에서 호출될 수 있다. myMethod();
        - if 블록은 조건이 참인 경우 자동적으로 실행될 뿐, 호출 매커니즘이 없다.
    - 파라미터와 반환 값
        - 메서드는 파라미터와 반환값이 있다.
        - if문은 둘 다 없다.
        
        ```java
        // METHOD - can have parameters and return values
        public int calculate(int a, int b) {
            return a + b;
        }
        
        // IF BLOCK - no parameters, no return values
        if (x > 5) {
            // Just executes code, can't return anything
        }
        ```
        
    - 둘의 유일한 공통점은 “변수에 대한 범위를 생성한다” 뿐이다.
        
        ```java
        public void example() {
            if (condition) {
                int localVar = 10; // Local to if block
            }
            
            public void someMethod() {
                int methodVar = 20; // Local to method
            }
        }
        ```
        

## 파라미터

<aside>
💡

메서드 시그니처 내에 선언된 변수.

</aside>

## 특징

- 메서드에 값을 전달할 목적으로 선언된다.
- 메서드의 stack frame의 일부이다.

### JVM에서 할당되는 위치

- Stack 영역에 할당된다.
- 메서드가 호출될 때 생성되고 메서드의 사용이 종료되는 즉시 제거된다.
- 지역 변수는 코드 블록, 파라미터는 메서드 블록이라는 점을 제외하면 둘의 생애주기가 동일하다.

**코드 예시**

```java
public void setAge(int studentAge) { // 'studentAge' is a parameter
    // studentAge is accessible here like a local variable
    this.age = studentAge;
}
```

- ➕ “메서드의 stack frame의 일부”라는 말의 의미
    - Call Stack(Stack): JVM이 메서드의 호출과 메서드의 지역 데이터를 추적하는 메모리의 공간. LIFO 방식으로 작동한다.
    - Stack Frame(Activation Record): 메서드가 호출될 때 마다 Call Stack에 쌓이는 메모리의 블록. 메서드를 호출하여 실행할 때 필요한 모든 데이터를 포함한다. 메서드가 끝나면(반환하면) 전체 Stack Frame이 Stack에서 제외되고 삭제된다.
    - Stask Frame에 파라미터(아규먼트), 지역 변수, 반환 주소(메서드 호출이 완료될 수 코드의 실행을 재개할 위치), “this”에 대한 참조(인스턴스 메서드인 경우. 현재 객체를 의미함), 피연산자 Stack(JVM이 계산할 때 사용함)

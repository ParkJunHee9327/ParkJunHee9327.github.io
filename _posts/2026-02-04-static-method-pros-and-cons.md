---
layout: single
title: "Hello, first GitHub Blog!"
---

## static 메서드의 특징

- 인스턴스가 아닌 클래스 그 자체에 속해 있다.
- Java 7 이전에는 Permanent Generation의 Method 영역에 보관되었고, Java 8과 그 이후에는 Metaspace의 Class Metadata에 보관된다.
- 인스턴스 멤버에 접근할 수 없다.
- static 멤버의 종류
    
    ```java
    public class Example {
        // Static variable
        static int staticCounter = 0;
        
        // Static method
        static void printMessage() {
            System.out.println("Static method called");
        }
        
        // Static block
        static {
            System.out.println("Static block executed");
        }
        
        // Static nested class
        static class NestedClass {
            void display() {
                System.out.println("Static nested class");
            }
        }
    }
    ```
    

## 장점

- Heap 영역에 인스턴스를 할당할 일이 없기에 메모리 효율성이 좋다.
- 인스턴스가 많이 생성되어도 복사되지 않고 하나의 값만 존재하기에 메모리 상의 이점이 있다.
- 참조값을 가지고 Heap에 있는 객체를 추적하는 과정이 없어서 실행이 빠르다.

## 단점

- 인스턴스를 사용하지 않기에 의존성 주입, 모킹 및 stubbing이 어렵다. 따라서 테스트를 하기 어려워진다.
- 생성자에 포함되지 않으면서도 클래스에서 사용되기 때문에 클래스에 숨겨진 의존성을 형성할 수 있다.
    - 클래스의 생성자는 해당 클래스가 무엇에 의존하고 있는가를 보여주는 명시적인 요소이다.
    - static 메서드는 인스턴스를 사용하지 않으므로 생성자에 포함되지 않는다.
    - 생성자에 없으면서도 클래스 내에서 쓰인다는 점에서 숨겨진 의존성을 형성한다고 하는 것이다.
    
    ```java
    public class UserService {
        public void createUser(String name) {
            // Hidden dependencies everywhere
            Database.connect();  // Static
            Validator.validate(name);  // Static  
            Logger.log("Creating user");  // Static
            EmailSender.notifyAdmin();  // Static
        }
    }
    // Can't swap implementations, can't test in isolation
    ```
    
- 정적인 디자인이기 때문에 확장 및 수정이 어렵다.

## static 메서드의 OOP 원칙 위배

- static 메서드는 자신의 구조 상 OOP의 상속, 다형성, 캡슐화의 이점을 누리기 힘들다.
- 상속
    - 자식 클래스가 부모 클래스의 static 메서드를 상속받는 건 가능하다.
    - 자식 클래스가 부모 클래스의 static 메서드와 시그니처가 동일한 static 메서드를 지니고 있을 시 오버라이딩 되지도 못한다.
    
    ```java
    class Parent {
        static void inheritedStatic() { System.out.println("Inherited"); }
    }
    
    class Child extends Parent {
        // No static method declaration
    }
    
    // This works - the static method IS inherited
    Child.inheritedStatic();  // Output: "Inherited"
    ```
    
- 다형성
    - static 메서드는 컴파일 때 선언된 변수의 타입을 통해 처리된다.
    - 런타임에 객체의 타입을 확인하는 원리로 이뤄지는 다형성을 지원할 수 없다.
    
    ```java
    interface Formatter {
        static String format(String text) {  // Static in interface (Java 8+)
            return "Default: " + text;
        }
    }
    
    class UppercaseFormatter implements Formatter {
        // Can't override static method!
        // Have to create a separate method
        static String myFormat(String text) {
            return text.toUpperCase();
        }
    }
    
    // What we WANT to do (but can't):
    List<Formatter> formatters = Arrays.asList(new UppercaseFormatter());
    formatters.get(0).format("hello");  // Always calls Formatter.format()
                                         // Never UppercaseFormatter.myFormat()
    ```
    
- 캡슐화
    - 캡슐화는 관련된 데이터들을 클래스의 내부에 배치하여 외부로부터의 직접적인 접근을 제한하거나, 제한적인 접근 방식만 허락하는 원칙이다.
    - static 멤버들은 인스턴스 멤버들에 접근할 수 없고, 다른 클래스에 있는 인스턴스 멤버들도 마찬가지다.
    - 캡슐화를 구현할 때 일반적으로 인스턴스 private 필드와 제한적인 접근자 메서드를 쓰는데, static 멤버들은 이 요소들에 접근할 수 없다.
    
    ```java
    public class Employee {
        private double salary; // Private instance field
        
        public double getSalary() { return salary; }
        public void setSalary(double salary) { 
            if (salary < 0) throw new IllegalArgumentException();
            this.salary = salary; 
        }
    }
    
    public class Payroll {
        // Static method can't directly validate or access Employee's private state
        public static void giveRaise(Employee emp, double amount) {
            // Can only use public interface
            double current = emp.getSalary(); // Might already be invalid!
            emp.setSalary(current + amount); // Relies on setter validation
        }
    }
    ```
    

## 남용하면 안 되는 이유

- final로 선언되지 않을 시 여러 인스턴스들이 값을 수정할 수 있어서 값을 유지하기 힘들다.
- 여러 인스턴스들이 한꺼번에 접근할 수 있기 때문에 동시성 문제가 발생한다.

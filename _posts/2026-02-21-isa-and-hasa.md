---
layout: single
title: extends와 implements
---

## Is-a 관계 (상속)

- 클래스를 상속 받거나 인터페이스를 구현함을 나타낸다.
- 하나의 클래스가 다른 클래스의 구체화된 버전이라는 의미다.
- 커플링이 강하여 부모가 자식에게 영향을 준다.
- 컴파일 타임에 위계 구조가 고정된다.
    - 런타임에 상속받을 부모 클래스를 변경하거나, 조건부적으로 상속받을 수 없다.
- 다형성이 필요하거나 클래스들 간 관계가 변경될 일이 없을 때 사용한다.
- 코드 예시
    
    ```java
    // Base class
    class Vehicle {
        void start() {
            System.out.println("Vehicle starting");
        }
    }
    
    // Derived class - Car IS-A Vehicle
    class Car extends Vehicle {
        void drive() {
            System.out.println("Car driving");
        }
    }
    
    // Interface example
    interface Flyable {
        void fly();
    }
    
    // Bird IS-A Flyable
    class Bird implements Flyable {
        public void fly() {
            System.out.println("Bird flying");
        }
    }
    ```
    

## Has-a 관계 (구성/집합)

- 구성(composition) 또는 집합(aggregation) 관계를 나타낸다.
    - Composition: 강한 전체와 부품의 관계. 부품은 전체가 없이는 존재할 수 없다.
    - Aggregation: 약한 전체와 부품의 관계. 부품은 전체가 없어도 존재할 수 있다.
- 하나의 클래스가 다른 클래스의 요소를 포함하거나 사용한다는 의미이다.
- 커플링이 약하여 포함된 객체가 변경될 수 있다.
- 런타임에 행동이 변경 가능하다.
- 클래스들 간의 관계가 추후에 변경될 수 있거나 상속의 관계 없이 다른 클래스의 기능을 사용하고 싶을 때 따르는 구조이다.
- 코드 예시
    
    ```java
    // Engine class
    class Engine {
        void ignite() {
            System.out.println("Engine ignited");
        }
    }
    
    // Car HAS-A Engine (Composition)
    class Car {
        private Engine engine;  // Composition
        
        public Car() {
            this.engine = new Engine();  // Engine created with Car
        }
        
        void start() {
            engine.ignite();
        }
    }
    
    // Department class
    class Department {
        private String name;
        
        public Department(String name) {
            this.name = name;
        }
    }
    
    // University HAS-A Department (Aggregation)
    class University {
        private List<Department> departments;  // Aggregation
        
        public University(List<Department> departments) {
            this.departments = departments;  // Departments exist independently
        }
    }
    ```
    

### ➕상속보다 구성 원칙(Composition Over Inheritance principle)

- 객체들을 상속보다 구성의 방식으로 구축하는 방식을 제안하는 OOP 디자인 원칙 중 하나다.
- 상속의 강한 커플링, 클래스의 경우 하나밖에 상속할 수 없는 점, 부모 클래스가 자식 클래스에게 끼치는 영향 등상속의 단점을 해소하기 위한 방식이다.
- 상속 대신에 인터페이스 또는 컴포지션 방식을 택한다.
    
    ```java
    // Instead of inheritance, use interfaces and composition
    interface FlyBehavior {
        void fly();
    }
    
    interface QuackBehavior {
        void quack();
    }
    
    class FlyWithWings implements FlyBehavior {
        public void fly() {
            System.out.println("Flying with wings");
        }
    }
    
    class FlyNoWay implements FlyBehavior {
        public void fly() {
            System.out.println("Can't fly");
        }
    }
    
    class Duck {
        private FlyBehavior flyBehavior;
        private QuackBehavior quackBehavior;
        
        public Duck(FlyBehavior flyBehavior, QuackBehavior quackBehavior) {
            this.flyBehavior = flyBehavior;
            this.quackBehavior = quackBehavior;
        }
        
        void performFly() {
            flyBehavior.fly();
        }
        
        void performQuack() {
            quackBehavior.quack();
        }
        
        // Dynamically change behavior at runtime
        void setFlyBehavior(FlyBehavior fb) {
            this.flyBehavior = fb;
        }
    }
    
    // Usage
    Duck mallard = new Duck(new FlyWithWings(), new LoudQuack());
    Duck rubberDuck = new Duck(new FlyNoWay(), new Squeak());
    ```
    
- 장점
    - 유연성 및 런타임 행동의 변화 가능
        
        ```java
        class Character {
            private WeaponBehavior weapon;  // Composition
            
            void setWeapon(WeaponBehavior weapon) {
                this.weapon = weapon;  // Change behavior at runtime
            }
            
            void fight() {
                weapon.useWeapon();
            }
        }
        
        // Character can switch weapons dynamically
        character.setWeapon(new Sword());
        character.fight();  // Uses sword
        
        character.setWeapon(new Bow());
        character.fight();  // Now uses bow
        ```
        
    - 상속보다 나은 캡슐화
        
        ```java
        // Inheritance exposes protected members
        class Base {
            protected String internalData;  // Exposed to subclasses
        }
        
        // Composition keeps implementation hidden
        class Container {
            private Base base;  // Internal implementation hidden
            public void doSomething() {
                // Use base internally
            }
        }
        ```
        
    - 클래스의 과도한 추가 방지
        
        ```java
        // Without composition: combinatorial explosion
        // class FlyingSwordWieldingElf extends Elf {}
        // class FlyingAxeWieldingElf extends Elf {}
        // class WalkingSwordWieldingElf extends Elf {}
        // ... and so on
        
        // With composition: combine behaviors
        class GameCharacter {
            private MovementBehavior movement;
            private WeaponBehavior weapon;
            private Race race;
            
            // Combine independent behaviors
            GameCharacter elfArcher = new GameCharacter(
                new FlyingMovement(),
                new BowWeapon(),
                new ElfRace()
            );
        }
        ```
        
- 실무에서의 구현 패턴
    - Strategy 패턴 (행동적인 구성)
        
        ```java
        interface PaymentStrategy {
            void pay(double amount);
        }
        
        class CreditCardPayment implements PaymentStrategy {
            private String cardNumber;
            
            public void pay(double amount) {
                System.out.println("Paid " + amount + " using credit card");
            }
        }
        
        class PayPalPayment implements PaymentStrategy {
            private String email;
            
            public void pay(double amount) {
                System.out.println("Paid " + amount + " using PayPal");
            }
        }
        
        class ShoppingCart {
            private PaymentStrategy paymentStrategy;
            
            public void setPaymentStrategy(PaymentStrategy strategy) {
                this.paymentStrategy = strategy;
            }
            
            public void checkout(double amount) {
                paymentStrategy.pay(amount);
            }
        }
        ```
        
    - Decorator 패턴 (구조적인 구성)
        
        ```java
        interface Coffee {
            double getCost();
            String getDescription();
        }
        
        class BasicCoffee implements Coffee {
            public double getCost() { return 5.0; }
            public String getDescription() { return "Basic coffee"; }
        }
        
        abstract class CoffeeDecorator implements Coffee {
            protected Coffee decoratedCoffee;
            
            public CoffeeDecorator(Coffee coffee) {
                this.decoratedCoffee = coffee;
            }
        }
        
        class MilkDecorator extends CoffeeDecorator {
            public MilkDecorator(Coffee coffee) {
                super(coffee);
            }
            
            public double getCost() {
                return decoratedCoffee.getCost() + 1.5;
            }
            
            public String getDescription() {
                return decoratedCoffee.getDescription() + ", Milk";
            }
        }
        
        // Usage: Compose features dynamically
        Coffee myCoffee = new MilkDecorator(
                            new SugarDecorator(
                                new BasicCoffee()));
        ```
        
    - 의존성 주입 (생성자 구성)
        
        ```java
        class DataProcessor {
            private final Repository repository;
            private final Validator validator;
            
            // Dependencies injected via constructor
            public DataProcessor(Repository repo, Validator validator) {
                this.repository = repo;
                this.validator = validator;
            }
            
            public void process(Data data) {
                if (validator.isValid(data)) {
                    repository.save(data);
                }
            }
        }
        ```
        
- 그럼에도 상속을 써야 하는 경우
    - 공유하는 정체성이 있는 실제 “is-a” 관계
        
        ```java
        // Good inheritance: Shape hierarchy
        abstract class Shape {
            abstract double area();
            abstract double perimeter();
        }
        
        class Circle extends Shape {  // Circle IS-A Shape
            private double radius;
            
            double area() {
                return Math.PI * radius * radius;
            }
        }
        ```
        
    - 위계 구조가 필요한 프레임워크 / API 디자인
        
        ```java
        // Java Collections Framework
        public abstract class AbstractList<E> implements List<E> {
            // Provides skeletal implementation
            // Subclasses fill in specifics
        }
        ```
        
    - Template Method 패턴
        
        ```java
        abstract class DataParser {
            // Template method with fixed algorithm
            public final void parse() {  // final to prevent overriding
                readData();
                processData();  // Abstract - subclass implements
                saveData();
            }
            
            protected abstract void processData();
        }
        ```
        
- Composition을 지원하는 Java의 특징
    - 인터페이스의 default 메서드
        
        ```java
        interface Logger {
            void log(String message);
            
            // Default implementation - like inheritance but composable
            default void logError(String error) {
                log("ERROR: " + error);
            }
        }
        
        // Multiple interfaces can be composed
        class FileLogger implements Logger, AutoCloseable {
            public void log(String message) { /* write to file */ }
            public void close() { /* close file */ }
        }
        ```
        
    - 데이터 구성을 위한 레코드
        
        ```java
        // Records automatically compose equals(), hashCode(), toString()
        record Point(int x, int y) {}
        
        record Line(Point start, Point end) {  // Composition
            double length() {
                return Math.hypot(end.x() - start.x(), 
                                 end.y() - start.y());
            }
        }
        ```

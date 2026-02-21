---
layout: single
title: 얕은 복사와 깊은 복사
---

## 개요

- 프로그래밍에서 데이터 구조를 복제하는 방식들이다.
- 다른 객체들에 대한 참조를 처리하는 방식에 차이가 있다.

## 얕은 복사(Shallow Copy)

- 새로운 객체를 생성하지만 중첩된 객체들은 참조값을 복사한다. (즉, 중복된 객체들 그 자체에 대한 복사본을 만들지 않음)
- 최고 레벨의 구조는 복사되지만 중첩된 객체들은 원본 객체와 복제본 객체 사이에 공유된다.
- 중첩된 객체들에 대한 변화가 원본 객체와 복제복 객체 모두에게 영향을 준다.
- 코드 예시
    
    ```java
    class Address {
        String city;
        String street;
        
        public Address(String city, String street) {
            this.city = city;
            this.street = street;
        }
        
        @Override
        public String toString() {
            return city + ", " + street;
        }
    }
    
    class Person implements Cloneable {
        String name;
        int age;
        Address address;  // Mutable reference
        
        public Person(String name, int age, Address address) {
            this.name = name;
            this.age = age;
            this.address = address;
        }
        
        // Shallow copy implementation
        @Override
        protected Object clone() throws CloneNotSupportedException {
            return super.clone();  // Default clone() creates shallow copy
        }
        
        @Override
        public String toString() {
            return name + " (" + age + ") - " + address;
        }
    }
    ```
    

## 깊은 복사(Deep Copy)

- 원본 객체와 독립적인 완전한 복사본을 생성한다.
- 모든 중첩된 객체들은 재귀적으로 복제된다.
- 공유하는 객체가 없어서 복제본에 가해지는 변경 사항들이 원본 객체에 영향을 주지 않는다.
- 코드 예시
    
    ```java
    class PersonDeep implements Cloneable {
        String name;
        int age;
        Address address;
        
        public PersonDeep(String name, int age, Address address) {
            this.name = name;
            this.age = age;
            this.address = address;
        }
        
        // Deep copy implementation
        @Override
        protected Object clone() throws CloneNotSupportedException {
            PersonDeep cloned = (PersonDeep) super.clone();
            // Create a new Address object with copied values
            cloned.address = new Address(this.address.city, this.address.street);
            return cloned;
        }
        
        @Override
        public String toString() {
            return name + " (" + age + ") - " + address;
        }
    }
    ```
    

### ➕Java에서 객체를 복사하는 실무적인 방법

- Object.clone() + Cloneable 인터페이스
    - 기본 동작 방식은 얕은 복사다.
    - Cloneable marker 인터페이스를 구현해야 한다.
    - clone()은 protected로 정의되어 있기 때문에 사용하기 위해서 public으로 오버라이드해야 한다.
    - 생성자를 호출하지 않고 새로운 객체를 생성함에 유의해야 한다.
    - 레거시로 간주되거나 망가진 기능으로 취급되기 때문에 사용하지 않는 게 좋다.
        - Effective Java의 저자이자 자바 플랫폼의 핵심적 디자이너인 Joshua Bloch는 “실패한 실험”이라 칭하기도 했다.
        - Cloneable 인터페이스는 마커 인터페이스로, 아무런 메서드가 없다.
            - 인터페이스는 “계약 사항을 정의하여 클래스가 구현하게 한다”는 이점이 있다. clone()이 Cloneable에 없어서 이러한 이점을 살릴 수 없다.
            - Object에 protected로 정의된 clone()을 개발자가 public으로 알아서 오버라이드 해야 한다. 번거롭다.
        - 기본적으로 얕은 복사 방식을 쓴다.
            - 객체가 List라도 담고 있으면, 그 List를 수정하다가 원본 객체에도 영향을 준다.
            - 깊은 복사를 하고 싶으면 모든 변경 가능한 필드들을 정밀하고 난잡한 코드를 짜서 수작업으로 복제해야 한다.
        - 생성자를 통과한다.
            - clone()은 어떠한 생성자도 사용하지 않고 객체를 생성한다.
            - 이는 “밀반입 통로”같은 역할을 하여 클래스의 비즈니스 규칙을 무시할 수 있다.
        - final 필드를 제대로 다루지 못한다.
            - 깊은 복사를 생성할 때 새로운 값을 필드에 할당해야만 한다.
            - 필드가 final로 설정되어 있으면 새로운 값을 필드에 할당할 수 없다.
            - 개발자가 final을 쓸 지, clone()을 쓸 지 선택해야 한다. 대개는 final이 더 좋은 선택지다.
    
    ```java
    class Person implements Cloneable {
        String name;
        Address address; // Reference type
        
        @Override
        protected Object clone() throws CloneNotSupportedException {
            return super.clone(); // Default: shallow copy
        }
    }
    ```
    
- 복사 생성자 / 팩토리 메서드
    - clone()과 Cloneable의 대안책이다.
    - 컴파일 타임 안전성을 보장하며, 오버라이딩 없이 공적으로 접근 가능하다.
    
    ```java
    class Person {
        private String name;
        private Address address;
        
        // Copy constructor
        public Person(Person other) {
            this.name = other.name;
            this.address = new Address(other.address); // Deep copy
        }
        
        // Static factory method alternative
        public static Person newInstance(Person other) {
            Person p = new Person();
            p.name = other.name;
            p.address = new Address(other.address);
            return p;
        }
    }
    
    class Address {
        private String city;
        
        // Also needs copy constructor for deep copy
        public Address(Address other) {
            this.city = other.city;
        }
    }
    ```

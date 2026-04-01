---
layout: single
title: equals()와 hashcode()를 함께 오버라이딩해야 하는 이유
---
## JLS(Java Language Specification)에 따른 이유

- 두 객체들이 equals()에 따라 동등하다면 그 객체들은 반드시 동일한 hashcode()를 지녀야 한다.
    - 그래서 equals()와 hashCode()를 모두 오버라이딩 해야 한다.
- 두 객체들이 동일한 hashcode()를 지녔다고 해서 반드시 그 객체들이 동일해야 하는 것은 아니다. (hash collision 때문)

## equals()와 hashCode()를 동시에 오버라이딩하지 않으면 발생하는 일

- equals()만 오버라이딩 할 경우: 객체들이 동등하기는 하지만 해시 값은 달라서 HashMap, HashSet같은 자료형을 의도대로 사용할 수 없다.
- hashcode()만 오버라이딩 할 경우: equals()가 객체들의 메모리 주소를 비교하기 때문에 실무에 맞지 않은 방식으로 동등성을 비교할 수 있다.
    - equals()는 오버라이딩되지 않으면 객체들의 메모리 주소를 비교한다. equals()의 기본 동작이다. (==의 작동 방식과 동일)

➕ 두 메서드들을 오버라이딩하는, 권장되는 방식

- equals()와 hashCode()에 동일한 필드를 사용하여 결과의 일관성을 유지한다.
- final로 설정된 필드들을 사용한다.
    - 필드의 값이 바뀌면 동등성도 달라질 수 있고 해시 값도 달라질 수 있다.
    - 특히 Hash와 연관된 자료형(HaspMap 등)을 사용할 때 치명적이다.
- 객체의 신원을 나타내거나 주요한 의미를 지니는 필드를 사용한다.
    - equals()가 객체들의 논리적인 동등성을 따지기 때문이다.
    - ✅ Account 객체들의 email, password 값이 동일하니까 동일한 객체들이야.
    - ❌ Account 객체들의 updatedAt, isActive 값이 동일하니까 동일한 객체들이야.

## ➕ Hash Collision

- 질문 배경
    - 왜 equals()로 동등하면 객체들의 해시 값이 동일해야 하는데, 해시 값이 동일하다고 equals()로 동등하리라는 보장은 없음?
- 서로 다른 내용물을 지닌 객체들의 해시 값이 동일한 경우이다.
- 발생 이유
    - 유한한 결과값 공간: 해시 코드는 자바에서 32비트의 정수다. (약 40억 개의 값이 가용함)
    - 무한한 입력값 공간: 객체의 상태로 가능한 경우는 무한에 가깝다. (객체의 필드를 구성하는 경우의 수 + 해시를 생성할 때 쓰는 필드의 경우의 수)
    - 비둘기장 칸(pigeonhold) 원칙: 해시 값보다 많은 수의 객체가 있다면 해시 값의 충돌은 불가피하다.
- 가능한 해시 값이 40억개나 된다면서, 객체들의 해시 값이 겹칠 수 있다고?
    - 40억이 직관적으로는 많아 보이지만, 사실은 그렇게 많지 않다.
    - 23명의 사람만 있어도 그 중 두 명이 생일을 공유할 확률이 50%나 된다는 걸 생각해보자. (생일의 역설)
    - 생일의 역설에 따르면 객체 77,000개 정도만 있어도 두 객체가 해시 값을 공유할 확률이 50% 정도가 된다.
- 해시 충돌을 해소하는 방법
    - HashMap의 내부적인 구조
        
        ```java
        // Simplified HashMap bucket structure
        HashMap<K, V> {
            Node<K, V>[] table;  // Array of buckets
            
            // Each bucket can contain multiple entries (linked list or tree)
            class Node<K, V> {
                final int hash;
                final K key;
                V value;
                Node<K, V> next;  // For collisions
            }
        }
        ```
        
    - Chaining
        
        ```java
        // What happens during a collision
        HashMap<String, String> map = new HashMap<>();
        
        map.put("Aa", "Value1");  // Hash 2112 -> Bucket 0
        map.put("BB", "Value2");  // Hash 2112 -> Same bucket 0 (collision!)
        
        // Internally, bucket 0 becomes a linked list:
        // ["Aa" -> "Value1"] -> ["BB" -> "Value2"]
        ```
        
- 해시 충돌이 성능에 미치는 영향
    - Java 8버전 이후 부터는 Java가 linked list를 red-black tree로 변환하여 최악의 경우에 발생하는 성능을 O(n)에서 O(log n)으로 개선했다.
    - 좋은 해시 함수 (충돌이 덜 남)
        
        ```java
        Bucket 0: [Entry1]
        Bucket 1: [Entry2]
        Bucket 2: [Entry3]
        ...
        // O(1) operations
        ```
        
    - 나쁜 해시 함수 (충돌이 자주 남)
        
        ```java
        Bucket 0: [Entry1] -> [Entry2] -> [Entry3] -> [Entry4] -> ...
        Bucket 1: [Entry5]
        Bucket 2: []
        ...
        // Degrades to O(n) operations in worst case
        ```
        
- 핵심 요약
    - 해시 기반 컬렉션에서 충돌은 일반적이고 예측 가능한 현상이다.
    - 서로 다른 객체들의 해시가 동일할 수 있고, 동일할 것이다.
    - 컬렉션은 chaining을 사용하여 충돌을 내부적으로 해소한다. (linked list, tree)
    - 동등한 객체들은 해시 코드 값이 같지만, 해시 코드 값이 같다고 해서 동등한 객체라는 의미는 아니다.
    - 해시 함수를 잘못 짜면 성능이 나빠진다. 좋은 해시 함수는 성능이 O(1)과 비슷하다.

```java
String a = "Aa";
String b = "BB";

System.out.println(a.hashCode());  // 2112
System.out.println(b.hashCode());  // 2112 - Same hash code!
System.out.println(a.equals(b));   // false - Different objects
```

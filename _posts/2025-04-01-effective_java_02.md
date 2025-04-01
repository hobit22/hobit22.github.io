---
title: 이펙티브 자바 02
layout: post
tag: [java]
toc: true
---

자바를 잘 다루기 위해 [이펙티브 자바](https://product.kyobobook.co.kr/detail/S000001033066)를 읽고 정리한 내용입니다. :)

## 3장 모든 객체의 공통 메서드

Java의 모든 객체는 `Object` 클래스를 상속한다. 따라서 `equals`, `hashCode`, `toString`, `clone`, `finalize` 등의 메서드를 상속받게 되며, 이 중 대부분은 상황에 맞게 재정의가 필요하다.  
이 장에서는 언제, 어떻게 재정의해야 하는지를 다룬다.

---

### 10. equals는 일반 규약을 지켜 재정의하라

`equals` 메서드는 객체의 **논리적 동치성**을 정의할 때 재정의한다. 재정의할 때는 반드시 다음의 규약을 지켜야 한다:

1. **반사성**: `x.equals(x)는 true 이다`
2. **대칭성**: `x.equals(y)가 true 라면` → `y.equals(x)는 true이다`
3. **추이성**: `x.equals(y)가 true` && `y.equals(z)가 true` → `x.equals(z)는 true이다`
4. **일관성**: 두 객체가 변하지 않는 한 `x.equals(y)`의 결과는 항상 같아야 한다.
5. **null-비허용**: `x.equals(null)는 항상 false 이다.`

#### equals 구현 팁

- `==` 연산자로 자기 자신인지 확인
- `instanceof` 연산자로 타입 확인
- 적절한 타입으로 캐스팅
- 핵심 필드들을 비교하여 동등성 판단

> **IDE가 생성해주는 `equals`는 꽤 잘 만들어진다. 특별한 이유가 없다면 수정하지 말자.**

---

### 11. equals를 재정의하려거든 hashCode도 재정의하라

Java의 `Object` 명세에 따르면, 두 객체가 `equals`로 같다고 판단되면 `hashCode`도 같아야 한다.  
이를 지키지 않으면 `HashMap`, `HashSet` 같은 컬렉션에서 오류가 발생할 수 있다.

> `equals`를 재정의했다면 **항상** `hashCode`도 재정의해야 한다.

#### 예시 (Kotlin)

```kotlin
data class Position(
    val x: Double,
    val y: Double
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false

        other as Position

        if (x != other.x) return false
        if (y != other.y) return false

        return true
    }

    override fun hashCode(): Int {
        var result = x.hashCode()
        result = 31 * result + y.hashCode()
        return result
    }
}
```

> Kotlin의 data class, Java 14의 record는 equals, hashCode, toString을 자동 생성해 준다.

---

### 12. toString을 항상 재정의하라

toString은 객체의 상태를 문자열로 표현할 수 있게 해준다. 디버깅이나 로깅에서 유용하며, 다음과 같은 차이가 있다:

- 기본값: `com.example.Position@fe100000`  
- 재정의: `Position(x=1.0, y=2.0)` 
- 연관된 객체가 많거나 순환참조가 있을 경우 StackOverflow가 날 수 있으니 주의
- Kotlin의 data class, Java의 record는 toString도 자동으로 제공

--- 

### 13. clone 재정의는 주의해서 진행하라

Object.clone()은 얕은 복사를 수행한다. 복잡한 객체에서는 깊은 복사가 필요할 수 있다.

Cloneable 인터페이스를 구현해야 clone()이 정상 작동함   
복잡한 경우, 복사 생성자나 팩토리 메서드 방식이 더 안전함   
- clone보다 명시적인 복사 메서드를 만드는 것이 더 좋은 선택인 경우가 많다.

--- 

### 14. Comparable을 구현할지 고민하라

정렬이 필요한 객체라면 `Comparable<T>` 인터페이스를 구현해 `compareTo()`를 정의한다.

```java
public class Person implements Comparable<Person> {
    private String name;
    private int age;

    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age); // 나이 기준 정렬
    }
}
```

```kotlin
data class Person(
    val name: String,
    val age: Int
) : Comparable<Person> {
    override fun compareTo(other: Person): Int {
        return this.age.compareTo(other.age)
    }
}
```
Java 8 이상에서는 Comparator.comparing() 등을 활용해 유연하게 정렬 기준을 정의할 수도 있다.

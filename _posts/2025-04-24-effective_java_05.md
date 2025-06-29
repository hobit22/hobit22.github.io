---
title: 이펙티브 자바 06
layout: post
tag: [java]
toc: true
---

자바를 잘 다루기 위해 [이펙티브 자바](https://product.kyobobook.co.kr/detail/S000001033066)를 읽고 정리한 내용입니다. 😊

## 6장. 열거 타입과 애너테이션

이 장에서는 자바에서 `int` 상수나 명명 규칙보다 더 안전하고 강력한 대안인 **열거 타입(enum)**과, 코드에 메타데이터를 부착할 수 있는 **애너테이션(annotation)**에 대해 다룬다.

---

### 열거 타입(Enum)이란?

`enum`은 서로 관련된 상수들을 하나의 타입으로 묶어 표현할 수 있게 해주는 특수한 클래스다.  
단순한 `int` 상수보다 **타입 안전성**, **코드 명확성**, **기능 확장성**이 뛰어나다.

```java
public enum OrderStatus {
    PENDING, SHIPPED, DELIVERED, CANCELLED
}
```

---

### 애너테이션(Annodation)이란?

애너테이션은 코드에 의미 있는 **메타데이터를 추가**할 수 있게 해준다.  
컴파일러, 프레임워크, IDE 등이 애너테이션을 활용하여 **자동 검사, 코드 생략, 문서화** 등을 수행할 수 있다.

```java
@Override
public String toString() {
    return "Hello";
}
```

---

### 34. int 상수 대신 열거 타입을 사용하라

- `int` 상수는 타입이 섞일 수 있어 안전하지 않다.  
- `enum`을 사용하면 컴파일 타임에 오류를 잡을 수 있다.
- 상태값이나 역할(Role)을 다룰 때 enum을 도입하면 조건문도 훨씬 명확해진다.

```java
public enum Direction {
    NORTH, SOUTH, EAST, WEST
}
```

---

### 35. ordinal 메서드 대신 인스턴스 필드를 사용하라

- `ordinal()`은 선언 순서에 의존해 깨지기 쉽다.  
- 명시적인 필드를 사용해서 의미 있는 값을 부여하자.
- DB 저장 시 `ordinal()`을 쓰면 enum 순서가 바뀌는 순간 망한다. `name()` 또는 직접 필드를 매핑하자.

```java
public enum Grade {
    BASIC(1), VIP(2);

    private final int value;
    Grade(int value) { this.value = value; }
    public int getValue() { return value; }
}
```


---

### 36. 비트 필드 대신 EnumSet을 사용하라

- `EnumSet`은 내부적으로 비트 연산 기반이라 빠르면서도 안전하다.
- 권한, 옵션 등 **다중 선택 가능한 enum**에서 활용하면 깔끔하고 효율적이다.

```java
public enum Permission { READ, WRITE, EXECUTE }

EnumSet<Permission> perms = EnumSet.of(Permission.READ, Permission.WRITE);
```


---

### 37. ordinal 인덱싱 대신 EnumMap을 사용하라

- 배열 인덱스로 enum을 쓰는 것보다 `EnumMap`이 더 명확하고 안전하다.
- 통계, 설정, 매핑 테이블 등에 `EnumMap` 쓰면 성능과 가독성 둘 다 잡는다.

```java
Map<DayOfWeek, String> dayNames = new EnumMap<>(DayOfWeek.class);
dayNames.put(DayOfWeek.MONDAY, "월요일");
```


---

### 38. 확장할 수 있는 열거타입이 필요하면 인터페이스를 사용하라

- enum은 상속 불가 → 여러 타입에서 동작 통일하려면 인터페이스를 활용하자.
- 전략 패턴에서 인터페이스 + enum 조합을 잘 쓰면 코드 관리가 쉬워진다.

```java
public interface DiscountPolicy {
    int discount(int amount);
}

public enum BasicPolicy implements DiscountPolicy {
    FIXED { public int discount(int amount) { return 1000; } },
    RATE { public int discount(int amount) { return amount / 10; } }
}
```


---


### 39. 명명 패턴보다 애너테이션을 사용하라

- 애너테이션은 명시적이고 안정적이며, 도구와의 연동도 좋다.
- JUnit, Spring, Lombok 등 거의 모든 프레임워크가 애너테이션 기반으로 움직인다.

```java
@Test
public void shouldReturnTrueWhenInputIsValid() {
    ...
}
```

---

### 40. @Override 애너테이션을 일관되게 사용하라

- 오버라이딩 시 실수를 막고, IDE도 도움을 줄 수 있다.  
- 인터페이스 구현에도 붙이는 게 좋다(Java 6 이상).
- 빠진 오버라이드 때문에 아무 일도 안 하는 버그가 종종 생긴다. 무조건 붙이자!

```java
@Override
public void run() {
    ...
}
```

---

### 41. 정의하려는 것이 타입이라면 마커 인터페이스를 사용하라

- `interface`는 타입 체크가 가능하고 문서화도 쉽다.  
- `@Marker` 애너테이션은 런타임만 확인 가능하다.
- 단순한 구분이지만, `instanceof`로 분기처리할 수 있다는 점에서 활용도가 높다.

```java
public interface FastSerializable extends Serializable {}
```


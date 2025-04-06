---
title: 이펙티브 자바 03
layout: post
tag: [java]
toc: true
---

자바를 잘 다루기 위해 [이펙티브 자바](https://product.kyobobook.co.kr/detail/S000001033066)를 읽고 정리한 내용입니다. :)

## 4장. 클래스와 인터페이스

클래스와 인터페이스는 객체지향 프로그래밍의 중심이다. 이 장에서는 **클래스와 인터페이스의 설계, 구현, 접근 제어, 상속 및 컴포지션 활용**에 대해 다룬다.

---

### 15. 클래스와 멤버의 접근 권한을 최소화하라

- 정보 은닉(캡슐화)은 모듈성과 유지보수성을 높인다.
- 가능한 한 **모든 필드와 메서드는 private**로 선언하고, 꼭 필요한 경우에만 좁은 범위로 공개하자.
- 패키지 전용(default)보다는 **protected → public 순서로 신중하게 공개 범위 확장**할 것.

> 내부 구현을 외부에서 의존하지 않도록 방지하는 게 핵심이다.    


```java
// BAD
public class Point {
    public double x;
    public double y;
}

// GOOD
public class Point {
    private double x;
    private double y;

    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }

    public double getX() { return x; }
    public double getY() { return y; }
}

```

---

### 16. public 클래스에서 public 필드가 아닌 접근자 메서드를 사용하라

- **불변 클래스라도 public 필드는 지양**하자.
- 필드에 직접 접근하게 하면 캡슐화가 깨진다.
- 대신 **getter 메서드**로 값을 노출하자.

```java
// Bad
public class Point {
    public double x;
    public double y;
}

// Good
public class Point {
    private double x, y;
    public double getX() { return x; }
    public double getY() { return y; }
}
```

```kotlin
// BAD
class Point {
    var x: Double = 0.0
    var y: Double = 0.0
}

// GOOD
data class Point(
    val x: Double,
    val y: Double
)
``` 

> Kotlin에서는 data class를 사용하면 자동으로 getter가 생기고, 불변 필드도 만들 수 있어서 간편하다. 

---

### 17. 변경 가능성을 최소화하라

- **불변 객체**는 동시성 문제에서 안전하고, 예측하기 쉽다.
- 가능하면 모든 필드를 `final`로 선언하고, 객체 자체도 `final`로 만들어라.
- 가변 상태가 필요하다면 변경 범위를 최소화하고 명확히 하자.

> 클래스를 만들 땐 항상 "불변으로 만들 수 있을까?"를 먼저 고민하자.

```java
// BAD
public class Config {
    public String environment;

    public void setEnvironment(String env) {
        this.environment = env;
    }
}

// GOOD
public final class Config {
    private final String environment;

    public Config(String environment) {
        this.environment = environment;
    }

    public String getEnvironment() {
        return environment;
    }
}

```
```kotlin
// BAD
class Config {
    var environment: String = ""
}

// GOOD
data class Config(
    val environment: String
)

``` 

> Kotlin의 val과 data class 조합은 기본적으로 불변 객체를 만드는 훌륭한 도구다. 

---

### 18. 상속보다는 컴포지션을 사용하라
- 상속 : 하위 클래스가 상위 클래스의 특성을 재정의 한것 > (IS-A) 관계
- 컴포지션 : 기존 클래스가 새로운 클래스의 구성요소가 되는것 > (HAS-A) 관계
- 상속은 내부 구현에 강하게 결합되므로 깨지기 쉽다.
- **재사용이나 확장**이 목적이라면, 기존 클래스를 필드로 갖고 위임(delegate)하는 **컴포지션**을 우선 고려하자.  

```java
// BAD
public class MyList<E> extends ArrayList<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    public int getAddCount() {
        return addCount;
    }
}
// 문제점: ArrayList의 다른 addAll() 같은 메서드는 오버라이드되지 않아서 addCount가 부정확할 수 있음.

// GOOD
public class MyList<E> {
    private final List<E> list = new ArrayList<>();
    private int addCount = 0;

    public boolean add(E e) {
        addCount++;
        return list.add(e);
    }

    public int getAddCount() {
        return addCount;
    }
}
```

```kotlin
// BAD
class MyList<E> : ArrayList<E>() {
    var addCount = 0

    override fun add(element: E): Boolean {
        addCount++
        return super.add(element)
    }
}
// GOOD
class MyList<E> {
    private val list = mutableListOf<E>()
    var addCount = 0
        private set

    fun add(element: E): Boolean {
        addCount++
        return list.add(element)
    }

    fun getAll(): List<E> = list
}
```

> 컴포지션은 외부 구현 변경에 영향을 덜 받는다. 위임을 통해 필요한 기능만 제한적으로 노출할 수 있다.


---

### 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라

- 클래스가 상속될 것을 염두에 뒀다면 **protected 필드/메서드**, **hook 메서드** 등을 명시적으로 제공해야 한다.
- 그렇지 않다면 `final`로 선언하거나 생성자를 `private`으로 만들어 하위 클래스 생성을 막아야 한다.

---

### 20. 추상 클래스보다는 인터페이스를 사용하라

- 인터페이스는 다중 구현이 가능하고 유연성이 더 크다.
- Java 8부터는 `default` 메서드도 지원되므로, 일부 구현도 제공할 수 있다.

---

### 21. 인터페이스는 구현하는 쪽을 생각해 설계하라

- 메서드는 최소한만 정의하고, 꼭 필요한 것만 포함시키자.
- 지나치게 구체적인 설계는 다양한 구현을 방해한다.

---

### 22. 인터페이스는 타입을 정의하는 용도로만 사용하라

- 인터페이스는 **계약** 또는 **타입**을 정의하는 데 집중해야 한다.
- 상수를 모아 놓는 용도로 사용하는 "constant interface" 패턴은 지양하자. → `public static final` 상수는 별도 `Constants` 클래스로 분리하자.

---

### 23. 태그 달린 클래스보다는 클래스 계층 구조를 활용하라

- `type` 필드를 통해 동작을 분기하는 태그 달린 클래스보다, **클래스 계층 구조 + 다형성**을 활용하는 것이 더 깔끔하고 유지보수에 좋다.
- 태그 달린 클래스를 써야 하는 상황은 거의 없다. 계층구조를 사용하자.

```java
// BAD
class Figure {
    enum Shape { RECTANGLE, CIRCLE };

    // 태그 필드 - 현재 모양을 나타낸다.
    final Shape shape;

    // 다음 필드들은 모양이 사각형(RECTANGLE)일 때만 쓰인다.
    double length;
    double width;

    // 다음 필드는 모양이 원(CIRCLE)일 때만 쓰인다.
    double radius;

    // 원용 생성자
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }

    // 사각형용 생성자
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }

    double area() {
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}

// GOOD
abstract class Figure {
    abstract double area();
}
class Rectangle extends Figure {
    final double length;
    final double width;

    Rectangle(double length, double width) {
        this.length = length;
        this.width  = width;
    }
    @Override double area() { return length * width; }
}

class Circle extends Figure {
    final double radius;

    Circle(double radius) { this.radius = radius; }

    @Override double area() { return Math.PI * (radius * radius); }
}

```

---

### 24. 멤버 클래스는 되도록 static으로 만들라

- 내부 클래스 중 **외부 클래스 인스턴스에 의존하지 않는 경우**, static으로 선언하는 것이 좋다.
- 그렇지 않으면 불필요한 참조가 생겨 **메모리 누수**나 **성능 저하**를 유발할 수 있다.

---

### 25. 톱레벨 클래스는 한 파일에 하나만 담으라

- 파일명과 클래스명이 일치하는 것이 기본 규칙이며, 가독성과 유지보수를 위해서도 이 원칙을 지키자.

---

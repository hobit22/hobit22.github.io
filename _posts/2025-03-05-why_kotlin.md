---
title: 왜 코틀린일까?
layout: post
tag: [java, kotlin]
toc: true
---

# 왜 코틀린일까?

이직을 준비하며 여러 회사를 살펴보니, 많은 곳에서 **코틀린 + 스프링(코프링)** 조합을 사용하고 있었다. 이를 계기로 코틀린에 대한 필요성을 느끼고 본격적으로 공부를 시작하게 되었다.

## 1. 코틀린의 등장 배경
프로그래밍 언어는 시대의 흐름에 따라 발전해 왔다. 자바는 오랫동안 백엔드와 안드로이드 개발의 대표적인 언어로 자리 잡았지만, 반복적인 보일러플레이트 코드, NPE(NullPointerException) 문제, 복잡한 비동기 처리 방식 등 여러 가지 단점이 존재했다. 이를 해결하기 위해 JetBrains는 **개발자의 생산성을 높이고, 안정성을 강화한 새로운 언어**로 코틀린을 개발했다.

## 2. 코틀린이 인기 있는 이유
코틀린은 2017년 구글이 안드로이드 공식 개발 언어로 채택하면서 빠르게 성장했다. 하지만 안드로이드뿐만 아니라 **백엔드(Spring Boot), 멀티플랫폼(Kotlin Multiplatform), 데이터 과학** 등 다양한 분야에서도 점점 사용이 늘어나고 있다.

특히, **코틀린을 선택하는 개발자와 기업이 많아지는 이유**는 단순히 새로운 언어이기 때문이 아니라, 다음과 같은 강력한 장점들 덕분이다.

## 3. 코틀린의 강력한 장점
### 1) 간결한 문법으로 생산성 증가
코틀린은 자바 대비 30~40% 더 적은 코드량으로 같은 기능을 구현할 수 있다.

#### 자바 코드 예시
```java
public class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
}
```

#### 코틀린 코드 예시
```kotlin
data class Person(val name: String, val age: Int)
```
**클래스 선언만으로 `getter`, `setter`, `toString`, `equals` 등의 기능을 자동으로 제공**하여 불필요한 코드를 줄일 수 있다.

### 2) 널 안정성 (Null Safety)
자바에서는 `NullPointerException`(NPE)이 자주 발생하는데, 코틀린은 이를 원천적으로 방지할 수 있는 **널 안정성**을 제공한다.

```kotlin
var name: String? = null // ?를 붙이면 null 값을 가질 수 있음
println(name?.length) // 안전 호출
```

이렇게 `?.`(안전 호출) 또는 `!!`(널 강제 처리)를 통해 개발자가 의도적으로 널 처리를 할 수 있어 런타임 오류를 줄일 수 있다.

### 3) 코루틴을 활용한 비동기 처리
코틀린은 기존의 `Thread`나 `ExecutorService` 대신 **코루틴(Coroutine)** 을 이용해 간단하고 효율적으로 비동기 처리를 할 수 있다.

```kotlin
suspend fun fetchData() {
    delay(1000) // 1초 대기
    println("데이터 로딩 완료")
}
```

코루틴을 사용하면 기존 콜백 방식보다 **코드 가독성이 좋아지고, 성능도 최적화**된다.

### 4) 확장 함수(Extension Function) 지원
기존 클래스를 수정하지 않고도 **새로운 기능을 추가할 수 있는 확장 함수**를 지원한다.

```kotlin
fun String.isEmailValid(): Boolean {
    return this.contains("@")
}

println("test@example.com".isEmailValid()) // true
```

기본 라이브러리 함수처럼 사용할 수 있어 유지보수가 용이하다.

### 5) 함수형 프로그래밍 지원
코틀린은 **함수형 프로그래밍(FP)**을 지원하여 `map`, `filter`, `reduce` 등의 고차 함수를 활용할 수 있다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
val evenNumbers = numbers.filter { it % 2 == 0 }
println(evenNumbers) // [2, 4]
```

함수형 스타일을 사용하면 **더 읽기 쉬운 코드**를 작성할 수 있다.

### 6) 멀티플랫폼 지원 (Kotlin Multiplatform, Kotlin/Native)
코틀린은 **하나의 코드로 안드로이드, iOS, 웹, 백엔드를 동시에 개발할 수 있는 멀티플랫폼 지원**이 가능하다. 이를 통해 **중복 코드를 줄이고, 유지보수를 편하게 할 수 있다.**

```kotlin
expect fun platformName(): String
```
이렇게 **플랫폼별로 다르게 구현할 수 있도록 `expect/actual` 키워드를 제공**하여 멀티플랫폼 개발을 쉽게 만든다.

## 4. 결론: 코틀린이 인기있는 이유
코틀린은 **생산성, 안정성, 유지보수성, 멀티플랫폼 지원** 등 개발자들에게 필요한 다양한 이점을 제공한다.

자바 개발자라도 쉽게 배우고 적용할 수 있으며, 점점 더 많은 기업과 프로젝트에서 코틀린을 선택하고 있다.

**"왜 코틀린일까?"**라는 질문에 대한 답은 명확하다.

✅ **더 적은 코드로 더 많은 기능을 구현할 수 있기 때문**  
✅ **NPE(NullPointerException)를 줄여 안정적인 코드를 만들 수 있기 때문**  
✅ **코루틴과 확장 함수로 가독성과 유지보수성을 높일 수 있기 때문**  
✅ **멀티플랫폼을 지원하여 다양한 환경에서 활용할 수 있기 때문**




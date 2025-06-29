---
title: 이펙티브 자바 04
layout: post
tag: [java]
toc: true
---

자바를 잘 다루기 위해 [이펙티브 자바](https://product.kyobobook.co.kr/detail/S000001033066)를 읽고 정리한 내용입니다. :)

## 5장. 제네릭  
제네릭(Generics)은 타입을 클래스나 메서드 정의 시점이 아닌, 사용하는 시점에 지정할 수 있게 해주는 자바의 기능이다.
이 덕분에 컴파일 타임에 타입 안전성을 확보하면서도, 중복 없이 재사용 가능한 코드를 만들 수 있다.

---

### 26. 로 타입은 사용하지 마라.    
- 제네릭이 도입되기 전에는 List, Map처럼 타입 매개변수가 없는 로(raw) 타입을 사용했지만, 지금은 타입 안전성 확보를 위해 피해야 한다.
- 로 타입을 사용하면 컴파일러가 타입 체크를 해주지 않아 런타임 오류 가능성이 커진다.

```java
// 로 타입 - 비추천
List list = new ArrayList();
list.add("문자열");
list.add(123); // 컴파일 오류 없음 → 런타임 오류 가능

// 제네릭 타입 - 추천
List<String> list = new ArrayList<>();
list.add("문자열");
// list.add(123); // 컴파일 에러 발생
```

---

### 27. 비검사 경고를 제거하라.
- 컴파일 시 발생하는 unchecked 경고는 가능하면 제거하자.
- 제거가 불가능한 경우, @SuppressWarnings("unchecked")를 최소 범위에만 적용해야 한다.

```java
@SuppressWarnings("unchecked")
public <T> T[] toArray(T[] a) {
    if (a.length < size) {
        // 배열 생성 부분에만 적용
        return (T[]) Arrays.copyOf(elements, size, a.getClass());
    }
    System.arraycopy(elements, 0, a, 0, size);
    if (a.length > size)
        a[size] = null;
    return a;
}
```
---

### 28. 배열보다는 리스트를 사용하라.
- 배열은 공변(예: Object[]에 String[]이 들어갈 수 있음)이고, 제네릭은 불공변(예: List<Object>에 List<String> 못 넣음).
- 배열은 런타임에 타입 오류가 발생할 수 있고, 제네릭은 컴파일 타임에 잡아준다.
- 따라서 타입 안정성과 유연성 측면에서 List<E>를 사용하는 것이 좋다.

```java
// 배열 사용 시 문제
Object[] array = new String[1];
array[0] = 42; // ArrayStoreException 발생

// 제네릭 사용 시 컴파일 에러
List<String> list = new ArrayList<>();
list.add(42); // 컴파일 에러
```

---

### 29. 이왕이면 제네릭 타입으로 만들어라.
- 클래스나 인터페이스를 설계할 때 타입 매개변수를 사용하면 재사용성과 타입 안정성이 높아진다.

```java
// 제네릭 X
public class StringStack {
    private Stack<String> stack = new Stack<>();
}

// 제네릭 O
public class Stack<E> {
    private E[] elements;
    public void push(E e) { ... }
    public E pop() { ... }
}
```

---

### 30. 이왕이면 제네릭 메서드로 만들어라.
- 클래스 전체가 제네릭일 필요는 없다. 메서드 하나만 제네릭으로 만들 수 있다.
- 유틸리티 메서드에 특히 유용하다.

```java
// 제네릭 메서드 예시
public static <T> T pickRandom(List<T> list) {
    return list.get(new Random().nextInt(list.size()));
}
```
---

### 31. 한정적 와일드카드를 사용해 API 유연성을 높이라.

- `<? extends T>`: T나 T의 하위 타입 → 읽기 전용
- `<? super T>`: T나 T의 상위 타입 → 쓰기 전용
- PECS(Producer Extends, Consumer Super) 원칙을 기억하자!

```java
// 읽기 전용
public void printList(List<? extends Number> list) {
    for (Number n : list) {
        System.out.println(n);
    }
}

// 쓰기 전용
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}
```
---

### 32. 제네릭과 가변인수를 함께 쓸 때는 신중하라.

- 가변인수(varargs) 배열은 실체화 불가 → 힙 오염 가능
- `@SafeVarargs` 사용 시, 메서드는 변경 불가능하고 타입 안정해야 함

```java
@SafeVarargs
public static <T> List<T> asList(T... elements) {
    return Arrays.asList(elements);
}
```
---

### 33. 타입 안전 이종 컨테이너를 고려하라.

- Class 객체를 키로 사용해 다양한 타입의 값을 안전하게 저장할 수 있다.
- `Class<T>`와 `cast()` 활용

```java
public class Favorites {
    private Map<Class<?>, Object> favorites = new HashMap<>();

    public <T> void putFavorite(Class<T> type, T instance) {
        favorites.put(type, type.cast(instance));
    }

    public <T> T getFavorite(Class<T> type) {
        return type.cast(favorites.get(type));
    }
}
```

---

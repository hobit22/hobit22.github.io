---
title: 이펙티브 자바 01
layout: post
tag: [java]
toc: true
---

자바를 잘 다루기 위해 [이펙티브 자바](https://product.kyobobook.co.kr/detail/S000001033066)를 읽고 정리한 내용입니다. :)

## 2장. 객체 생성과 파괴

### 1. 생성자 대신 정적 팩토리 메서드를 사용하라
####  정적 팩토리 메서드의 장점
1. 이름을 가질 수 있다.
2. 호출될 때마다 인스턴스를 새로 생성하지 않아도 된다.
3. 반환 타입의 하위 타입 객체를 반환할 수 있는 능력이 있다.
4. 입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다.
5. 정적 팩토리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다.      

#### 정적 팩토리 메서드의 단점
1. 상속을 하려면 public이나 protected 생성자가 필요하니 정적 패토리 메서드만 제공하면 하위 클래스를 만들 수 없다.
2. 정적 팩토리 메서드는 프로그래머가 찾기 어렵다.

#### 네이밍 예시
- from : 매개변수를 하나 받아서 해당 타입의 인스턴스를 반환하는 형변환 메서드        
    ```java
    Date d = Date.from(instance);
    ```
    
- of: 여러 매개변수를 받아 적합한 타입의 인스턴스를 반환하는 집계 메서드        
    ```java
    Set<Rank> faceCards = EnumSet.of(JACK, QUEEN, KING);
    ```
    
- valueOf : from 과 of 의 더 자세한 버전        
    ```java
    BigInteger prime = BigInteger.valueOf(Integer.MAX_VALUE);
    ```
    
- instance 혹은 getInstance : 매개변수로 명시한 인스턴스를 반환하지만, 같은 인스턴스임을 보장하진 않는다.        
    ```java
    StackWalker luke = StackWalker.getInstance(options);
    ```
    
- create 혹은 newInstance : instance 혹은 getInstance와 같지만, 매번 새로운 인스턴스를 생성해 반환함을 보장한다.        
    ```java
    Object newArray = Array.newInstance(classObject, arreyLen);
    ```
    
- getType : getInstance 와 같으나, 생성할 클래스가 아닌 다른 클래스에 팩토리 메서드를 정의할 때 쓴다.        
    ```java
    FileStore fs = Files.getFileStore(path)
    ```
    
- newType : new Instance와 같으나, 생성할 클래스가 아닌 다른 클래ㅡ에 팩토리 메서드를 정의할 때 쓴다.        
    ```java
    BufferedReader br = Files.newBufferedReader(path);
    ```
    
- type : getType과 newType의 간결한 버전
    ```java
    List<Complaint> litany = Collections.list(legacyLitany);
    ```


### 2. 생성자에 매개변수가 많다면 빌더를 고려하라
#### 점층적 생성자 패턴
- 매개변수가 많아질수록 생성자 오버로딩이 많아져서 유지보수하기 어려움.
- 매개변수의 순서를 기억해야 함

```java
public class Pizza {
    private final String dough;
    private final String sauce;
    private final String cheese;
    private final String topping;

    // 점층적 생성자 패턴 (생성자 오버로딩)
    public Pizza(String dough) {
        this(dough, "Tomato", "Mozzarella", "None");
    }

    public Pizza(String dough, String sauce) {
        this(dough, sauce, "Mozzarella", "None");
    }

    public Pizza(String dough, String sauce, String cheese) {
        this(dough, sauce, cheese, "None");
    }

    public Pizza(String dough, String sauce, String cheese, String topping) {
        this.dough = dough;
        this.sauce = sauce;
        this.cheese = cheese;
        this.topping = topping;
    }
}
```

```java
Pizza pizza = new Pizza("Thin Crust", "Tomato", "Cheddar", "Pepperoni");
```

#### 자바빈즈 패턴
- 객체를 완전히 만들기 전에 불완전한 상태로 존재할 수 있음.(`setDough()`만 호출하고 `setSauce()`를 안 할 수도 있음)
- 객체가 불변(immutable)하지 않음 → 멀티스레드 환경에서 동기화 문제가 발생할 수 있음.

```java
public class Pizza {
    private String dough;
    private String sauce;
    private String cheese;
    private String topping;

    public Pizza() {} // 기본 생성자

    public void setDough(String dough) { this.dough = dough; }
    public void setSauce(String sauce) { this.sauce = sauce; }
    public void setCheese(String cheese) { this.cheese = cheese; }
    public void setTopping(String topping) { this.topping = topping; }
}
```

```java
Pizza pizza = new Pizza();
pizza.setDough("Thin Crust");
pizza.setSauce("Tomato");
pizza.setCheese("Cheddar");
pizza.setTopping("Pepperoni");
```

#### 빌더 패턴
- 메서드 체이닝 방식으로 가독성이 좋음.
- 객체가 완전히 생성된 후 변경할 수 없음 (불변성 보장).
- 기본값을 설정할 수 있음.
- 매개변수 개수가 많아도 안전하게 관리할 수 있음.

```java
public class Pizza {
    private final String dough;
    private final String sauce;
    private final String cheese;
    private final String topping;

    // 내부 정적 빌더 클래스
    public static class Builder {
        private String dough;
        private String sauce = "Tomato"; // 기본값 설정 가능
        private String cheese = "Mozzarella";
        private String topping = "None";

        public Builder(String dough) {
            this.dough = dough;
        }

        public Builder sauce(String sauce) {
            this.sauce = sauce;
            return this;
        }

        public Builder cheese(String cheese) {
            this.cheese = cheese;
            return this;
        }

        public Builder topping(String topping) {
            this.topping = topping;
            return this;
        }

        public Pizza build() {
            return new Pizza(this);
        }
    }

    private Pizza(Builder builder) {
        this.dough = builder.dough;
        this.sauce = builder.sauce;
        this.cheese = builder.cheese;
        this.topping = builder.topping;
    }
}
```

```java
Pizza pizza = new Pizza.Builder("Thin Crust")
        .sauce("Tomato")
        .cheese("Cheddar")
        .topping("Pepperoni")
        .build();
```
        
### 3. private 생성자나 열거 타입으로 싱글턴임을 보증하라
#### 싱글턴을 구현하는 방법
1. private 생성자 + static final 필드
    - `INSTANCE`는 클래스가 로드될 때 단 한 번 생성됨.
    - **싱글턴이 보장됨** (항상 같은 인스턴스를 사용).
    - **하지만** API를 보면 `public static final Singleton INSTANCE;`가 **필드 공개**라서       
        → 필요하면 **정적 팩토리 메서드를 사용하는 게 더 좋을 수도 있음**.
    
    ```java
    public class Singleton {
        public static final Singleton INSTANCE = new Singleton(); // static final 필드
    
        private Singleton() { } // private 생성자 (외부에서 객체 생성 불가)
    }    
    ```
    
    ```java
    Singleton singleton1 = Singleton.INSTANCE;
    Singleton singleton2 = Singleton.INSTANCE;
    System.out.println(singleton1 == singleton2); // true (같은 인스턴스)    
    ```
    
2. private 생성자 + 정적 팩토리 메서드
    - `getInstance()`를 통해 **싱글턴 객체를 반환**.
    - **필요할 경우 → 싱글턴이 아니게 변경하는 것도 가능!**(예: 나중에 싱글턴을 제거하고 여러 개의 객체를 허용할 수 있음).
    - 리플랙션 api를 활용하면 싱글턴이 깨질 수 있음
    - but 이를 막을 수도 있음
    
    ```java
    public class Singleton {
        private static final Singleton INSTANCE = new Singleton();
    
        private Singleton() { }
    
        public static Singleton getInstance() {  // 정적 팩토리 메서드
            return INSTANCE;
        }
    }    
    ```
    
    ```java
    Singleton singleton1 = Singleton.getInstance();
    Singleton singleton2 = Singleton.getInstance();
    System.out.println(singleton1 == singleton2); // true (같은 인스턴스)    
    ```
    
3. enum 타입
    - 리플렉션을 통한 싱글턴 깨짐 **방지됨!**
    - 직렬화/역직렬화 시 **싱글턴 유지됨!**
    - **하지만** Enum은 상속을 지원하지 않기 때문에 **싱글턴 클래스를 확장해야 한다면 Enum을 쓸 수 없음**.
    
    ```java
    public enum Singleton {
        INSTANCE;  // enum 요소 자체가 싱글턴 인스턴스
    
        public void doSomething() {
            System.out.println("Singleton using Enum");
        }
    }    
    ```
    
    ```java
    Singleton singleton1 = Singleton.INSTANCE;
    Singleton singleton2 = Singleton.INSTANCE;
    System.out.println(singleton1 == singleton2); // true (같은 인스턴스)    
    ```
    

**가장 안전한 방법?**
- **Enum을 사용한 싱글턴** (`enum Singleton { INSTANCE; }`)
- 상속이 필요하면 **private 생성자 + 정적 팩토리 메서드 + 리플렉션 방지 코드** 추가

### 4. 인스턴스화를 막으려거든 private 생성자를 사용하라    
인스턴스화가 필요 없는 유틸리티 클래스의 경우에 보통 사용

#### 인스턴스화가 가능할 때
- `MathUtils`는 **정적 메서드만 제공하는데, 인스턴스를 만들 수 있음** → 불필요한 객체가 생성될 수 있음.
- 의도하지 않은 **잘못된 사용 가능성 증가**.

```java
public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
}

MathUtils utils = new MathUtils(); // ❌ 불필요한 객체 생성 가능
```
    
#### private 생성자로 인스턴스화 방지    
```java
public class MathUtils {
    // ✅ private 생성자로 인스턴스화 방지
    private MathUtils() {
        throw new AssertionError("Cannot instantiate utility class");
    }

    public static int add(int a, int b) {
        return a + b;
    }
}

MathUtils utils = new MathUtils(); // ❌ 컴파일 에러 또는 AssertionError 발생!
```
        
### 5. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라
#### 직접 자원을 명시하는 경우
- `SpellChecker` 클래스가 **Dictionary 구현체에 강하게 결합되어 있음 (하드코딩)**
- 나중에 다른 `Dictionary`를 사용하려면 **코드를 직접 수정해야 함**
- **테스트가 어렵다** → `Dictionary`의 동작을 변경하려면 `SpellChecker` 내부를 수정해야 함.

```java
public class SpellChecker {
    private final Dictionary dictionary = new Dictionary(); // ❌ 직접 생성

    public boolean isValid(String word) {
        return dictionary.contains(word);
    }
}    
```
    
#### 의존 객체 주입
- **다양한 `Dictionary` 구현체를 쉽게 주입할 수 있음**
- `Dictionary` 구현을 바꾸려면 **생성자에서 다른 객체를 넣어주기만 하면 됨**
- **테스트가 쉬워짐** → 가짜(Fake) `Dictionary`를 주입해서 단위 테스트 가능!

```java
public class SpellChecker {
    private final Dictionary dictionary;

    // ✅ 의존성 주입 (Dependency Injection)
    public SpellChecker(Dictionary dictionary) {
        this.dictionary = dictionary;
    }

    public boolean isValid(String word) {
        return dictionary.contains(word);
    }
}
```

```java
Dictionary englishDictionary = new EnglishDictionary();
SpellChecker spellChecker = new SpellChecker(englishDictionary); // 주입
```

#### 인터페이스를 활용하면 확장성이 좋아진다.

```java
// 사전(Dictionary) 인터페이스
public interface Dictionary {
    boolean contains(String word);
}

// 영어 사전 구현체
public class EnglishDictionary implements Dictionary {
    public boolean contains(String word) {
        return word.equalsIgnoreCase("hello");
    }
}

// 프랑스어 사전 구현체
public class FrenchDictionary implements Dictionary {
    public boolean contains(String word) {
        return word.equalsIgnoreCase("bonjour");
    }
}    
```    
#### 주입 방법
1. 생성자 주입(추천)
2. setter 주입
3. 메서드 주입
    
### 6. 불필요한 객체 생성을 피하라
1. 같은 값을 가진 객체를 계속 생성하지 말고, 기존 객체를 재사용할 방법을 고려하라!
2. 객체 생성 비용이 큰 경우, 정적 팩토리 메서드(`valueOf()`, `Pattern.compile()`)를 활용하라.
3. 불변 객체(String, Boolean, Integer 등)는 캐싱을 활용하여 재사용하라.
4. 불변 리스트나 컬렉션을 매번 새로 만들 필요 없이 `Collections.unmodifiableList()` 등을 활용하라.

    
### 7. 다 쓴 객체 참조를 해제하라
1. GC가 자동으로 메모리를 관리하지만, 개발자가 직접 객체 참조를 해제해야 할 때가 있다!
2. 컬렉션(List, Map, Set 등)에 객체를 추가하면 필요할 때 꼭 제거하자.
3. pop() 등의 메서드에서 다 쓴 객체는 `null`로 초기화하자.
4. 캐시(Cache)를 사용할 때는 `remove()` 또는 `WeakHashMap`을 활용하자.
5. 이벤트 리스너와 콜백도 필요할 때 해제해야 한다.


### 8. finalizer와 cleaner 사용을 피하라
1. `finalize()`는 절대 사용하지 말자! (`Cleaner`도 마찬가지)
2. 자원을 해제해야 한다면 `AutoCloseable`을 구현하고 `try-with-resources`를 사용하자.
3. 명시적으로 `close()`를 호출하는 것이 `finalize()`보다 훨씬 안전하다.
4. 고급 기법인 `PhantomReference`는 꼭 필요한 경우에만 사용하자.


## 아이템 9. try-finally 보다는 try-wth-resources 를 사용하라
### try-finally
- `reader.close();`를 반드시 호출해야 하므로 코드가 지저분해짐.
- `finally` 블록에서 close()를 호출할 때 예외가 발생하면, 원래 예외가 사라질 수도 있음!
- 실수로 `reader.close();`를 호출하지 않으면 자원이 해제되지 않아 메모리 누수 가능.

```java
import java.io.*;

public class TryFinallyExample {
    public static void main(String[] args) {
        BufferedReader reader = null;
        try {
            reader = new BufferedReader(new FileReader("test.txt"));
            System.out.println(reader.readLine());
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                if (reader != null) reader.close(); // ❌ 반드시 close()를 호출해야 함
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}
```

### try-with-resources
- `close()`를 직접 호출할 필요가 없음 → 자동으로 해제됨.
- 코드가 깔끔하고 가독성이 좋아짐.
- 예외 발생 시 원래 예외를 유지하면서도 자원을 안전하게 정리할 수 있음.

```java
import java.io.*;

public class TryWithResourcesExample {
    public static void main(String[] args) {
        try (BufferedReader reader = new BufferedReader(new FileReader("test.txt"))) { // ✅ 자동 해제
            System.out.println(reader.readLine());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
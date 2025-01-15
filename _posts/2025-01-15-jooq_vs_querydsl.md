---
title: jooq vs querydsl
layout: post
tag: [java, orm, querydsl, jooq]
toc: true
---

## jOOQ vs QueryDSL

### 1. ORM 등장 배경

애플리케이션 개발에서 객체지향 프로그래밍(OOP)과 관계형 데이터베이스(RDB)의 데이터 모델 간의 불일치를 해결하기 위해 ORM(Object-Relational Mapping)이 등장한다. ORM은 프로그래머가 SQL을 직접 작성하지 않고도 데이터베이스와 상호작용할 수 있는 추상 계층을 제공한다. 이를 통해 코드의 유지보수성과 가독성을 높이고, 데이터베이스 종속성을 줄이는 것이 가능하다.

초기에는 JDBC와 같은 저수준 API를 이용해 SQL 쿼리를 작성해야 한다. 하지만 이 방식은 복잡한 코드와 높은 유지보수 비용을 초래한다. 이를 해결하기 위해 Hibernate, JPA 같은 ORM 프레임워크가 등장하고, QueryDSL과 같은 타입 안전한 쿼리 빌더가 이를 보완하는 역할을 한다.

```java
    String query = "SELECT * FROM users WHERE age > ?";
    try (Connection connection = DriverManager.getConnection(url, user, password);
        PreparedStatement preparedStatement = connection.prepareStatement(query)) {
        preparedStatement.setInt(1, 18);
        ResultSet resultSet = preparedStatement.executeQuery();
        while (resultSet.next()) {
            System.out.println(resultSet.getString("name"));
        }
    } catch (SQLException e) {
        e.printStackTrace();
    }
```

초기에는 JDBC와 같은 저수준 API를 이용해 SQL 쿼리를 작성해야 했습니다. 하지만 이 방식은 복잡한 코드와 높은 유지보수 비용을 초래했습니다. 이를 해결하기 위해 Hibernate, JPA 같은 ORM 프레임워크가 등장했고, QueryDSL과 같은 타입 안전한 쿼리 빌더가 이를 보완하는 역할을 했습니다.

---

### 2. QueryDSL의 등장

QueryDSL은 ORM과의 자연스러운 통합을 목표로 등장한 타입 안전 쿼리 빌더다. 주로 JPA와 함께 사용되며, JPQL을 타입 안전하게 작성할 수 있도록 지원한다. QueryDSL은 다음과 같은 장점을 제공한다:

- **타입 안전성**: 컴파일 타임에 쿼리 오류를 잡을 수 있음.
- **코드 가독성**: JPQL이나 HQL과 유사한 문법으로 쉽게 이해 가능.
- **생산성 향상**: 자동 생성된 메타 모델 클래스를 활용하여 빠르게 쿼리 작성 가능.

```java
    QUser user = QUser.user;
    JPQLQuery<User> query = new JPAQuery<>(entityManager);
    List<User> users = query.select(user)
                            .from(user)
                            .where(user.age.gt(18))
                            .fetch();
    users.forEach(u -> System.out.println(u.getName()));
```

하지만 최근 몇 년 동안 QueryDSL 프로젝트는 유지보수와 업데이트가 거의 이루어지지 않고 있다. GitHub 저장소를 기준으로 마지막 커밋 날짜는 오래되었고, 이슈 관리도 활발하지 않은 상태다. 이는 다음과 같은 문제를 야기한다:

- **신뢰도 하락**: 새로운 JPA 표준과의 호환성 부족.
- **커뮤니티 지원 약화**: 사용자들이 문제를 해결하기 어려움.

QueryDSL의 업데이트 정체는 개발자들로 하여금 대안을 모색하도록 만든다.

[querydsl github](https://github.com/querydsl/querydsl) 

---

### 3. jOOQ의 등장

jOOQ는 SQL 중심의 애플리케이션 개발을 목표로 등장한 라이브러리다. ORM과 달리, jOOQ는 SQL의 강력한 기능을 직접적으로 활용할 수 있는 DSL(Domain Specific Language)을 제공한다. jOOQ의 특징은 다음과 같다:

- **SQL 중심 설계**: ORM처럼 객체와의 매핑이 아닌, SQL 문법을 코드로 표현.
- **데이터베이스 기능 활용**: 특정 데이터베이스에 특화된 기능과 최적화를 지원.
- **타입 안전성**: QueryDSL처럼 컴파일 타임에 쿼리 오류를 방지.
- **강력한 코드 생성기**: 데이터베이스 스키마 기반으로 코드 생성.

```java
    DSLContext context = DSL.using(configuration);
    Result<Record> result = context.select()
                                .from("users")
                                .where(DSL.field("age").gt(18))
                                .fetch();
    result.forEach(record -> System.out.println(record.get("name")));
```

jOOQ는 현재도 활발히 유지보수되고 있으며, 최신 SQL 표준과 데이터베이스 별 기능을 빠르게 통합하는 데 강점을 보인다. 특히 복잡한 SQL 쿼리가 필요한 프로젝트에서 큰 장점을 제공한다.

[jooq github](https://github.com/jOOQ/jOOQ)

---

이전 회사에서 QueryDSL을 많이 사용하고, 인프런의 여러 강의를 통해 이를 학습한 경험이 있다. 하지만 최근 들어 QueryDSL은 업데이트가 이루어지지 않으며 유지보수 상태가 좋지 않다. 반면, jOOQ는 지속적으로 유지보수되고 있으며, 최신 SQL 표준과 데이터베이스 기능을 지원하는 데 강점을 가진다. 따라서 죽어가는 라이브러리보다 활발히 유지보수되는 도구에 적응하는 것이 장기적인 관점에서 더 나은 선택이라고 생각한다.
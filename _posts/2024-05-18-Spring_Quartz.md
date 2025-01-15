---
title: Spring Quartz
layout: post
tag: [WIL, Quartz, Schedule]
toc: true
---


## Spring Quartz?
Spring Quartz는 Spring Framework와 Quartz Scheduler를 통합하여 작업 스케줄링을 더욱 쉽게 만들어주는 오픈 소스 라이브러리입니다. 주기적인 작업을 예약하거나 특정 시간에 작업을 실행할 수 있도록 도와주며, 다양한 환경에서 안정적으로 동작합니다.

## Quartz의 특징
- 유연한 스케줄링: Cron 표현식이나 간단한 반복 시간 설정을 통해 다양한 작업 스케줄링이 가능합니다.
- 통합성: Spring Framework와 긴밀하게 통합되어 있어 기존의 Spring 애플리케이션에 쉽게 추가할 수 있습니다.
- 확장성: 분산 환경에서도 안정적으로 동작할 수 있도록 설계되었습니다.
- 이벤트 리스너 지원: JobListener와 TriggerListener를 통해 작업 전후에 특정 로직을 추가할 수 있습니다.

## 설정방법
```xml
<dependency>
    <groupId>org.quartz-scheduler</groupId>
    <artifactId>quartz</artifactId>
    <version>2.3.0</version>
</dependency>

```

```gradle 
// https://mvnrepository.com/artifact/org.quartz-scheduler/quartz
implementation group: 'org.quartz-scheduler', name: 'quartz', version: '2.3.0'
```
[Maven Central repository](https://mvnrepository.com/artifact/org.quartz-scheduler/quartz)

## Quartz API
프레임워크의 핵심은 스케줄러입니다. 우리 애플리케이션의 런타임 환경을 관리하는 역할을 합니다.

확장성을 보장하기 위해 Quartz는 다중 스레드 아키텍처를 기반으로 합니다. 시작하면 프레임워크는 스케줄러가 작업을 실행하는 데 사용하는 작업자 스레드 세트를 초기화합니다.

이것이 프레임워크가 많은 작업을 동시에 실행할 수 있는 방법입니다. 또한 스레드 환경을 관리하기 위해 느슨하게 결합된 스레드 풀 관리 구성 요소 집합에 의존합니다.

API의 주요 인터페이스는 다음과 같습니다:
- Scheduler : 프레임워크의 스케줄러와 상호 작용하기 위한 기본 API
- Job : 실행하고자 하는 구성 요소에 의해 구현되는 인터페이스
- JobDetail : 작업의 인스턴스를 정의하는 데 사용됩니다
- Trigger : 특정 작업을 수행할 일정을 결정하는 구성 요소
- JobBuilder : Job의 인스턴스를 정의하는 JobDetail 인스턴스를 구축하는 데 사용됩니다
- TriggerBuilder : Trigger 인스턴스 구축에 사용

각 구성 요소를 살펴보겠습니다.

### Job
```java
public class SimpleJob implements Job {
    public void execute(JobExecutionContext arg0) throws JobExecutionException {
        System.out.println("This is a quartz job!");
    }
}
```
SimpleJob은 Job 인터페이스를 구현하는 클래스입니다.

Job의 트리거가 실행되면 스케줄러의 작업자 스레드 중 하나에서 execute() 메서드가 호출됩니다.

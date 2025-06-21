## Spring 기초

### Spring?

Java 기반의 어플리케이션 프레임워크로 IoC(제어의 역전)과 DI(의존성 주입), AOP(관점 지향 프로그래밍)을 중심으로 구성되어 있으며, 모듈화와 확장성, 테스트 용이성에 강점을 가진다.

* POJO 기반 개발

  * Java 객체를 단순하게 설계하고 Spring이 의존성과 생명주기를 관리하도록 함

* 제어의 역전

  * 객체의 생성부터 생명주기의 관리까지 모든 객체에 대한 제어권이 역전되는 것을 의미한다 개발자가 프레임워크에 필요한 객체를 개발하고 조립하고 나면 프레임워크가 내부에서 결정된 대로 그것을 호출하고 관리한다. 

* 의존성 주입(DI)

  * 느슨한 결합, 유닛 테스트 편의성 확보
  * 스프링 프레임워크에서 지원하는 제어의 역전의 형태로서 클래스 사이의 의존 관계를 빈 설정 정보를 바탕으로 컨테이너가 자동 연결
    * 스프링 컨테이너는 자바 객체의 생명주기를 관리하고, 추가적인 기능을 제공
    * `BeanFactory`와 `ApplicationContext`가 존재(후자가 전자 상속)
  * 주입에는 수정자 주입, 필드 주입, 생성자 주입이 존재
    * 수정자 주입: `setter`를 public으로 제공하므로 변경될 위험 존재
    * 필드 주입: 외부에서 변경 불가
    * 생성자 주입: 생성 시점에 1번 호출 후 `final`로 불변으로 설계 가능
  * 주입받는 대상이 변하더라도 그 구현 자체를 수정할 일이 없거나 수정이 줄어들게 되므로 의존성이 줄어들어 클래스의 재사용성이 높아지고, 유지 보수가 편리
  * 의존성 주입으로 인해 stub, mock 객체를 사용한 unit 테스트의 이점

  * 의존성 주입을 위한 선행 작업이 필요하며, 코드 추적이 상대적으로 어려움

* AOP 지원

  * 트랜잭션, 로깅, 보안 등 횡단 관심사 분리
  * Aspect: 공통 관심사(로깅 등)
  * JoinPoint: Advice가 적용될 수 있는 지점(메서드 호출 등)
  * Advice: 언제 어떤 로직을 실행할지 정의(`@Before` 등)
  * Pointcut: 어떤 JoinPoint에 Advice를 적용할지 결정
  * Weaving: Advice를 실제 객체에 적용하는 과정

* 풍부한 생태계

  * Spring MVC, Spring Security, Spring Data, Spring Batch 등 각 도메인별 모듈 존재 및 방대한 레퍼런스와 문제 해결 경험

| 구분              | Spring(Java)                                          | Node.js                      | Django(Python)                              |
| ----------------- | ----------------------------------------------------- | ---------------------------- | ------------------------------------------- |
| **언어**          | Java                                                  | JavaScript                   | Python                                      |
| **멀티스레딩**    | ✔️ Thread per request                                  | ❌ 싱글 스레드 (non-blocking) | ✔️ Thread per request                        |
| **비동기 처리**   | 기본은 동기 + 별도 비동기 지원 (`@Async`, Reactor 등) | 기본 비동기 (Event loop)     | 기본 동기, 비동기는 Celery/Channels 등 필요 |
| **성능**          | 고성능 (JVM 튜닝 가능)                                | I/O에 강함 (CPU 연산은 약함) | 적당한 성능, 빠른 개발                      |
| **생산성**        | 높은 학습 곡선, 성숙한 생태계                         | 빠른 프로토타이핑, 적은 코드 | 쉬운 문법, 빠른 개발                        |
| **적합한 시스템** | 복잡한 도메인, 대규모 시스템                          | 실시간 I/O 처리, 채팅, 게임  | 간단한 CRUD, MVP 앱                         |
| **기본 철학**     | DI 기반, 구조화 강조                                  | 자유롭고 유연함              | MTV(Model-Template-View), 명확한 컨벤션     |
| **비용**          | 고성능 서버 필요할 수 있음                            | 저렴한 서버로도 가능         | 빠른 개발/배포에 적합                       |

#### Spring Bean

* 스프링 컨테이너 안에서 관리되는 객체
* 어노테이션 혹은 xml 기반으로 등록되며 스프링 컨테이너에 의해 주입되어 쉽게 사용 가능
* 스프링 컨테이너가 생성되고 나서 생성 되며, 컨테이너가 종료되기 전 콜백 발생
* 스코프
  * 싱글톤: Spring 프레임워크의 기본, 스프링 컨테이너 시작과 종료까지 1개 객체로 유지
  * 프로토타입: 빈 생성, 의존관계 주입, 초기화 이후로는 컨테이너에서 관리하지 않는 스코프로 매번 요청마다 새로 생성
  * 웹 스코프
    * request: 각 요청이 들어오고 나갈 때까지 유지
    * session: 세션이 생성되고 종료될 때까지 유지
    * application: 웹의 서블릿 컨텍스트와 동일한 생명주기 - 서블릿 컨텍스트는 웹 어플리케이션 내 모든 서블릿을 관리하며 정보를 공유할 수 있도록 하는데, 톰캣 컨테이너가 실행 시 애플리케이션당 하나의 서블릿 컨텍스트 생성

### Spring boot

* Spring boot는 Spring을 실무에서 더 쉽고 빠르게 사용하기 위해 만들어진 프레임워크로, Spring이 기반 기술이라면 Spring boot는 생산성 향상을 위한 wrapper 역할

| 항목        | Spring Framework          | Spring Boot                                |
| ----------- | ------------------------- | ------------------------------------------ |
| 설정        | XML 또는 Java Config      | 거의 모든 설정을 자동화                    |
| 서버 실행   | 외부 WAS 필요 (Tomcat 등) | 내장 WAS (Tomcat/Jetty/Netty 등) 포함      |
| 시작 속도   | 느림 (설정 필요 많음)     | 빠름 (`starter` 의존성 기반)               |
| 개발자 경험 | 수동 설정, 복잡함         | opinionated, production-ready default 제공 |
| 모니터링    | 없음 (직접 구성해야 함)   | actuator 제공                              |



### 참고

#### stub & mock

테스트에서 외부 의존성을 대체하기 위해 사용하는 방식임은 둘 다 동일하지만, 목적과 행위 추적 방식에 차이 존재

##### 공통점

* 외부 시스템(네트워크, DB, 메시지 브로커 등)에 의존하지 않고 단위 테스트를 빠르고 안정적으로 수행하기 위해 사용
* 테스트 대상이 의존하는 객체를 대체하여 원하는 응답 생성 가능

##### 차이점

| 항목      | Stub                                   | Mock                                                |
| --------- | -------------------------------------- | --------------------------------------------------- |
| 목적      | **고정된 응답값**을 제공               | **호출 여부, 순서, 횟수 등** 행위 검증              |
| 주 용도   | 테스트 대상의 동작 유도                | 테스트 대상이 **외부와 어떻게 상호작용하는지 검증** |
| 구현 방식 | 보통 하드코딩된 값 리턴                | 호출 정보 추적 및 검증 기능 내장                    |
| 예시      | 특정 파라미터 입력 시 고정된 결과 리턴 | 특정 메서드가 몇 번 호출되었는지 검증               |
| 의도      | **상태 기반 테스트**                   | **행위 기반 테스트**                                |

##### 예시

```java
// 테스트 대상
class UserService {
    private EmailSender emailSender;
    public UserService(EmailSender emailSender) {
        this.emailSender = emailSender;
    }

    public void register(String userEmail) {
        // ... 회원 등록 로직
        emailSender.sendWelcomeEmail(userEmail); // 외부 시스템 호출
    }
}
```

```java
// stub
class StubEmailSender implements EmailSender {
    public void sendWelcomeEmail(String email) {
        // 아무 일도 하지 않음 (혹은 로그 출력 등)
    }
}

// 테스트
UserService service = new UserService(new StubEmailSender());
service.register("test@example.com");
// -> 실제 이메일 전송 없이 로직만 테스트

```

```java
// mock
EmailSender mockSender = Mockito.mock(EmailSender.class);
UserService service = new UserService(mockSender);

service.register("test@example.com");

Mockito.verify(mockSender).sendWelcomeEmail("test@example.com");
// -> 해당 메서드가 정확히 호출되었는지 검증

```

> `Stub`는 결과값을 조작하는 것을 중심으로 하여 단순 테스트, 값 대입 등에 활용된다. 직접 구현하거나 `Mockito`와 같은 라이브러리를 사용 가능하다. `Mock`은 행위를 검증하는 것을 중심으로 TDD나 인터페이스 명세 확인에 활용된다. `Mockito`, `JMock`, `EasyMock` 등이 사용 가능하다.

#### Spring AOP에서의 어노테이션 처리 흐름

* `@Aspect`가 붙은 클래스를 스프링 빈으로 등록하면...
* `AnnotationAwareAspectJAutoProxyCreator` 또는 `InfrastructureAdvisorAutoProxyCreator`가 등록된 Aspect 인식하고, 즉 AOP 설정을 감지하고 프록시 객체 생성 적용 대상인지 판단
* 리플렉션을 통해 Pointcut 매칭
* Pointcut과 매칭되는 메서드가 있으면 해당 메서드에 대한 Advisor 등록
* Advisor는 Pointcut + Advice로 구성된 객체
* 인터페이스가 있으면 JDK 동적 프록시, 없으면 CGLIB을 통해 프록시 객체 생성
* 실제 객체 대신 이 프록시가 빈으로 등록
* 프록시 객체가 메서드 호출시 pointcut에 매칭되면 Advice 실행 (joinPoint.proceed() 전/후 작업), 매칭 안 되면 그냥 메서드 실행

##### + 일반 Java 어노테이션의 동작

* 참고) spring은 프레임워크가 런타임에 리플렉션으로 어노테이션 처리
* 컴파일러가 의미를 해석(`@Override` 등)
* 어노테이션 프로세서: 컴파일 시 코드를 생성하거나 검사 - lombok이나 Dagger(Annoation Processing Tool 기반)
  * 작성한 어노테이션 보고 APT로 새 코드를 생성해서 최종 class 파일 생성

 
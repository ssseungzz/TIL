### ThreadLocal이란?
`ThreadLocal<T>`은 각 스레드가 독립적인 값을 가지도록 하는 클래스이다. 여러 스레드가 하나의 `ThreadLocal` 객체를 공유하더라도 각 스레드마다 자신만의 값을 가지게 된다. 이를 통해 공유 자원을 쓰지 않고 안전하게 스레드 값을 보관할 수 있다.
#### ThreadLocal의 내부 구현
각 `Thread` 객체는 `ThreadLocalMap`이라는 내부 맵을 가지고 있고, 이 맵은 키가 `ThreadLocal` 객체이며, 값이 해당 스레드에 대한 값이다. `ThreadLocal.set(value)`를 호출하면 현재 스레드의 `ThreadLocalMap`에 저장되고, `ThreadLocal.get()` 호출 시 현재 스레드의 `ThreadLocalMap` 에서 결과 값을 얻는다.

```java
// ThreadLocal.get의 내부 구현
private T get(Thread t) {  
    ThreadLocalMap map = getMap(t); // 현재 스레드 t에서 ThreadLocalMap 가져옴
    if (map != null) {  
        ThreadLocalMap.Entry e = map.getEntry(this);  
        if (e != null) {  
            @SuppressWarnings("unchecked")  
            T result = (T) e.value;  
            return result;  
        }  
    }  
    return setInitialValue(t);  
}
```

```java
ThreadLocal<String> local = new ThreadLocal<>();

local.set("Hello");
String value = local.get(); // 같은 쓰레드라면 "Hello" 반환
local.remove(); // 메모리 누수 방지를 위해 종료 시 제거

```

#### 사용 시 주의 사항
1. 메모리 누수
	* `ThreadLocal`의 키가 약한 참조로 되어 있어서, 참조가 끊기면 GC는 가능하지만 값(value)은 강한 참조이기 때문에 `Thread`가 살아 있는 동안 계속 남아 있을 수 있다. 특히 스레드 풀에서 스레드를 재사용할 때 큰 문제를 유발할 수 있으므로, 반드시 `remove()` 호출해서 정리해 줘야 한다.
2. 스레드 풀과의 사용 주의
	* 스레드가 재사용되기 때문에 이전 요청의 데이터가 남아 있을 수 있다. `try-finally` 블럭해서 항상 `remove()`를 호출해 줘야 한다.
3. 복잡한 데이터 흐름에는 부적합
	* `ThreadLocal`은 명시적인 인자 전달 없이 데이터를 전달하게 되므로, 코드 가독성이 떨어질 수 있다.

### ThreadLocal과 VirtualThread 
Java 21 버전부터 도입된 Virtual Thread에도 일반적인 (Carrier) Thread와 마찬가지로 `ThreadLocal`이 존재한다. Virtual Thread도 `Thread`의 서브 클래스고, 내부적으로 `ThreadLocalMap`을 가지므로 기존 `ThreadLocal` API들을 사용 가능하다.
```java
ThreadLocal<String> local = new ThreadLocal<>();

Runnable task = () -> {
    local.set("value");
    System.out.println(local.get()); // value 출력
    local.remove();
};

Thread.startVirtualThread(task);

```
참고로 Virtual Thread는 Carrier Thread 위에서 실행된다. 그런데 기존 `ThreadLocal`은 현재 실행 중인 물리적 스레드에 값이 저장되는 구조이다. 이러면 Virtual Thread마다 고유하게 값을 가질 수 있어야 하는 상황에서 값이 Carrier Thread에 저장되면 안 되므로 `ThreadLocal` 관련 코드는 Virtual Thread-aware하게 만들어졌다.
```java
// ThreadLocal 내부 구현 일부
T getCarrierThreadLocal() {  
    assert this instanceof CarrierThreadLocal<T>;  
    return get(Thread.currentCarrierThread());  
}

public void set(T value) {  
    set(Thread.currentThread(), value);  
    if (TRACE_VTHREAD_LOCALS) { // jdk.traceVirtualThreadLocals 옵션 활성화 시 디버깅용 출력  
        dumpStackIfVirtualThread();  
    }  
}

boolean isCarrierThreadLocalPresent() {  
    assert this instanceof CarrierThreadLocal<T>;  
    return isPresent(Thread.currentCarrierThread());  
}
```
다만 Virtual Thread에서는 다음과 같은 이유들로 `ThreadLocal` 사용에 더 주의가 필요하다. 
1. Lightweigh Thread
	* 스케줄러가 자유롭게 멈췄다 재개한다. OS 스레드가 아니라, Carrier Thread 위에서 실행된다.
	* 실행 중 다른 Carrier Thread로 옮겨질 수 있다.
	* `ThreadLocal`은 현재 `Thread`에 종속되어 있으므로, Virtual Thread 자체에 `ThreadLocal`을 거는 대신  Carrier Thread에 걸면 Carrier Thread가 바뀌게 되는 경우 값이 유지되지 않는다.
2. 스레드 풀의 경우보다 위험 존재
	* Carrier Thread(Platform Thread)는 보통 생성되고 나면 종료되기 전까지 같은 자원에서 계속 돌기 때문에 `ThreadLocal`이 유지되는 경우가 많지만 Virtual Thread는 짧게 살기 때문에, 값이 있는 상태에서 사라지는 경우 GC가 더 어렵고 누수 위험이 커질 수 있다.
		* `ThreadLocal`은 weak reference + stroing reference를 섞어서 사용한다. `ThreadLocalMap`은 key로 `WeakReference<ThreadLocal<?>>` 를 사용하지만 value는 일반적인 `Object` 타입의 Strong reference이다. Virtual Thread를 여러 이유로 JVM이 예상보다 오래 잡고 있어 완전히 GC 되지 못한 상태에서, `ThreadLocalMap`안의 value가 강하게 참조하고 있어 GC 루트에서 참조 경로가 끊어지지 않을 수 있다. 이로 인해 생명 주기의 예측이 어려워질 수 있다.
		위와 같은 이유로 [JDK 개발팀도 Virtual Thead에서는 `ScopedValue` 를 사용할 것을 권장](https://openjdk.org/jeps/464?utm_source=chatgpt.com)한다.

### ThreadLocal vs ScopedValue

#### 설계 철학과 목적의 차이

##### ThreadLocal

* 각 스레드마다 독립적인 값을 저장하기 위해 사용되는 가변 변수
* 양방향 업데이트 가능(`get()`, `set()` 호출 가능)
* 값 유지 기간이 스레드 수명과 동일

##### ScopedValue

* 실행의 특정 블록(스코프) 동안 불변 데이터를 전파하기 위한 새로운 구조
* 한 방향 전파만 허용(immutable)
* 라이프타임이 명확하게 한정(`run()` 블록 종료 시 자동 해제)

#### 가변성 및 생명 주기 비교

| 항목               | ThreadLocal                         | ScopedValue                     |
| ------------------ | ----------------------------------- | ------------------------------- |
| 가변성(mutability) | 변경 가능                           | 불변                            |
| 범위(scope)        | 스레드 전체 수명동안 유지           | `run()` 범위 동안만 유효        |
| 해제 시점          | `remove()` 호출 시 또는 스레드 종료 | `.run()` 블록 종료 시 자동 해제 |

#### Virtual thread에서 사용 시?

##### ThreadLocal

* 라이프타임이 스레드 수명과 같아 가볍게 생성되는 virtual thread 수십만 개와 맞지 않음
* 값이 계속 남아 있어 메모리 누수 또는 불필요한 오버헤드 발생 가능

##### Scoped Value

* 메모리 사용 경량(immutable 특성과 바인딩 범위 덕분)
* virtual thread 및 structured concurrency와 자연스럽게 연결됨

###### Scoped Value와 structured concurrency, virtual thread

> structure concurrency란 하나의 작업을 구성하는 여러 하위 작업들이, 부모 작업의 생명 주기 안에서 함께 관리되도록 만드는 프로그래밍 모델

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user  = scope.fork(() -> findUser());
    Future<Integer> order = scope.fork(() -> countOrders());
    scope.join();          // 모든 작업 완료될 때까지 대기
    scope.throwIfFailed(); // 하나라도 실패하면 예외 throw
}
```

* `scope`라는 컨텍스트 안에서 여러 virtual thread를 실행하고, 부모가 끝나면 자식도 정리되며 실패도 함께 전파
  * 스레드 생명 주기가 트리 구조로 관리
* `ScopedValue`는 불변, 스코프(`run()` 또는 `call()`  블록의 생명 주기) 한정, 컨텍스트 전파(해당 블록 안에서 실행되는 모든 virtual thread에게 자동 전파)라는 특징을 가짐

이로 인해,

*  Structured concurrency의 작업 범위를 명확히 나누고 그 범위 안에서만 데이터를 관리하자는 목적에 `ScopedValue`가 적합
  * 블록이 끝나면 자동으로 값이 해제되는 특성은 곧 데이터도 스코프에 갇혀 있는 구조가 되기 때문!
* virtual thread는 가볍게 만들어지고 자주 생성/삭제되므로 carrier thread가 유지되는 동안 가변적인 값을 보관하는 `ThreadLocal`보다 (스코프 내에서) virtual thread에 자동으로 전파되면서 생명 주기가 명확히 끝나는 `ScopedValue`가 적합

```java
ScopedValue.where(USER_CONTEXT, user).run(() -> {
    // 여기서 실행되는 모든 Virtual Thread도 USER_CONTEXT를 참조 가능
});
```

###### + StructuredTaskScope

`StructuredTaskScope`는 Java 21부터 도입된 Structured Concurrency를 실제로 구현한 핵심 클래스

* 여러 개의  virtual thread를 트리 구조로 묶고, 동시 실행 + 에러 관리 + 리소스 정리까지 자동화 가능
  * 여러 task(보통 virtual thread)를 하나의 구조적 컨텍스트 안에서 실행하고, 해당 task들의 완료, 예외, 취소 등을 부모 컨텍스트에서 통합적으로 관리할 수 있게 해 줌
* `java.util.concurrent` 패키지에 위치
* 여러 작업을 동시에 실행하고 싶을 때, 일부 작업이 실패하면 전체를 빠르게 정리하고 싶을 때, virtual thread 기반 병렬 처리를 간단히 하고 싶을 때

```java
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<String> user  = scope.fork(() -> fetchUser());
    Future<Integer> order = scope.fork(() -> fetchOrders());

    scope.join();           // 모든 작업이 끝날 때까지 기다림
    scope.throwIfFailed();  // 하나라도 실패하면 예외 던짐

    String u = user.resultNow();
    Integer o = order.resultNow();
}
```

* 주요 메서드 및 역할
  * `scope.fork(task)`: 새로운 virtual thread에서 task 실행
  * `scope.join()`: 모든 task가 끝날 때까지 기다림
  * `scope.throwIfFailed()`: 예외가 발생한 task가 있으면 예외 던짐
  * `scope.resultNow()`: 결과 즉시 반환
  * `ShutdownOnFailure`: 한 task라도 실패하면 나머지를 취소
  * `ShutdownOnSuccess`: 첫 성공이 나오는 순간 나머지 취소

#### 사용 예시 비교

##### ThreadLocal

```java
static final ThreadLocal<Context> CONTEXT = new ThreadLocal<>();

void serve(...) {
    CONTEXT.set(ctx);
    callA(); callB();
    CONTEXT.remove(); // 미호출시 값이 계속 남아 있을 수 있음!
}
```

##### ScopedValue

```java
static final ScopedValue<Context> CONTEXT = ScopedValue.newInstance();

void serve(...) {
    ScopedValue.where(CONTEXT, ctx).run(() -> {
        callA(); callB();
    });
}
// .run()이 끝나면 CONTEXT는 자동 해제
```

> ThreadLocal은 여전히 유효하지만 주로 가변 캐싱, ConnectioPool 등 한 스레드 내에서 장기 상태 유지가 목표인 경우에 적합하며, ScopedValue는 특히 virtual thread + 구조적 동시성 환경에서, 한 방향으로의 데이터 전파 목적으로 권장된다. [참고](https://openjdk.org/jeps/464?utm_source=chatgpt.com)

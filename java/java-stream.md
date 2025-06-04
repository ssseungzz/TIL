## Java Stream

### Java Stream이란?

자바 8부터 도입된 기능으로, 데이터의 집합(`Collection` 등)을 함수형 스타일로 처리할 수 있도록 도와주는 API

* 선언형 처리 방식: 루프 없이 `filter`, `map`, `reduce` 같은 연산으로 데이터 처리
* 파이프라인 처리: 여러 연산을 연결해서 구성 가능
* 내부 반복: 개발자가 루프를 직접 제어하지 않음
* **지연 연산**(lazy evaluation): 최종 연산이 호출되기 전까지 중간 연산들이 실행되지 않음

#### 주요 구성 요소

* 중간 연산(Intermediate)
  * Stream을 변형하고 또 다른 Stream 리턴
  * `map()`, `filter()`, `sorted()` 등
* 최종 연산(Terminal)
  * Stream을 소비하고 결과를 생성
  * `collect()`, `forEach()`, `reduce()` 등
* 중간 연산은 lazy하게 평가되나, 최종 연산이 실행되면 전체 파이프라인 작동

> Stream은 내부적으로 데이터를 한 번만 흘려보내면서 처리함
>
> 즉, `Iterator` 처럼 작동하기 때문에 데이터를 읽고 나면 다시 읽을 수 없음(재사용 불가)

### Stream의 구조

```css
 [Source] → [Intermediate Operations] → [Terminal Operation]
     \              ↓                      ↓
      \----> [Pipeline] ------> [Sink] → [Traversal/Execution]
```

* Source: 데이터의 시작점
  * `Collection`, `Array`, `Files.lines(path)`, `Stream.of(...)` 의 데이터 구조에서 `Stream`이 시작
  * 이 시점에서는 아무 데이터도 처리하지 않고, 데이터 흐름을 시작할 준비만
* Pipeline 구성: 중간 연산(Intermediate Ops)
  * 스트림은 내부적으로 각 중간 연산을 단일 파이프라인으로 결합
  * 중간 연산은 `Stream`을 리턴하며, 평가되지 않은 상태로 대기
  * 가령, 아래의 예시는 `filter()` → `StatelessOp` 객체 생성 - `map()` → 또 하나의 `StatelessOp` 객체 생성으로 둘 다 내부적으로 `Sink`를 감싸는 구조

```java
stream
    .filter(s -> s.startsWith("a"))
    .map(String::toUpperCase)

// 내부적으로 Sink를 감싸는 구조
Sink<String> mapSink = new Sink<String>() {
    void accept(String s) {
        downstream.accept(s.toUpperCase());
    }
};
```

* Sink: 실행 시점 처리
  * 데이터를 실제로 소비하는 역할
  * 최종 연산이 실행되면, 각 중간 연산이 감싼 `Sink` 체인을 통해 데이터를 하나씩 흘려보내며 처리

```kotlin
data → filterSink → mapSink → collectSink
```

* Traversal: 순회와 소비
  * 최종 연산이 호출되면 스트림은 source 데이터를 하나씩 **순회**하며 위의 Sink 체인을 타고 데이터 처리
  * 중간 연산은 lazy라서 최종 연산 전까지 실행 X
  * 최종 연산이 발생하면 데이터 순회가 발생하고, 그 과정에서 Sink 체인 작동
* 병렬 처리 시 내부 구조(Parallel Stream)
  * 병렬 스트림 사용 시 내부적으로 `ForkJoinPool`과 `Spliterator` 사용
  * `Spliterator`: 데이터를 분할할 수 있는 iterator로 parallelStream은 이걸 사용해서 데이터 쪼갬
  * `ForkJoinPool`: 각각의 분할된 작업을 다른 스레드에서 병렬 실행
  * 모든 경우에 병렬 스트림이 빠른 건 아니고, 오버헤드가 크므로 CPU 바운드 작업 및 대량 데이터에서 유리

### Sink 내부 구현체

`Sink`는 자바 스트림 내부에서 중간 연산과 최종 연산을 연결해 주는 핵심 인터페이스로, Stream API의 실질적인 파이프라인 처리 로직을 담당!

#### Sink란?

* `Sink<T>`는 `java.util.stream` 패키지에 있는 패키지 프라이빗 인터페이스로, Stream 내부에서 데이터를 하나씩 처리하기 위한 **데이터 소비자**

> 공식 정의: A receiver of elements that can be chained together with other sinks to process streams.

```java
interface Sink<T> extends Consumer<T> {
    void begin(long size);
    void end();
    boolean cancellationRequested();
}
```

* `begin(long size)`: 처리 시작 시 호출(ex - 초기화)
* `accept(T t)`: 실제 데이터 처리(Consumer에서 상속)
* `end()`: 모든 데이터 처리 후 호출
* `cancellationRequested`: 단락 평가 가능 여부 (ex - `findFirst()`)

`Sink` 자체는 인터페이스지만, 내부 구현체는 아래와 같은 곳에서 정의됨

* `AbstractPipeline`
* `ReferencePipeline$StatelessOp`
* `ReferencePipeline$StatefulOp`
* `ReduceOps`, `ForEachOps` 등의 내부 클래스

```java
// ex - map() 연산 시 sink 내부 구조
// java 코드
Stream<String> stream = list.stream()
    .map(s -> s.toUpperCase())
    .filter(s -> s.startsWith("A"))
    .forEach(System.out::println);

// 내부 구조
ForEachOp$ForEachSink.accept(String)
↑
FilterOp$FilterSink.accept(String)
↑
MapOp$MapSink.accept(String)
  
// map 연산은 내부적으로 아래와 같이 Sink 감싼다
Sink<T> downstream = ...;
Sink<T> mapSink = new Sink<T>() {
    @Override
    public void accept(T t) {
        R result = mapper.apply(t);
        downstream.accept(result);
    }
};

```

* **중간 연산은 자신의 연산을 수행하고 `downstream.accept()`로 다음 Sink에 넘겨 주는 방식**

#### Stateless vs Stateful Sink

* Stateless
  * 상태 없이 한 요소만 처리
  * `map`, `filter`, `peek`
* Stateful
  * 내부에 상태 유지
  * `distinct`, `sorted`, `limit`
  * 내부적으로 버퍼를 써서 중간에 데이터를 저장하기 때문에 처리 방식이 조금 더 복잡

#### 최종 연산의 Sink

* 최종 연산에서는 실제로 데이터를 소비하는 Sink가 만들어짐

```java
// ex - forEach
Sink<String> terminalSink = new Sink<String>() {
    @Override
    public void accept(String s) {
        System.out.println(s); // accept로 다음 Sink 호출 X
    }
};

```

* `Sink.end()`는 최종 연산의 루프가 끝나고 Sink 체인을 따라 아래에서 위로 호출

```java
list.stream()
    .map(x -> x + 1)
    .filter(x -> x > 5)
    .forEach(System.out::println);
```

* 위와 같은 코드에서, 내부적으로는 아래와 같은 처리 흐름
  * `Spliterator`를 통해 요소를 하나씩 꺼냄
  * 각 요소는 `Sink.accept()` 체인을 거쳐 최종 Sink로 전달
  * 모든 요소 순회가 끝나면, `Sink.end()` 호출
* 실제 호출 위치는 최종 연산 내부(`TerminalOp` 계열 클래스)

```java
// ex ForEachOps.ForEachTask -> AbstractPipeline.copyInto()
// AbstractPipeline.java
private void copyInto(Sink<P_OUT> sink, Spliterator<P_OUT> spliterator) {
    sink.begin(spliterator.getExactSizeIfKnown());
    spliterator.forEachRemaining(sink);
    sink.end(); // 여기서 호출됨
}
```

* `begin()`: 순회 전 초기화
* `forEachRemaining(sink)`: 실제 `accept()` 연산
* `end()`: 순회 끝나고 마지막에 호출
  * 가장 마지막 Sinkdp서 맨앞 Sink까지 순차적 호출
  * 버퍼 정리, 자원 해제, 상태 초기화 등의 역할 수행
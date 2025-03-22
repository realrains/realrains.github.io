---
layout: post
title: Java 24 - Stream Gatherer (스트림 수집기)
date: 2025-03-20
---

{% include image.html url='/assets/image/stream.png' width='400' description='' %}

## 들어가며

Java 8에서 도입된 Stream API는 요소 소스(a source of elements), 중간 연산(intermediate operation), 최종 연산(terminal operation) 의 크게 세 가지 요소로 구성됩니다.

```java
Stream.of("the", "", "fox", "jumps", "over", "the", "", "dog")  // 요소 소스
    .filter(Predicate.not(String::isEmpty))                     // 중간 작업
    .count();                                                   // 최종 작업
```

* **요소 소스**는 스트림의 입력 데이터를 제공하며, `of` 처럼 정적으로 정의할 수도 있고 `iterate`, `generate` 처럼 동적으로 생성할 수도 있습니다.

* **중간 연산**은 스트림 처리 과정에서 각 요소를 변환(`map`), 필터링(`filter`), 축소(`reduce`)하는 작업을 담당하며, 지연 평가(lazy evaluation)됩니다.

* **최종 연산**은 스트림의 처리를 마무리짓는 작업으로, `toList` 로 컬렉션을 만들거나, `count` 로 개수를 계산하거나, `anyMatch`, `allMatch` 처럼 불리언 값을 반환하거나, `forEach` 로 각 요소에 동작을 수행할 수 있습니다.

## 스트림 사용자 확장 포인트의 과거와 한계

사실 `toList`, `count`, `anyMatch`, `allMatch` 등 대부분의 최종 연산은 `Stream::collect` 로 대체할 수 있습니다. `collect` 는 `Collector` 인터페이스를 인자로 받으며, 이를 통해 다양한 맞춤형 최종 연산을 구현할 수 있습니다. 또한 `Collectors.toSet`, `groupingBy`, `joining` 등 Stream 인터페이스에 직접 포함되지 않은 유용한 연산들도 [Collectors 유틸리티 클래스](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html)를 통해 제공됩니다.

하지만 중간 연산은 이러한 사용자 확장 포인트가 존재하지 않습니다. 즉, `map`, `filter` 등 Stream 인터페이스에 정의된 기본 메서드만 사용할 수 있으며, 새로운 중간 연산을 직접 정의할 수 없습니다. 이로 인해 복잡한 변환을 구현할 때는 여러 연산을 조합하거나 별도의 처리 단계를 추가해야 해 스트림 코드가 길어지고 복잡해지는 경우가 많았습니다.


## Stream Gatherer 의 등장

이러한 한계를 극복하기 위해 Java 24에서는 Stream Gatherer라는 새로운 기능이 [(JEP 485)](https://openjdk.org/jeps/485) 를 통해 도입되었습니다. Stream의 일반화된 중간 연산을 위한 `Stream::gather` 메서드와, 이를 구현하기 위한 인터페이스 `Gatherer` 가 추가되었습니다.

먼저 Java 24에 함께 도입된 몇 가지 표준 Gatherer 구현체를 살펴보겠습니다.

**fixedWindow**

원소를 고정된 크기만큼 묶어 `List<T>` 로 반환합니다.

```java
Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.fixedWindow(3))
    .toList()
// [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
```

**windowSliding**

슬라이딩 윈도우 방식으로 원소를 묶어 `List<T>` 로 반환합니다.

```java
Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.windowSliding(3))
    .toList()
// [[1, 2, 3], [3, 4, 5], [5, 6, 7], [7, 8, 9]]
```

**fold**

`reduce` 와 유사하게, 초기값과 원소에 folding 함수를 적용하여 누적합니다.

```java
Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.fold(() -> 0, (a, b) -> a + b))
    .toList();
// [45]
```

**scan**

초기값과 원소에 scanner 함수를 적용하고, 그 중간 결과를 모두 누적해 반환합니다.

```java
Stream.of(1, 2, 3, 4, 5, 6, 7, 8, 9)
    .gather(Gatherers.scan(() -> 0, (a, b) -> a + b))
    .toList();
// [1, 3, 6, 10, 15, 21, 28, 36, 45]
```

## 사용자 정의 Gatherer 구현

`Gatherer<T, A, R>` 는 입력 타입(T), 내부 상태 타입(A), 출력 타입(R)의 세 타입 파라미터를 받습니다. 이 중 내부 상태 타입은 스트림 연산에 필요한 변경 가능한 상태를 나타냅니다.

다음은 기존의 `distinct()` 연산이 `equals` 기반으로만 동작하는 한계를 극복하기 위해, 특정 키에 따라 중복을 제거하는 `distinctBy`를 `Gatherer` 를 확장해 직접 구현한 예시입니다.

```java
public class DistinctBy<INPUT extends @Nullable Object>
        implements Gatherer<INPUT, DistinctBy.State, INPUT> {

    private final Function<INPUT, Object> keyExtractor;

    DistinctBy(final Function<INPUT, @Nullable Object> keyExtractor) {
        Objects.requireNonNull(keyExtractor, "Mapping function must not be null");
        this.keyExtractor = keyExtractor;
    }

    @Override
    public Supplier<State> initializer() {
        return State::new; // 내부 상태 초기화
    }

    @Override
    public Integrator<DistinctBy.State, INPUT, INPUT> integrator() {
        return Integrator.ofGreedy((state, element, downstream) -> {
            if (state.seen.add(keyExtractor.apply(element))) {
                downstream.push(element);
            }
            return !downstream.isRejecting();
        });
    }

    public static class State {
        final Set<@Nullable Object> seen = new HashSet<>();
    }
}
```

사용 예시는 다음과 같습니다.

```java
record Person(String name, Integer age) {};

Stream.of(
    new Person("John", 20),
    new Person("Jane", 22),
    new Person("Jane", 30),
    new Person("Seth", 19),
    new Person("Joth", 19))
    .gather(new DistinctBy(Person::name))
    .toList();
// [Person("John", 20), Person("Jane", 22), Person("Seth", 19)] 
```

## 마치며

벌써 [gatherers4j](https://github.com/tginsberg/gatherers4j) 처럼 Gatherer 구현체를 모아둔 오픈소스 라이브러리도 등장하고 있습니다. Java 스트림의 중간 연산을 보다 유연하게 확장하고 싶다면, 이러한 라이브러리의 구현을 참고해보는 것도 좋은 접근이 될 수 있겠습니다.

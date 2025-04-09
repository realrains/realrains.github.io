---
layout: post
title: 엔티티 리스너 클래스와 이벤트
date: 2025-04-09
---

최근 [Toss Tech 블로그 포스트](https://toss.tech/article/34481)를 보다가 흥미로운 Kotlin/Spring 코드가 있어 정리해본다.

```kotlin
// 출처: https://toss.tech/article/34481

@Component
class TermsAgreementEntityListener {
    private val eventPublisher: ApplicationEventPublisher by lazy {
        SpringContext.context
    }

    @PostUpdate
    @PostRemove
    @PostPersist
    fun postProcess(termsAgreement: TermsAgreement) {
        eventPublisher.publishEvent(
	        StrongConsistencyCacheEvictEvent.of(termsAgreement = termsAgreement)
        )
    }
}
```

첫 번째로 눈에 띈 건 `TermsAgreementEntityListener`의 `postProcess` 메서드에 붙어 있는 `@PostUpdate`, `@PostRemove`, `@PostPersist` 같은 엔티티 라이프사이클 어노테이션들이었다. 이 어노테이션들은 보통 엔티티 클래스 내부에 정의된 메서드에 붙여 사용하는게 익숙해서, 이렇게 별도의 엔티티 리스너 클래스를 정의해 사용하는 방식은 낯설게 느껴졌다.

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
class MyEntity {
    // ...
    @CreatedDate
    private LocalDateTime createdAt;
    @LastModifiedDate
    private LocalDateTime lastModifiedAt;
}
```

예컨대 `@CreatedDate`, `@LastModifiedDate` 등을 사용할 때 같이 쓰는 `AuditingEntityListener` 도 사실 구조는 동일한데, 직접 리스너 클래스를 정의하는 방식으로도 활용할 수 있음을 새삼 깨닫게되었다.

두 번째로 흥미로웠던 건 `TermsAgreementEntityListener` 에서 `ApplicationEventPublisher` 를 사용해 이벤트를 발행하는 부분이었다. `@EntityListeners` 를 통해 등록된 엔티티 리스너 클래스는 스프링 컨테이너가 아닌 JPA에 의해 직접 생성되기 때문에, 생성자 주입이나 `@Autowired` 등을 통해 스프링 빈을 주입받을 수 없다.

그렇다면 어떻게 `ApplicationEventPublisher` 를 사용할 수 있었던 걸까? 블로그의 예제에서는 `SpringContext` 를 통해 `ApplicationEventPublisher` 가 지연 초기화되고 있는데, 이에 대한 구현이 본문에 생략되어 있어 추측해 보면 아래와 같을 것으로 보인다:

```kotlin
@Component
class SpringContext : ApplicationContextAware {
    companion object {
        lateinit var context: ApplicationContext
            private set
    }

    override fun setApplicationContext(applicationContext: ApplicationContext) {
        context = applicationContext
    }
}
```

* `SpringContext` 빈은 `ApplicationContextAware`를 구현해, `ApplicationContext`를 setter를 통해 주입받는다.
* `ApplicationContext`는 `ApplicationEventPublisher`를 상속하고 있으므로, 엔티티 리스너 내 `eventPublisher` 필드에 타입 캐스팅 없이 할당될 수 있다.

정리하자면, `TermsAgreementEntityListener`는 `SpringContext` 빈의 정적 필드인 `context`를 통해 간접적으로 `ApplicationEventPublisher`를 주입받는다. (여기서 다만 이 리스너는 여전히 스프링 빈이 아니기 때문에, 클래스에 붙은 `@Component` 어노테이션은 실제로는 필요가 없다.)


블로그 포스트에서는 `TermsAgreementEntityListener`를 활용해, 엔티티가 생성/수정/삭제될 때마다 캐시 무효화를 위한 애플리케이션 이벤트를 발행하고, 이를 별도의 이벤트 리스너(`@TransactionalEventListener`)에서 받아 캐시 무효화 작업을 수행하는 구조를 보여주고 있다.

한편, Spring Data JPA를 사용할 경우에는 엔티티 클래스가 `AbstractAggregateRoot`를 상속받도록 하고, 이벤트 발행이 필요할 때마다 `registerEvent` 메서드를 통해 이벤트를 등록해두면, 해당 엔티티가 `repository.save()`될 때 누적된 이벤트가 한 번에 발행된다. 이 방식은 직접 `ApplicationEventPublisher`를 주입받아 제어하거나, 이벤트 발행을 담당할 별도의 서비스 클래스를 만들 필요 없이 깔끔하게 처리할 수 있으며, [Spring Data JPA 에서도 권장하는 접근](https://docs.spring.io/spring-data/jpa/reference/repositories/core-domain-events.html#page-title)이다. (물론 이때의 이벤트는 취지에 맞게 ‘Aggregate Root’에 대한 ‘도메인 이벤트’여야 하겠지만.)

물론 도메인 이벤트가 단순히 생성, 수정, 삭제 세 가지로만 구성되진 않겠지만, 결국 유형으로 따지면 셋 중 하나일 것이다. 그렇다면 도메인 이벤트 성격별 최상위 타입을 정의하고, 캐시 무효화를 처리하는 이벤트 리스너에서 이 최상위 타입을 받아 일괄 처리하는 방식도 가능했을 것 같다. 그렇다 보니 엔티티 리스너에서 직접 애플리케이션 이벤트를 발행하는 방식이 과연 적절했는가에 대한 의문이 들었다.

Toss Tech 블로그의 작성자가 왜 이 구조를 선택했는지는 명확하진 않지만, 몇 가지 추측은 가능하다:

- **도메인 이벤트 자체가 없는 설계**  
  해당 시스템이 도메인 모델 중심 설계를 필요로 할 만큼 복잡하지 않은 경우일 수 있다. 예를 들어 트랜잭션 스크립트 기반의 단순한 구조에서는 굳이 `AggregateRoot`를 구분하거나 도메인 이벤트를 정의하지 않아도 기능 구현이 가능하기 때문에, 자연스럽게 `AbstractAggregateRoot`를 사용한 이벤트 발행 방식은 고려 대상이 아니었을 수 있다.

- **캐시 무효화를 횡단 관심사로 분리하고 싶은 경우**  
  캐시 무효화는 핵심 도메인의 일부라기보다는 기술적인 책임에 가까우므로, 도메인 이벤트 흐름과 섞지 않고 별도의 리스너 계층에서 처리하고 싶었을 수 있다. 이런 경우에는 도메인 레이어에서 이벤트를 등록하는 것보다, JPA 라이프사이클 훅을 이용해 후처리하는 방식이 더 깔끔하게 느껴질 수도 있다. (그렇다면 엔티티리스너에서 바로 캐시 무효화를 하면 되지 않나 싶기도 했는데, 트랜잭션 경계에 대한 문제도 있어서 이벤트로 처리해야 할 것 같다.)

- **Soft Delete 가 아닐 경우 문제**  
  `repository.delete()` 를 통해 엔티티를 제거할 경우 `AbstractAggregateRoot` 를 활용한 방식에서는 삭제 이벤트를 별도로 발행하기가 어렵다.

어찌되었건 이 케이스에 한해서는 개인적으로 도메인 이벤트를 활용했을 것 같지만 특정 기능을 수행하는 클래스가 스프링 빈이 될 수 없는 상황에서 어플리케이션 컨텍스트에 등록된 빈을 참조하고 싶을 때에는 비슷한 방법을 사용해 볼 수 있을 것 같다.

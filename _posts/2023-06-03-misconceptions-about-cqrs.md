---
layout: post
title: CQRS에 대한 오해와 읽기모델 분리하기
date: 2023-06-03
---

## CQRS에 대한 오해

CQRS(Command and Query Responsibility Seperation) 패턴은 읽기(Query) 요구사항과 생성, 삭제, 변경을 포함하는 명령(Command) 요구사항을 처리하는데에 대한 책임을 분리하는 것을 의미합니다. CQRS 패턴은 클린 아키텍처, 이벤트 소싱등의 아키텍처와 주로 자주 같이 언급되기 때문에 일종의 아키텍처로 여겨지거나 복잡한 응용기술중 하나로 오해받기도 합니다.

그러나 CQRS는 매우 **간단한 패턴**이며 기존 어플리케이션 구성에서 **단계적 리팩터링을 통해 적용할 수 있는 패턴**입니다. 본 글에서는 도메인 주도 설계 (DDD, Domain Driven Design)로 만들어진 어플리케이션에서 발생할 수 있는 문제점을 바탕으로 CQRS 패턴을 설명하지만 CQRS 패턴은 우리가 흔히 아는 다른 디자인 패턴들과 마찬가지로 아키텍처 독립적인 기술입니다.


## 도메인 모델을 읽기 요구사항에서 사용할 때의 문제점

계층형 구조를 가지고 있는 많은 어플리케이션에서는 관계형 데이터베이스와 같은 데이터 소스로부터 가져온 데이터를 어플리케이션의 데이터 접근 계층에서 추상화된 도메인 모델로 변환하고, 해당 클래스를 통해 여러 비즈니스 로직과 관련된 동작들을 수행하게 됩니다. 이러한 동작들을 구현하기 위해서는 필연적으로 데이터 일관성을 보장하기 위해 여러가지 검증 로직이 추가되고, 여러 프로세스가 같은 데이터에 접근할 때 일어나는 경합을 막기 위한 동시성 제어 로직들이 수반되게 됩니다.

어플리케이션의 읽기 요구사항에서 명령 요구사항에서와 동일한 도메인 모델을 사용하게되면 데이터 일관성을 보장하기 위해 구현된 여러가지 로직들이 읽기 요구사항에도 많은부분 동일하게 적용되어 불필요한 오버헤드를 발생시킬 수 있습니다. 예를 들어, Spring Data JPA 와 같은 ORM 프레임워크에서 영속성 컨텍스트가 제공하는 캐싱 및 변경감지 로직들은 읽기 요구사항에서는 불필요합니다. 또한 읽기 요구사항에서 필요한 데이터 필드만을 선별적으로 가져오기가 어려워지고, 복잡한 도메인 클래스 - DTO 간 변환 로직을 구현해야 합니다. 또 엔티티 관계 구성에 따라 불필요한 테이블 조인이 발생할 수도 있습니다.

무엇보다 가장 큰 문제는 읽기 요구사항이 도메인 클래스와 연관 클래스들의 설계에 영향을 줄 수 있다는 것입니다. 읽기 요구사항이 추가될 때마다 도메인의 핵심 기능과는 무관한 속성이 추가될 가능성이 있고 도메인 모델의 복잡도는 증가하게 됩니다. 특히 애그리거트(Aggregate) 하위에 존재하는 루트가 아닌 엔티티를 조회해야 하는 요구사항이 발생할 때 도메인 주도 설계 원칙에 위배되는 하위 엔티티에 대한 레포지터리들이 무분별하게 생겨날 위험도 있습니다.


## CQRS 패턴으로 리팩터링하기 (in Spring)

위와같은 문제가 발생하는 근본적인 문제는 도메인 주도 설계가 도메인 상태의 일관성을 유지하는 것을 주된 목표로 애그리거트 단위의 업데이트를 강제하기 때문에 읽기 요구사항이 필요한 기능과 코드레벨에서 불일치가 발생하기 때문입니다. 따라서 읽기 요구사항에 의해 도메인 모델이 복잡해지는 경우 도메인 모델에서 읽기 모델을 분리할 수 있습니다. 다음은 스프링 프레임워크가 적용된 자바 어플리케이션을 CQRS 패턴을 통해 리팩터링 하는 예제입니다.

입학원서를 나타내는 Application 엔티티와 1:N 관계를 가지는 증빙파일 Attachment 엔티티가 있습니다. Attachment 는 Application 의 하위 도메인이므로 Application 도메인을 통해서만 추가, 제거되어야 하며 repository 도 root 엔티티인 Application 만 가져야 합니다. 마찬가지로 입학 원서를 심사하는 심사자 Reviewer 엔티티도 1:N 으로 존재합니다.


```java
@Entity
public class Application {
    @Id
    private Long id;
    private Long applicantId;
    @OneToMany
    private List<Attachment> attachments;
    @OneToMany
    private List<Reviewer> reviewers;

    public void addAttachment(String filePath) { /* */ }
    public void deleteAttachment(Long attachmentId) { /* */ }
}


@Entity
public class Attachment {
    @Id
    private Long id;
    private Long applicationId;
    private String filePath;
    private LocalDateTime createdAt;
}

@Entity
public class Reviewer {
    @Id
    private Long id;
    private Long professorId;
}

public interface ApplicationRepository extends CrudRepository<Application, Long> {
    Optional<Application> findById(Long id);
}
```

이 구조에서 Application 에 대한 Attachment 추가나 삭제는 꽤 잘 동작할 것입니다. 모든 추가/삭제 요청은 Application ID 를 통해 Application 를 찾은뒤에 이루어질 테니까요. 만약 지원자 (applicantId) 가 올린 모든 Attachment 를 조회해야 할 경우는 어떨까요?

```java
public interface ApplicationRepository extends CrudRepository<Application, Long> {
    Optional<Application> findById(Long id);
    List<Application> findByApplicantId(Long applicantId);
}
```

위와 같이 새로운 조회 메서드 `findByApplicantId` 를 ApplicationRepository 에 추가해 지원자 기준으로 모든 Application 을 가져온 뒤 stream 을 활용해 모든 Attachment 리스트를 만들어 낼 수 있겠지요. Attachment 만을 가져오기 위해 Reviewer 와도 불필요한 조인이 발생할 것입니다. fetch type 을 LAZY 로 설정해 해결할 수도 있겠지만 이는 Application 을 가져오는 다른 모든 기능들에 영향을 주기 때문에 다른 곳에서 성능 저하가 발생할 수 있습니다. 읽기 요구사항을 만족시키기 위한 변경사항이 다른곳에 영향을 주는 대표적인 예입니다.

그렇다면 Attachemnt 만을 조회할 수 있는 JPA Repository 를 추가하면 되지 않을까요? 조금은 맞았지만 이 역시 좋은 방법은 아닙니다. Attachement 가 root 엔티티가 아님에도 추가/삭제가 가능한 인터페이스가 열려 있으면 조심한다 하더라도 데이터 일관성이 깨질 수 있는 잠재적인 위험이 있기 때문입니다. 읽기 요구사항에 불필요한 영속성 컨텍스트가 여전히 동작하기도 하고요.

```java
@Transactional(readOnly = true)
public interface AttachmentRepository extends CrudRepository<Attachment, Long> {
    @Query(/* */)
    List<Attachment> findByApplicantId(Long applicantId);
}
```

위와 같이 읽기 전용 트랜잭션을 가지도록 제한하거나 JPA Projection 을 활용할수도 있겠습니다. 업데이트 동작을 제한할수도 있고 영속성 컨텍스트를 거치지 않게 할수도 있죠. 그러나 @Query 어노테이션에 JPQL 을 작성해야 하는 번거로움과 JPA 구현체 하나당 엔티티 하나만을 조회할 수 있다는 제한 때문에 읽기 요구사항이 생길때마다 하위 도메인에 대한 읽기 전용 Repository 가 추가되어야 합니다.

가장 좋은 방법은 읽기 요구사항만을 만족시키기 위한 별도의 쿼리 서비스를 구현하는 것입니다. 대표적으로 QueryDSL 이 있습니다. 엔티티 하나당 Repository 를 하나하나 만들 필요도 없고 번잡한 JPQL 이나 네이티브 쿼리를 작성할 필요도 없습니다.


## 참고 자료
- [CQRS 패턴 - Microsoft Learn](https://learn.microsoft.com/ko-kr/azure/architecture/patterns/cqrs)
---
layout: post
title: 도커 컴포즈 확장을 통한 스프링 부트 개발 및 테스트 환경 구성
date: 2025-02-02
---

스프링 부트 (Spring Boot) 3.1 부터 `ConnectionDetails` 인터페이스를 기반으로 도커 컴포즈 (Docker Compose) 지원이 추가되었다. 기존엔 어플리케이션 개발시 데이터베이스를 비롯한 외부 인프라스트럭처를 코드를 통해 구성하고 싶다면 도커 컴포즈를 이용해 구성 정보를 관리하고, 개발자가 어플리케이션 시작과 종료시 직접 컨테이너를 띄우고 종료해야 했다.

예를 들어, 데이터베이스로 MySQL 을 사용하고 싶다면 아래와 같이 도커 컴포즈 파일을 작성한 뒤에 어플리케이션을 시작하기 전에 `docker-compose up` 명령을 통해 컨테이너와 네트워크를 미리 구성하고, 어플리케이션 종료 후에는 별도로 `docker-compose down` 을 실행해 컨테이너를 종료하고 제거해야 한다.

**docker-compose.yml**

```yml
services:
  database:
    image: mysql:8
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: database
```

그리고 개발 환경의 스프링 부트 어플리케이션 프로퍼티에는 아래와 같이 도커 컴포즈에서 설정한 데이터베이스와 동일한 드라이버 클래스, JDBC URL, 유저 이름과 패스워드를 입력해주어야 한다.

**application.properties**
```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/database
spring.datasource.username=root
spring.datasource.password=database
```

## 개발환경에서의 도커 컴포즈 통합 예제

*테스트 환경: Spring Boot 3.4.1*

스프링 부트에서 제공하는 도커 컴포즈 통합을 사용하면 위에서 언급했던 과정이 좀 더 단순해진다. 이를 위해 먼저 spring-boot-docker-compose 의존성을 추가한다. JPA 데이터 소스를 자동으로 추가하는지 확인하기 위해 JPA 와 MySQL 드라이버 의존성도 추가했다.

```groovy
dependencies {
    // ...
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    runtimeOnly 'com.mysql:mysql-connector-j'
    developmentOnly 'org.springframework.boot:spring-boot-docker-compose'
}
```

의존성이 추가되면 스프링 부트는 작업 디렉터리에서 다음과 같은 이름의 도커 컴포즈 설정파일을 찾는다. ([참고](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/docker/compose/core/DockerComposeFile.html#find(java.io.File)))

* `compose.yaml`
* `compose.yml`
* `docker-compose.yaml`
* `docker-compose.yml`

이제 스프링 부트 어플리케이션을 시작해보자. 사용된 컴포즈 파일과 어플리케이션 프로퍼티는 다음과 같다.

**docker-compose.yml**

```yml
services:
  database:
    image: mysql:8
    ports:
      - "3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: database
```

**application.properties**

```properties
# spring.datasource 관련 설정을 입력하지 않았음
spring.jpa.show-sql=true
spring.jpa.generate-ddl=true
```

**JPA Entity**

```java
@Entity
class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private Integer age;
}

interface People extends JpaRepository<Person, Long> {
}
```


{% include image-full.html url='/assets/image/spring-docker-compose-1.webp' width='100%' description='' %}

로그를 살펴보면, 어플리케이션 시작 후 프로젝트 루트에 있는 `docker-compose.yml` 파일을 찾아서 DockerCli 클래스를 통해 컨테이너를 구성하는 것을 확인할 수 있다. 컨테이너 헬스체크가 완료된 후에는 데이터 소스를 구성하고 JPA 가 DDL 을 정상적으로 수행한 것을 볼 수 있다.

어플리케이션 프로퍼티에서 별도로 어떤 RDBMS 를 사용하는지 설정 않았음에도 드라이버 클래스와 JDBC URL 을 찾았고, 도커 컴포즈 파일에서 MySQL 컨테이너의 포트 포워딩을 명시적으로 선언하지 않았음에도 랜덤으로 설정되는 포트를 찾아 연결한 점이 인상적이다.

도커 컴포즈 통합의 기본 라이프사이클 설정은 `start-and-stop` 이기 때문에 스프링 부트 어플리케이션이 종료되면 구성했던 도커 컴포즈도 같이 종료된다. 만약 이를 변경하고 싶다면 `spring.docker.compose.lifecycle-management` 설정을 다음 중 하나로 변경하면 된다.

* `start-and-stop`: 도커 컴포즈가 없으면 어플리케이션 시작시 도커 컴포즈 시작, 종료시 도커 컴포즈 종료
* `start-only`: 도커 컴포즈가 없으면 어플리케이션 시작시 도커 컴포즈 시작, 종료시 아무 것도 하지 않음
* `none`: 어플리케이션 시작과 종료시 도커 컴포즈를 제어하지 않음

## 지원하는 컨테이너와 커스텀 이미지 적용

기본적으로 스프링 부트 도커 컴포즈 통합은 도커 컴포즈 설정 내 이미지 이름을 보고 자동으로 설정을 구성한다. 예를 들어 이미지 이름이 "mysql", "postgres", "mariadb" 등이면 JDBC 와 R2DBC 타입의 `ConnectionDetails` 클래스를 구성한다. 그 밖에 엘라스틱 서치나 레빗엠큐, 카프카 등의 이미지도 자동으로 인식하는데, 어떤 컨테이너 이미지들에 대해 `ConnectionDetails` 를 지원하는지 확인하려면 [스프링 부트 공식문서](https://docs.spring.io/spring-boot/reference/features/dev-services.html#features.dev-services.docker-compose.service-connections)를 참고하면 된다.

**지원하는 이미지와 커넥션 중 일부**

| Connection Details               | Container Name                                 |
| -------------------------------- | ---------------------------------------------- |
| `ElasticsearchConnectionDetails` | "elasticsearch"                                |
| `JdbcConnectionDetails`          | "mariadb", "mssql/server", "mysql", "postgres" |
| `MongoConnectionDetails`         | "mongo", "bitnami/mongodb"                     |
| `Neo4jConnectionDetails`         | "neo4j", "bitnami/neo4j"                       |
| `R2dbcConnectionDetails`         | "mariadb", "mssql/server", "mysql", "postgres" |
| `RabbitConnectionDetails`        | "rabbitmq", "bitnami/rabbitmq"                 |
| `RedisConnectionDetails`         | "redis", "bitnami/redis"                       |


만약, 문서에서 명시하지 않은 커스터마이징 된 이미지를 사용하고 싶다면 아래와 같이 도커 컴포즈 서비스에 별도의 레이블을 지정하면 된다.

**레이블 적용 예시**

```yml
services:
  database:
    image: 'mycompany/mycustommysql:8.0'
    ports:
      - "3306"
    labels:
      org.springframework.boot.service-connection: mysql
```

## 통합 테스트 환경에서 도커 컴포즈 사용하기

아래와 같이 어플리케이션 프로퍼티에서 `skip-in-tests` 옵션을 `false` 로 바꾸면 개발환경에서 도커 컴포즈를 자동으로 구성하는 것 뿐만 아니라 테스트 환경에서도 구성되도록 할 수 있다. (기본값: `true`)

```properties
spring.docker.compose.skip.in-tests=false
```

그리고 gradle 의존성 레벨을 `testAndDevelopmentOnly` 로 변경해 준다.

```groovy
testAndDevelopmentOnly 'org.springframework.boot:spring-boot-docker-compose'
```

이후 `@DataJpaTest` 를 통해 JPA Repository 에 엔티티 하나를 저장하는 테스트를 작성했다.

```java
@DataJpaTest
class AssetRepositoryTest {
    @Autowired
    private People people;

    @Test
    void saveTest() {
        people.save(new Person("John", 20));
    }
}
```

그러나 테스트를 실행해보면 아래와 같이 도커 컴포즈 설정파일을 찾을 수 없다는 에러와 함께 테스트가 실패한다.

{% include image-full.html url='/assets/image/spring-docker-compose-2.webp' width='100%' description='' %}

원인은 테스트 실행시의 작업 디렉터리가 달라 상대 경로로 도커 컴포즈 파일을 찾을 수 없기 때문인데, `build.gradle` 설정에서 다음과 같이 `test` 테스크의 작업 디렉터리를 프로젝트 루트로 수정해주면 된다.

**build.gradle**

```groovy
test {
    workingDir = rootProject.projectDir
    useJUnitPlatform()
}
```

이제 다시 테스트를 실행하면 테스트 환경에서도 도커 컴포즈 컨테이너가 정상적으로 구성되는 것을 확인할 수 있다.

{% include image-full.html url='/assets/image/spring-docker-compose-3.webp' width='100%' description='' %}


## DataJpaTest 가 도커 컴포즈에서 설정한 소스를 인식하는 원리

`@DataJpaTest` 에는 기본적으로 `@AutoConfigureTestDatabase` 어노테이션이 포함되어 있다. 해당 어노테이션은 테스트 환경에서 `DataSource` 빈을 대체하는 설정을 제공하는데 `replace` 옵션에 따라 다음 세 가지 동작을 하게 된다.

* `ANY`: DataSource 가 자동구성 되었거나 직접 구성되었을 때 모두 대체함
* `AUTO_CONFIGURED`: DataSource 가 자동구성 되었을 때만 대체함
* `NONE`: 어플리케이션의 기본 DataSource 를 대체하지 않음

Spring Boot 3.4.0 버전 이전에서는 `AutoConfigureTestDatabase` 의 `replace` 옵션의 기본값이 `ANY` 였기 때문에 모든 상황에서 H2 와 같은 테스트용 임베디드 데이터베이스 구성으로 대체하려고 시도하고 이러한 임베디드 데이터베이스 구성에 필요한 의존성이나 설정이 누락되어 있으면 에러를 발생시켰다.

**AutoConfigureTestDatabase 3.3.x 버전**
```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {

    @PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
    Replace replace() default Replace.ANY;

    EmbeddedDatabaseConnection connection() default EmbeddedDatabaseConnection.NONE;

    enum Replace {
        ANY,
        AUTO_CONFIGURED,
        NONE
    }
}
```

**AutoConfigureTestDatabase 3.4.x 버전**

```java
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@ImportAutoConfiguration
@PropertyMapping("spring.test.database")
public @interface AutoConfigureTestDatabase {

    @PropertyMapping(skip = SkipPropertyMapping.ON_DEFAULT_VALUE)
    Replace replace() default Replace.NON_TEST;

    EmbeddedDatabaseConnection connection() default EmbeddedDatabaseConnection.NONE;

    enum Replace {
        NON_TEST,
        ANY,
        AUTO_CONFIGURED,
        NONE
    }
}
```

Spring Boot 3.4 버전부터는 `replace` 옵션에 `NON_TEST` 가 추가되고 이것이 기본 옵션이 되었는데 DataSource 빈이 자동 설정되어 있고, **테스트 테이터베이스**에 연결되어 있지 않은 경우에만 DataSource 빈을 임베디드 데이터소스로 교체하게 된다.

여기서 **테스트 데이터베이스**란 아래 세가지 케이스를 말한다.

* `ContainerImageMetadata` 를 포함하는 모든 빈 정의 (예: `@ServiceConnection` 이 지정된 Testcontainer 데이터베이스 및 Docker Compose를 사용하여 생성된 연결)
* `@DynamicPropertySource` 를 통해 설정된 `spring.datasource.url` 을 사용하는 모든 연결
* Testcontainers JDBC 문법을 사용하여 설정된 `spring.datasource.url` 을 사용하는 모든 연결

Docker Compose 를 통해 스프링 부트 어플리케이션을 구성하는 경우가 첫 번째 케이스에 해당하므로 임베디드 데이터소스로 대체되지 않고 테스트에서 컨테이너로 구성된 데이터베이스를 DataSource 로 사용하게 된다.

한편 이러한 기능 변화로 인해 TestContainers 를 통해 테스트 데이터베이스를 구성할 때 명시적으로 `AutoConfigureTestDatabase` 의 `replace` 옵션을 `NONE` 으로 설정해주어야 했던 부분도 마찬가지로 불필요하게 되었다.

## 참고

* [Spring Blog - Docker Compose Support in Spring Boot 3.1](https://spring.io/blog/2023/06/21/docker-compose-support-in-spring-boot-3-1)
* [Spring Docs - Docker Compose Support](https://docs.spring.io/spring-boot/reference/features/dev-services.html#features.dev-services.docker-compose)

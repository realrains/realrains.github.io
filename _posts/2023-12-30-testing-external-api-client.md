---
layout: post
title: 외부 API 클라이언트 테스트하기
date: 2023-12-30
---

마이크로서비스 아키텍처 (MSA) 가 일반화된 요즘의 개발 환경에서는 어플리케이션 외부의 다양한 서비스와 여러 방식으로 통신하게 됩니다. 기능 구현을 위해서 한 어플리케이션 내에서 아무리 적어도 서너개 이상의 외부 서비스와 연동하게 되곤 하는데, 내 서비스의 API 를 노출할 때와 다른 서비스의 API 를 사용할 때 모두 HTTP 프로토콜을 사용하는 것이 가장 일반적이라 사용할 API 제공자가 정의한 스펙에 맞춰 해당 HTTP API 를 호출하는 코드를 작성하는 상황이 상당히 자주 일어납니다.

이와 관련해 언어나 프레임워크 별로 어떤 HTTP 클라이언트를 사용할지, 어떤 방식으로 예외를 처리할지, 다수의 API 를 호출해야 하는 상황에서 성능은 어떻게 개선할지 등 여러 중요한 엔지니어링적 고민사항들이 많이 생겨나곤 합니다. 이번 포스트에서는 그 중에서도 외부 API 클라이언트의 테스트에 대해 이야기해보고, 스프링 프레임워크 기반의 개발 환경에서의 테스트 사례도 공유해드리도록 하겠습니다.

## 외부 API 클라이언트 테스트가 필요한가요?

외부 API 클라이언트에 대한 테스트를 작성한다고 했을 때, 그 필요성 자체에 의문을 품는 분들이 종종 있습니다. 외부 API 는 우리 어플리케이션에서 제어 불가능한 외부 프로세스 자원이기 때문에 테스팅을 하는 것이 의미가 없다고요. 우리 어플리케이션의 정상 동작이 결국 실 환경에서 외부 API 의 상태와 기능 변경에 영향을 받기 때문에 개발이나 스테이징 환경에서의 통합 테스트 정도만 의미가 있다고 말하기도 합니다.

{% include image.html url='/assets/image/diagram_exteranl_api_test.png' %}

하지만 이 말은 절반은 맞고 절반은 틀렸습니다. 외부 API 는 분명히 우리 어플리케이션에서 제어 불가능한 외부 자원이기 때문에 우리 어플리케이션에서 아무리 잘 테스트 하더라도 항상 정상임을 보증할 수 없습니다. 그러나 **테스트에서 확인하고자 하는 것은 언제까지나 우리가 작성한 코드이지 외부의 서비스가 아닙니다.** 

외부 HTTP API 연동 클라이언트를 작성해 본 경험이 있는 분들이라면, 꽤나 많이 직렬화 (Serialization) 와 역직렬화 (Deserialization) 관련된 오류를 경험해 본 적이 있으실 겁니다. DTO 클래스의 필드명을 잘못 입력해서 요청/응답의 JSON 매핑에서 예외가 발생할 수도 있고, 더 심한 경우 매퍼의 설정에 따라 누락된 DTO 필드가 기본값으로 초기화 되어서 버그를 눈치채지 못하는 경우도 있습니다.

따라서 우리는 외부 서비스 제공자가 정의한 API 요청 및 인증 스펙에 맞춰 우리가 '잘' 요청하고 있는지, 그들이 제공하기로 한 응답을 우리가 '잘' 가져오고 있는지 즉, 계약 (Contract) 를 잘 준수하고 있는지 테스트 해야합니다. 이는 외부 서비스의 정상 작동여부와 완전히 별개의 문제이지요.

## 테스트는 또 하나의 문서

테스트를 작성하는 것의 또 다른 장점은 그 자체가 문서로 기능할 수 있다는 것입니다. 많은 경우 처음 API 를 연동할 때 연동 클라이언트를 개발한 개발자 이외에는 해당 외부 서비스에 대한 정보가 없는 상황이 대부분일 것입니다. 나중에 해당 클라이언트에 대한 유지 보수가 필요한 상황이 발생했을 때 클라이언트 코드가 예외 처리등 풍부한 상황에 대응하는 코드를 개발해두지 않았다면, 새로 유지보수를 맡은 개발자는 직접 따로 API 를 호출해보거나 사내 포털을 뒤적이면서 API 제공 부서의 문서를 찾아다닌 후에야 코드에서 제대로 드러나지 않는 API 스펙을 유추해 낼 수 있을 것입니다.

```java
// API 에서 예외가 발생한다면 예외는 어떤 형식이지..?
public ProductDto getProduct(Long id) {
    return client.exchange("/product/{id}", HttpEntity.EMPTY, ProductDto.class, id).getBody();
}
```

그러나 만약 해당 클라이언트에 대한 테스트 코드가 작성되어 있다면, 해당 테스트 코드를 통해 어떤 API 를 호출하는지, 어떤 예외가 발생할 수 있는지, 어떤 응답을 받을 수 있는지 등을 쉽게 파악할 수 있을 것입니다. 최종적으로 API 제공 부서의 문서를 참고해야 할 지라도 기본적인 기능을 파악하는데 드는 시간을 훨씬 단축시킬 수 있을 것입니다.


## MockServer 를 활용한 단위 테스트

[MockServer](https://github.com/mock-server/mockserver) 를 활용하면 외부 API 를 호출하는 클라이언트 코드를 테스트할 때, 외부 API 서버를 실제로 호출하지 않고도 테스트를 할 수 있습니다. Spring 에서 제공하는 `@RestClientTest` 가 RestTemplate 및 RestTemplateBuilder 가 사용된 코드만을 테스트 가능한 것에 비해 클라이언트 라이브러리 중립적으로 Mock 테스트가 가능하다는 장점이 있습니다.

### Dependency

```groovy
dependencies {
    testImplementation 'org.mock-server:mockserver-netty:5.15.0'
}
```

### Client Code

HTTP Client 는 Spring Boot 3.2 부터 사용 가능한 `RestClient` 로 구현하였습니다.

```java
public record ProductDto (
        Long id,
        String name,
        String description,
        Integer price
) {}

@Component
public class RestProductClient {

    private final RestClient client;

    public RestProductClient(String host) {
        this.client = RestClient.create(host);
    }

    public ProductDto getProduct(Long id) {
        return client.get()
                .uri("/products/{id}", id)
                .retrieve()
                .body(ProductDto.class);
    }
}
```

### Test Code

```java
class RestProductClientTest {

    private ClientAndServer mockServer;
    private RestProductClient client;

    @BeforeEach
    void setUp() {
        mockServer = ClientAndServer.startClientAndServer(9876);
    }

    @AfterEach
    void tearDown() {
        mockServer.stop();
    }

    @Test
    void getProductById() {
        // Given
        var client = new RestProductClient("http://localhost:9876"); // (1)
        var productId = 1L;
        var responseBody = """
                {
                  "id": 1,
                  "name": "iPhone 12",
                  "description": "Apple iPhone 12 64GB",
                  "price": 1000000
                }
                """;
    

        // When
        mockServer
                .when(
                    request() // (2)
                        .withMethod("GET")
                        .withPath("/products/1")
                ).respond(
                    response() // (3)
                        .withStatusCode(200)
                        .withBody(json(responseBody))
                );

        var result = client.getProduct(1L); // (4)
        var expected = new ProductDto(
                1L,
                "iPhone 12",
                "Apple iPhone 12 64GB",
                1000000
        );

        // Then
        assertEquals(expected, result); // (5)
    }
}
```

* (1) : mockserver 와 동일한 포트로 접속하도록 클라이언트 인스턴스를 생성합니다.
* (2) : 요청을 받길 기대하는 요청 메서드와 경로를 지정합니다.
* (3) : 요청에 대한 응답을 지정합니다.
* (4) : 클라이언트 코드를 실행하고, 응답을 바인딩합니다.
* (5) : 응답이 기대하는 결과와 일치하는지 검증합니다.


## API 클라이언트가 협력 객체인 경우

{% include image.html url='/assets/image/integration-test-diagram.png' %}

만약 API 클라이언트 구현 자체에 대한 테스트가 아니라 위에서 구현한 `RestProductClient` 와 같은 클라이언트가 다른 비즈니스 객체의 협력 객체로 사용되는 테스트라면 어떻게 해야 할까요? 마찬가지로 MockServer 를 이용해서 HTTP 통신은 Mocking 할 수 있지만, 테스트의 핵심 관심사는 HTTP 통신이 아니라 외부 서비스로부터 받은 응답을 바탕으로 기대하는 추가적인 비즈니스 로직이 제대로 수행되는지이기 때문에 MockServer 를 사용하는건 다소 불필요하고 번거로운 일일 수 있습니다. 우리가 Repository 와 협력관계에 있는 클래스를 테스트할 때 DB 를 Mocking 하지 않는 것을 상상해보면 이해하기 쉽습니다.

```java
@Service
public class ProductService {

    private final RestProductClient client;

    public ProductService(RestProductClient client) {
        this.client = client;
    }
    
    public Integer getProductPrice(Long id) {
        return client.getProduct(id).price();
    }

}
```

### Mock 을 활용한 단위 테스트

ProductService 에 대한 단위 테스트를 수행할 경우, 협력 객체인 RestResourceClient 를 Mocking 해서 테스트를 수행할 수 있습니다.

```java
class ProductServiceTest {

    private ProductService productService;
    private RestProductClient productClient;

    @Test
    void getProductPrice() {
        // Given
        productClient = mock(RestProductClient.class);
        given(productClient.getProduct(1L)).willReturn(new ProductDto(
                1L,
                "iPhone 12",
                "Apple iPhone 12 64GB",
                1000000
        ));
        productService = new ProductService(productClient);


        // When
        var result = productService.getProductPrice(1L);

        // Then
        assertEquals(1000000, result);
    }

}
```

### MockBean 을 활용한 통합 테스트

스프링 어플리케이션에 대한 통합 테스트를 수행할 경우, `@MockBean` 을 활용해서 협력 객체를 Mocking 할 수 있습니다.

```java
@SpringBootTest
class ProductServiceIntegrationTest {

    @Autowired
    private ProductService service;

    @MockBean
    private RestProductClient client;

    @Test
    void getProductPrice() {
        // Given
        given(client.getProduct(1L)).willReturn(new ProductDto(
                1L,
                "iPhone 12",
                "Apple iPhone 12 64GB",
                1000000
        ));

        // When
        var result = service.getProductPrice(1L);

        // Then
        assertEquals(1000000, result);
    }
}
```

### 인터페이스를 추가하고 직접 테스트 더블 구현하기

{% include image.html url='/assets/image/service-stub-diagram.png' %}

위의 두 방법은 구체 클래스인 `RestProductClient` 를 Mocking 하는 방법이었습니다. 하지만 매 테스트 코드마다 외부 의존성을 가지는 클래스를 mocking 하는 것은 쉽지 않은 일이고, `@MockBean` 을 사용할 경우 매 테스트마다 Application Context 가 재생성되기 때문에 테스트 수행 시간이 길어질 수 있습니다. 이런 경우에는 인터페이스를 추가하고 이를 구현하는 테스트 더블 구현체를 만들어서 테스트를 수행하는 것도 하나의 방법이 될 수 있습니다.

```java
public interface ProductClient {
    ProductDto getProduct(Long id);
}

@Profile({"dev", "production"})
@Component
public class RestProductClient implements ProductClient {

    private final RestClient client;

    public RestProductClient(@Value("${product.host}") String host) {
        this.client = RestClient.create(host);
    }

    @Override
    public ProductDto getProduct(Long id) {
        return client.get()
                .uri("/products/{id}", id)
                .retrieve()
                .body(ProductDto.class);
    }
}

// 테스트 환경에서만 사용할 Stub 구현체
@Profile("test")
@Component
public class StubProductClient implements ProductClient {

    @Override
    public ProductDto getProduct(Long id) {
        return new ProductDto(id, "Product " + id, "Description " + id, 100 * id.intValue());
    }

}
```

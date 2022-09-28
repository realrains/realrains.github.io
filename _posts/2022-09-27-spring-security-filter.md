---
layout: post
title: 스프링 시큐리티 - Filter
date: 2022-09-27
---

[참고 문서 : Spring Security / Servlet Applications / Architecture](https://docs.spring.io/spring-security/reference/servlet/architecture.html)

스프링 시큐리티의 기능은 서블릿 필터 (Servlet Filter) 를 기반으로 구현되어 있다. 그렇다면 이 `Filter` 란 뭘까? 스프링이 아닌 기본적인 서블릿 어플리케이션으로 돌아가 보자.

```java
// https://github.com/realrains/my-servlet-sample

@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    @Override
    public void doGet(HttpServletRequest request, HttpServletResponse response) throws IOException {
        response.getWriter().print("Hello Servlet!");
        System.out.println("HelloServlet.doGet");
    }
}
```

서블릿 어플리케이션에서 위와 같은 서블릿을 선언하면 톰캣과 같은 서블릿 컨테이너가 request path 를 확인한후 적절한 서블릿을 찾아 서블릿에서 반환하는 값을 클라이언트에 되돌려 주게 된다.

그러나 특정한 서블릿으로 요청이 전달되기 전에 일괄적으로 수행해야할 작업이 필요할 수 있다. 클라이언트의 요청 로그를 남긴다거나, 요청에 담긴 데이터의 인코딩을 변경하거나, 요청 헤더에 담긴 토큰과 같은 인증정보를 확인하고 요청을 승인할 것인지 말것인지 결정하는 작업이 있을 수 있다.

이러한 작업들을 각 서블릿에서 직접 구현하게 되면 번거롭기도 하고 단일 책임 원칙을 위반하기 때문에 복잡도가 증가하고 테스트가 어려워질 수밖에 없다. 따라서 서블릿 외부에서 공통기능을 핸들링 한후 요청을 전달해주는것이 좋은데 그러한 상황에서 사용할 수 있는것이 바로 `Filter` 이다.


### Filter

[JavaDoc (Jakarta EE) - Filter](https://jakarta.ee/specifications/servlet/5.0/apidocs/jakarta/servlet/filter)

```java
public interface Filter {

    default public void init(FilterConfig filterConfig) throws ServletException {}

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException;

    default public void destroy() {}
}
```
`Filter` 는 리소스에 대한 접근을 필터링하는 작업을 수행하는 객체를 위한 인터페이스이다. 필터가 인스턴스화 될때 서블릿 컨테이너가 호출하는 `init` 메서드, **요청/응답 체인**에서 해당 필터를 거쳐갈 때 호출되는 `doFilter`, 필터의 작업이 종료되거나 중단될 때 clean-up 작업을 수행하는 `destroy` 메서드로 구성되어 있다.

```java
@WebFilter(filterName = "LoggingFilter", urlPatterns = "/*")
public class LoggingFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("LoggingFilter.doFilter Starts");
        chain.doFilter(request, response);
        System.out.println("LoggingFilter.doFilter Ends");
    }
}
```

간단히 요청 전후로 콘솔로그를 프린트하도록 하는 `LoggingFilter` 를 구현해보았다. `GET /hello` 요청을 서버로 전송하면 서버측 콘솔에서는 아래와 같은 결과를 확인할 수 있다.

```
LoggingFilter.doFilter Starts
HelloServlet.doGet
LoggingFilter.doFilter Ends
```

이때 등장하는 체인이란 무엇일까? 단어의 의미에서 유추할 수 있듯 연속된 필터로 이루어진 체인을 말한다. `doFilter` 메서드에 파라미터로 주어지는 `FilterChain` 이 바로 그것이다. 필터는 1개 이상이 존재할 수 있으며 각각의 필터는 맡은 작업을 수행하고 작업이 다음 필터에서 수행될 수 있도록 제어권을 넘겨주면서 최종적으로 서블릿까지 도달한다.

{% include image.html url='/assets/image/filterchain.png' description='https://docs.spring.io/spring-security/reference/_images/servlet/architecture/filterchain.png' alt='filter chain' %}

```java
@WebFilter(filterName = "ResponseAppendFilter", urlPatterns = "/*")
public class ResponseAppendFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("ResponseAppendFilter.doFilter Starts");
        chain.doFilter(request, response);
        response.getWriter().append(" ResponseAppendFilter");
        System.out.println("ResponseAppendFilter.doFilter Ends");
    }
}
```

response 에 특정 문자열을 추가하는 `ResponseAppendFilter` 를 추가하고 이전과 동일하게 `GET /hello` 요청을 날려보면 서버 측에서 아래와 같은 로그를 확인할 수 있다.

```
ResponseAppendFilter.doFilter Starts
LoggingFilter.doFilter Starts
HelloServlet.doGet
LoggingFilter.doFilter Ends
ResponseAppendFilter.doFilter Ends
```

`ResponseAppendFilter`, `LoggingFilter` 순으로 로그가 찍히는 것을 확인할 수 있고 이때 `Filter` 내부에서 디버그를 통해 파라미터로 전달된 `FilterChain` 를 확인해보면

{% include image.html url='/assets/image/filterchain-debug.png' description='Filter 로 전달된 FilterChain, ApplicationFilterChain 은 Tomcat의 구현체이다' alt='FilterChain Instance' %}

로그에서 확인한 순서대로 필터 설정을 보관하고 있음을 확인할 수 있다. `Filter` 가 서블릿 request 와 response 를 수정할 수 있다는 것에서 알 수 있듯이 순서가 앞선 필터는 그 뒤의 필터에 영향을 끼칠수 있게 되므로 `FilterChain` 에 정의된 **필터의 순서는 웹 어플리케이션의 동작에 매우 큰 영향을 미칠 수 있음에 유의하여야 한다.**


### DelegatingFilterProxy

`Filter` 는 Java EE 스펙이고 이는 Tomcat 같은 서블릿 컨테이너 영역에서 사용하는 객체이다. 위에서 살펴봤던 방식처럼 표준적인 방식으로 구현, 등록되었을 경우 해당 필터는 당연히 스프링 어플리케이션 컨텍스트 외부의 객체이므로 스프링 빈을 사용할 수 없다.

이를 해결하기 위해 spring-web 모듈에서는 `DelegatingFilterProxy` 라는 `Filter` 구현체를 제공한다. 해당 객체가 Tomcat 과 같은 서블릿 컨테이너와 스프링 어플리케이션 컨텍스트 사이의 일종의 브릿지 역할을 함으로서 스프링 빈을 사용하는 필터를 구현할 수 있게 해준다.

{% include image.html url='/assets/image/delegatingfilterproxy.png' description='https://docs.spring.io/spring-security/reference/_images/servlet/architecture/delegatingfilterproxy.png' alt='DelegatingFilterProxy' %}

`DelegatingFilterProxy` 는 생성시에 빈 필터 이름을 파라미터로 받는다. 내부 코드를 보면 `initDelegate` 에서 해당 빈 이름으로 어플리케이션 컨텍스트로부터 빈을 가져오고 동작을 위임할 `Filter` 로 세팅한다. 이후 `doFilter` 에서 빈으로 선언된 필터 객체에게 동작을 위임하는 것을 확인할 수 있다.

```java
// DelegatingFilterProxy.class 구현 일부
protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
    String targetBeanName = getTargetBeanName();
    Assert.state(targetBeanName != null, "No target bean name set");
    // 어플리케이션 컨텍스트에서 필터 빈을 가져온다.
    Filter delegate = wac.getBean(targetBeanName, Filter.class);
    if (isTargetFilterLifecycle()) {
        delegate.init(getFilterConfig());
    }
    return delegate;
}

@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {

    // Lazily initialize the delegate if necessary.
    Filter delegateToUse = this.delegate;
    if (delegateToUse == null) {
        synchronized (this.delegateMonitor) {
            delegateToUse = this.delegate;
            if (delegateToUse == null) {
                WebApplicationContext wac = findWebApplicationContext();
                if (wac == null) {
                    throw new IllegalStateException("No WebApplicationContext found: " +
                            "no ContextLoaderListener or DispatcherServlet registered?");
                }
                delegateToUse = initDelegate(wac);
            }
            this.delegate = delegateToUse;
        }
    }

    // 필터 빈에 doFilter 동작을 위임한다.
    invokeDelegate(delegateToUse, request, response, filterChain);
}

protected void invokeDelegate(
        Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {

    delegate.doFilter(request, response, filterChain);
}
```

### FilterChainProxy

스프링 시큐리티에서는 위에서 설명한 `DelegatingFilterProxy` 에서 필터 동작을 위임할 빈 필터를 `FilterChainProxy` 라는 클래스로 제공한다. `FilterChainProxy` 는 내부에 `SecurityFilterChain` 을 구현한 필터들을 리스트로 가지고 있는데, `FilterChainProxy` 가 `doFilter` 동작을 위임받으면 다시 내부의 `SecurityFilterChain` 리스트들에게 필터링 동작을 위임하게 된다. 이러한 구조를 그림으로 도식화하면 다음과 같다.

{% include image.html url='/assets/image/securityfilterchain.png' description='https://docs.spring.io/spring-security/reference/_images/servlet/architecture/securityfilterchain.png' alt='SecurityFilterChain' %}

스프링 시큐리티 의존성을 추가한 뒤, 실제로 `DelegatingFilterProxy` 클래스에서 delegate filter 로 어떤 빈이 사용되는지 브레이크 포인트를 걸고 확인해보면 아래와 같이 `FilterChainProxy` 가 할당된 것을 볼 수 있고, `FilterChainProxy.filterChains` 에는 스프링 시큐리티에서 기본적으로 제공하는 `SecurityFilterChain` 한개가 들어 있는것을 볼수 있고, 해당 필터 체인에서 제공하는 필터 기능들을 `SecurityFilterChain.filters` 에서 확인할 수 있다. `FilterChainProxy` 가 스프링 시큐리티에서 제공하는 서블릿 필터링 기능의 시작점인 셈이다.

{% include image.html url='/assets/image/filterchainproxy-debug-point.png' description='' alt='DelegateingFilterProxy Debug Point' %}

{% include image.html url='/assets/image/securityfilterchain-list.png' description='filterChains 에 SecurityFilterChain 이 등록되어 있다.' alt='SecurityFilterChainList' %}

```java
@Configuration
public class WebSecurityConfig {
    @Bean
    @Order(1)
    public SecurityFilterChain helloSecurityFilterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.antMatcher("/hello/**");
        return httpSecurity.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain globalSecurityFilterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity.antMatcher("/**");
        return httpSecurity.build();
    }
}
```

새로운 `SecurityFilterChain` 을 정의하려면 위와 같이 `HttpSecurity` 를 주입받아 빈으로 생성하면 된다. (`WebSecurityConfigurerAdapter` 의 확장 클래스를 통해 설정하는 방법은 스프링 시큐리티 5.7.3 기준으로 deprecated 된 상태이다.)

예시에서는 `/hello/**` 경로의 서블릿 요청을 핸들링하는 `helloSecurityFilterChain` 과 모든 경로의 요청을 핸들링하는 `globalSecurityFilterChain` 을 선언했다.

`FilterChainProxy` 에 여러개의 `SecurityFilterChain` 이 존재할 경우 `FilterChainProxy`는 조건에 맞는 필터 체인 중 가장 먼저 매칭되는 `SecurityFilterChain`을 사용하게 된다. 따라서 각 빈의 순서를 `@Order`를 통해 알맞게 정의하는게 중요하다.

{% include image.html url='/assets/image/multiple-securityfilterchain.png' description='/hello 요청에 대해 0번째 SecurityFilterChain이 매칭된다' alt='Multiple SecurityFilterChain' %}


---
title: "Filter와 Interceptor를 제대로 구분해 사용하는 법"
excerpt: "비슷한 두 기능의 차이에 대하여"
comments: true

categories:
  - Spring
last_modified_at: 2020-11-11
---

### 개요
Spring 기반 web application을 공부하면서 filter를 검색하면 같이 나오는 게 interceptor입니다.    
filter는 자바 서블릿에서 소개되는 기능이고, interceptor는 Spring에서 제공하는 기능입니다. 둘 다 요청과 응답 정보를 변경하거나 흐름을 처리할 수 있습니다.   
filter와 interceptor를 사용하면 공통적인 책임을 분리할 수 있어 나중에 변경 사항이 생겨도 수정할 부분이 적어져 유지보수에 탁월하다는 장점이 있습니다. 그렇다면 둘의 차이점은 정확히 뭘까요.

<br>

### 설정
filter의 경우 톰캣을 사용한다면 web.xml에 설정하는 게 보통 알려진 방법입니다. 하지만 Spring Boot부터는 web.xml이 없어져서 FilterRegistrationBean으로 등록하거나 @WebFilter 어노테이션을 적용할 수 있습니다.
interceptor는 Spring의 context (application context 또는 servlet context)에서 HandlerMapping bean을 통해 설정할 수 있습니다.

<br>

### 작동 순서
Spring MVC 패턴의 경우 Dispatcher Servlet을 기준으로 filter는 Dispatcher 앞단에서, interceptor는 Dispatcher가 실행된 후에 작동합니다. 
이렇게 봤을 때 filter는 서버와 클라이언트가 주고 받는 HTTP 요청과 응답에 대해 일괄적인 전처리/후처리를 하고, interceptor는 Dispatcher Servlet이 호출하는 Controller와 View page 사이에 오가는 신호를 처리한다고 보면 됩니다.

<br>

### 선호되는 사용법 
Spring 공식 도큐먼트에 HandlerInterceptor를 설명하는 페이지에 짧게 나마 filter와 interceptor의 차이를 말합니다. 관심 가시면 직접 읽어보셔도 좋습니다.      
[HandlerInterceptor](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/HandlerInterceptor.html)

해당 글에서 filter는 interceptor보다 더 강력한 기능을 제공한다고 합니다. 요청 데이터 변경, multipart나 GZIP 압축 같은 view, 이미지 컨텐츠에 대한 처리에 유용하다고 합니다.    
위 내용과 부합할 것 같은 filter 구현 클래스는 아래와 같습니다.    
* CharacterEncodingFilter 클래스
* FormContentFilter 클래스
* ForwardedHeadersFilter 클래스
* ShallowEtagHeaderFilter 클래스
* CorsFilter 클래스

<br>

interceptor는 허가나 승인 확인, handler 관련 전처리에 유용하다고 합니다.
* LocaleChangeInterceptor 클래스
* MappedInterceptor 클래스
* ThemeChangeInterceptor 클래스
* UserRoleAuthorizationInterceptor 클래스
* WebContentInterceptor 클래스
* WebRequestHandlerInterceptorAdapter 클래스

filter와 interceptor 모두 이미 구현된 클래스와 인터페이스를 사용해도 되고, 직접 커스터마이징해 사용할 수 있습니다.

<br>

### 예외 처리 차이
filter는 예외 처리를 위해 Dispatcher type에 error를 지정해야 합니다. 그리고 web.xml에 <error-page>에 대한 설정을 적용해 매핑된 예외 처리 페이지로 돌려야 하는 번거로움이 있습니다.
interceptor는 예외 처리 방법은 각 예외 별 처리, Controller 별 처리, 전역 처리가 있습니다. 보통 전역 예외 처리는 컨트롤러에서 직접 하기 보다 Dispatcher Servlet에서 바로 @ControllerAdvice를 이용할 수 있습니다. 

<br>

### 구현된 인터페이스
Filter 인터페이스에는 세 가지 메서드로 구성되어 있습니다.
* init(FilterConfig config)    
필터의 초기화를 담당하며 처음 Filter 객체를 생성할 대 한 번 작동합니다. 인자로 받는 FilterConfig 객체를 통해 여러 설정값을 넘겨 받을 수 있습니다.

* doFilter(ServletRequest request, ServletResponse response, FilterChain chian)     
클라이언트 요청이나 응답이 있을 때마다 동작하는 메서드입니다. 3개의 메서드 중 유일하게 여러 번 동작합니다.
request와 response로 요청과 응답을 조작하고 FilterChain으로 다른 filter나 서블릿으로 이동하게 해줍니다.

* destroy()    
모든 동작이 끝나고 난 후 필터 객체를 제거합니다.

<br>

interceptor는 HandlerInterceptorAdapter 추상 클래스나 HandlerInterceptor 인터페이스를 사용해 구현할 수 있습니다. 보통 Override만 하면 바로 사용할 수 있는 추상 클래스를 더 선호합니다. 두 방법 모두 3가지 메서드를 사용할 수 있습니다.
* preHandle()
Controller로 요청이 들어가기 전에 수행

* postHandle()
Controller 내 메서드 처리가 끝난 후 view로 가기 전 수행

* afterCompletion()
화면 처리까지 끝난 다음 수행

<br>

### 결론
결론적으로 filter와 interceptor는 일괄적 전처리/후처리라는 공통 기능을 수행하지만 이용 목적에 따라 사용할 수 있습니다.
filter는 encoding, 헤더, HTML 변환, multipart, GZIP, image 등 요청으로 들어오는 데이터의 변환에, interceptor는 테마 변경, 사용자 인증, 허가 및 승인 확인에 쓰이면 좋습니다.

<br>

### 출처
* 최범균의 JSP 2.3 웹프로그래밍 기초부터 중급까지
* https://spring.io/blog/2013/11/01/exception-handling-in-spring-mvc
* https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/servlet/HandlerInterceptor.html
* https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/filter/package-summary.html

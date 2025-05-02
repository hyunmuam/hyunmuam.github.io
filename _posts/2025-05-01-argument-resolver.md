---
title: Controller에서는 어떻게 파라미터를 받을까?
date: 2025-05-01 19:00:00 +0900
categories: [Spring]
tags: [spring]
image:
  path: /assets/img/posts/2025-05-01-argument-resolver/01.png
---
Spring에서 `Controller`는 사용자의 요청을 받아서 처리하고, 그 결과를 클라이언트에 반환하는 역할을 한다. 

요청 데이터를 컨트롤러 메소드에 전달하는 방법에는 여러 가지가 있다.
`@RequestParam` 애노테이션을 사용하여 요청 파라미터(즉, 쿼리 파라미터나 폼 데이터)를 컨트롤러의 메서드 매개변수에 바인딩할 수 있다. `@RequestBody`를 이용하여 요청 본문(Body)를 읽어 객체로 역직렬화 할 수 있다.URL 경로에 포함된 값을 메서드의 매개변수로 받아올 때는 `@PathVariable`를 사용한다.

이처럼 우리는 별도의 파싱이나 변환 작업 없이, 다양한 요청 데이터를 컨트롤러 메서드의 파라미터로 손쉽게 받을 수 있다. 이러한 자동 바인딩의 핵심에는 바로 `ArgumentResolver`가 있다. `ArgumentResolver`는 Spring MVC에서 컨트롤러 메서드의 파라미터에 요청 데이터를 자동으로 변환하여 주입해주는 역할을 한다.

이번 포스트에서는 이 `ArgumentResolver`에 대해 정리해보자.

## ArgumentResolver가 없다면?
---
`ArgumentResolver`를 사용하지 않을 경우를 가정해서 해보자.

아래 예시는 회원 정보, 클라이언트의 IP 정보, 그리고 쿼리 파라미터가 필요한 요청을 처리하는 컨트롤러 코드다. 이 경우, 모든 정보를 직접 HttpServletRequest에서 꺼내와야 합
```java
@PostMapping("/users")
public ResponseEntity<Void> create1(HttpServletRequest request) {
  Long userId = Long.valueOf(request.getParameter("userId"));
  String param = request.getParameter("param");

  // userId와 ipAddress를 사용하여 요청 처리

  return ResponseEntity.ok().build();
}
```
서블릿 컨테이너(Servlet Container)에서 만들어주는 `HttpServletRequest`를 통해서 요청에 대한 정보들을 가져올 수 있는 것을 확인할 수 있다. 그리고 생각보다 어렵지도 않다.
하지만 모든 엔드포인트마다 각각 구현하려면 많은 중복이 발생하게 될 것이다.

그럼 다시 `ArgumentResolver`를 사용한 코드를 보면 훨씬 더 깔끔해진 것을 확인할 수 있다.
```java
@PostMapping("/users")
public ResponseEntity<Void> create2(
  @RequestParam("userId") Long userId,
  @RequestParam("param") String param) {

  // userId와 ipAddress를 사용하여 요청 처리

  return ResponseEntity.ok().build();
}
```

## ArgumentResolver는 어떻게 동작할까?
---
Spring MVC에서 클라이언트의 요청이 컨트롤러 메서드에 도달하기까지는 여러 단계가 있다. 이 과정에서 `ArgumentResolver`가 파라미터 바인딩을 담당한다. 전체 동작흐름을 간단히 정리하면 다음과 같다.

![Desktop View](/assets/img/posts/2025-05-01-argument-resolver/02.png){: width="972" height="589" }
_Spring MVC에서 요청 흐름도_

1. 요청
  - `DispatcherServlet`이 HTTP 요청을 받는다.
2. 핸들러(Controller)찾기
  - `DispatcherServlet`은 핸들러 매핑을 사용해, 해당 요청을 처리할 컨트롤러(핸들러)를 찾는다.
3. 핸들러 어뎁터 조회
  - 핸들러 매핑이 반환한 컨트롤러를 실제로 실행할 수 있는 핸들러 어뎁터를 찾는다.
4. 핸들러 어뎁터 실행
  - 디스패처 서블릿이 조회한 핸들어 어뎁터를 실행하면서 핸들러 정보도 함께 넘겨준다.
5. **ArgumentResolver를 통해 파라미터 바인딩**
  - `HandlerAdapter`가 컨트롤러 메서드를 호출할 때, 각 파라미터에 어떤 값을 넣어줘야하는지 결정해야한다. 이 때 `ArgumentResolver`들이 순차적으로 실행되어, `@RequestParam`, `@RequestBody`, `@PathVariable` 등 어노테이션이 붙은 파라미터에 대해 요청 데이터(쿼리 파라미터, 경로 변수, 본문 등)를 알맞게 변환하여 컨트롤러 메서드의 인자로 주입한다.
6. 핸들러 호출
  - 컨트롤러 메서드 실행

이처럼 `ArgumentResolver`는 `HandlerAdapter` 내부에서 컨트롤러 메서드의 각 파라미터를 해석하고 값을 주입하는 역할을 담당한다. **핸들러 매핑과 핸들러 어댑터는 컨트롤러를 찾고 실행**하는 큰 틀을 담당하고, `ArgumentResolver`는 그 안에서 **파라미터 바인딩**이라는 세부적인 역할을 맡는다고 볼 수 있다.

## ArgumentResolver 구조
스프링에서는 30개가 넘는 `ArgumentResolver`를 기본으로 제공한다. 이것이 바로 우리가 따로 코들르 작성하지 않아도 컨트롤러 메서드에서 요청에 대한 데이터를 받을 수 있는 이유이다.

이 ArgumentResolver들은 `HandlerMethodArgumentResolver`를 상속받아서 구현된다.
우리는 이 인터페이스를 확장해서 원하는 `ArgumentResolver` 를 만들 수도 있다.

```java
public interface HandlerMethodArgumentResolver {

  /*
  - 이 메서드는 해당 리졸버가 특정 메서드 파라미터를 지원하는지 여부를 반환한다.
  - 즉, 어떤 파라미터에 대해 이 리졸버가 동작할지 조건을 정의하는 곳이다.
  - 예를 들어, 파라미터에 특정 어노테이션이 붙어 있거나, 특정 타입일 때만 true를 반환하도록 구현할 수 있다.
  */
  boolean supportsParameter(MethodParameter parameter);

  /*
  - supportsParameter에서 true가 반환된 파라미터에 대해 호출된다.
  - 이 메서드는 실제로 파라미터에 주입할 값을 생성하거나 변환하는 로직을 구현한다.
  - 예를 들어, 요청 객체에서 값을 꺼내거나, 세션에서 객체를 가져오거나, 커스텀 로직을 통해 파라미터 값을 만들어 반환할 수 있다.
  - 반환된 객체가 컨트롤러 메서드의 해당 파라미터로 전달된다.
  */
  @Nullable
  Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
      NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}
```
## 직접 사용해보기
---
> 전체 프로젝트 코드는 [Github](https://github.com/tjvm0877/blog-code/tree/main/argument-resolver)에 있으니 참고해주세요.
{: .prompt-info }
이제 `ArgumentResolver`를 직접 구현해보도록 하자.

상황은 다음과 같다. `Authorization`헤더에 있는 JWT로부터 유저정보를 가져오고 추가로 요청한 회원의 IP를 DTO형태로 받을 수 있도록 하고 싶다.

### 기존 코드
코드 자체는 간단하지만 JWT 관련로직, IP 추출 로직이 각 컨트롤러 메서드에서 중복될 가능성이 있다.
앞서 설명명한 ArgumentResolver를 이용하여 이 중복되는 로직을 줄여보도록하자.
```java
@PostMapping("/users")
public ResponseEntity<Void> create1(HttpServletRequest request) {

  String token = request.getHeader("Authorization");
  jwtProvider.isValidToken(token);

  Long userId = jwtProvider.getMemberIdFromToken(token);
  String ipAddress = request.getRemoteAddr();

  // userId와 ipAddress를 사용하여 요청 처리

  return ResponseEntity.ok().build();
}
```
{: file='UserControllerV1'}

### ArgumentResolver 구현
```java
@RequiredArgsConstructor
public class UserArgumentResolver implements HandlerMethodArgumentResolver {

  private final JwtProvider jwtProvider;

  private static final String AUTHORIZATION_HEADER = "Authorization";

  @Override
  public boolean supportsParameter(MethodParameter parameter) {
    // 컨트롤러 메서드에 UserDto 타입이 있는지 확인
    // True를 반환할 시 resolveArgument() 실행
    return parameter.getParameterType().equals(UserDto.class);
  }


  // 컨트롤러에서 반복된 HTTP 헤더로부터 JWT관련 로직, 클라이언트 IP 가져오기 로직을 넣어준다. 
  // 최종적으로 UserDto 를 생성해서 반환
  @Override
  public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
    NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

    HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
    String token = request.getHeader(AUTHORIZATION_HEADER);
    jwtProvider.isValidToken(token);

    Long userId = jwtProvider.getUserIdFromToken(token);
    String ip = request.getRemoteAddr();
    return new UserDto(userId, ip);
  }
}
```
{: file='UserArgumentResolver'}

### ArgumentResolver 등록
`WebMvcConfigurer` 를 구현한 클래스에서 방금 만든 UserArgumentResolver 를 Argument Resolver로 등록한다.
```java
@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {

  private final JwtProvider jwtProvider;

  @Override
  public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {
    resolvers.add(userArgumentResolver());
  }

  @Bean
  public UserArgumentResolver userArgumentResolver() {
    return new UserArgumentResolver(jwtProvider);
  }
}
```
{: file='WebConfig'}

### Controller에 적용
```java
@PostMapping("/users")
public ResponseEntity<Void> create1(UserDto userDto) { // UserArgumentResolver에서 UserDto를 생성해서 넘겨준다.
  // userDto.id, userDto.ip 를 사용하여 요청 처리
  // ...
  return ResponseEntity.ok().build();
}
```
{: file='UserControllerV2'}

### 어노테이션으로 바인딩하기
@RequestParam, @RequestBody 처럼 클래스가 아닌 어노테이션 여부로 자동으로 바인딩 되도록 만들고 싶을 수 있다.
이를 커스텀 어노테이션을 통해서 만들어보자.
```java
@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface UserInfo {
}
```
{: file='UserInfo'}

```java
@Override
public boolean supportsParameter(MethodParameter parameter) {
  // 컨트롤러 메서드에 방금 선언한 UserInfo 어노테이션의 유무를 바인딩 조건으로 만들어준다.
  return parameter.hasParameterAnnotation(UserInfo.class);
}
```
{: file='UserArgumentResolverV2'}

```java
@PostMapping("/users")
public ResponseEntity<Void> create1(@UserInfo UserDto userDto) {

  // userDto.id, userDto.ip 를 사용하여 요청 처리
  // ...

  return ResponseEntity.ok().build();
}
```
{: file='UserControllerV3'}
위와 같이 `@UserInfo`가 붙은 경우에만 `UserDto` 가 바인딩 되도록 만들 수 있다.

### PageableHandlerMethodArgumentResolver
여담으로 [저번 포스트](https://tjvm0877.github.io/posts/spring-data-jpa-pagination/)에서 ArgumentResolver를 통해서 Pageable을 받을 수 있다고 했다.
해당 동작을 가능하도록 만들어주는 것이 바로 `PageableHandlerMethodArgumentResolver`이다.
ArgumentResolver의 구조를 위에서 보았기 때문에 구현체만 봐도 어떻게 동작하는지 알 수 있다.
```java
public boolean supportsParameter(MethodParameter parameter) {
  // 컨트롤러 메서드에서 Pageable을 파라미터로 받는지 확인
  return Pageable.class.equals(parameter.getParameterType());
}

public Pageable resolveArgument(MethodParameter methodParameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) {
  // 요청에서 Page번호를 받는다.
  String page = webRequest.getParameter(this.getParameterNameToUse(this.getPageParameterName(), methodParameter));

  // 요청에서 Page 크기를 가져온다.
  String pageSize = webRequest.getParameter(this.getParameterNameToUse(this.getSizeParameterName(), methodParameter));

  // 요청에서 정렬 기준을 가져온다.
  Sort sort = this.sortResolver.resolveArgument(methodParameter, mavContainer, webRequest, binderFactory);

  // 요청에서 가져온 정보로 Pagealbe을 만들어주고 return해준다.
  Pageable pageable = this.getPageable(methodParameter, page, pageSize);
  if (!sort.isSorted()) {
      return pageable;
  } else {
      return (Pageable)(pageable.isPaged() ? PageRequest.of(pageable.getPageNumber(), pageable.getPageSize(), sort) : Pageable.unpaged(sort));
  }
}
```

클라이언트에서 `?page=0&size=10&sort=name,asc`와 같은 파라미터를 전송할 경우 컨트롤러 메서드에서는 `Pageable`을 바로 받아올 수 있다. 

## 마무리
---
이번 글에서는 Spring MVC에서 컨트롤러가 다양한 방식으로 요청 파라미터를 받을 수 있는 원리와, 그 핵심에 있는 ArgumentResolver의 구조와 동작 방식을 살펴보았다.
`ArgumentResolver` 덕분에 우리는 반복적이고 번거로운 파라미터 추출 및 변환 코드를 직접 작성하지 않고도, 어노테이션이나 타입만으로 간편하게 요청 데이터를 받을 수 있었다.

또한, 직접 `ArgumentResolver`를 구현하고 등록함으로써, 프로젝트의 요구에 맞는 커스텀 바인딩 로직을 적용할 수 있다는 점도 확인했다.
이처럼 `ArgumentResolver`는 스프링 MVC의 유연함과 확장성을 뒷받침하는 중요한 요소로, 실무 개발에서 코드의 중복을 줄이고 유지보수를 용이하게 해준다.

실제 프로젝트에서도 반복되는 파라미터 처리 로직이 있다면 컨트롤러 코드를 더욱 깔끔하게 유지하고, 스프링이 제공하는 강력한 자동 바인딩의 이점을 최대한 누릴 수 있도록 커스텀 `ArgumentResolver`를 적극적으로 활용해봐야겠다.

## 참고
---
[스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술 \| 인프런](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-mvc-1)

[스프링 공식문서 - Handler Methods](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods.html)

---
title: "쿠키부터 JWT까지 흐름을 파악해보자"
date: 2025-06-16 17:00:00 +0900
categories: [Security]
tags: [security, session, cookie, jwt]
image:
  path: /assets/img/posts/2025-06-16-cookie-to-jwt/01.png
mermaid: true
---
최근 포스팅에서 인증/인가와 관련된 내용들을 많이 다뤄왔다. 문득 이런 인증/인가 방식의 흐름에 대해서 의문점을 가지게 되었다. 이번 포스팅에서는 가장 많이 언급되는 쿠키, 세션, JWT 이 세가지 키워드가 어떤 식으로 연결되었는지 확인해보고자 한다.

## HTTP의 무상태성과 인증의 필요성
---
HTTP(Hypertext Transfer Protocol)프로토콜은 웹에서 데이터를 주고받는 서버-클라이언트 모델의 프로토콜이다. 클라이언트-서버 프로토콜이란 수신자 측에 의해 요청이 초기화되는 프로토콜을 의미한다.

HTTP 프로토콜은 상태를 관라하지 않는 **무상태성(Stateless)**이라는 특징을 가지고 있다. 서버와 클라이언트 간에 요청(request)과 응답(response)를 교환하는 동안 상태를 관리하지 않기 때문에, 이전에 보낸 응답에 대해 저장하지 않는다. 이러한 특징은 서버의 메모리 부담을 줄이고 많은 데이터를 빠르고 정확하게 처리할 수 있게 해주었다.

하지만 웹이 발전함에 따라 요청과 요청 간의 상태를 유지할 필요가 생기게 되었다. **무상태성이라는 HTTP 프로토콜의 중요한 특징 중 하나가 발목을 잡게 된 것이다.**

이 문제의 대표적인 예가 바로 사용자 인증이다. 상태가 유지되지 않으면 사용자는 페이지를 이동할 때마다 계속 새로운 로그인을 해야 하는 문제가 발생하게 된다. 또한 온라인 쇼핑몰의 장바구니 기능도 상품을 장바구니에 담았다가 다른 페이지로 이동하면 장바구니가 비워지는 상황이 발생할 수 있다.

하지만 현재 우리가 사용하는 웹 서비스에서는 이러한 문제를 전혀 겪지 않는다. 바로 무상태성이라는 특징을 보완할 다양한 기술들이 개발되었기 때문이다.

## 쿠키(Cookie) - 상태 관리의 시작
---
위 문제를 해결하게 위해 등장한 첫 번째 해결책이다. 쿠키는 서버가 사용자의 웹 브라우저에 전송하는 작은 데이터 조각이다. 브라우저는 이 데이터 조각들을 저장해 놓았다가 동일한 서버에 재 요청 시 저장된 데이터를 함께 전송한다. 쿠키는 두 요청이 동일한 사용자(브라우저)에서 들어왔는지 아닌지를 판단할 수 있게 되었다.

### 쿠키의 동작 원리
쿠키는 `Set-Cookie`와 `Cookie` 헤더를 통해 동작한다. 아래 시퀀스 다이어그램은 쿠키의 기본적인 흐름을 보여준다.
```mermaid
%%{init: {
  'theme': 'default',
  'themeVariables': {
    'actorBkg': 'transparent',
    'actorBorder': '#8b949e',
    'actorTextColor': 'currentColor',
    'actorLineColor': '#8b949e',
    'signalColor': 'currentColor',
    'signalTextColor': 'currentColor',
    'primaryBorderColor': '#8b949e',
    'primaryTextColor': 'currentColor',
    'lineColor': '#8b949e'
  },
  'sequence': {
    'showSequenceNumbers': false,
    'mirrorActors': false
  }
}}%%

sequenceDiagram
  participant C as 클라이언트
  participant S as 서버
  
  C->>S: POST /login
  activate S
  S-->>C: Set-cookie: user=user01
  deactivate S

  C->>S: GET /members/info<br>cookie: user=user01
  activate S
  S-->>C: 유저 정보
  deactivate S
```
1. 클라이언트가 로그인 요청을 보내면, 서버는 응답에 Set-Cookie 헤더를 포함해 쿠키를 발급
  ```
  HTTP/1.0 200 OK
  Content-type: text/html
  Set-Cookie: user=user01
  ```
  {: file='HTTP 응답 메시지'}
2. 이후 클라이언트는 모든 요청에 해당 쿠키를 Cookie 헤더에 담아 서버로 전송
  ```
  GET /members/info HTTP/1.1
  Host: www.example.org
  Cookie: user=user01
  ```
  {: file='HTTP 요청 메시지'}
4. 서버는 쿠키 값을 바탕으로 사용자를 식별하고, 맞춤형 응답을 제공

### 쿠키의 주요 속성
쿠키는 다양한 속성을 통해 동작 범위와 보안을 제어할 수 있다.

`Domain`: 쿠키가 전송될 도메인을 지정, 하위 도메인까지 적용할 수 있음<br>
`Path`: 쿠키가 전송될 URL 경로를 지정, 특정 경로에서만 쿠키가 전송되도록 제한할 수 있음<br>
`Secure`: HTTPS 환경에서만 쿠키가 전송되도록 설정<br>
`HttpOnly`: JavaScript에서 쿠키 접근을 차단하여 XSS 공격을 방지<br>
`SameSite`: 크로스 사이트 요청 시 쿠키 전송을 제한해 CSRF 공격을 방지<br>
`Expires/Max-Age`: 쿠키의 만료 시점을 설정, 명시하지 않으면 세션 쿠키로 동작<br>

Set-Cookie 헤더는 아래와 같이 전송될 수 있다.
```
Set-Cookie: <cookie-name>=<cookie-value>
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>
Set-Cookie: <cookie-name>=<cookie-value>; Expires=<date>
Set-Cookie: <cookie-name>=<cookie-value>; HttpOnly
Set-Cookie: <cookie-name>=<cookie-value>; Max-Age=<number>
Set-Cookie: <cookie-name>=<cookie-value>; Partitioned
Set-Cookie: <cookie-name>=<cookie-value>; Path=<path-value>
Set-Cookie: <cookie-name>=<cookie-value>; Secure

Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Strict
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=Lax
Set-Cookie: <cookie-name>=<cookie-value>; SameSite=None; Secure

// 한번에 여러 속성을 지정할 수도 있다.
Set-Cookie: <cookie-name>=<cookie-value>; Domain=<domain-value>; Secure; HttpOnly
```

### 쿠키의 한계
쿠키는 모든 브라우저에서 지원되며 서버와 클라이언트 간 상태 정보를 자동으로 주고받을 수 있기 때문에 구현이 간단하다. 또한 만료 시간 설정을 통해 일시적 또는 영구적으로 데이터 저장이 가능하다.

하지만 쿠키에는 다음과 같은 중요한 한계점들이 있다. 이러한 한계점들, 특히 보안 문제는 더 안전하고 확장 가능한 상태 관리 방법인 세션의 등장 배경이 되었다.

용량 제한
: 쿠키는 개별적으로 약 4KB의 크기 제한을 가지며, 도메인당 약 50개까지만 생성할 수 있다. 브라우저별로 차이가 있지만, 대부분의 브라우저에서 지원하는 최소한의 제한사항을 준수해야 한다. 이러한 제약으로 인해 대량의 데이터를 저장하기에는 부적합하다.

보안 취약성
: 쿠키는 클라이언트 측에 평문으로 저장되기 때문에 다양한 보안 위험에 노출된다. 특히 PC방이나 공용 컴퓨터 환경에서는 하드디스크에 저장된 쿠키 정보를 쉽게 탈취할 수 있다. 또한 개발자 도구를 통해 쿠키 값을 임의로 변경할 수 있어, memberId=1을 memberId=2로 수정하여 다른 사용자로 위장하는 공격이 가능하다. XSS(Cross-Site Scripting) 공격을 통해서도 JavaScript로 쿠키에 접근하여 정보를 탈취할 수 있다.

네트워크 오버헤드
: 모든 HTTP 요청에 쿠키가 자동으로 포함되어 전송되기 때문에 불필요한 네트워크 트래픽이 발생할 수 있다. 특히 쿠키가 많거나 크기가 클 경우 성능에 영향을 미칠 수 있다.

## 세션(Session) - 보안성의 강화
---
쿠키는 클라이언트 측에 데이터가 저장된다는 구조적 한계로 인해 보안 취약점에 노출될 수 있다. 이를 보완하기 위해 등장한 방식이 바로 세션(Session)이다. 세션은 **민감한 정보를 서버 측에 저장**하고, **클라이언트에는 단순 식별자인 세션 ID만 전달**하는 구조로, 인증과 상태 관리의 보안성을 크게 강화할 수 있다.

이 방식은 민감한 정보를 서버에 저장함으로써 쿠키 탈취, 변조 등의 보안위협을 줄일 수 있다. 또한 세션 만료 및 무효화등 서버가 직접 정보를 관리할 수 있게 되었다.

### 세션의 동작 흐름과 장점
```mermaid
%%{init: {
  'theme': 'default',
  'themeVariables': {
    'actorBkg': 'transparent',
    'actorBorder': '#8b949e',
    'actorTextColor': 'currentColor',
    'actorLineColor': '#8b949e',
    'signalColor': 'currentColor',
    'signalTextColor': 'currentColor',
    'primaryBorderColor': '#8b949e',
    'primaryTextColor': 'currentColor',
    'lineColor': '#8b949e'
  },
  'sequence': {
    'showSequenceNumbers': false,
    'mirrorActors': false
  }
}}%%

sequenceDiagram
  participant C as 클라이언트
  participant S as 서버
  
  C->>S: 사용자 인증 요청
  activate S
  S-->>S: 세션 생성 및 저장
  S-->>C: 세션 ID를 쿠키에 담아 클라이언트로 전송
  deactivate S

  C->>S: 세션 ID를 포함하여 요청 전송
  activate S
  S-->>S: 세션 ID를 통해 사용자 정보 확인
  S-->>C: 유저 정보
  deactivate S
```
1. 사용자 로그인: 사용자가 ID와 비밀번호로 로그인을 시도
2. 세션 생성 및 저장: 서버는 로그인 정보가 유효하면 고유한 세션 ID를 생성하고, 해당 ID와 매핑되는 사용자 정보(회원 ID, 권한 등)를 서버의 메모리나 데이터베이스 같은 세션 저장소에 저장
3. 세션 ID 전송: 서버는 응답 시 Set-Cookie 헤더를 통해 생성된 세션 ID를 클라이언트에 전달
4. 세션 ID 재전송: 클라이언트는 이후의 모든 요청에 이 세션 ID를 쿠키에 담아 서버로 전송
5. 사용자 식별: 서버는 수신한 세션 ID를 통해 세션 저장소에서 사용자 정보를 조회하여 요청을 처리


### 세션 불일치 문제와 확장성 이슈
현대 웹 서비스는 트래픽 분산을 위해 여러 대의 서버를 운영하는 것이 일반적이다. 이때 사용자의 요청이 이전과 다른 서버로 전달되면 해당 서버에는 세션 정보가 없어 인증이 풀리는 **세션 불일치 문제**가 발생할 수 있다. 이 문제를 해결하기 위한 대표적인 방법은 다음과 같다.

Sticky Session
: 로드 밸런서가 특정 사용자의 요청을 항상 첫 세션이 생성된 서버로만 보내도록 고정하는 방식입. 구현이 간단하지만, 특정 서버에 트래픽이 몰릴 수 있고 해당 서버에 장애가 발생하면 세션 정보가 유실되는 단점이 있음.

Session Clustering
: 여러 서버 간에 세션을 실시간으로 복제하여 모든 서버가 동일한 세션 정보를 갖게 하는 방식. 한 서버에 장애가 발생해도 다른 서버에서 세션을 이어갈 수 있지만, 서버 수가 늘어날수록 복제에 따른 성능 부하가 심해지는 단점이 있음.

Session Storage
: 세션을 서버 내부가 아닌 Redis, Memcached와 같은 외부 메모리 기반 저장소에 중앙 집중식으로 관리하는 방식. 모든 서버는 이 외부 저장소를 통해 세션 정보를 공유하므로, 어떤 서버로 요청이 가든 동일한 세션 상태를 유지할 수 있다. 확장성이 뛰어나고 서버의 상태 의존성을 제거(Stateless)할 수 있어 현재 가장 널리 사용되는 해결책.

### 세션의 한계
세션은 쿠키의 보안 문제를 효과적으로 해결했지만  여전히 몇 가지 한계를 가지고 있다.

서버 부하
: 사용자 수가 증가할수록 서버의 메모리 사용량이 늘어나고, 매 요청마다 세션 저장소를 조회해야 하므로 부하가 발생

확장성 제약
: 외부 세션 저장소를 사용하더라도 이 저장소 자체의 성능과 안정성을 관리해야 하는 추가적인 포인트가 생김

상태 유지의 부담
: 서버가 사용자의 상태를 계속 유지해야 하므로, 완전한 무상태(Stateless) 아키텍처를 구현하기 어려움

이러한 세션의 한계점들은 서버의 상태 저장 부담을 없애고 수평적 확장에 더 용이한 **JWT(JSON Web Token)**와 같은 토큰 기반 인증 방식의 등장 배경이 되었다.

## JWT(JSON Web Token) - 확장성 강화
---
세션 기반 인증은 서버가 상태를 관리해야 하므로, 대규모 트래픽이나 마이크로서비스 환경에서 확장성에 한계가 있다. 서버 간 세션 동기화, 외부 저장소 관리, 서버 재시작 시 세션 손실 등 운영상의 복잡성도 증가하게 되었다. 이러한 문제를 해결하고, 서버가 상태를 직접 관리하지 않아도 되는 인증 방식이 필요해졌다.

JWT(JSON Web Token)는 2010년대 들어 API 중심의 서비스, 모바일 앱, 마이크로서비스 아키텍처의 확산과 함께, 무상태(stateless) 인증과 확장성을 동시에 달성하기 위한 표준 토큰 기반 인증 방식으로 등장했다.

### JWT 구조: Header, Payload, Signature
JWT는 세 부분으로 구성된 문자열로, 각 부분은 Base64Url로 인코딩되어 마침표(.)로 구분된다.
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30
```
{: file='JWT 예시'}

#### Header
토큰의 타입(typ)과 서명에 사용된 알고리즘(alg) 정보를 담고 있다.
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### Payload
인증 대상에 대한 정보(클레임, claims)가 담깁니다. 예를 들어 사용자 ID, 권한, 만료 시간 등이 여기에 포함된다.
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "admin": true,
  "iat": 1516239022
}
```
클레임은 크게 세 가지로 분류된다.

Registered Claims
: 표준으로 정의된 클레임 (`sub`, `exp`, `iat` 등)

Public Claims
: 공개적으로 사용 가능한 클레임

Private Claims
: 특정 애플리케이션에서만 사용하는 클레임

### Signature
JWT의 마지막 부분인 Signature(서명)는 토큰의 위변조를 방지하기 위한 역할을 한다.
서명은 Header와 Payload를 각각 Base64Url로 인코딩한 후, 비밀키(혹은 공개키/개인키 쌍)와 함께 지정된 알고리즘(예: HS256, RS256 등)으로 암호화하여 생성된다.

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```
이 서명 덕분에 **서버는 토큰이 발급 이후 변조되지 않았음을 검증할 수 있다.** 만약 클라이언트가 Payload의 내용을 임의로 수정하면, 서명이 일치하지 않아 인증이 거부된다.

### JWT의 주요 장점
위와 같이 간단한 구성만으로도 개발자는 다음과 같은 장점을 취할 수 있게 되었다.

확장성과 무상태성
: JWT는 서버가 상태를 저장하지 않는 무상태(Stateless) 방식으로 동작하여 서버 확장성이 뛰어나다. 클라이언트 인증 정보를 저장하는 세션과 다르게, 서버는 무상태가 되어 수평적 확장이 용이해진다.

데이터 위변조 방지
: Header와 Payload를 가지고 Signature를 생성하므로 데이터 위변조를 막을 수 있다. JWT는 토큰에 대한 기본 정보와 전달할 정보 및 토큰이 검증되었음을 증명하는 서명 등 필요한 모든 정보를 자체적으로 지니고 있다.

별도 저장소 불필요
: 인증 정보에 대한 별도의 저장소가 필요 없어 서버 부하를 줄일 수 있다. 서버에서 가장 피해야 할 것은 데이터베이스 조회인데, JWT는 이러한 부담을 크게 줄여준다.

다양한 플랫폼 지원
: 토큰 기반으로 다른 로그인 시스템에 접근 및 권한 공유가 가능하며, 모바일 애플리케이션 환경에서도 잘 동작한다. 쿠키 기반 세션과 달리 모바일 앱에서도 제약 없이 사용할 수 있다.

## 마무리
---
이렇게 쿠키->세션->JWT의 방식으로 각 방식이 어떤 배경에서 등장하게 되었고 어떤 방식으로 각 방식의 단점을 보완했는지 간단하게 알아보았다. 마지막으로 나온 JWT가 서술한 내용만으로는 만능으로 느낄 수 있다. 하지만 다른 방식들과 마찬가지로 단점 또한 가지고 있다. 만료 관리등 토큰자체가 무상태성을 띄기 때문에 대표적으로 토큰 탈취, 만료 관리 등 보안 이슈에 대해서 고려가 필요하다. 다음 포스팅에서는 이런 단점을 보완하기 위해서 Refresh Token을 다뤄보고 더 나아가 Spring을 이용하여 구현해보고자 한다.
## 참고
---
[JWT 공식 홈페이지](https://jwt.io/)

[MDN \| HTTP 쿠키](https://developer.mozilla.org/ko/docs/Web/HTTP/Guides/Cookies)

[MDN \| Set_Cookie](https://developer.mozilla.org/ko/docs/Web/HTTP/Reference/Headers/Set-Cookie)


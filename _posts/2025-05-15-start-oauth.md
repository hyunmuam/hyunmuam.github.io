---
title: "OAuth 개념과 동작 원리를 정리해보자"
date: 2025-05-15 14:30:00 +0900
categories: [Security]
tags: [auth]
image:
  path: /assets/img/posts/2025-05-15-start-oauth/01.png
---
요즘 다양한 서비스에서 소셜로그인을 기본적으로 지원하는 모습을 쉽게 볼 수 있다. 이 소셜로그인은 사용자가 별도의 회원가입 없이 구글, 카카오, 네이버 등 외부 플랫폼 계정으로 간편하게 로그인할 수 있게 해주는데, 그 동작 원리는 바로 OAuth 2.0 프로토콜에 기반한다.

소셜로그인의 핵심 동작 원리는 OAuth 2.0 권한 부여 프로토콜을 활용해 사용자의 인증과 권한 위임을 안전하게 처리하는 데 있다. 이번 포스트에서는 OAuth의 개념을 대해서 정리해보자.

## OAuth란?
---
**OAuth**("**O**pen **Auth**orization")는 **인터넷 사용자들이 비밀번호를 제공하지 않고 다른 웹사이트 상의 자신들의 정보에 대해 웹사이트나 애플리케이션의 접근 권한을 부여할 수 있는 공통적인 수단**으로서 사용되는, 접근 위임을 위한 개방형 표준이다.

![Desktop View](/assets/img/posts/2025-05-15-start-oauth/02.png){: width="972" height="589" }
_OAuth 예시_

OAuth를 통해 위 예시와 같이 외부의 플랫폼에서 자신의 리소스(구글 드라이브 파일 등)에 접근할 수 있다.

## OAuth 등장 배경
---
2006년 11월, 트위터의 OpenID 구현 과정과 소셜 북마크 서비스 Ma.gnolia의 인증 요구에서 출발했다. 당시 Ma.gnolia는 회원들이 OpenID로 대시보드 위젯 등 외부 애플리케이션에서 자신의 서비스에 접근할 수 있도록 허용하는 방법이 필요했다. 이 과정에서 트위터와 Ma.gnolia, 그리고 OpenID 커뮤니티가 논의한 결과, **_API 접근 권한 위임에 대한 공개 표준이 존재하지 않는다._** 는 문제를 인식하게 되었다.

기존에는 사용자가 외부 애플리케이션이나 서비스에 자신의 계정 정보를 직접 제공해야 했다. 이 방식은 보안상 매우 취약하며, 비밀번호 유출 시 모든 데이터가 위험에 노출되는 문제가 있었다.

이러한 문제를 해결하기 위해, **사용자의 비밀번호를 외부 서비스에 제공하지 않고도 제한된 권한만을 위임할 수 있는 표준 프로토콜의 필요성이 대두**되었다. 이에 따라 2007년 4월 OAuth 논의 그룹이 결성되어, 구글 등 주요 기업과 커뮤니티의 참여로 2007년 12월 OAuth 1.0의 최종 초안이 발표되었다. 이것이 바로 OAuth의 시작이다.

## OAuth 1.0
---
OAuth에서 사용하는 용어를 먼저 알아보자.
1. `Client`
	- OAuth 인증된 요청을 수행할 수 있는 HTTP 클라이언트
2. `Server`
	- OAuth 인증된 요청을 수락할 수 있는 HTTP 서버
3. `Protected Resource`
	- OAuth 인증된 요청을 사용하여 서버로부터 얻을 수 있는 접근이 제한된 리소스
4. `Resource Owner`
	- 서버에 인증하기 위해 자격 증명을 사용하여 보호된 리소스에 접근하고 제어할 수 있는 주체

![Desktop View](/assets/img/posts/2025-05-15-start-oauth/03.png){: width="972" height="589" }
_OAuth 1.0 워크플로우_

OAuth 1.0 프로토콜 런타임 워크플로우는 다음과 같은 단계로 구성된다
1. `Client` 가 인증 프로세스를 시작하기 위해  `Server` 에 임시 자격 증명(temporary credentials)을 요청한다. 이 자격 증명은 `Server` 에 대한 개별 클라이언트 요청을 구분하는 데 사용된다.
2. `Server` 는 요청을 검증한 후, `Client`에게 임시 자격 증명 세트를 반환한다.
3. `Client`는 `Resource Owner`를 인증 URI로 리다이렉트하여 `Protected Resource`에 접근하기 위한 승인을 얻는다.
4. `Resource Owner`는 자신의 자격 증명을 사용하여 `Server`에 인증하고 `Client`의 요청을 승인한다.
5. `Server`는 임시 자격 증명을 검증하고, `Resource Owner`가 `Client`를 승인한 후 인증 코드(verification code)를 생성한다.
6. `Resource Owner`는 이전 요청에서 `Client`가 제공한 콜백 URI로 리다이렉트 된다.
7. `Client`는 임시 자격 증명과 인증 코드를 사용하여 액세스 토큰을 요청한다.
8. `Server`는 요청을 검증하고 `Protected Resource`에 접근하기 위한 액세스 토큰을 'Client'에게 반환한다.

중요한 것은 이 흐름속에서 한번도 `Resource Owner`의 자격증명(아이디/비밀번호)가 클라이언트에게 노출되지 않는 것이다. 이를 통해 사용자는 개인 정보를 외부 서비스에 제공하지 않고도 제한된 권한만을 위임할 수 있게 되었다. 

### OAuth 1.0의 문제점
OAuth 1.0에도 여러 문제가 있었다.
1. 토큰이라는 값만 있었으면 모든 리소스에 접근이 가능하다.
	- 토큰이 발급되면 사용자 리소스 전체에 접근이 가능했다. 이는 세밀한 권한 제어가 어렵다는 단점으로 지적되었다.
2. 전달한 토큰의 유효기간 개념이 부족하다.
	- 토큰이 탈취되면 어뷰징 위험이 있다.
	- OAuth 1.0은 토큰 만료나 갱신에 대한 명확한 표준이 없었고, 이로 인해 토큰이 탈취될 경우 장기간 악용될 수 있는 보안적 취약점이 존재했다.
3. 클라이언트 구현이 복잡하다.
	- 보안성을 가져가기 위해서 암호학적 기반 보안책들을 사용했다.
	- OAuth 1.0은 요청마다 서명(signature), nonce, timestamp 등 암호학적 절차를 요구해 클라이언트 구현이 복잡했다.
4. 역할이 확실하게 나눠지지 않았다.
	- OAuth 서버는 리소스 오너 인증, 인가 토큰 발급, 보호된 리소스 관리의 모든 역할을 담당해야 했다.
5. 사용환경이 제한적이다.
	- OAuth 1.0은 주로 웹 환경을 염두에 두고 설계되어 모바일 등 다양한 플랫폼 지원에 한계가 있었다.

## OAuth 2.0
---
OAuth 2.0은 이름에서 알 수 있듯 앞서 서술한 OAuth1.0의 단점을 보완하고 특히 모바일 등의 다양한 플랫폼의 확산에 대응하기 위해 등장하였다.

OAuth 2.0에서는 네 가지 역할을 정의한다. OAuth 1.0에서의 용어와 크게 다르지 않다.

`Resource Owner`  
Protected Resource에 대한 접근 권한을 부여할 수 있는 주체로 Resource Owner가 사람인 경우, 이를 엔드유저(end-user)라고도 한다.

`Resource Server`  
Protected Resource를 호스팅하며, Access Token을 사용한 Protected Resource 요청을 수락하고 응답할 수 있는 서버이다.

`Client`
Resource Owner를 대신하여, 그리고 Resource Owner의 인가를 받아 Protected Resource 요청을 수행하는 애플리케이션이다. "Client"라는 용어는 애플리케이션이 서버, 데스크톱, 기타 장치 등 어디에서 실행되는지와 같은 특정 구현 특성을 의미하지 않는다.

`Authorization Server`  
Resource Owner를 성공적으로 인증하고 인가를 획득한 후, Client에게 Access Token을 발급하는 서버이다.

`Authorization Server`는 `Resource Server`와 동일한 서버일 수도 있고, 별도의 주체일 수도 있다. 하나의 `Authorization Server`가 여러 `Resource Server`에서 수락되는 `Access Token`을 발급할 수도 있다.

![Desktop View](/assets/img/posts/2025-05-15-start-oauth/04.png){: width="972" height="589" }
_OAuth 2.0 워크플로우_

1. `Client`는 `Resource Owner`에게 인증 요청을 보낸다. 인증 요청은 `Resource Owner`에게 직접 할 수도 있고(위 그림과 같이), 또는 `Authorization Server`를 중개자로 하여 간접적으로 할 수도 있다.
2. `Client`는 인가 허가(authorization grant)를 받게된다. 인가 허가는 `Resource Owner`의 인가를 나타내는 자격 증명으로, 아래에서 더 자세히 다루겠다.
3. `Client`는 `Authorization Server`에 인증하고, 인가 허가를 제시하여 엑세스 토큰을 요청한다.
4. `Authorization Server`는 `Client`를 인증하고 인가 허가를 검증한 후, 유효하다면 Access Token을 발급한다.
5. `Client`는 `Resource Server`의 `Protected Resource`를 요청하고, Access Token을 제시하여 인증을 수행한다.
6. `Resource Server`는 Access Token을 검증하고, 유효하다면 요청을 처리한다.

`Client`가 리소스 소유자로부터 인가 허가(`Authorization grant`)를 얻는데 있어 권장되는 방법은 `Authorization Server`를 중계자로 사용하는 것이다.

공식 명세(RFC 6749)와 주요 기술 문서에서 정의된 주요 권한 부여 방식은 다음과 같다.
1. `Authorization Code Grant`
	- 사용자가 인정 서버를 통해 인증 및 승인을 완료하면, `Client`는 일회성 권한 부여 코드를 받게 되고 이 코드를 이용해 AccessToken을 교환하는 방식
	- 리다이렉션 기반 흐름으로, `Client`는 `Resource Owner`의 사용자 에이전트(일반적으로 웹 브라우저)와 상호 작용할 수 있어야 하며, `Authorization Server`로부터 수신 요청을 받을 수 있어야 한다.
2. `Implicit Grant`
	- 인증 서버가 권한 부여 코드 없이 직접 엑세스 토큰을 발급받는 방식
	- JavaScript와 같은 스크립팅 언어를 사용하여 브라우저에서 구현된 `Client`를 위해 최적화된 간소화된 인증 코드 흐름이다.
	- 인증 코드를 발급하는 대신 클라이언트에게 직접 액세스 토큰이 발급된다.
3. `Resource Owner Password Credentials Grant`
	- 사용자가 자신의 아이디와 비밀번호를 `Client`에 직접 입력하여 AccessToken을 발급받는 방식
	- 리소스 소유자와 클라이언트 간에 높은 신뢰도가 있을 때만 사용해야 한다.
	- 다른 권한 부여 유형을 사용할 수 없을 때 사용된다.
	- 일반적으로 이 방식의 사용을 권장하지 않는다.
4. `Client Credentials Grant`
	- `Client`가 자신의 권한 하에 있는 보호된 리소스에 접근하거나 `Authorization Server`와 사전에 협의된 `Protected Resource`에 접근할 때 사용


### OAuth 1.0의 문제 해결 방식
1. 토큰이라는 값만 있었으면 모든 리소스에 접근이 가능하다.
	- OAuth 2.0은 세분화된 권한 관리를 위해 **scope** 매개변수를 추가하였다. 클라이언트는 액세스 토큰을 요청할 때 특정 리소스에 대한 접근 범위(예: `read`, `write`)를 명시할 수 있으며, 리소스 서버는 이 범위를 기반으로 접근을 제한한다.
2. 전달한 토큰의 유효기간 개념이 부족하다.
	- 단기 AccessToken과 장기 생성 Refresh Token을 분리하여 보안을 강화
	- AccessToken: 일반적으로 1시간 이내의 짧은 유효기간을 가지며, 유출 시 피해를 최소화
	- RefreshToken: 액세스 토큰이 만료되면 재인증 없이 새로운 액세스 토큰을 발급받을 수 있어 사용자 경험을 개선
3. 클라이언트 구현이 복잡하다.
	- OAuth 1.0의 복잡한 암호화 서명 요구사항을 제거하고, HTTPS 기반 보안으로 전환하였다.
	- 서명 생성 단계(HTTP 메서드, URL, 파라미터 조합 등)가 없어져 개발자 부담이 크게 줄였다.
	- 다양한 클라이언트 유형(모바일, 데스크톱, IoT)에 맞춘 **다중 인증 흐름**(Grant Types)을 지원을 통해 적절한 보안성을 유지하였다.
4. 역할이 확실하게 나눠지지 않았다.
	- 앞서 서술했듯 역할을 4가지로 분리하고 책임 영역을 구분하였다.
	- 이 분리는 시스템 확장성과 유지보수성을 높였습니다. 예를 들어, 인가 서버와 리소스 서버를 별도로 운영해 부하 분산이 가능하도록 하였다.
5. 사용환경이 제한적이다.
	- OAuth 2.0은 위에 서술한 다양한 `Authorization grant`를 도입해 다양한 플랫폼과 시나리오를 지원


## OAuth 2.0 실습
---
구글에서는 [OAuth 2.0 Playground](https://developers.google.com/oauthplayground/)를 통해 인증 및 권한 부여 흐름에 실습하고 테스트할 수 있는 웹 기반 도구를 제공한다.

이를 이용하여 실제 OAuth 2.0를 단계별 HTTP메시지를 분석해보고 직접 엑세스 토큰과 리프레시 토큰을 받급받아 다양한 GooGle API 호출을 테스트할 수 있다.
이 도구를 이용하여 Google Drive에 있는 파일 목록을 확인해보자.

### 1. 인증 요청
```
HTTP/1.1 302 Found
Location:
  https://accounts.google.com/o/oauth2/v2/auth?
  redirect_uri=https%3A%2F%2Fdevelopers.google.com%2Foauthplayground&
  prompt=consent&response_type=code&
  client_id=407408718192.apps.googleusercontent.com&
  scope=https%3A%2F%2Fwww.googleapis.com%2Fauth%2Fdrive&
  access_type=offline
```
이 HTTP 응답은 OAuth 2.0 인증 흐름의 첫 단계로, 클라이언트가 사용자를 Google의 인증 서버로 리다이렉트하는 과정이다. 상태 코드 `302 Found`는 임시 리다이렉션을 의미하며, Location 헤더에 사용자가 이동할 URL이 포함되어 있다.

| 파라미터        | 설명                                                                                                                    |
| :-------------- | :---------------------------------------------------------------------------------------------------------------------- |
| `redirect_uri`  | 인증 완료 후 사용자가 리다이렉트될 URI                                                                                  |
| `prompt`        | 사용자에게 동의 화면을 표시하도록 요청한다.                                                                             |
| `response_type` | `Authorization Code` 방식을 사용함을 나타낸다.<br>이는 서버 사이드 애플리케이션에서 주로 사용되는 방식이다.             |
| `client_id`     | Google API 콘솔에 등록된 애플리케이션 고유 식별자                                                                       |
| `scope`         | 애플리케이션이 요청하는 권한 범위<br>이 경우 Google Drive에 대한 접근을 요청하고 있다.                                  |
| `access_type`   | 오프라인 접근을 요청하여 사용자가 로그아웃한 후에도<br>애플리케이션이 리프레시 토큰을 통해 API에 접근할 수 있도록 한다. |

리다이렉트되면 다음과 같은 익숙한 화면을 볼 수 있다.
![Desktop View](/assets/img/posts/2025-05-15-start-oauth/05.png){: width="972" height="589" }
_리다이렉트된 URL 화면_

### 2. 인가 허가 부여
```
GET /oauthplayground/?
  code=4/0AUJR-x4C6wsmRRMzhwTiq_qyA8sr6j1upmq_PLeLE7S4vkRMqQ3W6_6KJ1ZRT2RwurR5KQ&
  scope=https://www.googleapis.com/auth/drive 
  HTTP/1.1
Host: developers.google.com
```
이 HTTP 요청은 OAuth 2.0 인증 흐름의 두 번째 단계로, 사용자가 인증 및 권한 부여를 완료한 후 Google 인증 서버가 클라이언트의 리다이렉트 URI(OAuth Playground)로 사용자를 리다이렉트하는 과정이다. 이 요청에는 인증 코드와 승인된 스코프 정보가 포함되어 있다.

| 파라미터 | 설명                                                                                                                                                                                             |
| :------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| code     | 인증 서버가 발급한 일회용 인증 코드<br>이 코드는 짧은 수명을 가지며, 클라이언트가 Access Token과 Refresch Token을 요청하는 데 사용된다.<br>이 코드는 보안상의 이유로 한 번만 사용할 수 있습니다. |
| scope    | 사용자가 실제로 승인한 권한 범위이다.<br>이 경우 Google Drive에 대한 전체 접근 권한을 의미한다.                                                                                                  |

### 3. 인가 허가 제시
```
POST /token HTTP/1.1
Host: oauth2.googleapis.com
Content-length: 261
content-type: application/x-www-form-urlencoded
user-agent: google-oauth-playground

code=4%2F0AUJR-x4C6wsmRRMzhwTiq_qyA8sr6j1upmq_PLeLE7S4vkRMqQ3W6_6KJ1ZRT2RwurR5KQ&
  redirect_uri=https%3A%2F%2Fdevelopers.google.com%2Foauthplayground&
  client_id=407408718192.apps.googleusercontent.com&
  client_secret=************&scope=&
  grant_type=authorization_code
```
OAuth 2.0 인증 흐름의 세 번째 단계로, 클라이언트가 이전 단계에서 받은 인증 코드를 사용하여 Google의 토큰 엔드포인트에 액세스 토큰을 요청하는 과정이다.

| 파라미터      | 설명                                                                                                                                                   |
| :------------ | :----------------------------------------------------------------------------------------------------------------------------------------------------- |
| code          | 이전 단계에서 받은 인증 코드. 이 코드는 일회용이며 짧은 수명을 가진다.                                                                                 |
| redirect_uri  | 인증 코드를 받을 때 사용한 리다이렉트 URI와 정확히 일치해야 한다.<br>이 파라미터는 인증 코드 요청에 포함되었다면 토큰 요청에도 반드시 포함되어야 한다. |
| client_id     | Google API 콘솔에 등록된 클라이언트 애플리케이션의 고유 식별자                                                                                         |
| client_secret | 클라이언트 애플리케이션의 비밀 키. 이 값은 절대 공개되어서는 안된다.                                                                                   |
| grant_type    | 요청하는 토큰의 유형. `authorization_code`는 인증 코드를 액세스 토큰으로 교환하는 OAuth 2.0 흐름을 지정                                                |

### 4. Access Token 발급
```
HTTP/1.1 200 OK
...
Content-type: application/json; charset=utf-8
{
  "access_token": "ya29.a0AW4...", 
  "refresh_token_expires_in": 604799, 
  "expires_in": 3599, 
  "token_type": "Bearer", 
  "scope": "https://www.googleapis.com/auth/drive", 
  "refresh_token": "1//04G5qSUv4..."
}
```
OAuth 2.0 인증 흐름의 네 번째 단계로, 인증 서버(Google)가 클라이언트의 토큰 요청에 대한 응답으로 액세스 토큰과 리프레시 토큰을 발급하는 과정이다.
이 단계 이후, 클라이언트는 발급받은 액세스 토큰을 사용하여 Google Drive API와 같은 보호된 리소스에 접근할 수 있으며, 액세스 토큰이 만료되면 리프레시 토큰을 사용하여 새로운 액세스 토큰을 요청할 수 있다.

| 파라미터                 | 설명                                                                                                        |
| :----------------------- | :---------------------------------------------------------------------------------------------------------- |
| access_token             | 보호된 리소스에 접근하기 위한 자격 증명, 이 토큰을 API 요청의 `Authorization` 헤더에 포함시켜 사용          |
| refresh_token            | `Access Token`이 만료된 후 새로운 `Access Token`을 얻기 위해 사용하는 토큰                                  |
| scope                    | 발급된 토큰이 접근할 수 있는 권한 범위                                                                      |
| token_type               | 토큰의 유형, Bearer는 HTTP 요청의 `Authorization` 헤더에 `Bearer` 접두사와 함께 토큰을 포함시켜야 함을 의미 |
| expires_in               | `Access Token`의 유효 기간(초)                                                                              |
| refresh_token_expires_in | `Refresh Token`의 유효 기간(초)                                                                             |

### 4. Access Token 제출

```
GET /drive/v3/files HTTP/1.1
Host: www.googleapis.com
Content-length: 0
Authorization: Bearer ya29.a0AW4...
```
OAuth 2.0 인증 흐름의 최종 단계로, 클라이언트가 발급받은 액세스 토큰을 사용하여 Google Drive API에 접근하는 요청이다.
Authorization헤더에 이전 단계에서 발급받은 AccessToken을 넣은것을 알 수 있다.

### 5. Protected Resource 전달
```
HTTP/1.1 200 OK
Content-location: https://www.googleapis.com/drive/v3/files
Content-type: application/json; charset=UTF-8
...

{
  "files": [...]
}
  ...
```
정상적으로 Google Drive의 데이터를 받아온 것을 확인할 수 있다.

## 마무리
---
OAuth 2.0은 보안 강화와 사용자 편의성 사이의 균형을 찾으면서도 개발자 친화적인 구조로 진화했다는 것을 알 수 있다. 현대 웹 서비스에서 소셜 로그인이 보편화된 이유는 바로 이러한 OAuth 2.0의 발전 덕분이다.

앞서 서술했지만 다시 한 번 OAuth 2.0의 주요 장점을 정리하자면 다음과 같다.
- 사용자 정보 보호: 사용자는 제3자 애플리케이션에 직접 계정 정보를 제공하지 않고도 안전하게 서비스를 이용할 수 있다.
- 세분화된 권한 관리: scope 개념을 통해 애플리케이션이 접근할 수 있는 리소스의 범위를 제한할 수 있다.
- 토큰 기반 인증: 단기 액세스 토큰과 장기 리프레시 토큰의 분리로 보안성을 강화하면서도 사용자 경험을 해치지 않는다.
- 다양한 플랫폼 지원: 웹, 모바일, IoT 등 다양한 환경에 맞는 인증 흐름을 제공한다.
- 역할 분리: 인가 서버와 리소스 서버의 분리를 통해 시스템 확장성과 유지보수성을 향상시켰다.

Spring Security와 같은 프레임워크는 OAuth 2.0 구현을 간소화하여 개발자가 복잡한 인증 로직을 직접 구현하지 않고도 안전한 소셜 로그인을 쉽게 구현할 수 있도록 지원하고 있다. 이러한 발전은 사용자와 개발자 모두에게 이점을 제공하며, 웹 서비스의 보안성과 사용성을 동시에 향상시키고 있다.

이러한 발전은 사용자와 개발자 모두에게 이점을 제공하며, 웹 서비스의 보안성과 사용성을 동시에 향상시키고 있다. OAuth 2.0은 현대 웹 서비스의 인증 및 인가 표준으로 자리 잡았기 때문에, 한 번 쯤은 정리의 필요성을 느끼고 있었다. 이번 포스트를 통해 OAuth의 모든 것을 다루지는 못했지만, 시작할 포인트를 알 게 되었다.

## 참고
---
[\[NHN FORWARD 22\] 로그인에 사용하는 OAuth : 과거, 현재 그리고 미래](https://youtu.be/DQFv0AxTEgM?si=0M5PtLX39_B56e_S)

[ The OAuth 2.0 Authorization Framework \| RFC 문서](https://datatracker.ietf.org/doc/html/rfc6749)

[oauth.net](https://oauth.net/)

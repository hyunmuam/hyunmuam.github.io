---
title: WebSocket!! 이론부터 구현까지
date: 2025-04-24 22:00:00 +0900
categories: [WebSocket]
tags: [websocket]
image:
  path: /assets/img/posts/2025-04-25-WebSocket/01.png
---
최근 진행했던 프로젝트에서 실시간 채팅을 담당하게 되었다. 그 과정에서 STOMP를 사용할 기회가 생겼는데, 어찌어찌 구현하긴했지만 제대로 알지 못하고 사용한다는 마음의 짐을 늘 가지고 있었다. Stomp는 웹소켓 위에서 동작하는 메시징 프로토콜이기 때문에 이번기회에 웹소켓에 대하여 정리하고자 한다.

## HTTP의 한계
---
웹개발을 할 때 가장 흔하게 사용하는 HTTP프로토콜은 요청-응답 프로토콜로 설계되었다. 웹 브라우저와 같은 클라이언트에서 어떤 요청을 보내고, 서버에서는 그 요청을 처리하고 응답하게 된다. 즉 **단방향 통신**을 하게된다.  그리고 이 과정이 완료된 이후에 클라이언트와 서버간의 연결은 바로 끊기게 된다.

대부분의 기능을 구현할 때 이런 통신 방식은 서버로 하여금 다수의 클라이언트에서 들어오는 요청을 적은 하드웨어 리소스를 가지고 효율적으로 처리할 수 있도록 도와준다.

하지만 실시간으로 여러 사용자와 양방향으로 상호작용을 하는 애플리케이션을 만드는데 이러한 장점은 제약사항이 될 수도 있다. 대표적으로 2가지 정도를 꼽을 수 있다.

1. **클라이언트가 요청하기 전까지 서버에서는 할 수 있는 것이 없다.**
  - HTTP 프로토콜에서는 항상 클라이언트가 연결을 시작하는 주체가 된다. 즉 서버는 클라이언트의 요청을 기다려야하는 입장이 되어야 한다. 만약 서버에서 데이터 변경과 같은 이벤트가 발생하여 클라이언트에게 알려줘야할 때 클라이언트의 요청이 오기까지 기다려야한다.
  - 이 문제를 보완하는 `Polling`과 같이 주기적으로 서버에 요청하여 호출하는 기법을 사용할 수 있지만, 만약 이벤트가 발생하지 않은 경우 무의미한 호출만 많아지게 될 수 있다.
2. **연결이 유지되지 않기 때문에 매 연결 시 오버헤드가 발생**
  - HTTP프로토콜에서는 서버와 클라이언트 간의 연결이 유지되지 않는다. 기존 문맥과 상태를 유지하면서 최소한의 정보를 효과적으로 주고받기가 어려워진다.
  - 기본적으로 요청/응답 메시지에는 헤더(header)가 차지하는 공간이 크기 때문에, 단문의 메시지를 주고받을 경우 배보다 배꼽이 더 커질 수 있는 상황이 될 수 있다.
  - HTTP/1.1에는 TCP/IP 연결을 재사용하는 [영구 연결](https://www.nginx.com/blog/http-keepalives-and-web-performance/)이 도입되어 일부 성능이 향상됬다고 한다. 그러나 이러한 영구 연결의 세부 사항은 서버마다 다르며 대부분의 경우 비활성 시간 초과에 따라 결국 닫힌다.

## WebSocket?
---
이러한 한계를 극복하기 위해 등장한 것이 바로 웹소켓 프로토콜이다. 웹소켓 프로토콜은 클라이언트와 서버간의 보다 효율적인 효율적인 실시간 양방향 통신을 가능하게 하는 웹 표준 기술이다. 서버와 브라우저 간 연결을 유지한 상태로 데이터를 교환할 수 있다. 이 웹소켓에 대해서 조금만 더 알아보자

### 통신 과정
웹소켓 프로토콜은 크게 핸드셰이크(HandShake)와 데이터 전송(Data Trasfer)로 이루어져 있다.
연결부터 종료까지의 과정을 정리하자면 다음과 같다. 


#### **연결 수립(Opening HandShake)**
웹소켓 연결은 HTTP 프로토콜을 이용한 업그레이드 요청(Upgrade Request)으로 시작한다.
클라이언트가 서버로 HTTP GET 요청을 보내고, `Upgrade: websocket`, `Connection: Upgrade` 등의 헤더를 포함한다.
```text
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Origin: http://example.com
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Version: 13
```

요청을 받은 서버가 웹소켓을 지원한다면 서버는 `101 Switching Protocols` 응답과 함께 웹소켓 프로토콜로의 업그레이드를 승인한다.
```text
(Client -> Server)
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

이 과정을 통해 기존 HTTP 연결이 웹소켓 연결로 전환되며, 이후부터는 웹소켓 고유의 프로토콜로 통신이 이루어지게 된다. 
<br>핸드셰이크가 성공적으로 완료되면, 클라이언트와 서버는 지속적인 연결 상태를 유지하며 데이터 전송을 시작할 수 있게된다.

#### **데이터 전송 (Data transfer)**

핸드셰이크가 성공적으로 이루어진 후, 클라이언트와 서버는 데이터를 주고받기 시작하며, 이때 전송되는 데이터의 논리적 단위를 이 명세에서는 **메시지(message)**라고 부른다. 실제 네트워크 상에서 메시지는 하나 이상의 **프레임(frame)**으로 구성된다.
<br>연결이 된 이후 서버와 클라이언트 양쪽 모두 독립적으로 메시지를 전송할 수 있다.


**프레임(frame) 구조**
```
    0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
  +-+-+-+-+-------+-+-------------+-------------------------------+
  |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
  |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
  |N|V|V|V|       |S|             |   (if payload len==126/127)   |
  | |1|2|3|       |K|             |                               |
  +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
  |     Extended payload length continued, if payload len == 127  |
  + - - - - - - - - - - - - - - - +-------------------------------+
  |                               |Masking-key, if MASK set to 1  |
  +-------------------------------+-------------------------------+
  | Masking-key (continued)       |          Payload Data         |
  +-------------------------------- - - - - - - - - - - - - - - - +
  :                     Payload Data continued ...                :
  + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
  |                     Payload Data continued ...                |
  +---------------------------------------------------------------+
```

| 필드명                     | bit  | 설명                                                                                                                |
| :------------------------- | :--- | :------------------------------------------------------------------------------------------------------------------ |
| `FIN`                      | 1    | 메시지의 마지막 조각임을 나타냄. <br> 첫 번째 조각이 마지막 조각일 수도 있습니다.                                   |
| `RSV1`<br>`RSV2`<br>`RSV3` | 1    | 확장 프로토콜에서 사용하기 위한 예약 비트.<br> 일반적으로 0으로 설정                                                |
| `opcode`                   | 4    | 프레임의 타입(데이터/제어 프레임 등)을 지정                                                                         |
| `MASK`                     | 1    | 페이로드 데이터가 마스킹되었는지 여부를 나타낸다.<br>클라이언트 → 서버 전송 시 항상 1, 서버 → 클라이언트는 0입니다. |
| `Payload len`              | 7~63 | 페이로드(실제 데이터)의 길이를 나타낸다.<br>7비트(0~125), 126(16비트 확장), 127(64비트 확장)                        |
| `Masking-key`              | 1    | (옵션) 마스킹에 사용되는 4바이트 키<br>Mask 비트가 1일 때만 포함됩니다.                                             |
| `Payload Data`             | 가변 | 실제 전송되는 데이터(텍스트, 바이너리, 제어 프레임 등)                                                              |

#### **연결 종료(Closing Handshake)**
  연결 종료(`Closing Handshake`)는 처음에 실행된 연결 수립(`Opening HandShake`)보다 훨씬 간단하다.
  1. 연결을 종료하려는 클라이언트 또는 서버에서 Close 프레임(`opcode`를 8로 지정)을 전송하여 종료 의사를 표시
  2. 상대방이 아직 Close 프레임을 보내지 않았다면, 응답으로 Close 프레임을 보냄
  3. Close 프레임이 양쪽에서 모두 교환되면 더 이상 데이터 전송이 이루어지지 않고, TCP 연결이 안전하게 종료

### WebSocket와 HTTP 비교
이제 HTTP의 앞서 말했던 제약사항을 웹소켓은 이 문제를 어떻게 보완하는지 생각해보자

1. 클라이언트가 요청하기 전까지 서버에서는 할 수 있는 것이 없음
   - HTTP는 요청-응답 기반의 단방향 통신이기 때문에, 서버는 항상 클라이언트의 요청을 기다려야만 데이터를 전송할 수 있다.
   - 반면, WebSocket은 전이중(Full Duplex) 통신을 지원하므로, 서버도 클라이언트에게 자유롭게 이벤트나 데이터를 실시간으로 보낼 수 있다.

2. 연결이 유지되지 않기 때문에 매 연결 시 오버헤드가 발생
   - HTTP는 요청마다 새로운 연결을 생성하거나, Keep-Alive를 쓰더라도 일정 시간 후 연결을 끊는다. 이로 인해 매번 헤더 등 부가 데이터가 반복 전송되어 오버헤드가 크다.
   - WebSocket은 초기 핸드셰이크 이후 하나의 연결을 지속적으로 유지하며, 이후에는 최소한의 프레임 헤더만으로 데이터를 주고받는다.
   - 따라서 대용량 데이터나 빈번한 실시간 메시지 전송에서도 오버헤드가 현저히 줄어들고, 네트워크 효율성이 크게 향상된다.

## 간단한 구현
---
> 전체 프로젝트 코드는 [Github](https://github.com/tjvm0877/blog-code/tree/main/web-socket-app)에 있으니 참고해주세요.
{: .prompt-info }
웹소켓에 대한 개념을 웹소켓을 이용한 여러 클라이언트를 연결하는 간단한 대화방 앱을 만들어보자.
![Desktop View](/assets/img/posts/2025-04-25-WebSocket/02.png){: width="972" height="589" }
웹소켓 연결 이후 클라이언트가 서버로 메시지를 보내게 되면 서버는 해당 메시지를 참가자 전체에게 보내주도록하는 앱이다.

### WebSocket in Spring Boot
Spring Framework는 WebSocket 메시지를 처리하는 클라이언트 및 서버 측 애플리케이션을 작성하는 데 사용할 수 있는 WebSocket API를 제공해주기 때문에 단순한 앱의 경우 아주 간단하게 만들 수 있다.

<br>먼저 웹소켓을 통해 메시지를 받고나서 어떻게 할지를 정의해 준다.

```java
public class MyWebSocketHandler extends TextWebSocketHandler {

  // WebSocketSession은 클라이언트-서버 간 웹소켓 연결을 추적·관리하는 세션 객체이다.
  // 여러 세션을 관리함으로써 실시간 채팅, 브로드캐스트 등 다양한 실시간 기능 구현이 가능
  private final Map<String, WebSocketSession> SESSIONS = new ConcurrentHashMap<>();

  // 웹소켓 연결 성립 시 호출
  @Override
  public void afterConnectionEstablished(WebSocketSession session) throws Exception {
    // 세션을 저장
    SESSIONS.put(session.getId(), session);
  }

  // 텍스트 메시지 수신 시 호출
  @Override
  protected void handleTextMessage(WebSocketSession session, TextMessage message) throws Exception {
    // 메시지 발신자를 제외한 Session에 수신 메시지 Broadcast
    for (WebSocketSession session1 : SESSIONS.values()) {
      if (!session1.getId().equals(session.getId())) {
        session1.sendMessage(message);
      }
    }
  }

  // 웹소켓 연결 종료 시 호출
  @Override
  public void afterConnectionClosed(WebSocketSession session, CloseStatus status) throws Exception {
    // 저장된 세션을 삭제
    SESSIONS.remove(session.getId());
  }
}
```
{: file='MyWebSocketHandler'}

이제 웹소켓의 엔드포인트를 정의한다.
```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

  @Override
  public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
    registry.addHandler(webSocketHandler(), "/chat") // 핸들러 객체와 엔드포인트 '/chat' 연결
      .setAllowedOrigins("http://localhost:5173"); // 허용할 Origin 제한 (CORS 정책)
  }

  @Bean
  public WebSocketHandler webSocketHandler() {
    return new MyWebSocketHandler();
  }
}
```
{: file='WebSocketConfig'}

### WebSocket in JS
> 전체 코드는 [Github](https://github.com/tjvm0877/blog-code/tree/main/web-socket-app/client)에 있으니 참고해주세요.
{: .prompt-info }
WebSocket API는 브라우저에서 지원하는 웹 표준 API가 있다. JavaScript를 통해 접근할 수 있으며, 브라우저에 내장된 기능이다. 이 API를 이용하여 간단하게 웹소켓을 사용할 수 있도록 도와준다.

웹소켓 다음과 같이 어떠한 이벤트가 실행되었을 때 핸들링하주는 코드만 추가해주면 된다.
```javascript
// WebSocket 객체를 생성하고 서버에 연결을 시도
const ws = new WebSocket(url);

// WebSocket 연결이 성공적으로 열렸을 때 실행되는 이벤트 핸들러
ws.onopen = () => {
  setIsConnected(true);
  console.log('연결 성공');
};

// 서버로부터 메시지를 수신했을 때 실행되는 이벤트 핸들러
ws.onmessage = (event) => {
  addMessage({
    type: 'RECEIVED',
    content: event.data,
    timestamp: new Date(),
  });
};

// WebSocket 연결이 종료되었을 때 실행되는 이벤트 핸들러
ws.onclose = () => {
  setIsConnected(false);
  console.log('연결 종료');
};

// WebSocket에서 에러가 발생했을 때 실행되는 이벤트 핸들러
ws.onerror = () => {
  console.log('에러 발생');
};
```

### 실행
![Desktop View](/assets/img/posts/2025-04-25-WebSocket/03.gif){: width="400" height="300" }

## 한계
---
여기까지는 WebSocket을 사용 시 위에 서술한 것과 같이 여러 장점을 주지만 관련 정보를 찾아보면서 그에 못지 않는 비용을 지불해야 한다는 것을 알게되었다.
대표적인 발생할 수 있는 어려운 점 또는 문제점은 다음과 같았다.
1. WebSocket은 지속적인 연결을 유지하기 때문에 서버 리소스에 상당한 부담을 줄 수 있습니다. <br>각 클라이언트마다 별도의 연결을 유지해야 하므로 동시 접속자 수가 많아질수록 서버의 메모리와 CPU 사용량이 급증할 수 있음
2. 브라우저 호환성 이슈 <br>WebSocket은 현대 브라우저에서 대부분 지원되지만, 여전히 일부 오래된 브라우저나 특정 환경에서는 호환성 문제가 발생할 수 있음
3. 보안 이슈 <br>WebSocket은 지속적인 연결을 제공하기 때문에 보안 측면에서 추가적인 고려사항이 필요

## 마무리
---
이번 글 작성을 위해 여러 게시물들을 찾아보았다. 이 글에서는 아주 간단한 예시를 사용했지만 실제로 제공되는 서비스로 웹소켓 기능을 제공하기 위해서는 훨씬 더 복잡해진다는 것을 알게되었다. 나중에 기회가 된다면 이부분들을 다뤄봐야겠다.<br>웹소켓이 HTTP의 한계를 보완할 수 있다고 했지만 위에 서술 했듯이 웹소켓 또한 모든 경우에 유용할 수 없다. 이 기술이 그렇기 때문에 프로젝트 혹은 기능에 꼭 필요한 기술인지 잘 체크하고 수용여부를 결정해야 겠다는 생각으로 글을 마무리한다.

## 참고
---
[WebSocket 대 HTTP 통신 프로토콜](https://sendbird.com/ko/developer/tutorials/websocket-vs-http-communication-protocols)<br>
[RFC-6455 문서](https://www.rfc-editor.org/rfc/rfc6455)<br>
[Spring 공식문서](https://docs.spring.io/spring-framework/reference/web/websocket.html)<br>
[javascript.info](https://javascript.info/websocket)

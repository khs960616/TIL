HTTP 프로토콜 

정의 : 인터넷에서 데이터를 주고 받을 수 있는 프로토콜이다.

간략한 추가 정보
1. 서버 - 클라이언트 모델로 이루어져있으며, Request, Response 두 가지의 메시지가 존재한다. 
2. connection less : 서버-클라이언트 사이에 데이터를 주고 받은 이후 커넥션을 끊는다.
   ->(http/1.1 부터는 default로 persistent connection을 유지한다.)
3. state-less : 서버가 클라이언트의 이전 상태에 대한 정보를 가지고 있지 않는다. 

---
### Keep Alive ? 

HTTP 1.0+ 부터 지원한다.
Keep-Alive : HTTP 헤더로 지속적인 연결이 열린 상태를 유지할 기간을 제어한다. 
(사용시 connection : keep-alive, keep-alive: timeout, max들 지정한다.)
max : 해당 connection으로 최대 처리 가능한 request 수,
timeout : 해당 connection이 idle 한경우 얼마나 오랜 기간 connection을 유지할 것인지에 대한 정보 

Persistent Connection이 필요할 때 사용한다. 
site locality (클라이언트에서 특정 웹페이지를 요청하는 경우 서버에 연속적인 request를 보내는 경우)와 같이, 
서버에 연속적으로 동일한 클라이언트가 여러 요청을 보낼 때 사용된다. 


장점 : 전반적인 네트워크 비용을 감소시킬 수 있으며 latency가 감소한다.
(connection request수가 줄어들어 네트워크 혼잡이 감소되며, 하나의 connection으로 여러 요청을 처리하므로 
네트워크 비용과 latency가 감소하게 된다.)

--------
### 세션과 쿠키

실제 우리가 사용하는 웹 어플리케이션들을 확인해보면, 마치 서버가 클라이언트의 정보를 알고 있는 것 처럼 동작한다.
HTTP는 statelesss한 프로토콜인데 이것이 어떻게 가능한 것일까?

1. 쿠키 
- stateless한 http의 한계를 극복하기 위해 사용되는 서버가 웹 브라우저에 전송하는 작은 데이터 조각으로, 
- 세션 관리, 개인 설정유지, 사용자 트래킹을 위해 사용된다.

쿠키의 제약 조건
쿠기 하나당 4KB의 용량 제한이 있으며 최대 300개 까지만 저장이 가능하다.
(세션 쿠키의 경우 브라우저가 종료되는 시점에 쿠키가 삭제되고, 일반 쿠키의 경우 설정된 만료 날짜까지 유지된 후 삭제된다.)

2. 세션 
- 클라이언트가 서버에 접속하는 순간, 각 클라이언트마다 고유한 세션 ID를 부여하고, 서버내에서 해당 클라이언트들의 상태
- 정보 저장을 위해 추가적인 공간을 사용한다. 

둘다 쿠키를 활용하며, 일반적으로 쿠키는 브라우저의 저장소에 보관되므로 보안이 세션에 비해 취약하나, 서버에 부하를 줄일 수 있으며,
세션의 경우 쿠키보다는 보안상 이점이 있으나, 서버에 부하를 준다는 trade off가 존재한다. 
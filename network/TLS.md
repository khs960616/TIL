### TLS (Transport Layer Security)
인터넷 상의 커뮤니케이션을 위한 개인 정보와 데이터 보안을 하기 위해 채택된 보안 프로토콜

기능
1. 암호화 : 제 3자로부터 전송되는 데이터를 숨긴다.
2. 인증 : 정보를 교환하는 당사자가 요청된 당사자임을 보장한다.
3. 무결성 : 데이터가 위조되거나 변조되지 않았는지를 확인한다.

TLS 인증서가 있어야한다. 


대칭키 암호화 방식 : 인코딩과 디코딩에 동일한 키를 사용하는 암호화 방식 
단점 : 발송자와 수신자가 대칭키 암호화를 사용하기 위해서는 '공유하는 키' 값을 가져야한다. 
(실제 네트워크 상에서 암호화를 위한 이런 대칭키를 서로 어떻게 공유할 것인가? 에 대한 문제)


공개키 암호화 방식 : 인코딩과 디코딩에 서로 다른 키를 사용하는 알고리즘 

ex) A키로 암호화를 하면 B키로 복호화할 수 있고, B키로 암호화하면 A키로 복호화 할 수 있는 방식.
public key는 공개되어 있으며, secret key는 각 호스트가 소유한다.
메세지의 인코딩은 누구나 할 수 있지만, 디코딩은 키를 소유한 소유자만이 할 수 있다.

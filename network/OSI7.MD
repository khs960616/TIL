# 프로토콜
두 사람 또는 두 장치간에 이루어지는 일련의 통신은 프로토콜을 필요로 한다.

프로토콜이란? 통신을 주도하는 규칙들의 집합이다. 

-> 정해진 규약이 없이 개체간의 서로 다른 방식으로 데이터를 전송하게 되면 다른 모든 개체들과 통신하기 어렵다. 표준화된 프로토콜이 필요하다

따라서 프로토콜은 무엇을, 어떻게 그리고 언제 통신할 것인가를 규정해야한다.

---
# OSI 7 계층 (Open System Interconnection)

캡슐화 : 

Application Layer : 

Presentation Layer :

Session Layer : 

transport Layer : 전체 메시지의 프로세스 to 프로세스의 전달을 책임진다. 

Network Layer
```
1. 논리 주소를 지정한다. (IP 주소)
2. 경로지정 : 독립된 네트워크 
```

Data Link Layer 
```
데이터 링크 레이어의 역할과 책임
1. 프레임 구성 : 데이터 링크 계층은 상위 계층인 네트워크 계층으로부터 받은 비트 스트림을 프레임(frame)이라는 단위로 나눈다. 
2. Mac주소 지정 : 프레임이 네트워크 상에 서로 다른 시스템에 분배해서 보내야 한다면, 데이터 링크 계층은 해당 프레임의 송신자, 수신자를 나타내는
헤더 정보를 프레임에 추가해서 히위 계층으로 전달한다.
3. 흐름 제어 : 수신자에 의해 처리되는 데이터율이 송신자가 보낸 데이터율보다 낮다면 데이터 링크 계층은 수신자가 흐름 제어 메커니즘을 수행한다.
4. 오류 제어 : 손상되거나, 손실된 프레임을 탐지하고 재전송하는 매커니즘을 추가하여, Physical Layer의 신뢰성을 높인다.
5. 접근 제어
```

Physical Layer : 물리 매체를 통해 비트 스트림을 전달하는 데 필요한 기능을 조정한다.

(각 비트들을 한 노드에서 다음 노드로 전달하는 책임을 가진다.)

---

# 주소 지정

TCP/IP 프로토콜을 이용한 인터넷은 3개의 서로 다른 계층의 주소가 사용된다.

### Physical Address 물리 주소
- LAN이나 WAN에서 정의된 노드의 주소. 데이터링크 계층에 의해 사용되는 프레임에 포함된다. 물리주소는 링크상의 권한을 가진다.
- 물리 주소의 크기와 형식은 네트워크에 따라 다양하다.
> Eternet은 네트워크 Network Interface Card(NIC)에 있는 6바이트(48)비트의 물리주소 사용
> LocalTalk 매번 지국의 주소가 변하는 1바이트의 동적 주소 

### Logical Address 논리(IP)주소 

### Port Address 포트번호 

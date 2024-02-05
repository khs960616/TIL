```c++
#include <sys/socket.h>

int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```

생성되어있는 소켓의 속성값을 설정해주는 함수 

**level** : opt_name에서 지정되는 속성이, 어떤 level에서 설정된 것임을 알려주는 값
```
SOL_SOCKET : optname이 socket level에서 설정하는 option
IPPROTO_IP : optname이 IP protocol level에서 설정하는 option
IPPROTO_TCP : optname이 TCP protocol level에서 설정하는 option
```

**optname** : 소켓에 설정할 속성
```
 SO_ACCEPTCONN : accept된 connection 여부 조회(get only) : 1이면 accept(2)된 connection.
 SO_BROADCAST : datagram socket에 boradcast flag값을 (set/get)
 SO_DOMAIN : socket에 설정된 domain값 (ex. AF_INET, AF_UNIX 등...)을 얻는다. (get only)
 SO_ERROR : socket error를 읽고 지움. (get only)
 SO_DONTROUTE : gateway를 통해서 전송을 금지하고 직접 연결된 host끼리만 전달하게 함. (set/get)
 SO_KEEPALIVE : cconnection-oriented socket에서 keep alive message를 전송할 수 있도록 함. (set/get)
 SO_LINGER : linger option 설정 (set/get)
     struct linger {
         int l_onoff;    /* linger active */
         int l_linger;   /* how many seconds to linger for */
     };
     l_onoff를 1로 설정하면, close(2), shutdown(2) 함수를 실행하면 미전송된 데이터를 정상적으로 전송하거나
     linger timeout이 도래되면 함수를 return함.  그렇지 않으면 바로 return되고 background로 작업하게 됨.

 SO_OOBINLINE : out of bound data를 직접 읽을 수 있게 set/get (주로 X.25에서 사용)
 SO_PROTOCOL : socket에 설정된 protocol을 읽음.
 SO_RCVBUF : socket에서 읽을 수 있는 최대 buffer의 크기를 set/get함
 SO_REUSEADDR : bind(2) 시에 local 주소를 재사용할 것인지 여부를 set/get함
 SO_SNDBUF : socket에서 write할 수 있는 최대 buffer의 크기를 set/get함
 SO_TYPE : 설정된 socket의 type(ex. SOCK_STREAM, SOCK_DGRAM0을 get함 
```

**optval**: optname에 속성 설정에 필요한 값,  **optlen**: optval의 길이

---
SO_REUSEADDR: (optval 1로 설정 시, 재할당 가능, 0인 경우 불가)

TIME_WAIT상태에 있는 포트번호를 새로 할당되는 소켓에 할당할 수 있도록 하는 옵션 


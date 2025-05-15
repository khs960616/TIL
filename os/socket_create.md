근본적으로는 unix domain socket 설정 시, 개별 소켓별 커널영역의 버퍼를 어느정도 먹는지 체크하기 위함
(커널 파라미터별로는 별도로 알아서 고려할것, setsockopt등의 옵션에서 send, rcv버퍼크기 설정의 경우, 2배 설정 또는 커널내부에서 알아서 처리한다
(맨페이지에 적혀있는 내용인데, 이거 뭐 필요하면 코드 체크하자 일단은 메모리 할당량만 어느정도 잡힐지 추산)

## struct socket
vfs 계층의 업무 처리, inode 할당 & iattr설정등의 역할  (inode attribute)

sock_alloc
-> allocated a new inode and socket obj
-> out of inode return NULL

sockfs_setattr(struct mnt_idmap *idmap,
  simple_setattr (inode에 atttribute 설정)


/* proto? transport layer! 
 * Networking protocol blocks we attach to sockets.
 * socket layer -> transport layer interface
 */ 

이후 입력받은 net_families에 대한 pf를 가져와서 sock 구조체를 만듬 (여기가 프로토콜 관련된 로직들 처리) 

(uds의 경우 af_unix.c 쪽에 인터페이스에 들어갈 함수들 정의되어있으니 체크) 

static struct sock *unix_create1(struct net *net, struct socket *sock, int kern, int type)
-> sk_alloc이후에, 
-> sock_init_data(sock, sk);
  -> sock_init_data_uid
    ->  sk_init_common: 송신큐, 수신큐, 에러큐, lock 관련 설정 (별도 프로토콜 종속적인 내용은 여기선 없다)
    -> 이후에 
       	sk->sk_rcvbuf		=	READ_ONCE(sysctl_rmem_default);
	      sk->sk_sndbuf		=	READ_ONCE(sysctl_wmem_default);  여기서 가져온다 


sk_buff_head -> sock객체에 있는 큐들의 형태 (양방향 링크드 리스트 표현용 + lock)

sk_buff 구조체는 패킷을 표현하는 구조체 형태 

(/proc/sys/net/core/wmem_default, /proc/sys/net/core/rmem_default 설정된값) 
https://elixir.bootlin.com/linux/v6.12.1/source/net/core/sock.c#L2478 

sk_wmem_queued + 추가되려는 사이즈가 결국에 설정된값들을 초과하는지 체크하는게 전부임 



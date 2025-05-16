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

현재까지 alloc된 사이즈랑 snd_buf 설정값을 비교하는구나.. 그리고 wait하러들어감 

```c
	for (;;) {
		err = sock_error(sk);
		if (err != 0)
			goto failure;

		err = -EPIPE;
		if (READ_ONCE(sk->sk_shutdown) & SEND_SHUTDOWN)
			goto failure;

		if (sk_wmem_alloc_get(sk) < READ_ONCE(sk->sk_sndbuf))
			break;

		sk_set_bit(SOCKWQ_ASYNC_NOSPACE, sk);
		set_bit(SOCK_NOSPACE, &sk->sk_socket->flags);
		err = -EAGAIN;
		if (!timeo)
			goto failure;
		if (signal_pending(current))
			goto interrupted;
		timeo = sock_wait_for_wmem(sk, timeo);
	}
```

결론
- 단순 소켓 생성 개수만으로는 버퍼값 설정으로 인한 메모리 사용량을 걱정할 필요는 없을 것이다.
- 리눅스 계열 os 사용시 네트워크 bandwidth이 좁거나, 지연이 심한 환경에서 각 값들을 비정상적으로 크게 잡아두는 경우 불필요하게 메모리 낭비가 생길 것 같긴함.

- 코드 보기 전까진 책이나 강의들에서 그림그려서 설명하는거만 보고 소켓별로 설정된 버퍼크기만큼 할당해놓고, 
  copy from user로 kernal 영역 버퍼로 copy해놓은 후, 이후 패킷 단위로 끊어서 보내는줄 알았음
  일단은 6버전 위로는 그렇게 동작 안하고 있음 (recvq, sendq 자체는 아직 처리되지 못한 sk_buff들이 달려있는 양방향 링크드 리스트고, 각 SO_RCVBUF등으로 설정한 값들은
  여기에 sk_buff를 최대 얼마까지 쌓아둘수있는지 allocated 되는 양을 제어함, 그리고 유저레벨에서 전달받은 msg는 sk_buff가 할당 가능한 시점에, copy 가능한 양만큼씩 copy됨

- 부가적으로... socket == vfs 및 proto faimliy등의 handler를 다루는 layer, sock == 실제 소켓관련 인터페이스들 제공,  sk_buff 소켓 버퍼 
  (전송하려는 메시지가 패킷 단위로 끊어져 있는 형태, slab alloc을 통해 가져옴) 참고로 위에 코드관련 내용들은 sock에서 generic handler 및 af_unix만 보고 보고 적어둔 것이다. 
  -> proto family 내용이 필요하다면 경우에 따라 ipv4, ipv6, af_unix, udp 등등.. 필요한 쪽 가서 코드 찾아야됨

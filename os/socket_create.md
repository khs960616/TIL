
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


블로그글들에서 syn queue가 있다길래 코드 뒤져보는데 아무리봐도, request_sock_queue로 관리되는게, 
listen socket에 달려있는 accept queue 밖에 없어서 헷갈렸는데, 

https://stackoverflow.com/questions/63232891/confusion-about-syn-queue-and-accept-queue  여기 정리가 너무 잘되있음 


tfo cookie등 별도 로직은 일단 배제한 설명. 

```
1) connect -> syn 전달
2) listen -> (syn을 받는 경우, request_sock 생성 & ehash에 저장 & syn+ack전달 
   참고) https://elixir.bootlin.com/linux/v6.0.12/source/net/ipv4/inet_hashtables.c#L519
3) client쪽에서는 해당 syn+ack 전달받으면, ack 보내주면서 connect 함수 나옴 (실제 connect가 됬다고 가정하고 fd형태로 이 시점에 떨궈놓음)
4) server쪽에서 ack받으면, ehash에서 제거 & qlen 감소시키고 listen 소켓에 accept queue에 달아둠
5) 실제 accpet하면, accept queue에서 해당 request_sock 뺌 & 커널 레벨에서 do_accept 호출하면서 sock 객체 생성, tcp소켓 관련 file 생성 &  유저한테 fd반환함 
```


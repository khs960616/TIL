```c
/*
만약 현재 task에 이미 plug가 설정되어있다면 아무것도하지 않고 나간다.
task에 plug가 설정되있지 않다면 초기화 해준다. 
*/

blk_start_plug_nr_ios(plug, 1);
  ... 
	plug->mq_list = NULL;   // 플러그가 모아둔 요청들의 리스트 헤더 
	plug->cached_rq = NULL; // 최근에 할당했던 request를 재활용하기 위함 
	plug->nr_ios = min_t(unsigned short, nr_ios, BLK_MAX_REQUEST_COUNT); //   nr_ios 또는 blk_max_request_count값중 작은 값으로 설정
	plug->rq_count = 0;       // 플러그에 쌓인 요청의 개수 
	plug->multiple_queues = false; // 하나의 플러그가 여러 hctx를 사용중인지 여부 (단일 디바이스에 보내는 요청이라면 false)
	plug->has_elevator = false;       // plug에 포함된 요청들이 io 스케쥴러를 통과하는지 여부 
	plug->nowait = false;                  // plug가 non-blocking 여부인지 나타냄 (REQ_NOWAIT 플래그가 걸린 i/o만 모였는지)
	INIT_LIST_HEAD(&plug->cb_list); // plug 완료시 실행할 콜백 리스트 

```

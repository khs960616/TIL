## SLAB Allocator

SunOS 5.4 kernel memory에서 처음 소개된 allocator

```
However, in many cases the cost of initializing and
destroying the object exceeds the cost of allocating
and freeing memory for it
```

객체 초기화, 소멸 비용이 allocate 비용보다 더 큰 경우도 상당히 많음. 따라서 자주 사용되는 객체의 경우, 
allocator 자체의 성능 증가만큼, object 캐싱을 해두는 것 또한 성능 향상에 많은 도움이됨 

### Object Caching

자주 allocate / free되는 객체를 다루기 위한 테크닉

객체가 사용될 때마다 생성/소멸을 반복하지 않도록, 객체에서 변하지 않는 부분을 보존시켜놓는 것이 핵심 아이디어.

예를 들어, mutex_init등과 같이 초기화 비용이 큰 function들은 처음 할당할때만 호출되도록 하고, 사용을

다해도 mutex_destory등은 호출하지 않고 캐싱해두면, 자주 alloc/free될때 비용을 줄일 수 있음 (lock, mutex, ref count, read-only data, list of other obj 등의 초기화 비용)

---

The design of an object cache is
straightforward:

1) To allocate an object
```
if (there’s an object in the cache)
  take it (no construction required);
else {
  allocate memory;
  construct the object;
}
```

2) To free an object
```
return it to the cache (no destruction required);
```

3)  To reclaim memory from the cache:
```
take some objects from the cache;
destroy the objects;
free the underlying memory;
```

요약: 초기화 및 소멸자 호출 비용이 allocate비용보다 더 큰 경우가 많으니, 이것을 한번만 호출되게해서 캐싱해놓고 쓰자는 것이 obj caching 


## 

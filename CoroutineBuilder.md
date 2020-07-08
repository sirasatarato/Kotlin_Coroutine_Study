# 코루틴 빌더
> 사용된 코루틴스코프에 정의된 CoroutineContext 를 기반으로 필요한 코루틴을 생성한다.

## 종류
- launch
- async
- coroutineScope
- withContext
- etc...

## launch
- 반환형: Job
- Job: launch로 실행된 코루틴 블록을 저장해서 제어 가능케 해주는 객체

#### 코드
```
val job = launch { ... }

job.join() // 완료 대기
```

```
val job1 : Job = launch { ... }
val job2 = launch { ... }

job1.join()
job2.join()
or
joinAll(job1, job2)
```


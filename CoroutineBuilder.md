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

## async
- 반환형: Deferred
- Deferred: async 실행된 코루틴 블록을 저장해서 제어 가능케 해주는 동시에 코루틴 블록에서 계산된 결과값을 반환 받을 수 있다.

#### 코드
```
val deferred : Deferred<T> = async {
    ...
    "result" // 결과값
}

val msg = deferred.await() // 완료 대기후 결과값 반환
println(msg) // result 출력
```

```
val deferred1 = async {
    ...
    "result1"
}

val deferred2 = async {
    ...
    "result2"
}

val result1 = deferred1.await()
val result2 = deferred2.await()
or
val result3 = awaitAll(deferred1, deferred2)

println("$result1 , $result2") // result1 , result 2 출력
println("$result3") // result3 출력
```


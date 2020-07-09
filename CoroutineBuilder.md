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

### 지연실행
> launch 코루틴 블록과 async 코루틴 블록을 처리 시점을 뒤로 미룰수 있다.
```
val job = launch (start = CoroutineStart.LAZY) { ... }
or
val deferred = async (start = CoroutineStart.LAZY) { ... }

job.start()
or
job.join()

deferred.start()    // 블록의 수행 결과를 반환하지 않고 코루틴 블록이 완료 되는 것을 기다리지 않는다.
deferred.await()
```

## coroutineScope
> coroutineScope{ } 빌더: 어떤 코루틴들을 위한 사용자 정의 스코프가 필요한 경우에 사용  
이 빌더를 통해 생성 된 코루틴은 모든 자식 코루틴들이 끝날때까지 종료되지 않는 스코프를 정의하는 코루틴이다.

```
fun main(args: Array<String>) = runBlocking {
    launch {
        delay(200L)
        println("Task from runBlocking")
    }

    coroutineScope {
        launch {
            delay(500L)
            println("Task from nested launch")
        }
        delay(100L)
        println("Task from coroutine scope")
    }
    println("Coroutine scope is over")
}

Task from coroutine scope
Task from runBlocking
Task from nested launch
Coroutine scope is over
```

## withContext
> withContext 함수는 로직이 담긴 코드블럭과 이것이 실행될 코루틴 컨텍스트를 함수의 인자로 받습니다.

- NonCancellable: object 이므로 구현 객체이고 AbstractCoroutineContextElement를 상속하는 컨텍스트 요소
  - isActive: NonCancellable 객체 안에 있는 isActive 함수는 항상 true를 반환한다.

```
fun main(args: Array<String>) = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                delay(1000)
                println("main : I'm running finally!")
            }
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

I'm sleeping 0 ...
I'm sleeping 1 ...
I'm sleeping 2 ...
main : I'm tired of waiting!
main : I'm running finally!
main : Now I can quit.
```

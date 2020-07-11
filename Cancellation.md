# 취소
> 코루틴에서 중단 함수는 실행 중 취소 가능한 구간마다 취소 요청이 있었는지 확인하고 요청이 있었다면 실행을 즉시 취소하도록 구현되어야 한다.  
만약 취소 됐다면 CancellationException을 발생시키며 종료한다.  
kotlinx.coroutines 라이브러리의 모든 중단함수는 이러한 취소 요청에 대응 하도록 구현되어 있다.

## Builder.cancel()
```
fun cancellingCoroutineExecution() = runBlocking {
    val job = launch {
        repeat(1000) { i ->
            println("job: I'm sleeping $i ...")
            delay(500L)
        }
    }
    
    delay(1300L)
    println("main: I'm tired of waiting!")
    job.cancel()
    job.join()
    // job.cancelAndJoin()
    println("main: Now I can quit.")
}

job: I'm sleeping 0 ...
job: I'm sleeping 1 ...
job: I'm sleeping 2 ...
main: I'm tired of waiting!
main: Now I can quit.
```

> builder 함수로 객체를 생성 후, cancel()를 호출하면 취소할 수 있다. 하지만....

```
fun cancellationIsCooperative() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 10) {
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
  
    delay(1300L)
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
```

## 취소가 가능한 코루틴 블록
#### yield()
> 해당 위치에서 코루틴의 취소 여부를 확인하는 함수

```
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        for (i in 1..10) {
            yield()
            println("I'm sleeping $i ...")
            Thread.sleep(500L)
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}
```

#### isActive
> CoroutineScope의 Boolean 값의 확장 프로퍼티이며, 코루틴 블록이 아직 취소 되지 않았는지 상태를 가지는 속성

```
fun main(args: Array<String>) = runBlocking {
    val job = launch(Dispatchers.Default) {
        for (i in 1..10) {
            if (!isActive) {
                break
            }
            println("I'm sleeping $i ...")
            Thread.sleep(500L)
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}
```

## 코루틴 작업 취소 시 마무리 작업
> 취소 가능한 중단함수들은 취소되면 CancellationException을 발생한다.

#### try-finally
```
fun main(args: Array<String>) = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            println("main : I'm running finally!")
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}
```

#### try-finally + withContext
> 만약 finally에서 어떤 리소스 사용이 마무리를 대기해야 할시 kotlinx.coroutines 패키지의 중단함수를 finally문에서 사용하면 CancellationException이 발생한다.  
이러한 동작이 반드시 필요한 경우 withContext()함수를 사용한다.

```
fun runNonCancellableBlock() = runBlocking {
    val job = launch {
        try {
            repeat(1000) { i ->
                println("job: I'm sleeping $i ...")
                delay(500L)
            }
        } finally {
            withContext(NonCancellable) {
                println("job: I'm running finally")
                delay(1000L)
                println("job: And I've just delayed for 1 sec because I'm non-cancellable")
            }
        }
    }
  
    delay(1300L)
    println("main: I'm tired of waiting!")
    job.cancelAndJoin()
    println("main: Now I can quit.")
}
```

#### use
```
fun main(args: Array<String>) = runBlocking {
    val job = launch {
        SleepingBed().use {
            it.sleep(1000)
        }
    }

    delay(1300L)
    println("main : I'm tired of waiting!")
    job.cancelAndJoin()
    println("main : Now I can quit.")
}

class SleepingBed : Closeable {
    suspend fun sleep(times: Int) {
        repeat(times) { i ->
            println("I'm sleeping $i ...")
            delay(500L)
        }
    }

    override fun close() {
        println("main : I'm running close() in SleepingBed!")
    }
}
```

## 타임아웃
> 해당 작업의 시간이 지정한 시간을 초과했을 경우 자동으로 취소되도록 하는 스코프 빌더

```
// withTimeout()함수는 첫번째 인자로 작업을 수행할 시간, 두번째 인자로 수행할 블록 함수를 받는다.
작업 취소 시 TimeoutCancellationException이 발생한다.

withTimeout(1300L) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500L)
    }
}
```
```
// 시간 내 정상 종료 시 값을 반환할 수도 있고, 만약 시간 내 처리되지 못한다면 null이 반환된다.

val result = withTimeoutOrNull(1300L) {
    repeat(1000) { i ->
        println("I'm sleeping $i ...")
        delay(500L)
    }
    "Done"
}

println("Result is $result")
```

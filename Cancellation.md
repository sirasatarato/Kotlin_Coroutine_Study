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

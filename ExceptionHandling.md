# Exception handling
> 취소 동작 중 예외가 발생하거나 2개 이상의 자식 코루틴들에서 동시에 예외를 처리하는 핸들러

## 예외 전파
> 코루틴 빌더들은 예외를 처리하는 방식에 따라 다음과 같이 두 가지 타입이 있다.

- 예외를 자동으로 전파
  - launch
  - actor
  - 처리되지 않은 예외로 간주

- 사용자에게 노출하여 예외처리를 일임
  - async
  - produce
  - 예외를 처리하는 예외 처리 핸들러에게 그 처리 방식 수행

#### 예외 자동 전파 예시 코드
```
fun main() = runBlocking {
    val job = GlobalScope.launch {
        println("Throwing exception from launch")
        throw IndexOutOfBoundsException()
    }

    job.join()
    println("Joined failed job")
    val deferred = GlobalScope.async {
        println("Throwing exception from async")
        throw ArithmeticException()
    }

    try {
        deferred.await()
        println("Unreached")
    } catch (e: ArithmeticException) {
        println("Caught ArithmeticException")
    }
}
```

## 코루틴 예외 핸들러
> CoroutineExceptionHandler 컨텍스트 요소를 로깅이나 예외처리를 위한 코루틴의 범용적인 예외 처리 블록으로 사용할 수 있다.

- CoroutineExceptionHandler: JVM에서 ServiceLoader를 통해 등록함으로서 모든 코루틴들을 위한 범용 예외 처리 핸들러를 재정의 할 수 있다.
- uncaughtExceptionPreHandler: Android에 설치된 범용 코루틴 예외 처리기

```
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception")
    }
    val job = GlobalScope.launch(handler) {
        throw AssertionError()
    }
    val deferred = GlobalScope.async(handler) {
        throw ArithmeticException()
    }

    joinAll(job, deferred)
}

Caught java.lang.AssertionError
```


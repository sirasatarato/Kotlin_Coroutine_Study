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

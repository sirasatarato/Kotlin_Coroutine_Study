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

## 취소와 예외
> 코루틴은 내부적으로 취소하기 위해서 CancellationException을 사용하는데,  
이런 예외들은 모든 핸들러들이 무시하므로 catch 블록을 이용하여 부가적인 디버깅 정보 용으로만 사용될 수 있다.  

```
fun main() = runBlocking {
    val job = launch {
        val child = launch {
            try {
                delay(Long.MAX_VALUE)
            } catch (e: Exception) {
                e.printStackTrace()
            } finally {
                println("Child is cancelled")
            }
        }
        yield()
        println("Cancelling child")
        child.cancel()
        child.join()
        yield()
        println("Parent is not cancelled")
    }
    job.join()
}

Cancelling child
Child is cancelled
Parent is not cancelled
kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelling}@26aa12dd
```

> 만약 코루틴이 CancellationException이외의 예외를 만나면 부모를 취소시키게 된다. 이러한 것을 방지하기 위해 핸들러를 사용한다.

```
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception")
    }
    val job = GlobalScope.launch(handler) {
        launch {
            try {
                delay(Long.MAX_VALUE)
            } finally {
                withContext(NonCancellable) {
                    println("Children are cancelled, but exception is not handled until all children terminate")
                    delay(100)
                    println("The first child finished its non cancellable block")
                }
            }
        }
        launch {
            delay(10)
            println("Second child throws an exception")
            throw ArithmeticException()
        }
    }
    job.join()
}

Second child throws an exception
Children are cancelled, but exception is not handled until all children terminate
The first child finished its non cancellable block
Caught java.lang.ArithmeticException
```

## 감독
> 취소는 전체 코루틴 계층을 통해 전파되는 양방향 관계이다.  
하지만 단방향 취소가 필요한 경우, 예를들어 UI 컴포넌트가 종료되면 모든 자식들의 Job도 취소된다.

#### 감독 작업
> 일반적인 Job과 유사하지만 예외로 인한 취소가 아래 방향(부모 -> 자식)으로만 전파된다.

```
fun main() = runBlocking {
    val supervisor = SupervisorJob()
    with(CoroutineScope(coroutineContext + supervisor)) {
        val firstChild = launch(CoroutineExceptionHandler { _, _ ->  }) {
            println("First child is failing")
            throw AssertionError("First child is cancelled")
        }

        val secondChild = launch {
            firstChild.join()
            println("First child is cancelled: ${firstChild.isCancelled}, but second one is still active")
            try {
                delay(Long.MAX_VALUE)
            } finally {
                println("Second child is cancelled because supervisor is cancelled")
            }
        }

        firstChild.join()
        println("Cancelling supervisor")
        supervisor.cancel()
        secondChild.join()
    }
}

First child is failing
First child is cancelled: true, but second one is still active
Cancelling supervisor
Second child is cancelled because supervisor is cancelled
```

#### 감독 범위
> 취소를 단방향으로 전파하며 오직 자신이 실패했을때만 모든 자식을 취소하는 스코프

```
fun main() = runBlocking {
    try {
        supervisorScope {
            launch {
                try {
                    println("Child is sleeping")
                    delay(Long.MAX_VALUE)
                } finally {
                    println("Child is cancelled")
                }
            }
            yield()
            println("Throwing exception from scope")
            throw AssertionError()
        }
    } catch(e: AssertionError) {
        println("Caught assertion error")
    }
}
```

#### 감독중인 코루틴들에서의 예외
> 일반적인 Job 과 Supervisor Job의 중요한 차이점은 예외 처리 방식이다.  
각각의 자식은 예외처리 매커니즘을 통해서 각자의 예외를 처리한다는 점은 동일하지만 자식의 예외가 부모로 전파되지 않는다는 차이점이 있다.

```
fun main() = runBlocking {
    val handler = CoroutineExceptionHandler { _, exception ->
        println("Caught $exception")
    }
    supervisorScope {
        launch(handler) {
            println("Child throws an exception")
            throw AssertionError()
        }
        println("Scope is completing")
    }
    println("Scope is completed")
}

Scope is completing
Child throws an exception
Caught java.lang.AssertionError
Scope is completed
```

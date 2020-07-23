# 플로우 - 2
## 플로우 컨텍스트
> 플로우의 수집은 항상 호출한 코루틴의 컨텍스트 안에서 수행된다.  
이러한 플로우의 특성은 컨텍스트 보존이라고 한다.

```
fun log(string: String) {
    println("${Thread.currentThread().name} $string")
}

fun foo(): Flow<Int> = flow {
    log("Started foo flow")
    for (i in 1..3) {
        emit(i)
    }
}

fun main() = runBlocking {
    foo().collect { value -> log("Collected $value") }
}

main Started foo flow
main Collected 1
main Collected 2
main Collected 3
```

## withContext를 통한 잘못 된 방출
> withContext는 코루틴을 사용하는 코드에서 컨텍스트를 전환하기 위해서 사용된다.  
하지만 flow{} 빌더 내부의 코드는 컨텍스트 보존 특성을 지켜야하기 때문에 다른 컨텍스트에서 값을 방출하는 것이 허용되지 않는다.

```
fun foo(): Flow<Int> = flow {
    withContext(Dispatchers.Default) {
        for (i in 1..3) {
            Thread.sleep(100)
            emit(i)
        }
    }
}

fun main() = runBlocking {
    foo().collect { value -> println(value) }
}
```

## flowOn 연산자
> 플로우에서 컨텍스트 전환을 동작해주는 연산자

```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        Thread.sleep(100)
        log("Emitting $i")
        emit(i)
    }
}.flowOn(Dispatchers.Default)

fun main() = runBlocking {
    foo().collect { value -> log("Collected $value") }
}

DefaultDispatcher-worker-1 Emitting 1
main Collected 1
DefaultDispatcher-worker-1 Emitting 2
main Collected 2
DefaultDispatcher-worker-1 Emitting 3
main Collected 3
```

## 버퍼링
> buffer 연산자는 방출 코드가 수집 코드와 동시에 수행되도록 만들어 주는 연산자

```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking {
    val time = measureTimeMillis {
        foo()
                .buffer()
                .collect { value ->
                    delay(300)
                    println(value)
                }
    }
    println("Collected in $time ms")
}

1
2
3
Collected in 1100 ms
```

## 병합
> conflate 연산자는 수집기의 처리가 너무 느릴 경우, 방출 된 중간 값들을 스킵하는 연산자

```
fun main() = runBlocking {
    val time = measureTimeMillis {
        foo()
                .conflate()
                .collect { value ->
                    delay(300)
                    println(value)
                }
    }
    println("Collected in $time ms")
}

1
3
Collected in 788 ms
```

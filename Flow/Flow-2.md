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

## 최신 값 처리
> xxxLatest 연산자는 새로운 값이 방출될 때 마다 느린 수집기를 취소하고 재시작할 수 있도록 새로운 값이 방출되면 그들의 코드블록을 취소시키는 연산자

```
val time = measureTimeMillis {
    foo()
        .collectLatest { value ->
            println("Collecting $value") 
            delay(300)
            println("Done $value") 
        } 
}   
println("Collected in $time ms")

Collecting 1
Collecting 2
Collecting 3
Done 3
Collected in 716 ms
```

## 다중 플로우 합성
#### zip
> 두개의 플로우들의 값들을 병합하는 연산자

```
val nums = (1..3).asFlow()
    val strs = flowOf("one", "two", "three")
    nums.zip(strs) { a, b -> "$a -> $b" }
            .collect { println(it) }

1 -> one
2 -> two
3 -> three
```

#### Combine
> combine 연산자는 어떤 플로우가 어떤 연산이나 상태의 최근 값을 나타낼 때, 그 플로우의 최근 값에 추가 연산을 수행하거나 또는 별도의 업스트림 플로우가 값을 방출할 때마다 다시 그 추가 연산을 수행해야 할 수 있다.

```
val nums = (1..3).asFlow().onEach { delay(300) }
    val strs = flowOf("one", "two", "three").onEach { delay(400) }
    val startTime = System.currentTimeMillis()
    nums.combine(strs) { a, b -> "$a -> $b" }
            .collect { value ->
                println("$value at ${System.currentTimeMillis() - startTime} ms from start")
            }

1 -> one at 470 ms from start
2 -> one at 684 ms from start
2 -> two at 890 ms from start
3 -> two at 985 ms from start
3 -> three at 1292 ms from start
```

## flatMapConcat
> 다음 플로우의 수집을 시작하기 전에 현재 플로우가 완료될 때까지 기다리는 연산자

```
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    (1..3).asFlow().onEach { delay(100) }
            .flatMapConcat { requestFlow(it) }
            .collect { value ->
                println("$value at ${System.currentTimeMillis() - startTime} ms from start")
            }
}

fun requestFlow(i: Int): Flow<String> = flow {
    emit("$i: First")
    delay(500)
    emit("$i: Second")
}

1: First at 143 ms from start
1: Second at 646 ms from start
2: First at 747 ms from start
2: Second at 1249 ms from start
3: First at 1351 ms from start
3: Second at 1852 ms from start
```

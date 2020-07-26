# 플로우 - 3
## 플로우 예외 
#### 수집기의 try & catch
```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    try {
        foo().collect { value ->         
            println(value)
            check(value <= 1) { "Collected $value" }
        }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}  
```

#### 예외 투명성
> flow{} 빌더 코드 블록 안에서 try/catch 블록으로 예외를 처리한 후 값을 방출하는 것은 예외 투명성을 위반한다.  
방출 로직은 이러한 예외 투명성을 보존하기 위해서 catch 연산자를 사용할 수 있으며 이를 통해 그 예외 처리 로직의 캡슐화가 가능하다.  
catch 연산자의 구현 블록은 예외를 분석하고 발생한 예외의 타입에 따라 각기 다른 대응이 가능하다.

- throw 연산자를 통한 예외 다시 던지기
- catch 로직에서 emit 을 사용하여 값 타입으로 방출
- 다른 코드를 통한 예외 무시, 로깅, 기타 처리

```
fun foo(): Flow<String> = 
    flow {
        for (i in 1..3) {
            println("Emitting $i")
            emit(i) // emit next value
        }
    }
    .map { value ->
        check(value <= 1) { "Crashed on $value" }                 
        "string $value"
    }

fun main() = runBlocking<Unit> {
    try {
        foo()
            .catch { e -> emit("Caught $e") }
            .collect { value -> println(value) }
    } catch (e: Throwable) {
        println("Caught $e")
    } 
}

Emitting 1
string 1 
Emitting 2 
Caught java.lang.IllegalStateException: Crashed on 2
```

#### catch 예외 투명성
> 예외 투명성을 지키는 catch 중간 연산자는 오직 업 스트림에서 발생하는 예외들에 대해서만 동작하며 다운 스트림에서 발생한 예외에 대해서는 처리하지 않는다.

```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking<Unit> {
    foo()
        .catch { e -> println("Caught $e") } // does not catch downstream exceptions
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}      
```

## 플로우 종료
> 플로우의 수집이 종료되면 그 이후 동작을 수행해야 할 수 있습니다.  
이는 Imperative 방식과 Declarative 방식으로 구현한다.

#### Imperative finally block
> 수집할 때 try/catch에 추가적으로 수집 종료 시 실행할 코드를 finally 블록을 통해 정의할 수 있다.

```
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking {
    try {
        foo().collect { value -> println(value) }
    } finally {
        println("Done")
    }
}

1 
2 
3 
Done
```

#### Declarative handling
> onCompletion 연산자는 catch와는 달리 예외를 처리하지는 않는다.  
예외는 여전히 다운 스트림으로 전달되고 결국 onCompletion 연산자를 거쳐 catch 연산자로 처리된다.

```
fun foo(): Flow<Int> = flow {
    emit(1)
    throw RuntimeException()
}

fun main() = runBlocking {
    foo()
        .onCompletion { cause -> if (cause != null) println("Flow completed exceptionally") }
        .catch { cause -> println("Caught exception") }
        .collect { value -> println(value) }
}
```

- 업 스트림 예외에 국한됨
> catch 연산자와 동일하게 onCompletion연산자도 업 스트림에서 전달되는 예외만 식별하고 처리할 수 있으며 다운 스트림의 예외는 알지 못한다.

```
fun foo(): Flow<Int> = (1..3).asFlow()

fun main() = runBlocking<Unit> {
    foo()
        .onCompletion { cause -> println("Flow completed with $cause") }
        .collect { value ->
            check(value <= 1) { "Collected $value" }                 
            println(value) 
        }
}
```

## 플로우 실행
> 어떤 소스로부터 발생하는 비동기 이벤트는 플로우를 통해 쉽게 표현할 수 았다.   
이러한 경우를 위해서 일반적으로 들어오는 이벤트들에 대응하는 처리 코드를 addEventListener를 통해 등록하고 이후 필요한 일을 진행해 가는 방식을 사용하곤 하는데 플로우 에서는onEach 연산자가 대신한다.  
하지만 onEach 는 중간 연산자이기 때문에 플로우 수집을 시작시키기 위해서 종단 연산자가 필요하다.  

```
(1..3).asFlow().onEach { delay(100) }
        .onEach { event -> println("Event: $event") }
        .collect()
println("Done")

Event: 1 
Event: 2 
Event: 3 
Done
```

> launchIn 종단 연산자는 플로우의 수집을 다른 코루틴에서 수행할 수 있게하는 연산자

```
(1..3).asFlow().onEach { delay(100) }
        .onEach { event -> println("Event: $event") }
        .launchIn(this)
println("Done")

Done
Event: 1
Event: 2
Event: 3
```

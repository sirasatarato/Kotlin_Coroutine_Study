# 플로우 - 1
주의: 플로우의 내용이 방대한 분량을 차지하기 때문에 3등분으로 나눠 정리하였습니다.
> 플로우는 어떤 연산 후 두 개 이상의 값을 반환하는 중단함수

## 두 개 이상의 값을 반환하는 방법
#### forEach
> 다중 값은 컬렉션으로 나타낼 수 있다.  
대표적으로는 리스트가 있다.

```
fun main() {
    listOf(1, 2, 3).forEach { value -> print("$value ") }
}

1 2 3 
```

#### 시퀀스
> CPU 연산이 요구되는 연산을 바로 수행하지 않고 나중에 처리함으로서 CPU 효율을 높이는 객체

```
fun foo(): Sequence<Int> = sequence {
    for (i in 1..3) {
        Thread.sleep(100)
        yield(i)
    }
}

fun main() {
    foo().forEach { value -> print("$value ") }
}

1 2 3 
```

#### 중단 함수
> 중단 함수를 코루틴 스코프에서 호출 하여 메인 스레드의 정지 없이 실행할 수 있고 그 결과를 리스트로 반환한다.

```
suspend fun foo(): List<Int> {
    delay(1000)
    return listOf(1, 2, 3)
}

fun main() = runBlocking {
    foo().forEach { value -> print("$value ") }
}

1 2 3 
```

**이전 예제와 Flow와의 차이점**
1. Flow 타입을 생성은flow {} 빌더를 이용
2. flow { ... } 블록 안의 코드는 중단 가능
3. foo() 함수는 더이상 suspend 로 마킹 되지 않음
4. 결과 값들은 flow 에서 emit() 함수를 이용하여 방출됨
5. flow 에서 방출된 값들은 collect 함수를 이용하여 수집됨

## 콜드 스트림
> 플로우는 콜드 스트림이다.  
flow{} 빌더 내부의 코드 블록은 플로우가 수집되기 전까지는 실행되지 않는다.

```
fun foo(): Flow<Int> = flow {
    print("Flow started ")
    for (i in 1..3) {
        delay(100)
        emit(i)
    }
}

fun main() = runBlocking {
    println("Calling foo...")
    val flow = foo()
    println("Calling collect...")
    flow.collect { value -> print("$value ") }
    println("\nCalling collect again...")
    flow.collect { value -> print("$value ") }
}

Calling foo...
Calling collect...
Flow started 1 2 3 
Calling collect again...
Flow started 1 2 3 
```

## 플로우 취소
> 플로우는 코루틴의 취소 매커니즘을 준수하지만  
플로우 인프라스트럭쳐는 부가적으로 취소 지점을 제공하는 기능은 없다.  
플로우가 취소 가능한 중단함수에서 중단 되었을 때만 취소 가능하다.

```
fun foo(): Flow<Int> = flow {
    for (i in 1..3) {
        delay(100)
        println("Emitting $i")
        emit(i)
    }
}

fun main() = runBlocking {
    withTimeoutOrNull(250) { 
        foo().collect { value -> println(value) }
    }
    println("Done")
}

Emitting 1
1
Emitting 2
2
Done
```

## 플로우 빌더
> 코루틴 프레임워크에는 플로우 정의를 돕기위한 다양한 다른 빌더들이 있다.
- flowOf{} 빌더: 고정된 값들을 방출하는 플로우를 정의
- .asFlow() 확장 함수: 다양한 컬렉션들과 시퀀스들을 플로우로 변환

```
(1..3).asFlow().collect { value -> println(value) }
```

## 연산자
#### 플로우 중간 연산자
> 플로우도 연산자를 사용할 수 있다.  
중간 연산자는 업스트림 플로우에 적용되어 다운스트림 플로우를 반환하며 콜드 타입으로 동작한다.  
중간 연산자의 호출은 중단 함수가 아니므로 새롭게 변형된 플로우를 즉시 반환한다.  
시퀀스와의 차이점은 이 연산자들로 수행되는 코드 블록에서 중단 함수들을 호출 할 수 있다는 점이다.

```
suspend fun performRequest(request: Int): String {
    delay(1000)
    return "response $request"
}

fun main() = runBlocking {
    (1..3).asFlow()
            .map { request -> performRequest(request) }
            .collect { response -> println(response) }
}

response 1
response 2
response 3
```

#### 변환 연산자
> transform 연산자는 map처럼 단순한 변환이나 혹은 복잡한 다른 변환들을 구현하기 위해 사용한다.  
transform 연산자를 사용하여 임의의 횟수로 임의의 값들을 방출할 수 있다.  


```
suspend fun performRequest(request: Int): String {
    delay(1000)
    return "response $request"
}

fun main() = runBlocking {
    (1..3).asFlow()
            .transform { request ->
                emit("Making request $request")
                emit(performRequest(request))
            }
            .collect { response -> println(response) }
}

Making request 1
response 1
Making request 2
response 2
Making request 3
response 3
```

#### 크기 제한 연산자
> take같은 크기 제한 중간 연산자는 정의된 크기를 초과하면 실행을 취소한다.   

```
// 아래 예제에서 예외가 발생한 이유는 코루틴에서 취소는 언제나 예외를 발생시키는 방식으로 수행하기 때문이다.

fun numbers(): Flow<Int> = flow {
    try {
        emit(1)
        emit(2)
        println("This line will not execute")
        emit(3)
    } finally {
        println("Finally in numbers")
    }
}

fun main() = runBlocking {
    numbers()
            .take(2)
            .collect { value -> println(value) }
}

1
2
Finally in numbers
```


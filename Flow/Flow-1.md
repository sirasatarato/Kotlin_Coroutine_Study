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


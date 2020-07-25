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

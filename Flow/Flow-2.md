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

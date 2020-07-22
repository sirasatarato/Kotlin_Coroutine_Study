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


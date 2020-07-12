# 채널
> 지연 값을 사용하면 서로 다른 코루틴들이 손쉽게 하나의 값을 공유할 수 있다.  
채널을 이용하면 서로 다른 코루틴들간에 데이터 스트림을 공유할 수 있다.  
채널은 BlockingQueue와 유사하게 동작하지만 값을 넣을 땐 put() 대신 send()를 사용하고, 값을 꺼낼 때 take() 대신 recieve()를 사용한다.

```
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
    }

    repeat(5) {
        println(channel.receive())
    }
    println("Done!")
}
```

## 닫기
> 큐와 달리 채널은 닫을 수 있고, 닫힌 채널은 더이상 값이 전달되지 않는다.  
channel.close()

```
fun main() = runBlocking {
    val channel = Channel<Int>()
    launch {
        for (x in 1..5) channel.send(x * x)
        channel.close()
    }

    for (y in channel) println(y)
    println("Done!")
}
```

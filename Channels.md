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

## 채널 프로듀서
> 코루틴이 어떤 데이터 스트림을 생성해내는 일은 동시성 코드에서 흔히 접할 수 있던 producer-consumer 패턴의 일부 입니다.  
이러한 프로듀서의 생성 작업을 추상화 하기 위해 채널을 파라미터로 전달받는 생성 함수로 만들 수 있다.  
코루틴에서는 프로듀서의 생성 작업을 용이하게 하는 produce{ } 코루틴 빌더와 이렇게 생성된 프로듀서가 생성하는 값들의 수신을 돕는 consumeEach() 확장 함수를 제공한다.

```
fun main() = runBlocking {
    val squares = produceSquares(5)
    squares.consumeEach { println(it) }
    println("Done")
}

fun CoroutineScope.produceSquares(max: Int): ReceiveChannel<Int> = produce {
    for (x in 1..max) {
        send(x * x)
    }
}
```

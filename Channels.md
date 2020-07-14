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

## 파이프라인
> 하나의 코루틴이 데이터 스트림을 생산해내고 다른 하나 이상의 코루틴들이 이 스트림을 수신 받아 필요한 작업을 수행 한 후 가공된 결과를 다시 전송하는 패턴

```
fun main() = runBlocking {
    val numbers = produceNumbers(5)
    val doubledNumbers = produceDouble(numbers)
    doubledNumbers.consumeEach { println(it) }
    println("Done")
}

fun CoroutineScope.produceNumbers(max: Int): ReceiveChannel<Int> = produce {
    for (x in 1..max) {
        send(x)
    }
}

fun CoroutineScope.produceDouble(numbers: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    numbers.consumeEach { send(it * 2) }
}
```

## Fan-out
> 하나의 채널로 부터 두개 이상의 수신 코루틴들이 데이터를 분배하여 수신 받을 수 있다.

```
fun main() = runBlocking {
    val producer = produceNumbers()
    repeat(5) { launchProcessor(it, producer) }
    delay(950L)
    producer.cancel()
}

fun CoroutineScope.produceNumbers() = produce {
    var x = 1
    while (true) {
        send(x++)
        delay(100L)
    }
}

fun CoroutineScope.launchProcessor(id: Int, channel: ReceiveChannel<Int>) = launch {
    for (msg in channel) {
        println("Processor #$id received $msg")
    }
}
```

## Fan-in
> 두 개 이상의 코루틴들이 동일한 하나의 채널로 데이터를 전송할 수 있다.

```
fun main() = runBlocking {
    val channel = Channel<String>()
    launch { sendString(channel, "Foo", 200L) }
    launch { sendString(channel, "Bar", 500L) }
    repeat(6) {
        println(channel.receive())
    }
    coroutineContext.cancelChildren()
}

suspend fun sendString(channel: SendChannel<String>, text: String, time: Long) {
    while (true) {
        delay(time)
        channel.send(text)
    }
}
```

## Buffered channels
> 송신이 먼저 일어나거나 수신이 먼저 일어나면 송수신자는 중단된다.  
Channel\<T>() 팩토리 함수와 produce { } 빌더는 모두 capacity로 버퍼 사이즈를 설정할 수 있다.  
버퍼는 송신자가 중단되기 전에 버퍼의 수용량만큼 더 송신할 수 있다.  
단, 수용량의 최대치에 도달하면 송신자는 중단됩니다.

```
fun main() = runBlocking {
    val channel = Channel<Int>(4)

    val sender = launch {
        repeat(10) {
            print("Try to send $it : ")
            channel.send(it)
            print("Done\n")
        }
    }

    delay(1000)
    sender.cancel()
}
```

## 채널 공정성
> 2개 이상의 코루틴들이 하나의 채널로 송수신을 수행한다면 실행 순서는 그 호출 순서에 따라 할당되며 FIFO방식으로 스케쥴링한다.  
다시말해, 처음 receive() 를 호출한 코루틴이 데이터를 먼저 수신합니다.

```
data class Ball(var hits: Int)

fun main(args: Array<String>) = runBlocking<Unit> {
    val table = Channel<Ball>()

    launch { player("ping", table) }
    launch { player("pong", table) }

    table.send(Ball(0))
    delay(1000)
    coroutineContext.cancelChildren()
}

suspend fun player(name: String, table: Channel<Ball>) {
    for (ball in table) {
        ball.hits++
        println("$name $ball")
        // Comment out below delay to see the fairness a bit more.
        delay(300)
        table.send(ball)
    }
}
```

## Ticker channels
> 마지막 수신 이후에 지정된 지연 시간이 지나면 Unit 오브젝트를 송신하는 채널

```
fun main() = runBlocking {
    val tickerChannel = ticker(delayMillis = 100, initialDelayMillis = 0)

    var nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Initial element is available immediately: $nextElement")

    nextElement = withTimeoutOrNull(50) { tickerChannel.receive() }
    println("Next element is not ready in 50 ms: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 100 ms: $nextElement")

    println("Consumer pauses for 300ms")
    delay(300)

    nextElement = withTimeoutOrNull(1) { tickerChannel.receive() }
    println("Next element is available immediately after large consumer delay: $nextElement")

    nextElement = withTimeoutOrNull(60) { tickerChannel.receive() }
    println("Next element is ready in 50ms after consumer pause in 150ms: $nextElement")

    tickerChannel.cancel()
}
```

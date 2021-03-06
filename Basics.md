# 기초
> 코루틴은 2018년 10월 29일 Kotlin 1.3부터 추가된 기능이다.

**용어정리**  
- 서브루틴  
함수와 매우 비슷하지만 한 가지 차이점이 있다.   
함수는 일반적으로 값을 CPU의 레지스터를 통해 반환하지만 서브루틴은 반환하지 않는다.

- 함수  
함수는 시작과 끝 지점이 한 곳으로 정해져 있다.  
일반적으로 함수를 호출하면 내부 로직이 끝날 때 까지 다음 코드의 동작이 중지되며 함수의 실행이 다른 외부 요인에 의해 중지되지 않는다.

- 코루틴  
함수와는 다르게 시작과 끝이 상관없이 어느 부분에서라도 시작과 종료가 이루어질 수 있다.   
실행을 일시중지하고 다른 코루틴으로 이동할 수도 있다.  
간단한 문법으로 비동기 태스크를 처리할 수 있으며 기존의 스레드를 사용하는 것보다 훨씬 적은 자원을 소비한다.  

## Thread vs Coroutine
- Thread  
OS의 Native Thread에 직접 링크되어 동작하여 많은 시스템 자원을 사용한다.  
Thread간 전환 시에도 CPU의 상태 체크가 필요하므로 비용이 발생한다.

- Coroutine  
JetBrains에선 경량 스레드(Light-Weighted-Thread)라고 불린다.  
코루틴간 전환시 Context Switch가 일어나지 않으며 루틴을 언제 실행, 종료할지 지정이 가능하다.  
이렇게 생성한 루틴은 작업 전환 시에 OS의 영향을 받지 않아 그에 따른 비용이 발생하지 않는다.

## 실제 코드
```
"Hello," 출력한 다음 1초 흐르고, "World!" 출력한 다음 2초가 지날 때 종료

fun main() {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    Thread.sleep(2000L)
}

Hello,
World!
```

#### 코드 해설
* GlobalScope: CoroutineScope 중 하나, 프로그램 전반에 걸처 백그라운드에서 실행
* launch: Scope의 확장함수 또는 코루틴 빌더 중 하나, 코루틴을 실행하는 역할 담당
* delay: 코루틴의 작업을 잠시 중지시키는 중단 함수, Thread.sleep과 역할이 같지만 중지시키는 대상이 다름

## 중단 함수(Blocking function)
> 스레드를 멈추는 역할을 수행하는 함수

## runBlocking
> 중단함수를 코드상에서 명시적으로 나타내거나 일반적인 함수에서 중단 함수를 호출하기 위한 코루틴 빌더
```
// delay() 중단함수를 호출하기 위한 코드
fun main() { 
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    runBlocking {
        delay(2000L)
    } 
}

// 위와 같은 코드지만 보다 자연스럽게 나타낸 코드
fun main(args: Array<String>) = runBlocking {
    GlobalScope.launch {
        delay(1000L)
        println("World!")
    }
    println("Hello,")
    delay(2000L)
}

Hello,
World!
```

#### runBlocking 심화
1. runBlocking 같은 코루틴 빌더는 현재 실행 중인 스레드가 호출될 시 코루틴 빌더가 실행됨.
2. runBlocking은 작업 중인 **스레드를 차단하고 혼자 점유**하기 때문에 **남용을 하거나 안드로이드에서 사용하는 것을 권장하지 않습니다.**

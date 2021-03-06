# 코루틴 컨텍스트
> 코루틴은 항상 어떤 context 하에서 실행된다.  
context: CoroutineContext type을 가지며 Kotlin 표준 라이브러리에 정의되었다.  
Coroutine context: 여러개의 요소를 가진 집합이다.  
주요요소는 Job, dispatcher 등이 있다.

## 구조
```
public interface CoroutineContext {
    public operator fun <E : Element> get(key: Key<E>): E?
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R
    public operator fun plus(context: CoroutineContext): CoroutineContext = ...impl...
    public fun minusKey(key: Key<*>): CoroutineContext
}

public interface Key<E : Element>
public interface Element : CoroutineContext {
  public val key: Key<*>
  
  ...overrides...
}
```

#### 역할
- get(): 주어진 key에 해당하는 컨텍스트 요소를 반환  
- fold(): 초기값을 시작으로 제공된 병합 함수를 이용하여 대상 컨텍스트 요소들을 병합한 후 결과를 반환
  - ex) 예를들어 초기값을 0을 주고 특정 컨텍스트 요소들만 찾는 병합 함수(filter 역할)를 주면 찾은 개수를 반환
- plus(): 현재 컨텍스트와 주어진 컨텍스트가 갖는 요소들을 모두 포함하는 컨텍스트를 반환
  - 중복된 요소는 제거한 후 반환
- minusKey(): 현재 컨텍스트에서 주어진 키를 갖는 요소들을 제외한 새로운 컨텍스트를 반환
  - Key: Element 타입을 제네릭 타입으로 가짐
  - Element: CoroutineContext를 상속하며 key를 멤버 속성으로 가짐 

> 코루틴 컨텍스트에는 코루틴 컨텍스트를 상속한 요소들이 등록될 수 있다.  
각 요소들이 등록될 때는 요소의 고유한 키를 기반으로 등록된다.

## 종류
- EmptyCoroutineContext: 특별히 컨텍스트가 명시되지 않을 경우, 가장 기본적인 것들만 만들어진 컨텍스트
- CombinedContext: 두 개 이상의 컨텍스트가 명시되면 컨텍스트 간 연결을 위한 컨테이너 역할을 하는 컨텍스트
- Element: 컨텍트의 각 요소들도 CoroutineContext를 구현

## + 연산자
> 아래 이미지는 GlobalScope.launch{}를 수행할 때 launch 함수의 첫번째 파라미터인 CoroutineContext에 어떤 값을 넘기는지에 따라 변화하는 코루틴 컨텍스트의 상태를 나타낸다.

<img src="https://miro.medium.com/max/1400/1*K9Ky5pV6CMvaULvaxenqIQ.png">

각각의 요소를 + 연산자를 이용해 연결하고 있는데 Element + Element + … 는 결국 하나로 병합 된 CoroutineContext (e.g. CombinedContext)를 만든다.

## Dispatchers
> Dispatchers는 CoroutineContext를 상속받아 어떤 스레드를 이용해서 어떻게 동작할것인지를 미리 정의되었다.  
Coroutine dispatcher는 코루틴의 실행을 특정 하나의 스레드에 한정 시킬 수 있고, thread pool 에 던질 수도 있고, 정의되지 않은채로 실행시킬 수 있다.

## 종류
- Dispatchers.Default: CPU 사용량이 많은 작업에 사용한다.  
메인 스레드에서 작업하기에는 너무 긴 작업에 적합하다.
- Dispatchers.IO: 네트워크, 디스크 사용 할때 사용합니다.  
파일 읽고, 쓰고, 소켓을 읽고, 쓰고 작업을 멈추는것에 최적화되어 있다.
- Dispatchers.Main: 프로그램 메인 스레드, 안드로이드의 경우 UI 스레드
- [그외](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html)

```
fun main(args: Array<String>) = runBlocking<Unit> {
    launch {
        // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) {
        // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) {
        // will get dispatched to DefaultDispatcher
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) {
        // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}

Unconfined : I’m working in thread main
Default : I’m working in thread DefaultDispatcher-worker-2
newSingleThreadContext: I’m working in thread MyOwnThread
main runBlocking : I’m working in thread main
```

1. launch 빌더가 파라미터 없이 호출될 경우: 실행된 코루틴 스코프로부터 상속 받은 컨텍스트를 그대로 이용한다.
2. Dispatchers.Unconfined: 호출 스레드에서 코루틴을 시작하지만  중단점 이후에 코루틴이 재개될 때는 중단 함수를 재개한 스레드에서 수행된다.
3. Dispatchers.Default: 코루틴이 GlobalScope에서 실행될 경우에 사용되며 공통으로 사용되는 백그라운드 스레드 풀을 이용한다. 즉, launch(Dispatchers.Default)와 GlobalScope.launch는 동일한 디스패처를 사용한다.
4. newSingleThreadContext: 새로운 스레드를 생성한다.

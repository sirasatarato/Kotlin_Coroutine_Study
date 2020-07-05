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


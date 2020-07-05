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

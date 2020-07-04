# 코루틴 스코프
> 코루틴의 범위, 코루틴 블록을 묶음으로 제어할수 있는 단위(역할)  
또는 CoroutineContext 하나만 멤버 속성으로 정의하고 있는 인터페이스(코드)

## 종류
- GlobalScope: 미리 정의된 방식으로 프로그램 전반에 걸쳐 백그라운드 에서 동작합니다.
- CoroutineScope: 사용자 정의 스코프, 모든 자식 코루틴들이 끝날때까지 종료되지 않는 스코프

## 구조
```
public interface CoroutineScope {
    /**
     * Context of this scope.
     */
    public val coroutineContext: CoroutineContext
}

코루틴 스코프는 CoroutineScope 의 확장 함수인 코루틴 빌더들로 정의됨.
즉, 빌더들은 CoroutineScope의 함수들인 것이고 
이들이 코루틴을 생성할 때는 소속된 CoroutineScope 에 정의된 CoroutineContext 를 기반으로 필요한 코루틴들을 생성함.
```

## GlobalScope
```
object GlobalScope : CoroutineScope {
    override val coroutineContext: CoroutineContext
        get() = EmptyCoroutineContext
}
```
> GlobalScope는 싱글톤 object로서 EmptyCoroutineContext를 가지고 있다.

- EmptyCoroutineContext: 구현해야할 CoroutineContext 멤버 함수들에 대해서 기본 구현을 한 컨텍스트 
- 이 컨텍스트는 생명주기에 바인딩 된 Job 이 정의되어 있지 않기 때문에 애플리케이션 프로세스와 동일한 생명주기를 갖는다.


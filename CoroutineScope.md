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

## CoroutineScope
```
// 코루틴스코프 함수는 코루틴스코프 인터페이스를 상속받아서 ContextScope를 반환한다.
@Suppress("FunctionName")
public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())
    

// ContextScope는 CoroutineContext를 그대로 가지는 클래스
internal class ContextScope(context: CoroutineContext) : CoroutineScope {
    override val coroutineContext: CoroutineContext = context
}
```
> CoroutineScope에 CoroutineContext가 주어지면 그것을 기반으로 코루틴을 생성함

## 안드로이드에서의 코루틴 스코프
```
class MyActivity : AppCompatActivity(), CoroutineScope {

  // Job을 등록할 수 있도록 초기화
  lateinit var job: Job

  // 기본 Main Thread 정의와 job을 함께 초기화
  override val coroutineContext: CoroutineContext
      get() = Dispatchers.Main + job

  override fun onCreate(savedInstanceState: Bundle?) {
      super.onCreate(savedInstanceState)
      job = Job()
  }

  // 작업 중이던 모든 job을 종 children을 종료 처리
  override fun onDestroy() {
      super.onDestroy()
      job.cancel()
  }
}
```
> 일반적으로 이렇게 사용하지만 여러 방법으로도 사용할 수 있다.

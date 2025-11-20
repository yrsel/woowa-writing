# 반응형 프로그래밍 In Android

반응형 프로그래밍에 대해서 들어보신 적 있으신가요?

위키피디아의 [반응형 프로그래밍](https://en.wikipedia.org/wiki/Reactive_programming) 정의입니다.

> 데이터 스트림과 변경 사항의 전파를 다루는 선언적 프로그래밍 패러다임이다.  
> 정적 또는 동적 데이터 스트림을 쉽게 표현할 수 있으며, 관련 실행 모델 내에 추론된 종속성이 존재함을 전달하여 변경된 데이터 흐름의 자동 전파를 용이하게 한다.

안드로이드 앱 개발에서도 사용자에게 정보를 보여주기 위해 RxJava, LiveData, Flow 등을 활용하여 반응형 프로그래밍을 구현할 수 있습니다.  
필자가 안드로이드 개발자로 참여중인 튜립 프로젝트는 데이터 변화를 감지하고 UI를 자동으로 갱신하기 위해 <u>Android Jetpack</u> 구성요소인 `LiveData`를 활용하고 있습니다.  
튜립에서 LiveData를 선택한 이유는 다음과 같습니다.  
1. 러닝커브가 낮고 모든 팀원들이 사용 경험이 있다.
2. 생명주기 관리를 개발자가 하지 않아도 되기에 그 외 다른 요소들에 집중할 수 있다.
  
하지만 이 글을 작성하는 시점에 `LiveData`에서 `Flow`로 이전을 고민하고 있어 `LiveData`와 `Flow`의 장단점을 비교하고 정리해봅니다.

📌 cf. Android Jetpack : 여러 곳에서 재사용되는 코드를 줄여주고 여러 안드로이드 버전, 기기에서 일관된 작업을 할 수 있도록 도움을 주는 라이브러리 모음

###  예상 독자
- 안드로이드 앱 개발에 관심있는 분들
- 안드로이드에서의 반응형 데이터 스트림이 무엇인지 대략적으로 알고 계신 분들
- `LiveData`를 사용하는 환경에서 `Flow`로 기술 전환을 고려하고 계신 분들


## LiveData: 안드로이드 생명주기에 특화된 데이터 홀더

`LiveData`가 무엇인지, 장단점과 사용 예시를 통해 알아보겠습니다.

안드로이드 공식 문서를 살펴보면 `LiveData`는 주어진 생명주기 내에서 관찰할 수 있는 데이터 홀더 클래스로 정의됩니다.  
`LiveData`는 다음과 같은 특징이 있습니다.
-  Observer 패턴으로 구현되어 있어 데이터 변경 시, Observer 객체가 개발자 대신 UI 업데이트를 진행합니다.  
-  생명주기를 인식합니다.  
    -  연결된 생명주기가 끝나면 자원을 정리하여 메모리 누수 문제가 없습니다.  
    -  화면이 활성화되지 않은 상태에서는 데이터를 수신하지 않습니다.  
    -  화면이 활성화된다면 최신 데이터를 수신합니다.  

### 사용 예시

ViewModel
```kotlin
class FavoritePlaceViewModel : ViewModel() {
    private val _uiState: MutableLiveData<FavoritePlaceUiState> = MutableLiveData()
    val uiState: LiveData<FavoritePlaceUiState> get() = _uiState
}
```

UI(Activity, Fragment)
```kotlin
class FavoritePlaceFragment {
    private val viewModel: FavoritePlaceViewModel by viewModels()

    override fun onViewCreated(
        view: View,
        savedInstanceState: Bundle?,
    ) {
        viewModel.uiState.observe(viewLifecycleOwner) { state ->
                ...
        }
    }
}
```
비즈니스 로직을 처리하기 위한 ViewModel 내부에서는 변경 가능한 상태인 `MutableLiveData`로 데이터를 수정하고, UI에서는 변경할 수 없는 `LiveData` 타입을 관찰하도록 해 예측 가능한 변경 지점을 구성합니다. 

### LiveData의 장점
- 예시처럼 UI가 생성되는 시점에 Observer를 등록하면 생명주기에 맞게 관리됩니다.
- RxJava, RxKotlin, Flow에 비해 러닝커브가 낮아서 쉽게 적용할 수 있습니다.
### LiveData의 아쉬운점
- 생명주기를 인식할 수 있다는 장점이 있지만 Activity, Fragment에 강하게 결합된다는 단점도 있습니다.
- UI 계층 외부(순수 Kotlin, Data 계층)에서 사용하기 적합하지 않습니다.
- 이벤트를 처리하기 위해 Event Wrapper 또는 SingleLiveData 개념을 활용해야합니다.
- [구글 개발자의 글](https://medium.com/androiddevelopers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54)에 따르면 이상적인 ViewModel은 안드로이드 의존성을 갖지 않는 것을 권장합니다.
   > Don’t let ViewModels (and Presenters) know about Android framework classes

## 튜립에서 `LiveData`에서 `Flow`로 전환을 고민하게 된 4가지 이유
1. 모든 계층에서 사용할 수 없다.
2. Coroutine을 활용한 비동기 작업 처리의 유연성
3. Toast, Snackbar 같은 이벤트 처리 과정에서의 아쉬움
4. Compose로 마이그레이션

##  Flow: Kotlin 표준 비동기 스트림

`Flow`는 무엇이고, `LiveData` 사용하며 아쉬웠던 부분들을 `Flow`에서는 어떻게 해결할 수 있는 지 알아봅시다.
![flow-definition](/images/flow-definition.png)
코틀린 공식문서에서 `Flow`는 값을 순차적으로 정상 방출하거나 예외가 발생할 수 있는 비동기 데이터 스트림이라고 정의합니다.  
`Flow`는 다음과 같은 특징이 있습니다.
- `kotlinx-coroutines-core` 패키지에 포함되어 있습니다.
- publisher-subscriber 구조를 재구성한 구조입니다.
    - publisher : suspend 함수 기반으로 데이터를 발행
    - subscriber : 데이터를 요청
    - subscriber가 데이터를 요청할 때 publisher는 데이터를 발행합니다.
    - publisher가 suspend 함수로 값을 발행하기에 <u>backpressure</u> 제어가 가능합니다.
    - subscriber가 데이터를 빨리 소비하지 못하면, publisher는 suspend 상태로 대기합니다.

📌 cf. [backpressure](https://en.wikipedia.org/wiki/Back_pressure) : 이벤트 기반 아키텍처에서 데이터 흐름을 조절하는 기술, 컴포넌트가 과부화되지 않도록 보장한다.

## `LiveData` 사용할 때 아쉬웠던 점을 하나씩 비교해보겠습니다.

### 1. 모든 계층에서 사용할 수 없다.
`LiveData`는 생명주기를 인식하고 데이터 변경을 화면에 보여주는 역할을 하며 메인스레드에서 작동합니다. 그러므로 UI와 관련없는 계층(데이터 계층)에서의 사용은 좋지 않다고 생각합니다.   
또한, ViewModel에서 안드로이드의 의존성을 최소화하는 것을 권장하고 있습니다.  

`Flow`는 코틀린에 종속적이기에 데이터 계층, UI 계층 전반적으로 사용할 수 있습니다.   
ViewModel에서 안드로이드 의존성을 제거할 수 있어 테스트하기 쉬우며 KMP로의 확장에도 유연하게 대처할 수 있습니다.  
또한, `DataStore`, `Room` 등 <u>여러 `Android Jetpack` 라이브러리에서 `Flow`를 지원합니다.</u>

📌 cf. [안드로이드 공식문서에서 언급](https://developer.android.com/kotlin/flow?hl=en#jetpack)

### 2. Coroutine을 활용한 비동기 작업 처리의 유연성

`LiveData`에서 비동기 작업을 처리하는 방법
```kotlin
private val _nameState = MutableLiveData<UiState<String>>()
val nameState: LiveData<UiState<String>> get() = _nameState

fun loadName() {
    viewModelScope.launch {
        _nameState.value = UiState.Loading
        try {
            val loadName = repository.getName()
            _nameState.value = UiState.Success(loadName)
        } catch (e: Exception) {
            _nameState.value = UiState.Error(e)
        }
    }
}
```
- 비동기 작업을 위해 코루틴 스코프를 직접 호출해야합니다.(viewModelScope.launch)
- 예외 처리를 try-catch 또는 runCatching 으로 직접 관리해야합니다.
- UI 상태 변경을 위해 직접 할당해야 합니다.
- 데이터 변환을 위해 MediatorLiveData, LiveData.map 을 사용해야 합니다.

`Flow`에서 비동기 작업을 처리하는 방법
```kotlin
val nameState: StateFlow<UiState<String>> = flow {
    emit(UiState.Loading)
    val name = repository.getName()
    emit(UiState.Success(name))
}
.catch { e -> emit(UiState.Error(e)) }
.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(5000),
    initialValue = UiState.Empty
)

``` 
- suspend 기반으로 발행하므로, 비동기로 실행할 수 있습니다.
- onStart, catch, map 등 다양한 연산자를 지원하며 선언형으로 표현할 수 있습니다.
- 예외 처리를 위해 catch 연산자를 지원합니다.
- 하나의 파이프라인으로 완성할 수 있습니다.

### 3. Toast, Snackbar 같은 이벤트 처리 과정에서의 아쉬움

`LiveData`에서 이벤트 처리
```kotlin
private val _toastEvent = MutableLiveData<String>()
val toastEvent: LiveData<String> get() = _toastEvent

fun onSubmit() {
    _toastEvent.value = "상품이 등록되었습니다."
}
```
위 코드에서 이벤트가 발생하고나서 <u>Configuration Change</u>가 발생하거나 백스택에 있던 UI가 다시 활성화된다면 어떻게 될까요?
- `LiveData`는 observe할 때 마지막 값을 반환합니다.
- 화면이 보여지고 있지 않을 때는 inActive 상태로 존재하다가 보여지게 된다면 Active 상태로 변화됩니다.
- 결과적으로 이벤트가 발생하지 않았음에도 마지막에 방출했던 이벤트가 방출됩니다.

`LiveData`에서 이벤트 처리 방법 : Event wrapper, EventObserver
```kotlin
open class Event<out T>(private val content: T) {

    var hasBeenHandled = false
        private set 

    fun getContentIfNotHandled(): T? {
        return if (hasBeenHandled) {
            null
        } else {
            hasBeenHandled = true
            content
        }
    }

    fun peekContent(): T = content
}

// ViewModel 
class ProductViewModel : ViewModel() {
    private val _toastEvent = MutableLiveData<Event<String>>()
    val toastEvent: LiveData<Event<String>> get() = _toastEvent

    fun onSubmit() {
         _toastEvent.value = Event("상품이 등록되었습니다.")
    }
}
```
- 구글 개발자가 제시한 `LiveData`에서의 [<u>이벤트 처리를 위한 방법</u>](https://medium.com/androiddevelopers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)입니다.
- 매번 `Event`로 타입을 감싸주는 번거로움을 해결하기 위해 [<u>EventObserver</u>](https://gist.github.com/JoseAlcerreca/e0bba240d9b3cffa258777f12e5c0ae9)를 사용하는 방법도 존재합니다.
- 덕분에 `LiveData`에서 이벤트 처리가 가능하게 됐지만 프로젝트 사용시 위 코드 추가가 필요합니다.

📌 cf. [Configuration Change](https://developer.android.com/guide/topics/resources/runtime-changes) : 시스템이 구성 변경이 발생하는 경우, Activity를 재생성하게 됩니다.  구성 변경의 예시로는 화면 방향 전환, 지역 설정 변경, 다크 모드와 라이트 모드 변경 등 있습니다.

`Flow`에서 이벤트 처리
```kotlin
private val _toastEvent = MutableSharedFlow<String>(replay = 0)
val toastEvent = _toastEvent.asSharedFlow()

fun onSubmit() {
    viewModelScope.launch {
        _toastEvent.emit("상품이 등록되었습니다.")
    }
}
```
- SharedFlow는 replay 값을 설정해 최근 n개의 값을 캐싱해둘 수 있으며 `replay = 0` 의 경우 이전 이벤트를 재방출하지 않습니다.
    - `replay = 0` : 당장 발생한 이벤트만 즉시 전달합니다.
- extraBufferCapacity, onBufferOverflow 같은 설정을 제공해 이벤트 누락, 중복, 지연 전송 등을 세밀하게 제어할 수 있습니다.

### 4. Compose로 마이그레이션

Compose에서 `LiveData` 사용하는 방법
```kotlin
val name by viewModel.nameState.observeAsState()
Text(text = name ?: "")
```
- observeAsState() : LiveData를 관찰하기 시작하고 State를 통해 값을 나타냅니다.
- Compose UI는 remember, collectAsState() 등 Coroutine 기반으로 상태를 감시하지만 `LiveData`는 Coroutine Context에 완전 통합되지 않아, 복잡한 비동기 데이터 흐름을 표현하기 어렵습니다.

Compose에서 `Flow` 사용하는 방법
```kotlin
val name by viewModel.nameState.collectAsStateWithLifecycle()
Text(text = name)
```
- collectAsStateWithLifecycle(), collectAsState() : 값을 수집하여 State로 변환합니다.
- Compose에서 <u>UDF 패턴</u>과 Flow의 pub-sub 구조가 비슷합니다.
- LaunchedEffect, rememberCoroutineScope 등 Coroutine Context 안에서 안전하게 연동할 수 있습니다.

📌 cf. [UDF(단방향 데이터 흐름)](https://developer.android.com/develop/ui/compose/architecture?hl=en#udf) : 상태는 아래로 이동하고 이벤트는 위로 이동하는 디자인 패턴입니다.

### LiveData와 Flow 비교 요약

| 구분           | LiveData                          | Flow                                                     |
|----------------|-----------------------------------|----------------------------------------------------------|
| 플랫폼 종속성  | Android 전용(LifecycleOwner 필요)                      | Kotlin 표준 라이브러리 기반                                              |
| 비동기 처리    | 메인스레드에서 작동(postValue() 비동기 메서드 지원)                    | Coroutine 기반 비동기 처리 (suspend, emit, ...)                |
| 연산자         | 제한적(MediatorLiveData)                            | 다양 (map, filter, combine, debounce 등)                           |
| 데이터 특성 (Cold/Hot)        | Hot Stream(항상 마지막 값 유지)                          | Cold Flow, Hot Flow 모두 지원          |
| 테스트 용이성  | Android 환경 필요            | JVM 단위 테스트 가능                                     |
| UI 바인딩      | XML 기반 ViewSystem 친화적(DataBinding, observe())        | Compose 환경에서 자연스럽게 통합 (collectAsState())                        |

##  결론

`LiveData`를 사용해 개발하며 생명주기를 자동으로 인식하여 데이터 안정성을 보장할 수 있었습니다.  
다만, 현재 튜립 프로젝트는 View System에서 Compose로 마이그레이션을 계획하고 있어 이 과정에서 `Flow`를 도입하는 방향으로 긍정적으로 생각하고 있습니다.    
`Flow`를 사용하면 생명주기를 개발자가 관리해야하기에 메모리 누수가 발생할 수 있다는 단점이 존재하지만 코드 작성시점과 코드 리뷰시점에  이중으로 검토하며 보완할 수 있다고 생각했고 전체적인 내용을 종합해보면 `LiveData`보다 `Flow`를 선택했을 때의 이점이 많다고 느꼈습니다.  
또한, `LiveData`와 `Flow`간 형변환을 해주는 메서드들도 존재해서 점진적으로 마이그레이션도 가능하다고 생각했습니다.  

LiveData와 Flow는 모두 데이터 옵저빙을 위한 훌륭한 도구입니다.  
LiveData는 UI 생명주기에 맞는 데이터 안전성을 위해 만들어졌고, Flow는 비동기 데이터 스트림의 일관성을 위해 설계되었습니다.   
`LiveData`에서 `Flow`로의 전환이 고민된다면 프로젝트에서 중요하게 여기는 부분과 상황을 고려하고 판단하셔서 더 나은 선택하시길 바라겠습니다.  
긴 글을 읽어주셔서 감사합니다. ☘️
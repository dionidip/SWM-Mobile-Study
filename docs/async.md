<!-- title : [Android] 비동기 처리 -->
# 동기 vs 비동기
![](https://images.velog.io/images/dorazi/post/dadf63e9-5994-4967-bc3f-ca0bc173897c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202020-03-31%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%202.29.45.png)
## 동기(Synchronous)
사전 그대로 번역하면 '동시성의' 라는 의미이다. 즉, 요청과 결과가 동시에 일어나는 작업을 동기라고 한다. 동기적으로 작성된 프로그램은 수행하는 작업이 완료될 때까지 다른 작업을 하지 않고 기다린다. 

---
## 비동기(Asynchronous)
동기와 반대로, 동시에 일어나지 않는 것을 의미한다. 그러므로 비동기적으로 작성된 프로그램은 수행하는 작업의 완료와 상관없이 계속해서 다른 작업도 수행할 수 있다. 
Android는 `UI thread`라고 불리는 메인쓰레드가 UI 작업을 관리하고 처리한다. UI thread는 사용자의 앱 사용성에 직결된 요소이기 때문에 항상 막힘없이 처리되어야 하는데, 만약 이 메인쓰레드에서 네트워크나 DB로부터 자료를 받아오는 작업을 수행한다면 어플리케이션은 `ANR`(Application Not Responding) 상태가 될 것이다. 따라서 비동기라는 개념이 중요하며, 여러 작업들을 각각의 쓰레드에 나눠서 block 없이 수행할 수 있도록 설계해야 한다.

### 안드로이드에서 비동기 처리를 위한 방법
Android에서 이런 비동기 처리를 해주기 위해 `RxJava`, `RxKotlin`, `Coroutine` 같은 방법이 존재한다. 이 포스팅에서는 모두 살펴보기 보다는 `Coroutine` 하나만 공부해보자.

---

# Coroutine
> 💡 코루틴(Coroutine)은 구글에서 공식적으로 `AsyncTask`를 deprecated 선언한 후 대체 기술로 사용할 것을 추천하고 있다.

코루틴은 비동기적으로 실행되는 코드를 간소화하기 위해 Android에서 사용할 수 있는 동시 실행 설계 패턴이다. **코루틴은 Kotlin만의 기능이 아니다.** 이미 많은 언어들(C#, Python, Go, JavaScript 등)에서 지원하고 있던 개념이다.
Wikipedia에서는 코루틴을 다음과 같이 정의하고 있다.
> 코루틴은 실행과 재개를 허용함으로써 **비선점형 멀티태스킹**을 위한 서브루틴을 일반화한 컴퓨터 프로그램의 구성 요소이다.

즉, 코루틴은 싱글 쓰레드로 동시성을 제공하고 선점형 멀티테스킹 방식인 쓰레드보다 비용이 적은 멀티테스킹 방식이다.

---
## Coroutine의 기능
### 경량
코루틴은 `Suspend`를 지원하는데, 여러개의 코루틴은 실행한다고 했을 때 코루틴1을 실행하던 중에 Context Switching으로 쓰레드를 바꾸며 코루틴2를 실행하는 것이 아니라 기존 쓰레드 위에서 코루틴1을 정지시키고 코루틴2를 실행한다. 이렇게 하면 단일 쓰레드에서 많은 작업을 수행할 수 있고 메모리 비용 측면에서 이득이다.

### 메모리 누수 감소
코루틴은 `structured concurrency`이론을 따르는데, 새로운 코루틴은 코루틴의 수명을 제한하는 특정 `CoroutineScope`에서만 실행할 수 있다.

### 작업 취소 지원
sunflower 코드를 보면
```kotlin
private fun search(query: String) {
    // Make sure we cancel the previous job before creating a new one
    searchJob?.cancel()
    searchJob = lifecycleScope.launch {
        viewModel.searchPictures(query).collectLatest {
            adapter.submitData(it)
        }
    }
}
```
다음과 같이, 코루틴에서 제공하는 클래스형의 객체인 `searchJob`을 `launch`하기 전에 `cancel` 작업을 수행한다. 코루틴에서는 이런 함수를 기본으로 제공해주고 있다.

### Jetpack 통합
많은 Jetpack 라이브러리에 코루틴은 지원하는 확장프로그램을 포함하고 있다.
특히, `Lifecycle KTX`, `viewModel KTX`, `workManager KTX` 에서 CoroutineScope를 지원해준다.

---
## Coroutine 사용
### coroutine scope
모든 코루틴은 이 `scope` 안에서 실행되어야 한다. 그래야 액티비티나 프래그먼트의 생명주기에 따라 소멸될 때 관련 코루틴을 한번에 취소할 수 있고, 이것은 메모리 누수를 방지한다. scope는 내장된 범위를 사용하거나 필요에 따라 커스텀도 가능하다.
기본적으로 내장된 scope는 아래의 4가지이다.
- GlobalScope : 앱의 생명주기와 함께 동작해서 실행 도중에 별로도 생명주기를 관리해 줄 필요가 없다. 시작부터 종료까지 장기간 실행되는 코루틴 작업에 적합하다.
- CoroutineScope : 서버로부터 파일이나 이미지를 받는 등 필요에 따라 열거나 닫아주는 작업에 적합하다.
- ViewModelScope : Jetpack 아키텍처의 viewModel 컴포넌트를 사용시 ViewModel 인스턴스에서 사용하기 위해 제공되는 scope이다. viewModelScope로 실행되는 코루틴은 ViewModel 인스턴스가 소멸될 때 자동으로 취소된다.
- LifecycleScope : LifecycleScope는 각 Lifecycle 객체에서 정의되고, 이 범위에서 정의된 코루틴은 Lifecycle이 끝날 때 취소된다.

### 사용예시
그럼 이 코루틴을 어떻게 사용하는지 예시코드를 살펴보자.
```kotlin
fun addPlantToGarden() {
    viewModelScope.launch {
        gardenPlantingRepository.createGardenPlanting(plantId)
        _showSnackbar.value = true
    }
}
```
sunflower 코드 중 일부이며, ViewModel() 클래스를 overload하는 `PlantDetailViewModel` 클래스 안에 정의된 멤버함수이다. viewModelScope를 launch 주기함수를 통해 사용하고 있고, 다른 스코프와는 다르게 별도로 cancel()은 호출하지 않고 사용한다.

다른 예시도 보자.
```kotlin
private fun search(query: String) {
    // Make sure we cancel the previous job before creating a new one
    searchJob?.cancel()
    searchJob = lifecycleScope.launch {
        viewModel.searchPictures(query).collectLatest {
            adapter.submitData(it)
        }
    }
}
```
코루틴에서 실행하는 하나의 프로세스를 `Job`이라고 하고, `searchJob`은 Job의 인스턴스이다. 이전에 실행중이던 Job이 있다면 취소해주고, 새로운 Job을 시작하는 코드이다.
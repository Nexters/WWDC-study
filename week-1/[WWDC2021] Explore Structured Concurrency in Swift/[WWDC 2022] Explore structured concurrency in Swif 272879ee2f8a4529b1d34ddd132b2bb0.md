# [WWDC 2022] Explore structured concurrency in Swift

> 영상 1줄 요약: Structured Concurrency를 Swift에서 지원한다.
> 

## 시작에 앞서…

---

시작에 앞서서 알아두어야 할 용어들이 있습니다!

- 흔히 애플 공식문서를 읽다보면, “새로운 실행 context가 생성된다.” 라는 말이 반복되는데.
- ‘실행 context’가 무엇이길래 강조해서 말하는 것일까요?
- 일반적으로 프로그램은 위에서 아래로, top-down 방식으로 실행됩니다.

![Untitled](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/Untitled.png)

- 그래서 위 코드는 print(A) → print(B) → print(C) 순으로 실행되죠.
- 하지만 만일 print(B)를 다음과 같이 `Global Queue` 에서 실행되도록 한다면 어떻게 될까요?

![Untitled](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/Untitled%201.png)

- 코드를 실행하는 흐름이 `print(A) → print(C)` 를 실행하는 흐름과 `print(B)`를 실행하는 흐름으로 분리되고, 두 실행흐름은 동시에 실행됩니다.
- 이렇듯, 코드를 실행하는 새로운 흐름이 생겼을 때, “새로운 실행 context 가 생겼다” 라고 표현합니다.

- `scope` : 변수가 생성된 중괄호(`{}`) 범위

```swift
func anyfunction() {
	let value = "value"
}
```

- 변수 `value`의 scope는 `anyfunction()`입니다.
= `anyfunction()` 을 넘어가면 `value`는 메모리에서 해제됩니다.

## 여러 작업을 병렬적으로 처리하고 싶을 때: `Task` 를 사용해라‼️

---

1. Task는 코드를 비동기적으로 실행할 수 있는 context를 만들어줍니다.
(새로운 Task = 새로운 실행 context)
    
    ```swift
    Task {
    	// 비동기적으로 실행하고 싶은 코드
    }
    ```
    
    - `Task` 를 생성할 때, 중괄호(`{ }`) 안에 작성하는 코드는 비동기적으로 실행됩니다.
2. 각 Task는 다른 실행 context와 동시에 실행됩니다.
3. 내부에서 `async-await`메서드를 호출할 수 있습니다.
4. Swift는 개발자가 버그를 일으킬만한 코드를 가지고 Task를 생성하는지 체크해줍니다.
5. 하지만 async-await 메서드를 호출하는 것만으로는 새로운 실행 context가 생성되지 않으며, 꼭 Task를 생성해야만 새로운 실행 context가 생성됩니다.

## Task의 종류

---

Task의 종류로는 크게 4가지가 있습니다.

1. structured task
    1. async - let
    2. Task Group
2. Unstructured Task
3. Detached Task

### 1. async-let

![스크린샷 2022-07-28 오후 5.32.06.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_5.32.06.jpg)

- async let을 사용하는 방법은 다음과 같이 변수를 선언할 때 앞에 `async` 키워드를 붙이면 됩니다.
- 프로세스는 코드를 실행하다가 `async-let` 구문을 만나면, 새로운 Task를 생성합니다. 이로 인해 새로운 실행 context가 생성됩니다.
- 따라서 `async let` 코드를 실행한 이후에는 2개의 실행 context가 존재하게 됩니다.
    - 첫번째로, async let 이전까지 코드를 실행하던 기존의 실행 context가 존재하고
    두번째로, 기존의 실행 context가 `async let`을 만나서 생성하는 Task(새로운 실행 context)가 존재합니다.
- 새로운 실행 Context `=` 의 오른쪽에 위치한 URLSession.shard.data`()` 를 실행하고,
    
    기존의 실행 context는 `async let` 변수에 placeholder값을 담은 후, 이어서 다음 명령어를 실행합니다.
    
- 그러다 `async let` 변수를 사용해야 하는 위치를 만나면, 새로운 실행 context가 작업을 완료할 때까지 대기합니다.
- 새로운 실행 context는 작업을 모두 완료하면, 작업 결과를  async let 변수에 저장합니다.
- 기존 실행 context는 aysnc let 변수값을 가지고 작업을 처리합니다.

- ***그런데, 위에서 `asyc-let` 코드를 만나서 새로운 Task를 생성한, 기존의 실행 context도 하나의 `Task` 입니다.***
- 즉, 모든 코드는 `Task`에 포함된 것입니다.
- 그리고 `async-let` 구문 처럼,  한 `Task` 에서 새로운 `Task` 를 생성하면, 두 `Task` 간에 부모 자식 관계가 형성됩니다.

- ex)

```swift
class MainViewController: UIViewController {
	override func viewDidLoad() {
		fetchOneThumbnail()
	}
}
```

![스크린샷 2022-07-28 오후 5.52.25.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_5.52.25.jpg)

- 예를 들어서 위 코드는 `fetchOneThumbnail()` 을 실행하는 `MainViewController`  를 나타냅니다.
- 그럼 `MainViewController`를 실행했을 때, `fetchOneThumbnail()` 를 실행하는 `Task`가 존재하게 됩니다.
- 그리고 `fetchOneThumbnail()` 은 2개의 `async-let` 구문을 실행하여, 2개의 `Task`가 새로 생성되며, 
이 `Task`들은 `fetchOneThumbnail()` 을 실행한 `Task` 를 부모로 갖게 됩니다.
- ***이처럼 새롭게 생성된  `Structured Task` 는 기존의 `Task` 와 부모-자식 관계로 연결됩니다.***

### Task 간 부모-자식 관계에서 원칙 - `Structured Task` 특징

1. 자식 Task가 모두 종료된 후에, 부모 Task가 종료될 수 있습니다‼️
    
    → 이 특징 덕분에, 종료되어야 하는 Task가 계속해서 실행되는 현상(Task leaking)을 방지할 수 있습니다.
    
2. 그리고 이 원칙은 Task 실행 중에 오류가 발생했을 때도 지켜집니다.
3. 자식 Task는 부모 Task의 속성(어디 스레드에서 실행되는지, 실행 우선순위 등)을 물려 받습니다.
4. 에러가 발생하면, 형제 Task와 부모 Task에 전파됩니다.

- 예를 들어서, `metadata` 를 다운로드 받는 작업이 실행중에 error가 발생했다면, `fetchOneThumbnail`는 발생한 error를 throw 하고, 함수 실행을 종료합니다.
- 이때 만일 `data` 를 다운로드 받는 Task가 아직 실행 중이라면, 
이 Task를 바로 중단시켜 버리는 것이 아니라, 취소되었다고(`cancelled` ) 표시하고 Task가 완료될 때까지 기다립니다.
- 그리고 취소된 Task는 작업을 끝까지 완료하고, 작업 결과만 사용하지 않고 버립니다.

![스크린샷 2022-07-28 오후 6.08.27.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_6.08.27.jpg)

- 또한, 한 `Task` 를 `cancelled` 표시하면, 그 `Task` 의 모든 자식 `Task` 도 `cancelled` 표시되어 종료됩니다.

![스크린샷 2022-07-28 오후 6.08.58.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_6.08.58.jpg)

- 이렇게 해서, `fetchOneThumbnail()` 메서드는 모든 자식 `Task` 가 종료된 후에야, error throw 합니다.

- 그리고 Task는 바로 종료되지 않기 때문에, Task가 실행중일 때만 해야 하는 작업이 있다면, Task가 취소되었는지 체크하고 그 작업을 수행하도록 만들어야 합니다.
- Task가 취소되었는지 체크하는 메서드
    1. `try Task.checkCancellation`
        - 현재 Task가 cancell됬으면 throw error
    2. `Task.isCancelled`
        1. 현재 Task가 cancell됬으면 return true

## Task Group

---

![스크린샷 2022-07-28 오후 7.30.46.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.30.46.jpg)

- 이 코드는 `async-let`의 특성상, 하나의 thumbnail id에 해당하는 data를 가져오는 작업과 metadata를 가져오는 작업만 동시에 실행됩니다. → 한번에 자식 Task가 최대 2개까지만 생성됩니다.
- 만일, 모든 thumbnail id에 대한 `data`와 `metadata`를 동시에 다운 싶다면, 이때는 `Task Group`을 사용해야 합니다.

### Task Group이란?

- 동시에 실행되어야 하는 `Task`의 개수를 정확하게 모를 때 사용
- 위 예시에서 Task Group을 사용해야 하는 이유
    - 위 예시에서 모든 thumbnail id에 대해서 data와 metadata를 동시에 다운 받도록 할 때, 생성되는 총 Task의 개수는 `ids` 배열의 길이와 같습니다.
    - 이 말인즉슨, 실제로 메서드 호출되어 ids 파라미터 값이 결정되기 전까지는 Task의 개수를 알 수 없습니다.
    → Task Group을 사용해야 합니다.
- `Task Group`을 사용한 코드

![스크린샷 2022-07-28 오후 7.36.04.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.36.04.jpg)

- `withThrowingTaskGroup(of:)` 를 이용하여 `Task Group`을 생성합니다.
    - `withThrowingTaskGroup(of:)` 내부에서 `group.async` 를 호출하면 새로운 Task가 생성되어 group에 추가되며, group에 추가된 Task는 바로 실행됩니다.
    - group에 추가된 Task마다 `fetchOneThumbnail()`을 호출하며, 
    `fetchOneThumbnail()`은 2개의 task ( `data`를 가져오는 `async-let` Task, `metadata` 를 가져오는 `async-let` Task)를 생성됩니다.
    - ***결과적으로 (배열  `ids` 길이 *  2)개의 task가 생성되어, 모든 id에 대한 `data`와 `metadata`를 가져오는 Task가 동시에 실행됩니다.***
- 그리고 group의 모든 `Task`가 실행완료 된 후에야   `withThrowingTaskGroup(of:)` 의 다음 명령어(여기서는 `return thumbnails`)가 실행됩니다.
- `Task`는 랜덤 순서대로 실행 완료됩니다.

### @Sendable

- 그런데 이 코드에서는 문제점이 발생합니다. 
여러 `Task`에서 `thumbnails` 수정하려고 하기 때문에, data race가 발생합니다‼️
- `Task` 이전에는 이런 코드가 있더라도 빌드가 가능했지만, `Task` 를 사용하면 컴파일 타임에 data race가 발생하는 코드인지 아닌지 체크해서 우리가 사전에 알 수 있도록 만들어줍니다.
- 그렇다면, `Task` 에서 data race를 어떻게 탐지하는 것일까요?
    - `***@Sendable` 클로저 활용합니다‼️***
    - Task가 파라미터로 받는 클로저는 `@Sendable` 클로저입니다.
    - `@Sendable` 클로저는 내부에서 mutable 변수를 캡처하지 못하도록 합니다.
    - 따라서 `@Sendable` 클로저 내부에서는 value type (`Int`, `String` 등), `actor` , 동기화 처리된 class 만 사용할 수 있습니다.
    
- 해결 방법

![스크린샷 2022-07-28 오후 8.10.14.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_8.10.14.jpg)

- `Task` 에서는 `id`와 `fetchOneThumbnail(withId:)` 결과를 tuple로 리턴합니다.
- 그리고 `for await loop` 을 이용해서 결과값을 가져와서 `thumbnails` 딕셔너리에 저장합니다.
    - `for await loop` 은 작업이 완료된 Task 순서대로 한번에 하나의 결과값을 가져와서 처리하기 때문에, `thumbnails`에 동시에 접근하지 않습니다.
    

> ***이렇듯 부모-자식 관계를 명확하게 이루면서 생성되는 Task를 Structured Task라고 합니다!***
> 

structured Task가 존재하든, unstructrued Task도 존재합니다.!

## Unstructured Task

---

- unstructured Task가 필요한 때
1. `non async`메서드에서 `async` 코드를 실행해야 할 때 
(some tasks need to launch from non-async contexts)
2. `Task` 가 한정된 범위에서만 실행되는 것이 아니라, 더 오랫동안 실행되도록 하고 싶을 때 
(some tasks live beyond the confines of a single scope)

- 특징
1. 자신을 생성한 Task의 실행 우선순위와 실행 actor를 상속받습니다.
    
    Inherit actor isolation and priority of the origin context)
    
2. 생명주기가 특정 범위로 제한되지 않습니다. 
(자신이 생성된 위치의 scope가 종료되더라도, 계속해서 실행됩니다.)
    
    Lifetime is not confined to any scope
    
3. async-await 메서드가 아니더라도 생성될 수 있습니다.
    
    Can be launched anywhere, even non-async functions
    
4. 작업을 취소(cancelled)하는 작업, 작업이 완료될 때까지 대기하는 작업(awaited)은 수동으로 구현해야 합니다.
Must be manually cancelled or awaited

- 사용 예시

![스크린샷 2022-07-28 오후 8.19.13.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_8.19.13.jpg)

- collection view에서 보여줄 데이터를 가져오기 위해서 delegate메서드에서 `fetchThumbnails()` 라는 `async-await` 메서드를 호출하고 싶지만, delegate 메서드는 `async-await` 메서드가 아니라서 호출할 수 없습니다.
- 이렇듯, `async-await` 메서드가 아닌 곳에서 `async-await` 메서드를 호출하고 싶을 때 `Unstructured Task`를 사용합니다.

![스크린샷 2022-07-28 오후 8.22.08.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_8.22.08.jpg)

- `Task`를 하나 생성하고, 내부에서 async-await 메서드를 호출하면 됩니다.
- 코드가 실행되다가 `Task` 를 생성하는 코드를 만나면, Task가 생성됩니다.  
그리고 Swift는 생성된 `Task` 가 현재 코드를 실행하는 `actor`(우선, 스레드라고 생각하면 쉽다.)와 동일한 `actor` 에서 실행되도록 스케줄링합니다.
- 여기서는 새로운 Main Thread에 스케줄링합니다.

- Task 취소 예시

```swift
@MainActor
class MyDelegate: UICollectionViewDelegate {
  var thumbnailTasks: [IndexPath: Task<Void, Never>] = [:]
  
  func collectionView(_ view: UICollectionView, willDisplay cell: UICollectionViewCell, forItemAt item: IndexPath) {
    let ids = getThumbnailIDs(for: item)
    thumbnailTasks[item] = Task { // 👈🏻 create and store unstructured tasks
      defer { thumbnailTasks[item] = nil } // 👈🏻 we remove the task when it's finished, so we don't cancel it when it's finished already
      let thumbnails = await fetchThumbnails(for: ids)
      display(thumbnails, in: cell)
    }
  }
  
  func collectionView(_ view: UICollectionView, didEndDisplay cell: UICollectionViewCell, forItemAt item: IndexPath) {
    thumbnailTasks[item]?.cancel() // 👈🏻 we cancel said task when that cell is no longer displayed
  }
}
```

- 위 코드는 collection view cell에서 보여줄 데이터를 다운로드 받는 작업을 시작했더라도, 
다운로드가 완료되기 전에 cell이 뷰에서 사라진 경우에는 작업을 취소하도록 구현한 것입니다.

1. `thumbnailsTasks` 딕셔너리에 Unstructured Task를 저장해둡니다.
2. Task에서 `defer` 는 작업을 완료한 후에 실행되는 코드입니다. 
여기서 `defer` 는 작업이 끝난 Task는 `thumbnailsTasks`딕셔너리에서 삭제되도록 만듭니다.
3. 만일, cell이 화면에서 사라졌을 때 `thumbnailsTasks` 딕셔너리에 Task가 남아있다면, 아직 Task가 실행중이라는 의미이기 때문에 `cancel()`을 호출해서 작업을 취소합니다.

## Detached Task

---

- unstrcutred task와 달리, 자신을 생성한 실행 context(부모 Task)로 부터 아무것도 상속받지 않습니다.
    
    (Unstructured tasks inherit traits from that task’s originating context, detached tasks don't.)
    
- unstrcutred task와 마찬가지로, 자신이 생성된 scope보다 더 오랫동안 실행되고, 
취소 작업와 완료 대기 작업은 수동으로 해야합니다.
    
    (Unscoped lifetime, manually cancelled and awaited)
    
- 실행 우선순위는 default 값을 갖지만, 
생성할 때 파라미터를 통해서 어디서, 어떻게 실행될지를 지정할 수 있습니다.

- 예시 코드

![Untitled](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/Untitled.jpeg)

- detched Task 내부에서 group task를 생성해서 사용할 수 있습니다.

![스크린샷 2022-07-28 오후 8.51.04.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_8.51.04.jpg)

## 정리

---

![스크린샷 2022-07-28 오후 8.54.35.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_8.54.35.jpg)

## Structured Concurrency 란❓

---

- [위키피디아](https://en.wikipedia.org/wiki/Structured_concurrency)에 따르면 concurrent 코드를 캡슐화해서, concurrent 코드를 명확하고(clarity), 좋은 품질(quality)의 코드로 빠르게 개발하는 프로그래밍 패러다임
- how❓
    
    concurrency programming에서  “scope 내에서 생성된 실행 context는, scope가 종료되기 전에 전부 실행완료 되어야 한다.” 원칙을 지킨다.
    
- 장점
    - 위의 원칙 덕분에 프로그래머는 concurrency programming을 할 때 다음과 같은 장점을 누릴 수 있다.
    1. error handling을 프로그래밍 언어에서 제공하는 기능 (throw - catch)를 이용해서 할 수 있다.
    2. 실행 context 간에 부모-자식 관계가 존재하는데, 자식 실행 context에서 발생한 error는 부모, 형제 실행 context로 전파된다.
    3. 병렬성/동시성 프로그래밍을 하면, 프로그램의 흐름을 파악하기 어렵지만, 
    structured concurrency에서는 실행 흐름을 명확하게 파악할 수 있다.
    

## Q&A

---

### 아래 사진에서 `cannot use error handling`, `cannot use loop` 가 무슨 의미?

![스크린샷 2022-07-28 오후 10.05.31.jpg](%5BWWDC%202022%5D%20Explore%20structured%20concurrency%20in%20Swif%20272879ee2f8a4529b1d34ddd132b2bb0/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-07-28_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_10.05.31.jpg)

- `cannot use error handling`:
    
    `escaping closure`에서는 error를 `throw` 하지 못한다.
    
- `cannot use loop` :
    
    위 코드에서 `prepareThumbnail()`의 결과값을 가지고 `fetchThumbnails()`를 반복해서 호출하고 싶어도, `completion handler`로 인해 `for loop`을 사용하지 못하고, 재귀호출을 사용해야 한다.

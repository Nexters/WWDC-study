# [2021] Protect mutable state with Swift actors

- concurrent 프로그램을 만들 때 어려운 점: data race가 발생하지 않도록 하는 것
- data race는 간헐적으로 발생하고, 디버깅하기 어렵다.
- 발생 원인: shared mutable state
    1. shared: 여러 Task에서 공유되지 않으면 발생 X
    2. mutable: mutable하지 않다면 발생 X
    

### 해결 방법1. value type을 사용

- all mutation is local
- let으로 선언하면 unmutable하다.

- value type을 var변수로 선언하고, mutating function을 사용할 경우에도 data race발생할 수 있지 않는가?

![스크린샷 2022-08-25 오후 3.12.13.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_3.12.13.jpg)

- value type을 var 변수로 선언했을 때, 컴파일러가 unsafe 한 코드라고 알려준다.

하지만 shared mutable state가 필요하다면

1. 동기화 로직 필요
- 동기화를 지원하는 primitve들
1. atmocis
2. locks
3. Serial dispatch queue

→ 단점: 사용할 때 주의사항을 지키면서 해야함, 그렇지 않으면 data race 발생

### actor

- 타입 like class, struct 등
- 타입이 갖는 기능들 제공
    - property, method, initialzier, subscript 등
    - protocol, extension
- [The difference is that the actor will ensure the value](https://developer.apple.com/videos/play/wwdc2021-10133/?time=371) [isn't accessed concurrently.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=374)
- instance data를 접근할 때마다, 동기화 매커니즘을 실행 
instance data를 프로그램의 다른 부분으로부터 독립시킨다. (isolate)
- 참조 타입
    
    이유: actor의 목적이 shared mutable state를 안전하게 제공하는 것이기 때문에
    
    참조타입은 mutable state를 공유하기에 편하다.
    
- actor가 갖는 mutable한 프로퍼티는 actor를 통해서만 접근이 가능함
- actor에 접근할 때마다 동기화 매커니즘이 실행되어, 다른 Task에서 actor의 state를 참고하고 있는지 체크합니다.
만일 data race를 일으킬 가능성이 있다면, 컴파일에러가 발생
    - ->  프로그래머가 shared mutable state를 접근할 때 동기화 매커니즘을 실행하는 것을 누락해서 발생하는 이슈 예방

[Whenever you interact with an actor from the outside,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=442) [you do so asynchronously.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=444)

[If the actor is busy, then your code will suspend](https://developer.apple.com/videos/play/wwdc2021-10133/?time=447) [so that the CPU you're running on can do other useful work.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=449)

[When the actor becomes free again,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=454) [it will wake up your code -- resuming execution --](https://developer.apple.com/videos/play/wwdc2021-10133/?time=455) [so the call can run on the actor.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=458)

- actor의 메서드 (extension으로 추가한 메서드)
    1. actor의 instance data에 직접 접근 가능
    2. 다른 actor 메서드를 호출하기 위해 await 키워드를 붙일 필요 X
    - 이유: actor 내부에 존재하기 때문이다.
    
    ## Actor에서 async code 혹은 다른 actor의 메서드를 실행한다면?
    
    💡 await 이후에 유지되지 않을 수 있는 state에 대한 가정을 하지 않아야 한다.
    
    [The fix here is to check our assumptions after the await.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=709)
    
    1. [If there's already an entry in the cache when we resume,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=713) [we keep that original version and throw away the new one.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=716)
    2. 더 나은 해결책은 중복 다운로드를 완전히 피하는 것이다.
    
    [Actor reentrancy prevents deadlocks](https://developer.apple.com/videos/play/wwdc2021-10133/?time=728) [and guarantees forward progress,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=730) [but it requires you to check your assumptions](https://developer.apple.com/videos/play/wwdc2021-10133/?time=732) [across each await.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=734)
    
    ### apple에서 제안하는 해결책
    
    1. actor의 data를 변경하는 것은 synchronous code에서 수행하기
        
        [State changes can involve temporarily putting our actor](https://developer.apple.com/videos/play/wwdc2021-10133/?time=749) [into an inconsistent state.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=752)
        
        [Make sure to restore consistency before an await.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=754)
        
    2. async-await 메서드로인해 프로그램이 실행이 중단된 동안, actor의 상태가 변했다고 가정하라
    
    → async-await function에서 resumse한 후에, actor의 상태를 체크하고 조치취하기.
    

## actor isolation

- actor isoloation = actor의 상태 변경을 한번에 한 Task만 할 수 있도록 제한하는 것
- how to isolate actor:Asynchronous = 비동기 활용하여, actor의 상태를 변경시키는 메서드를 한 Task에서 사용중이면, 사용이 끝날 때까지 다른 Task에서 대기하도록 한다.

## Actor isolocation이 protoocl, closure, class 등과 어떻게 적용되는지 볼 것!

- 프로토콜 사용가능
- 예시1

```swift
actor LibraryAccount {
	let idNumber: Int
	var booksOnLoan: [Book] = []
}

extension LibraryAccount: Equatable {
    static func == (lhs: LibraryAccount, rhs: LibraryAccount) -> Bool {
        return lhs.idNumber == rhs.idNumber
    }
}
```

- 프로토콜 메서드 중 static method : actor에 바깥에 있는 것으로 취급 (이유: static)
- 예시의 static method에서 actor의 state에 접근 가능한 이유: immutable state이기 때문이다.
    
    → actor의 immutable state에는 직접 접근 가능 
    
- Hashable 프로토콜
- hash(into)를 구현해야 한다. → 컴파일러 에러 발생
- 컴파일 에러 이유: 
actor 외부에서  이 메서드를 호출한다.  하지만 hash(into)는 async 메서드가 아니다. 
actor isolcation은 async-await를 이용하는데, hash(into)를 호출할 때는 actor isolocation을 할 수 없다.
- 예시2

```swift
actor LibraryAccount {
	let idNumber: Int
	var booksOnLoan: [Book] = []
}
```

![스크린샷 2022-08-25 오후 6.58.04.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_6.58.04.jpg)

- Hashable 프로토콜의 hash(into:) 구현시 컴파일 에러 발생
- hash(into:)는 actor의 메서드이지만, async method가 아니기 때문에, 
이 메서드를 호출했을 때 actor-isolocation을 할 수 없다.

- ***해결방법: hash(into:)를 non-isolcated 로 만든다.***
    
    ![스크린샷 2022-08-25 오후 7.01.59.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.01.59.jpg)
    
    - `nonisolated` ? actor 의 외부에 있는 메서드로 취급하겠다.
    - actor의 외부에 존재하기때문에, actor의 mutable state에 접근할 수 없다.
        
        ![스크린샷 2022-08-25 오후 7.09.57.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.09.57.jpg)
        
    

<aside>
⭐ **정리**

---

actor 관점에서 **메서드 종류**

1. **actor-isolated 메서드**
    - **actor 내부에 존재하는 메서드**
    - 사용 시도시, actor-isolation을 한다.
2. nonisolated 메서드
    - 코드 상에서 actor 내부 존재, actor 밖에 있는 것으로 취급
    - 사용 시도시, actor-isolation을 하지 않는다.

- actor에서 프로토콜을 구현할 때, non-async function은 `nonisolated` 메서드로 선언해야 한다.
</aside>

### closure

- actor 관점에서 closure 종류 2가지 존재
1. **actor-iolated closure**
    - actor-isolated function 내부에서 생성된 closure
2. **nonisolated closure**

- 예시
    
    ```swift
    extension LibraryAccount {
        func readSome(_ book: Book) -> Int { return .zero }
        
        func read() -> Int {
            return boosOnLoan.reduce(0) { partialResult, book in
                readSome(book)
            }
        }
    
    		func readLater() {
    		        Task.detached {
    		            await self.read()
    		        }
    		 }
    }
    ```
    
1. read()에서 reduce의 클로저 = actor-isolated 클로저
    
    read()가 actor-isolated 메서드이기 때문에, read()내부에 생성된 클로저도 actor-isolated 클로저
    

1. detached task는 클로저를 actor의 다른 코드와 병렬적으로 실행된다.
- detached task 의 클로저는 data race를 발생시킬 수 있다.
- ⇒ detached task의 closure는 nonisolated closure이다.
- datached task의 closure에서는 actor의 메서드를 await 키워드와 함게 호출해야 한다.

## Sendable Type

![스크린샷 2022-08-25 오후 6.48.19.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_6.48.19.jpg)

- 여러 actor가 서로 공유해도 안전한 타입을 Sendable한 타입이라고 한다.
- Sendable 한 타입 종류
    1. Value 타입
        - 예시
            1. book 이 struct인 경우 
            
            ![스크린샷 2022-08-25 오후 7.24.34.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.24.34.jpg)
            
            - random book을 얻더라도, 그것은 원본 book이 아닌 복사본이다.
            - 따라서 book의 타이틀을 변경하는 코드는 복사본을 수정하는 코드이기 때문에 원본 book에도, actor에도 영향을 주지 않는다.
            
            1. book 이 class인 경우
            - random book을 얻었을 대, 그것은 원본 book에 대한 참조이다.
            - 따라서 book의 타이틀을 변경한다면, 원본 book에 영향을 주어 actor의 state가 변경된다.
            - 현재 visit()메서드가 actor의 밖에 존재하기 때문에, 이는 data race로 이어진다.
            
    2. Actor 
    3. Immutable class: Immutable한 프로퍼티로만 구성
    4. 내부적으로 동기화 로직을 갖는 class
    5. @Sendable 타입의 function

- actor는 Sendable 타입을 이용하여 외부와 의사소통해야 한다.
    
    actor 바깥으로 non Sendable 타입을 전달하면, Swift가 컴파일 에러를 일으킨다.
    

### Sendable 타입 만드는 방법: Sendable 프로토콜 구현

![스크린샷 2022-08-25 오후 7.31.52.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.31.52.jpg)

- 모든 프로퍼티가 Sendable 프로퍼티일 때 Sendable 프로토콜을 구현할 수 있다.
그렇지 않으면 컴파일 에러 발생.

- 제네릭 타입은 모든 구체 타입이 Sendable일 때 Sendable 프로토콜을 구현할 수 있다.
    
    ![스크린샷 2022-08-25 오후 7.34.12.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.34.12.jpg)
    
- Array는 모든 구성요소가 Sendable 타입일 때, Sendable 타입이다.

### @Sendable functions

- actor에서 서로 주고받을 수 있는 function

```swift
func test(closure: @Sendable () -> Bool) {}
```

- Sendable 클로저를 만들기 위한 조건: data race를 일으키면 안된다.
    1. mutable local 변수를 캡쳐할 수 없다.
    2. 클로저가 캡쳐하는 모든 데이터의 타입이 Sendable해야 한다.
    3. synchronous sendable 클로저는 actor-isolated 될 수 없다. 
    (syncrhonous sendable & actor-isolated :  actor의 코드가 actor 밖에서 실행될 수 있기 때문이다.)
- 이 조건에서 벗어나면 컴파일 에러 발생
- 언제 사용? data race를 발생시키지 않는 코드를 만들기 위해
- non-sendable 타입에서는 호출할 수 없다.
- 장점
    
    [Sendable types and closures help maintain actor isolation](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1388) [by checking that mutable state isn't shared across actors,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1392) [and cannot be modified concurrently.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1396)
    

## MainActor

![스크린샷 2022-08-25 오후 7.50.14.jpg](%5B2021%5D%20Protect%20mutable%20state%20with%20Swift%20actors%2000e80c74a42e427da12f91f949b46166/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2022-08-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE_7.50.14.jpg)

- **Main Thread를 MainActor라고 표현한다.**
    - main thread 밖에서 main thread에게 일을 요청할 때, 만일 main thread가 현재하고 있는일이 있다면 끝날 때까지 기다려야 한다.
    - ⇒ main thread와 상호작용하는 것은 actor와 상호작용하는 것과 같다.
    
- 다른 actor와 차이점
1. main dispatch queue를 활용하여 동기화 수행 → DispatchQueue.main을 대신사용할 수 있다.
2. MainActor(MainThread)의 데이터와 코드는 actor 외부에 퍼져있다.

### @MainActor

- 함수, 타입 선언에 붙일 수 있다.
- @MainActor 표시를 하면, main actor(Main Thread)에서만 실행된다.
Main Actor 외부에서 실행하려면 await 키워드를 붙여줘야 한다.
(Main Thread에서 비동기적으로 실행될 것이기 때문이다.)
- @Main Actor 사용시 장점
    - 언제 DispatchQueue.main을 사용해야 하는지 고민할 필요 없다.
    Swift에서 알아서 Main Thread에서 실행시키기 때문이다.
- @MainActor 타입에서도 메서드를 nonisolated 로 선언할 수 있다.
    
    ```swift
    @MainActor class MyViewController: UIViewController {
      func onPress(...) { ... }// implicitly @MainActor
    
      nonisolated func fetchLatestAndDisplay() async { ... }
    }
    
    ```
    

## 요약

[using actor isolation and by requiring asynchronous access](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1623) [from outside the actor to serialize execution.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1627)

[Use actors to build safe, concurrent abstractions](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1631) [in your Swift code.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1634)

[In implementing your actors, and in any asynchronous code,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1636) [always design for reentrancy; an await in your code](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1640) [means the world can move on and invalidate your assumptions.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1644)

[Value types and actors work together](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1649) [to eliminate data races.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1651)

[Be aware of classes that don't handle](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1653) [their own synchronization, and other non-Sendable types](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1656) [that reintroduce shared mutable state.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1659)

[Finally, use the main actor on your code](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1662) [that interacts with the UI to ensure that the code](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1665) [that must be on the main thread always runs on the main thread.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1668)

[To learn more about how to use actors](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1673) [within your own application, check out our session](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1675) [on updating an app for Swift concurrency.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1678)

[And to learn more about the implementation](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1681) [of Swift's concurrency model, including actors,](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1683) [check out our "Behind the scenes" session.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1686)

[Actors are a core part of the Swift concurrency model.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1691)

[They work together with async/await](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1694) [and structured concurrency to make it easier to build](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1697) [correct and efficient concurrent programs.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1700)

[We can't wait to see what you build with them.](https://developer.apple.com/videos/play/wwdc2021-10133/?time=1703)
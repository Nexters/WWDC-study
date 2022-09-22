### 다룰 주제들
- continuation vs Actor
  - stack에서 non-async 왜 안 pop?
  - continuation이 중지되고 다시 돌아오는 방법
- Swift concurrency와 atomicity
- thread와 semaphore, lock etc


## Async frame

#### References
- http://www.tcpschool.com/c/c_memory_stackframe
- https://eliez3r.github.io/post/2019/10/16/study-system.Stack-Frame.html
- https://en.wikipedia.org/wiki/Call_stack

### Call stack & stack frame
>메모리의 스택(stack) 영역은 함수의 호출과 관계되는 지역 변수와 매개변수가 저장되는 영역입니다.
스택 영역은 함수의 호출과 함께 할당되며, 함수의 호출이 완료되면 소멸합니다.

![](https://velog.velcdn.com/images/enebin777/post/54bb15fa-b869-4efd-a9e5-3cb7d943cee5/image.png)

- 스택 프레임의 반환주소에는 서브루틴이 생성된 부모 루틴의 주소가 들어있다.

### Heap frame for async 
#### Refernces
- https://velog.io/@wansook0316/Swift-ConcurrencyBehind-the-scenes-Part.-01

![](https://velog.velcdn.com/images/enebin777/post/3f09ad0e-97c8-4185-9025-f4e1de3182b6/image.png) from - https://velog.io/@wansook0316/Swift-ConcurrencyBehind-the-scenes-Part.-01

``` Swift
func updateDatabase(with articles: [Article], for feed: Feed) async throws {
	...
    try await feed.add(articles)
}

func add(_ newArticles: [Article]) async throws {
    let ids = try await database.save(newArticles, for: self) 
    for (id, article) in zip(ids, newArticles) { 
        articles[id] = article
    }
}

func save(_ newArticles: [Article], for feed: Feed) async throws -> [ID] { /* ... */ }
```

- Async 함수도 함수므로 콜 스택에 지역변수, 매개변수, 반환 주소 등의 정보가 쌓인다. 
- 한가지 다른 점이 있다면 얘네는 정보가 힙에도 쌓인다는 것이다. 위 자료를 보면 알 수 있는데,
- `id`, `article`은 await(suspension point)으로 기다릴 필요가 없는 지역변수이므로 stack에 저장한다.
- 반면 `newArticles`는 await으로 기다릴 필요가 있는 변수, suspension point를 넘나들어 사용될 수 있는 변수이므로 heap의 `async frame`에 저장한다. 
- `add`에서 suspension point를 만나 실행이 중지(suspend)되면 곧이어 `save`가 실행된다. 그러면 stack의 `add` 프레임은 pop 되어 교체된다.
- heap의 `add`프레임은? pop 되지 않는다.
- stack의 frame이 pop이 되는 것은 heap의 해당 정보가 이미 저장되어 있기 때문이다. 저장되지 않은 정보는 이미 사용한 후일 것임.

![](https://velog.velcdn.com/images/enebin777/post/d027becd-43cc-4ca2-975b-c77260061ce6/image.png)
- `save`가 중지된 후의 상황도 비슷하다. 대신 힙의 여러 continuation, async frame 중 한 개가 쓰레드로 복귀한 후 자신의 suspension point 이후의 일을 계속 할 것이다.

![](https://velog.velcdn.com/images/enebin777/post/b03eb978-70bc-4434-9203-0cdc75037b38/image.png)
- 마침내 `save`로 돌아온 뒤하던 일을 마저 하기 시작한다.
- `save`가 반환한 뒤 `add`가 실행되고 그 안의 서브루틴`zip`이 실행된다. 
- `zip`은 새로운 새로운 스택프레임을 만드는 것을 볼 수 있다. 이는 (내 추측인데) `zip`이 heap에 프레임을 만들 수 없는 non-async 함수이기 때문이 아닐까?


## Actor
#### References
- https://developer.apple.com/documentation/swift/actor (도움 안 됨)
- https://www.hackingwithswift.com/quick-start/concurrency/what-is-an-actor-and-why-does-swift-have-them

### Continuation
- https://en.wikipedia.org/wiki/Continuation

Continuation(연속성)은 CS 개념 중 하나로 프로세스 실행 상태를 가진 자료구조고 런타임에 해제되지 않으며 프로그래밍 언어로 접근할 수 있다. 
>In computer science, a continuation is an abstract representation of the control state of a computer program. A continuation implements (reifies) the program control state, i.e. the continuation is a data structure that represents the computational process at a given point in the process's execution; the created data structure can be accessed by the programming language, instead of being hidden in the runtime environment. 

>Continuations are useful for encoding other control mechanisms in programming languages such as exceptions, generators, coroutines, and so on.

### Actor의 등장 배경

#### 데이터 레이스가 나는 흔한 경우와 흔한 해결법(queue.sync)에 대한 문제
- https://www.swiftbysundell.com/articles/swift-actors/
``` Swift
class UserStorage {
    private var users = [User.ID: User]()
    private let queue = DispatchQueue(label: "UserStorage.sync")

    func store(_ user: User) {
        queue.sync {
            self.users[user.id] = user
        }
    }

    func user(withID id: User.ID) -> User? {
        queue.sync {
            self.users[id]
        }
    }
}
```
- 여기서 sync를 안 시키면 여러군데서 접근할 때 data race가 발생하게 된다. 
- sync도 나름의 문제가 있는데 블록이 실행되는 중에는 쓰레드가 막혀버리기 때문에 막힌만큼의 손해가 발생한다.
- 이걸 `Data contention`이라고 한단다.

``` Swift
class UserStorage {
    private var users = [User.ID: User]()
    private let queue = DispatchQueue(label: "UserStorage.sync")

    func store(_ user: User) {
        queue.async {
            self.users[user.id] = user
        }
    }

    func loadUser(withID id: User.ID,
                  handler: @escaping (User?) -> Void) {
        queue.async {
          handler(self.users[id])
		}
    }
}
```
- async를 하면 사실 해결된다.
- 이것도 문제가 있다. 자료에선 `closure`쓰기 귀찮아서라고만 하고 딱히 언급을 안하는데 그건 여기서만든 큐가 serial 큐라서 인 것 같고
- WWDC 세션에 따르면 concurrent 큐의 경우 작업 중인 쓰레드를 피해서 새로운 쓰레드를 계속 만들기 때문에 `thread explosion`이 발생하는 문제가 있다.

#### Actor로 바꿔서 해결해보자
``` Swift
actor UserStorage {
    private var users = [User.ID: User]()

    func store(_ user: User) {
        users[user.id] = user
    }

    func user(withID id: User.ID) -> User? {
        users[id]
    }
}
```

액터는 마치 클래스처럼 동작한다. 다만,

- 액터는 **무조건** 프로퍼티와 메소드에 대한 접근을 serialize, 즉 한 번에 하나의 접근만 허용할 수 있게 만든다. 자동으로.
- (내 추측) 그리고 코어 하나 당 하나의 스레드만 실행되고 고로 블록이 하나 끝나야 다음 것이 실행되는 구조이므로 데이터레이스가 날 수가 없다.
- `Actor`는 서브클래싱을 할 수 없다. 왜냐면 애초에 클래스가 아니기 때문.
- 참고로 class를 액터가 참조하려면 `final`로 선언해서 바뀔일이 없다고 말해줘야 함. 

사용은 이런 식으로 `await`을 붙여주면 된다.
```
class UserLoader {
    private let storage: UserStorage

	...

    func loadUser(withID id: User.ID) async throws -> User {
        if let storedUser = await storage.user(withID: id) {
            return storedUser
        }

        let url = URL.forLoadingUser(withID: id)
        let (data, _) = try await urlSession.data(from: url)
        let user = try decoder.decode(User.self, from: data)

        await storage.store(user)

        return user
    }
}
```

- `UserStorage` 여러번 사용하면 그건 다시 data race를 만들어버림.

- Actor가 `async-await`을 위해 만들어졌다기 보다는 `Actor`는 그냥 Swift의 새로운 태스크 연속성 관리 단위(프로토콜)고 `async`선언 된 인스턴스는 자동으로 `Actor`를 따른다고 보는 것이 맞는 듯


---

## Atomicity
### 원자성이란
- https://ko.meaniit.com/words/4170

수행 도중 중단될 수 없는 하나의 동작 단위.

예를 들어 하나의 프로세서 명령어(instruction)가 수행 중이라면, 어떤 인터럽트가 발생하더라도 그 명령어의 수행은 중단되지 않으며, 프로세서는 그 명령어의 수행이 종료된 이후에 인터럽트를 처리할 것이다. 그러므로, 개별 프로세서 명령어들은 각각 원자적 행위라 할 수 있다. 이 개념은 프로세스 동기화와 상호배제 측면에서 매우 중요하다.

데이터베이스 분야에서의 트랜잭션도 원자적 행위 중의 하나인데, 이는 본래 원자성이 없는 명령어들을 묶어 원자성을 가진 실행 단위로 만든 것이라 할 수 있다. 트랜잭션을 수행하는 도중에 중단 요청이 발생하면 그 종류에 따라 트랙잭션을 모두 수행 후에 그 인터럽트를 처리하거나, 또는 트랜잭션 수행이 시작되기 이전 지점으로 복구한다.

ex. 함수가 atomic하다는 것은 전 영역에 통틀어 1개만 실행되고 있다는 뜻이다.

### Atomicity가 깨지므로 lock을 걸지 마세요?
WWDC에서는 `await` 주변에서 명백히 Atomicity가 깨지므로 lock을 걸어서는 안된다고 한다. 원자성이 한 명령어가 끝날때까지 수행되는 것이 보장되는 성질이기에 `await`에서 원자성이 깨진다는 설명은 이해가 된다. 하지만 lock을 걸어서는 안된다는 것이 대체 무슨 뜻일까.
#### lock <-> atomicity
![](https://velog.velcdn.com/images/enebin777/post/770c22af-2882-41a8-b7c4-77f0460b277d/image.png)

- https://stackoverflow.com/questions/55202201/relationship-between-locks-and-atomic-operations-in-thread-programming
- https://stackoverflow.com/questions/7612602/why-cant-i-use-the-await-operator-within-the-body-of-a-lock-statement

Lock은 atomicic한 동작을 보장하는 여러 방법 중 하나다. 즉, lock은 thread-safe한 동작을 보장하게끔 추상화된 인터페이스의 구현체이고 그 과정에서 atomic한 연산을 이용했다고 말할 수 있다. 별개의 컨셉이라기보단 방법과 구현의 차이인 것.

그러면 왜 await에서 atomicity가 보장되지 않으므로 lock을 쓰지 말라고 하는 것인지? 한 줄로 요약하면 다른 쓰레드가 lock을 서로 관리하는 과정에서 lock ordering inversion이 발생할 수 있으며 이는 데드락을 발생시킨다. 그러면 또 lock ordering inversion은 무엇일까..

### lock ordering inversion
- https://softwareengineering.stackexchange.com/questions/425016/what-is-a-lock-ordering-inversion
- 알아둘 것: lock은 쓰레드 단위로 걸린다.
> If Thread 1 seizes lock A, then Thread 2 will wait at Lock A until thread 1 releases it. Whilst Thread 2 waits at Lock A, Thread 1 is free to proceed and also seize Lock B. At some point Thread 1 will release Lock A (either before or after releasing B, it doesn't matter), and Thread 2 will then proceed. If Thread 1 wanted to retake Lock A for whatever reason, then it would only do so after first releasing Lock B. This is the correct ordering, in the sense that it involves no risk of deadlock. A is always taken before B.
>
> In the case of inverted ordering, Thread 1 proposes to seize A before B, whilst Thread 2 proposes to seize B before A. With suitable timing, a situation can come about in which each have seized the first of the locks they proposed to take, yet each also needs access to the other lock in order to proceed. This is a deadlock situation.

- 기차 선로 비유 : 선로가 2개인 기차역이 있다고 하자. 사고를 미연에 방지하기 위해 각 선로에는 통행을 막는 차단봉이 달려있다. 1번 선로에 2번 선로에 있는 기차가 역을 통과하기 전까지 올라가지 않는 차단봉이 내려왔다. 공교롭게도 2번 선로 역시 1번 선로에 있는 기차가 역을 통과하기 전까지 올라가지 않는 차단봉이 내려왔다. 양 선로의 기차들은 서로가 통과할 때까지 기다리지만 모두 차단봉(락)이 내려와 있으므로 영원히 움직일 수 없다. 
- 이 이유로 2개 이상의 쓰레드에서 서로 락을 관리하는 것은 위험하다. 1번 기차(쓰레드)가 락을 걸고 튀어버리면 그걸 풀 방법이 없기 때문이다. 
- Swift concurrency는 태스크가 같은 쓰레드에서 실행되는 걸 보장하지 않는다. 그러므로 lock을 금지시키는 것. 



#### Semaphore도 마찬가지인데 그 전에 Runtime contract를 먼저 알아야 함
### Runtime contract
Swift concurrency가 런타임에 보장해야 하는 계약같은 것
- https://stackoverflow.com/questions/70962534/swift-await-async-how-to-wait-synchronously-for-an-async-task-to-complete

#### Rule no.1: Forward progress
- 대충 멈추지 않고 나아간다 이런 뜻인 것 같다. 
- 무슨 뜻이냐 하면 Swift concurrency는 cooperative pool을 사용하므로 태스크가 쓰레드를 막지 않는 것이 원칙이다. 
- 그러므로 Actor context 안에서는 그 어떤 것도 쓰레드를 막으면 안된다. 길을 막지 말고 다음 작업을 위한 공간을 마련하라는 것. 

![](https://velog.velcdn.com/images/enebin777/post/cdba43b7-830a-476a-8a31-242939daa8f3/image.png)
- 이 경우 문제가 있는 코드의 예시인데 semaphore가 wait을 시키는 것을 볼 수 있다. 이는 쓰레드를 막는 행위다. 
- Cooperative pool에서 쓰레드가 막혔으니 다음 actor가 실행될 수 없고 그러므로 `semaphore.signal()`을 호출하는 것도 불가능하다. OS는 가능한 thread를 탐색한후 데드락을 발생시킨다. 
- 근데 또 lock은 쓸 수 있다 함. -> Semaphore는 런타임에 dependecy를 안 알려주고 execution 타임에 알려줘서 그렇다는데 이해 할 수 없음. 
















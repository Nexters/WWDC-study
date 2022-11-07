# Swift Concurrency Deep Dive

## Swift Concurrency는 GCD의 대체재

---

![클래식한 GCD](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled.png)

클래식한 GCD

![새로운 Swift concurrency](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%201.png)

새로운 Swift concurrency

여러 WWDC 세션에서 애플은 Swift Concurrency로 GCD를 대체하려는 의도를 비춰왔다. 이는 [Swift concurrency: Behind the scenes - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10254/) , [Meet async/await in Swift - WWDC21 - Videos](https://developer.apple.com/videos/play/wwdc2021/10132/?time=1396)에서 매우 잘 드러난다. GCD가 흔히 발생시킬 수 있는 에러를 예로 들어가며 설명을 하는데 대표적인 사례는 다음 2가지이다.

### GCD의 단점 1: Thread explosion에 취약함

Thread explosion은 주로 Concurrent queue에서 실행시간이 긴 메소드를 dispatch할 때 발생한다. 다음은 해당 문제가 발생하는 코드의 예시다. 

```swift
import Foundation

let queue = DispatchQueue(label: "test", attributes: .concurrent)

for i in 0..<100 {
		queue.async {
        print(i, Thread.current)
        sleep(5)
    }
}
```

위 코드는 매 루프마다 5초 이상이 소비되는 작업을 큐에 하나씩 더하는 것을 가정했다. 용량이 큰 파일을 로딩하거나 네트워크에서 파일을 다운로드 하는 작업들이 이에 해당할 수 있겠다. 

`queue`가 serial 하지 않고 큐에 블록을 dispatch 할 때 `sync`로 이전 블록의 반환을 기다리지도 않으므로 OS는 최대한 많은 작업을 동시에 처리하려 할 것이다. 그 결과 OS는 다음과 같은 결정을 내린다. 

```swift
0 <NSThread: 0x600003938000>{number = 5, name = (null)}
5 <NSThread: 0x60000392d380>{number = 6, name = (null)}
6 <NSThread: 0x60000392c540>{number = 7, name = (null)}
2 <NSThread: 0x60000392cbc0>{number = 8, name = (null)}

...

95 <NSThread: 0x600003938540>{number = 42, name = (null)}
98 <NSThread: 0x60000392c740>{number = 50, name = (null)}
97 <NSThread: 0x600003920740>{number = 46, name = (null)}
99 <NSThread: 0x60000392c800>{number = 25, name = (null)}
```

**쓰레드를 가능한 많이 만들어버린다.** 이런 블록이 100개를 넘어서 1000개를 향해 달려간다면 시스템에 무리가 오는 것이 당연하다.

### GCD의 단점 2: 잦은 문맥교환 시 발생하는 오버헤드

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%202.png)

운이 좋아서 시스템이 뻗기 직전에 쓰레드 생성이 멈출 수도 있다. 이 경우 시스템이 멈추지는 않을지라도 전체적인 성능의 저하를 유발할 수 있다. 쓰레드가 많을수록 CPU의 문맥교환이 빈번해지기 때문이다. 세션에서는 이를 두고 **Full thread context switch**라고 일컫는다. 이 **Full thread context switch**가 빈번할수록 CPU의 성능이 낭비되는 것은 익히 알려진 사실이다.

### 이 Swift concurrency는 무료로 해줍니다

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%203.png)

Swift Concurrency의 장점은 여러가지가 있겠지만 GCD의 대체자라는 측면에서는 위 두가지가 제일 눈에 띄는 부분이다. 그나저나 Swift Concurrency가 이를 어떻게 해결할 수 있다는 것인가? 이는 쓰레드를 블록하지 않는 설계적 특성때문에 가능해진다. 

결론부터 말해보면 **Swift Concurrency로 처리하는 작업은 특정 쓰레드에 귀속되지 않는다**. ********조금 더 풀어서 말해보면 해당 작업이 `await` 키워드로 인해 **정지**(suspend, 중지와 다르다)되어 기다리다 남은 일을 처리하려 작업 쓰레드로 돌아올 때 그 쓰레드가 원래 작업을 시작했던 쓰레드가 아닐 수도 있다는 것이다. 

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%204.png)

이를 통해 가능해지는 것이 하나 있다. 바로 **유휴상태의 쓰레드에 다른 작업을 배정해 실행**시키는 것이다. 쓰레드에서 작업이 정지(`sync`)되었을 때 실행중인 작업의 반환을 기다리는 기존의 방식과 달리 S.C는 실행을 기다리는 **다른** 작업을 찾아 유휴 쓰레드 중 한 군데에 배정시켜버린다. 이로써 최소한의 유휴 쓰레드를 유지하게 되고 결과적으로 1코어 1쓰레드라는 이상에 한발짝 더 다가설 수 있게 되는 것이다.

<aside>
💡 from [WWDC Session](https://developer.apple.com/videos/play/wwdc2021-10254/?time=723)
This means that we now only pay the cost of a function call instead. So the runtime behavior that we want for Swift concurrency is to create only as many threads as there are CPU cores, and for threads to be able to cheaply and efficiently switch between work items when they are blocked.

</aside>

그러나 이는 GCD의 두번째 단점, 잦은 문맥교환 문제를 되풀이하는 것처럼 보인다. 하지만 S.C는 이 문제 대해 자유롭다. **Full thread context switching을 하지 않기 때문이다**. WWDC 세션을 인용하자면 SC가 작업을 교환하는 데는  “함수를 호출하는 정도의 비용”밖에 들지 않고 결론적으로 문맥 교환에 들던 비용이 없어지게 된 것이다.

S.C의 쓰레드 정책에 대해서는 간단한 실험으로 알아볼 수 있다. 다음은 앞서 GCD의 예시와 같이 100번의 print를 하는 코드다.

```swift
import Foundation

Task {
    await withThrowingTaskGroup(of: Int.self) { group in
        for i in 0..<100 {
            group.addTask {
                try await Task.sleep(nanoseconds: 1_000_000_000)
                print(i, Thread.current)
                return i
            }
        }
    }
}
```

각각 1초 후에 프린트문을 실행하는 태스크의 그룹이다. `TaskGroup`은 동적으로 구성 가능한 태스크의 묶음으로 이를 이용하면 그룹의 child 태스크들을 병렬적으로 실행한 후 완료 시 콜백을 받을 수 있다.  `try await Task.sleep(nanoseconds: 1_000_000_000)`는 해당 정지점(suspension point)에서 10^9 나노초, 즉 1초간 정지(suspend)하라는 것을 의미한다. 위 구문의 실행 결과는 다음과 같다. 

```swift
0 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
1 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
3 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
2 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}

...

96 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
99 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
98 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
92 <NSThread: 0x6000011e88c0>{number = 7, name = (null)}
```

GCD의 예시과 같이 병렬적으로 함수를 실행했지만 모두 같은 쓰레드 위에서 작업이 이루어졌다는 차이점을 찾아볼 수 있다. 

여기서 GCD처럼 `sleep(1)` 을 사용했다가는 각 태스크가 마치 `sync` 함수를 쓴 것 같이 직렬적(serial)으로 실행되는 것을 관찰할 수 있다. 이는 `sleep`이 쓰레드를 **중지**(block)하는 메소드기 때문이다. 

Apple은 이와 같이 쓰레드를 블록하는 동작을 **Forward progress하지 않은 동작**이라고 정의하며 S.C의 컨텍스트에서는 가능한 사용을 자제하기 권한다. 자의적으로 쓰레드를 블록하다 데드락과 같은 문제를 발생시켜 시스템 전체에 예기치 못한 영향을 끼칠 수 있기 때문이다.

SC에 이런 특징이 있다는 것에 대해서는 알게 되었다. 하지만 이 모든 것이 대체 어떤 마법 덕분에 가능해진 걸까? 이는 SC가 **연속성**(Continuation)을 구현했기 때문이다. 

## 연속성(Continuation)

---

### 연속성의 개념

[Continuation - Wikipedia](https://en.wikipedia.org/wiki/Continuation)

위키백과의 정의를 빌려 설명해보면 **연속성은 명령어의 진행 상태를 저장하는 프로그래밍 언어의 기능**을 일컫는다. 다음은 연속성에 대한 비유 중 하나인 ‘샌드위치 연속성’이다.

<aside>
📔 from [Here](https://groups.google.com/g/perl.perl6.language/c/-KFNPaLL2yE/m/_RzO8Fenz7AJ?pli=1)
당신은 샌드위치를 떠올리며 부엌의 냉장고 앞에 서있습니다. 그 상태에서 당신은 주머니에 **연속성**을 집어 넣습니다. 그러고나서 햄과 빵을 냉장고에서 꺼낸 후 샌드위치를 만들어 카운터에 올려 놓습니다. 여기서 당신은 **연속성**을 불러오고 냉장고 앞에서 샌드위치를 떠올리고 있는 자신을 발견합니다. 운 좋게도 이번엔 카운터에 샌드위치가 놓여있고 재료는 이미 다 사라져버렸습니다. 이제 샌드위치를 먹습니다.

</aside>

샌드위치를 데이터의 일부라고 생각한다면 샌드위치를 만드는 것은 프로그램이 데이터를 가지고 하는 작업이라고 볼 수 있을 것이다. 연속성은 이 때 샌드위치를 만들기 전의 상황을 저장하고 있는 일종의 세이브포인트다. 연속성을 가진 상태라면 샌드위치를 만든 후 샌드위치를 만들기 직전의 지점을 불러와 해당 포인트에서 다시 시작할 수 있을 것이다. 단 냉장고는 연속성에 포함되지 않았으므로 재료의 상태에 대한 부분은 이전으로 돌아가지 않는다. 

다시 정리하면, **연속성은 프로그램의 실행 상태를 다시 가져올 수 있는 공간에 저장해 어디서든 다시 불러올 수 있게 만들어 주는 기능**이다. 이러한 연속성을 구현하는 방법은 다양한데 몇 가지 방법들이 [이 곳에](https://wiki.c2.com/?ContinuationImplementation) 소개되어 있다.

이를 함수에 직접적으로 전달하는 경우 역시 존재한다. 이는 [continuation-passing style](https://en.wikipedia.org/wiki/Continuation-passing_style)이라고 불리며 다른 함수에 현재의 연속성을 전달해 그 다음의 작업을 처리하게끔 하는데 *Scheme*, *Coroutine* 등의 프로그래밍 언어가 이러한 기능을 가지고 있다고 한다.

### Swift에서의 연속성

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%205.png)

Swift는 정지점(suspension point, `await`)에서 사용할 상태, 즉 연속성을 **heap에 저장**하는 방식을 사용하며 **async frame**을 저장한다고 말한다. 이는 non-async 코드에서 사용하는 방법인 stack frame으로서의 저장과 대비되는 개념이다. Stack에 저장되는 자료들과 다르게 실행중인 함수가 정지되거나 반환된다고 하더라도 정보가 비워지지 않기 때문이다.

Swift는 연속성을 수동으로 만들 수 있는 기능을 제공한다. `(Un)CheckedContinuation`이라는 타입을 이용하며 인스턴스 메소드 `resume`을 통해 다시 불러올 수 있다. 

이는 본래 클로저를 통한 컴플리션 핸들러의 형태로 처리되던 비동기 코드들을 S.C로 결합할 수 있는 API로서 제공함이 목적이었다. 다만 원래 S.C의 맥락에서 벗어난 코드로서 정지상태에서 벗어나는 순간을 잡아주어야 하며 이를 메소드 `resume`을 통해 알려주는 것이다. 

Uncheck과 Check의 차이는 중복된 `resume`등의 문제를 체크할지 안 할지에 대한 표시로 인터페이스 자체는 동일하다([출처](https://github.com/apple/swift-evolution/blob/main/proposals/0300-continuation.md)). Apple은 별다른 의도가 없다면 안전한 타입인 `CheckedContinuation`을 사용하길 권장한다.  다음은 사용 예시다.

```swift
import Foundation

func doSomethingWithDelay(completion: @escaping () -> Void) {
    sleep(1)
    completion()
}

Task {
    await withCheckedContinuation { continuation in
        doSomethingWithDelay {
            print("Completed")
            continuation.resume()
        }
    }
}
```

`withCheckedContinuation`를 사용하면 클로저 안의 코드들을 S.C의 맥락 안으로 가져올 수 있다.

### 연속성과 S.C의 상관관계

연속성은 프로그램의 실행 상태를 **heap**에 저장한다고 했다. 이는 언제든지 비워질 위험이 있는데다 순차적인 실행을 보장해야 하는 stack을 벗어나 공유된 자원으로서 어디서든 접근 가능하고 영속적일 수 있는 데이터를 제공할 수 있음을 뜻한다. 

이를 통해 연속성은 프로세스의 재진입성(reentrancy)을 보장하며 앞서도 언급한 바 있는 “함수를 호출하는 정도의 비용”만을 소비하는 것을 가능하게 한다. 이는 Full thread context switching이 없다는 말과 동치이다. 메모리 주소를 이용해 호출하므로 쓰레드 교환을 할 필요가 없기 때문이다.

### 쓰레드 블록 금지. 왜?

S.C에서는 `NSLock`과 `DispatchSemaphore`같이 동시성을 관리하기 위해 기존에 사용하던 쓰레드 블로킹 방법들을 사용할 수 없다. 예시를 통해 살펴보자. 

```swift
import Foundation

let lock = NSLock()

func doSomething(_ num: Int) async throws {
    Task {
				********try await Task.sleep(nanoseconds: 1_000_000_000)
        print("Job: \(num)", Thread.current)
        lock.unlock()
    }
}

// main
Task {
    for i in 0..<100 {
        lock.lock()
        try await doSomething(i)
    }
}
```

위 예시는 데드락이 발생하는 대표적인 코드다. 세션에서는 `DispatchSemaphore(value: 0)`을 사용했지만 `NSLock`을 사용해도 별 차이가 없다.

본래 위 코드에서 의도한 것은 다음과 같을 것이다. 

1. `main` 파트에서 `doSomething`을 100회 실행할 것이다. `doSomething`은 `async` 메소드이므로 `await`으로 정지점을 잡아주어야 한다. 또한 해당 메소드가 동기적(순차적)으로 실행되길 원하므로 전역적으로 공유된 `NSLock`을 이용해 접근 가능 여부를 체크할 것이다.
2. `doSomething`은 작업을 1초동안 정지한  후 `print`를 실행하는 간단한 함수다. 모든 함수 실행을 마치면 다음 작업이 접근할 수 있게 lock을  `unlock`한다.

하지만 미리 언급했듯 위 코드는 데드락이 발생해 아무것도 출력하지 않는다. 그 이유는 무엇일까? 이는 **Structured concurrency**와 `Task`에 대한 이해가 필요하다.

## 구조적 프로그래밍(Structured Programming)

---

### 구조적 프로그래밍 패러다임

[Structured programming - Wikipedia](https://en.wikipedia.org/wiki/Structured_programming)

이번에도 위키백과의 정의를 빌리면 구조적 프로그래밍은 “프로그래밍 패러다임의 일종인 절차적 프로그래밍의 하위 개념으로 볼 수 있다. GOTO문을 없애거나 GOTO문에 대한 의존성을 줄여주는 것으로 가장 유명”하다. 구조적 프로그래밍을 따른다는 것은 프로그램이 코드를 보이는 순서대로, 즉 절차 지향적으로 실행한다는 것을 뜻한다.

구조적 프로그래밍을 따르는 코드의 예시는 다음과 같다. 

```swift
var preparedDishes = 0

for i in 1...100 {
		cook()
    preparedDishes += 1
}

if preparedDishes == 100 {
		print("It's done")
}

```

위 코드의 경우 선행 코드가 모두 완료되어야만 그 뒤의 코드가 실행되는 것을 볼 수 있다. 만약 이를 Swift의 completion handler를 이용해 표현한다면 다음과 같을 것이다.

```swift
var preparedDishes = 0

while true {
    if preparedDishes == 100 {
        print("It's done")
        break
    }
}

for i in 1...100 {
    cook(completion: {
        self.preparedDishes += 1
    })
}
```

하지만 이 경우 첫 번째 예시와 다르게 후행 코드의 결과(`completion`)가 선행 코드에 영향을 미치는 것을 볼 수 있다. 이를 **비구조적(unstructured)**으로 설계되었다고 표현한다. 

S.C가 소개되기 이전 Swift의 비동기 프로그래밍은 대부분이 비구조적인 방식을 사용해야 했다. 대표적인 비동기 처리 방식인 `DispatchQueue`를 비롯해 최근 소개된 `Combine`의 역시 후행 코드가 완료된 후 콜백을 통해 선행된 코드의 작동에 영향을 미치는 방식이기 때문이다. 

후행코드가 선행코드에 영향을 미칠 수 있는 비구조적 프로그래밍은 시각적인 흐름을 역행하므로서 가독성을 해치고 에러를 유발하기 쉬운 구조를 만든다. 이로 인해 Swift 내 비동기 프로그래밍에서 구조적인 코드의 필요성이 대두되었고 그 결과 Swift Concurrency가 만들어지게 된 것이다. 

그만큼 구조적 프로그래밍은 S.C의 토대라고 할 수 있으며 제안문서 역시 Structured Concurrency라는 제목으로 쓰여졌을만큼 핵심적인 개념이라고 할 수 있을 것이다([제안 문서](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md), [세션](https://developer.apple.com/videos/play/wwdc2021/10134/)).

그러면 이제 `URLSession`을 통해 구조적/비구조적 프로그래밍의 차이를 살펴보도록 하자. 첫 번째는 S.C가 도입되기 전 사용했던 `completionHandler`를 이용한 방식이다.

```swift
var completionHandler: ((Result<UIImage, Error>) -> Void)?
let urlRequest = URLRequest(url: URL(string: "www.example.com")!)
URLSession.shared.dataTask(with: urlRequest) { data, response, error in
    
    // ...
    
    guard let completion = completionHandler else { return }
    
    guard let data = data else {
        completion(.failure(URLError.cannotDecodeContentData))
        return
    }
    
    let image = UIImage(data: data)
    completion(.success(image))
}
```

- `dataTask`가 완료된 시점에 completion handler를 이용해 콜백을 받는 방식이다.
- 이 경우 `compeltionHandler`를 직접 호출해 가공된 이미지를 전달해야 한다. 또한 에러를 가시적으로 전달하기 위해 `Result` 타입을 사용하였다.
- 만약 `compeltionHandler`의 호출을 실수로 빠트린 경우 외부에서 컴플리션을 참조한 곳에서 아무 일도 일어나지 않는 문제가 있다.

아래는 위 코드를 `async/await`을 이용해 바꾼 코드다.

```swift
let urlRequest = URLRequest(url: URL(string: "www.example.com")!)
Task {
    let data = try await URLSession.shared.data(for: urlRequest)
    let image = UIImage(data: data.0)
		return image
}
```

- 코드가 쓰여진 순서대로 실행된다.
- `try`를 사용하여 에러를 바로 던질 수 있다. `throw`도 사용이 가능하다.
- `compeltionHandler`를 사용하지 않아도 정보를 전달할 수 있다.

- await으로 데이터 로드가 멈춰 있으므로 코드는 저 지점에서 멈춘 뒤 작업이 완료되면 data에 해당 자료가 저장될 것이다
- 그 다음 정지점에서 다시 시작된 코드는 해당 데이터를 가지고 image를 만들어 리턴한다.

여기서 `async/await`외에 `Task`라는 키워드를 사용했다. 앞에서부터 말도 없이 `Task`를 자주 사용했는데 이것은 또 무엇일까? 

## Task

---

<aside>
💡 from **[제안문서](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)**
A task is the basic unit of concurrency in the system. Every asynchronous function is executing in a task. In other words, a *task* is to *asynchronous functions*, what a *thread* is to *synchronous functions*.

</aside>

**Task는 Swift Concurrency에서 비동기 작업의 단위**로 위의 제안 문서에서도 정의하듯 모든 비동기 함수들은 반드시 태스크를 통해 실행한다. 

(해당 문서를 읽다 보면 Swift 팀은 SC를 GCD의 대체재로 생각한다는 것은 물론 더 나아가 Thread 자체를 다루지 않아도 될 수 있게끔 의도하고 있다는 생각을 엿볼 수 있다. 일례로 Task 안에서 `Thread.current`를 실행하는 것이 현재(Swift 5.7)는 Warning에 그치는 반면 Swift 6부턴 컴파일 에러로 아예 금지된다.)

<aside>
💡  from [Document](https://developer.apple.com/documentation/swift/task)
When you create an instance of `Task`, you provide a closure that contains the work for that task to perform. Tasks can start running immediately after creation; you don’t explicitly start or schedule them. After creating a task, you use the instance to interact with it — for example, to wait for it to complete or to cancel it

</aside>

<aside>
💡 from [Language Guide](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html#ID643)

In addition to the structured approaches to concurrency described in the previous sections, Swift also supports unstructured concurrency. Unlike tasks that are part of a task group, an *unstructured task* doesn’t have a parent task. … To create an unstructured task that runs on the current actor, call the `[Task.init(...)](https://developer.apple.com/documentation/swift/task/3856790-init)` initializer.

</aside>

태스크는 제공되는 API를 통해 생성, 관리되며 **비구조적 동시성(Unstructured Concurrency)**를 가능케한다. 지금껏 Swift Concurrency는 구조적인 방식으로 설계되었다고 했다. 그런데 비구조적인 동시성을 지원한다는 것은 어폐가 있어 보인다. 과연 어떻게 된 일일까?

이는  `Task`를 통해 명시적으로 루틴을 실행할 경우 `await`으로 정지점을 설정할 수 없기 때문이다. `Task`로 비동기 태스크를 만들 경우 어느 태스크에도 귀속되지 않은 채로 코드가 실행되며 이는 해당 맥락에서 결과를 전달해야 하는 부모 태스크를 갖지 않는다는 것과 같은 뜻이다. 즉, 가이드에서 정의한 구조적과 비구조적의 차이는 자식 태스크로서 이를 실행하느냐 그렇지 않느냐를 의미한다.

단 이는 Task가 완료되기 전까지 해당 부분에서 작업이 정지되는 것을 기대할 수 없다는 의미로 Task를 변수로 설정해 내부에서 반환되는 결과값을 `await`을 이용해 받아보는 것은 가능하다. 상당히 헷갈릴 수 있는 부분이며 실제로 많이들 헷갈려 하는 듯 보인다([예시](https://stackoverflow.com/questions/71575803/difference-of-creating-regular-or-detached-task-from-task-less-context)).

정리해보면, `**Task`는 이 태스크의 완료를 기다리는 부모 태스크가 없거나, 모종의 이유로 현재 액터의 맥락에서 벗어난 작업을 실행해야 할 때 사용할 수 있다.** 

### Detached Task

위 두 용례 중 **모종의 이유로 현재 액터의 맥락에서 벗어난 작업을 실행**하는 케이스를 들여다 보자. 이를 위해서는`Task.detached`라는 메소드로 태스크를 생성해야 한다. 이렇게 생성된 태스크는 현재 자신이 속한 액터에 포함되지 않고 또한 태스크의 로컬 변수들 또한 상속하지 않는다. 

액터에 대해서는 바로 다음에 자세히 설명할 것이다. 현재는 서로 다른 쓰레드(맥락)에서 실행하는 방법이라고만 알아두어도 좋다.

앞서 예시로 든 `NSLock`의 데드락 문제는 detached 태스크를 사용하면 해결 할 수 있다. 다시 돌아와 코드를 살펴보자.

```swift
import Foundation

let lock = NSLock()

func doSomething(_ num: Int) async throws {
    print("Ready to do:", num)
    
    Task {
        try await Task.sleep(nanoseconds: 1_000_000_000)
        print("Job: \(num)", Thread.current)
        lock.unlock()
    }
    
    print("Returned: ", num)
}

// main
Task {
    for i in 0..<100 {
        print("Entered: ", i)
        lock.lock()
        try await doSomething(i)
    }
}
```

가시적인 확인을 위해 프린트 코드를 몇가지 추가했다. 이제 함수가 실행되는 과정을 살펴보자. 

1. main의 `Task` 안에서 for문이 실행된다.
2.  for 루프에서 `doSomething`을 실행하기 전 공유된 lock을 `lock`한다. 
    1. 따라서 어디선가 lock을 `unlock`하지 않는다면 `i`가 2 이후, 즉 두 번째 루프 이후로는  `doSomething`이 실행될 수 없을 것이다.
3. `doSomething`이 실행되면 내부에서 현재 쓰레드를 `print`하는 **비구조적 태스크**를 스케줄링한다. 
    1. 비구조적으로 태스크가 만들어졌기 때문에 함수는 Task의 완료를 기다리지 않고 반환될 것이다. 
    2. 함수가 Task의 완료를 기다리기 위해서는 `await`을 만들 수 있는 부모 태스크가 있어야 한다.
4. 위 코드를 실행하면 다음과 같은 로그가 찍힌다.

```clojure
Entered:  0 
Ready to do: 0
Returned:  0
Entered:  1
```

- Entered 0: for문의 첫번째 루프에 진입했다.
- Ready to do 0: 첫 번째 루프에서 호출한 `doSomething`에  진입했다.
- Returned 0: `doSomething` 의 실행이 끝나(기 직전이)다.
- Entetred 1: for문의 두번째 루프에 진입했다. lock을 확인했더니 잠겨있어 unlock이 될 때까지 무제한 대기한다.
    - 하지만 SC의 쓰레드 큐는 serial이므로 앞 작업이 끝나기 까진 다음 작업을 할 수 없다. 다음 작업이 하필이면 `unlock`이므로 이 작업은 끝까지 진행될 수 없다. → **데드락**

이를 `DispatchQueue`를 이용해 쓰레드 레벨에서 표현하면 다음과 같을 것이다.

```swift
import Foundation

let lock = NSLock()
let queue = DispatchQueue(label: "test")

func doSomething(_ num: Int) {
    print("Ready to do:", num)
    
    queue.async {
        print("Job: \(num)", Thread.current)
        lock.unlock()
    }
    
    print("Returned: ", num)
}

// main
queue.async {
    for i in 0..<100 {
        print("Entered: ", i)
        lock.lock()
        doSomething(i)
    }
}
```

- 위 상황을 해결하기 위해서는 `NSLock`을 `NSRecursiveLock`으로 변경해 Lock을 여러번 걸 수 있도록 설정해주는 방법이 있다. 하지만 이 경우 태스크가 이전 태스크를 기다리지 않고 병렬적으로 실행되고 결론적으로는 Lock을 사용하지 않는 것과 별 다른 점이 없으므로 옳은 방법은 아니다.
- `Task.detached`를 이용한 방법이 있다. lock을 다른 쓰레드에서 `unlock` 해주어 멈춰있던 작업을 진행시키는 것이 그 원리다. 이 경우 이전 task가 다음 task를 기다리므로 소기의 목적을 달성했다고 할 수 있다. 하지만 detached 태스크는 현 맥락에서 벗어나는 특성 상 예기치 못한 오류를 일으킬 수 있는 방법이다. 때문에 도큐먼트에서는 특별한 경우가 아니라면 `detached`사용을 자제하기를 권하고 있다. ([문서](https://developer.apple.com/documentation/swift/task/detached(priority:operation:)-3lvix))

사실 이 경우 다음과 같이 사용할 때 락을 걸지 않고도 진행하는 것이 가능하며 따라서 데드락을 유발하지 않는 코드를 만드는 것이 가능하다. 

```swift
import Foundation

func doSomething(_ num: Int) async throws {
    try await Task.sleep(nanoseconds: 1_000_000_000)
    print("Job: \(num)", Thread.current)
}

// main
Task {
    for i in 0..<100 {
        try await doSomething(i)
    }
}
```

- 이 경우 쓰레드를 중지(block)시키는 코드가 존재하지 않으며 따라서 **Forward progress를 만족한다**.

앞서 액터라는 요소에 대한 언급이 있었다. 액터는 SC에서 데이터 레이스를 방지하는 데 중요한 역할을 담당하는 타입 중 하나이자 추후 `DispatchQueue.main`을 대체하게 될 중요한 타입이기도 하다.

## 액터(Actor)

---

### Actor Model

[Actor model - Wikipedia](https://en.wikipedia.org/wiki/Actor_model)

액터는 악명높은 동시성 프로그램의 공유자원 관리를 돕기 위해 고안된 모델이다. **액터 내의 모든 프로퍼티 접근은 마치 queue처럼 반드시 하나씩, 순차적으로 실행**되는 것이 원칙이며 변할 수 있는 상태(변수 등)를 공유해서는 안된다. 덕분에 액터는 동시성 프로그래밍의 고질적인 문제점인 교착상태, 경쟁상태에서 어느정도 자유로워진다. 

### Actor in Swift

<aside>
💡 from [Language Guide](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html#Actors)
You can use tasks to break up your program into isolated, concurrent pieces. Tasks are isolated from each other, which is what makes it safe for them to run at the same time, but sometimes you need to share some information between tasks. Actors let you safely share information between concurrent code.

</aside>

Swift Concurrency도 이 액터 모델을 사용하고 있으며  `actor` 키워드를 이용해 직접 액터를 만들 수 있다([제안문서](https://github.com/apple/swift-evolution/blob/main/proposals/0306-actors.md)). `actor`는 Apple이 표현하기를 “**동시성의 바다(Sea of concurrency)**”에서 고립된(isolated) 상태를 유지하고 싶을 때 사용할 수 있다 ([출처](https://developer.apple.com/videos/play/wwdc2021/10133/)).

### Cooperative thread pool

[Swift Concurrency threading model questions - Development / Compiler - Swift Forums](https://forums.swift.org/t/swift-concurrency-threading-model-questions/49520/2)

많은 태스크가 액터의 맥락에서 실행되긴 하지만 모든 태스크가 그런 것은 아니다. 태스크는 액터와는 별개로 실행될 수 있다. 액터와 별개로 실행되는 태스크는 ‘무소속’으로 **Cooperative thread pool**라는 공유된 쓰레드 풀을 이용한다. 

Cooperative thread pool은 S.C가 병렬적인 태스크 실행을 위해 미리 만들어진 쓰레드 후보군이다. `Task`로 만들어지는 unstructured concurrency는 물론 `async let`과 `TaskGroup`으로 만들어지는 child 태스크들이 어떤 액터에 포함되어있지 않다면 반드시 이 쓰레드 풀에서 작업을 실행한다. 

### MainActor

메인액터는 `DispatchQueue`의 `main` 큐와 정확히 같은 역할을 하며 유일하게  `main` 쓰레드를 사용하는 시스템에 단 하나만 존재하는 액터다. 이전 메인 큐가 그러했듯 최우선으로 처리해야 하는 작업들이 이 곳에서 이루어진다. `@MainActor` 래퍼를 이용해 만들 수 있으며 이 경우 자동으로 해당 타입의 메소드들이 Swift Concurrency를 따르게 된다.

최신 버전의 UIKit에서 `UIViewController`같은  UI-related 클래스의 경우 이미 메인 액터로 선언된 것을 확인할 수 있다. 

### 액터와 thread pool 사이의 퍼포먼스 이슈

[Visualize and optimize Swift concurrency - WWDC22 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2022/110350/)

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%206.png)

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%207.png)

![Untitled](Swift%20Concurrency%20Deep%20Dive%20677409e88d3c4e8484c2ce9a34abf4a3/Untitled%208.png)

액터는 한 번에 하나의 태스크만 실행시키기 때문에 오히려 액터로 모든 태스크를 실행하면 병목현상이 발생할 수 있다. 그러므로 `nonisoloated` 키워드를 적절히 이용해 액터가 필요할 때만 접근케 하고 나머지 작업은 Cooperative thread pool에서 돌아가게끔 하는 것이 적절하다. 

### Task와의 관계

['current actor' semantics question - Using Swift - Swift Forums](https://forums.swift.org/t/current-actor-semantics-question/60120/4)

그러면 태스크와의 관계는 어떻게 되는 것인가? 태스크는 S.C 작업의 단위일 뿐이다. `actor` 안에서 실행되는 태스크가 있으며 그렇지 않은 태스크가 있다.

헷갈릴 수 있는데 다시 말하지만 모든 태스크가 액터 안에서 실행되는 것이 아니다. 액터는 그저 태스크 간의 안전한 자료 공유를 위해서 존재하는 보조 타입일 뿐이다. 

### Unstructured Concurrency와의 관계

`Task.init`을 통해 만드는 Uncstructured concurrency는 도큐먼트에 의하면 만들어진 맥락에 속하는 액터를 상속한다. 반면 `detached`를 통해 만드는 경우 액터를 상속하지 않는다. 그 때 만약 현 맥락의 태스크가 속한 액터가 없다면 `detached` 태스크는 일반 태스크와 같은 것일까? 

Task의 로컬 변수 상속 여부를 제외하면 둘다 Cooperative thread pool에서 돌아갈 것이고 그러므로 이는 옳다. ([제작자피셜](https://forums.swift.org/t/current-actor-semantics-question/60120/4))

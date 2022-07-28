#### References
- https://developer.apple.com/videos/play/wwdc2021/10134

# Swift concurrency의 structured programming

## Structured programming?
---
<p align="center">
    <img src="https://velog.velcdn.com/images/enebin777/post/e174c3dc-8640-4ce4-84f7-efd738d7a605/image.png" width=300/> 
  <img src="https://velog.velcdn.com/images/enebin777/post/f32629c1-b15e-431c-b961-8b3559a7f468/image.png" width=300/>
</p>

이전의 언어들은 첫번째 이미지 같은 구조(aka. `goto`)로 되어있어 가독성이 떨어지는 문제가 있었다. **Structured programming flow**는 그런데서 벗어나 더욱 가독성 좋고 위에서 아래로 흐르는 듯한 구조를 일컫는다. 이는 컨트롤 플로우에 유동성을 더해 코드를 이해하기 쉽게 만들어 준다.([wikipedia](https://ko.wikipedia.org/wiki/%EA%B5%AC%EC%A1%B0%EC%A0%81_%ED%94%84%EB%A1%9C%EA%B7%B8%EB%9E%98%EB%B0%8D))

#### 예시
<p align="center">
        <img src="https://velog.velcdn.com/images/enebin777/post/4865dea6-a53d-456e-8adc-e23c2baa3707/image.png" width=400/> 
</p>


## Swift concurrency의 structured 구조
---
### completion 방식
![](https://velog.velcdn.com/images/enebin777/post/213f4b5f-5378-4ee2-ae9a-d8fef7a2bbf7/image.png)
위는 completion을 사용한 지저분한 플로우의 예시다. 함수 리턴 시 결과를 못 받아올 뿐더러 어느 시점에 어느 클로저가 결과를 반환할지 쉽게 예측이 되지 않는다. 위에서 아래로 자연스럽게 흐르는 구조가 아니며 따라서 structured와는 거리가 멀다.  

### async/await 방식
![](https://velog.velcdn.com/images/enebin777/post/ea8de6d7-a33f-4936-b03a-e58bb658daf1/image.png)
이런 식으로 `async/await`을 사용하면 시퀀셜하게 함수를 리턴 받으며 진행할 수 있으며 고로 이는 structured 하다고 볼 수 있다. 또한 completion의 큰 실수 포인트였던 에러체크도 바로 할 수 있다는 것이 장점이다.


## Task
---
동시성을 관리한다는 것은 힘든 일이다. 이를 관리하기 위해 Swift는 Task라는 개념을 도입했다.

### structured concurrency의 기본개념
![](https://velog.velcdn.com/images/enebin777/post/654a43e4-f0c7-478f-a7e9-e63b2146a939/image.png)
변수를 사용하기 위해서는 값을 저장해야 한다. 만약 어느 함수의 결과가 변수에 저장되어야 한다면 Swift는 해당 함수의 작업이 끝날 때까지 기다려야 한다. 그래야 다음에 쓰니까. 

### `async let `
![](https://velog.velcdn.com/images/enebin777/post/6025cbca-63aa-468c-ac38-1f337819ce27/image.png)

그런데 마냥 기다리기엔 너무 많은 리소스가 낭비된다. 그럴 때엔 기다리는 동안 다른 작업을 하는 것이 해결책이 될 수 있다. 변수 앞에 async let 바인딩을 박아주면 이것이 가능해진다. 

엥 근데 왜 `async var`는 안 될까? 그 이유는 [여기](https://www.hackingwithswift.com/quick-start/concurrency/why-cant-we-call-async-functions-using-async-var) 설명되어 있다. 대충 var 선언을 할 경우 await으로 기다리는 동안 값이 변한다면 그 값을 쓸 건지 await으로 받아온 값을 넣을 건지, 그렇다면 불변을 보장해야 하는지 머리아프니까 let을 박았다는 이야기다.


![](https://velog.velcdn.com/images/enebin777/post/def8d004-33ee-48f0-a696-c97cf59b5f9f/image.png)

`async let`과 함께라면 깔쌈하게 structured를 구현하면서 async-way로 작업을 돌릴 수 있다. 일단 async로 던지기는 했는데 이제 이걸 어디서 받을지가 문제다. 그건 나중에 await이 있는데서 받으면 된다. 

#### Task의 3원칙
![](https://velog.velcdn.com/images/enebin777/post/389e41c2-a186-4b02-afa1-ab11695e656d/image.png)
- 특히 3번이 async let과 관련있는 내용인데 async로 선언하는 순간에는 태스크가 생성되지 않는다는 이야기.


### 예시
#### 잘못된 예시
![](https://velog.velcdn.com/images/enebin777/post/2ac93ffe-132e-439c-a9cf-76e91bbb7c18/image.png)
왜 잘못 되었을까? `URLSession.shared.data`에 	`await`을 박아버리면 `let (data, _) = ...` 라인에서 코드가 기다려버린다. 그러면 그 다음 `let (meataData, _) = ...`라인이 async하게 처리될 수 있음에도 불구하고 실행되지 못하고 `let image = ...` 라인은 기다릴대로 기다린다.

#### 옳은 예시
![](https://velog.velcdn.com/images/enebin777/post/d03a0cb4-3a66-49d4-b2f1-9f6d077479ac/image.png)
변수를 사용하는 쪽으로 await을 던져주면 효율적으로 작업을 처리할 수 있다. 


## Task의 부모-자식 트리 구조
---

![](https://velog.velcdn.com/images/enebin777/post/581bfd8c-63dd-45cc-aa4e-22c1c85a633c/image.png)

Swift는 트리 구조로 태스크를 관리를 한다. 하위 태스크는 상위 태스크는 엄격한 parent-child(위가 취소되면 아래도 날아가는) 관계를 갖는 것은 아니지만 하위 태스크는 상위 태스크에 감시당한다(scoped). 

이 트리구조의 룰은 **부모 태스크는 자식 태스크가 모두 완료되어야만 같이 완료**된다는 것이다. 이는 자식 태스크를 딱히 기다릴 필요가 없는 **비정상 플로우**에도 적용된다.

### 비정상 플로우 - cancellation
``` Swift
func fetchOneThumbnail(withId id: String) async throws -> UIImage {
    let imageReq = imageRequest(for: id), metadataReq = metadataRequest(for: id)
    async let (data, _) = URLSession.shared.data(for: imageReq) 
    async let (metadata, _) = URLSession.shared.data(for: metadataReq) 

    guard let size = parseSize(from: try await metadata),
          let image = try await UIImage(data: data)?.byPreparingThumbnail(ofSize: size) else {
            throw ThumbnailFailedError()
          }
    return image
}
```
위 코드에서 `metaData`를 await하는 도중 얘가 에러를 던지는 경우가 있을 수 있다. 동시성을 지원하지 않는 보통의 함수는 throw와 동시에 `return`되어 버려야 맞지만 Swift concurrency는 그대신 다른 방식, `cancellation`을 한다. 

![](https://velog.velcdn.com/images/enebin777/post/6b0ba027-8a1f-4cd3-9dce-6aa5315f2a96/image.png)

`metaData` Task가 도중에 알 수 없는 오류가 생겨 완료와 함께 에러를 때린다. 그러면 Swift는 다른 자식 Task들을(여기선 image data task) `cancel` 처리한다. 이는 loop의 `break`처럼 작업을 중지할 것을 의미하는 것이 아니라 해당 결과를 받기는 할 것이나 필요하지는 않음을 알리는 것이다.

![](https://velog.velcdn.com/images/enebin777/post/1f54b9be-134e-4711-9cfb-0a18ffe8e78c/image.png)
부모 task가 cancel될 경우 자식 task들에게도 cancellation이 전파(propagate)된다. 단, Task의 진행 여부에 영향을 미치는 것은 모든 자식 task가 finished로(초록 깃발 여기선) 표시되었는지의 여부다. 

![](https://velog.velcdn.com/images/enebin777/post/a74c2f9e-a8af-4a07-96e4-4164dfdaf5d7/image.png)
이렇게 말이다. 만약 추가적인 작업이 또 있다면 모든 하위 작업들이 완료했다고 리포트가 되어야 최상위 task(`fetchOne`)도 완료가 된다. 이는 ARC가 메모리를 관리하는 것처럼 컴파일러가 동시성을 관리할 수 있게 도와준다. 분할정복같은 느낌이다.

#### 한마디로 정리
![](https://velog.velcdn.com/images/enebin777/post/89d1edf0-5c52-4c63-a751-9e84d0296ffd/image.png)


#### 에러 헨들링  다른 예시
``` Swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    for id in ids {
        try Task.checkCancellation() // Throw error if the Task cancelled
        thumbnails[id] = try await fetchOneThumbnail(withID: id)
    }
    return thumbnails
}
```
`try`를 이용해 특정 태스크가 캔슬이 되었는지 확인하고 에러를 던질 수 있다. 에러를 던지면 모든 결과를 다 기다릴 필요 없이 task를 취소시킬 수 있다.

![](https://velog.velcdn.com/images/enebin777/post/b5f3a392-4627-4cb2-b505-de09e4b4c41f/image.png)
변수로도 체크할 수 있음.


## 그룹 태스크
---
![](https://velog.velcdn.com/images/enebin777/post/6ff3493d-d3a6-47bc-945f-af9ae799739c/image.png)
앞에서 봤던 것 처럼 `fetchThumbnails`는 루프를 돌면서 `fetchOneThumbnail`의 결과를 기다린다. `fetchOneThumbnail`는 또 두 하위 태스크(`URLSession.shared.data`)가 끝나야 완료된다. 이럴 경우 루프 안에서 이 모든 태스크를 기다리는 낭비가 생긴다. 이럴 때는 Group task를 사용해보도록 하자. 

``` Swift 
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: Void.self) { group in
        for id in ids {
            group.addTask {
                thumbnails[id] = try await fetchOneThumbnail(withID: id) 
            }
            
        }
    }
    return thumbnails
}
```

#### `addTask`
![](https://velog.velcdn.com/images/enebin777/post/c6901909-13f8-4cd8-afd1-41588e3ca5ba/image.png)
그룹 태스크는 `addTask` 메소드로 그룹에 하나하나 태스크를 더해서 await들을 또 하나의 태스크로 취급하게끔 만들어준다. 이렇게 퉁쳐서 관리하면 

![](https://velog.velcdn.com/images/enebin777/post/114cbf7a-cb86-40bb-abb2-66763dc32623/image.png)
하나의 큰 async 태스크로 만들 수 있다. 그러면 근데 또 뭐가 문제냐. 동시에 여러개의 태스크가 하나의 데이터를 바라보고 있으니 데이터의 atomic함을 보장할 수가 없게 된다. 
![](https://velog.velcdn.com/images/enebin777/post/749893b0-3254-42ee-bcd1-63c6ece25839/image.png)

다행히 컴파일러가 이를 잡아준다. 그건 알겠는데 이걸 해결하려면 또 어떻게 해야 하는지..?


# Sendable
#### Reference
- https://developer.apple.com/videos/play/wwdc2022/110351/
- https://developer.apple.com/documentation/swift/sendable

Group에 add 되는 task 클로저 들은 `Sendable` 프로토콜을 준수해야 한다. `Sendable`은 클로저가 캡처하는 값 중 atomic을 보장할 수 없는 타입이 존재하는지를 체크한다. 이는 data race를 막기 위함이다.

### 가능한 타입
Document에 따르면 다음 타입이 Sendable을 따른다.
> - Value types
> - Reference types with no mutable  storage
> - Reference types that internally manage access to their state
> - Functions and closures (by marking them with `@Sendable`)

특히 `class`는 `final`로 마크되어야 사용 가능하며 immutable한 stored property를 사용하면 안되고 다른 클래스를 상속해서는 안 된다.(이럴 거면 그냥 안 쓰는게..)

클로저나 함수의 경우 Sendable protocol을 채택하게 만드는 대신 `@Sendable` attribute를 붙이면 된다. 단, 함수와 클로저가 캡처하는 모든 값들은 반드시 `sendable`해야 한다.

Data race는 동시성을 지원하는 프로그램에서 서로 다른 작업(쓰레드)이 공유된 자원을 서로 사용, 수정하려고 할 때 조작의 타이밍이나 순서 등이 결과값에 영향을 줄 수 있는 상태를 말한다.

### 에러 고치기
Swift 컴파일러는 lexical context에서 mutable 한 variable을 캡처하면 데이터 레이스가 생길 수 있다는 에러를 표시한다(위 사진처럼). 하여튼 이를 고치려면 어떻게 해야할까.

``` Swift
func fetchThumbnails(for ids: [String]) async throws -> [String: UIImage] {
    var thumbnails: [String: UIImage] = [:]
    try await withThrowingTaskGroup(of: (String, UIImage).self) { group in 
        for id in ids {
            group.addTask {
                return (id, try await fetchOneThumbnail(withID: id)) 
            }
        }
        for try await (id, thumbnail) in group { 
            thumbnails[id] = thumbnail
        }
    }
    return thumbnails
}
```
간단한 해결법 -> task에서 데이터를 안 바꾸면 된다. 대신 task는 단순히 값을 리턴하기만 하면 된다. 그러면 `for (try) await` 구문을 이용해 후에 그룹에서 리턴되는 값들을 처리할 수 있다. 그룹은 어차피 한 번에 리턴되기 때문에(맞을까?).


### 그룹 내에서의 cancellation
그렇다면 그룹 안에 속한 task가 에러를 던지면 어떻게 될까? 이 경우 그룹 안의 모든 task가 implicit하게 cancel된다. 이는 `async let`과 같은 방식이다. 하지만 상위 task의 cancel은 implicit하지 않다. 이 경우 반드시 `cancelAll`을 이용해 태스크를 취소해야 한다. 그렇지 않으면 상위 태스크는 모든 하위 태스크가 완료될 때까지 기다릴 것이다.

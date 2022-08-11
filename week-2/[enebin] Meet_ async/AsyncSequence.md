#### References
- https://developer.apple.com/videos/play/wwdc2021/10132/

### 서론

![](https://velog.velcdn.com/images/enebin777/post/f09439de-8991-4084-ac03-f42087dbb625/image.png)
- 첫번째는 sync 코드 두번째는 async 코드이며 우리가 지금까지 잘 사용해 왔던 구조입니다.

![](https://velog.velcdn.com/images/enebin777/post/c842f55f-8b00-445c-a9f4-c70d9b2ed23d/image.png)

![](https://velog.velcdn.com/images/enebin777/post/f1709e6f-a517-4b6e-b2e1-478553b7972d/image.png)
- 비동기 코드를 사용하였을 때 시퀀셜한 작업이 필요한 순간 위와 같이 completion을 적절히 이용해서 결과를 처리해왔습니다. 

### async/await
![](https://velog.velcdn.com/images/enebin777/post/9ffd80b1-bbd8-4a68-8562-c236c80e6f05/image.png)
- async/await를 사용하면 이제 그러지 않아도 됩니다.


### Testing async
![](https://velog.velcdn.com/images/enebin777/post/7ed6ffb3-5168-4043-8dfb-d6c622295aeb/image.png)

테스트 할 때도 마찬가지입니다. 예전엔 `XCExpectation`을 썼는데 

![](https://velog.velcdn.com/images/enebin777/post/e062b6e6-d6c8-4d46-8b82-8cde043e6a60/image.png)
이제는 더 편해졌습니다.


---



## AsyncSequence

### Sequence가 뭘까요
- https://developer.apple.com/documentation/swift/sequence

> A sequence is a list of values that you can step through one at a time. The most common way to iterate over the elements of a sequence is to use a for-in loop:

Iterator같은 느낌인데 또 Iterator는 아닙니다. `Sequence`를 채택한 대표적인 타입은 `Array`, `Set`, `Dictionary` 등의 `Collection`이 있는데 `IteratorProtocol`을 채택하거나 `makeIterator`를 구현하면 커스텀 타입도 Iterator가 되어 `for in` 문법을 채택할 수 있습니다.

``` Swift
var iterator = values.makeIterator()
while let value = iterator.next() {
	// Do something	
}
```

``` Swift
struct Countdown: Sequence, IteratorProtocol {
    var count: Int

    mutating func next() -> Int? {
        if count == 0 {
            return nil
        } else {
            defer { count -= 1 }
            return count
        }
    }
}

let threeToGo = Countdown(count: 3)
for i in threeToGo {
    print(i)
}
```

#### Expected Performance
_Document에서 발췌_
> A sequence should provide its iterator in O(1). The Sequence protocol makes no other requirements about element access, so routines that traverse a sequence should be considered O(n) unless documented otherwise.

iterator는 O(1)의 복잡도로 제공되므로 시퀀스를 횡단하는 동작은 별다른 표기가 없다면 O(n)이 될 것입니다.


### Async에서의 Sequence
AsyncSequence는 Async Task들이 시퀀스로 구현되어있는 형태입니다. 예시부터 보면,

``` Swift
struct QuakesTool {
    static func main() async throws {
        let endpointURL = URL(string: "https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_month.csv")!

        for try await event in endpointURL.lines.dropFirst() {
            let values = event.split(separator: ",")
            let time = values[0]
            let latitude = values[1]
            let longitude = values[2]
            let magnitude = values[4]
            print("magnitude: \(magnitude), time: \(time), latitude: \(latitude), longitude: \(longitude)")
        }
    }
}
```
![](https://velog.velcdn.com/images/enebin777/post/7b4caf97-14f7-4074-864b-6834cb6ac5ef/image.png)

- 배열을 다루듯 자유롭게 `event`를 관리하고 `dropFirst`를 사용할 수도 있습니다. `Sequence`에서 사용하는 함수 중 상당수가 사용 가능합니다.
- AsyncSequence는 그룹으로 관리되어 그룹 안에 속하는 모든 Task가 완료 시 배열 전체를 한꺼번에 비동기 반환합니다.
- 참고로 이 케이스(`URL.lines`)에서는 task 완료 순서대로 `for`문 안의 context를 실행시킵니다. 즉, Task가 실행되는 데에는 정해진 순서가 없습니다.

![](https://velog.velcdn.com/images/enebin777/post/50e884f3-d013-4846-83c6-4b2f1b90e3dd/image.png)
- 필터도 됩니다.

![](https://velog.velcdn.com/images/enebin777/post/400ada59-b9c9-49be-93ef-515cf6243ab0/image.png)
- 여튼 보셨다시피 AsyncSequence는 그냥 Async가 붙은 시퀀스입니다.
- 현 context에서 에러를 던질 수도 완료, 에러로 인해 종료될 수도 있습니다.


### 동작 원리
![](https://velog.velcdn.com/images/enebin777/post/ae8ad5d3-b219-4722-8ceb-2627a0a6725d/image.png)
- Iterator가 async가 아닌 그냥 일반적인 경우라면 `for in` 안에서 컴파일러는 Iterator를 이런 식으로 핸들링합니다
- for in을 while let으로 치환합니다

![](https://velog.velcdn.com/images/enebin777/post/65dd531e-a1e5-45de-97f8-76e910bc18fa/image.png)

- async라고 크게 다를 것은 없습니다.
- 다만 일반 시퀀스처럼 처리하면 await으로 대기처리를 할 수 없으니 약간의 코드가 더해집니다.

![](https://velog.velcdn.com/images/enebin777/post/dae1ad8f-8768-45a7-a18c-5117f1214d28/image.png)
- 이런식으로..!















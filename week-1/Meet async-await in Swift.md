# Meet async/await in Swift

우리는 비동기 프로그래밍을 많이 사용합니다. Async/await가 도움이 될 수 있습니다. Async/await를 사용하면 비동기 코드를 일반적인 코드처럼 쉽게 작성할 수 있습니다. 또한 안전합니다. (왜 안전한지는 뒤에 나와요) 그 외에도 SDK에는 사용할 수 있는 수백 가지 대기 가능한(awaitable) 메서드가 있습니다. 예를 들어 UIKit은 UIImage에서 썸네일을 형성하는 기능을 제공합니다. 실제로 해당 작업을 완료하기 위해 동기 및 비동기 기능을 모두 제공합니다.

<img width="941" src="https://user-images.githubusercontent.com/60538517/181510802-45636746-1472-4c85-a4c2-0c57e8d5743f.png">

[Apple Developer Documentation](https://developer.apple.com/documentation/uikit/uiimage/3750845-preparethumbnail)

빠르게 보자면, 동기 함수(즉, 일반 이전 함수)를 호출하면 스레드가 차단되어 해당 함수가 완료되기를 기다립니다. 따라서 fetchThumbnail 함수가 prepareThumbnail(UIKit이 제공하는 동기 함수)을 호출하면 완료될 때까지 스레드는 다른 작업을 수행할 수 없습니다. 대조적으로, 해당 함수의 비동기 버전인 prepareThumbnail(of:completionHandler:)을 호출하는 경우 실행되는 동안 스레드는 자유롭게 다른 작업을 수행할 수 있습니다. 완료되면 완료 핸들러를 호출하여 알려줍니다. SDK는 많은 비동기 함수를 제공합니다. 그들은 몇 가지 다른 방법으로 완료되었음을 알려줍니다. 이와 같이 completion handler를 사용하는 것도 있고delegate callbacks를 사용하는 것도 있습니다. 그리고 많은 것들이 비동기로 표시되고 값만 반환합니다.

<img width="1829" alt="스크린샷 2022-07-24 오후 9 15 59" src="https://user-images.githubusercontent.com/60538517/181510833-35a7412e-3252-4bfe-b99b-80dfc440ce3f.png">
<img width="1826" alt="스크린샷 2022-07-24 오후 9 27 00" src="https://user-images.githubusercontent.com/60538517/181510846-5a6ebf34-5aa8-4660-91f3-c850b4243abb.png">

이중에 dataTask와 prepareThumbnail은 오래걸리는 작업입니다. 이것이 SDK가 이러한 작업을 완료하기 위해 비동기 함수를 제공하는 이유입니다. 따라서 이러한 호출은 비동기식이어야 합니다.

```swift
func fetchThumbnail(for id: String, completion: @escaping (UIImage?, Error?) -> Void) {
    let request = thumbnailURLRequest(for: id)
    let task = URLSession.shared.dataTask(with: request) { data, response, error in
        if let error = error {
            completion(nil, error)
        } else if (response as? HTTPURLResponse)?.statusCode != 200 {
            completion(nil, FetchError.badID)
        } else {
            guard let image = UIImage(data: data!) else {
                completion(nil, FetchError.badImage)
                return
            }
            image.prepareThumbnail(of: CGSize(width: 40, height: 40)) { thumbnail in
                guard let thumbnail = thumbnail else {
                    completion(nil, FetchError.badImage)
                    return
                }
                completion(thumbnail, nil)
            }
        }
    }
    task.resume()
}
```

함수는 인수로 문자열, 첫 번째 작업에 대한 입력 및 호출자에게 출력을 다시 제공하는 데 사용되는 완료 처리기를 사용합니다.

fetchThumbnail이 호출되면 먼저 thumbnailURLRequest를 호출합니다. 이 메서드는 동기식이므로 완료 처리기가 필요하지 않습니다. 

다음으로 shared URLSession 인스턴스에서 dataTask를 호출하여 해당 URLRequest와 completion handler를 전달합니다. 비동기 작업을 시작하기 위해 resume되어야 하는 URLSessionDataTask를 동기적으로 생성합니다. 

그런 다음 FetchThumbnail이 반환되고 스레드는 다른 작업을 자유롭게 수행할 수 있습니다. 

이미지를 다운로드하는 데 시간이 걸리고 데이터가 스트리밍되기를 기다리는 스레드를 차단하고 싶지 않기 때문에 이는 정말 중요합니다. 

결국 이미지 다운로드가 완료되거나 문제가 발생합니다. 어느 쪽이든 요청이 완료되고 dataTask에 전달된 완료 핸들러가 데이터, 응답 및 오류와 같은 몇 가지 선택적 값과 함께 호출됩니다.

FetchThumbnail의 호출자는 fetchThumbnail이 실패하더라도 작업이 완료되면 알림을 받을 것으로 예상합니다. "guard else return"을 작성하는 데 너무 익숙해서 완료 핸들러를 호출하는 것을 잊을 수 있습니다. 

그렇기 때문에 fetchThumbnail의 작성자인 우리는 무슨 일이 있어도 호출자에게 알리는 것이 매우 중요합니다. 오류가 발생하면 완료 핸들러를 호출하고 오류를 전달해야 합니다. Swift에게 fetchThumbnails와 같은 완료 핸들러는 클로저에 불과합니다. 항상 호출되도록 하고 싶지만, Swift에서는 그렇게 하도록 강제할 방법이 없습니다. 그래서 방금 두 guard에서 return했을 때 컴파일 오류가 발생하지 않았습니다. 완료 핸들러가 결국 호출되는지 확인하는 것은 사용자에게 달려 있습니다. 

위 4가지 작업을 순서대로 수행하는 것을 원했지만 결과는 따라가기 어렵고 제대로 하기도 어렵고, 우리의 의도를 흐리게 합니다.

### Safe way = async/await

이를 좀 더 안전하게 만들 수 있는 방법이 있습니다. 예를 들어 표준 라이브러리의 Result 유형을 사용할 수 있습니다. 이것은 조금 더 안전하지만 코드를 더 못생기고 약간 더 길게 만듭니다. 사람들은 또한 futures와 같은 기술을 사용하여 다른 방식으로 비동기 코드를 개선했습니다. 그러나 이러한 접근 방식 중 어느 것도 간단하고 쉽고 안전한 코드를 제공하지 않습니다. async/await를 사용하면 더 잘할 수 있습니다. 이번에는 async/wait를 사용했습니다.

```swift
func fetchThumbnail(for id: String) async throws -> UIImage {
    let request = thumbnailURLRequest(for: id)  
    let (data, response) = try await URLSession.shared.data(for: request)
    guard (response as? HTTPURLResponse)?.statusCode == 200 else { throw FetchError.badID }
    let maybeImage = UIImage(data: data)
    guard let thumbnail = await maybeImage?.thumbnail else { throw FetchError.badImage }
    return thumbnail
}
```

위 코드에서 Swift는 값을 리턴하든지 에러를 throw하도록 보장합니다. 이는 조용한 실패가 일어나지 않도록 합니다. 거기다 코드가 20줄에서 6줄로 줄었습니다.

이는 async/await를 사용하여 더 안전하고 더 짧고 더 직관적인 코드를 작성하는 한 가지 예일 뿐입니다.

### Async Property

함수 뿐만 아니라 프로퍼티도 async일 수 있습니다. 

thunbnail 프로퍼티는 SDK 일부가 아니라 로버트가 추가한 코드입니다.

```swift
extension UIImage {
    var thumbnail: UIImage? {
        get async {
            let size = CGSize(width: 40, height: 40)
            return await self.byPreparingThumbnail(ofSize: size)
        }
    }
}
```

여기서 주목해야 할 점이 두 가지 있습니다.

첫째, 명시적 getter가 있습니다. 이것은 속성을 비동기로 표시하는 데 필요합니다. Swift 5.5부터 속성 getter도 throw할 수 있습니다. 그리고 비동기 함수 시그니처와 마찬가지로 속성이 비동기이면서 throw인 경우 async 키워드는 throw 바로 앞에 옵니다.

둘째, 속성에는 setter가 없습니다. 읽기 전용 속성만 async일 수 있습니다.

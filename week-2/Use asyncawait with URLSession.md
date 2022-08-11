# 기존 URLSession

### `URLSession` 에서 `async/await` 가 어떻게 동작하는지부터 알아보자

- Swift concurrency
    - Linear
    - Concise
    - Native error handling
    - 

네트워킹은 본질적으로 비동기이며, 

`iOS 15 및 macOS Monterey` 에서 Swift 동시성 기능을 활용할 수 있도록 URLSession에 새로운 API를 도입함

<br>

<br>


# ⚠️ `URLSession` 에서`completionHandler` 를 기반

![https://velog.velcdn.com/images%2Fun1945%2Fpost%2F61c0d335-b62d-4f69-ab45-a757980b642c%2F%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202022-03-18%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%206.52.33.png](https://velog.velcdn.com/images%2Fun1945%2Fpost%2F61c0d335-b62d-4f69-ab45-a757980b642c%2F%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202022-03-18%20%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB%206.52.33.png)

`URLSession` 에서`completionHandler` 를 기반 편의 메서드를 사용 
코드는 간단해 보이며, 제한된 테스트에서 작동했습니다.


<br>


## 적어도 3가지의 오류가 있음

## 1. 제어 흐름

<img width="507" alt="스크린샷 2022-08-11 오후 8 09 27" src="https://user-images.githubusercontent.com/79178843/184120572-7fe1c227-bda6-465c-8117-038d87cbac42.png">



우리는 `data task` 를 만들고 그것을 `resume(실행)` 합니다.

그러고 나서 한번의 task 가 끝나면 우리는 `completion handler` 로 들어갑니다. 

response 를 체크하고, image 를 만들고 나면 제어 흐름이 종료됩니다.

코드가 앞뒤로 너무 왔다갔다함


<br>


<br>



## 2. threading

<img width="507" alt="스크린샷 2022-08-11 오후 8 09 59" src="https://user-images.githubusercontent.com/79178843/184120663-d82d778f-58ab-489f-b8cd-c8f900c30bee.png">


 총 3개의 서로 다른  `excution contexts` 가 있음

- 가장 바깥 레이어는 호출자의 `thread` 혹은 `Queue` 에서 동작하고
- `URLSessionTask completionHandler` 는 session 의 `delegate Queue` 에서 동작하고
- 마지막 completion handler 는 `main Queue` 에서 동작
- 

compiler 는 여기에서 우리를 도와줄 수 없기 때문에, 
data races 와 같은 모든 threading issues 를 피하기 위해서 주의를 기울여야함  !! 


<br>

<br>


## 3. completionHandler에 대한 호출이 main Queue에 일관적으로 전달되지 않음

<img width="500" alt="스크린샷 2022-08-11 오후 8 10 32" src="https://user-images.githubusercontent.com/79178843/184120749-4d7ddd61-93c0-41e3-9c85-8bafe586329a.png">


만약 오류가 발생되면 `completionHandler` 는 두 번 호출될 수 있음

-> 이것은 호출자가 만든 가정을 위반할 수 있습니다.

그리궁 `UIImage` 생성이 실패할 수 있습니다. 

만약 정확하지 않은 형태의 데이터라면, `UIImage 생성자` 는 nil을 반환하고

 `nil image` 와 `nil error` 를 가지고 있는 `completionHandler` 를 호출할 수도 있음 

<br>

---

# ✅ async/await 적용

<img width="500" alt="스크린샷 2022-08-11 오후 8 11 22" src="https://user-images.githubusercontent.com/79178843/184120861-8cd21288-f1c9-4116-ba47-d55031b8d967.png">


- 제어흐름은 위부터 아래로 선형이고
- 이 함수의 모든 것이 같은 `concuurency context` 에서 작동하므로
- 더 이상 `threading issues` 에 대해 걱정할 필요가 없음

<br>

<br>


<img width="687" alt="스크린샷 2022-08-11 오후 8 11 41" src="https://user-images.githubusercontent.com/79178843/184120905-f5bf41fa-234b-4221-8576-ed7c7ea45442.png">


`URLSession` 에서 새로운 async `data` method 를 사용함

```swift
URLSession.shared.data(from:)
```

현재 `실행 콘텍스트(코드들이 실행되기 위한 환경)` 를 차단하지 않고 멈춘 상태로,
`URLSession` 이 성공적으로 완료하면 `data` 와 `response` 를 반환하거나 `error` 를 던짐

<br>

<br>

<img width="500" alt="스크린샷 2022-08-11 오후 8 12 02" src="https://user-images.githubusercontent.com/79178843/184120953-01321f2c-c888-4a7a-a7b1-1d65eb01a6f4.png">


또한 `error` 를 던지기 위해 `throw` 키워드를 사용했음

호출자가 Swift 기본 error 처리 를 사용하여 error 를 처리하고, error 를 잡을 수 있음

<br>

<br>
 
<img width="666" alt="스크린샷 2022-08-11 오후 8 12 33" src="https://user-images.githubusercontent.com/79178843/184121020-5eeec579-136d-4aa2-8202-8d9b92dcb63c.png">

마지막으로, 만약 이 함수로부터 `optional UIImage` 를 반환하려고 시도한다면 `compiler` 는 주의를 주며, 기본적으로 `nil` 을 올바르게 처리하도록 강요합니다.

<br>

<br>

<br>

---

# ✅ URLSession.data

<img width="500" alt="스크린샷 2022-08-11 오후 8 13 05" src="https://user-images.githubusercontent.com/79178843/184121096-da6cd46e-05c2-4b0b-9db1-4c33f5e31463.png">

여기는 네트워크로부터 데이터를 가져오기 위해 사용한 메서드의 `signatures(부분)` 이다.

`URLSession.data 메서드`는 `URL` 혹은 `URLRequest` 를 채택

이것은 `data task 편의 메서드` 에 존재하는 것과 동일합니다.


<br>

<br>

<br>

---

# ✅ URLSession.upload

<img width="500" alt="스크린샷 2022-08-11 오후 8 14 28" src="https://user-images.githubusercontent.com/79178843/184121430-f7085efd-7f22-488b-96d8-3273e76c2c12.png">

URLSession.upload는 `data` 혹은 `file` 을 업로드 할 수 있음 

기본 메서드 GET 이 업로드를 더 이상 지원하지 않기 때문에

요청을 보내기 전에 정확한 `HTTP 메서드` 설정을 확실하게 해야함 


<br>

<br>

<br>

---

# ✅ URLSession.download

<img width="500" alt="스크린샷 2022-08-11 오후 8 14 02" src="https://user-images.githubusercontent.com/79178843/184121261-3479e79c-9755-42b2-988b-65c40327be19.png">

URLSession.download 는 response body 를 메모리가 아닌 **파일형태** 로 저장하고 자동으로 파일을 삭제하지 않음 → 직접 삭제해주는 것 잊지말기. 



<br>

<br>

<br>

---



# ✅ Cancellation

<img width="686" alt="스크린샷 2022-08-11 오후 8 13 46" src="https://user-images.githubusercontent.com/79178843/184121208-17326030-d198-4954-a8f7-50b37934e38f.png">


`Swift concurrency's cancellation` 은 `URLSession async 메서드` 에서 동작

 취소하는 방법

- concurrency `Task.Handle`  사용
여기서 async를 호출하여, 두 리소스를 하나씩  로드하는 동시성 작업을 만든다.
나중에 `task.cancel()`을 사용하여 현재 실행중인 작업을 취소할 수 있음

<br>

<br>

<br>

---



# ✅ URLSession.bytes

response header가 수신될 때 response body의 byte를 AsyncSequence로 전달함  

<img width="500" alt="스크린샷 2022-08-11 오후 8 17 45" src="https://user-images.githubusercontent.com/79178843/184121896-1de4edbf-4cef-4e73-b009-af30ac07025c.png">


```swift
func onAppearHandler() async throw {
	let (bytes, response) = try await URLSession.shared.bytes(from: Self.eventStreamURL)
   guard let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 200 else {
       throw DogsError.invalidServerResponse
   }
    // parse JSON Data 
	 for try await line in bytes.lines {
	     let photoMetadata = try JSONDecoder().decode(PhotoMetadata.self, from: Data(line.utf8))
       await updateFavoriteCount(with: photoMetadata)
	 }
}
```



<br>

<br>

<br>

---


## AsyncSequence

<img width="500" alt="스크린샷 2022-08-11 오후 8 16 50" src="https://user-images.githubusercontent.com/79178843/184121752-fdc0cc80-d17e-4c38-bbd5-37e51284b767.png">

- background URLSession에서 지원되지 않음

---

<img width="500" alt="스크린샷 2022-08-11 오후 8 16 29" src="https://user-images.githubusercontent.com/79178843/184121688-daa9dca8-5367-47e6-bdaa-1a62ae31be0f.png">

delegate 는 인스턴스변수가 아니며, 작업이 완료되거나 실패할 때 까지 작업이 유지됨

<img width="500" alt="스크린샷 2022-08-11 오후 8 15 58" src="https://user-images.githubusercontent.com/79178843/184121599-d5a7a159-c7d8-4ed2-8aeb-a7cb17f7228f.png">


→ delegate를 사용하여 URLSession task의 인스턴스에 특정한 이벤트를 처리할 수 있음

→ URLSession task에만 적용이 되고, 나머지에는 적용되지않아서 편리함

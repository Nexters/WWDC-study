### Meet AsyncSequence

[Meet AsyncSequence - WWDC21 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10058/)

---
---
---


# Async Sequence

- 비동기로 동작한다는 점을 제외하면 Sequence와 동일
- 예외를 던질 수 있음
- Sequence가 끝나거나 오류가 발생할 수 있음
- 기존 Sequence와 동일하게 for문에서 break, continue를 사용할 수 있음

<br>

<br>

<br>



# What is AsyncSequence?

아래 코드에서 모든 다운로드를 기다리고 싶지않음..

하지만! 순서대로 보여주고 싶음

<img width="500" alt="스크린샷 2022-08-11 오후 8 23 31" src="https://user-images.githubusercontent.com/79178843/184122841-82cb1880-2818-4d72-878f-ab0567fedca9.png">


# Iterating a AsyncSequence

<img width="723" alt="스크린샷 2022-08-11 오후 8 23 59" src="https://user-images.githubusercontent.com/79178843/184122937-bdc68962-9dd5-43c9-bc1f-92b352014424.png">

- Async Sequence의  문 사용은 기존 Sequence의 for문 사용과 거의 동일함
- **차이점**은 iterator에서 다음 값을 받아올 때 **비동기로 받아옴**



<img width="388" alt="스크린샷 2022-08-11 오후 8 24 15" src="https://user-images.githubusercontent.com/79178843/184122967-31f93613-3f7a-496e-b173-5cc864de4ee3.png">


✅  `for try` 구문을 사용한 경우 do - catch 문을 통해서 예외 처리를 할 수 있음

✅  전체 for문을 비동기로 실행하기 위해서 전체 블록을 async로 감쌀 수 있음

✅ 비동기로 처리 중인 작업을 이후에 cancel 함수를 호출하여 취소할 수 있음

<br>

<br>



# Bytes from a FileHandle

<img width="685" alt="스크린샷 2022-08-11 오후 8 24 32" src="https://user-images.githubusercontent.com/79178843/184123005-6083e982-db02-4797-92ec-6fab60233830.png">

파일에서 각 byte를 읽을 때, 비동기로 읽을 수 있음 ㅎ

<br>

<br>

<br>


# Read lines from a URL

<img width="676" alt="스크린샷 2022-08-11 오후 8 26 17" src="https://user-images.githubusercontent.com/79178843/184123317-85675a40-1520-4e2c-b21d-31df15abe8cb.png">


URL에서 바이트나 한 줄씩 데이터를 받아올 때도 비동기로 처리할 수 있음 ㅎ

<br>

<br>

<br>



# Bytes from a URLSession

<img width="680" alt="스크린샷 2022-08-11 오후 8 26 01" src="https://user-images.githubusercontent.com/79178843/184123267-7dc4b978-fc1c-4956-a216-13ae64d054b4.png">

`URLSession`에서 바이트를 읽을 때에도 비동기로 읽어올 수 있음 ㅎ 

<br>

<br>

<br>


# Notifications

<img width="682" alt="스크린샷 2022-08-11 오후 8 25 50" src="https://user-images.githubusercontent.com/79178843/184123229-64852272-c220-45dc-824c-a00a4d61f534.png">

notification 이벤트를 기다려서 받아올 때도 비동기

<br>

<br>

<br>



# AsyncStream

<img width="656" alt="스크린샷 2022-08-11 오후 8 25 38" src="https://user-images.githubusercontent.com/79178843/184123202-6c87dcca-c0da-4237-be80-05859f02d4e5.png">


✅  `AsyncStream` 객체를 사용하여 Async Sequence를 생성할 수 있음

✅  전달받은 continuation의 yield를 호출하여 데이터를 전달할 수 있음

✅  continuation에 onTermination 콜백을 등록하여 취소되었을 때의 처리를 추가할 수 있음


<img width="682" alt="스크린샷 2022-08-11 오후 8 25 18" src="https://user-images.githubusercontent.com/79178843/184123159-9052bae7-4b10-4fca-b5bf-6e898072635f.png">

- `AsyncStream`은 기존 callback이나 delegate를 사용하여, 
코드를 Async Sequence로 변경해 줄 수 있음
- 버퍼링이나 취소 등의 비동기 상황에 필요한 일을 처리할 수 있음
- 예외가 필요한 경우에는 동일한 기능에 예외까지 지원하는 `AsyncThrowingStream`을 사용할 수 있음

<br>

<br>

<br>


---

# 핵심 사항 요약

- Async 함수를 사용하면 await키워드를 사용하여 call back 없이 동시 코드를 작성할 수 있음
- 호출은 일시 중단된 다음 값이나 오류가 생성될 때 다시 시작됨
- `AsyncSequence`는 각 요소에서 suspend되고 iterator가 값을 생성하거나 throw할 때 시작됨

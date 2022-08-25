
![](https://velog.velcdn.com/images/enebin777/post/78cd5798-1d0f-42d7-a870-94314a2ccd34/image.png)
- 이런 식으로 태스크가 관리되는 코드를 만들고 싶다. 
- 네트워크는 concurrent하게 작업하고 데이터베이스 저장은 serial하게 작업하고 싶다. 

![](https://velog.velcdn.com/images/enebin777/post/10841fc6-0375-4244-a7db-19239887a155/image.png)
- 코드로는 이렇게 구현하면 된다.
- 그런데 얘는 근데 문제가 하나 있는데

![](https://velog.velcdn.com/images/enebin777/post/3eb1b022-754a-4528-9447-57038800d0c5/image.png)
- concurrent 큐에서 async하게 작업을 진행하던 쓰레드가 작업 도중 데이터베이스(serial) 큐의 sync를 호출하면 CPU는 해당 쓰레드를 킵해놓고 다음 쓰레드의 작업을 하게 된다.
- 이 경우 코어가 놀지 않게끔 새로운 쓰레드를 만들게 된다.


![](https://velog.velcdn.com/images/enebin777/post/8b7a68a3-6bf0-4d8e-ae5f-25db8ed67c3d/image.png)
- 만들고 만들다 보면 결국 CPU가 감당할 수 없는 수준으로 쓰레드를 만들게 된다. 이를 **쓰레드 익스플로전**이라고 하고 이는 심각한 문맥교환 오버헤드를 발생시킨다.
- 다음 파트에서 자세히 알아보도록 하자.


### 쓰레드 익스플로전
#### 메모리 핵낭비
![](https://velog.velcdn.com/images/enebin777/post/dc1832e1-6a9b-43cb-b0e3-ab722ed38918/image.png)
- 킵해놓은 쓰레드는 메모리 어딘가에서 떠돌고 있을 것이다.
- 각각의 쓰레드에서는 실행 흐름을 저장하는 stack이 있고, 그와 관련된 kernal 자료구조가 쓰레드를 추적하고 있다. 
- 몇몇 스레드는 lock(semaphore)으로 붙잡혀있어, 다른 쓰레드의 동작이 필요할 수도 있다. 
- 그러므로 코어 수를 뛰어 넘어 처리되지 않는 스레드가 오래 유지될 경우 이는 결국 메모리의 낭비를 초래함.


#### 스케줄링 오버헤드
![](https://velog.velcdn.com/images/enebin777/post/ba4263e2-be75-4f78-a23f-789cbdf2e54f/image.png)
- 쓰레드가 많으면 그만큼 context switching도 많아진다. 
- 이는 코어의 퍼포먼스를 저해한다. 


### Swift concurrency
#### `Continuation`
![](https://velog.velcdn.com/images/enebin777/post/fd3b3692-88e4-4d28-95f6-f00b627c70a6/image.png)
![](https://velog.velcdn.com/images/enebin777/post/8504ac91-e589-4d3d-a5fa-21e1efee039f/image.png)
- 쓰레드 블록을 날려버리고 `continuation`이라는 새로운 개념을 도입했다.
- Swift는 core 수에 맞는 쓰레드만 사용하며 따라서 Blocking Thread와 Full Thread Context Switching이 없어진다.
- `continuation`은 작업을 힙에 저장하는데 이를 간단히 Full Thread Context Switching대신 function call로 대체할 수 있는 장점이 있다. 
- 위 두 개념의 차이에 주목할 필요가 있음. 쓰레드가 들락날락 하는 건 각자 관리하는 스택이 저장되었다 복구되었다 하는 귀찮은 과정을 거치지만 Continuation이 들락날락 하는 것은 이런 저장-복구가 없어도 된다는 뜻임. 그래서 쓰레드가 코어당 1개로 유지된다는 이야기가 나오는 것.



#### GCD -> Swift concurrency 예시
![](https://velog.velcdn.com/images/enebin777/post/0aa5b1a1-69a6-4154-b51b-198304045b11/image.png)


### S.C는 쓰레드를 버린다
- GCD에서 Concurrent queue는 작업을 하려면 새로운 쓰레드를 만들었지만, 얘는 걍 자신의 쓰레드를 버리고 다른 작업에 양보한다. (`suspend`, not `block`)
- 이는 자기 쓰레드의 반환을 기다리지 않고 다른 쓰레드에 챡 하고 가서 붙을 수 있는 장점이 있다. 
- 어떻게 가능할까??

#### 원래의 모습
![](https://velog.velcdn.com/images/enebin777/post/0f6db3fb-c52a-43d7-9416-021688e5a0d2/image.png)
- 원래 쓰레드는 상태를 저장하는 스택을 **각자** 가지고 있다

![](https://velog.velcdn.com/images/enebin777/post/e0ba1044-dda1-4892-80c2-a0ee831153ae/image.png)
- 새로운 함수가 호출되면 해당 함수의 frame이 stack에 push된다. 이 frame은 지역 변수 저장, 반환 주소 전달 등등을 위해 사용된다.
- 일 끝나면 pop 돼서 해제될 것이다.

#### Async
![](https://velog.velcdn.com/images/enebin777/post/90e971f3-9eb4-4f8a-9e05-83a0f66efc1b/image.png)
![](https://velog.velcdn.com/images/enebin777/post/68be1079-807c-4e7d-8d07-b6d511d06f5d/image.png)

- 얘도 시작은 비슷하다.
- 각 순간에 필요한 정보를 스택에 밀어넣는데 거기에 규칙이 있다. await 등으로 추후 사용이 필요하지 않은 변수들만 들어간다. 
- 여기서는 `id`, `article`이 들어간다.

![](https://velog.velcdn.com/images/enebin777/post/704545f1-fe8b-488c-b379-9f1871775548/image.png)
![](https://velog.velcdn.com/images/enebin777/post/c314c752-e6ef-4204-9f7b-a37bc0008a95/image.png)

- 힙에는 `await`등의 이유로 지연된 사용이 필요한 정보들이 들어간다 이 경우 `newArticles`가 들어간다.(`async frame`) 
- 이의 존재 이유는, suspendsion point를 넘나들어 저장해야 하는 정보를 담기 위함이다.

![](https://velog.velcdn.com/images/enebin777/post/91dd1402-8870-4f8b-8831-16ddb40f7ffa/image.png)
- 이제 save 함수가 실행되면, add는 save를 위한 새로운 stack frame으로 대체된다.
- 이전처럼 새로운 stack frame을 push하는 것 대신, top에 있는 stack frame을 pop하고 push한다.
- 이는 후에 필요한 변수(newArticles)가 이미 힙에 저장되어있기 때문이다.


![](https://velog.velcdn.com/images/enebin777/post/3af1e560-e35d-4559-a6f3-37f99cfaa2a0/image.png)
- 이렇게 힙에는 여러 개의 정보가 저장되어 있다. 대신 stack은 깔끔하게 관리 됨. 
- 싱크 걸리면 블록걸리고 다른 쓰레드를 기다리는 게 아니라 그냥 다른 거를 실행시키면 된다.(단일 쓰레드기 때문)

![](https://velog.velcdn.com/images/enebin777/post/6f89c3da-fbc1-471e-aad4-591e10d46655/image.png)
- 이런 식으로 위 과정을 반복하며 스택을 자유롭게 사용한다. 
- non-async는 대체하지 않고 대신 위에 쌓인다.

---
## Swift concurrency의 장점 한번 더
![](https://velog.velcdn.com/images/enebin777/post/e149ec9c-c400-499e-b5d2-3686608992f3/image.png)


![](https://velog.velcdn.com/images/enebin777/post/2d10405b-f7b0-4d2d-a0a7-ef656025bd0b/image.png)

- SC는 코어 수 만큼만 쓰레드를 만든다


![](https://velog.velcdn.com/images/enebin777/post/03afc2bc-a17f-442c-9e60-2dda28d87e76/image.png)
- GCD 쓸 땐 하나의 시스템 당 하나의 시리얼 큐만 넣기로 권장함
- 그런거 신경쓸 바에는 SC 써라 - Cooperative pool

---
## Best practices

![](https://velog.velcdn.com/images/enebin777/post/2f856515-0646-4672-9444-5afc146b42f6/image.png)
- 굳이 이걸 await으로 해서 막을 필요가 있는지. 태스크 관리 비용이 더 크다

![](https://velog.velcdn.com/images/enebin777/post/91a2a371-2453-4616-8112-f92af3a6038d/image.png)
- 얘네는 atomicity가 없다.


![](https://velog.velcdn.com/images/enebin777/post/78f2e57d-96b4-479f-9c28-f636975a75e6/image.png)
- `await`에 락을 걸 수 없다. 왜냐하면 쓰레드에 락을 걸고 다른 쓰레드로 가버리면 그 쓰레드는 절대 해제될 기회가 없기 때문이다?

![](https://velog.velcdn.com/images/enebin777/post/e430223b-ac24-495a-b715-eaadfb7c8bb0/image.png)
- 터지는 코드

![](https://velog.velcdn.com/images/enebin777/post/9cc358a1-f04a-429d-be02-1e170a9a33f7/image.png)

--- 
## Mutex
![](https://velog.velcdn.com/images/enebin777/post/8eb73cf7-43a4-4b56-9d3f-471964513301/image.png)
- sc는 자동으로 뮤텍스를 보장한다. sync, async를 구분하지 않아도 쓰레드를 재사용하며 쓰레드를 블록하지도 않는다.

![](https://velog.velcdn.com/images/enebin777/post/e8d23f7d-5123-40d7-8e36-9de7d2979345/image.png)
![](https://velog.velcdn.com/images/enebin777/post/21756e53-ab76-4c64-9528-905df335b872/image.png)
- 이런 식으로 큐 하나가 여러개의 액터로 새롭게 관리된다.
- 액터들은 Cooperative thread pool에서 쓰레드가 관리된다.

![](https://velog.velcdn.com/images/enebin777/post/cc34b20c-b54f-455d-be0b-92c18da9fa64/image.png)
- 액터가 관리되는 예시다. 
- 액터들은 쓰레드를 새로 만들지도, 쓰레드를 블록하지도 않는다. 
- 쓰레드를 공유하며 다음에 어떤 작업을 할지 유동적으로 선택함.




### 우선순위 역전
#### GCD가 처리했던 방법
![](https://velog.velcdn.com/images/enebin777/post/a166c4ae-eb82-4747-aed9-f5f0f2fcb6a5/image.png)

![](https://velog.velcdn.com/images/enebin777/post/3930d9bf-20fa-4245-9cba-88a55878d5b9/image.png)
- 우선순위 역전이란 시리얼 큐에서 QoS가 높은 작업이 뒤로 밀려있어 우선순위가 높음에도 앞 작업들이 끝날 때까지 아무것도 하지 못하는 상태를 일컫는다. 
- GCD는 그냥 앞에 우선순위를 다 올려버려서 해결함. 


#### SC는? -> reetracny(재진입성)
![](https://velog.velcdn.com/images/enebin777/post/517fca89-1503-4f46-9007-802e4e2a7f5e/image.png)
- 얘는 그냥 위로 올려버린다. 
- 왜냐하면 재진입성을 가지고 있기 때문. 


![](https://velog.velcdn.com/images/enebin777/post/83bcc37a-709a-40fd-8ead-5adff8b390af/image.png)
![](https://velog.velcdn.com/images/enebin777/post/dea77a52-03db-4d77-a690-461b60e5cde8/image.png)

---
### 메인 액터
![](https://velog.velcdn.com/images/enebin777/post/79592bad-04a7-40b7-b3de-cc070962e892/image.png)
- 메인 액터는 메인쓰레드 같은 거다.
- 메인 액터 쓰레드는 Cooperative thread pool이랑 다른 거다.
- 그러므로 이 둘 사이의 전환은 context switching을 발생시킨다. 

![](https://velog.velcdn.com/images/enebin777/post/f959ad5e-02a6-43d1-9e40-5e11db81e572/image.png)
![](https://velog.velcdn.com/images/enebin777/post/5e4adab0-252f-4622-b9da-a1456e796d2a/image.png)

- 위 코드는 database에서 article을 로드하고 각 기사의 UI를 업데이트하는 예시이다. loop에서 적어도 두번의 context switching이 일어난다.
  - main actor에서 database actor로
  - database actor에서 main actor로
- 이러면 오버헤드가 발생한다. 

![](https://velog.velcdn.com/images/enebin777/post/daf9ab73-eb59-4831-82a7-50b417a2d02f/image.png)
- 루프가 없도록 코드를 수정해서 이렇게 깔쌈하게 만들 수 있다. 조심하도록 하자. 

`MainActor?` https://developer.apple.com/documentation/swift/mainactor












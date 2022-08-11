
![](https://velog.velcdn.com/images/enebin777/post/78cd5798-1d0f-42d7-a870-94314a2ccd34/image.png)
- 이런 식으로 태스크가 관리되는 코드를 만들고 싶다. 
- 네트워크는 concurrent하게 작업하고 데이터베이스 저장은 serial하게 작업하고 싶다. 

![](https://velog.velcdn.com/images/enebin777/post/10841fc6-0375-4244-a7db-19239887a155/image.png)
- 코드로는 이렇게 구현하면 된다.
- 그런데 얘는 근데 문제가 하나 있는데

![](https://velog.velcdn.com/images/enebin777/post/3eb1b022-754a-4528-9447-57038800d0c5/image.png)
- concurrent 큐에서 async하게 작업을 진행하던 쓰레드가 작업 도중 데이터베이스(serial) 큐의 sync를 호출하면 CPU는 해당 쓰레드를 킵해놓고 다음 쓰레드의 작업을 하게 된다.
- 이 경우 쓰레드를 새로 만들게 된다.


![](https://velog.velcdn.com/images/enebin777/post/8b7a68a3-6bf0-4d8e-ae5f-25db8ed67c3d/image.png)
- 만들고 만들다 보면 결국 CPU가 감당할 수 없는 수준으로 쓰레드를 만들게 된다. 이를 **쓰레드 익스플로전**이라고 하고 이는 심각한 문맥교환 오버헤드를 발생시킨다.
- 다음 파트에서 자세히 알아보도록 하자.


### 쓰레드 익스플로전
#### 메모리 핵낭비
![](https://velog.velcdn.com/images/enebin777/post/dc1832e1-6a9b-43cb-b0e3-ab722ed38918/image.png)
- 킵해놓은 쓰레드는 메모리 어딘가에서 떠돌고 있을 것이다.
- 각각의 쓰레드에서는 실행 흐름을 저장하는 stack이 있고, 그와 관련된 kernal 자료구조가 쓰레드를 추적하고 있다. 
- 몇몇 스레드는 lock으로 붙잡혀있어, 다른 쓰레드의 동작이 필요할 수도 있다. 
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



#### GCD -> Swift concurrency 예시
![](https://velog.velcdn.com/images/enebin777/post/0aa5b1a1-69a6-4154-b51b-198304045b11/image.png)


### S.C는 쓰레드를 버린다
- GCD에서 Concurrent queue는 작업을 하려면 새로운 쓰레드를 만들었지만, 얘는 걍 자신의 쓰레드를 버리고 다른 작업에 양보한다. (`suspend`, not `block`)
- 어떻게 가능할까??

#### 원래의 모습
![](https://velog.velcdn.com/images/enebin777/post/0f6db3fb-c52a-43d7-9416-021688e5a0d2/image.png)
- 원래 쓰레드는 각자의 상태를 저장하는 스택을 가지고 있다

![](https://velog.velcdn.com/images/enebin777/post/e0ba1044-dda1-4892-80c2-a0ee831153ae/image.png)
- 새로운 함수가 호출되면 해당 frame이 stack에 push된다. 이 frame은 지역 변수 저장, 반환 주소 전달 등등을 위해 사용된다.
- 일 끝나면 pop 된다. 

#### Async
![](https://velog.velcdn.com/images/enebin777/post/90e971f3-9eb4-4f8a-9e05-83a0f66efc1b/image.png)

- 얘도 시작은 비슷하다.

![](https://velog.velcdn.com/images/enebin777/post/704545f1-fe8b-488c-b379-9f1871775548/image.png)
- 서스펜션 포인트(`await`)에 스택에 있는 정보를 힙(`async frame`으로)에 저장한다. 
- 이의 존재 이유는, suspendsion point를 넘나들어 저장해야 하는 정보를 담기 위함이다.

![](https://velog.velcdn.com/images/enebin777/post/91dd1402-8870-4f8b-8831-16ddb40f7ffa/image.png)
- 이제 save 함수가 실행되면, add는 save를 위한 새로운 stack frame으로 대체된다.
- 새로운 stack frame을 push하는 것 대신, top에 있는 stack frame을 대체한다.
- 이는 후에 필요한 변수(newArticles)가 이미 힙에 저장되어있기 때문이다.


![](https://velog.velcdn.com/images/enebin777/post/3af1e560-e35d-4559-a6f3-37f99cfaa2a0/image.png)
- 싱크 걸리면 블록되는게 아니라 그냥 다른 거를 실행시키면 된다.

![](https://velog.velcdn.com/images/enebin777/post/6f89c3da-fbc1-471e-aad4-591e10d46655/image.png)
- 이런 식으로 위 과정을 반복하며 스택을 자유롭게 사용한다. 










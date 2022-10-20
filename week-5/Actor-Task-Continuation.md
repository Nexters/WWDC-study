## Actor model
#### References
- https://hyojabal.tistory.com/1
- https://www.avanderlee.com/swift/actors/ -> 대박!
- https://www.avanderlee.com/swift/nonisolated-isolated/
#### Docs
- https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html

#### Session
- https://developer.apple.com/videos/play/wwdc2022/110350

### 액터는
1. 다른 액터에게 유한한 개수의 메세지를 보낼 수 있습니다.
2. 유한한 개수의 새로운 액터를 만들 수 있습니다.
3. 다른 액터가 받을 메세지에 수반될 행동(behavior)을 지정할 수 있습니다.

Actor는 서로간에 공유하는 자원이 없고 서로간의 상태를 건드릴 수도 없다. 오직 message만을 이용해서 정보를 전달할 수 있다. 이로써 isolated한 환경을 구축할 수 있다.

---

### Task
![](https://velog.velcdn.com/images/enebin777/post/f08cd39a-8056-445e-8f35-1bdafcde6edd/image.png)
![](https://velog.velcdn.com/images/enebin777/post/fe9f9d8b-9496-429f-ae3d-3ae9f3df3ce3/image.png)

Task: 작업 단위
- Cancel같은 작업들을 처리함

### Actor
> **From official docs**
You can use tasks to break up your program into isolated, concurrent pieces. Tasks are isolated from each other, which is what makes it safe for them to run at the same time, but sometimes you need to share some information between tasks. __Actors let you safely share information between concurrent code.__
![](https://velog.velcdn.com/images/enebin777/post/66a12425-72d7-4794-8025-40cc5641775c/image.png)

Actor: 작업 단위 및 처리주체
![](https://velog.velcdn.com/images/enebin777/post/133a2178-a661-4bbc-9569-5fdcb15fd3f5/image.png)
Actor가 태스크를 처리하는 것이므로 이런 문제가 생길 수도 있다. 
![](https://velog.velcdn.com/images/enebin777/post/b8bf8df9-ebb3-4ac1-803a-7374cb6d6190/image.png)
![](https://velog.velcdn.com/images/enebin777/post/0bf3d783-ed2b-4031-bcf5-7e7f6ce9b045/image.png)

이렇게 해결한다. 
![](https://velog.velcdn.com/images/enebin777/post/5e30f42c-548a-418b-92b3-c1e6a4bfaefd/image.png)
![](https://velog.velcdn.com/images/enebin777/post/a4e10eb3-9461-4d8e-9bd9-45e2c2ac25d9/image.png)
한 태스크는 한 번에 한 액터만


### Continuation
- https://developer.apple.com/documentation/swift/checkedcontinuation
> A continuation is an opaque representation of program state. To create a continuation in asynchronous code, call the withUnsafeContinuation(function:_:) or withUnsafeThrowingContinuation(function:_:) function. To resume the asynchronous task, call the resume(returning:), resume(throwing:), resume(with:), or resume() method.



![](https://velog.velcdn.com/images/enebin777/post/52289276-4173-4a8b-8647-92017afe334f/image.png)
![](https://velog.velcdn.com/images/enebin777/post/82bb375b-90c5-4141-a45c-2302cb9d1a4c/image.png)

Continuation: 개념

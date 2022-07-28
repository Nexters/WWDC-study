**🔥** 새롭게 등장한 Swift Collections는 `Deque`, `OrderedSet`, `OrderedDictionary`를 제공

### Queue
<img width="332" alt="스크린샷 2022-07-29 오전 12 24 14" src="https://user-images.githubusercontent.com/79178843/181576187-f47dc899-d19b-44fd-b771-f38105ad689a.png">

`.append()`

`.removeFirst()`

`.removeLast()`



### Deque

Deque는 배열처럼 인덱스를 활용하며, 자료의 양쪽 끝의 삽입과 삭제는 배열보다 뛰어난 성능을 보여줌.

또한, 배열과 같이 `RangeReplaceableCollection`, `MutableCollection`, `RandomAccessCollection`를 준수하여 자료를 다루는데 친숙한 인터페이스를 제공

<img width="555" alt="스크린샷 2022-07-29 오전 12 24 30" src="https://user-images.githubusercontent.com/79178843/181576247-54dc8f3b-97c2-4a3a-9040-027244d71023.png">

삽입했을 때 Array와 Deque의 비교했을 때 Deque는 배열처럼 인덱스를 활용하며, 자료의 양쪽 끝의 삽입과 삭제는 배열보다 뛰어난 성능을 보임
<img width="556" alt="스크린샷 2022-07-29 오전 12 27 28" src="https://user-images.githubusercontent.com/79178843/181576977-7da96909-ddbc-4bab-8086-80492d79f16f.png">


`.removeSubrange`
<img width="556" alt="스크린샷 2022-07-29 오전 12 27 37" src="https://user-images.githubusercontent.com/79178843/181577021-6a0bf508-9a54-4443-b0ee-0d5cebab9a5c.png">


하지만 맨 앞이 아닌 임의의 인덱스에 대한 작업을 주로 할 때는 `Deque`의 도입을 조금 더 고민해봐야 합니다.

그래프 상에서 볼 수 있듯, 인덱스를 통한 접근은 `Deque`가 배열보다 속도가 약간 느리기 때문에

모든 것을 `Deque`로 바꾸는 것은 좋지 않음




### Unordered sets
<img width="561" alt="스크린샷 2022-07-29 오전 12 29 19" src="https://user-images.githubusercontent.com/79178843/181577381-c27ab875-21a1-472f-947c-7f9c000f9613.png">


### Ordered sets

`OrderedSet`은 Hashable Protocol을 준수하는 원소 타입을 가진 정렬된 집합을 제공

Array와 같이 순서를 유지하지만, 동시에 Set처럼 **동일한 요소는 하나만 추가할 수 있음!**

원소의 정렬이 중요한 집합이 요구될 때, 기존에 있던 `Set`의 좋은 대안이 됨

두 OrderedSet의 원소들이 같지만, 정렬된 순서가 다르면 두 `OrderedSet`은 다른 집합이 됨

원소만 같은지 비교하기 위해선 unordered를 통해 비교해야 함

<img width="556" alt="스크린샷 2022-07-29 오전 12 29 30" src="https://user-images.githubusercontent.com/79178843/181577417-410eedd9-f9b2-4134-9d84-d7d832940b96.png">


`.append()` `.insert()` `.remove()`   모두 O(1) 시간복잡도를 가짐
<img width="559" alt="스크린샷 2022-07-29 오전 12 30 36" src="https://user-images.githubusercontent.com/79178843/181577668-d5fe0732-9571-48b3-aa6f-0177547b5a8e.png">



### **Dictionary와 OrderedDictionary**

`Dictionary`는 키와 값의 두 해시테이블로 구성되나, 
`OrderedDictionary`는 키와 값의 두 배열과 그 인덱스가 담긴 해시테이블로 구성됩니다.

<img width="492" alt="스크린샷 2022-07-29 오전 12 30 50" src="https://user-images.githubusercontent.com/79178843/181577733-f3c33b3b-06be-44b4-875e-3003b6f43e2c.png">


`OrderedDictionary`는 인덱스를 통한 참조를 할 수도 있지만 키를 통한 참조와 구별되어야 함

인덱스를 통한 참조를 위해서 무엇을 참조할지 Keys, Values 같은 요소를 경유해야만 함

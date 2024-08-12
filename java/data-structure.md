### ArrayList vs LinkedList

-   ArrayList의 경우 배열을 통해서 구현하지만 LinkedList의 경우 Node간의 연결을 통해서 구현한다.  
    그렇기 때문에 ArrayList의 경우 배열처럼 랜덤액세스가 가능하지만 LinkedList의 경우 순회해서 접근해야 한다.
-   반대로 삭제의 경우 LinkedList는 해당 노드의 연결만 끊고 앞뒤 노드를 연결해주면 되지만  
    ArrayList의 경우 마지막 인덱스의 값을 삭제하는게 아닐 경우 삭제한 값 뒤의 값들을 한칸씩 이동시켜주어야 한다.
-   LinkedList는 Deque의 구현체로 사용할 수 있지만 ArrayList는 불가능하다.

### Vector vs ArrayList

-   Vector는 java 1.0부터 존재하였고 ArrayList는 java 1.2에서 추가되었으며 Vector의 개선된 버전이라 볼 수도 있다.
-   둘의 가장 큰 차이는 Vector의 경우 동기화를 지원하여 멀티스레드 환경에서 안전하게 사용될 수 있는 반면,  
    ArrayList는 동기화를 제공하지 않는다. 대신 단일 스레드 환경의 경우 더 빠른 성능을 제공한다.
-   메모리적 차이로는 Vector의 경우 배열 공간이 부족하면 배열의 크기를 100% 증가 시키는 반면  
    ArrayList는 배열 공간이 부족하면 크기를 50% 증가 시킨다. (ArrayDeque도 50%씩 증가한다)
-   Vector는 Enumeration 인터페이스로 요소를 순회하는 반면, ArrayList는 Iterator 인터페이스로 요소를 순회한다.

### Vector vs LinkedList

-   둘 다 List의 구현체이다. 하지만 Vetor는 Stack의 부모클래스이며 LinkedList는 Queue의 구현체이다.

### ArrayDeque vs Stack & LinkedList(Queue) vs PriorityQueue

-   ArrayDeque의 경우 동기화를 지원하지 않기 때문에 싱글 스레드 환경에서 Stack보다 빠르게 push, pop 한다.
-   ArrayDeque의 경우 배열 기반으로 구현되었기 때문에 LinkedList보다 빠르게 offer, poll 한다.  
    (시작과 끝을 나타내는 포인터 역할을 하는 필드를 갖는다는 공통점이 있다.)
-   따라서 Stack이나 Queue(또는 Deque) 자료구조를 사용한다면 ArrayDeque 클래스를 사용하는 것을 추천한다.
-   PriorityQueue는 힙을 기반으로 구현하여  
    요소의 값의 크기(Comparator 구현체에 의해 비교한다)에 따라서 나오는 순서가 정해진다.

### HashSet vs LinkedHashSet vs TreeSet
Map계열도 같은 비교점들이 있다.
-   해쉬 값을 기반으로 중복되지 않는 저장한다는 공통점이 있다.
-   HashSet의 경우 순서를 보장하지 않지만(일반적으로 Hash값을 기반으로 순서를 정한다),  
    LinkedHashSet의 경우 삽입순서를 유지한다. 대신 이를 위한 오버헤드가 존재한다.
-   HashSet 계열은 Hash 값을 기반으로 요소를 저장하는 반면,  
    TreeSet은 균형 이진 트리(정확히는 Red-Black Tree로 구현됨)로 요소를 저장한다.  
    따라서 요소의 값의 크기(Comparator 구현체에 의해 비교한다)에 따라서 순서가 정해진다.  
    이로인해 HashTable(O(1))과 Balanced Binary Tree(O(log n))에 의한 시간 복잡도를 갖는다.

### HashTable vs HashMap

-   HashTable은 동기화를 지원하는 반면 HashMap은 동기화를 지원하지 않기 때문에 이에 따른 성능 차이가 존재한다.
-   HashTable은 key와 value로 null값을 허용하지 않지만 HashMap은 이를 허용한다.
-   HashMap은 보조해시를 사용하기 때문에 HashTable에 비해 해시 충돌이 덜 발생할 수 있어 성능상 이점이 있다.
-   HashTable은 Enumeration 인터페이스로 요소를 순회하는 반면, HashMap은 Iterator 인터페이스로 요소를 순회한다.
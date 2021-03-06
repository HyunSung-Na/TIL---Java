# Effective Java item 64



### 객체는 인터페이스를 사용해 참조하라



아이템 51에서 매개변수 타입으로 클래스가 아니라 인터페이스를 사용하라고 했다. 이 조언을 "객체는 클랜스가 아닌 인터페이스로 참조하라" 고 까지 확장할 수 있다. **적합한 인터페이스만 있다면 매개변수뿐 아니라 반환값, 변수, 필드를 전부 인터페이스 타입으로 선언하라.** 객체의 실제 클래스를 사용해야 할 상황은 '오직' 생성자로 생성할 때뿐이다. 

예를 들어 다음은 Set 인터페이스를 구현한 LinkedHashSet 변수를 선언하는 올바른 모습이다.

```java
// 좋은 예. 인터페이스를 타입으로 사용했다.
Set<Son> sonSet = new LinkedHashSet<>();
```

다음은 좋지 않은 예다.



```java
// 나쁜 예. 클래스를 타입으로 사용했다!
LinkedHashSet<Son> sonSet = new LinkedHashSet<>();
```

**인터페이스를 타입으로 사용하는 습관을 길러두면 프로그램이 훨씬 유연해질 것이다.** 나중에 구현 클래스를 교체하고자 한다면 그저 새 클래스의 생성자(혹은 다른정적 팩터리)를 호출해주기만 하면 된다. 예컨대 처음 선언은 다음처럼 바뀔 것이다.

```java
Set<Son> sonSet = new HashSet<>();
```

이것으로 끝! 다른코드는 전혀 손대지 않고 새로 구현한 클래스로의 교체가 완료됐다.



- 단, 주의할 점이 하나 있다. 원래의 클래스가 인터페이스의 일반 규약 이외의 특별한 기능을 제공하며, 주변 코드가 이 기능에 기대어 동작한다면 새로운 클래스도 반드시 같은 기능을 제공해야 한다.
- 예컨대 첫 번째 선언의 주변 코드가 LinkedHashSet이 따르는 순서 정책을 가정하고 동작하는 상황에서 이를 HashSet으로 바꾸면 문제가  될수 있다. HashSet은 반복자의 순회 순서를 보장하지 않기 때문이다.
- 구현 타입을 바꾸려 하는 동기는 무엇일까? 원래 것보다 성능이 좋거나 멋진 신기능을 제공하기 때문일 수 있다. 예를 들어 HashMap을 참조하던 변수가 있다고 하자. 이를 EnumMap으로 바꾸면 속도가 빨라지고 수노히 순서도 키의 순서와 같아진다. 단 EnumMap은 키가 열거 타입일 때만 사용할 수 있다. 한편 키 타입과 상관없이 사용할 수 있는 LinkedHashMap으로 바꾼다면 성능은 비슷하게 유지하면서 순회 순서를 예측할 수 있다.



🚀 **적합한 인터페이스가 없다면 당연히 클래스로 참조해야 한다.** 

1. String과 BigInteger 같은 값 클래스가 그렇다. 값 클래스를 여러 가지로 구현될 수 있다고 생각하고 설계하는 일은 거의 없다.
2. 클래스 기반으로 작성된 프레임워크가 제공하는 객체들이다. 이런 경우라도 특정 구현 클래스보다는 (보통은 추상 클래스인) 기반 클래스를 사용해 참조하는 게 좋다. OutputStream등 java.io 패키지의 여러 클래스가 이 부류에 속한다.
3. 인터페이스에 없는 특별한 메서드를 제공하는 클래스들이다. 예를 들어 PriorityQueue 클래스는 Queue 인터페이스에는 없는 comparator 메서드를 제공한다. 클래스 타입을 직접 사용하는 경우는 이런 추가 메서드를 꼭 사용해야 하는 경우로 최소화해야 하며, 절대 남발하지 말아야 한다.



이상의 세 부류는 인터페이스 대신 클래스 타입을 사용해도 되는 예도 있음을 보여주기 위한 것일 뿐이므로 모든 상황을 다 설명하지는 못한다. 실전에서는 주어진 객체를 표현할 적절한 인터페이스가 있는지 찾아서 그 인터페이스로 참조하면 더 유연하고 세련된 프로그램을 만들 수 있다.



- **적합한 인터페이스가 없다면 클래스의 계층구조 중 필요한 기능을 만족하는 가장 덜 구체적인(상위의 ) 클래스를 타입으로 사용하자.**




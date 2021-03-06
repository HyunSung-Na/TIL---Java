# Effective Java item 83



### 지연 초기화는 신중히 사용하라



지연 초기화는 필드의 초기화 시점을 그 값이 처음 필요할 때까지 늦추는 기법이다. 그래서 값이 전혀 쓰이지 않으면 초기화도 결코 일어나지 않는다.

- 이 기법은 정적 필드와 인스턴스 필드 모두에 사용할 수 있다. 지연 초기화는 주로 최적화 용도로 쓰이지만, 클래스와 인스턴스 초기화 때 발생하는 위험한 순환 문제를 해결하는 효과도 있다.
- '필요할 때까지는 하지 말라', 지연 초기화는 양날의 검이다. 클래스 혹은 인스턴스 생성 시의 초기화 비용은 줄지만 그 대신 지연 초기화하는 필드에 접근하는 비용은 커진다.
- 그럼에도 지연 초기화가 필요할 때가 있다. 해당 클래스의 인스턴스 중 그 필드를 사용하는 인스턴스의 비율이 낮은 반면, 그 필드를 초기화하는 비용이 크다면 지연 초기화가 제 역할을 해줄 것이다. 하지만 안타깝게도 정말 그런지를 알 수 있는 유일한 방법은 지연 초기화 적용 전후의 성능을 측정해보는 것이다.
- 멀티스레드 환경에서는 지연 초기화를 하기가 까다롭다. 지연 초기화하는 필드를 둘 이상의 스레드가 공유한다면 어떤 형태로든 반드시 동기화해야 한다. 그렇지 않으면 심각한 버그로 이어질 것이다.



**대부분의 상황에서 일반적인 초기화가 지연 초기화보다 낫다.** 다음은 인스턴스 필드를 선언할 때 수행하는 일반적인 초기화의 모습이다. final 한정자를 사용했음에 주목하자.

```java
// 인스턴스 필드를 초기화하는 일반적인 방법

private final FieldType field = computeFieldValue();
```



**지원 초기화가 초기화 순환성을 깨뜨릴 것 같으면 synchronized를 단 접근자를 사용하자,** 이 방법이 가장 간단하고 명확한 대안이다.



```java
// 인스턴스 필드의 지연 초기화 - synchronized 접근자 방식

private FieldType field;

private synchronized FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    return field;
}
```

이상의 두 관용구(보통의 초기화와 synchronized 접근자를 사용한 지연 초기화) 는 정적 필드에도 똑같이 적용된다. 물론 필드와 접근자 메서드 선언에 static 한정자를 추가해야 한다.



**성능 때문에 정적 필드를 지연 초기화해야 한다면 지연 초기화 홀더 클래스 관용구를 사용하자.** 클래스는 클래스가 처음 쓰일 때 비로소 초기화된다는 특성을 이용한 관용구다.



```java
// 정적 필드용 지연 초기화 홀더 클래스 관용구

private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}

private static FieldType getField() { return FieldHolder.field; }
```

getField가 처음 호출되는 순간 FieldHolder.field가 처음 읽히면서, 비로소 FieldHolder 클래스 초기화를 촉발한다. 이 관용구의 멋진 점은 getField 메서드가 필드에 접근하면서 동기화를 전혀 하지 않으니 성능이 느려질 거리가 전혀 없다는 것이다.



- **성능 때문에 인스턴스 필드를 지연 초기화해야 한다면 이중검사 관용구를 사용하라.** 이 관용구는 초기화된 필드에 접근할 때의 동기화 비용을 없애준다. 필드의 값을 두 번 검사하는 방식으로, 한 번은 동기화 없이 검사하고, 두 번째는 동기화하여 검사한다. 두 번째 검사에서도 필드가 초기화되지 않았을 때만 필드를 초기화한다.
- 필드가 초기화된 후에는 동기화하지 않으므로 해당 필드는 반드시 volatile로 선언해야 한다.

```java
// 인스턴스 필드 지연 초기화용 이중검사 관용구

private FieldType getField() {
    FieldType result = field;
    
    if (result != null) { // 첫 번째 검사 (락 사용 안 함)
        return result;
    }
    
    synchronized(this) {
        if (field = null) // 두 번째 검사 (락 사용)
            field = computeFieldValue();
        return field;
    }
}
```

result라는 지역 변수가 필요한 이유는 무엇일까? 이 변수는 필드가 이미 초기화된 상황에서는 그 필드를 딱 한 번만 읽도록 보장하는 역할을 한다.

- 이중 검사에는 언급해둘 만한 변종이 두 가지 있다. 이따금 반복해서 초기화해도 상관없는 인스턴스 필드를 지연 초기화해야 할 때가 있는데, 이런 경우라면 이중검사에서 두 번째 검사를 생략할 수 있다. 이 변종의 이름은 자연히 단일검사 관용구가 된다. 필드는 여전히 volatile로 선언했음을 확인하자.

```java
// 단일 검사 관용구 - 초기화가 중복해서 일어날 수 있다.

private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null)
        field = result = computeFieldValue();
    return result;
}
```



이번 아이템에서 이야기한 모든 초기화 기법은 기본 타입 필드와 객체 참조 필드 모두에 적용할 수 있다. 이중검사와 단일검사 관용구를 수치 기본 타입 필드에 적용한다면 필드의 값을 null 대신 0과 비교하면 된다.



> 핵심 정리
>
> 대부분의 필드는 지연시키지 말고 곧바로 초기화해야 한다. 성능 때문에 혹은 위험한 초기화 순환을 막기 위해 꼭 지연 초기화를 써야 한다면 올바른지연 초기화 기법을 사용하자. 인스턴스 필드에는 이중검사 관용구를, 정적 필드에는 지연 초기화 홀더 클래스 관용구를 사용하자. 반복해 초기화해도 괜찮은 인스턴스 필드에는 단일검사 관용구도 고려 대상이다.
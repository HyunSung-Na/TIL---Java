# Effective Java item 79



### 과도한 동기화는 피하라



아이템 78에서 충분하지 못한 동기화의 피해를 다뤘다면, 이번 아이템에서는 반대 상황을 다룬다. 과도한 동기화는 성능을 떨어뜨리고, 교착상태에 빠뜨리고, 심지어 예측할 수 없는 동작을 낳기도 한다.



**응답 불가와 안전 실패를 피하려면 동기화 메서드나 동기화 블록 안에서는 제어를 절대로 클라이언트에 양도하면 안된다.** 

- 예를 들어 동기화된 영역 안에서는 재정의할 수 있는 메서드는 호출하면 안 되며, 클라이언트가 넘겨준 함수 객체를 호출해서는 안 된다.
- 동기화된 영역을 포함한 클래스 관점에서는 이런 메서드는 모두 바깥 세상에서 온 외계인이다. 그 메서드가 무슨 일을 할지 알지 못하며 통제할 수도 없다는 뜻이다. 외계인 메서드가 하는 일에 따라 동기화된 영역은 예외를 일으키거나, 교착상태에 빠지거나, 데이터를 훼손할 수도 있다.



구체적인 예를 보자. 다음은 Set을 감싼 래퍼 클래스이고 클아이언트는 집합에 원소가 추가되면 알림을 받을 수 있다. 바로 관찰자 패턴이다.

```java
// 잘못된 코드, 동기화 블록 안에서 외계인 메서드를 호출한다.

public class ObservableSet<E> extends ForwardingSet<E> {
    public ObservableSet(Set<E> set) { super(set); }
    
    private final List<SetObserver<E>> observers
        = new ArrayList<>();
    
    public void addObserver(SetObsever<E> observer) {
        synchronized(observers) {
            observers.add(observer);
        }
    }
    
    public boolean removeObserver(SetObserver<E> observer) {
        synchronized(observers) {
            observers.remove(observer);
        }
    }
    
    private void notifyElementAdded(E element) {
        synchronized(observers) {
            for (SetObserver<E> observer : observers)
                observer.added(this, element);
        }
    }
    
    @Override
    public boolean add(E element) {
        boolean added = super.add(element);
        
        if (added)
            notifyElementAdded(element);
        return added;
    }
    
    @Override
    public boolean addAll(Collection<? extends E> c) {
        boolean result = false;
        for (E element : c)
    // 왼쪽의 피연산자를 오른쪽의 피연산자와 비트 OR 연산한 후, 그 결괏값을 왼쪽의 피연산자에 대입함.
			result |= add(element); // notifyElementAdded를 호출한다.
        return result;
    }
}
```

관찰자들은 addObserver와 removeObserver 메서드를 호출해 구독을 신청하거나 해지한다. 두 경우 모두 다음 콜백 인터페이스의 인스턴스를 메서드에 건넨다.



```java
@FunctionalInterface
public interface SetObserver<E> {
    // ObservableSet에 원소가 더해지면 호출된다.
    
    void added(ObservableSet<E> set, E element);
}
```

이 인터페이스는 구조적으로 BiConsumer<ObservableSet<E>, E>와 똑같다. 그럼에도 커스텀 함수형 인터페이스를 정의한 이유는 이름이 더 직관적이고 다중콜백을 지원하도록 확정할 수 있어서다. 하지만 BiConsumer를 그대로 사용했더라도 별 무리는 없었을 것이다.

- 눈으로 보기에 ObservableSet은 잘 동작할 것 같다. 예컨대 다음 프로그램은 0부터 99까지를 출력한다.



```java
public static void main(String[] args) {
    observableSet<Integer> set =
        new ObservableSet<>(new HashSet<>());
    
    set.addObserver((s, e) -> System.out.println(e));
    
    for (int i = 0; i < 100; i++)
        set.add(i);
}
```

이제 조금 흥미진진한 시도를 해보자. 평상시에는 앞서와 같이 집합에 추가된 정숫값을 출력하다가, 그 값이 23이면 자기 자신을 제거(구독해지)하는 관찰자를 추가해보자.



```java
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23)
            s.removeObserver(this);
    }
});
```

> 람다를 사용한 이전 코드와 달리 익명 클래스를 사용했다. s.removeObserver 메서드에 함수 객체 자신을 넘겨야 하기 때문이다. 람다는 자신을 참조할 수단이 없다.

이 프로그램은 0부터 23까지 출력한 후 관찰자 자신을 구독해지한 다음 조용히 종료할 것이다. 그런데 실제로 실행해 보면 그렇게 진행되지 않는다!

이 프로그램은 23까지 출력한 다음 ConcurrentModificationException을 던진다. 관찰자의 added 메서드 호출이 일어난 시점이 notifyElementAdded가 관찰자들의 리스트를 순회하는 도중이기 때문이다. 

added 메서드는 ObservableSet의 removeObserver메서드를 호출하고, 이 메서드는 다시 observers.remove 메서드를 호출한다. 여기서 문제가 발생한다. 리스트에서 원소를 제거하려고 하는데 마침 이 리스트를 순회하는 도중이다. 즉, 허용되지 않은 동작이다. notifyElementAdded 메서드에서 수행하는 순회는 동기화 블록 안에 있으므로 동시 수정이 일어나지 않도록 보장하지만, 정작 자신이 콜백을 거쳐 되돌아와 수정하는 것까지 막지는 못한다.



- 이번에는 이상한 것을 시도해보자. 구독해지를 하는 관찰자를 작성하는데, removeObserver를 직접 호출하지 않고 실행자 서비스를 사용해 다른스레드한테 부탁할 것이다.

```java
// 쓸데없이 백그라운드 스레드를 사용하는 관찰자

set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23) {
            ExecutorService exec = 
                Excutors.newSingleThreadExecutor();
            try {
                exec.submit(() -> s.removeObserver(this)).get();
            } catch (ExecutionException | InterruptedException ex) {
                throw new AssertionError(ex);
            } finally {
                exec.shutdown();
            }
        }
    }
});
```

> 이 프로그램은 catch 구문 하나에서 두 가지 예외를 잡고 있다. 다중 캐치라고도 하는 이 기능은 자바 7부터 지원한다. 이 기법은 똑같이 처리해야 하는 예외가 여러 개일때 프로그램 크기를 줄이고 코드 가독성을 크게 개선해준다.



이 프로그램을 실행하면 예외는 나지 않지만 교착상태에 빠진다. 백그라운드 스레드가 s.removeObserver를 호출하면 관찰자를 잠그려 시도하지만 락을 얻을 수 없다. 이미 메인 스레드가 락을 가지고 있기 때문이다. 그와 동시에 메인 스레드는 백그라운드 스레드가 관찰자를 제거하기만을 기다리는 중이다. 교착상태가 걸린 것이다.



이렇게 교착상태가 걸릴 뿐만아니라 일관성이 깨지고 불변식이 깨지는 상황도 발생할 수 있다. 이 경우엔 더 최악이다.



- 다행이 이런 문제는 대부분 어렵지 않게 해결할 수 있다. 외계인 메서드 호출을 동기화 블록 바깥으로 옮기면 된다. notifyElementAdded 메서드에서라면 관찰자 리스트를 복사해 쓰면 락 없이도 안전하게 순회할 수 있다. 이 방식을 적용하면 앞서의 두 예제에서 예외 발생과 교착상태 증상이 사라진다.

```java
// 외계인 메서드를 동기화 블록 바깥으로 옮겼다.

private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    
    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```

사실 외계인 메서드 호출을 동기화 블록 바깥으로 옮기는 더 나은 방법이 있다. 자바의 동시성 컬렉션 라이브러리의 CopyOnWriteArrayList가 정확히 이 목적으로 특별히 설계된 것이다. 이름이 말해주듯 ArrayList를 구현한 클래스로 내부를 변경하는 작업은 항상 깨끗한 복사본을 만들어 수행하도록 구현했다.



```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    private static final long serialVersionUID = 8673264195747942595L;

    final transient Object lock = new Object();

    private transient volatile Object[] array;

    final Object[] getArray() {
        return array;
    }

    final void setArray(Object[] a) {
        array = a;
    }

    /**
     * Creates an empty list.
     */
    public CopyOnWriteArrayList() {
        setArray(new Object[0]);
    }

    /**
     * Creates a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection of initially held elements
     * @throws NullPointerException if the specified collection is null
     */
    public CopyOnWriteArrayList(Collection<? extends E> c) {
        Object[] es;
        if (c.getClass() == CopyOnWriteArrayList.class)
            es = ((CopyOnWriteArrayList<?>)c).getArray();
        else {
            es = c.toArray();
            // defend against c.toArray (incorrectly) not returning Object[]
            // (see e.g. https://bugs.openjdk.java.net/browse/JDK-6260652)
            if (es.getClass() != Object[].class)
                es = Arrays.copyOf(es, es.length, Object[].class);
        }
        setArray(es);
    }

    /**
     * Creates a list holding a copy of the given array.
     *
     * @param toCopyIn the array (a copy of this array is used as the
     *        internal array)
     * @throws NullPointerException if the specified array is null
     */
    public CopyOnWriteArrayList(E[] toCopyIn) {
        setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
    }

```

다른 용도로 쓰인다면 끔찍히 느리겠지만, 수정할 일은 드물고 순회만 빈번히 일어나는 관찰자 리스트 용도로는 최적이다.



- 동기화 영역 바깥에서 호출되는 외계인 메서드를 열린 호출이라 한다. 얼마나 오래 실행될지 알 수 없는데, 동기화 영역 안에서 호출된다면 그동안 다른 스레드는 보호된 자원을 사용하지 못하고 대기해야만 한다. 따라서 열린 호출은 실패 방지 효과외에도 동시성 효율을 크게 개선해 준다.
- **기본 규칙은 동기화 영역에서는 가능한 일을 적게 하는 것이다.** 락을 얻고, 공유 데이터를 검사하고, 필요하면 수정하고, 락을 놓는다. 오래 걸리는 작업이라면 아이템 78의 지치을 어기지 않으면서 동기화 영역 바깥으로 옮기는 방법을 찾아보자.





✔ 지금까지 정확성에 관해 이야기했으니 이제 성능 측면도 살펴보자.



- 자바의 동기화 비용은 빠르게 낮아져 왔지만, 과도한 동기화를 피하는 일은 중요하다. 과도한 동기화가 초래하는 진짜 비용은 락을 얻는데 드는 CPU시간이 아니다. 바로 경쟁하느라 낭비하는 시간, 즉 병렬로 실행할 기회를 잃고, 모든 코어가 메모리를 일관되게 보기 위한 지연시간이 진짜 비용이다.
- 가상 머신의 코드 최적화를 제한한다는 점도 과도한 동기화의 또 다른 숨은 비용이다.



⭐ 가변 클래스를 작성하려거든 다음 두 선택지 중 하나를 따르자.

1. 동기화를 전혀 하지 말고, 그 클래스를 동시에 사용해야 하는 클래스가 외부에서 알아서 동기화하게 하자.
2. 동기화를 내부에서 수행해 스레드 안전한 클래스로 만들자. 단 클라이언트가 외부에서 객체 전체에 락을 거는 것보다 동시성을 월등히 개선할 수 있을 때만 두 번째 방법을 선택해야 한다. 



- java.util(이제 구식이 된 Vector와 Hashtable을 제외하고) 첫 번째 방식을 취했고, java.util.concurrent는 두 번째 방식을 취했다.
- StringBuffer 인스턴스는 거의 항상 단일 스레드에서 쓰였음에도 내부적으로 동기화를 수행했다. 뒤늦게 StringBuilder가 등장한 이유기도 하다. (Builder는 동기화하지 않은 Buffer다) 비슷한 이유로 java.util.Random은 동기화하지 않는 버전인 java.util.concurrent.ThreadLocalRandom으로 대체되었다.



> 핵심정리
>
> 교착상태와 데이터 훼손을 피하려면 동기화 영역 안에서 외계인 메서드를 절대 호출하지 말자. 일반화해 이야기 하면, 동기화 영역 안에서의 작업은 최소한으로 줄이자. 가변 클래스를 설계할 때는 스스로 동기화해야 할지 고민하자. 멀티코어 세상인 지금은 과도한 동기화를 피하는 게 과거 어느 때보다 중요하다. 합당한 이유가 있을 때만 내부에서 동기화하고, 동기화했는지 여부를 문서에 명확히 밝히자.


# Effective Java item 21



### 인터페이스는 구현하는 쪽을 생각해 설계하라



자바 8 전에는 기존 구현체를 깨뜨리지 않고는 인터페이스에 메서드를 추가할 방법이 없었다. 인터페이스에 메서드를 추가하면 보통은 컴파일 오류가 나는데, 추가된 메서드가 우연히 기존 구현체에 이미 존재할 가능성은 아주 낮기 때문이다. 잡자 8에 와서 기존 인터페이스에 메서드를 추가할 수 있도록 디폴트 메서드를 소개했지만 위험이 완전히 사라진 것은 아니다.



:notebook_with_decorative_cover: 디폴트 메서드를 선언하면, 그 인터페이스를 구현한 후 디폴트 메서드를 재정의하지 않은 모든 클래스에서 디	 폴트 구현이 쓰이게 된다. 이처럼 자바에도 기존 이너페이스에 메서드를 추가하는 길이 열렸지만 모든 기존 구	현체들과 매끄럽게 연동되리라는 보장은 없다.

:notebook_with_decorative_cover: 자바 8에서는 핵심 컬렉션 인터페이스들에 다수의 디폴트 메서드가 추가되었다. 주로 람다를 활용하기 위해서	다. 자바 라이브러리의 디폴트 메서드는 코드 품질이 높고 범용적이라 대부분 상황에서 잘 작동한다.



- **생각할 수 있는 모든 상황에서 불변식을 해치지 않는 디폴트 메서드를 작성하기란 어려운 법이다.**



```java
// 자바 8의 Collection 인터페이스에 추가된 디폴트 메서드

default boolean removeIf(Predicate<? super E> filter) {
    
    Objects.requireNonNull(filter);
    
    boolean result = false;
    
    for (Iterator<E> it = iterator(); it.hasNext();) {
        if (filter.test(it.next())) {
        it.remove();
        result = true;
        }
    }
    return result;
}
```

이 코드보다 더 범용적으로 구현하기도 어렵겠지만, 그렇다고 해서 현존하는 모든 Collection 구현체와 잘 어우러지는 것은 아니다.



- 대표적인 예가 org.apache.commons.collections4.collection.SynchronizedCollection 이다. 아파치 커먼즈 라이브러리의 이 클래스는 java.util의 Collections.synchronizedCollection 정적 팩토리 메서드가 반환하는 클래스와 비슷하다. 아파치 버전은 클라이언트가 제공한 객체로 락을 거는 능력을 추가로 제공한다. 즉 모든 메서드에서 주어진 락 객체로 동기화한 후 내부 컬렉션 객체에 기능을 위임하는 래퍼 클래스다.
- 아파치의 SynchronizedCollection 클래스는 지금도 활발히 관리되고 있지만, 이 책을 쓰는 시점엔 removeIf 메서드를 재정의하지 않고 있다. 이 클래스를 자바 8과 함께 사용한다면 (그래서 removeIf의 디폴트 구현을 물려받게 된다면), 자신이 한 약속을 더 이상 지키지 못하게 된다. 다시 말해 모든 메서드 호출을 알아서 동기화 해주지 못한다.
- removeIf의 구현은 동기화에 관해 아무것도 모르므로 락 객체를 사용할 수 없다. 따라서 SynchronizedCollection 인스턴스를 여러 스레드가 공유하는 환경에서 한 스레드가 removeIf를 호출하면 ConcurrentModificationException 이 발생하거나 다른예기치 못한 결과로 이어 질 수 있다.
- 자바 플랫폼 라이브러리에서도 이런 문제를 예방하기 위해 일련의 조치를 취했다. 예를 들어 구현한 인터페이스의 디폴트 메서드를 재정의하고, 다른메서드에서는 디폴트 메서드를 호출하기 전에 필요한 작업을 수행하도록 했다. 예컨대 SynchronizedCollection 이 반환하는 package-private 클래스들은 removeIf를 재정의하고, 이를 호출하는 다른메서드들은 디폴트 구현을 호출하기 전에 동기화를 하도록 했다. 하지만 자바 플랫폼에 속하지 않은 제 3의 기존 컬렉션 구현체들은 이런 언어 차원의 인터페이스 변화에 발맞춰 수정될 기회가 없었으며, 그중 일부는 여전히 수정되지 않고 있다.



##### 디폴트 메서드는 (컴파일에 성공하더라도) 기존 구현체에 런타임 오류를 일으킬 수 있다.

흔한 일은 아니지만, 나에게는 일어나지 않으리라는 보장도 없다. 자바 8은 컬렉션 인터페이스에 꽤 많은 디폴트 메서드를 추가했고, 그 결과 기존에 짜여진 많은 자바 코드가 영향을 받은 것으로 알려졌다.

:notebook_with_decorative_cover: 기존 인터페이스에 디폴트 메서드로 새 메서드를 추가하는 일은 꼭 필요한 경우가 아니면 피해야 한다. 추가하	려는 디폴트 메서드가 기존 구현체들과 충돌하지는 않을지 심사숙고해야 함도 당연하다. 반면, 새로운 인터페이	스를 만드는 경우라면 표준적인 메서드 구현을 제공하는 데 아주 유용한 수단이며, 그 인터페이스를 더 쉽게 구	현해 활용할 수 있게끔 해준다.

:notebook_with_decorative_cover: **핵심은 명백하다. 디폴트 메서드라는 도구가 생겼더라도 인터페이스를 설계할 때는 여전히 세심한 주의를 기	울여야 한다.** 디폴트 메서드로 기존 인터페이스에 새로운 메서드를 추가하면 커다란 위험도 딸려온다. 인터페	이스에 내재된 작은 결함도 사용자 입장에서는 짜증나는 일인데, 심각하게 잘못된 인터페이스라면 이를 포함한 	API에 어떤 재앙을 몰고 올지 알 수 없다.

:notebook_with_decorative_cover: 새로운 인터페이스라면 릴리스 전에 반드시 테스트를 거쳐야 한다. 수많은 개발자가 그 인터페이스를 나름의 방식으로 구현할 것이니, 여러분도 서로 다른 방식으로 최소한 세 가지는 구현해봐야 한다.



**인터페이스를 릴리스한 후라도 결함을 수정하는 게 가능한 경우도 있겠지만, 절대 그 가능성에 기대서는 안 된다.**



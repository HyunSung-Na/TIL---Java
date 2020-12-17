# Effective Java item 17



### 변경 가능성을 최소화하라



불변 클래스란 간단히 말해 그 인스턴스의 내부 값을 수정할 수 없는 클래스다. 불변 인스턴ㅌ스에 간직된 정보는 고정되어 객체가 파괴되는 순간까지 절대 달라지지 않는다. 자바 플랫폼 라이브러리에도 다양한 불변 클래스가 있다. String, BigInteger, BigDecimal이 여기 속한다.

**불변 클래스는 가변 클래스보다 설계하고 구현하고 사용하기 쉬우며, 오류가 생길 여지도 적고 훨씬 안전하다**



클래스를 불변으로 만들려면 다음 다섯 가지 규칙을 따르면 된다.

- 객체의 상태를 변경하는 메서드를 제공하지 않는다.
- 클래스를 확장할 수 없도록 한다.
- 모든 필드를 final로 선언한다.
- 모든 필드를 private로 선언한다.
- 자신 외에는 내부의 가변 컴포넌트에 접근할 수 없도록 한다.
  - 클래스에 가변 객체를 참조하는 필드가 하나라도 있다면 클라이언트에서 그 객체의 참조를 얻을 수 없도록 해야 한다. 이런 필드는 절대 클라이언트가 제공한 객체 참조를 가리키게 해서는 안 되며, 접근자 메서드가 그 필드를 반환해서도 안된다. 생성자, 접근자, readObject 메서드 모두에서 방어적 복사를 수행하라.



지금까지의 선보인 예제 클래스는 대부분 불변이다. 다음의 복잡한 예를 보자

```java
// 불변 복소수 클래스

public final class Complex {
    
    private final double re;
    private final double im;
    
    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }
    
    public double realPart() {return re;}
    public double imaginaryPart() {return im;}
    
    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }
    
    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }
    
    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                          re * c.im + im * c.re);
    }
    
    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                          (im * c.re - re * c.im) / tmp);
    }
    
    @Override
    public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof Complex))
            return false;
        Complex c = (Complex) o;
        
       	// == 대신 compare를 사용하는 이유는 63쪽을 확인하라
        return Double.compare(c.re, re) == 0
            && Double.compare(c.im, im) == 0;
    }
    
    @Override
    public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashcode(im);
    }
    
    @Override
    public String toString() {
        return "(" + re + " + " + im  + "i)";
    }
}
```

 이 클래스는 복소수를 표현한다. Object의 메서드 몇 개를 재정의했고, 실수부와 허수부 값을 반환하는 접근자 메서드와 사칙연산 메서드를 정의했다. 이 사칙연산 메서드들이 인스턴스 자신은 수정하지 않고 새로운 Complex 인스턴스를 만들어 반환하는 모습에 주목하자.

**이처럼 피연산자에 함수를 적용해 그 결과를 반환하지만, 피연산자 자체는 그대로인 프로그래밍 패턴을 함수형 프로그래밍이라 한다.**

이와 달리 절차적 혹은 명령형 프로그래밍에서는 메서드에서 피연산자인 자신을 수정해 자신의 상태가 변하게 된다. 또한 메서드 이름으로 동사 대신 전치사를 사용한 점에도 주목하자. 이는 해당 메서드가 객체의 값을 변경하지 않는다는 점을 강조하려는 의도다.

**불변 객체는 단순하다.** 불변 객체는 생성된 시점의 상태를 파괴될 때까지 그대로 간직한다. 모든 생성자가 클래스 불변식을 보장한다면 그 클래스를 사용하는 프로그래머가 다른 노력을 들이지 않더라도 영원히 불변으로 남는다.



- **불변 객체는 근본적으로 스레드 안전하여 따로 동기화할 필요 없다** 여러 스레드가 동시에 사용해도 절대 훼손되지 않는다. 사실 클래스를 스레드 안전하게 만드는 가장 쉬운 방법이기도 하다. 불변 객체에 대해서는 그 어떤 스레드도 다른스레드에 영향을 줄 수 없으니 **불변 객체는 안심하고 공유할 수 있다.** 따라서 불변 클래스라면 한번 만든 인스턴스를 최대한 재활용하기를 권한다. 가장 쉬운 재활용 방법은 자주 쓰이는 값들을 상수로 제공하는 것이다.

```java
public static final Complex ZERO = new Complex(0, 0);
public static final Complex ONE = new Complex(1, 0);
public static final Complex I = new Complex(0, 1);
```

불변 클래스는 자주 사용되는 인스턴스를 캐싱하여 같은 인스턴스를 중복 생성하지 않게 해주는 정적 팩토리를 제공할 수 있다. 이런 정적 팩토리를 사용하면 여러 클라이언트가 인스턴스를 공유하여 메모리 사용량과 가비지 컬렉션 비용이 줄어든다.

불변 객체를 자유롭게 공유할 수 있다는 점은 방어적 복사도 필요 없다는 결론으로 자연스럽게 이어진다.



- 불변 객체는 자유롭게 공유할 수 있음은 물론, 불변 객체끼리는 내부 데이터를 공유할 수 있다.
  - 예컨대 BigInteger 클래스는 내부에서 값의 부호와 크기를 따로 표현한다. 부호에는 int 변수를, 크기에는 int 배열을 사용하는 것이다. 한편 negate 메서드는 크기가 같고 부호만 반대인 새로운 BigInteger를 생성하는데, 이때 배열은 비록 가변이지만 복사하지 않고 원본 인스턴스와 공유해도 된다. 그 결과 새로 만든 BigInteger 인스턴스도 원본 인스턴스가 가리키는 내부 배열을 그대로 가리킨다.
- 객체를 만들 때 다른불변 객체들을 구성요소로 사용하면 이점이 많다. 값이 바뀌지 않는 구성요소들로 이뤄진 객체라면 그 구조가 아무리 복잡하더라도 불변식을 유지하기 훨씬 수월하기 때문이다.
- 불변 객체는 그 자체로 실패 원자성( 메서드에서 예외가 발생한 후에도 그 객체는 여전히 유효한 상태여야 한다. ) 을 제공한다. 상태가 절대 변하지 않으니 잠깐이라도 불일치 상태에 빠질 가능성이 없다.
- 불변 클래스에도 단점은 있다. 값이 다르면 반드시 독립된 객체로 만들어야 한다는 것이다. 값의 가짓수가 많다면 이들을 모두 만드는 데 큰 비용을 치러야 한다.
  - 이 문제에 대처하는 방법은 두가지다.
  - 첫번째는 흔히 쓰일 다단계 연산들을 예측하여 기본 기능으로 제공하는 방식이다. 예컨대 BigInteger는 모듈러 지수 같은 다단계 연산 속도를 높여주는 가변 동반 클래스를 package-private로 두고 있다.
  - 두번째는 클라이언트들이 원하는 복잡한 연산들을 정확히 예측할 수 있다면 package-private의 가변 동반 클래스만으로 충분하다. 그렇지 않다면 이 클래스를 public으로 제공하는 게 최선이다. 자바에서 이에 해당하는 예가 바로 String 클래스다. 그렇다면 String의 가변 동반 클래스는? 바로 StringBuilder다.



이상으로 불변 클래스를 만드는 기본적인 방법과 불변 클래스의 장단점을 보았다. 그 다음으로 불변 클래스를 만드는 또 다른 설계 방법 몇 가지 알아볼 차례다.

- 클래스가 불변임을 보장하려면 자신을 상속하지 못하게 해아한다. 자신을 상속하지 못하게 하는 가장 쉬운 방법은 final 클래스로 선언하는 것이지만 더 유연한 방법으로 모든 생성자를 private 혹은 package-private로 만들고 public 정적 팩토리를 제공하는 방법이다. 다음의 예를 보자



```java
// 생성자 대신 정적 팩토리를 사용한 불변 클래스

public class Complex {
    private final double re;
    private final double im;
    
    private Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }
    
    public static Complex valueOf(double re, double im) {
        return new Complex(re, im);
    }
    
    ...// 나머지 코드 생략
}
```

이 방법이 최선일 때가 많다. 바깥에서 볼 수 없는 private 구현 클래스를 원하는 만큼 만들어 활용할 수 있으니 훨씬 유연하다. 패키지 바깥의 클라이언트에서 바라본 이 불변 객체는 사실상 final이다. public 이나 protected 생성자가 없으니 다른 패키지에서는 이 클래스를 확장하는 게 불가능하기 때문이다. 정적 팩토리 방식은 다수의 구현 클래스를 활용한 유연성을 제공하고, 이에 더해 다음 릴리스에서 객체 캐싱 기능을 추가해 성능을 끌어올릴 수도 있다.



- BigInteger와 BigDecimal을 설계할 당시엔 불변 객체가 사실상 final이어야 한다는 생각이 널리 퍼지지 않았다. 그래서 이 두 클래스의 메서드들은 모두 재정의할 수 있게 설계되었고, 안타깝게도 하위 호환성이 발목을 잡아 지금까지도 이 문제를 고치지 못했다. 이 값들이 불변이어야 클래스의 보안을 지킬 수 있다면 인수로 받은 객체가 '진짜' BigInteger인지 반드시 확인해야 한다. 다시 말해 신뢰할 수 없는 하위 클래스의 인스턴스라고 확인되면, 이 인수들은 가변이라 가정하고 방어적으로 복사해 사용해야 한다.

```java
public static BigInteger safeInstance(BigInteger val) {
    return val.getClass() == BigInteger.class ?
        val : new BigInteger(val.toByteArray());
}
```

이번 아이템 초입에서 나열한 불변 클래스의 규칙 목록에 따르면 모든 필드가 final이고 어떤 메서드도 그 객체를 수정할 수 없어야 한다.

- 이 규칙은 좀 과한 감이 있어서, 성능을 위해 다음처럼 살짝 완화할 수 있다. "어떤 메서드도 객체의 상태 중 외부에 비치는 값을 변경할 수 없다.", 어떤 불변 클래스는 계산 비용이 큰 값을 나중에 계산하여 final이 아닌 필드에 캐시해놓기도 한다. 똑같은 값을 다시 요청하면  캐시해둔 값을 반환하여 계산 비용을 절감하는 것이다.



예컨대 PhoneNumber의 hashCode 메서드는 처음 불렸을 때 해시 값을 계산해 캐시한다. 지연 초기화의 예이기도 한 이 기법을 String도 사용한다.



> 직렬화할 때는 추가로 주의할 점이 있다. Serializable을 구현하는 불변 클래스의 내부에 가변 객체를 참조하는 필드가 있다면 readObject나 readResolve 메서드를 반드시 제공하거나, ObjectOutputStream.writeUnshared와 ObjectInputStream.readUnshared 메서드를 사용해야 한다. 플랫폼이 제공하는 기본 직렬화 방법이면 충분하더라도 말이다. 그렇지 않으면 공격자가 이 클래스로부터 가변 인스턴스를 만들어낼 수 있다.(이 주제는 아이템 88에서 다룬다.)



- 정리해 보자. getter가 있다고 해서 무조건 setter를 만들지는 말자. 클래스는 꼭 필요한 경우가 아니라면 불변이어야 한다. 불변 클래스는 장점이 많으며 단점이라곤 특정 상황에서의 잠재적 성능 저하뿐이다.
- 한편 모든 클래스를 불변으로 만들 수는 없다. **불변으로 만들 수 없는 클래스라도 변경할 수 있는 부분을 최소한으로 줄이자.** 꼭 변경해야 할 필드를 뺀 나머지 모두를 final로 선언하자. **다른합당한 이유가 없다면 모든 필드는 private final이어야 한다.**
- 생성자는 불변식 설정이 모두 완료된, 초기화가 완벽히 끝난 상태의 객체를 생성해야 한다. 확실한 이유가 없다면 생성자와 정적 팩토리 외에는 그 어떤 초기화 메서드도 public으로 제공해서는 안된다. 객체를 재활용할 목적으로 상태를 다시 초기화하는 메서드도 안된다. 복잡성만 커지고 성능 이점은 거의 없다.
# Effective Java item 84



### 프로그램의 동작을 스레드 스케줄러에 기대지 말라



여러 스레드가 실행 중이라면 운영체제의 스레드 스케줄러가 어떤 스레드를 얼마나 오래 실행할지 정한다. 정상적인 운영체제라면 이 작업을 공정하게 수행하지만 구체적인 스케줄링 정책은 운영체제마다 다를 수 있다. 따라서 잘 작성된 프로그램이라면 이 정책에 좌지우지돼서는 안된다.

- **정확성이나 성능이 스레드 스케줄러에 따라 달라지는 프로그램이라면 다른 플랫폼에 이식하기 어렵다.** 
- 견고하고 빠릿하고 이식성 좋은 프로그램을 작성하는 가장 좋은 방법은 실행 가능한 스레드의 평균적인 수를 프로세서 수보다 지나치게 많아지지 않도록 하는 것이다. 그래야 스레드 스케줄러가 고민할 거리가 줄어든다.
- 실행가능한 스레드 수를 적게 유지하는 주요 기법은 각 스레드가 무언가 유용한 작업을 완료한 후에는 다음 일거리가 생길 때까지 대기하도록 하는 것이다. **스레드는 당장 처리해야 할 작업이 없다면 실행돼서는 안 된다.** 
- 스레드는 절대 바쁜 대기 상태가 되면 안 된다. 공유 객체의 상태가 바뀔 때까지 쉬지 않고 검사해서는 안 된다는 뜻이다. 바쁜 대기는 스레드 스케줄러의 변덕에 취약할 뿐 아니라, 프로세서에 큰 부담을 주어 다른유용한 작업이 실행될 기회를 박탈한다. 다음 코드를 보자

```java
// 끔찍한 CountDownLatch 구현 - 바쁜 대기 버전!

public class SlowCountDownLatch {
    private int count;
    
    public SlowCountDownLatch(int count) {
        if (count < 0)
            throw new IllegalArgumentException(count + " < 0");
        this.count = count;
    }
    
    public void await() {
        while (true) {
            synchronized(this) {
                if (count == 0)
                    return;
            }
        }
    }
    
    public synchronized void countDown() {
        if (count != 0)
            count--;
    }
}
```

래치를 기다리는 스레드를 1000개 만들어 자바의 CountDownLatch와 비교해보니 약 10배가 느렸다. 하나 이상의 스레드가 필요도 없이 실행 가능한 상태인 시스템은 흔하게 볼 수 있다. 이런 시스템은 성능과 이식성이 떨어질 수 있다.



- 특정 스레드가 다른스레드들과 비교해 CPU 시간을 충분히 얻지 못해서 간신히 돌아가는 프로그램을 보더라도 **Thread.yield를 써서 문제를 고쳐보려는 유혹을 떨쳐내자.** 증상이 어느정도는 호전될 수도 있지만 이식성은 그렇지 않을 것이다. **Thread.yield는 테스트할 수단도 없다.**

[Thread.yield 참고](https://codingdog.tistory.com/entry/java-yield-%EB%A9%94%EC%84%9C%EB%93%9C-%EC%96%91%EB%B3%B4%ED%95%9C%EB%8B%A4%EB%8A%94-%ED%9E%8C%ED%8A%B8%EB%A5%BC-%EC%A4%80%EB%8B%A4)



- 이런 상황에서 스레드 우선순위를 조절하는 방법도 있지만, 역시 위험이 따른다. 스레드 우선순위는 자바에서 이식성이 가장 나쁜 특성에 속한다.



> 핵심 정리
>
> 프로그램의 동작을 스레드 스케줄러에 기대지 말자. 견고성과 이식성을 모두 해치는 행위다. 같은 이유로, Thread.yield와 스레드 우선순위에 의존해서도 안 된다. 이 기능들은 스레드 스케줄러에 제공하는 힌트일 뿐이다. 스레드 우선순위는 이미 잘 동작하는 프로그램의 서비스 품질을 높이기 위해 드물게 쓰일 수는 있지만, 간신히 동작하는 프로그램을 '고치는 용도'로 사용해서는 절대 안된다.
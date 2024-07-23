# 싱글톤 패턴

**목적**

* 오직 하나의 인스턴스만 제공해야 할 때 사용한다.

언제일까?

* RestTemplate, JDBC 같은 외부 커넥션 풀을 만드는 인스턴스
* 전역 캐시 데이터
* 로그 기록 담당 클래스
* 이런 클래스들이 요청마다 생성된다면 매번 불필요한 객체 생성 과정이 발생하거나 메모리를 굉장히 비효율적이게 사용하게 될 수 있다.

**특징**

* new를 사용해서 생성할 수 있게 하면 안된다.
* 싱글톤 인스턴스에 접근하게 해주는 클래스가 필요하다.

```
public class WaitingQueue {
    private Queue<String> queue = new PriorityQueue<>();
​
    private static WaitingQueue instance;
​
  //처음에만 생성하고 두번째 요청부터는 생성된 싱글톤 인스턴스 반환
    public static WaitingQueue getInstance() {
        if ( instance == null ) {
            instance = new WaitingQueue();
        }
        return instance;
    }
​
  //new 생성 방어
    private WaitingQueue() {
​
    }
}
```

```
public class SingleTonePattern {
    public static void main(String[] args) {
        WaitingQueue queue1 = WaitingQueue.getInstance();
        WaitingQueue queue2 = WaitingQueue.getInstance();
​
        System.out.println(queue1 == queue2);
    }
}
```

**thread safe하게 변경하기**

여기까지만 해도 기본적인 싱글톤 패턴이 간단하게 완성된다. 하지만 위의 싱글톤 패턴은 멀티 스레드 환경에서는 안전하지 않다. (not thread safe) getInstance() 메서드의 if문에 두 스레드가 매우 가까운 시점으로 처음 접근한다면 두 스레드 모두 새로운 인스턴스를 생성할 수 있다.

```
if ( instance == null ) {
       instance = new WaitingQueue();
}
```

* **synchronized**

가장 쉬운 해결방법은 synchronized 키워드를 사용하여 동시에 여러 스레드가 접근할 수 없도록 보장하는 방법이다. 하지만 synchronized 동기화 매커니즘 자체가 성능적으로 단점이 있다.

```
public static synchronized WaitingQueue getInstance() {
        if ( instance == null ) {
            instance = new WaitingQueue();
        }
        return instance;
    }
```

* **이른 초기화**

인스턴스를 미리 생성하는 방법이다. 만들어놓고 쓰질 않는다면 손해이다.

```
public class WaitingQueue {
    private Queue<String> queue = new PriorityQueue<>();
​
    private static final WaitingQueue INSTANCE = new WaitingQueue();
​
    public static synchronized WaitingQueue getInstance() {
        return INSTANCE;
    }
​
    private WaitingQueue() {
​
    }
}
```

* **Double checked locking**

위의 두 단점을 보완하기 위해 Double checked locking 기법을 사용할 수 있다. 위의 두 단점을 보완하지만(성능도 좋고 미리 생성도 안함), 사용이 어려움 (volatile 같은 키워드를 사용)

```
public class WaitingQueue {
    private Queue<String> queue = new PriorityQueue<>();
​
    private static volatile WaitingQueue instance;
​
    public static WaitingQueue getInstance() {
        if ( instance == null ) {
            synchronized (WaitingQueue.class) {
                if ( instance == null ) {
                    instance = new WaitingQueue();
                }
            }
        }
        return instance;
    }
​
    private WaitingQueue() {
​
    }
}
​
```

* **static inner class**

보다 간단한 방법으로 정적 내부 클래스 사용 정적 내부 클래스는 처음으로 호출될 때 로딩된다. 즉, getInstance() 메서드를 처음 호출할 때 WaitingQueueHolder 클래스가 초기화되면서 싱글톤 인스턴스가 생성된다.

```
package designpattern;
​
import java.util.PriorityQueue;
import java.util.Queue;
​
public class WaitingQueue {
    private Queue<String> queue = new PriorityQueue<>();
​
    private static class WaitingQueueHolder {
        private static final WaitingQueue INSTANCE = new WaitingQueue();
    }
​
    public static WaitingQueue getInstance() {
        return WaitingQueueHolder.INSTANCE;
    }
​
    private WaitingQueue() {
​
    }
}
```

* **enum**

이 간단한 코드는 위의 클래스들보다도 더 안전하다. 싱글톤 인스턴스가 미리 만들어지고 상속을 사용하지 못하는 단점이 있다.

```
public enum WaitingQueueEnum {
    INSTANCE;
}
```

**실무에서 사용하는 싱글톤 패턴**

* java Runtime class
* 스프링 빈의 싱글톤 스코프\

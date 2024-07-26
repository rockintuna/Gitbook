# @Transactional 어노테이션이 protected에서는 동작하지 않는 이유

@Transactional 어노테이션에 대한 이해도가 낮아 발생했던 문제를 해결하면서 공부한 내용을 기록한다.



실제 작성했던 코드는 아니지만 동일한 상황을 재현

```java
@Service
@RequiredArgsConstructor
public class MyService {
  private final MyEntityRepository myEntityRepository;
  private final MyEntityHistoryRepository myEntityHistoryRepository;

  public void saveEntity(String name) {
    saveEntityAndHistory(name);
  }

  @Transactional
  protected void saveEntityAndHistory(String name) {
    myEntityRepository.save(new MyEntity(name));
    myEntityHistoryRepository.save(new MyEntityHistory(name));
  }
}
```

Controller에서 saveEntity() 메서드를 호출한다는 가정으로 문자열을 받으면 해당 문자열로 Entity와 EntityHistory를 DB에 저장하는 매우 간단한 코드이다.

Entity와 EntityHistory는 동일한 메서드 내에서 @Transactional 어노테이션으로 인해 선언적 트랜젝션으로 동작하며 하나라도 save 작업에 문제가 있는 경우 두 처리 모두 rollback 될 것으로 예상했다.

오류가 발생하지 않는 경우 문제가 있다는 것을 인지하지 못했으나 saveEntityAndHistory() 메서드에서 오류가 발생했을 때 문제가 발생한다.

예를들어 아래와 같이 두 save() 중간에 Rumtime Exception을 발생시켜보았다.

```java
@Service
@RequiredArgsConstructor
public class MyService {
  private final MyEntityRepository myEntityRepository;
  private final MyEntityHistoryRepository myEntityHistoryRepository;

  public void saveEntity(String name) {
    saveEntityAndHistory(name);
  }

  @Transactional
  protected void saveEntityAndHistory(String name) {
    myEntityRepository.save(new MyEntity(name));
    throwRuntimeException();
    myEntityHistoryRepository.save(new MyEntityHistory(name));
  }

  private void throwRuntimeException() {
    throw new RuntimeException("Exception");
  }
}
```

나는 선언적 트랜젝션에 의해 Entity 저장 작업이 롤백되어 Entity, Entity History 모두 저장되지 않는 것을 예상했다.

그러나&#x20;

Entity History는 예상되로 저장되지 않았지만 Entity 롤백되지 않고 DB에 저장되었다.

@Transactional을 붙였음에도 잘 적용되지 않은 것일까?

Transaction이 활성화 되었는지 확인하는 코드를 추가했다.

```java
  @Transactional
  protected void saveEntityAndHistory(String name) {
    if (TransactionSynchronizationManager.isActualTransactionActive()) {
      System.out.println("Transaction is active.");
    } else {
      System.out.println("No active transaction.");
    }
    myEntityRepository.save(new MyEntity(name));
    throwRuntimeException();
    myEntityHistoryRepository.save(new MyEntityHistory(name));
  }
```

결과는 "No active transaction." ....

왜일까?

서비스 코드를 수정해보자

```java
@Service
@RequiredArgsConstructor
public class MyService {
  private final MyEntityRepository myEntityRepository;
  private final MyEntityHistoryRepository myEntityHistoryRepository;

  @Transactional
  public void saveEntity(String name) {
    if (TransactionSynchronizationManager.isActualTransactionActive()) {
      System.out.println("Transaction is active.");
    } else {
      System.out.println("No active transaction.");
    }
    myEntityRepository.save(new MyEntity(name));
    throwRuntimeException();
    myEntityHistoryRepository.save(new MyEntityHistory(name));
  }

  private void throwRuntimeException() {
    throw new RuntimeException("Exception");
  }
}
```

고친것&#x20;

1. @Transactional 어노테이션을 외부에서 호출하는 메서드에 달았다.
2. @Transactional 어노테이션이 달린 메서드가 public으로 변경되었다.

결과는 "Transaction is active.", Entity도 정상적으로 롤백되었다.



원인을 찾아야한다.

@Transactional의 공식문서에는 짧지만 알고 있으면 좋을 내용들이 많았다.

{% embed url="https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/annotations.html" %}

`The default mode (proxy) processes annotated beans to be proxied by using Spring’s AOP framework`

**@Transactional 어노테이션은 기본적으로 Spring AOP를 사용한다.**

@Transactional의 경우 Spring AOP의 대표적인 예시이며, AOP를 통해 메서드에 부가적인 기능(선언적 트랜젝션)을 추가한 것이다.

Spring AOP는 CGLIB 또는 JDK 동적 프록시를 사용하여 프록시 빈을 생성한다. 즉 MyService의 메서드가 외부에서 호출되는 경우 MyService에 직접 호출하는 게 아니고 프록시 빈을 거쳐서 호출이 전달된다.

즉 다른 인스턴스(프록시 빈)가 이 메서드를 호출 가능해야한다!

고로 private 메서드는 @Transactional를 적용할 수 없다. 나머지 접근제어는 아래에서.



`The @Transactional annotation is typically used on methods with public visibility. As of 6.0, protected or package-visible methods can also be made transactional for class-based proxies by default.`

사실 public만 사용해야 하는 줄 착각하고 있었고 문제도 그것 때문인 줄 알았다. 그러나 spring 6.0 이상에서는 class-based proxy에서 protected와 package-private(default)에서도 @Transactional을 선언할 수 있다.

class-based proxy라 함은 인터페이스 대신 클래스를 통해 프록시 빈이 만들어진, 즉 CGLIB로 상속을 통해 프록시를 만들게 되었을 때이다.

상속을 이용한 프록시 빈에서는 protected에 접근할 수 있으며, Transactional 구현에 따라 default에서도 접근할 수 있을 것이다.

인터페이스의 추상메서드는 public이므로 어차피 다른 접근 제어를 사용할 수 없다.

제목은 "@Transactional 어노테이션이 protected에서는 동작하지 않는 이유" 이지만 실제로는 가능하다.



`In proxy mode (which is the default), only external method calls coming in through the proxy are intercepted. This means that self-invocation (in effect, a method within the target object calling another method of the target object) does not lead to an actual transaction at runtime even if the invoked method is marked with @Transactional`

내 코드가 정상적이지 않았던 진짜 이유이다.

self-invocation이라고 표현된 메서드에서 동일한 클래스 내의 메서드 호출은 @Transactional이 메서드에 걸려 있어도 트랜젝션이 적용되지 않는다고 한다.

saveEntityAndHistory() 메서드는 MyService 클래스의 saveEntity()안에서 내부적으로 호출되는데 이는 프록시가 인터셉트하지 않기 때문이다.\
프록시가 인터셉트하지 않는다는 것은 AOP로 구현한 Aspect, 즉 트랜젝션이 적용되지 않는 것이다.

하지만 @Transactional의 전파를 통해 내부에서 호출되는 메서드의 오류를 해당 메서드를 호출하는 외부 접근 메서드에서 처리할 수 있다.

```java
@Service
@RequiredArgsConstructor
public class MyServiceImpl implements MyService {
  private final MyEntityRepository myEntityRepository;
  private final MyEntityHistoryRepository myEntityHistoryRepository;

  @Transactional
  public void saveEntity(String name) {
    saveEntityAndHistory(name);
  }

  private void saveEntityAndHistory(String name) {
    myEntityRepository.save(new MyEntity(name));
    myEntityHistoryRepository.save(new MyEntityHistory(name));
  }
}
```

위처럼 오류는 saveEntityAndHistory()에서 발생하지만, saveEntity에서 롤백을 할 수 있다.



@Transactional 어노테이션 및 AOP에 대한 이해가 부족한 상태에서 무의식적으로 코딩하는 것이 문제를 발생시킨 것 같다.

그래도 왜 문제가 발생했는지와 AOP에 대해서 조금더 알게된 좋은 경험이 된 것 같다.

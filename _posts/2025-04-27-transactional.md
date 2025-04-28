---
title: "@Transactional 제대로 알아보기 - 1"
date: 2025-04-28 16:00:00 +0900
categories: [Spring]
tags: ["Spring"]
image:
  path: /assets/img/posts/2025-04-27-transactional/01.png
---

최근 카카오페이 기술 블로그의 [**JPA Transactional 잘 알고 쓰고 계신가요?**](https://tech.kakaopay.com/post/jpa-transactional-bri/#%EC%9D%98%EB%8F%84%ED%95%98%EC%A7%80-%EC%95%8A%EC%9D%80-transactional-%EB%8F%99%EC%9E%91) 글을 통해 `@Transactional`의 실제 동작 그리고 올바른 사용법에 대해 더 깊이 고민하게 되었다. 단순히 스프링에서 트랜잭션을 관리하는 어노테이션 정도로만 알고있었는데 이번기회에 글을 정리할 필요성을 느껴서 글을 게시하게 되었다.

## 트랜잭션(Transaction)
**트랜잭션**은 데이터베이스의 상태를 변화시키는 하나의 논리적 작업 단위로, 여러 개의 연산을 하나로 묶어 모두 성공하거나 모두 실패하도록 처리한다.


간단한 송금예제를 통해서 트랜잭션의 필요성을 알아보자.

1. **테이블 생성 및 데이터 추가**
<br>먼저 테이블을 생성하고, 두 명의 사용자를 추가
```sql
CREATE TABLE member (
	member_id VARCHAR(10),
	money INTEGER NOT NULL DEFAULT 0,
	PRIMARY KEY (member_id)
);
insert into member(member_id, money) values ('member1',10000);
insert into member(member_id, money) values ('member2',10000);
```


2. **트랜잭션 없이 송금 처리**
<br>`member1`이 `member2`에게 1000원을 송금한다고 가정해보자. 트랜잭션을 사용하지 않고 단순히 두 개의 UPDATE 쿼리를 실행
```sql
UPDATE member SET money=10000-1000 WHERE member_id = 'member1';
UPDATE member SET money=10000+1000 WHERE member_id = 'member2';
```

3. **트랜잭션이 없을 때 발생할 수 있는 문제**
<br>이 두 쿼리는 별개의 작업으로 처리된다. 만약 첫 번째 쿼리(출금)는 성공했지만, 두 번째 쿼리(입금) 실행 전에 시스템 오류나 네트워크 장애가 발생하게 된다면
<br>`member1`의 잔액은 9,000원이 되지만,
<br>`member2`의 잔액은 여전히 10,000원입니다.
<br>즉, 돈이 사라지는 심각한 데이터 불일치가 발생할 수 있다.

4. **트랜잭션을 사용한 송금처리**
<br>이런 문제를 방지하기 위해 사용하는 것이 바로 트랜잭션이다. 위에 서술했듯 2개의 쿼리를 하나의 논리적 작업 단위로 묶는 것이다.
<br>트랜잭션을 사용해서 데이터를 추가할 때는 자동커밋을 꺼야한다. `START TRANSACTION` 명령을 통해 명시적으로 시작을 선언해야 한다.
```sql
START TRANSACTION; -- 트랜잭션 시작 (자동 커밋 OFF) 
-- member1에서 1000원 출금
UPDATE member SET money = money - 1000 WHERE member_id = 'member1'; 
-- member2에게 1000원 입금 
UPDATE member SET money = money + 1000 WHERE member_id = 'member2'; 
```
만약 모든 쿼리가 완료하면 COMMIT을 사용해 변경사항을 적용하면 된다.
```sql
-- 모든 작업 성공 시 커밋 (변경사항 영구 저장) 
COMMIT; 
```
중간에 실패했을 경우 변경사항을 폐기해한다.
```sql
-- 중간에 작업 실패 시 롤백 (변경사항 파기)
ROLLBACK;
```

이렇게 트랜잭션은 데이터의 정합성과 신뢰성을 보장하기 위해 꼭 필요하다.
특히 위와 같이 여러 단계의 데이터 변경이 모두 성공해야만 의미가 있는 경우, 트랜잭션을 사용하지 않으면 데이터베이스의 무결성이 쉽게 깨질 수 있다.

## 스프링에서의 트랜잭션 관리
앞선 예시를 통해 트랜잭션의 중요성을 알 수 있었다. 

스프링에서는 트랜잭션 관리의 핵심 인터페이스가 바로 `PlatformTransactionManager`이다.
```java
public interface PlatformTransactionManager extends TransactionManager {
  // 트랜잭션 획득(시작)
  TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;

  // 트랜잭션을 커밋
  void commit(TransactionStatus status) throws TransactionException;

  // 트랜잭션을 롤백
  void rollback(TransactionStatus status) throws TransactionException;
}
```

인터페이스와 구현체를 포함해서 **트랜잭션 매니저**로 줄여서 이야기하겠다. 트랜잭션 매니저는 크게 2가지 역할을 한다.
1. **트랜잭션 추상화**
  - 각각의 데이터 접근 기술들은 트랜잭션을 처리하는 방식에 차이가 있다. 추상화를 함으로써 이런 상황에서도 문제없이 트랜잭션 관리를 할 수 있게된다. 스프링에서 `PlatformTransactionManager`라는 인터페이스를 통해 트랜잭션을 추상화한다.
2. **리소스 동기화**
  - 트랜잭션은 커넥션 분산, 동시성, 커넥션 누수 등의 이유로 시작부터 종료까지 같은 커넥션을 사용해야 한다.
  - 트랜잭션 메니저는 트랜잭션 동기화 매니저를 통해 처음부터 끝까지 같은 커넥션을 사용하도록 한다.
  - 쓰레드 로컬을 사용하는 것을 확인할 수 있다. (`TransactionSynchronizationManager`를 참고)

그리고 위의 트랜잭션 매니저를 통해서 스프링은 트랜잭션을 관리를 지원한다.
1. 선언적 트랜잭션 관리([Declarative Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html))
   - 개발자가 트랜잭션 관련 코드를 직접 작성하지 않고, 클래스나 메서드에 `@Transactional`을 선언만 하면 스프링이 AOP(프록시 패턴)를 통해 트랜잭션의 시작, 커밋, 롤백을 자동으로 처리
2. 프로그래밍 방식의 트랜잭션 관리([programmatic transaction management](https://docs.spring.io/spring-framework/reference/data-access/transaction/programmatic.html))
   - 개발자가 트랜잭션을 시작(Begin), 커밋(Commit), 롤백(Rollback)하는 코드를 직접 작성

`@Transactional` 어노테이션은 스프링에서 선언적으로 트랜잭션을 관리하는 대표적인 방법이다.
메서드나 클래스에 선언만 하면, 스프링이 프록시 객체를 생성해 해당 메서드 실행 전 트랜잭션을 시작하고, 정상 종료 시 커밋, 예외 발생 시 롤백을 자동으로 처리한다.

## @Trnasactonal 동작 방식
어노테이션하나 붙이는 것만으로 어떻게 커밋 롤백을 자동으로 처리하는 것일까? 비밀은 바로 AOP에 있다.
`@Transactional`을 통해 선언적 트랜잭션 관리 방식을 사용하게 되면 기본적으로 프록시 방식의 AOP가 적용된다.

![Desktop View](/assets/img/posts/2025-04-27-transactional/02.png){: width="972" height="589" }
_[트랜잭션 프록시(transactional proxy)에서 메서드를 호출하는 과정](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)_

`@Transactional`의 AOP의 구성은 다음과 같다.
- 어드바이저: `BeanFactoryTransactionAttributeSourceAdvisor`
- 포인트컷: `TransactionAttributeSourcePointcut`
- 어드바이스: `TransactionInterceptor`



## @Transactional 사용
> 전체 프로젝트 코드는 [Github](https://github.com/tjvm0877/blog-code/tree/main/transactional)에 있으니 참고해주세요.
{: .prompt-info }

이제 간단 사용법을 알아보자, 예제 코드는 다음과 같다.
확인하고자 하는 상황은 크게 3가지이다.
1. 정상 동작
2. `Unchected Excaption` 발생
3. `Checked Exception` 발생

```java
@Entity
@Table(name = "orders")
@Getter
@Setter
public class Order {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String username; //정상, 예외, 잔고부족

  private String payStatus; //대기, 완료
}
```
{: file='Order'}

```java
public class NotEnoughMoneyException extends Exception {
  public NotEnoughMoneyException(String message) {
    super(message);
  }
}
```
{: file='NotEnoughMoneyException'}

```java
@Transactional
public void order(Order order) throws NotEnoughMoneyException {
  log.info("order 호출");
  orderRepository.save(order);

  log.info("결제 프로세스 진입");
  if (order.getUsername().equals("예외")) {
      log.info("시스템 예외 발생");
      throw new RuntimeException("시스템 예외");
  } else if (order.getUsername().equals("잔고부족")) {
      log.info("잔고 부족 비즈니스 예외 발생");
      order.setPayStatus("대기");
      throw new NotEnoughMoneyException("잔고가 부족합니다");
  } else {
      //정상 승인
      log.info("정상 승인");
      order.setPayStatus("완료");
  }
  log.info("결제 프로세스 완료");
}
```
{: file='OrderService'}

위 예제를 실행할 테스트코드는 다음과 같다.
```java
@SpringBootTest
class OrderServiceTest {

  @Autowired
  OrderService orderService;

  @Autowired
  OrderRepository orderRepository;

  @Test
  void success() throws NotEnoughMoneyException {
    Order order = new Order();
    order.setUsername("정상");

    orderService.order(order);

    // Order가 성공적으로 저장했는지 확인
    Order findOrder = orderRepository.findById(order.getId()).get();
    assertThat(findOrder.getPayStatus()).isEqualTo("완료");
  }

  @Test
  void whenUnCheckedExcaption() {

    Order order = new Order();
    order.setUsername("예외");

    // Unchected Excaption 발생 확인
    assertThatThrownBy(() -> orderService.order(order))
      .isInstanceOf(RuntimeException.class);

    // Unchected Excaption 발생 후 Order가 저장됬는지 확인
    Optional<Order> orderOptional = orderRepository.findById(order.getId());
    assertThat(orderOptional.isEmpty()).isTrue();
  }

  @Test
  void whenCheckedExcaption() {
    Order order = new Order();
    order.setUsername("잔고부족");

    // Chected Excaption 발생 확인
    assertThatThrownBy(() -> orderService.order(order))
      .isInstanceOf(NotEnoughMoneyException.class)
      .hasMessage("잔고가 부족합니다");

    // Chected Excaption 발생 후 Order가 성공적으로 저장됬는지 확인
    Order findOrder = orderRepository.findById(order.getId()).get();
    assertThat(findOrder.getPayStatus()).isEqualTo("대기");
  }
}
```
실행 후 결과는 다음과 같다.
1. 정상 동작
   - 문제 없음 
2. `Unchected Excaption` 발생
   - 롤백되어 데이터가 없음
3. `Checked Exception` 발생
   - `Order`가 제대로 저장되고 `order.payStatus`가 "대기"를 가지고 있는 것으로보아 결제 프로세스까지 제대로 실행되고 저장되었다는 것을 알 수 있음

`@Transactional`이 없는 경우는 다음과 같다.
![Desktop View](/assets/img/posts/2025-04-27-transactional/03.png){: width="972" height="589" }
_@Trnasactional이 없을 경우 테스트 결과_
1. 정상 동작
   - 문제 없음 
2. `Unchected Excaption` 발생
   - 롤백되어 데이터가 없음
3. `Checked Exception` 발생
   - `Order`가 제대로 저장되고 `order.payStatus`가 "대기"를 가지고 있는 것으로보아 결제 프로세스까지 제대로 실행되고 저장되었다는 것을 알 수 있음

위 테스트를 결과를 보면 알 수 있듯, `@Trnasactionla`을 통해서 스프링이 트랜잭션을 지원하는 것을 알 수 있다.

### `Checked Exception` 발생이 롤백되지 않는 이유
문득 그런 사람이 든다. `Checked Exception`도 예외인데 왜 롤백하지 않는가?

[스프링 공식문서](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/rolling-back.html)에서는 다음과 같이 설명한다.
>"The recommended way to indicate to the Spring Framework’s transaction infrastructure that a transaction’s work is to be rolled back is to throw an Exception from code that is currently executing in the context of a transaction. The Spring Framework’s transaction infrastructure code catches any unhandled Exception as it bubbles up the call stack and makes a determination whether to mark the transaction for rollback."<br>
Spring 프레임워크의 트랜잭션 인프라에 작업 롤백을 지시하는 권장 방식은, 현재 트랜잭션 컨텍스트에서 실행 중인 코드에서 Exception을 발생시키는 것입니다. Spring의 트랜켁션 인프라 코드는 호출 스택을 따라 전파되는 처리되지 않은 모든 예외를 포착한 후, 해당 트랜잭션을 롤백 처리할지 여부를 결정합니다.


그리고 해당 동작은 `@Transactional`이 롤백여부를 결정하는 [`rollbackOn`](https://github.com/spring-projects/spring-framework/blob/0c6a26a381505479ae62a67eb018a5e41447bbb7/spring-tx/src/main/java/org/springframework/transaction/interceptor/DefaultTransactionAttribute.java#L180)메서드를 볼 수 있다.
```java
/**
 * The default behavior is as with EJB: rollback on unchecked exception
 * ({@link RuntimeException}), assuming an unexpected outcome outside any
 * business rules. Additionally, we also attempt to rollback on {@link Error} which
 * is clearly an unexpected outcome as well. By contrast, a checked exception is
 * considered a business exception and therefore a regular expected outcome of the
 * transactional business method, i.e. a kind of alternative return value which
 * still allows for regular completion of resource operations.
 * <p>This is largely consistent with TransactionTemplate's default behavior,
 * except that TransactionTemplate also rolls back on undeclared checked exceptions
 * (a corner case). For declarative transactions, we expect checked exceptions to be
 * intentionally declared as business exceptions, leading to a commit by default.
 * @see org.springframework.transaction.support.TransactionTemplate#execute
 */
@Override
public boolean rollbackOn(Throwable ex) {
  return (ex instanceof RuntimeException || ex instanceof Error);
}
```
{: file='DefaultTransactionAttribute'}

해당 메서드의 주석을 통해 Defualt설정에서 `Checked Exception`을 롤백하지 않는 이유를 알 수 있다.
**`RuntimeException`**은 **비즈니스 규칙을 벗어난 예상치 못한 결과**로 간주하고, 또한 `Error`도 **명백히 예상치 못한 결과**로 간주한다. <br>
반면 체크예외는 비즈니스 예외로 간주되어 메서드의 **정상적인 예상 결과**로 판단하기 때문에 리소스 작업을 완료할 수 있는 일종의 반환값으로 간주한다.

## 마무리
여기까지 트랜잭션, `@Transactional`의 기본적인 사용법과 동작 방식에 대해 정리해보았다.
이 글을 통해 트랜잭션이 데이터의 정합성과 신뢰성을 보장하는 핵심 개념임을 다시 한 번 확인할 수 있다.
특히 스프링에서 @Transactional을 선언적으로 사용할 때, 내부적으로 프록시 기반의 AOP가 동작하여 트랜잭션의 시작과 종료(커밋/롤백)를 자동으로 관리해준다는 점을 예제과 함께 살펴보았다.

정리하다보니 여러 주의사항을 좀 빠뜨린 것을 알게되었다. 다음에 이부분에 대해서 포스트할 필요가 있을 것 같다. 그렇기 때문에 포스트의 제목 끝에 "1편:을 붙였다.

정리하자면, 트랜잭션은 단순히 데이터베이스의 일관성을 위한 기술적 장치 그 이상으로, 비즈니스 로직의 신뢰성과 예측 가능성을 보장하는 필수 도구다. 이 포스트를 통해 트랜잭션의 원리와 스프링에서의 작동 방식을 정리하고, @Trnasactional을 사용할 때 이부분들을 유의해야겠다.


## 참고
[스프링 공식 문서 \| Transactionality :: Spring Data JPA](https://docs.spring.io/spring-data/jpa/reference/jpa/transactions.html)

[JPA Transactional 잘 알고 쓰고 계신가요? \| 카카오페이 기술 블로그](https://tech.kakaopay.com/post/jpa-transactional-bri/#set_option%EC%9D%80-%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80)

[스프링 공식 문서 \| Transaction Management](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative.html)

[인프런 강의 \| 스프링 DB 1편 - 데이터 접근 핵심 원리](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-db-1/dashboard)

---
title: Spring Data JPA에서의 페이지네이션과 정렬
date: 2025-04-30 19:00:00 +0900
categories: [Spring]
tags: [spring, jpa]
---
## 페이지네이션 (Pagination)
---
사용자가 요청했을 때 데이터에비으세 있는 모든 데이터를 조회하여 제공한다면 부하가 굉장이 클 것이다. 이를 방지하기 위해 대부분의 서비스에서는 데이터를 일정 길이로 잘라 그 일부만을 사용자에게 제공하는 방식을 사용한다. 사용자는 현재 보고 있는 데이터의 다음, 이전 구간 혹은 특정 구간의 데이터를 요청하고, 전달한 구간에 해당하는 데이터를 제공받는다.

![Desktop View](/assets/img/posts/2025-04-30-spring-data-jpa-pagination/01.png){: width="400" height="300" }
_페이지네이션 예시_

이번 포스트에서는 Spring Data JPA를 통해 어떻게 페이지네이션을 구현하는지 알아보자.

## 순수 JPA의 페이지네이션
---
DB 벤더별로 페이지네이션을 처리하기 위한 SQL 문법이 다르다. 

```sql
/* MySql 페이지네이션 쿼리 예시 */
SELECT
  M.ID AS ID,
  M.AGE AS AGE,
  M.TEAM_ID AS TEAM_ID,
  M.NAME AS NAME
FROM
  MEMBER M
ORDER BY
  M.NAME DESC LIMIT ?, ?
```
```sql
/* Oracle 페이지네이션 쿼리 예시 */
SELECT * FROM
  ( SELECT ROW_.*, ROWNUM ROWNUM_
  FROM
    ( SELECT
      M.ID AS ID,
      M.AGE AS AGE,
      M.TEAM_ID AS TEAM_ID,
      M.NAME AS NAME
      FROM MEMBER M
      ORDER BY M.NAME
    ) ROW_
  WHERE ROWNUM <= ?
  )
WHERE ROWNUM_ > ?
```

JPA에서는 이런 DB벤더별 방언(dialect)을 다음 두 API로 추상화하여 하나의 방법으로 페이지네이션을 구현할 수 있도록 제공해준다.
- `setFirstResult(int startPosision)`: 조회 시작 위치(0부터 시작)
- `setMaxResults(int maxResult)`: 조회할 데이터 수

```java
// JPA 페이징 사용 예시
List<Item> items = entityManager.createQuery("select i from Item i", Item.class)
  .setFirstResult(0) // 0부터 조회
  .setMaxResults(10) // 10개 데이터르 가져온다.
  .getResultList();
```

## Spring Data JPA가 제공하는 페이지네이션
---
JPA로 페이지네이션 기능을 실제로 사용할 때는 생각보다 까다롭다. 전체 데이터 개수를 가져와서 **1)전체 페이지를 계산해야**하고, **2)현재 페이지가 첫번째 페이지인지, 마지막 페이지인지도 계산**해야하고, 예상치 못한 범위를 요청받았을 때 **3)예외 처리**도 해야한다.어렵지는 않지만 번거롭고 불편하다.

Spring Data JPA에서는 인터페이스에 메서드만 선언하면 해당 메서드의 이름으로 적절한 JPQL 쿼리를 생성해서 실행해주는 쿼리 메소드 기능을 가지고 있다.이 쿼리 메서드에 페이징과 정렬 기능을 사용할 수 있도록 2가지 파라미터를 제공한다.
- `Sort`: 정렬 기능
- `Pageable`: 페이징 기능(내부의 Sort 포함)

### Pagealbe과 PageRequest
![Desktop View](/assets/img/posts/2025-04-30-spring-data-jpa-pagination/02.png){: width="972" height="589" }
Pagealbe과 PageRequest는 Spring Data JPA에서 제공하는 페이지네이션 정보를 담기위한 인터페이스와 구현체이다. 이를 파라미터로 전달하여 반환되는 엔티티의 컬렉션에 대해 페이징 기능을 구현할 수 있다.

**Pageable 생성**
아래와 정적 팩토리 메서드를 사용하여 다음 코드와 같이 다양한 방식으로 생성할 수 있다.
```java
// 1. 페이지 번호와 페이지 크기
PageRequest.of(0, 10);

// 2. 페이지 번호, 페이지 크기, 정렬 방식, 정렬 기준으로 생성
PageRequest.of(0, 10, Sort.by("price").descending());
PageRequest.of(0, 10, Sort.by(Direction.DESC, "price"));
PageRequest.of(0, 10, Sort.by(Order.desc("price")));
PageRequest.of(0, 10, Direction.DESC, "price");
```

### Slice와 Page
![Desktop View](/assets/img/posts/2025-04-30-spring-data-jpa-pagination/03.png){: width="972" height="589" }
Spring Data JPA 레포지토리에 `Pageable`을 전달하면 반환타입으로 `Slice` 혹은 `Page`를 받을 수 있다. 두 인터페이스 모두 페이지네이션을 통한 조회 결과를 저장하는 역할을 한다. `Page`는 `Slice`를 상속받는다.

#### Slice
전체 페이지 개수를 알아내기 위해서는 `조건을 만족하는 전체 데이터 갯수 / 페이지의 크기`로 계산해야한다. 즉 전체 데이터의 개수를 알아야 한다. 이 전체 데이터 개수를 알아내기 위해서는 `count`쿼리를 실행해야 한다. 반대로 전체 페이지 개수가 굳이 필요 없는 경우에는 `count`쿼리를 굳이 실행할 필요가 없을 것이다.

`Slice`는 별도로 `count`쿼리를 실행하지 않는다. 따라서 전체 페이지의 개수와 전체 데이터의 개수를 알 수 없지만, 불필요한 `count`쿼리로 인한 성능 낭비는 발생하지 않는다.

다음 페이지의 존재 유무만 판단하고, 있다면 다음 페이지를 불러오기만 하면 되기 때문이다. 이런 기능을 구현할 때 `Slice`를 사용하기 적합하다.

Slice의 주요 메서드들을 알아보자
```java
public interface Slice<T> extends Streamable<T> {

  int getNumber(); // 현재 Slice 번호 반환

  int getSize(); // 현재 Slice 크기 반환

  int getNumberOfElements(); // 현재 Slice가 가지고 있는 데이터 갯수 반환

  List<T> getContent(); // Slice가 가지고 있는 데이터르 List로 반환

  boolean hasContent(); // Slice가 데이터를 가지고 있는지 여부 반환

  Sort getSort(); // 현재 Slice의 Sort객체 반환

  boolean isFirst(); // 첫번째 페이지인지 여부 반환

  boolean isLast(); // 마지막 페이지인지 여부 반환

  boolean hasNext(); // 다음페이지 존재 여부 반환

  boolean hasPrevious(); // 이전 페이지 존재 여부 반환

  // 현재 Slice를 통해 Pageable 생성하고 반환
  default Pageable getPageable() {
    return PageRequest.of(getNumber(), getSize(), getSort());
  }

  Pageable nextPageable(); // 다음 Pageable 반환

  Pageable previousPageable(); // 이전 Pageable 반환

  // Slece가 가지고 있는 데이터(엔티티)를 다른 객체로 매핑
  @Override
  <U> Slice<U> map(Function<? super T, ? extends U> converter); 

  // 다음 페이지가 있다면 다음 페이지의 Pageable을 반환하고, 마지막이라면 현재 Pagealbe을 반환
  default Pageable nextOrLastPageable() {
    return hasNext() ? nextPageable() : getPageable();
  }

  // 이전 페이지가 있다면 이전 Pageable을 반환하고, 첫번째라면 현재 Pagealbe을 반환
  default Pageable previousOrFirstPageable() {
    return hasPrevious() ? previousPageable() : getPageable();
  }
}
```

#### Page
`Page` 는 `Slice` 와 다르게 `count` 쿼리를 실행하여, 전체 데이터 개수와 전체 페이지 개수를 계산할 수 있다.
위에 첨부한 페이지네이션 예시와 같이 구현할 때 사용하기 적합하다.

Page의 주요 메서드는 다음과 같다.
```java
// 앞서 서술했던 것과 같이 Slice를 상속받는다. 그렇기 때문에 Slice의 기능은 그대로 가져온다.
public interface Page<T> extends Slice<T> {

  // 빈 Pageable을 생성
  static <T> Page<T> empty() {
    return empty(Pageable.unpaged());
  }

  // Pageable을 받아 빈 페이지를 생성하고 반환
  static <T> Page<T> empty(Pageable pageable) {
    return new PageImpl<>(Collections.emptyList(), pageable, 0);
  }

  // 전체 페이지 갯수를 반환
  int getTotalPages();

  // 전체 데이터 갯수 반환
  long getTotalElements();

  // Page가 가지고 있는 데이터(엔티티)를 다른 객체로 매핑
  @Override
  <U> Page<U> map(Function<? super T, ? extends U> converter);
}
```
## 직접 사용해보기
---
> 전체 프로젝트 코드는 [Github](https://github.com/tjvm0877/blog-code/tree/main/transactional)에 있으니 참고해주세요.
{: .prompt-info }

### 기본적인 페이지네이션
앞서 서술한 `Slice`와 `Page`를 직접 사용해보자.
예제로 사용할 Entity는 다음과 같다.
```java
@Entity
@Getter
@ToString(of = {"name", "price"})
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item {

  @Id
  @GeneratedValue(strategy = GenerationType.IDENTITY)
  private Long id;

  private String name;

  private int price;

  public Item(String name, int price) {
    this.name = name;
    this.price = price;
  }
}
```
{: file='Item'}

`Slice`와 `Page`를 사용할 `ItemRepository`은 다음과 같다.
```java
// Spring Data JPA의 쿼리 메서드에 페이징과 정렬 기능을 사용
public interface ItemRepository extends JpaRepository<Item, Long> {

  // price와 같은 아이템들을 Slice로 반환
  Slice<Item> findSliceByPrice(int price, Pageable pageable);

  // price와 같은 아이템들을 Page로 반환
  Page<Item> findPageByPrice(int price, Pageable pageable);
}
```
{: file='ItemRepository'}

이제 `Item`을 `Page`와 `Slice`로 각각 불러오고 관련 정보들을 확인해보자
```java
@Test
void getSlice() {
  // Item 저장
  for (int i = 0; i < 30; i++) {
    Item item = new Item("상품" + i, 1000);
    itemRepository.save(item);
  }

  // PageRequest 생성
  PageRequest pageRequest = PageRequest.of(0, 5);

  // 조회
  Slice<Item> itemSlice = itemRepository.findSliceByPrice(1000, pageRequest);

  // 페이지 번호
  System.out.println("itemSlice.getNumber() = " + itemSlice.getNumber());
  
  // 페이지 크기
  System.out.println("itemSlice.getSize() = " + itemSlice.getSize());

  // 첫번째 페이지 여부
  System.out.println("itemSlice.isFirst() = " + itemSlice.isFirst());

  // 마지막 페이지 여부
  System.out.println("itemSlice.isLast() = " + itemSlice.isLast());

  // 조회된 데이터 리스트 출력
  for (Item item : itemSlice.getContent()) {
    System.out.println(item.toString());
  }
}
```
{: file='SliceAndPageTest'}

```java
@Test
void getPage() {
  // Item 저장
  for (int i = 0; i < 30; i++) {
    Item item = new Item("상품" + i, 1000);
    itemRepository.save(item);
  }

  // PageRequest 생성
  PageRequest pageRequest = PageRequest.of(0, 5);

  // 조회
  Page<Item> itemPage = itemRepository.findPageByPrice(1000, pageRequest);

  // 페이지 번호
  System.out.println("itemPage.getNumber() = " + itemPage.getNumber());

  // 페이지 크기
  System.out.println("itemPage.getSize() = " + itemPage.getSize());

  // 첫번째 페이지 여부
  System.out.println("itemPage.isFirst() = " + itemPage.isFirst());

  // 마지막 페이지 여부
  System.out.println("itemPage.isLast() = " + itemPage.isLast());

  // 다음 페이지 존재 여부
  System.out.println("itemPage.hasNext() = " + itemPage.hasNext());

  // 전체 페이지 번호
  System.out.println("itemPage.getTotalPages = " + itemPage.getTotalPages());

  // 전체 데이터 갯수
  System.out.println("itemPage.getTotalElements = " + itemPage.getTotalElements());

  // 페이지 데이터 리스트 출력
  for (Item item : itemPage.getContent()) {
    System.out.println(item.toString());
  }
}
```
{: file='SliceAndPageTest'}

`getSlice()` 출력 결과
```
itemSlice.getNumber() = 0
itemSlice.getSize() = 10
itemSlice.isFirst() = true
itemSlice.isLast() = false
itemSlice.hasNext() = true
Item(name=상품0, price=1000)
Item(name=상품1, price=1000)
Item(name=상품2, price=1000)
Item(name=상품3, price=1000)
Item(name=상품4, price=1000)
```
`getPage()` 출력 결과
```
itemPage.getNumber() = 0
itemPage.getSize() = 10
itemPage.isFirst() = true
itemPage.isLast() = false
itemPage.hasNext() = true
itemPage.getTotalPages = 3 // 전체 페이지 갯수
itemPage.getTotalElements = 30 // 전체 데이터 갯수
Item(name=상품0, price=1000)
Item(name=상품1, price=1000)
Item(name=상품2, price=1000)
Item(name=상품3, price=1000)
Item(name=상품4, price=1000)
```
### 조회결과 DTO로 매핑하기
`Slice`와 `Page`에는 위에 주요 메서드에서 보았듯이 `map()`메서드를 사용해서 조회된 데이터(Entity)를 다른 객체로 매핑할 수 있다. 
이 매서드를 통해서 Item을 ItemDto로 매핑해보자.
```java
@Test
void test() {
  // Item 저장
  for (int i = 0; i < 30; i++) {
    Item item = new Item("상품" + i, 1000);
    itemRepository.save(item);
  }

  // 조회
  PageRequest pageRequest = PageRequest.of(0, 5);
  Slice<Item> itemSlice = itemRepository.findSliceByPrice(1000, pageRequest);

  // Slice 내부의 Item을 ItemDto로 매핑
  Slice<ItemDto> itemDtoSlice = itemSlice.map(ItemDto::from);
}
```
{: file='MappingTest'}
`Slice<Item>` 가 `Slice<ItemDto>` 로 잘 변환한 것을 확인할 수 있다.

## Slice는 어떻게 다음 페이지(Slice)의 존재 유무를 판단할까?
`Slice` 는 분명 `count` 쿼리를 실행하지 않아서 전체 페이지 개수를 알 수 없다고 했다. 그런데, `Slice` 인터페이스 코드를 보면 `isFirst()` , `isLast()` , `hasNext()` , `hasPrevious()` 와 같이 전체 페이지 개수를 알아야지만 실행할 수 있는 메소드를 제공할 수 있는 것 일까? 해답은 JPA에서 보내는 쿼리에 있었다.

```
select 
  i1_0.id,
  i1_0.name,
  i1_0.price 
from 
  item i1_0 
where 
  i1_0.price=1000 
fetch 
  first 6 rows only; // 5개를 조회했는데 6개를 요청
```
{: file='p6spy 라이브러리를 이용한 쿼리 파라미터 확인'}

Spring Data JPA는 **전달된 페이지 사이즈에 1을 더한 값으로 쿼리를 실행**한다. 즉, 총 6개의 데이터를 쿼리하는 것이다.

## 마무리
페이지네이션은 대용량 데이터를 효율적으로 다루기 위한 필수적인 기술이기 때문에 언제한번 정리가 필요하다고 생각했다.

특히 Spring Data JPA는 복잡한 SQL이나 로직 없이도 `Pageable`과 `Sort` 파라미터만으로 간단하게 페이징과 정렬을 구현할 수 있도록 도와준다. `Slice`와 `Page` 타입을 적절히 선택하면, 전체 데이터 개수가 필요한지 여부에 따라 불필요한 count 쿼리를 줄일 수 있고, 성능 최적화에도 도움이 된다. 또한, 조회 결과를 DTO로 매핑하는 map() 메서드 활용법까지 익혀두면, API 응답 구조를 유연하게 설계할 수 있을 것이다.

추가로 Spring MVC에서는 `HandlerMethodArgumentResolver`를 통해서 Controller계층에서 바로 `Pageable`을 파라미터로 받을 수 있다. 이 부분은 다음에 `ArgumentResolver` 관련 포스트에서 알아보도록 하자.

## 참고
자바 ORM 표준 JPA 프로그래밍, 김영한

[Spring Data JPA에서의 페이지네이션과 정렬 \| hudi.blog](https://hudi.blog/spring-data-jpa-pagination/)


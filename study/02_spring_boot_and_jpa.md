

### V1 엔티티를 Request Body에 직접 매핑
- 문제점
  - 엔티티에 프레젠테이션 계층을 위한 로직이 추가된다.
  - 엔티티에 API 검증을 위한 로직이 들어간다. (@NotEmpty 등등)
  - 실무에서는 회원 엔티티를 위한 API가 다양하게 만들어지는데, 한 엔티티에 각각의  
    API를 위한 모든 요청 요구사항을 담기는 어렵다.
  - 엔티티가 변경되면 API 스펙이 변한다.
- 결론
  - API 요청 스펙에 맞추어 별도의 DTO를 파라미터로 받는다.


## 지연 로딩과 조회 성능 최적화

### 간단한 주문 조회 V2: 엔티티를 DTO로 변환

- OrderSimpleApiController

``` java
  /**
   * V2. 엔티티를 조회해서 DTO로 변환(fetch join 사용X)
   * - 단점: 지연로딩으로 쿼리 N번 호출
   */
  @GetMapping("/api/v2/simple-orders")
  public List<SimpleOrderDto> ordersV2() {
      List<Order> orders = orderRepository.findAll();
      List<SimpleOrderDto> result = orders.stream()
               .map(o -> new SimpleOrderDto(o))
               .collect(toList());
     return result;
  }
```

``` java
  @Data
  static class SimpleOrderDto {
      private Long orderId;
      private String name;
      private LocalDateTime orderDate; //주문시간
      private OrderStatus orderStatus;
      private Address address;
      
      public SimpleOrderDto(Order order) {
         orderId = order.getId();
         name = order.getMember().getName();
         orderDate = order.getOrderDate();
         orderStatus = order.getStatus();
         address = order.getDelivery().getAddress();
      }
  }
``` 

- 엔티티를 DTO로 변환하는 일반적인 방법이다.
- 쿼리가 총 1 + N + N번 실행된다. (v1과 쿼리수 결과는 같다.)
  - order 조회 1번(order 조회 결과 수가 N이 된다.)
  - order -> member 지연 로딩 조회 N 번
  - order -> delivery 지연 로딩 조회 N 번
  - 예) order의 결과가 4개면 최악의 경우 1 + 4 + 4번 실행된다.(최악의 경우)
    - 지연로딩은 영속성 컨텍스트에서 조회하므로, 이미 조회된 경우 쿼리를 생략한다.

- OrderSimpleApiController

``` java
  /**
   * V3. 엔티티를 조회해서 DTO로 변환(fetch join 사용O)
   * - fetch join으로 쿼리 1번 호출
   * 참고: fetch join에 대한 자세한 내용은 JPA 기본편 참고(정말 중요함)
   */
  @GetMapping("/api/v3/simple-orders")
  public List<SimpleOrderDto> ordersV3() {
      List<Order> orders = orderRepository.findAllWithMemberDelivery();
      List<SimpleOrderDto> result = orders.stream()
                .map(o -> new SimpleOrderDto(o))
                .collect(toList());
      return result;
  }

```

- OrderRepository

``` java
  public List<Order> findAllWithMemberDelivery() {
      return em.createQuery(
               "select o from Order o" +
               " join fetch o.member m" +
               " join fetch o.delivery d", Order.class
      ).getResultList();
  }
```

- 엔티티를 페치 조인(fetch join)을 사용해서 쿼리 1번에 조회
- 페치 조인으로 order -> member , order -> delivery 는 이미 조회 된 상태 이므로 지연로딩X

### 간단한 주문 조회 V4: JPA에서 DTO로 바로 조회

- OrderSimpleApiController 

``` java
  /**
   * V4. JPA에서 DTO로 바로 조회
   * - 쿼리 1번 호출
   * - select 절에서 원하는 데이터만 선택해서 조회
   */
  @GetMapping("/api/v4/simple-orders")
  public List<OrderSimpleQueryDto> ordersV4() {
      return orderSimpleQueryRepository.findOrderDtos();
  }
``` 

- OrderSimpleQueryRepository

``` java
  public List<OrderSimpleQueryDto> findOrderDtos() {
        return em.createQuery(
                "select new jpabook.jpashop.repository.order.simplequery.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, m.address) from Order o " +
                " join o.member m" +
                " join o.delivery d", OrderSimpleQueryDto.class).getResultList();
  }
``` 

- Dto

``` java
  @Data
  public class OrderSimpleQueryDto {
  
      private Long orderId;
      private String name;
      private LocalDateTime orderDate; //주문시간
      private OrderStatus orderStatus;
      private Address address;
      
      public OrderSimpleQueryDto(Long orderId, String name, LocalDateTime
                                    orderDate, OrderStatus orderStatus, Address address) {
          this.orderId = orderId;
          this.name = name;
          this.orderDate = orderDate;
          this.orderStatus = orderStatus;
          this.address = address;
      }
  }
``` 

- 일반적인 SQL을 사용할 때 처럼 원하는 값을 선택해서 조회
- new 명령어를 사용해서 JPQL의 결과를 DTO로 즉시 변환
- SELECT 절에서 원하는 데이터를 직접 선택하므로 DB 애플리케이션 네트웍 용량 최적화(생각보다 미비)
- 리포지토리 재사용성 떨어짐, API 스펙에 맞춘 코드가 리포지토리에 들어가는 단점
  
**정리**
  엔티티를 DTO로 변환하거나, DTO로 바로 조회하는 두가지 방법은 각각 장단점이 있다. 둘중 상황에 따라
  서 더 나은 방법을 선택하면 된다. 엔티티로 조회하면 리포지토리 재사용성도 좋고, 개발도 단순해진다. 따라
  서 권장하는 방법은 다음과 같다. 

- 쿼리 방식 선택 권장 순서
- 1. 우선 엔티티를 DTO로 변환하는 방법을 선택한다.
- 2. 필요하면 페치 조인으로 성능을 최적화 한다. 대부분의 성능 이슈가 해결된다.
- 3. 그래도 안되면 DTO로 직접 조회하는 방법을 사용한다.
- 4. 최후의 방법은 JPA가 제공하는 네이티브 SQL이나 스프링 JDBC Template을 사용해서 SQL을 직접 사용한다.

## API 개발 고급 - 컬렉션 조회 최적화

주문내역에서 추가로 주문한 상품 정보를 추가로 조회하자.
Order 기준으로 컬렉션인 OrderItem 와 Item 이 필요하다.

앞의 예제에서는 toOne(OneToOne, ManyToOne) 관계만 있었다. 이번에는 컬렉션인 일대다 관계
(OneToMany)를 조회하고, 최적화하는 방법을 알아보자.

### 주문 조회 V3: 엔티티를 DTO로 변환 - 페치 조인 최적화

- OrderApiController

``` java
  @GetMapping("/api/v3/orders")
  public List<OrderDto> ordersV3() {
       List<Order> orders = orderRepository.findAllWithItem();
       List<OrderDto> result = orders.stream()
                               .map(o -> new OrderDto(o))
                               .collect(toList());
       return result;
  }
```

- OrderRepository에

``` java
  public List<Order> findAllWithItem() {
      return em.createQuery(
                 "select distinct o from Order o" +
                 " join fetch o.member m" +
                 " join fetch o.delivery d" +
                 " join fetch o.orderItems oi" +
                 " join fetch oi.item i", Order.class).getResultList();
  }
``` 

- 페치 조인으로 SQL이 1번만 실행됨
- distinct 를 사용한 이유는 1대다 조인이 있으므로 데이터베이스 row가 증가한다. 그 결과 같은 order  
  엔티티의 조회 수도 증가하게 된다. JPA의 distinct는 SQL에 distinct를 추가하고, 더해서 같은 엔티티가  
  조회되면, 애플리케이션에서 중복을 걸러준다. 이 예에서 order가 컬렉션 페치 조인 때문에 중복 조회 되는 것을 막아준다.

- 단점
  - 페이징 불가능

### 주문 조회 V3.1: 엔티티를 DTO로 변환 - 페이징과 한계 돌파

- 컬렉션을 페치 조인하면 페이징이 불가능하다.
  - 컬렉션을 페치 조인하면 일대다 조인이 발생하므로 데이터가 예측할 수 없이 증가한다.
  - 일다대에서 일(1)을 기준으로 페이징을 하는 것이 목적이다. 그런데 데이터는 다(N)를 기준으로 row 가 생성된다.
  - Order를 기준으로 페이징 하고 싶은데, 다(N)인 OrderItem을 조인하면 OrderItem이 기준이 되어 버린다.
  - (더 자세한 내용은 자바 ORM 표준 JPA 프로그래밍 - 페치 조인 한계 참조)
- 이 경우 하이버네이트는 경고 로그를 남기고 모든 DB 데이터를 읽어서 메모리에서 페이징을 시도한다. 최악의 경우 장애로 이어질 수 있다.

> **한계 돌파**
> 그러면 페이징 + 컬렉션 엔티티를 함께 조회하려면 어떻게 해야할까?
> 지금부터 코드도 단순하고, 성능 최적화도 보장하는 매우 강력한 방법을 소개하겠다. 대부분의 페이징 + 컬렉션 엔티티 조회 문제는 이 방법으로 해결할 수 있다.

- 먼저 **ToOne**(OneToOne, ManyToOne) 관계를 모두 페치조인 한다. ToOne 관계는 row수를 증가시  
  키지 않으므로 페이징 쿼리에 영향을 주지 않는다.

- 예시 코드
``` java
  public List<Order> findAllWithItem() {

        return em.createQuery(
                "select distinct o from Order o" +
                " join fetch o.member m" +
                " join fetch o.delivery d" +
                " join fetch o.orderItems oi" +
                " join fetch oi.item i", Order.class).getResultList();
  }
``` 

- 컬렉션은 지연 로딩으로 조회 한다.
- 지연 로딩 성능 최적화를 위해 hibernate.default_batch_fetch_size , @BatchSize 를 적용한다.
  - hibernate.default_batch_fetch_size: 글로벌 설정
  - @BatchSize: 개별 최적화
  - 이 옵션을 사용하면 컬렉션이나, 프록시 객체를 한꺼번에 설정한 size 만큼 IN 쿼리로 조회한다.

- 최적화 옵션(필살기)
``` yml
  spring:
    jpa:
      properties:
        hibernate:
          default_batch_fetch_size: 1000
``` 

- 개별로 설정하려면 @BatchSize 를 적용하면 된다. (컬렉션은 컬렉션 필드에, 엔티티는 엔티티 클래스에 적용)


- 장점
  - 쿼리 호출 수가 1 + N 1 + 1 로 최적화 된다.
  - 조인보다 DB 데이터 전송량이 최적화 된다. (Order와 OrderItem을 조인하면 Order가  
    OrderItem 만큼 중복해서 조회된다. 이 방법은 각각 조회하므로 전송해야할 중복 데이터가 없다.)
  - 페치 조인 방식과 비교해서 쿼리 호출 수가 약간 증가하지만, DB 데이터 전송량이 감소한다.
  - 컬렉션 페치 조인은 페이징이 불가능 하지만 이 방법은 페이징이 가능하다.
- 결론
  - ToOne 관계는 페치 조인해도 페이징에 영향을 주지 않는다. 따라서 ToOne 관계는 페치조인으로 쿼리 수를 줄이고 해결하고,  
    나머지는 hibernate.default_batch_fetch_size 로 최적화 하자.

> 참고: default_batch_fetch_size 의 크기는 적당한 사이즈를 골라야 하는데, 100~1000 사이를 선택
> 하는 것을 권장한다. 이 전략을 SQL IN 절을 사용하는데, 데이터베이스에 따라 IN 절 파라미터를 1000으
> 로 제한하기도 한다. 1000으로 잡으면 한번에 1000개를 DB에서 애플리케이션에 불러오므로 DB에 순간
> 부하가 증가할 수 있다. 하지만 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 하므로
> 메모리 사용량이 같다. 1000으로 설정하는 것이 성능상 가장 좋지만, 결국 DB든 애플리케이션이든 순간 부
> 하를 어디까지 견딜 수 있는지로 결정하면 된다


### 주문 조회 V4: JPA에서 DTO 직접 조회

- OrderApiController
``` java
  @GetMapping("/api/v4/orders")
  public List<OrderQueryDto> ordersV4() {
      return orderQueryRepository.findOrderQueryDtos();
  }

``` 

``` java
  @Repository
  @RequiredArgsConstructor
  public class OrderQueryRepository {
  
      private final EntityManager em;
  
      public List<OrderQueryDto> findOrderQueryDtos() {
          List<OrderQueryDto> result = findOrders();
  
          result.forEach(o -> {
              List<OrderItemQueryDto> orderItems = findOrderItems(o.getOrderId());
              o.setOrderItems(orderItems);
          });
  
          return result;
      }
  
      private List<OrderItemQueryDto> findOrderItems(Long orderId) {
          return em.createQuery(
                  "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                          " from OrderItem oi" +
                          " join oi.item i" +
                          " where oi.order.id = :orderId", OrderItemQueryDto.class)
                  .setParameter("orderId", orderId)
                  .getResultList();
  
      }
  
      public List<OrderQueryDto> findOrders() {
          return em.createQuery(
                  "select new jpabook.jpashop.repository.order.query.OrderQueryDto(o.id, m.name, o.orderDate, o.status, d.address)" +
                          " from Order o" +
                          " join o.member m" +
                          " join o.delivery d", OrderQueryDto.class)
                  .getResultList();
      }
  }
``` 

- Query: 루트 1번, 컬렉션 N 번 실행
- ToOne(N:1, 1:1) 관계들을 먼저 조회하고, ToMany(1:N) 관계는 각각 별도로 처리한다.
  - 이런 방식을 선택한 이유는 다음과 같다.
  - ToOne 관계는 조인해도 데이터 row 수가 증가하지 않는다. ToMany(1:N) 관계는 조인하면 row 수가 증가한다.
- row 수가 증가하지 않는 ToOne 관계는 조인으로 최적화 하기 쉬우므로 한번에 조회하고, ToMany 관계는   
  최적화 하기 어려우므로 findOrderItems() 같은 별도의 메서드로 조회한다.


### 주문 조회 V5: JPA에서 DTO 직접 조회 - 컬렉션 조회 최적화

- OrderApiController
``` java
  @GetMapping("/api/v5/orders")
  public List<OrderQueryDto> ordersV5() {
      return orderQueryRepository.findAllByDto_optimization();
  }
``` 

``` java
 /**
  * 최적화
  * Query: 루트 1번, 컬렉션 1번
  * 데이터를 한꺼번에 처리할 때 많이 사용하는 방식 *
  */
  public List<OrderQueryDto> findAllByDto_optimization() {
      List<OrderQueryDto> result = findOrders();

      List<Long> orderIds = result.stream()
              .map(o -> o.getOrderId())
              .collect(Collectors.toList());

      List<OrderItemQueryDto> orderItems = em.createQuery(
                      "select new jpabook.jpashop.repository.order.query.OrderItemQueryDto(oi.order.id, i.name, oi.orderPrice, oi.count)" +
                      " from OrderItem oi" +
                      " join oi.item i" +
                      " where oi.order.id in :orderIds", OrderItemQueryDto.class)
                .setParameter("orderIds", orderIds)
                .getResultList();

      Map<Long, List<OrderItemQueryDto>> orderItemMap = orderItems.stream()
                .collect(Collectors.groupingBy(orderItemQueryDto -> orderItemQueryDto.getOrderId()));

      result.forEach(o -> o.setOrderItems(orderItemMap.get(o.getOrderId())));

      return result;
  }
``` 

- Query: 루트 1번, 컬렉션 1번
- ToOne 관계들을 먼저 조회하고, 여기서 얻은 식별자 orderId로 ToMany 관계인 OrderItem 을 한꺼번 에 조회
- map을 사용해서 매칭 성능 향상(O(1))


### API 개발 고급 정리
**정리**
- 엔티티 조회
  - 엔티티를 조회해서 그대로 반환: V1 
  - 엔티티 조회 후 DTO로 변환: V2 
  - 페치 조인으로 쿼리 수 최적화: V3 
  - 컬렉션 페이징과 한계 돌파: V3.1
    - 컬렉션은 페치 조인시 페이징이 불가능
    - ToOne 관계는 페치 조인으로 쿼리 수 최적화
    - 컬렉션은 페치 조인 대신에 지연 로딩을 유지하고, hibernate.default_batch_fetch_size , @BatchSize 로 최적화
- DTO 직접 조회
  - JPA에서 DTO를 직접 조회: V4
  - 컬렉션 조회 최적화 - 일대다 관계인 컬렉션은 IN 절을 활용해서 메모리에 미리 조회해서 최적화: V5 
  - 플랫 데이터 최적화 - JOIN 결과를 그대로 조회 후 애플리케이션에서 원하는 모양으로 직접 변환: V6

**권장 순서**
- 1. 엔티티조회방식으로우선접근
  - 1. 페치조인으로 쿼리 수를 최적화 
  - 2. 컬렉션 최적화
    - 1. 페이징 필요 hibernate.default_batch_fetch_size , @BatchSize 로 최적화
    - 2. 페이징 필요X 페치 조인 사용
- 2. 엔티티 조회 방식으로 해결이 안되면 DTO 조회 방식 사용
- 3. DTO 조회 방식으로 해결이 안되면 NativeSQL or 스프링 JdbcTemplate


> 참고: 엔티티 조회 방식은 페치 조인이나, hibernate.default_batch_fetch_size , @BatchSize 같이 
> 코드를 거의 수정하지 않고, 옵션만 약간 변경해서, 다양한 성능 최적화를 시도할 수 있다. 반면에 DTO를 직 접 조회하는 방식은 
> 성능을 최적화 하거나 성능 최적화 방식을 변경할 때 많은 코드를 변경해야 한다.


> 참고: 개발자는 성능 최적화와 코드 복잡도 사이에서 줄타기를 해야 한다. 항상 그런 것은 아니지만, 보통 
> 성능 최적화는 단순한 코드를 복잡한 코드로 몰고간다.
> 엔티티 조회 방식은 JPA가 많은 부분을 최적화 해주기 때문에, 단순한 코드를 유지하면서, 성능을 최적화 할 수 있다.
> 반면에 DTO 조회 방식은 SQL을 직접 다루는 것과 유사하기 때문에, 둘 사이에 줄타기를 해야 한다.


**DTO 조회 방식의 선택지**
- DTO로 조회하는 방법도 각각 장단이 있다. V4, V5, V6에서 단순하게 쿼리가 1번 실행된다고 V6이 항상 좋은 방법인 것은 아니다.
- V4는코드가단순하다. 특정주문한건만조회하면이방식을사용해도성능이잘나온다.예를들어서조회 한 Order 데이터가 1건이면 OrderItem을 찾기 위한 쿼리도 1번만 실행하면 된다.
- V5는 코드가 복잡하다. 여러 주문을 한꺼번에 조회하는 경우에는 V4 대신에 이것을 최적화한 V5 방식을 사 용해야 한다. 예를 들어서 조회한 Order 데이터가 1000건인데,  
  V4 방식을 그대로 사용하면, 쿼리가 총 1 + 1000번 실행된다. 여기서 1은 Order 를 조회한 쿼리고, 1000은 조회된 Order의 row 수다.  
  V5 방식 으로 최적화 하면 쿼리가 총 1 + 1번만 실행된다. 상황에 따라 다르겠지만 운영 환경에서 100배 이상의 성 능 차이가 날 수 있다.
- V6는 완전히 다른 접근방식이다. 쿼리 한번으로 최적화 되어서 상당히 좋아보이지만, Order를 기준으로 페 이징이 불가능하다. 실무에서는 이정도 데이터면 수백이나,  
  수천건 단위로 페이징 처리가 꼭 필요하므로, 이 경우 선택하기 어려운 방법이다. 그리고 데이터가 많으면 중복 전송이 증가해서 V5와 비교해서 성능 차이도 미비하다.

## API 개발 고급 - 실무 필수 최적화 

### OSIV와 성능 최적화

- Open Session In View: 하이버네이트 
- Open EntityManager In View: JPA (관례상 OSIV라 한다.)

![](https://github.com/dididiri1/jpashop/blob/main/study/images/05_02.png?raw=true)

- spring.jpa.open-in-view : **true** 기본값


이 기본값을 뿌리면서 애플리케이션 시작 시점에 warn 로그를 남기는 것은 이유가 있다.
OSIV 전략은 트랜잭션 시작처럼 최초 데이터베이스 커넥션 시작 시점부터 API 응답이 끝날 때 까지 영속성 컨텍스트와 데이터베이스 커넥션을 유지한다. 
그래서 지금까지 View Template이나 API 컨트롤러에서 지 연 로딩이 가능했던 것이다.
지연 로딩은 영속성 컨텍스트가 살아있어야 가능하고, 영속성 컨텍스트는 기본적으로 데이터베이스 커넥션 을 유지한다. 이것 자체가 큰 장점이다.


그런데 이 전략은 너무 오랜시간동안 데이터베이스 커넥션 리소스를 사용하기 때문에, 실시간 트래픽이 중 요한 애플리케이션에서는 커넥션이 모자랄 수 있다. 이것은 결국 장애로 이어진다.
예를 들어서 컨트롤러에서 외부 API를 호출하면 외부 API 대기 시간 만큼 커넥션 리소스를 반환하지 못하 고, 유지해야 한다.

![](https://github.com/dididiri1/jpashop/blob/main/study/images/05_03.png?raw=true)

- spring.jpa.open-in-view: **false** OSIV 종료

OSIV를 끄면 트랜잭션을 종료할 때 영속성 컨텍스트를 닫고, 데이터베이스 커넥션도 반환한다. 
따라서 커넥션 리소스를 낭비하지 않는다.


OSIV를 끄면 모든 지연로딩을 트랜잭션 안에서 처리해야 한다. 따라서 지금까지 작성한 많은 지연 로딩 
코드를 트랜잭션 안으로 넣어야 하는 단점이 있다. 그리고 view template에서 지연로딩이 동작하지 않는다. 
결론적으로 트랜잭션이 끝나기 전에 지연 로딩을 강제로 호출해 두어야 한다.

- 커멘드와 쿼리 분리
실무에서 **OSIV를 끈 상태**로 복잡성을 관리하는 좋은 방법이 있다. 바로 Command와 Query를 분리하는 것이다.

참고: https://en.wikipedia.org/wiki/Command–query_separation

보통 비즈니스 로직은 특정 엔티티 몇 개를 등록하거나 수정하는 것이므로 성능이 크게 문제가 되지 않는다. 그런데 복잡한 화면을 출력하기 위한 쿼리는 화면에 맞추어 성능을 최적화 하는 것이 중요하다. 하지만 그 복 잡성에 비해 핵심 비즈니스에 큰 영향을 주는 것은 아니다.
그래서 크고 복잡한 애플리케이션을 개발한다면, 이 둘의 관심사를 명확하게 분리하는 선택은 유지보수 관 점에서 충분히 의미 있다.
단순하게 설명해서 다음처럼 분리하는 것이다.

- OrderService
  - OrderService: 핵심 비즈니스 로직
  - OrderQueryService: 화면이나 API에 맞춘 서비스 (주로 읽기 전용 트랜잭션 사용)

보통 서비스 계층에서 트랜잭션을 유지한다. 두 서비스 모두 트랜잭션을 유지하면서 지연 로딩을 사용할 수 있다.

> 참고: 필자는 고객 서비스의 실시간 API는 OSIV를 끄고, ADMIN 처럼 커넥션을 많이 사용하지 않는 곳에
서는 OSIV를 켠다.


> 참고: OSIV에 관해 더 깊이 알고 싶으면 자바 ORM 표준 JPA 프로그래밍 13장 웹 애플리케이션과 영속성
관리를 참고하자.

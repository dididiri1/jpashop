

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
                "select new jpabook.jpashop.repository.OrderSimpleQueryDto(o.id, m.name, o.orderDate, o.status, m.address) from Order o " +
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
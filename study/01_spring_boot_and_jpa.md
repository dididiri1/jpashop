# 실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발

## 프로젝트 생성

### 스프링 부트 스타터(https://start.spring.io/)

* 사용 기능(Dependencies)

  * Spring Web
  * thymeleaf
  * Spring Data JPA
  * h2
  * Lombok
    * groupId: japbook
    * artifactId: jpashop



- build.gradle 설정

``` java
plugins {
  id 'org.springframework.boot' version '2.4.1'
  id 'io.spring.dependency-management' version '1.0.10.RELEASE'
  id 'java'
}

group = 'jpabook'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '11'
        
configurations {
  compileOnly {
    extendsFrom annotationProcessor
  }
}

repositories {
  mavenCentral()
}
```


### H2. 데이터 베이스 설치

* [https://www.h2database.com](https://www.h2database.com/)

* 다운로드 및 설치
* 데이터베이스 파일 생성 방법
  * .199 버전으로 설치하자.
  * 처음에는 파일모드 : `jdbc:h2:~/jpashop3` 처럼 생성해서 `C:\Users\<userName>` 내부에 .mv.db 파일을 생성해야 한다.
    * 그 다음부터는 네트웍 모드 :  `jdbc:h2:tcp://localhost/~/jpashop3` 으로 접속하자.

> 참고: H2 데이터베이스의 MVCC 옵션은 H2 1.4.198 버전부터 제거되었습니다. 사용 버전이 1.4.199이므로 옵션 없이 사용하면 됩니다.
>
> 추가로 1.4.200 버전에서는 MVCC 옵션을 사용하면 오류가 발생합니다.

### 5. JPA와 DB 설정, 동작확인

* application.yml

- application.yml

```yml
spring:
  datasource:
    url: jdbc:h2:tcp://localhost/~/jpashop3;MVCC=TRUE
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        # show_sql: true
        format_sql: true
logging.level:
  org.hibernate.SQL: debug

```

- spring.jpa.hibernate.ddl-auto: create
  - 이 옵션은 애플리케이션 실행 시점에 테이블을 drop 하고, 다시 생성한다.

## 실제 동작하는지 확인하기

- 회원 엔티티

``` java

  @Entity
  @Getter @Setter
  public class Member {
  
      @Id @GeneratedValue
      private Long id;
  
      private String username;
  
      ...
  }

```

- 회원 리파지토리

``` java

  @Repository
  public class MemberRepository {
  
      @PersistenceContext
      private EntityManager em;
  
      public Long save(Member member) {
          em.persist(member);
  
          return member.getId();
      }
  
      public Member find(Long id) {
          return em.find(Member.class, id);
      }
  }

```

- 테스트 케이스

``` java

  @SpringBootTest
  public class MemberRepositoryTest {
  
      @Autowired MemberRepository memberRepository;
  
      @Test
      @Transactional
      @Rollback(value = false)
      public void testMember() throws Exception {
          //given
          Member member = new Member();
          member.setUsername("memberA");
  
          //when
          Long saveId = memberRepository.save(member);
          Member findMember = memberRepository.find(saveId);
  
          //then
          assertThat(findMember.getId()).isEqualTo(member.getId());
          assertThat(findMember.getUsername()).isEqualTo(member.getUsername());
          Assertions.assertThat(findMember).isEqualTo(member);
  
          System.out.println("(findMember == member) = " + (findMember == member));
          
          ...
  
      }
  }

```

## 도메인 분석 설계

### 요구사항 분석

### **기능 목록**
- 회원 기능
  - 회원 등록
  - 회원 조회
- 상품 기능
  - 상품 등록
  - 상품 수정
  - 상품 조회
- 주문 기능
  - 상품 주문
  - 주문 내역 조회
  - 주문 취소
- 기타 요구사항
  - 상품은 재고 관리가 필요하다.
  - 상품의 종류는 도서,음반,영화가 있다.
  - 상품을 카테고리로 구분할 수 있다.
  - 상품 주문시 배송 정보를 입력할 수 있다.


- 도메인 모델과 테이블 설계
![](https://github.com/dididiri1/jpashop/blob/main/study/images/02_02.png?raw=true)

- **회원,주문,상품의 관계**: 회원은 여러 상품을 주문할 수 있다. 그리고 한 번 주문할 때 여러 상품을 선택할 수 있으므로 주문과 상품은 다대다 관계다.
  하지만 이런 다대다 관계는 관계형 데이터베이스는 물론이고 엔티티에서도 거의 사용하지 않는다. 따라서 그림처럼 주문상품이라는 엔티티를 추가해서 다대다 관계를
  일대다, 다대일 관계로 풀어냈다.
- **상품 분류: 상품은 도서, 음반, 영화로 구붕되는데 상품이라는 공통 속성을 사용하므로 상속 구조로 표현했다.

- 회원 에티티 분석

![](https://github.com/dididiri1/jpashop/blob/main/study/images/02_03.png?raw=true)

- **회원(Member):** 이름과 임베디드 타입과 주소(Address), 그리고 주문(orders) 리스트를 가진다.

- **주문(OrderItem):** 주문한 상품 정보와 주문 금액(orderPrice), 주문 수량(count) 정보를 가지고 있다.
  상품의 종류로는 도서, 음반, 영화가 있늗데 각각은 사용하는 속성이 조금씩 다른다.

- **상품(Item):** 이름, 가격, 재고수량( stockQuantity )을 가지고 있다. 상품을 주문하면 재고수량이 줄어든 다. 상품의 종류로는 도서, 음반, 영화가 있는데 각각은 사용하는 속성이 조금씩 다르다.

- **배송(Delivery):** 주문시 하나의 배송 정보를 생성한다. 주문과 배송은 일대일 관계다. 

- **카테고리(Category):** 상품과 다대다 관계를 맺는다. parent , child 로 부모, 자식 카테고리를 연결한다.

- **주소(Address):** 값 타입(임베디드 타입)이다. 회원과 배송(Delivery)에서 사용한다.

>
>참고: 회원 엔티티 분석 그림에서 Order와 Delivery가 단방향 관계로 잘못 그려져 있다. 양방향 관계가 맞 다.
>

>
>참고: 회원이 주문을 하기 때문에, 회원이 주문리스트를 가지는 것은 얼핏 보면 잘 설계한 것 같지만, 객체 세상은
>     실제 세계와는 다르다. 실무에서는 회원이 주문을 참조하지 않고, 주문이 회원을 참조하는 것으로 충분하다.
>     여기서는 일대다, 다대일 양방향 연관관계를 설명하기 위해서 추가했다.
> 

- 회원 테이블 분석

![](https://github.com/dididiri1/jpashop/blob/main/study/images/02_04.png?raw=true)

- **MEMBER**: 회원 엔티티의 Address 임베디드 타입 정보가 회원 테이블에 그대로 들어갔다. 이것은
  DELIVERY 테이블도 마찬가지다.
- **ITEM**: 앨범, 도서, 영화 타입을 통합해서 하나의 테이블로 만들었다. DTYPE컬럼으로 타입을 구분한다.

>
> 참고: 테이블명이 ORDER가 아니라 ORDERS인 것은 데이터베이스가 order by 예약어가 있어서 그럼~
> 

>
> 참고: 실제 코드에서는 DB에 소문자 + _ (언더스코어) 스타일을 사용함.
> 데이터베이스 테이블명, 컬럼명에 대한 관례는 회사마다 다르다. 보통은 대문자 + _(언더스코어어)나 소문자
> + _(언더스코어) 방식 중에 하나를 지정해서 일관성 있게 사용한다. 
> 

### 연관관계 매핑 분석


### 엔티티 설계시 주의점

- 엔티티에는 가급적 Setter를 사용하지 말자
  - 변경 포인트가 너무 많아서, 유지보수가 어렵다.
- 모든 연관관계는 지연로딩으로 설정!
  - 즉시로딩(EAGER)은 예측이 어렵고, 어떤SQL이 실핼될지 추척하기 어렵다. 특히 JPQL을 실행할 때 N+1 문제
    가 자주 발생한다.
  - 실무에서 모든 연관관계는 지연로딩(LAZY)으로 설정해야 한다.
  - 연관된 엔티티를 함께 DB에서 조회해야 하면, fetch join 또는 엔티티 그래프 기능을 사용한다.
  - @XToOne(OneToOne, ManyToOne) 관계는 기본이 즉시로딩으로 설정 되어 있어서 지연으로 설정 해야됨.
- 컬렉션은 필드에서 초기화 하자
  - 컬렉션은 필드에서 바로 초기화 하는 것이 안전하다.
  - null 문제에서 안전하다.
  - 하이버네이트는 엔티티를 영속화 할 때, 컬렉션을 감싸서 하이버네이트가 제공하는 내정 컬렉션으로 변경한다. 만약
    getOrders() 처럼 임의의 메서드에서 컬렉션을 잘못 생성하면 하이버네이트 내부 메커니즘에 문제가 발생할 수 있다.
    따라서 필드레벨에서 생성하는 것이 가장 안전하고, 코드도 간결하다.

### 애플리케이션 아키텍처

- 계층형 구조 사용
  - controller, web: 웹 계층
  - service: 비즈니스 로직, 트랜잭션 처리
  - repository: JPA를 직접 사용하는 계층, 엔티티 매니저 사용
  - domain: 엔티티가 모여 있는 계층, 모든 계층에서 사용

- 패키지 구조
  - jpabook.jpashop
    - domain
    - exception
    - repository
    - service
    - web
    
### 개발 순서: 서비스, 리포지토리 계층을 개발하고, 테스트 케이스를 작성해서 검증, 마지막 웹 계층 적용


### @RequiredArgsConstructor
- final 혹은 @NotNull이 붙은 필드의 생성자를 자동으로 만들어준다.

``` java
@Service
@RequiredArgsConstructor // final인 애를 생성자로 만들어줌
public class MemberService {

    private final MemberRepository memberRepository;


    /*public MemberService(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }*/
    
    ```
}
```

### 테스트 케이스


### 메모리 DB

- test > resources > application.yml 테스트 케이스할때 main 꺼보다 우선시 된다.

- 메모리DB 모드 설정 
``` yml
  spring:
  datasource:
    url: jdbc:h2:mem:test
    username: sa
    password:
    driver-class-name: org.h2.Driver
```

- 위에 설정이 없어도 스프링부트는 기본적으로 메모리 모드 해준다.

## 상품 엔티티 개발(비즈니스 로직추가)

### 비즈니스 로직

- Service 쪽에서 로직을 처리 했는데 객체지향적으로 보면 데이터를
  가지고 있는 쪽에 엔티티에서 비즈니스로직을 처리하는것이 가장 좋음 응집력이 있음
- @Setter 없이 값을 넘을떄에는 비즈니스 매소드로 처리하는 것이 바람직하다.

``` java

  @Entity
  @Inheritance(strategy = InheritanceType.SINGLE_TABLE)
  @DiscriminatorColumn(name = "dtype")
  @Getter @Setter
  public abstract class Item {
  
      @Id @GeneratedValue
      @Column(name = "item_id")
      private Long id;

      ``` 
    
      // == 비즈니스 로직 == //

      /**
       * stock 증가
       */
      public void addStock(int quantity) {
          this.stockQuantity += quantity;
      }

      /**
       * stock 감소
       */
      public void removeStock(int quantity) {
          int restStock = this.stockQuantity - quantity;
          if (restStock < 0) {
              throw new NotEnoughStockException("need more stock");
          }
          this.stockQuantity = restStock;
      }
      
      ``` 
  }  

``` 
- 예외를 처리하기 위해 NotEnoughStockException 클래스를 만들고 RuntimeException을 상속받아 Override 함. 

``` java
  public class NotEnoughStockException extends  RuntimeException {

      public NotEnoughStockException() {
          super();
      }
  
      public NotEnoughStockException(String message) {
          super(message);
      }
  
      ``` 
  }
``` 

## 상품 리포지토리 개발

- id가 없으면 신규로 보고 persist() 실행
- id가 있으면 데이터베이스에 저장된 엔티티라고 보고, merge()를 실행

``` java

@Repository
@RequiredArgsConstructor
public class ItemRepository {

    private final EntityManager em;

    /**
     * 상품 저장
     */
    public void save(Item item) {
        if (item.getId() == null) {
            em.persist(item);
        } else {
            em.merge(item);
        }
    }

    /**
     * 상품 단건 조회
     */
    public Item findOne(Long id) {
        return em.find(Item.class, id);
    }

    /**
     * 상품 전체 조회
     */
    public List<Item> findAll() {
        return em.createQuery("select i from Item i", Item.class).getResultList();
    }
    
    ```    
}

``` 

- **생성 메서드(createOrder()):** 주문 엔티티를 생성할 때 사용한다. 주문회원, 배송정보, 주문상품의 정보를 받아서
  실제 주문 엔티티를 생성한다.

``` java
    public static Order createOrder(Member member, Delivery delivery, OrderItem... orderItems) {
        Order order = new Order();
        order.setMember(member);
        order.setDelivery(delivery);

        for (OrderItem orderItem : orderItems) {
            order.addOrderItem(orderItem);
        }
        order.setStatus(OrderStatus.ORDER);
        order.setOrderDate(LocalDateTime.now());
        return order;
    }
``` 

- **주문 취소(cancel()):** 주문 취소시 사용한다. 주문 상태를 취소로 변경하고 주문상품에 주문 취소를 알린다.
  만약 이미 배송을 완료한 상품이면 주문을 취소하지 못하도록 예외를 발생시킨다.

``` java
    public void cancel() {
        if (delivery.getStatus() == DeliveryStatus.COMP) {
            throw new IllegalStateException("이미 배송안료된 상품은 취소가 불가능합니다.");
        }

        this.setStatus(OrderStatus.CANSEL);
        for (OrderItem orderItem : orderItems) {
            orderItem.cancel();
        }
    }

```

- **전체 주문 가격조회:** 주문 시 사용한 전체 주문 가격을 조회한다. 전체 주문 가격을 알려면 각각의 주문상품 가격을 알아야 한다.
  로직을 보면 연관된 주문상품들의 가격을 조회해서 더한 값을 반환한다. (실무에서는 주로 주문에 전체 주문 가격 필드를 두고 역정규화 한다.)

``` java
    public int getTotalPrice() {
        int totalPrice = 0;
        for (OrderItem orderItem : orderItems) {
            totalPrice += orderItem.getTotalPrice();
        }
        return totalPrice;
    }
``` 

- **무변별한 생성자 막기**

``` java
  @NoArgsConstructor(access = AccessLevel.PROTECTED)
  
  // 같은 의미
  
  protected OrderItem() {
        
  }
```

>
> 참고: 주문 서비스의 주문과 주문 취소 메서드를 보면 비즈니스 로직 대부분이 엔티티에 있다. 서비스 계층 
> 은 단순히 엔티티에 필요한 요청을 위임하는 역할을 한다. 이처럼 엔티티가 비즈니스 로직을 가지고 객체 지향의
> 특성을 적극 활용하는 것을 **도메인 모델 패턴**(http://martinfowler.com/eaaCatalog/domainModel.html)이라고 한다. 
> 반대로 엔티티에는 비즈니스 로직이 거의 없고 서비스 계층에서 대부분의 로직을 처리하는 것을 트랙잭션 스크립트 패턴
> (http://martinfowler.com/eaaCatalog/transactionScript.html)이라 한다.
> 



>
> 참고: 뷰 템플릿 변경사항을 서버 재시작 없이 즉시반영하기
> 

- 1. spring-boot-devtools 추가
- 2. html 파일 build > Recomplie

``` yml
dependencies {
    implementation 'org.springframework.boot:spring-boot-devtools'
}
``` 

### 타임리프 ? 기능

- member.address?.city 해주면 null이면 값이 안나오게 됨. If문 안써도 되는 장점이 있음.

``` html
  <tr th:each="member : ${members}">
      <td th:text="${member.id}"></td>
      <td th:text="${member.name}"></td>
      <td th:text="${member.address?.city}"></td>
      <td th:text="${member.address?.street}"></td>
      <td th:text="${member.address?.zipcode}"></td>
  </tr>

``` 

### Member 엔티티와 MemberForm의 분리

- 엔티티는 핵심 비즈니스 로직만 가지고 있고 화면에 대한 로직은 없어야됨.
- 화면에 맞는 로직은 dto나 form 객체로 데이터로 전달하는 것으로 사용하는것이 좋음.

``` java
  @Getter @Setter
  public class MemberForm {
  
      @NotEmpty(message = "회원 이름은 필수입니다.")
      private String name;
  
      private String city;
      private String street;
      private String zipcode;
      
      ```
  }
```

- 실무에서는 엔티티를 직접 뿌리기 보다는 dto로 처리

``` java
  @GetMapping("/members")
  public String list(Model model) {
      List<Member> members = memberService.findMembers();
      model.addAttribute("members", members);

      return "members/memberList";
        
  }
``` 

### 변경 감지와 병합(merge)


- 준영속 엔티티?
영속성 컨텍스트가 더는 관리하지 않는 엔티티를 말한다.
(여기서는 itemService.saveItem(book) 에서 수정을 시도하는 Book 객체다. Book 객체는 이미 DB 에 한번 저장되어서 식별자가 존재한다.
이렇게 임의로 만들어낸 엔티티도 기존 식별자를 가지고 있으면 준 영속 엔티티로 볼 수 있다.)

- 준영속 엔티티를 수정하는 2가지 방법
  - 변경 감지 기능 사용
  - 병합 사용

- **변경 감지 기능 사용**
``` java
@Transactional
void update(Item itemParam) { //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티
    
    Item findItem = em.find(Item.class, itemParam.getId()); //같은 엔티티를 조회한다. 영속상태 JPA에서 관리함.

    findItem.setPrice(itemParam.getPrice()); //데이터를 수정한다. 
    findItem.setName(itemParam.getName());
    findItem.setStockQuantity(itemParam.getStockQuantity());
    
    ```
}

``` 


영속성 컨텍스트에서 엔티티를 다시 조회한 후에 데이터를 수정하는 방법
트랜잭션 안에서 엔티티를 다시 조회, 변경할 값 선택 트랜잭션 커밋 시점에 변경 감지(Dirty Checking) 
이 동작해서 데이터베이스에 UPDATE SQL 실행

- **병합 사용**
  - 병합은 준영속 상태의 엔티티를 영속 상태로 변경할 때 사용하는 기능이다.

``` java

@Transactional
void update(Item itemParam) { //itemParam: 파리미터로 넘어온 준영속 상태의 엔티티 
    
    Item mergeItem = em.merge(itemParam);
    
}

``` 

![](https://github.com/dididiri1/jpashop/blob/main/study/images/05_01.png?raw=true)

### 병합 동작 방식

- 1. merge()를실행한다.
- 2. 파라미터로 넘어온 준영속 엔티티의 식별자 값으로 1차 캐시에서 엔티티를 조회한다.
   2-1. 만약 1차 캐시에 엔티티가 없으면 데이터베이스에서 엔티티를 조회하고, 1차 캐시에 저장한다.
- 3. 조회한 영속 엔티티( mergeMember )에 member 엔티티의 값을 채워 넣는다. (member 엔티티의 모든 값
   을 mergeMember에 밀어 넣는다. 이때 mergeMember의 “회원1”이라는 이름이 “회원명변경”으로 바뀐다.)
- 4. 영속 상태인 mergeMember를 반환한다.

>
> 참고: 책 자바 ORM 표준 JPA 프로그래밍 3.6.5
> 

### 병합시 동작 방식을 간단히 정리

- 1. 준영속 엔티티의 식별자 값으로 영속 엔티티를 조회한다.
- 2. 영속 엔티티의 값을 준영속 엔티티의 값으로 모두 교체한다.(병합한다.)
- 3. 트랜잭션 커밋 시점에 변경 감지 기능이 동작해서 데이터베이스에 UPDATE SQL이 실행

>
> 주의: 변경 감지 기능을 사용하면 원하는 속성만 선택해서 변경할 수 있지만, 병합을 사용하면 모든 속성이
> 변경된다. 병합시 값이 없으면 null 로 업데이트 할 위험도 있다. (병합은 모든 필드를 교체한다.)
>

## 가장 좋은 해결 방법

### 엔티티를 변경할 때는 항상 변경 감지 사용할것! (merge를 안쓰면 됨.)

- 컨트롤러에서 어설프게 엔티티를 생성하지 말기.
- 트랙잭션이 있는 서비스 계층에 식별자(id)와 변경할 데이터를 명확하게 전달할것. (파라미터 or dto)
- 트랙잰션이 있는 서비스 계층에서 영속 상태의 엔티티를 조회하고, 엔티티의 데이터를 직접 변강할것
- 트랙잭션 커밋 시점에 변경 감지가 실행된다.

``` java
@Controller
@RequiredArgsConstructor
public class ItemController {

    ``` 

    @PostMapping("/items/{itemId}/edit")
    public String updateItem(@ModelAttribute("form") BookForm form, @PathVariable Long itemId) {

        itemService.updateItem(itemId, form.getName(), form.getPrice(), form.getStockQuantity());

        return "redirect:/items";

    }
  
    ``` 
}  
``` 

``` java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class ItemService {

    ``` 

    @Transactional
    public void updateItem(Long itemId, String name, int price, int stockQuantity) {
        Item findItem = itemRepository.findOne(itemId); // 영속 상태
        findItem.setName(name);
        findItem.setPrice(price);
        findItem.setStockQuantity(stockQuantity);
     
    }
  
    ``` 
}  
``` 

### @ModalAttribute 

- @ModalAttribute 지정되면 modal.addAttribute 자동으로 담겨있어서 사용할 수 있음

``` java
  @GetMapping("/orders")
  public String orderList(@ModelAttribute("orderSearch") OrderSearch orderSearch, Model model) {
      List<Order> orders = orderService.findOrders(orderSearch);
      model.addAttribute("orders", orders);
        
      //model.addAttribute("orderSearch", orderSearch);

      return "/order/orderList";
  }
``` 




### Reference

- [실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8-JPA-%ED%99%9C%EC%9A%A9-1/dashboard)
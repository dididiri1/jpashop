

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

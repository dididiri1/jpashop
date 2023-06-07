# 실전! 스프링 부트와 JPA 활용1 - 웹 애플리케이션 개발 /By. 김영한

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
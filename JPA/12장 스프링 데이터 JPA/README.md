# 스프링 데이터 JPA

- 대부분의 `데이터 접근 계층(Data Access Layer)`은 일명 `CRUD`로 부르는 유사한 등록, 수정, 삭제, 조회 코드를 반복해서 개발
    
    → `JPA`를 사용해서 데이터 접근 계층을 개발할 때도 이 같은 문제가 발생
    

## 스프링 데이터 JPA 소개

- `스프링 데이터 JPA`는 스프링 프레임워크에서 JPA를 편리하게 사용할 수 있도록 지원하는 프로젝트
- 데이터 접근 계층을 개발할 때 지루하게 반복되는 `CRUD` 문제를 세련된 방법으로 해결
    - `CRUD`를 처리하기 위한 공통 인터페이스를 제공
    - 레포지토리를 개발할 때 인터페이스만 작성하면 실행 시점에 스프링 데이터 `JPA`가 구현 객체를 동적으로 생성해서 주입
    
    **→ 데이터 접근 계층을 개발할 때 구현 클래스 없이 인터페이스만 작성해도 개발을 완료할 수 있음**
    

### 스프링 데이터 프로젝트

- **스프링 데이터 JPA**는 스프링 데이터 프로젝트의 하위 프로젝트 중 하나
- `스프링 데이터(Spring Data)` 프로젝트는 JPA, 몽고DB, NEO4J, REDIS, HADOOP, GEMFIRE 같은 다양한 데이터 저장소에 대한 접근을 추상화해서 개발자 편의를 제공하고 지루하게 반복하는 데이터 접근 코드를 줄여줌
- 스프링 데이터 JPA 프로젝트는 `JPA`에 특화된 기능을 제공

## 스프링 데이터 JPA 설정

- 필요 라이브러리 : spring-data-jpa
- 환경 설정
    - 스프링 설정에 `XML`을 사용하면 `<jpa:repositories>`를 사용하고 레포지토리를 검색할 `base-package`를 적는다
- 스프링 데이터 JPA는 애플리케이션을 실행할 때 basePackage에 있는 레포지토리 인터페이스들을 찾아서 해당 인터페이스를 찾아서 해당 인터페이스를 구현한 클래스를 동적으로 생성한 다음 스프링 빈으로 등록
    
    → 개발자가 직접 구현 클래스를 안만들어도 된다!
    

## 공통 인터페이스 기능

- 스프링 데이터 JPA를 사용하는 가장 단순한 방법은 `JpaRepository` 인터페이스를 상속받는 것
- 제네릭에 엔티티 클래스와 엔티티 클래스가 사용하는 식별자 타입을 지정

![image](https://user-images.githubusercontent.com/31034469/161358018-0163df28-dc4d-4c95-a91f-b31fc1afe7c6.png)

- 스프링 데이터 프로젝트가 공통으로 사용하는 인터페이스 : Repository, CrudRepository, PagingAndSortingRepository
- JpaRepository를 상속받으면 사용할 수 있는 주요 메소드(T : Entity, ID : 식별자, S : 엔티티와 그 자식 타입)
    - save(S) : 새로운 엔티티는 저장하고 이미 있는 엔티티는 수정
    - delete(T) : 엔티티 하나를 삭제. 내보에서 EntityManager.remove()를 호출
    - findOne(ID) : 엔티티 하나를 조회. 내부에서 EntityManager.find()를 호출
    - getOne(ID) : 엔티티를 프록시로 조회. 내부에서 EntityManager.getReference()를 호출
    - findAll(...) : 모든 엔티티를 조회. 정렬(sort)이나 페이징(pageable) 조건을 파라미터로 제공할 수 있음
    

## 쿼리 메소드 기능

- 메소드 이름만으로 쿼리를 생성하는 기능이 있는데 인터페이스에 메소드만 선언하면 해당 메소드의 이름으로 적절한 JPQL 쿼리를 생성해서 실행
    - 메소드 이름으로 쿼리 생성
    - 메소드 이름으로 JPA NamedQuery 호출
    - @Query 어노테이션을 사용해서 레포지토리 인터페이스에 쿼리 직접 정의

### 메소드 이름으로 쿼리 생성

- 인터페이스에 정의한 메소드를 실행하면 스프링 데이터 JPA는 메소드 이름을 분석해서 JPQL을 생성하고 실행
    
    → List<Member> findByEmailAndName(String email, String name);
    
- 정해진 규칙에 따라 메소드 이름을 지어야 함
- 엔티티의 필드명이 변경되면 인터페이스에 정의한 메소드 이름도 반드시 함께 변경해야하는데, 그렇게 하지 않으면 애플리케이션을 시작하는 시점에 오류가 발생

### JPA NamedQuery

- 메소드 이름으로 JPA Named 쿼리를 호출하는 기능을 제공
- 쿼리에 이름을 부여해서 사용하는 방법

### @Query, 레포지토리 메소드에 쿼리 정의

- 실행할 메소드에 정적 쿼리를 직접 작성하므로 이름 없는 Named 쿼리라 할 수 있음
- JPA Named 쿼리처럼 애플리케이션 실행 시점에 문법 오류를 발견할 수 있는 장점이 있음

### 파라미터 바인딩

- 스프링 데이터 JPA는 **위치 기반 파라미터 바인딩**과 **이름 기반 파라미터 바인딩**을 모두 지원 (기본값 : 위치 기반)
- 코드 가독성과 유지보수를 위해 이름 기반 파라미터 바인딩을 사용

```sql
select m from Memeber m where m.username = ?1 // 위치 기반
select m from Memeber m where m.username = :name // 이름 기반
```

### 벌크성 수정 쿼리

```java
// JPA 를 사용한 벌크성 수정 쿼리
int bulkPriceUp(String stockAmout){
  ...
  String sqlString = "update Product p set p.price = p.price * 1.1 where p.stockAmout < :stockAmout";
      
  int resultCount = em.createQuery(sqlString)
        .setParameter("stockAmout", stockAmout)
        .executeUpdate();
}

// 스프링 데이터 JPA 를 사용한 벌크성 수정 쿼리
@Modifying
@Query("update Product p set p.price = p.price * 1.1 where p.stockAmout < :stockAmout")
int bulkPriceUp(@Param("stockAmout") String stockAmout);
```

### 반환 타입

- 결과가 한건 이상이면 `컬렉션 인터페이스`를 사용하고, 단건이면 `반환 타입`을 지정
- 단건을 조회할 때 `NoResultException` 예외가 발생하면 예외를 무시하고 대신에 `null` 을 반환

```java
List<Member> findByName (String name); // 컬렉션
Member findByEmail (String email); // 단건
```

### 페이징과 정렬

- org.springframework.data.domain.Sort : 정렬 기능
- org.springframework.data.domain.Pageable : 페이징 기능(내부에 Sort 포함)

```java
// 페이징과 정렬 사용 예제
Page<Member> findByName(String name, Pageable pageable);
List<Member> findByName(String name, Pageable pageable);
List<Member> findByName(String name, Sort sort);

// Page 사용 예제 정의 코드
public interface MemberRepository extends Repository<Member, Long> {
  Page<Member> findByNameStartingWith(String name, Pageable pageable);
}

// Page 사용 예제 실행 코드
PageRequest pageRequest = new PageRequest(0, 10, new Sort(Direction.DESC, "name"));
Page<Member> result = memberRepository.findByNameStartingWith("김", pageRequest);

List<Member> members = result.getContent();
int totalPage = result.getTotalPages();
boolean hasNextPage = result.hasNextPage();
```

## 명세

- 명세를 이해하기 위한 핵심 단어는 술어 인데 이것은 단순히 참이나 거짓으로 평가
- AND, OR 같은 연산자로 조합할수 있고, 데이터를 검색하기 위한 제약 조건 하나하나를 술어라 할 수 있음

## 사용자 정의 레포지토리 구현

- 다양한 이유로 인터페이스만 정의하지않고 메소드를 직접 구현해야 할 때가 있는데 스프링 데이터 JPA는 이런 문제를 우회해서 필요한 메소드만 구현할 수 있는 방법을 제공
- 사용자 정의 인터페이스를 먼저 작성하는데 이름은 자유롭게 지으면 됨
- 사용자 정의 인터페이스를 구현한 클래스를 작성하는데 이름 규칙은 레포지토리 인터페이스 이름 + Impl로 지어야 하고 이렇게 하면 스프링 데이터 JPA 가 사용자 정의 구현 클래스로 인식

## Web 확장

- 식별자로 도메인 클래스를 바로 바인딩해주는 도메인 클래스 컨버터 기능과, 페이징과 정렬 기능

```java
@Configuration
@EnableWebMvc
@EnableSpringDataWebSupport
public class WebAppConfig{
  ...
}
```

## 스프링 데이터 JPA와 QueryDSL 통합

### **QueryDslPredicateExecutor 사용**

- 다음처럼 레포지토리에서 QueryDslPredicateExecutor를 상속받으면 된다.

```java
public interfavce ItemRepository extends JpaRepository<Item, Long>, QueryDslPredicateExecutor<Item> {}

// QueryDSL 사용 예제
QItem item = QItem.item;
Iterable<Item> result = itemRepository.findAll(
item.name.contains("장난감").and(item.price.between(1000,2000))
);
```

- QueryDslPredicateExecutor 인터페이스를 보면 QueryDSL을 검색조건으로 사용하면서 스프링 데이터 JPA가 제공하는 페이징 정렬 기능도 함께 사용할 수 있다.

```java
// QueryDslPredicateExecutor 인터페이스
public interface QueryDslPredicateExecutor<T> {
    T findOne(Predicate predicate);
    Iterable<T> findAll(Predicate predicate);
    Iterable<T> findAll(Predicate predicate, OrderSpecifier<?>... order);
    Page<T> findAll(Predicate predicate, Pageable pageable);
    long count(Predicate predicate);
}
```

- QueryDslPredicateExecutor는 스프링 데이터 JPA에서 편리하게 QueryDSL을 사용할 수 있지만 기능에 한계가 있는데 join, fetch를 사용할 수가 없다 (JPQL에서 이야기하는 묵시적 조인은 가능)
- QueryDSL이 제공하는 다양한 기능을 사용하려면 JPAQuery를 직접 사용하거나 스프링 데이터 JPA가 제공하는 QueryDslRepositorySupport를 사용해야 한다

### **QueryDslRepositorySupport 사용**

- QueryDSL의 모든 기능을 사용하려면 JPAQuery 객체를 직접 생성해서 사용하면 된다
- QueryDslRepositorySupport를 상속 받아 사용하면 조금 더 편리하게 QueryDSL를 사용할 수 있다

```java
// CustomOrderRepository 사용자 정의 레포지토리
public interface CustomOrderRepository {
    public List<Order> search(OrderSearch orderSearch);
}
```

- 스프링 데이터 JPA가 제공하는 공통 인터페이스를 직접 구현할 수가 없기 때문에 CustomOrderRepository라는 사용자 정의 레포지토리를 만들었다

```java
public class OrderRepositoryImpl extends QueryDslRepositorySupport implements CustomOrderRepository {
    public OrderRepositoryImpl() {
        super(Order.class);
    }
    
    @Override
    public List<Order> search(OrderSearch orderSearch) {
        QOrder order = QOrder.order;
        QMember member = QMember.member;
        
        JPAQuery query = from(order);
        
        if(StringUtils.hasText(OrderSearch.getMemberName())) {
            query.leftJoin(order.member, member)
                .where(member.name.contains(orderSearch.getMemberName()));
        }
        
        if(orderSearch.getOrderStatus() != null) {
            query.where(order.status.eq(orderSearch.getOrderStatus()));''
        }
        
        return query.list(order);
    }
}
```

- QueryDslRepositorySupport를 사용해서 QueryDSl로 구현한 예제인데 검색 조건에 따라 동적으로 쿼리를 생성한다
- QueryDslRepositorySupport에 엔티티 클래스 정보를 넘겨주어야 한다

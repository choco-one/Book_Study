> **프록시와 즉시로딩, 지연로딩**<br/>
> : 프록시를 사용하면 연관된 객체를 처음부터 데이터베이스에서 조회하는 것이 아니라, 실제 사용하는 시점에 조회할 수 있다.
> 
> **영속성 전이와 고아 객체**<br/>
> : JPA는 연관된 객체를 함께 저장하거나 삭제할 수 있는 영속성 전이와, 고아 객체 제거라는 편리한 기능을 제공한다.

## 💫 프록시
엔티티 조회 시 연관된 엔티티들이 항상 사용되지는 않는다. 비즈니스 로직에 따라 사용될 때도 있고, 그렇지 않은 때도 존재한다.

```java
// 회원과 팀 정보를 모두 출력하는 로직
public void printUserAndTeam(String memberId) {
  Member member = em.find(Member.class, memberId);
  Team team = member.getTeam();
  System.out.println("회원 이름: " + member.getUsername());
  System.out.println("소속팀 이름: " + team.getName());
}
```

```java
// 회원 정보만 출력하는 로직
public void printUser(String memberId) {
  Member member = em.find(Member.class, memberId);
  System.out.println("회원 이름: " + member.getUsername());
}
```
- `Member` 는 `Team` 에 대해 다대일 관계를 가지고 있다. (회원 엔티티에 `Team` 이 저장)
- `printUser()` 는 회원 엔티티만 사용하기에 `em.find()` 로 회원 엔티티 조회 시 회원과 연관된 팀 엔티티까지 조회하는 것은 효율적이지 않다. (어차피 사용하지 않음)
  
JPA는 이런 문제 해결을 위해 엔티티가 실제 사용될 떄까지 데이터베이스 조회를 지연하는 방법을 제공하는데, 이를 **지연 로딩**이라 한다.
- 즉, `team.getTeam()` 처럼 팀 엔티티의 값을 **실제 사용하는 시점에** 데이터베이스에서 팀 엔티티에 필요한 데이터를 조회하는 것이다.
- 지연 로딩을 위해서는 실제 엔티티 객체 대신에 **데이터베이스 조회를 지연할 수 있는 가짜 객체**가 필요한데, 이를 **프록시 객체**라 한다.

> JPA 표준 명세는 지연 로딩의 구현 방법을 JPA 구현체에 위임했다. 따라서 아래의 내용들은 Hibernate 구현체에 대한 내용이다. Hibernate는 프록시를 사용하는 방법과 바이트코드를 수정하는 2가지 방법으로 지연 로딩을 지원한다.

### ➰ 프록시 기초
JPA에서 식별자로 엔티티 하나를 조회할 때는 `em.find()` 를 사용한다. 이 메소드는 영속성 컨텍스트에 엔티티가 없으면 데이터베이스를 조회한다. 엔티티를 직접 조회하면, 조회한 엔티티를 **실제 사용하든, 사용하지 않든 데이터베이스를 조회**하게 된다.
- 엔티티를 실제 사용하는 시점까지 데이터베이스 조회를 미루고 싶다면 `em.getReference()` 메소드를 사용하면 된다.

```java
Member member = em.getReference(Member.class, "member1");
```
- 이 메소드를 호출할 때 JPA는 데이터베이스를 조회하지 않고 실제 엔티티 객체도 생성하지 않는다. 
- 대신 **데이터베이스 접근을 위임한 프록시 객체를 반환**한다.

**프록시의 특징**<br/>
프록시 클래스는 실제 클래스(실제 Entity 클래스)를 상속 받아 만들어지므로 **실제 클래스와 겉 모양이 같다.** 따라서 사용하는 입장에서는 이것이 진짜 객체인지 프록시 객체인지 구분하지 않고 사용하면 된다.

프록시 객체는 실제 객체에 대한 참조(`target`)를 보관한다. 그리고 프록시 객체의 메소드를 호출하면 **프록시 객체는 실제 객체의 메소드를 호출**한다.

**프록시 객체의 초기화**<br/>
프록시 객체는 `member.getName()` 같이 실제 사용될 떄 데이터베이스를 조회해 실제 엔티티 객체를 생성하는데, 이를 프록시 객체의 초기화라 한다. 아래는 프록시 객체의 초기화 과정이다.

```java
// MemberProxy 반환
Member member = em.getReference(Member.class, "id1");
member.getName(); // getName();
```

```java
class MemberProxy extends Member {
	Member target = null;

	public String getName() {
		if (target == null) {

			// 초기화 요청
			// DB 조회
			// 실제 엔티티 생성 및 참조 보관
			this.target = ...;
		}

		// target.getName();
		return target.getName();
	}
}
```
1. 프록시 객체에 `member.getName()` 을 호출해 실제 데이터를 조회한다.
2. 프록시 객체는 실제 엔티티가 생성되어 있지 않으면(`if(target == null)`), 영속성 컨텍스트에 실제 엔티티 생성을 요청하는데 이를 **초기화**라 한다. 
3. 영속성 컨텍스트는 데이터베이스를 조회해 실제 엔티티 객체를 생성한다.
4. 프록시 객체는 생성된 실제 엔티티 객체의 참조를 `Member target` 멤버변수에 보관한다.
5. 프록시 객체는 실제 엔티티 객체의 `getName()` 을 호출해 결과를 반환한다.

**프록시의 특징**<br/>
- 프록시 객체는 첫 사용 시에만 초기화된다.
- 프록시 객체를 초기화하는 것이 실제 엔티티로 바뀌는 것이 아니다. 프록시 객체가 초기화되면 프록시 객체를 통해 실제 엔티티에 접근할 수 있다.
- 프록시 객체는 원본 엔티티를 상속받은 객체이므로 타입 체크 시에 주의해서 사용해야 한다.
- 영속성 컨텍스트에 찾는 엔티티가 이미 있으면 데이터베이스를 조회할 필요가 없기에, `em.getReference()` 를 호출해도 프록시가 아닌 실제 엔티티를 반환한다.
- 초기화는 영속성 컨텍스트의 도움을 받아야 가능하다. 따라서 준영속 상태의 프록시를 초기화하면 Hibernate는 `org.hibernate.LazyInitializationException` 예외를 발생시킨다.

**준영속 상태와 초기화**<br/>
준영속 상태와 초기화에 관련된 코드이다.
```java
Member member = em.getReference(Member.class, "id1");
transaction.commit();
em.close(); // 영속성 컨텍스트 종료

member.getName(); // 준영속 상태인데 초기화를 시도!
```
- 이미 영속성 컨텍스트를 종료해서, `member` 는 준영속 상태이다.
- `getName()` 을 호출하면 프록시를 초기화해야 하는데 영속성 컨텍스트가 없으므로 실제 엔티티를 조회할 수 없다. 따라서 **예외가 발생**한다.

### ➰ 프록시와 식별자
엔티티를 프록시로 조회할 때, 식별자 값을 파라미터로 전달하는데 **프록시 객체는 이 식별자 값을 보관**한다.

```java
Team team = em.getReference(Team.class, "team1"); // 식별자 보관
team.getId(); // 초기화되지 않음
```
- 프록시 객체는 식별자 값을 이미 가지고 있으므로, 식별자 값을 조회하는 `getId()` 를 호출해도 프록시를 초기화를 하지 않는다.
  - 단 **엔티티 접근 방식을 `@Access(AccessType.PROPERTY)` 로 설정한 경우에만** 초기화하지 않는다.

> 엔티티 접근 방식을 `@Access(AccessType.FIELD)` 로 설정하면 JPA는 `getId()` 메소드가 `id` 만 조회하는 메소드인지 다른 필드까지 활용해 어떤 일을 하는 메소드인지 알지 못해 프록시 객체를 초기화한다.

> **`@Access`**<br/>
> : JPA가 엔티티에 접근하는 방식을 지정하는 annotation
> - 필드 방식
> 	- 필드에 직접 접근, `private` 이어도 접근
> - 프로퍼티 방식
> 	- 접근자(Getter)를 이용해 접근 

프록시는 다음처럼 연관관계를 설정할 떄 유용하게 사용할 수 있다.

```java
Member member = em.find(Member.class, "member1");
Team team = em.getReference(Team.class, "team1"); // SQL을 실행하지 않음
member.setTeam(team);
```
- 연관관계 설정 시에는 식별자 값만 사용하기에 프록시를 사용하면 데이터베이스 접근 횟수를 줄일 수 있다. (원래 같으면 INSERT하고, UPDATE하는 방식)
- 참고로 연관관계 설정 시에는 엔티티 접근 방식을 필드로 설정해도 프록시를 초기화하지 않는다.

### ➰ 프록시 확인
JPA가 제공하는 `PersistenceUnitUtil.isLoaded(Object entity)` 메소드를 사용하면 프록시 인스턴스의 초기화 여부를 확인할 수 있다.
- 아직 초기화되지 않았다면 `false` 를 반환한다.
- 이미 초기화되었거나 프록시 인스턴스가 아니면 `true` 를 반환한다.

```java
boolean isLoad = em.getEntityManagerFactory()
					.getPersistenceUnitUtil().isLoaded(entity);
// or, boolean isLoad = emf.getPersistenceUnitUtil().isLoaded(entity);
System.out.println("isLoad = " + isLoad); // 초기화 여부 확인
```

조회한 엔티티가 진짜 엔티티인지 프록시로 조회한 것인지 확인하려면 **클래스명을 직접 출력**해보면 된다. 출력된 클래스명 뒤에 `...javassist...` 라 되어있다면 프록시이다. (`member.getClass().getName()`)

> **프록시 강제 초기화**<br/>
> hibernate의 `initialize()` 메소드로 강제 초기화가 가능하다.
> ```java
> org.hibernate.Hibernate.initialize(order.getMember());
> ```

---

## 💫 즉시 로딩과 지연 로딩
프록시 객체는 **주로 연관된 데이터를 지연 로딩할 때 사용**한다.

회원 엔티티를 조회할 때, 연관된 팀 엔티티도 함께 데이터베이스에서 조회하는 것이 좋을까? 아니면 팀 엔티티는 실제 사용하는 시점에 데이터베이스에서 조회하는 것이 좋을까?
- JPA는 개발자가 연관된 엔티티의 조회 시점을 선택할 수 있도록 아래 두 가지 방법을 제공한다.
- 즉시 로딩 : 엔티티를 조회할 때 연관된 엔티티도 함께 조회한다.
  - `em.find()` 호출 시 연관된 엔티티도 함께 조회한다.
  - 설정 방법: **`@ManyToOne(fetch = FetchType.EAGER)`**
- 지연 로딩 : 연관된 엔티티를 실제 사용할 떄 조회한다.
  - `member.getTeam().getName()` 처럼 연관된 엔티티를 실제 사용하는 시점에 JPA가 SQL을 호출해서 조회한다.
  - 설정 방법 : **`@ManyToOne(fetch = FetchType.LAZY)`**

### ➰ 즉시 로딩
```java
@Entity
public class Member {

  //...
  @ManyToOe(fetch = FetchType.EAGER)
  @JoinColumn(name = "TEAM_ID")
  private Team team;
  // ...
}
```

```java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 객체그래프 탐색
```
- 즉시 로딩을 실행하는 코드이다. 회원을 조회할 때 팀 정보도 즉시 로딩한다.
- 회원과 팀 두 테이블을 조회해야 하므로, 쿼리를 2번 실행할 것 같지만, 대부분의 JPA 구현체는 **즉시 로딩 최적화를 위해 가등하면 조인 쿼리를 사용**한다.
  - 즉 위 예제에서는 회원과 팀을 조인해서 쿼리 한 번으로 두 엔티티를 모두 조회한다.

> **NULL 제약 조건과 JPA 조인 전략**<br/>
> 즉시 로딩 실행 SQL에서 외부 조인(LEFT OUTER JOIN)을 사용했다. 외래 키에 `NULL` 값을 허용해서 팀에 소속되지 않은 회원이 있을 수 있게 했다. 이 경우에는 팀에 소속되지 않은 회원과 팀의 내부 조인이 불가능하다. 
> 
> 내부 조인보다는 외부 조인이 성능과 최적화에서 유리한데, 내부 조인을 사용하려면 어떻게 해야 할까? 
> - 외래 키에 NOT NULL 제약 조건을 설정하면 값이 있는 것을 보장한다. 즉 내부 조인이 가능해진다.
> - JPA에도 이런 사실을 알려주기 위해 `@JoinColumn(nullable = false)` 를 설정해 이 외래 키는 NULL 값을 허용하지 않는다는 것을 명시한다. 이를 통해 JPA는 외부 조인 대신 내부 조인을 사용한다.
> 정리! <br/>
> JPA는 선택적 관계면 외부 조인을 사용하고, 필수 관계면 내부 조인을 사용한다.

### ➰ 지연 로딩
```java
@Entity
public class Member {
  // ...
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "TEAM_ID")
  private Team team;
  // ...
}
```

```java
Member member = em.find(Member.class, "member1");
Team team = member.getTeam(); // 객체그래프 탐색
team.getName(); // 팀 객체 실제 사용
```
- 지연 로딩 설정으로, `em.find()` 호출 시 회원만 조회하고 팀은 조회하지 않는다. 
- 대신 조회한 회원의 `team` 멤버 변수에 프록시 객체를 넣어둔다. (`Team team = member.getTeam()`)
- 반환된 팀 객체는 프록시 객체다. 이는 실 사용때까지 데이터 로딩을 미루기에 지연 로딩이라 한다. 그리고 실제 데이터가 필요한 순간 데이터베이스를 조회해서 프록시 객체를 초기화한다.

> 조회 대상이 영속성 컨텍스트에 이미 있으면 프록시 객체를 사용할 이유가 없다. 곧바로 실 객체를 사용한다.

### ➰ 즉시 로딩, 지연 로딩 정리
사실 상황에 따라 너무 다르다. 
- 지연 로딩 : 연관된 엔티티를 프록시로 조회한다. 프록시를 실제 사용할 때 초기화하면서 데이터베이스를 조회한다.
- 즉시 로딩 : 연관된 엔티티를 즉시 조회한다. Hibernate는 가능하면 SQL 조인을 사용해 한 번에 조회한다.

---

## 💫 지연 로딩 활용
지연 로딩을 활용하는 예시이다. 아래와 같은 요구사항이 있다.
- 회원은 팀 하나에만 소속할 수 있다. (N:1)
- 회원은 여러 주문내역을 가진다. (1:N)
- 주문내역은 상품정보를 가진다. (N:1)

그리고 이를 분석하면 다음과 같은 결과를 얻을 수 있다.
- 회원과 연관된 팀은 자주 함께 사용되었다. 따라서 이를 **즉시 로딩**으로 설정했다.
- 회원과 연관된 주문내역은 가끔 사용되었다. 따라서 **지연 로딩**으로 설정했다.
- 주문내역과 연관된 상품정보는 자주 함께 사용되었다. 따라서 **즉시 로딩**으로 설정했다.

```java
@Entity
public class Member {

  @Id
  private String id;
  private String username;
  private Integer age;

  @ManyToOne(fetch = FetchType.EAGER)
  private Team team;

  @OneToMany(mappedBy = "member", fetch = FetchType.LAZY)
  private List<Order> orders;
}
```

### ➰ 프록시와 컬렉션 래퍼
지연 로딩으로 설정하면 실제 엔티티 대신 프록시 객체를 사용한다. 프록시 객체는 실제 자신이 사용될 때까지 데이터베이스를 조회하지 않는다. Hibernate는 엔티티를 영속 상태로 만들 때 엔티티에 컬렉션이 있으면 컬렉션을 추적하고 관리할 목적으로 원본 컬렉션을 Hibernate가 제공하는 내장 컬렉션으로 변경하는데, 이를 **컬렉션 래퍼**라 한다. 
- 출력 결과에서 컬랙션 래퍼인 `org.hibernate.collection.internal.PersistentBag` 이 반환되는 것을 확인할 수 있다.

엔티티를 지연 로딩하면, 프록시 객체를 사용해 지연 로딩을 수행하지만 주문내역과 같은 **컬렉션은 컬렉션 래퍼가 지연 로딩을 처리**해준다. 컬렉션은 컬렉션에서 실제 데이터를 조회할 때 데이터베이스를 조회해서 초기화한다.

### ➰ JPA 기본 페치 전략
`fetch` 속성의 기본 설정값은 다음과 같다.
- `@ManyToOne` , `@OneToOne` :  즉시 로딩
- `@OneToMany` , `@ManyToMany` : 지연 로딩

JPA의 기본 페치 전략은, 연관된 엔티티가 하나면 즉시 로딩, 컬렉션이면 지연 로딩을 사용한다.
- 컬렉션을 로딩하는 것은 비용이 많이 들고 잘못하면 너무 많은 데이터를 로딩할 수 있기 때문이다. 

> 모든 연관관계에 지연 로딩을 사용하는 것을 추천한다. 개발이 어느 정도 완료되었다면 필요한 곳에만 즉시 로딩을 사용하도록 최적화한다. (SQL을 직접 사용하게 되면 이러한 유연성을 가지기 어렵다.)

### ➰ 컬렉션에 FetchType.EAGER 사용 시 주의점
즉시 로딩사용 시의 주의점이다.
- **컬렉션을 하나 이상 즉시 로딩하는 것은 권장하지 않는다.**
  - 컬렉션과 조인한다는 것은 데이터베이스 테이블로 본다면 일대다 조인이다. 이는 결과 데이터가 다 쪽에 있는 수만큼 증가하게 된다.
  - 문제는 서로 다른 컬렉션을 2개 이상 조인할 때 발생하는데, 결과 데이터가 너무 많아져 애플리케이션의 성능 저하를 발생시킬 수 있다. JPA는 이렇게 조회된 결과를 메모리에서 필터링해 반환한다. 
- **컬렉션 즉시 로딩은 항상 외부 조인을 사용한다.**
  - 다대일 관계인 회원 테이블과 팀 테이블 조인 시 회원 테이블의 외래 키에 `not null` 제약조건을 걸어두면 모든 회원은 팀에 소속되어 항상 내부 조인을 사용해도 된다.
  - 반대로 팀 테이블에서 회원 테이블로 일대다 관계를 조인할 때 회원이 한 명도 없는 팀을 내부 조인하면 팀까지 조회되지 않는 문제가 발생하고 이는 제약조건으로 막을 수 없는 문제이므로 외부 조인을 사용해야 한다.

> `FetchType.EAGER` 설정과 조인 전략
> - `@ManyToOne` , `@OneToOne`
>   - `(optional = false)` : 내부 조인
>   - `(optional = true)` : 외부 조인
> - `@OneToMany` , `@ManyToMany`
>   - `(optional = false)` : 외부 조인
>   - `(optional = true)` : 외부 조인

--- 

## 💫 영속성 전이: CASCADE
**특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속 상태로 만들고 싶으면** 영속성 전이 기능(Transitive Persistence)을 사용하면 된다.
- JPA는 `CASCADE` 옵션으로 이를 제공한다.
- 영속성 전이를 사용해 부모 엔티티를 저장할 때 자식 엔티티도 함께 저장할 수 있다.

```java
@Entity
public class Parent {

  @Id @GeneratedValue
  private Long id;

  @OneToMany(mappedBy = "parent")
  private List<Child> children = new ArrayList<>();
  ...

}
```

```java
@Entity
public class Child {

  @Id @GeneratedValue
  private Long id;

  @ManyToOne
  private Parent parent;
  ...
  
}
```
- 부모 엔티티가 여러 자식 엔티티를 가지고 있고, 부모 1명에 자식 2명을 저장한다면 아래와 같은 코드를 작성할 수 있다.

```java
private static void saveNoCascade(EntityManager em) {

  // 부모 저장
  Parent parent = new Parent;
  em.persist(parent);

  // 1번 자식 저장
  Child child1 = new Child();
  child1.setParent(parent);
  parent.getChildren().add(child1);
  em.persist(child1);

  // 2번 자식 저장
  Child child2 = new Child();
  child2.setParent(parent);
  parent.getChildren().add(child2);
  em.persist(child2);
}
```
- **JPA에서 엔티티를 저장할 때 연관된 모든 엔티티는 영속 상태**여야 한다. 위 예제에서도 부모 엔티티를 먼저 영속 상태로 만들고, 이후 자식 엔티티들을 영속 상태로 만든다.
- 이런 경우 영속성 전이를 사용해 부모만 영속 상태로 만들면, 연관된 자식까지 한 번에 영속 상태로 만들 수 있다.

### ➰ 영속성 전이: 저장
영속성 전이를 활성화하는 `CASCADE` 옵션을 적용해보자.
```java
@Entity
public class Parent {
  ...
  @OneToMany(mappedBy = "parent", cascade = CascadeType.PERSIST)
  private List<Child> children = new ArrayList<>();
  ...

}
```
- 부모를 영속화할 때 연관된 자식들도 함께 영속화하라는 옵션을 설정했다.

```java
private static void saveNoCascade(EntityManager em) {

  Child child1 = new Child();
  Child child2 = new Child();

  // 부모 저장
  Parent parent = new Parent;
  child1.setParent(parent);
  child2.setParent(parent);
  parent.getChildren().add(child1);
  parent.getChildren().add(child2);

  em.persist(parent);
}
```
- 간편하게 부모와 자식 엔티티를 한 번에 영속화하는 것을 확인할 수 있다.

> 영속성 전이는 연관관계를 매핑하는 것과는 관련이 없고, 단지 **엔티티를 영속화할 때 연관된 엔티티도 같이 영속화하는 편리한 기능**일 뿐이다.

### ➰ 영속성 전이: 삭제
영속화한 부모 자식 엔티티를 모두 제거하려면 각각의 엔티티를 하나씩 제거해야 한다.

```java
Parent findParent = em.find(Parent.class, 1L);
Child findChild1 = em.find(Child.class, 1L);
Child findChild2 = em.find(Child.class, 2L);

em.remove(findChild1);
em.remove(findChild2);
em.remove(findParent);
```

영속성 전이는 엔티티 삭제 시에도 사용할 수 있다. `CascadeType.REMOVE` 로 설정하고 부모 엔티티만 삭제하면 연관된 자식 엔티티도 함께 삭제된다.
- 모든 엔티티 제거를 위한 DELETE SQL을 실행하고, 삭제 순서는 외래 키 제약조건을 고려해 자식 먼저 삭제한 다음 부모를 삭제한다.
- 만약 위 설정 없이 부모 엔티티만 삭제하면, 자식 엔티티는 제거되지 않았지만 외래 키 제약조건으로 인해, 데이터베이스에서 외래 키 무결성 제약조건이 깨지는 예외가 발생한다.

### ➰ CASCADE의 종류
```java
public enum CascadeType {
  ALL, 
  PERSIST,
  MERGE,
  REMOVE,
  REFRESH,
  DETACH
}
```

> 참고로 `PERSIST` , `REMOVE` 는 `em.persist()` , `em.remove()` 실행 즉시 전이가 발생하는 것이 아니라 플러시 호출 시 전이가 발생한다.

---

## 💫 고아 객체
JPA는 **부모 엔티티와 연관관계가 끊어진 자식 엔티티를 자동으로 삭제**하는 기능을 제공하는데 이를 고아 객체(ORPHAN) 제거라 한다. 
- 이 기능으로 부모 엔티티의 컬렉션에서 자식 엔티티의 참조만 제거해 자식 엔티티가 자동으로 삭제되도록 해보자.

```java
@Entity
public class Parent {

  @Id @GeneratedValue
  private Long id;

  @OneToMany(mappedBy = "parent", orphanRemoval = true)
  private List<Child> Children = new ArrayList<>();
  ...

}
```
- 고아 객체 제거 기능 활성화를 명시하는 설정을 추가했다. 이제 컬렉션에서 제거한 엔티티는 자동으로 삭제된다.
- 컬렉션에서 첫 번째 자식을 제거하면, DELETE SQL을 실행하여 데이터베이스의 데이터 또한 삭제한다.
- 이 기능은 영속성 컨텍스트를 플러시할 때 적용되므로 플러시 시점에 DELETE SQL이 실행된다.
  - 모든 자식 엔티티 제거를 위해서는 컬렉션을 비우기만 하면 된다.

고아 객체 제거는 **참조가 제거된 엔티티는 다른 곳에서 참조하지 않는 고아 객체로 보고, 삭제하는 기능**이다.
- 따라서 참조하는 곳이 하나일때만 사용해야 한다! (즉, 특정 엔티티가 개인 소유하는 엔티티만 사용)
- 따라서 `orphanRemoval` 속성은 `@OneToOne` , `@OneToMany` 로 일대~인 경우만 사용할 수 있다.
- 추가 기능이 있는데, 부모를 제거하면 자식은 자연스레 고아가 된다. 따라서 **부모를 제거하면 자식도 함께 제거하는 기능**이 있다. 이는 `CascadeType.REMOVE` 를 설정한 것과 동일하다.
---

## 💫 영속성 전이 + 고아 객체, 생명주기
`CascadeType.ALL` + `orphanRemoval = true` 를 동시에 사용하면 어떻게 될까? 
- 일반적으로 엔티티는 `em.persist()` 를 통해 영속화되고, `em.remove()` 를 통해 제거된다. 이는 엔티티 스스로 생명주기를 관리한다는 뜻이다. 
- 그런데 두 옵션을 모두 설정하면 부모 엔티티를 통해 자식의 생명주기를 관리할 수 있다.

```java
// 자식을 저장하려면 부모에 등록만 하면 된다. (CASCADE)
Parent parent = em.find(Parent.class, parentId);
parent.setChild(child1);
```

```java
// 자식을 삭제하려면 부모에서 제겅하면 된다. (orphanRemoval)
Parent parent = em.find(Parent.class, parentId);
parent.getChildren().remove(removeObject);
```

## 📕 출처
**자바 ORM 표준 JPA 프로그래밍** - 김영한

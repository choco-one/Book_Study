> 엔티티들은 대부분 다른 엔티티와 **연관관계**가 있다. 그런데 객체는 참조(주소)를 사용하고, 테이블은 외래 키를 사용해 관계를 맺는다. 이 둘은 완전히 다른 특징을 가지므로, 객체 관계 매핑에서 가장 어려운 부분이라고 할 수 있다. 
> 
> **방향**
> - 단방향, 양방향이 있다. 객체관계에만 방향이 존재하고, 테이블에서는 항상 양방향이다.
> 
> **다중성**
> - N:1, 1:N, 1:1, N:M 다중성이 있다.
> 
> **연관관계의 주인**
> - 객체를 양방향 연관관계로 만들면 연관관계의 주인을 정해야 한다.

## **💫 단방향 연관관계**

연관관계 중에선 다대일(N:1) 단방향 관계를 가장 먼저 이해해야 한다.

- 회원과 팀이 있다.
- 회원은 하나의 팀에만 소속될 수 있다.
- 회원과 팀은 **다대일 관계**이다.

**객체 연관관계**

- 회원 객체는 `Member.team` 필드 (멤버변수)로 팀 객체와 연관관계를 맺는다.
- 회원 객체와 팀 객체는 **단방향 관계**다. 회원은 팀을 알 수 있지만, 반대로 팀은 회원을 알 수 없다. (팀에서 접근하는 필드가 없기 때문)

**테이블 연관관계**

- 회원 테이블은 `TEAM_ID` 외래 키로 팀 테이블과 연관관계를 맺는다.
- 회원 테이블과 팀 테이블은 **양방향 관계**다. 조인을 통해 서로서로에 접근할 수 있다.

외래 키 하나로 어떻게 양방향 조인이 가능할까?

```sql
SELECT * FROM MEMBER M JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID

SELECT * FROM TEAM T JOIN MEMBER M ON T.TEAM_ID = M.TEAM_ID
```

**객체 연관관계와 테이블 연관관계의 가장 큰 차이**

참조를 통한 연관관계는 항상 **단방향**이다. 객체 간 연관관계를 양방향으로 만들고 싶으면 반대쪽에도 필드를 추가해 참조를 보관해야 한다. (즉 연관관계를 하나 더 생성) 이는 정확하게는 **양방향 관계가 아닌 서로 다른 단방향 관계 2개**가 되는 셈이다.

반면에 테이블은 **외래 키 하나로 양방향 조인이 가능**하다.

**객체 연관관계 vs. 테이블 연관관계 정리**

- 객체는 참조(주소)로 연관관계를 맺는다.
    - 단방향 연관관계다.
    - 양방향이 되기 위해서는 단방향 연관관계를 1개 더 만들어야 한다.
- 테이블은 외래 키로 연관관계를 맺는다.
    - 양방향 연관관계다.

연관된 데이터를 조회할 때 객체는 `a.getB().getC()` 를 사용하지만 테이블은 조인을 사용한다.

### **➰ 순수한 객체 연관관계**

순수하게 객체만 사용한 연관관계 코드이다.

```java
public class Member {
  private String id;
  private String username;

  private Team team; // 팀 참조 보관

  public void setTeam(Team team) {
    this.team = team;
  }

  ...

}

public class Team {
  private String id;
  private String name;

  ...

}
```

아래 코드로 회원1과 2를 팀1에 소속시킨다.

```java
public static void main(String[] args) {
  Member member1 = new Member("member1", "회원1");
  Member member2 = new Member("member2", "회원2");
  Team team1 = new Team("team1", "팀1");

  member1.setTeam(team1);
  member2.setTeam(team1);

  Team findTeam = member1.getTeam();
}
```

이처럼 객체는 참조를 사용해 연관관계를 탐색할 수 있는데, 이를 **객체 그래프 탐색**이라 한다.

### **➰ 테이블 연관관계**

데이터베이스 테이블의 회원과 팀의 관계를 살펴본다. 두 테이블은 모두 `TEAM_ID` 를 가지고, 회원 테이블의 `TEAM_ID` 에 외래 키 제약조건을 설정했다.

- INSERT SQL로 회원1과 2를 팀1에 소속시킨다.
- SELECT SQL로 회원1이 소속된 팀을 조회한다.
    - 이때 `TEAM_ID` 에 대한 `JOIN` 을 사용한다.

이처럼 데이터베이스는 외래 키를 사용해 연관관계를 탐색할 수 있는데, 이를 **조인**이라 한다.

### **➰ 객체 관계 매핑**

지금까지 다룬 객체, 테이블 연관관계를 JPA를 사용한 매핑으로 바꿔본다.

- 객체 연관관계: 회원 객체의 `Member.team` 필드 사용
- 테이블 연관관계: 회원 테이블의 `MEMBER.TEAM_ID` 외래 키 컬럼 사용

```java
@Entity
public class Member {
  @Id
  @Column(name = "MEMBER_ID")
  private String id;

  private String username;

  @ManyToOne
  @JoinColumn(name = "TEAM_ID")
  private Team team;

  public void setTeam(Team team) {
    this.team = team;
  }

  ...

}
```

```java
@Entity
public class Team {
  @Id
  @Column(name = "TEAM_ID")
  private String id;

  private String name;

  ...

}
```

- `Member.team` 과 `MEMBER.TEAM_ID` 를 매핑하는 것이 연관관계 매핑이다. 이를 위해 사용한 annotation을 분석한다.
- `@ManyToOne`
    - 이름 그대로 다대일 관계라는 매핑 정보를 의미한다. 연관관계 매핑 시에는 이렇게 다중성을 나타내는 annotation을 필수로 사용해야 한다.
- `JoinColumn(name = "TEAM_ID")`
    - 외래 키를 매핑할 때 사용한다. `name` 속성으로 매핑할 외래 키 이름을 지정한다. 생략가능한 annotation이다.
    - 이를 생략하면, 외래 키를 찾을 때 기본 전략을 사용한다.
    >  **기본 전략** : 필드명 + _ + 참조하는 테이블의 컬럼명
    > - ex. 필드명(team) + _ + 참조하는 테이블의 컬럼명(TEAM_ID) = team_TEAM_ID 외래 키
    
---

## **💫 연관관계 사용**

연관관계를 등록, 수정, 삭제, 조회하는 예제를 통해 알아본다.

```java
public void testSave() {
  Team team1 = new Team("team1", "팀1");
  em.persist(team1);

  Member member1 = new Member("member1", "회원1");
  member1.setTeam(team1);
  em.persist(member1);

  Member member2 = new Member("member2", "회원2");
  member2.setTeam(team1);
  em.persist(member2);
}
```

<aside>
💡 JPA에서 엔티티 저장할 때 연관된 모든 엔티티는 영속 상태여야 한다.

</aside>

회원 엔티티는 팀 엔티티를 참조(`setTeam`)하고 저장(`persist`)했다. JPA는 참조한 팀의 식별자(`Team.id`)를 외래 키로 사용해 적절한 **등록 쿼리를 생성**할 것이다.

### **➰ 조회**

연관관계가 있는 엔티티를 조회하는 방법은 2가지이다.

- 객체 그래프 탐색(객체 연관관계를 사용한 조회)
- 객체지향 쿼리 사용(JPQL)

**객체 그래프 탐색**

`member.getTeam()` 을 사용해 회원과 연관된 팀 엔티티를 조회할 수 있다.

```java
member member = em.find(Member.class, "member1");
Team team = member.getTeam();
```

**객체지향 쿼리 사용**

예를 들어 회원을 대상으로 조회하는데 팀1에 소속된 회원만 조회하려면 회원과 연관된 팀 엔티티를 검색 조건으로 사용해야 한다. SQL은 연관된 테이블을 조인해서 검색조건을 사용하면 된다. JPQL도 조인을 지원한다.

```java
private static void queryLogicJoin(EntityManager em) {
  String jpql = "select m from Member m join m.team t where " + "t.name=:teamName";

  List<Member> resultList = em.createQuery(jpql, Member.class)
      .setParameter("teamName", "팀1")
      .getResultList();

  for (Member member : resultList) {
    System.out.println("[query] member.username=" + member.getUsername());
  }
}
```

회원이 팀과 관계를 가지고 있는 필드(`m.team`)을 통해 회원과 팀 엔티티를 조인했다. 그리고 `where` 절에서 조인한 `t.name` 을 검색조건으로 사용해 원하는 회원만 검색했다.

💡 `:teamName` 과 같이 :로 시작하는 것은 파라미터를 바인딩받는 문법이다.

### **➰ 수정**

팀1 소속이던 회원을 새로운 팀2에 소속하도록 수정한다.

```java
private static void updateRelation(Entitymanager em) {
  Team team2 = new Team("team2", "팀2");
  em.persist(team2);

  Member member = em.find(Member.class, "member1");
  member.setTeam(team2);
}
```

- JPA는 UPDATE SQL을 생성해 자동으로 처리한다.

### **➰ 연관관계 제거**

회원1을 팀에 소속하지 않도록 변경한다.

```java
private static void deleteRelation(Entitymanager em) {
  Member member = em.find(Member.class, "member1");
  member.setTeam(null);
}
```

- 연관관계를 `null` 로 설정했다. 이때는 다음과 같은 SQL이 생성된다.

```sql
UPDATE MEMBER SET TEAM_ID=null, ... WHERE ID='member1'
```

### **➰ 연관된 엔티티 삭제**

삭제를 위해서는 기존 연관관계를 먼저 제거하고 삭제해야 한다.

- 그렇지 않으면 **외래 키 제약조건**으로 인해 데이터베이스에서 오류가 발생한다.

```java
member1.setTeam(null);
member2.setTeam(null);
em.remove(team);
```

---

## **💫 양방향 연관관계**

이제 회원에서 팀으로 접근하고 반대 방향인 팀에서도 회원으로 접근할 수 있도록 양방향 연관관계로 매핑해본다.

- 회원과 팀은 다대일 관계다. 반대로 팀에서 회원은 일대다 관계다.
    - 일대다 관계는 여러 건과 연관관계를 맺을 수 있어 컬렉션을 사용해야 한다. `Team.members` 를 `List` 컬렉션으로 추가했다.

> 회원 → 팀 (`Member.team`)
> 팀 → 회원 (`Team.members`)

데이터베이스의 테이블은 **외래 키 하나로 양방향으로 조회**할 수 있다. 따라서 데이터베이스에서는 추가할 내용이 전혀 없다.

### **➰ 양방향 연관관계 매핑**

회원 엔티티에서는 변경할 부분이 없다. 팀 엔티티를 보자.

```java
@Entity
public class Team {
  @Id
  @Column(name = "TEAM_ID")
  private String id;

  private String name;

  @OneToMany(mappedBy = "team")
  private List<Member> members = new ArrayList();

  ...

}
```

- 팀과 회원은 일대다 관계다. 따라서 팀 엔티티에 컬렉션인 `List<Member> members` 를 추가했다.
- 또한 일대다 관계 매핑을 위한 annotation을 사용했다.
- `mappedBy` 속성은 양방향 매핑일 때 반대쪽 매핑의 필드 이름을 값으로 지정한다. (반대쪽 매핑이 `Member.team`)

### **➰ 일대다 컬렉션 조회**

팀에서 회원 컬렉션으로 객체 그래프 탐색을 사용해 특정 팀에 소속된 회원들을 조회할 수 있다.

```java
public void biDirection() {
  Team team = em.find(Team.class, "team1");
  List<Member> members = team.getMembers();

  for (Member member : members) {
    System.out.println("member.username = " + member.getUsername());
  }
}
```

---

## **💫 연관관계의 주인**

`@OneToMany` annotation 사용 시 `mappedBy` 는 왜 필요할까?

- 객체에는 양방향 연관관계라는 것이 없다. 서로 다른 단방향 연관관계가 2개일 뿐이다.
    - 반면, 데이터베이스는 외래 키 하나로 양쪽이 서로 조인하기에 양방향 연관관계를 맺을 수 있다.
- 엔티티를 단방향으로 매핑하면 참조를 하나만 사용해 이 참조로 외래 키를 관리하면 된다.
    - 하지만 양방향으로 매핑하면 "회원 → 팀" , "팀 → 회원" 으로 두 곳에서 서로를 참조한다.
    - 즉, **객체의 연관관계를 관리하는 포인트가 2곳으로 늘어나게 되고,** 참조는 둘인데 외래 키는 하나여서 차이가 발생한다.
        - 이때 JPA에서는 두 객체 연관관계 중 하나를 정해 테이블의 외래 키를 관리해야 하는데 이때 정해진 하나가 **연관관계의 주인**이 된다.

### **➰ 양방향 매핑의 규칙: 연관관계의 주인**

**연관관계의 주인만이 데이터베이스 연관관계와 매핑되고, 외래 키를 관리(등록, 수정, 삭제)할 수 있다. 반면에 주인이 아닌 쪽은 읽기만 가능하다.**

- 주인은 `mappedBy` 속성을 사용하지 않는다.
- 주인이 아니면 `mappedBy` 속성을 사용해 속성의 값으로 연관관계의 주인을 지정한다.

연관관계의 주인을 어떻게 정해야 할까? 이는 곧 **외래 키 관리자를 선택하는 것**이다.

- 회원과 팀의 예에서는 회원 테이블에 있는 `TEAM_ID` 외래 키를 관리할 관리자를 선택하는 것이다.
    - 만약 회원 엔티티에 있는 `Member.team` 을 주인으로 선택하면 자기 테이블에 있는 외래 키를 관리하면 된다.
    - 하지만 팀 엔티티에 있는 `Team.members` 를 주인으로 선택하면 물리적으로 전혀 다른 테이블의 외래 키를 관리해야 한다.

### **➰ 연관관계의 주인은 외래 키가 있는 곳**

"연관관계의 주인은 테이블에 외래 키가 있는 곳으로 정해야 한다." 위의 예에서는 `Member.team` 이 주인이 된다.

- 주인이 아닌 `Team.members`에는 `mappedBy="team"` 속성을 사용해 주인이 아님을 설정하고. 속성 값으로는 연관관계의 주인(`team`)을 명시한다.

이로서 연관관계의 주인만 데이터베이스 연관관계와 매핑되고 외래 키를 관리할 수 있다. 주인이 아닌 반대편은 읽기만 가능하고 외래 키를 변경하지는 못한다.

> 💡 데이터베이스 테이블의 다대일, 일대다 관계에서는 항상 다 쪽이 외래 키를 가진다. 따라서 `@ManyToOne` 에는 `mappedBy` 속성이 없다.

---

## **💫 양방향 연관관계 저장**

```java
public void testSave() {
  Team team1 = new Team("team1", "팀1");
  em.persist(team1);

  Member member1 = new Member("member1", "회원1");
  member1.setTeam(team1);
  em.persist(member1);

  Member member2 = new Member("member2", "회원2");
  member2.setTeam(team1);
  em.persist(member2);
}
```

- 회원에 연관관계의 주인인 `Member.team` 필드를 이용해 회원과 팀의 연관관계를 설정하고 저장했다. 이는 **단방향 연관관계에서 살펴본 회원과 팀을 저장하는 코드와 동일**하다.
- 연관관계의 주인이 아닌 방향은 값을 설정하지 않아도 데이터베이스에 외래 키 값이 정상 입력되고, 설령 입력한다고 해도 이는 **무시**되어 영향을 주지 않는다.

---

## **💫 양방향 연관관계의 주의점**

가장 흔히 하는 실수는 **연관관계의 주인에는 값을 입력하지 않고, 주인이 아닌 곳에만 값을 입력**하는 것이다. 이는 방금 언급했듯 무시되어 영향을 주지 않는 동작이기에 데이터베이스에 외래 키 값이 정상적으로 저장되지 않으면 이부터 확인해야 한다.

<aside>
💡 "연관관계의 주인만이 외래 키의 값을 변경할 수 있다."

</aside>

### **➰ 순수한 객체까지 고려한 양방향 연관관계**

그렇다면 연관관계의 주인에만 값을 저장하고 주인이 아닌 곳에는 값을 저장하지 않아도 될까?

- 사실 **객체 관점에서는 양쪽 방향에 모두 값을 입력해주는 것이 가장 안전하다.**
- 양쪽 방향 모두 값을 입력하지 않으면 JPA를 사용하지 않는 순수한 객체 상태에서 심각한 문제가 발생할 수 있다.

```java
// ex.JPA를 사용하지 않고 엔티티에 대한 테스트 코드를 작성하는 경우
public void test순수한객체_양방향() {
  Team team1 = new Team("team1", "팀1");
  Member member1 = new Member("member1", "회원1");
  Member member2 = new Member("member2", "회원2");
  member1.setTeam(team1);
  member2.setTeam(team1);
  
  List<Member> members = team1.getMembers();
  System.out.println("members.size = " + members.size());
}
```

- JPA를 사용하지 않는 순수한 객체다. `Member.team` 에만 연관관계를 설정했기에 **팀에 소속된 회원이 하나도 없다.**
- 양방향은 양쪽 모두 관계를 설정해야 한다. 즉 반대방향으로도 설정이 이뤄져야 한다.
    - 이로서 순수한 객체 상태에서도 동작하며, 테이블의 외래 키도 정상 입력된다. (외래 키 값은 연관관계의 주인인 `Member.team` 값을 사용한다.)

```java
member1.setTeam(team1); // 주인, 외래 키 관리
team1.getMembers().add(member1); // 주인 X, 저장 시 사용되지 않음
```

<aside>
💡 결론 : 객체의 양방향 연관관계는 양쪽 모두 관계를 맺어주자.

</aside>

### **➰ 연관관계 편의 메소드**

양방향 연관관계는 결국 양쪽 모두 신경 써야 한다. 실수로 두 설정 메소드 중 하나만 호출해 양방향이 깨지는 경우가 발생할 수도 있다.

- 따라서 두 코드를 하나인 것처럼 사용하는 것이 안전하다. `Member` 클래스의 `setTeam()` 메소드를 리팩토링한다.

```java
public class Member {
  private Team team;

  public void setTeam(Team team) {
    this.team = team;
    team.getMembers().add(this);
  }
  ...

}
```

- `setTeam` 메소드에서 양방향 관계를 모두 설정하도록 수정했다. 이렇게 한 번에 양방향 관계를 설정하는 메소드를 **연관관계 편의 메소드**라 한다.

### **➰ 연관관계 편의 메소드 작성 시 주의사항**

사실 위와 같은 편의 메소드에는 버그가 있다. **`setTeam()` 을 호출한 직후에도 이전의 관계가 제거되지 않는 것**이다.

- 연관관계 변경 시에는 기존 팀이 있으면 그 관계를 삭제하고 새로운 관계를 맺는 코드로 바꿔야 한다.

```java
public void setTeam(Team team) {
  if (this.team != null) this.team.getMembers().remove(this);
  this.team = team;
  team.getMembers().add(this);
}
```

---

## **💫 정리**

단방향 매핑과 비교해서 양방향 매핑은 복잡하다.

- 연관관계의 주인 지정
- 두 개의 단방향 연관관계를 양방향으로 만들기 위해 로직 관리

> 연관관계가 하나인 **단방향 매핑** → 언제나 연관관계의 주인
> **양방향 매핑** → 단방향 + 주인이 아닌 연관관계를 하나 추가
> 양방향의 장점은 **반대방향으로 객체 그래프 탐색 기능이 추가**된 것뿐!
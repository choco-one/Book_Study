### ➰ 서브 쿼리
JPQL도 SQL처럼 서브 쿼리를 지원한다. 다음과 같은 제약사항이 존재한다.
- 서브 쿼리를 WHERE, HAVING 절에서만 사용할 수 있고,
- SELECT, FROM 절에서는 사용할 수 없다.

> Hibernate의 HQL은 SELECT 절의 서브 쿼리도 허용한다.

```sql
select m from Member m where m.age > (select avg(m2.age) from Member m2)
```
- 평균보다 나이가 많은 회원을 찾는다.

**서브 쿼리 함수**<br/>
- [NOT] EXISTS (subquery)
- { ALL | ANY | SOME } (subquery)
- [NOT] IN (subquery)

**EXISTS**
- 서브 쿼리에 결과가 존재하면 참이다. NOT은 반대

**{ ALL | ANY | SOME }**
- 비교 연산자와 같이 사용한다.
- ALL: 조건을 모두 만족하면 참이다.
- ANY 혹은 SOME: 둘은 같은 의미이다. 조건을 하나라도 만족하면 참이다.

**IN**
- 서브 쿼리의 결과 중 하나라도 같은 것이 있으면 참이다. 서브 쿼리가 아닌 고셍서도 사용한다.

### ➰ 조건식
**타입 표현**
JPQL에서 사용하는 타입은 아래와 같이 표시하고, 대소문자는 구분하지 않는다.

|종류|설명|예제|
|:---:|:---:|:---:|
|문자|작은 따옴표 사이에 표현|'HELLO', 'She''s'|
|숫자|L(Long), D(Double), F(Float)|10L, 10D, 10F|
|날짜|Date{d 'yyyy-dd-dd'}<br/>TIME{t 'hh-mm-ss'}<br/>DATETIME{ts 'yyyy-dd-dd hh:mm:ss.f'}|{d '2012-03-24'}|
|Boolean|TRUE, FALSE||
|Enum|패키지명을 포함한 전체 이름을 사용해야 한다.|jpabook.MemberType.Admin|
|엔티티 타입|엔티티의 타입을 표현한다. 주로 상속과 관련해서 사용한다.|TYPE(m) = Member|

**연산자 우선 순위**
1. 경로 탐색 연산(.)
2. 수학 연산
3. 비교 연산
4. 논리 연산

**컬렉션 식**<br/>
컬렉션에만 사용하는 특별한 기능이다. 컬렉션은 컬렉션 식 이외에 다른 식은 사용할 수 없다.
- 빈 컬렉션 비교 식
  - { 컬렉션 값 연관 경로 } IS [NOT] EMPTY
  - 컬렉션에 값이 비었으면 참
- 컬렉션의 멤버 식
  - { 엔티티나 값 } [NOT] MEMBER [OF] { 컬렉션 값 연관 경로 }
  - 엔티티나 값이 컬렉션에 포함되어 있으면 참

**스칼라 식**<br/>
숫자, 문자, 날짜, case, 엔티티 타입 같은 가장 기본적인 타입들을 스칼라라고 한다.
- 수학 식
  - 단항 연산자, 사칙연산
- 문자함수
  - CONCAT, SUBSTRING, TRIM, LOWER, UPPER, LENGTH, LOCATE
- 수학 함수
  - ABS, SQRT, MOD, SIZE, INDEX
- 날짜 함수
  - 데이터베이스의 현재 시간을 조회한다.
  - CURRENT_DATE, CURRENT_TIME, CURRENT_TIMESTAMP

**CASE 식**<br/>
특정 조건에 따라 분기할 때 CASE 식을 사용한다.
- 기본 CASE
  - ```sql
    CASE {WHEN <조건식> THEN <스칼라식>} + 
      ELSE <스칼라식>
    END
    ```
- 심플 CASE
  - 조건식을 사용할 수 없지만, 문법이 단순하다.
  - ```sql
    CASE <조건대상> + {WHEN <스칼라식1> THEN <스칼라식2>} +
      ELSE <스칼라식>
    END
    ```
- COALESCE
  - 스칼라식을 차례대로 조회해서 `null` 이 아니면 반환한다.
  - `COALESCE(<스칼라식> {,<스칼라식>}+)`
- NULLIF
  - 두 값이 같으면 `null` 을 반환하고 다르면 첫 번째 값을 반환한다.
  - 집합 함수는 `null` 을 포함하지 않아 보통 집합 함수와 함께 사용한다.
  - `NULLIF(<스칼라식>, <스칼라식>)`

### ➰ 다형성 쿼리
JPQL로 부모 엔티티를 조회하면 그 자식 엔티티도 함께 조회한다.
- `Item` 의 자식 엔티티로 `Book` , `Album` , `Movie` 가 있다고 가정한다.
- `Item` 을 조회하면 그 자식도 함께 조회한다.

**TYPE**<br/>
엔티티의 상속 구조에서 조회 대상을 특정 자식 타입으로 한정할 때 사용한다.

```java
// Item 중 Book, Movie를 조회하라.

// JPQL
select i from Item i
where type(i) IN (Book, Movie)

// SQL
SELECT i FROM Item i
WHERE i.DTYPE in ('B', 'M')
```

**TREAT**<br/>
JPA 2.1에 추가된 기능으로, 자바의 타입 캐스팅과 비슷하다. 상속 구조에서 부모 타입을 특정 자식 타입으로 다룰 때 사용한다.\
```java
// JPQL 
select i from Item i where treat(i as Book).author = 'kim'
```
- 부모 타입인 `Item` 을 자식 타입인 `Book` 으로 다뤄 `author` 필드에 접근할 수 있다.

### ➰ 사용자 정의 함수 호출(JPA 2.1)
`function_invocation::= FUNCTION(function_name {, function_arg})`
- Hibernate 구현체를 사용하면 **방언 클래스를 상속**해서 구현하고 사용할 데이터베이스 함수를 미리 등록해야 한다.
- 그리고 `hibernate.dialect` 에 해당 방언을 등록해야 ㅎ나다.

### ➰ 기타 정리
- `enum` 은 = 비교 연산만 지원한다.
- 임베디드 타입은 비교를 지원하지 않는다.

**EMPTY STRING**<br/>
JPA 표준은 ''을 길이 0인 Empty String으로 정했지만, 데이터베이스에 따라 `NULL` 로 사용하는 데이터베이스도 있어 확인이 필요하다.

**NULL 정의**
- 조건을 만족하는 데이터가 하나도 없는 경우
- `NULL` 은 알 수 없는 값이다. `NULL` 과의 모든 수학적 계산 결과는 `NULL` 이다.
- `Null == Null` 은 알 수 없는 값이다.
- `Null is Null` 은 참이다.

### ➰ 엔티티 직접 사용
**기본 키 값**<br/>
객체 인스턴스는 참조 값으로 식별하고, 테이블 로우는 기본 키 값으로 식별한다.
- 따라서 **JPQL에서 엔티티 객체를 직접 사용**하면 **SQL에서는 해당 엔티티의 기본 키 값을 사용**한다.

**외래 키 값**<br/>
특정 팀에 소속된 회원을 찾는 예제로 확인한다.
```java
Team team = em.find(Team.class, 1L);

String qlString = "select m from Member m where m.team = :team";
List resultList = em.createQuery(qlString)
    .setParameter("team", team);
    .getResultList();
```
- 기본 키 값이 1L인 팀 엔티티를 파라미터로 사용하고 있다.
- `m.team` 은 현재 `team_id` 라는 외래 키와 매핑되어 있다.

```sql
select m.* from Member m where m.team_id = ?
```

엔티티 대신 식별자 값을 직접 사용할 수 있다.
```java
String qlString = "select m from Member m where m.team.id = :teamId";
List resultList = em.createQuery(qlString)
    .setParameter("teamId", 1L);
    .getResultList();
```
- `m.team.id` 를 보면 `Member` 와 `Team` 간 묵시적 조인이 발생할 것 같지만, `MEMBER` 테이블이 외래 키를 가지고 있어 발생하지 않는다.

> `m.team` 을 사용하든, `m.team.id` 를 사용하든 생성되는 SQL은 같다.

### ➰ Named 쿼리: 정적 쿼리
JPQL은 크게 동적 쿼리와 정적 쿼리로 나눌 수 있다.
- **동적 쿼리**
  - JPQL을 문자로 완성해서 직접 넘기는 것
  - 런타임에 특정 조건에 따라 동적으로 구성할 수 있다.
- **정적 쿼리**
  - 미리 정의한 쿼리에 이름을 부여해 필요할 때 사용하는 것, Named 쿼리라 한다.
  - 한 번 정의하면 변경할 수 없다.

Named 쿼리는 애플리케이션 로딩 시점에 JPQL 문법 체크 후 미리 파싱해둔다.
- 오류를 빨리 확인할 수 있고, 사용하는 시점에는 파싱된 결과를 재사용하여 성능상 이점도 있다.
- 정적 SQL을 생성하므로, 데이터베이스의 조회 성능 최적화에도 도움이 된다.

**Named 쿼리를 어노테이션에 정의**<br/>
`@NamedQuery` annotation을 사용한다.
```java
@Entity
@NamedQuery (
  name = "Member.findByUsername",
  query = "select m from Member m where m.username = :username")
public class Member {
  ...

}
```
- `name` 에 이름을 부여하고, `query` 에 사용할 쿼리를 입력한다.
- 사용 시에는 `em.createNamedQuery()` 메소드에 Named 쿼리 이름을 입력하여 사용한다.
- 여러 개의 쿼리를 사용하기 위해서는 `@NamedQueries` annotation을 사용할 수 있다.

> 엔티티 이름을 쿼리 이름 앞에 붙여 영속성 유닛 단위로 관리되는 Named 쿼리의 충돌을 방지한다.

`@NamedQuery` annotation
- `lockMode` : 쿼리 실행 시 락을 건다. (default: NONE)
- `hints` : SQL 힌트가 아니라 JPA 구현체에게 제공하는 힌트다. 

**Named 쿼리를 XML에 정의**<br/>
JPA에서 annotation으로 작성할 수 있는 것은 XML로도 작성할 수 있다. 
- 자바에서 멀티라인 문자를 다루는 것은 상당히 귀찮은데, XML을 사용하는 것이 현실적인 대안이 될 수 있다.

```xml
<!-- META-INF/ormMember.xml -->
<named-query name="Member.findByUsername">
  <query><CDATA[
    select m
    from Member m
    where m.username = :username
  ]></query>
</named-query>

<named-query name="Member.count">
  <query>select count(m) from Member m</query>
</named-query>
```

작성한 xml을 인식하도록 코드를 추가한다.

```xml
<!-- META-INF/persistence.xml -->
<persistence-unit name="jpabook">
  <mapping-file>META-INF/ormMember.xml</mapping-file>
  ...

```

> `META-INF/orm.xml` 은 기본 매핑 파일로 인식한다.

**환경에 따른 설정**<br/>
XML과 annotation에 같은 설정이 있다면 **XML이 우선권**을 가진다.

---

## 💫 Criteria
JPQL을 자바 코드로 작성하도록 도와주는 빌더 클래스 API이다.
- 문자가 아닌 코드로 JPQL을 작성해 문법 오류를 컴파일 단계에서 확인할 수 있다.
- 문자 기반의 JPQL보다 동적 쿼리를 안전하게 생성할 수 있다.
- 코드가 복잡하고 장황해서 직관적인 이해가 어렵다는 단점이 있다.

### ➰ Criteria 기초
Criteria API는 `javax.persistence.criteria` 패키지에 있다.

```java
CriteriaBuilder cb = em.getCriteriaBuilder();

CriteriaQuery<Member> cq = cb.createQuery(Member.class); // 반환타입 지정

Root<Member> m = cq.from(Member.class); // FROM 절
cq.select(m);

TypedQuery<Member> query = em.createQuery(cq);
List<Member> members = query.getResultList();
```
- Criteria 쿼리 생성을 위해서는 Criteria 빌더를 얻어야 한다. EntityManager나 EntityManagerFactory에서 얻을 수 있다.
- Criteria 쿼리 빌더에서 Criteria 쿼리를 생성한다. 이때 반환 타입을 지정할 수 있다.
- FROM 절을 생성한다. 반환된 값은 별칭이고, 이를 조회의 시작점으로 잡는다.

검색 조건과 정렬 조건을 추가한다.
```java
CriteriaBuilder cb = em.getCriteriaBuilder();

CriteriaQuery<Member> cq = cb.createQuery(Member.class); 

Root<Member> m = cq.from(Member.class); 

Predicate usernameEqual = cb.equal(m.get("username"), "회원1");

javax.persistence.criteria.Order ageDesc = cb.desc(m.get("age"));

cq.select(m)
    .where(usernameEqual)
    .orderBy(ageDesc);

List<Member> resultList = em.createQuery(cq).getResultList();
```

**쿼리 루트와 별칭**
- 쿼리 루트는 조회의 시작점이다.
- 별칭은 엔티티에만 부여할 수 있다.

`m.get("age")` 메소드는 `age` 의 타입 정보를 모르기 때문에 `m.<Integer>get("age")` 와 같이 제네릭으로 반환 타입 정보를 명시해야 한다.(`String` 같은 문자 타입은 지정하지 않아도 된다.)

### ➰ Criteria 쿼리 생성
`CriteriaBuilder.createQuery()` 메소드로 Criteria 쿼리를 생성해 사용한다.
- `CriteriaBuilder` interface를 보면, 쿼리 생성 시 파라미터로 쿼리 결과에 대한 반홭 타입을 지정할 수 있다.
  - 반환 타입을 지정했다면, `em.createQuery()` 에서 반환 타입을 지정하지 않아도 된다.
- 반환 타입을 지정할 수 없거나, 반환 타입이 둘 이상이면 타입 명시 없이 `Object` 로 반환받는다.
  - 물론 반환 타입이 둘 이상미녀 `Object[]` 를 사용하는 것이 편리하다.

### ➰ 조회
`CriteriaQuery` interface 의 `select` , `multiselect` 메소드

**조회 대상을 한 건, 여러 건 지정**
- `select` 에 조회 대상을 하나만 지정하려면 `cq.select(m)` 과 같이 작성한다.
- 조회 대상을 여러 건 지정하려면 `cq.multiselect(m.get("username"), m.get("age"));` 과 같이 작성한다.
  - 여러 건 지정은 `cb.array` 사용도 가능하다.

**DISTINCT**<br/>
`select` , `multiselect` 다음에 `distinct(true)` 로 사용한다.

**NEW, construct()**<br/>
JPQL에서 `select new 생성자()` 구문을 Criteria에서는 `cb.construct(클래스 타입, ...)` 로 사용한다.

**튜플**<br/>
Criteria는 `Map` 과 비슷한 **튜플**이라는 특별한 반환 객체를 제공한다.

```java
CriteriaQuery<Tuple> cq = cb.createTupleQuery();

cq.multiselect (
  m.get("username").alias("username"), // 튜플에서 사용할 별칭 지정
  m.get("age").alias("age")
);

TypedQuery<Tuple> query = em.createQuery(cq);
List<Tuple> resultList = query.getResultList();
for (Tuple tuple : resultList) {
  // 튜플 별칭으로 조회
  String username = tuple.get("username", String.class);
  Integer age = tuple.age("age", Integer.class);
}
```
- 튜플을 사용하려면 `cb.createTupleQuery` 또는 `cb.createQuery(Tuple.class)` 로 Criteria를 생성한다.
- 튜플은 튜플의 검색 키로 사용할 **튜플 전용 별칭을 필수로 할당**해야 한다.
- 선언해둔 튜플 별칭으로 데이터를 조회한다.
- 튜플은 이름 기반으로, 순서 기반의 `Object[]` 보다 안전하다. 
- `tuple.getElements()` 같은 메소드로 현재 튜플의 별칭과 자바 타입도 조회할 수 있다. 

### ➰ 집합
**GROUP BY**
```java
cq.groupBy(m.get("team").get("name"))
```
- JPQL의 `group by m.team.name` 과 같다.

**HAVING**
```java
cq.multiselect(m.get("team").get("name"), maxAge, minAge)
    .groupBy(m.get("team").get("name"))
    .having(cb.gt(minAge, 10));
```
- JPQL의 `having min(m.age) > 10` 과 같다.

### ➰ 정렬
`cb.desc()...` 또는 `cb.asc(...)` 로 생성할 수 있다.

### ➰ 조인
`join()` 메소드와 `JoinType` 클래스를 사용한다.

```java
public enum JoinType {

  INNER, (default)
  LEFT,
  RIGHT

}
```
```java
Root<Member> m = cq.from(Member.class);
Join<Member, Team> t = m.join("team", JoinType.INNER); // 내부 조인 설정

cq.multiselect(m, t);
  .where(cb.equal(t.get("name"), "팀A"));
```

FETCH JOIN은 다음과 같이 설정한다.
```java
Root<Member> m = cq.from(Member.class);
m.fetch("team", JoinType.LEFT);

cq.select(m);
```

### ➰ 서브 쿼리
**간단한 서브 쿼리**<br/>
메인 쿼리와 서브 쿼리 간 관련이 없는 서브 쿼리를 말한다.

```java
CriteriaQuery<Member> mainQuery = cb.createQuery(Member.class);

Subquery<Double> subQuery = mainQuery.subquery(Double.class);
Root<Member> m2 = subQuery.from(Member.class);
subQuery.select(cb.avg(m2.<Integer>get("age")));

Root<Member> m = mainQuery.from(Member.class);
mainQuery.select(m)
  .where(cb.ge(m.<Integer>get("age"), subQuery));
```
1. 서브 쿼리는 `mainQuery.subquery()` 로 생성한다.
2. 메인 쿼리는 `where(..., subQuery)` 에서 생성한 서브 쿼리를 사용한다.

**상호 관련 서브 쿼리**<br/>
메인 쿼리와 서브 쿼리 간 관련이 있는 경우이다.
- 서브 쿼리에서 메인 쿼리의 정보를 사용하려면 메인 쿼리에서 사용한 별칭을 얻어야 한다.
  - 메인 쿼리의 `Root` 나 `Join` 을 통해 생성된 별칭을 받아 사용한다. 이때 `Root<Member> subM = subQuery.correlate(m);` 을 사용해 메인 쿼리의 별칭을 가져올 수 있다.

### ➰ IN 식
Criteria 빌더에서 `in(...)` 메소드를 사용한다.

### ➰ CASE 식
`selectCase()` 메소드와 `when()` , `otherwise()` 메소드를 사용한다.

### ➰ 파라미터 정의
JPQL에서 `:PARAM1` 처럼 파라미터를 정의했듯 정의할 수 있다.

```java
cq.select(m)
    .where(cb.equal(m.get("username"), cb.parameter(String.class, "usernameParam"))); // 파라미터 정의

List<Member> resultList = em.createQuery(cq)
    .setParameter("usernameParam", "회원1") // 바인딩
    .getResultList();
```

### ➰ 네이티브 함수 호출
네이티브 SQL 함수 호출을 위해서는 `cb.function(...)` 메소드를 사용하면 된다.

### ➰ 동적 쿼리
다양한 검색 조건에 따라 실행 시점에 쿼리를 생성하는 것을 동적 쿼리라 한다.
- 코드 기반인 Criteria로 작성하는 것이 편리하다.
- 공백이나 `where` , `and` 의 위치로 인한 에러가 발생하지는 않는다.
- 하지만 장황하고 복잡하다는 단점이 크다.

### ➰ 함수 정리
Criteria는 JPQL 빌더 역할을 해 JPQL 함수를 코드로 지원한다.
- Expression의 메소드(`isNull()` , `isNotNull()` , `in()`)
- 조건 함수
- 스칼라와 기타 함수
- 집합 함수
- 분기 함수

### ➰ Criteria 메타 모델 API
코드 기반의 Criteria는 컴파일 시점에 오류를 발견할 수 있다.
- 하지만 `m.get("age")` 에서 `age` 는 문자다. 이를 `aggeadf` 라고 적어도 컴파일 시점에 에러를 발견하지 못한다.
  - 따라서 완전한 코드 기반이라 할 수 없다.
- 이런 부분까지 코드로 작성하기 위해서는 메타 모델 API를 사용하면 된다. 먼저 메타 모델 클래스를 생성한다.

**메타 모델 API 적용 전**
```java
cq.select(m)
    .where(cb.gt(m.<Integer>get("username"), 20))
    .orderBy(cb.desc(m.get("age")));
```
**메타 모델 API 적용 후**
```java
cq.select(m)
    .where(cb.gt(m.get(Member_.age),20))
    .orderBy(cb.desc(m.get(Member_.age)));
```
- 문자 기반에서 정적인 코드 기반으로 변경되었다. 
  - 이를 위해서는 `Member_` 클래스가 필요한데, 이를 **메타 모델 클래스**라 한다.
- 메타 모델 클래스는 해당 엔티티를 기반이고, 코드 자동 생성기가 `엔티티명_.java` 모양의 메타 모델 클래스를 생성해준다.
  - Hibernate 구현체는 `org.hibernate.jpamodelgen.JPAMetaModelEntityProcessor` 를 사용한다.

**코드 생성기 설정**<br/>
보통 메이븐이나 엔트, 그래들 같은 빌드 도구를 사용해 코드 생성기를 실행한다.
- 메이븐을 기준으로, `hibernate-jpamodelgen` 과 `maven-compiler-plugin` 설정을 추가한다.
- `mvn compile` 명령어를 사용해 메타 모델 클래스를 생성한다.
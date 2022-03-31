## QueryDSL
> ***JPA Criteria 장점 / 단점***  
> - 장점 : 문법 오류를 컴파일 단계에서 잡을 수 있고 IDE 자동완성 기능의 도움을 받을 수 있다.  
> - 단점 : 코드를 봐도 복잡성으로 인해 어떤 JPQL이 생성될지 파악하기 쉽지 않다.  

**QueryDSL**  
=> 쿼리를 문자가 아닌 코드로 작성해도 쉽고 간결하며 모양도 쿼리와 비슷하게 개발할 수 있는 프로젝트
- 오픈소스 프로젝트
- HQL을 코드로 작성할수 있도록 해주는 프로젝트로 시작해서 지금은 JPA, JDO, JDBC, 몽고DB, 자바 컬렉션 등을 다양하게 지원
- 데이터를 조회하는 기능에 특화

### 1. QueryDSL 설정
**필요 라이브러리**  
```xml
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-jpa</artifactId>
  <version>3.6.3</version>
</dependency>

<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>3.6.3</version>
  <scope>provided</scope>
</dependency>
```
- querydsl-jpa : QueryDSL JPA 라이브러리
- querydsl-apt : 쿼리 타입(Q)을 생성할 때 필요한 라이브러리

**환경설정**  
Criteria의 메타 모델처럼 엔터티를 기반으로 쿼리 타입이라는 쿼리용 클래스를 생성해야 함  
쿼리 타입 생성용 플러그인을 pom.xml에 추가  
=> 콘솔에서 mvn compile을 입력하면 outputDirectory에 지정한 target/generated-sources 위치에 QMember.java처럼 Q로 시작하는 쿼리 타입들이 생성됨  

### 2. 시작

```java
public void queryDSL() {
  EntityManager em = emf.createEntityManager();
  
  JPAQuery query = new JPAQuery(em);
  QMember qMember = new QMember("m");
  List<Member> members = 
    query.from(qMember)
      .where(qMember.name.eq("회원1"))
      .orderBy(qMember.name.desc())
      .list(qMember);
}
```

- com.mysema.query.jpa.impl.JPAQuery 객체를 생성하기 위해 엔터티 매니저를 생성자에게 넘겨준다.  
- 사용할 쿼리 타입(Q)을 생성할 때, 생성자에 별칭을 주고 이 별칭을 JPQL에서 별칭으로 사용  

**기본 Q 생성**  
쿼리 타입(Q)은 사용하기 편리하도록 기본 인스턴스를 보관하고 있지만  
같은 엔터티를 조인하거나 같은 엔터티를 서브쿼리에 사용하면 같은 별칭이 사용되므로 별칭을 직접 지정해서 사용해야 한다.  
```java
QMember qMember = new QMember("m"); // 직접 지정
QMember qMember = QMember.member;   // 기본 인스턴스 사용
```
쿼리 타입의 기본 인스턴스를 사용하면 `import static`을 활용해서 코드를 더 간결하게 작성할 수 있음  

### 3. 검색 조건 쿼리  

```java
JPAQuery query = new JPAQuery(em);
QItem item = QItem.item;
List<Item> list = query.from(item)
      .where(item.name.eq("좋은상품").and(item.price.gt(20000)))
      .list(item);  //조회할 프로섹션 지정
```
실행하면 다음과 같은 JPQL이 생성/실행됨  
```sql
select item
from Item item
where item.name = ?1 and item.price > ?2
```
- where 절에는 and나 or을 사용할 수 있다.  
- 콤마를 이용해서 여러 검색 조건을 사용할 수 있음 (and 연산)  
- between, contains, startsWith 등 필요한 대부분의 메소드를 명시적으로 제공함  

### 4. 결과 조회  
결과 조회 메소드를 호출하면 실제 데이터베이스를 조회한다.  
보통 uniqueResult()나 list()를 사용하고 파라미터로 프로젝션 대상을 넘겨준다.  
- uniqueResult() : 조회 결과가 한 건일 때 사용 / 결과가 없으면 null을 반환 / 결과가 하나 이상이면 com.mysema.query.NonUniqueResultException 예외 발생  
- singleResult() : uniqueResult()와 같지만 결과가 하나 이상이면 처음 데이터를 반환  
- list() : 조회 결과가 하나 이상일 때 사용 / 결과가 없으면 빈 컬렉션을 반환  

### 5. 페이징과 정렬  
```java
QItem item = QItem.item;

query.from(item)
  .where(item.price.gt(20000))
  .orderBy(item.price.desc(), item.stockQuantity.asc())
  .offset(10).limit(20)
  .list(item);
```
- 정렬은 orderBy를 사용하고 쿼리 타입(Q)이 제공하는 asc(), desc()를 사용  
- 페이징은 offset과 limit을 적절히 조합해서 사용  
- 페이징은 restrict() 메소드에 com.mysema.query.QueryModifiers를 파라미터로 사용해도 된다.  
- 실제 페이징 처리를 하려면 검색된 전체 데이터 수를 알아야 하는데 list() 대신 listResults()를 사용  
- listResults()를 사용하면 전체 데이터 조회를 위한 count 쿼리를 한번 더 실행하고 전체 데이터 수를 조회할 수 있는 객체인 SearchResults를 반환  

### 6. 그룹  
```java
query.from(item)
  .groupBy(item.price)
  .having(item.price.gt(1000))
  .list(item);
```
groupBy를 사용하고 그룹화된 결과를 제한하려면 having을 사용  

### 7. 조인  
innerJoin, leftJoin, rightJoin, fullJoin을 사용할 수 있고, JPQL의 on과 성능 최적화를 위한 fetch 조인도 사용할 수 있다.  
첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭으로 사용할 쿼리 타입을 지정하면 된다.  
```java
QOrder order = QOrder.order;
QMember member = QMember.member;
QOrderItem orderItem = QOrderItem.orderItem;

query.from(order)
  .join(order.member, member)
  .leftJoin(order.orderItems, orderItem)
  .list(item);
```

### 8. 서브 쿼리  
com.mysema.query.jpa.JPASubQuery를 생성해서 사용  
서브 쿼리의 결과가 하나면 unique(), 여러 건이면 list()를 사용  
```java
QItem item = QItem.item;
QItem itemSub = new QItem("itemSub");

query.from(item)
  .where(item.price.eq(
    new JPASubQuery().from(itemSub).unique(itemSub.price.max())
  ))
  .list(item);
```

### 9. 프로젝션과 결과 반환  
프로젝션 : select 절에 조회 대상을 지정하는 것  

**프로젝션 대상이 하나**  
프로젝션 대상이 하나면 해당 타입으로 반환  

**여러 컬럼 반환과 튜플**  
프로젝션 대상으로 여러 필드를 선택하면 기본으로 com.mysema.query.Tuple이라는 Map과 비슷한 내부 타입을 사용한다.  
조회 결과는 tuple.get() 메소드에 조회한 쿼리 타입을 지정하면 된다.  

**빈 생성**  
쿼리 결과를 엔터티가 아닌 특정 객체로 받고 싶으면 빈 생성(Bean population) 기능을 사용한다.  
QueryDSL은 객체를 생성하는 다양한 방법을 제공한다.  
- 프로퍼티 접근  
```java
QItem item = QItem.item;
List<ItemDTO> result = query.from(item).list(
  Projections.bean(ItemDTO.class, item.name.as("username"), item.price));
```
쿼리 결과와 매핑할 프로퍼티 이름이 다르면 as를 사용해서 별칭을 주면 된다.  

- 필드 직접 접근  
```java
QItem item = QItem.item;
List<ItemDTO> result = query.from(item).list(
  Projections.fields(ItemDTO.class, item.name.as("username"), item.price));
```
필드를 private로 설정해도 동작함  

- 생성자 사용  
```java
QItem item = QItem.item;
List<ItemDTO> result = query.from(item).list(
  Projections.constructor(ItemDTO.class, item.name, item.price));
```
지정한 프로젝션과 파라미터 순서가 같은 생성자가 필요하다.  

**DISTINCT**  
```java
query.distinct().from(item)...
```

### 10. 수정, 삭제 배치 쿼리  
JPQL 배치 쿼리와 같이 영속성 컨텍스트를 무시하고 데이터베이스를 직접 쿼리한다.  
```java
QItem item = QItem.item;
JPAUpdateClause updateClause = new JPAUpdateClause(em, item);
long count = updateClause.where(item.name.eq("시골개발자의 JPA 책"))
  .set(item.price, item.price.add(100))
  .execute();
```
수정 배치 쿼리는 com.mysema.query.jpa.impl.JPAUpdateClause를 사용  
```java
QItem item = QItem.item;
JPADeleteClause deleteClause = new JPADeleteClause(em, item);
long count = deleteClause.where(item.name.eq("시골개발자의 JPA 책"))
  .execute();
```
삭제 배치 쿼리는 com.mysema.query.jpa.impl.JPADeleteClause를 사용  

### 11. 동적 쿼리  
com.mysema.query.BooleanBuilder를 사용하면 특정 조건에 따른 동적 쿼리를 편리하게 생성할 수 있다.  
```java
SearchParam param = new SearchParam();
param.setName("시골개발자");
param.setPrice(10000);

QItem item = QItem.item;

BooleanBuilder builder = new BooleanBuilder();
if (StringUtils.hasText(param.getName())) {
  builder.and(item.name.contains(param.getName()));
}
if (param.getPrice() != null) {
  builder.and(item.price.gt(param.getPrice()));
}
List<Item> result = query.from(item)
  .where(builder)
  .list(item);
```

### 12. 메소드 위임  
쿼리 타입에 검색 조건을 직접 정의할 수 있다.  
- 메소드 위임 기능을 사용하려면 우선 정적 메소드를 만들고 @com.mysema.query.annotations.QueryDelegate 어노테이션에 속성으로 이 기능을 적용할 엔터티를 지정  
- 정적 메소드의 첫 번째 파라미터에는 대상 엔터티의 쿼리 타입(Q)을 지정하고 나머지는 필요한 파라미터를 정의  
```java
public class ItemExpression {
  @QueryDelegate(Item.class)
  public static BooleanExpression isExpensive(QItem item, Integer price) {
    return item.price.gt(price)
  }
}
```

---

## 네이티브 SQL  

JPQL은 표준 SQL이 지원하는 대부분의 문법과 SQL 함수들을 지원하지만 특정 데이터베이스에 종속하는 기능은 지원 X  
=> 특정 데이터베이스에 종속적인 기능을 지원하는 방법  
- 특정 데이터베이스만 지원하는 함수  
  - JPQL에서 네이티브 SQL 함수를 호출할 수 있다.  
  - 하이버네이트는 데이터베이스 방언에 각 데이터베이스에 종속적인 함수들을 정의해두었다. (직접 호출할 함수 정의도 가능)
- 특정 데이터베이스만 지원하는 SQL 쿼리 힌트  
  - 하이버네이트를 포함한 몇몇 JPA 구현체들이 지원  
- 인라인 뷰(From 절에서 사용하는 서브쿼리), UNION, INTERSECT  
  - 하이버네이트는 지원하지 않지만 일부 JPA 구현체들이 지원  
- 스토어드 프로시저  
  - JPQL에서 스토어드 프로시저를 호출할 수 있다.  
- 특정 데이터베이스만 지원하는 문법  
  - 오라클의 CONNECT BY처럼 특정 데이터베이스에 너무 종속적인 SQL 문법은 지원 X, 네이티브 SQL을 사용해야 함  

네이티브 SQL : JPQL을 사용할 수 없을 때 JPA에서 SQL을 직접 사용할 수 있도록 하는 기능  
JPQL을 사용하면 JPA가 SQL을 생성하지만, 네이티브 SQL은 이 SQL을 개발자가 직접 정의하는 것이다.  

> 네이티브 SQL vs JDBC API
> - 네이티브 SQL : 엔터티를 조회할 수 있고 JPA가 지원하는 영속성 컨텍스트의 기능을 그대로 사용할 수 있다.  
> - JDBC API : 직접 사용하면 단순히 데이터의 나열을 조회할 뿐이다.  

### 1. 네이티브 SQL 사용  

**엔터티 조회**  
```java
//SQL 정의
String sql = 
  "SELECT ID, AGE, NAME, TEAM_ID " + 
  "FROM MEMBER WHERE AGE > ?";
  
Query nativeQuery = em.createNativeQuery(sql, Member.class)
  .setParameter(1, 20);
List<Member> resultList = nativeQuery.getResultList();
```
em.createNativeQuery(SQL, 결과 클래스)를 사용  
첫 번째 파라미터는 네이티브 SQL을 입력하고, 두 번째 파라미터는 조회할 엔터티 클래스의 타입을 입력  
네이티브 SQL로 SQL만 직접 사용할 뿐이지 나머지는 JPQL을 사용할 때와 같다.  
조회한 엔터티도 영속성 컨텍스트에서 관리된다.  

**값 조회**  
```java
//SQL 정의
String sql = 
  "SELECT ID, AGE, NAME, TEAM_ID " + 
  "FROM MEMBER WHERE AGE > ?";
  
Query nativeQuery = em.createNativeQuery(sql)
  .setParameter(1, 10);
List<Member> resultList = nativeQuery.getResultList();
for (Object[] row : resultList) {
  System.out.println("id = " + row[0]);
  System.out.println("age = " + row[1]);
  System.out.println("name = " + row[2]);
  System.out.println("team_id = " + row[3]);
}
```
여러 값으로 조회하려면 em.creteNativeQuery(SQL)의 두 번째 파라미터를 사용하지 않으면 된다.  
JPA는 조회한 값들을 Object[]에 담아서 반환한다.  

**결과 매핑 사용**  

```java
//SQL 정의
String sql = 
  "SELECT ID, AGE, NAME, TEAM_ID, I.ORDER_COUNT " + 
  "FROM MEMBER M " + 
  "LEFT JOIN " +
  "   (SELECT IM.ID, COUNT(*) AS ORDER_COUNT " + 
  "   FROM ORDERS O, MEMBER IM " + 
  "   WHERE O.MEMBER_ID = IM.ID) I " + 
  "ON M.ID = I.ID";
  
Query nativeQuery = em.createNativeQuery(sql, "memberWithOrderCount");
List<Member> resultList = nativeQuery.getResultList();
for (Object[] row : resultList) {
  Member member = (Member) row[0];
  BigInteger orderCount = (BigInteger) row[1];
  
  System.out.println("member = " + member);
  System.out.println("orderCount = " + orderCount);
}
```
em.createNativeQuery(sql, "memberWithOrderCount")의 두 번째 파라미터에 결과 매핑 정보의 이름이 사용  
엔터티와 스칼라 값을 함께 조회하는 것처럼 매핑이 복잡해지면 @SqlResultSetMapping을 정의해서 결과 매핑을 사용해야 한다.  

**결과 매핑 어노테이션**  
@SqlResultSetMapping 속성  
- name : 결과 매핑 이름
- entities : @EntityResult를 사용해서 엔터티를 결과로 매핑
- columns : @ColumnResult를 사용해서 컬럼을 결과로 매핑

@EntityResult 속성  
- entityClass : 결과로 사용할 엔터티 클래스를 지정
- fields : @FieldResult를 사용해서 결과 컬름을 필드와 매핑
- discriminatorColumn : 엔터티의 인스턴스 타입을 구분하는 필드(상속에서 사용됨)

@FieldResult 속성  
- name : 결과를 받을 필드명  
- column : 결과 컬럼명

@ColumnResult 속성  
- name : 결과 컬럼명

### 2. Named 네이티브 SQL  
JPQL처럼 네이티브 SQL도 Named 네이티브 SQL을 사용해서 정적 SQL을 작성할 수 있다.  
@NamedNativeQuery로 Named 네이티브 SQL을 등록하고  
JPQL Named 쿼리와 같은 createNamedQuery 메소드를 사용하므로 TypeQuery를 사용할 수 있다.  
Named 네이티브 쿼리에서 resultSetMapping = "memberWithOrderCount"로 조회 결과를 매핑할 대상까지 지정한다.  

**@NamedNativeQuery**
- name : 네임드 쿼리 이름(필수)  
- query : SQL 쿼리(필수)
- hints : 벤더 종속적인 힌트  
- resultClass : 결과 클래스  
- resultSetMapping : 결과 매핑 사용  

### 3. 네이티브 SQL XML에 정의  
XML에 정의할 때는 `<named-native-query>`를 먼저 정의하고 `<sql-result-set-mapping>`을 정의하는 순서를 지켜야 한다.  

### 4. 스토어드프로시저  
**스토어드 프로시저 사용**  
단순히 입력 값을 두 배로 증가시켜 주는 proc_multiply라는 스토어드 프로시저가 있는데 이 프로시저는 첫 번째 파라미터로 값을 입력받고 두 번째 파라미터로 결과를 반환한다.  
스토어드 프로시저를 사용하려면 em.createStoredProcedureQuery() 메소드에 사용할 스토어드 프로시저 이름을 입력하면 된다.  
그 후, registerStoredProcedureParameter() 메소드를 사용해서 프로시저에서 사용할 마라미터를 순서, 타입, 파라미터 모드 순으로 정의하면 된다.  

**Named 스토어드 프로시저 사용**  
스토어드 프로시저 쿼리에 이름을 부여해서 사용하는 것을 Named 스토어드 프로시저라 한다.  
@NamedStoredProcedureQuery로 정의하고 name 속성으로 이름을 부여한다.  
procedureName 속성에 실제 호출할 프로시저 이름을 적어주고, @StoredProcedureParameter를 사용해서 파라미터 정보를 정의  
둘 이상을 정의하려면 @NamedStoredProcedureQueries를 사용  

---

## 객체지향 쿼리 심화  
### 1. 벌크 연산  
여러 엔터티를 한 번에 수정하거나 삭제하기 위해 벌크 연산 사용  
```java
String qlString = 
  "update Product p " +
  "set p.price = p.price * 1.1 " +
  "where p.stockAmount < :stockAmount";
  
int resultCount = em.createQuery(qlString)
                    .setParameter("stockAmount", 10)
                    .executeUpdate();
```
벌크 연산은 executeUpdate() 메소드를 사용하고 이 메소드는 벌크 연산으로 영향을 받은 엔터티 건수를 반환한다.  

**벌크 연산의 주의점**  
벌크 연산이 영속성 컨텍스트를 무시하고 데이터베이스에 직접 쿼리한다는 점에 주의해야 한다.  
이런 문제를 해결하는 방법으로는  
- **em.refresh() 사용**  
  `em.refresh()`를 사용해서 데이터베이스에서 다시 조회  
  
- **벌크 연산 먼저 실행**  
  가장 실용적인 해결책은 벌크 연산을 가장 먼저 실행하는 것  
  JPA와 JDBC를 함께 사용할 때도 유용  

- **벌크 연산 수행 후 영속성 컨텍스트 초기화**  
  벌크 연산을 수행한 직후에 바로 영속성 컨텍스트를 초기화해서 영속성 컨텍스트에 남아 있는 엔터티를 제거  
  영속성 컨텍스트를 초기화하면 이후 엔터티를 조회할 때 벌크 연산이 적용된 데이터베이스에서 엔터티를 조회한다.  
  
### 2. 영속성 컨텍스트와 JPQL  

**쿼리 후 영속 상태인 것과 아닌 것**  
JPQL로 엔터티를 조회하면 영속성 컨텍스트에서 관리되지만 엔터티가 아니면 영속성 컨텍스트에서 관리되지 않는다.  
임베디드 타입은 조회해서 값을 변경해도 영속성 컨텍스트가 관리하지 않으므로 변경 감지에 의한 수정이 발생하지 않는다.  
조회한 엔터티만 영속성 컨텍스트가 관리한다.  

**JPQL로 조회한 엔터티와 영속성 컨텍스트**  
JPQL로 데이터베이스에서 조회한 엔터티가 영속성 컨텍스트에 이미 있으면 JPQL로 데이터베이스에서 조회한 결과를 버리고 대신에 영속성 컨텍스트에 있던 엔터티를 반환한다.  
1. JPQL을 사용해서 조회 요청  
2. JPQL은 SQL로 변환되어 데이터베이스를 조회  
3. 조회한 결과와 영속성 컨텍스트를 비교  
4. 식별자 값을 기준으로 member1은 이미 영속성 컨텍스트에 있으므로 버리고 기존에 있던 member1이 반환 대상이 된다.  
5. 식별자 값을 기준으로 member2는 영속성 컨텍스트에 없으므로 영속성 컨텍스트에 추가한다.  
6. 쿼리 결과인 member1, member2를 반환한다. 여기서 member1은 쿼리 결과가 아닌 영속성 컨텍스트에 있던 엔터티다.  

=>  
- JPQL로 조회한 엔터티는 영속 상태다.  
- 영속성 컨텍스트에 이미 존재하는 엔터티가 있으면 기존 엔터티를 반환한다.  

영속성 컨텍스트는 영속 상태인 엔터티의 동일성을 보장한다.  
`em.find()`로 조회하든 JPQL을 사용하든 영속성 컨텍스트가 같으면 동일한 엔터티를 반환한다.  

**find() vs JPQL**  
- `em.find()`
  엔터티를 영속성 컨텍스트에서 먼저 찾고 없으면 데이터베이스에서 찾는다.  
  해당 엔터티가 영속성 컨텍스트에 있으면 메모리에서 바로 찾으므로 성능상 이점이 있다.  
- **JPQL**  
  항상 데이터베이스에 SQL을 실행해서 결과를 조회한다.  
 
`em.find()` 메소드는 영속성 컨텍스트에서 엔터티를 먼저 찾고 없으면 데이터베이스를 조회하지만 JPQL을 사용하면 데이터베이스를 먼저 조회한다.  

**JPQL 특징**  
- JPQL은 항상 데이터베이스를 조회한다.  
- JPQL로 조회한 엔터티는 영속 상태다.  
- 영속성 컨텍스트에 이미 존재하는 엔터티가 있으면 기존 엔터티를 반환한다.  

### 3. JPQL과 플러시 모드  
플러시는 영속성 컨텍스트의 변경 내역을 데이터베이스에 동기화하는 것  
JPA는 플러시가 일어날 때, 영속성 컨텍스트에 등록, 수정, 삭제한 엔터티를 찾아서 INSERT, UPDATE, DELETE SQL을 만들어 데이터베이스에 반영한다.  
`em.flush()`를 직접 사용해도 되지만 보통 FlushMode에 따라 커밋하기 직전이나 쿼리 실행 직전에 자동으로 플러시가 호출된다.  
```java
em.setFlushMode(FlushModeType.AUTO);  //커밋 또는 쿼리 실행 시 플러시 (기본값)
em.setFlushMode(FlushModeType.COMMIT); // 커밋 시에만 플러시
```

**쿼리와 플러시 모드**  
JPQL은 영속성 컨텍스트에 있는 데이터를 고려하지 않고 데이터베이스에서 데이터를 조회한다.  
그래서 JPQL을 실행하기 전에 영속성 컨텍스트의 내용을 데이터베이스에 반영해야 한다.  

- 플러시 모드를 COMMIT으로 설정하면 쿼리를 실행할 때 플러시를 자동으로 호출하지 않는다.  
- 쿼리 실행 전에 플러시를 호출하고 싶으면  
  1. `em.flush()`를 사용해서 수동으로 플러시 하거나  
  2. `setFlushMode()`로 해당 쿼리에서만 사용할 플러시 모드를 **AUTO**로 변경하면 된다.  
    (이렇게 쿼리에 설정하는 플러시 모드는 엔터티 매니저에 설정하는 플러시 모드보다 우선권을 가진다.)  

**플러시 모드와 최적화**  
**COMMIT**모드를 사용하면 트랜잭션을 커밋할 때만 플러시하고 쿼리를 실행할 때는 플러시하지 않으므로 데이터 무결성에 문제를 일으킬 수 있다.  
하지만 플러시가 너무 자주 일어나는 상황에 이 모드를 사용하면 쿼리 시 발생하는 플러시 횟수를 줄여서 성능을 최적화할 수 있는 장점이 있다.  

- `FlushModeType.AUTO` : 쿼리와 커밋할 때 총 4번 플러시한다.  
- `FlushModeType.COMMIT` : 쿼리 시에만 1번 플러시한다.  

JPA를 통하지 않고 JDBC로 쿼리를 직접 실행하면 JPA는 JDBC가 실행한 쿼리를 인식할 방법이 없다.  
-> 별도의 JDBC 호출은 플러시 모드를 **AUTO** 설정해도 플러시가 일어나지 않는다.  
=> JDBC로 쿼리를 실행하기 직전에 `em.flush()`를 호출해서 영속성 컨텍스트의 내용을 데이터베이스에 동기화하는 것이 안전하다.  

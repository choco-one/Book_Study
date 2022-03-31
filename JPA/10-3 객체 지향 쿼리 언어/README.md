## QueryDSL
> JPA Criteria 장점 / 단점  
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
em.createNativeQuery(SQL, 결과 클래스)를 사용  
첫 번째 파라미터는 네이티브 SQL을 입력하고, 두 번째 파라미터는 조회할 엔터티 클래스의 타입을 입력  
네이티브 SQL로 SQL만 직접 사용할 뿐이지 나머지는 JPQL을 사용할 때와 같다.  
조회한 엔터티도 영속성 컨텍스트에서 관리된다.  


# 객체 지향 쿼리 언어

- 객체지향 쿼리 소개
- JPQL
- Criteria
- QueryDSL
- 네이티브 SQL
- 객체지향 쿼리 심화

- **JPQL → 가장 중요한 객체지향 쿼리 언어**

## 객체지향 쿼리 소개

- `EntityManager.find()` 를 사용하면 식별자로 엔티티 하나를 조회할 수 있음

- 식별자로 조회 `EntityManager.find()`
- 객체 그래프 탐색 `(a.getB().getC())`

- ORM을 사용하면 데이터베이스 테이블이 아닌 엔티티 객체를 대상으로 개발하므로 검색도 테이블이 아닌 엔티티 객체를 대상으로 하는 방법이 필요 → `JPQL`

- JPQL의 특징
    - 테이블이 아닌 객체를 대상으로 검색하는 객체지향 쿼리
    - SQL을 추상화해서 특정 데이터베이스 SQL에 의존하지 않음

- SQL → 데이터베이스 테이블을 대상으로 하는 데이터 중심의 쿼리
- JPQL → 엔티티 객체를 대상으로 하는 객체지향 쿼리
    
    → JPQL을 사용하면 JPA는 이 JPQL을 분석한 다음 적절한 SQL을 만들어 데이터베이스를 조회하고, 조회한 결과로 엔티티 객체를 생성해서 반환
    

- JPA가 공식 지원하는 기능
    - JPQL(Java Persistance Query Language)
    - Criteria 쿼리 : JPQL을 편하게 작성하도록 도와주는 API, 빌더 클래스 모음
    - 네이티브 SQL : JPA에서 JPQ 대신 직접 SQL을 사용할 수 있다
    
- 알아둘 가치가 있는 것
    - QueryDSL : Criteria 쿼리 처럼 JPQL을 편하게 작성하도록 도와주는 빌더 클래스 모음, 비표준 오픈소스 프레임워크
    - JDBC 직접사용, Mybatis 같은 SQL 매퍼 프레임워크 사용 : 필요하면 JDBC를 직접 사용할 수 있다

→ 제일 중요한 것은 `JPQL`!!!!

### JPQL 소개

- 엔티티 객체를 조회하는 객체지향 쿼리
- 문법은 SQL과 비슷하고 ANSI 표준 SQL이 제공하는 기능을 유사하게 지원
- SQL을 추상화해서 특정 데이터베이스에 의존하지 않음 → 방언만 변경하면 됨
- SQL보다 코드가 간결

```java
//쿼리 생성
String jpql = "select m from Member as m where m.username = 'kim'";
List<Member> resultList = em.createQuery(jpql, Member.class).getResultList();
```

→ Member 는 엔티티 이름, m.username은 테이블 컬럼명이 아니라 엔티티 객체의 필드명

### Criteria 쿼리 소개

- `JPQL`을 생성하는 빌더 클래스
- 문자가 아닌 `query.select(m).where(...)` 처럼 프로그래밍 코드로 JPQL을 작성할 수 있음
- JPQL은 오타가 있어도 컴파일은 성공하고 애플리케이션은 서버에 배포할 수 있지만, 해당 쿼리가 실행되는 런타임 시점에 오류가 발생
- 컴파일 시점에 오류를 발견할 수 있음
- IDE 쿼리를 사용하면 코드 자동완성을 지원한다
- 동적 쿼리를 작성하기 편하다
- 복잡하고 장황해서 사용하기 불편하고 Criteria로 작성한 코드도 한눈에 들어오지 않음

→ "select m from Member as m where m.username = 'kim'"를 Criteria로 작성

```java
//Criteria 사용준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);

//루트 클래스 (조회를 시작할 클래스)
Root<Member> m = query.from(Member.class);

//쿼리 생성
CriteriaQuery<Member> cq =  query.select(m)
								.where(cb.equal(m.get("usernamen", "kim"));

List<Meinber> resultList = em.createQuery(cq).getResultList();
```

→ 메타 모델을 사용하면 온전히 코드만 사용해서 쿼리를 작성할 수 있다

### QueryDSL 소개

- Criteria처럼 JPQL 빌더 역할
- 코드기반이며 단순하고 사용하기 쉬움
- QueryDSL과 Criteria를 비교하면 Criteria는 너무 복잡함

```java
//준비
JPAQuery query = new JPAQuery(em);
QMember member = QMember.member;

//쿼리, 결과조회
List<Member> members =  query.from(member)
    .where(member.username.eq("kim"))
    .list(member);
```

### 네이티브 SQL 소개

- `JPA`가 `SQL`을 직접사용할 수 있게 해주는 기능
- 특정 데이터베이스에 의존하는 기능을 사용해야 할 때가 있음
    
    → 오라클 데이터베이스 : `CONNECT BY`, 특정데이터베이스에서만 동작하는 `SQL` 힌트
    
- 특정 데이터베이스에 의존하는 `SQL`을 작성해야하는 단점
    
    → 데이터베이스를 변경하면 `SQL`도 수정해야함
    

```java
String sql = "SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = 'kim'";
List<Member> resultList =  em.createNativeQuery(sql, Member.class).getResultList();
```

### JDBC 직접 사용 마이바티스 같은 SQL 매퍼 프레임워크 사용

- **JDBC 커넥션**에 직접 접근하고 싶으면 `JPA`는 **JDBC 커넥션**을 획득하는 API를 제공하지 않으므로 JPA 구현체가 제공하는 방법을 사용해야 함
- JDBC나 마이바티스를 JPA와 함께 사용하면 영속성 컨텍스트를 적절한 시점에 강제로 플러시 해야함
- `JPA`를 우회하는 `SQL`에 대해서는 `JPA`가 전혀 인식하지 못하는 문제
- 영속성 컨텍스트와 데이터베이스를 불일치 상태로 만들어 데이터 무결성을 훼손하는 최악의 상황이 발생할 수도 있음
    
    → `JPA`를 우회해서 `SQL`을 실행하기 직전에 **영속성 컨텍스트**를 수동으로 플러시해서 데이터베이스와 영속성 컨텍스트를 동가화하면 됨
    

## JPQL

- `JPQL` 특징
    - `JPQL`은 객체지향 쿼리 언어, 따라서 테이블을 대상으로 쿼리하는 것이 아니라 엔티티 객체를 대상으로 쿼리
    - `JPQL`은 `SQL`을 추상화해서 특정 데이터베이스 `SQL`에 의존하지 않음
    - `JPQL`은 결국 `SQL`로 변환됨
    

### 기본 문법과 쿼리 API

- `JPQL`도 `SQL`과 비슷하게 `SELECT, UPDATE, DELETE` 문을 사용할 수 있음
- 엔티티를 저장할 때는 `EntityManager.persist()`를 사용하면 되므로 `INSERT`문은 없음

- **SELECT 문**
    - 대소문자 구분 : 엔티티와 속성은 **대소문자를 구분**
    - 엔티티 이름 : `JPQL`에서 사용하는 객체는 클래스 명이 아니라 엔티티 명이고 `@Entity(name=”XXX”)`로 지정할 수 있음
    - 별칭은 필수 : `JPQL`은 별칭을 필수로 사용해야 함
    
- **TypeQuery, Query**
    - 작성한 JPQL을 실행하려면 쿼리 객체를 만들어야 함
    - 반환할 타입을 명확하게 지정할 수 있으면 `TypeQuery` 객체를 사용
    - 반환 타입을 명확하게 지정할 수 없으면 `Query` 객체를 사용
    - `Query` 객체는 `SELECT` 절의 조회 대상이 둘 이상이면 `Object[]` 를 반환하고, 조회 대상이 하나면 `Object`를 반환
    - 타입을 변환할 필요가 없는 TypeQuery를 사용하는 것이 더 편리

- **결과 조회**
    - `query.getResultList()` : 결과를 예제로 반환, 결과가 없으면 빈 컬렉션을 반환
    - `query.getSingleResult()` : 결과가 정확히 하나일 때 사용
        - 결과가 없으면 `NoResultException` 발생
        - 1개보다 많으면 `NonUniqueResultException` 발생

### 파라미터 바인딩

- 이름 기준 파라미터
    - 파라미터를 이름으로 구분하는 방법
    - 앞에 : 를 사용
    
- 위치 기준 파라미터
    - ? 다음에 위치 값을 주면 됨

→ 위치 기준 파라미터 방식보다는 이름 기준 파라미터 바인딩 방식을 사용하는 것이 더 명확하다

→ 파라미터 바인딩 방식은 선택이 아닌 필수!

### 프로젝션

- SELECT 절에 조회할 대상을 지정하는 것
- 프로젝션 대상은 엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자 등 기본 데이터 타입)이 있음

- 엔티티 프로젝션
    - 원하는 객체를 바로 조회한 것
    - 컬럼을 하나하나 나열해서 조회해야하는 SQL과는 차이가 있음
    - 조회한 엔티티는 영속성 컨텍스트에서 관리

- 임베디드 타입 프로젝션
    - JPQL에서 임베디드 타입은 엔티티와 거의 비슷하게 사용
    - 임베디드 타입은 조회의 시작점이 될 수 없다는 제약이 있음
    - 임베디드 타입은 엔티티 타입이 아닌 값 타입이기 때문에 직접 조회한 임베디드 타입은 영속성 컨텍스트에서 관리 되지 않음

- NEW 명령어
    - 실제 애플리케이션 개발시에는 Object[]를 직접 사용하지 않고 의미있는 객체로 변환해서 사용
    - 패키지 명을 포함한 전체 클래스명을 입력해야 함
    - 순서와 타입이 일치하는 생성자가 필요함
    

### 페이징 API

- 페이징 처리용 SQL을 작성하는 일은 지루하고 반복적
- 데이터베이스마다 페이징을 처리하는 SQL 문법이 다름

- JPA는 페이징을 다음 두 API로 추상화
    - setFirstResult(int startPosition) : 조회 시작 위치(0부터 시작한다)
    - setMaxResults(int maxResult) : 조회할 데이터 수

### 집합과 정렬

- 집합 함수와 함께 통계 정보를 구할 때 사용

- 집합 함수
    
    
    | 함수 | 설명 |
    | --- | --- |
    | COUNT | 결과 수를 구한다. 반환 타입 : Long |
    | MAX,MIN | 최대, 최소 값을 구한다. 문자, 숫자, 날짜 등에 사용한다 |
    | AVG | 평균값을 구한다. 숫자 타입만 사용할 수 있다. 반환 타입 : Double |
    | SUM | 합을 구한다. 숫자 타입만 사용할 수 있다. 반환 타입 : 정수합→Long, 소수합→Double, BigInteger합→BigInteger, BigDecima합→BigDecimal  |
- GROUP BY, HAVING
    - GROUP BY 는 통계 데이터를 구할 때 특정 그룹끼리 묶어 줌
    - HAVING 은 그룹화한 통계 데이터를 기준으로 필터링

- 정렬(ORDER BY)
    - 결과를 정렬할 때 사용
    - ASC : 오름차순(default)
    - DESC : 내림차순

### JPQL 조인

- SQL 조인과 기능은 같고 문법만 약간만 다름

- 내부 조인
    - INNER JOIN을 사용(INNER 는 생략 가능)
    - 연관필드를 사용
        
        → 다른 엔티티와 연관관계를 가지기 위해 사용하는 필드
        

- 외부 조인
    - 기능상 SQL의 외부 조인과 같음
    - OUTER는 생략 가능해서 보통 LEFT JOIN 으로 사용

- 컬렉션 조인
    - 일대다 관계나 다대다 관계처럼 컬렉션을 사용하는 곳에 조인하는 것

- 세타 조인
    - WHERE 절을 사용해서 세타 조인을 할 수 있음
    - 내부 조인만 지원
    - 전혀 관계 없는 엔티티도 조인할 수 있음

- JOIN ON절(JPA 2.1)
    - ON 절을 사용하면 조인 대상을 필터링하고 조인할 수 있음
    - 외부조인에서만 사용

### 페치 조인

- SQL에서 이야기하는 조인의 종류는 아니고 JPQL에서 성능 최적화를 제공하는 기능
- 연관된 엔티티나 컬렉션을 한번에 같이 조회하는 기능

- 엔티티 페치 조인
    - 연관된 엔티티나 컬렉션을 함께 조회
    - 별칭을 사용할 수 없음 - (하이버네이트는 페치 조인에도 별칭을 허용)
    
    ```java
    String jpql = "select m from Member m join fetch m.team";
    List<Member> members = em.createQuery(jpql, Member.class).getResultList();
    for (Member member : members) {
      //페치 조인으로 회원과 팀을 함께 조회해서 지연 로딩 발생 안 함
      Systern.out.printin("username = " + member.getUsername () + ", " +
        "teamname = " + member.getTeam().name());
    }
    ```
    

- 컬렉션 페치 조인
    
    ```sql
    select t
    from Team t join fetch t.members
    where t.name = '팀A'
    ```
    
    - 일대다 조인은 결과가 증가할 수 있지만 일대일, 다대일 조인은 결과가 증가하지 않는다

- 페치 조인과 DISTINCT
    - SQL의 DISTINCT는 중복된 결과를 제거하는 명령어
    - JPQL의 DISTINCT 명령어는 SQL에 DISTINCT를 추가하는 것은 물론이고 애플리케이션에서 한 번 더 중복을 제거함

- 페치 조인과 일반 조인의 차이
    - 페치 조인을 사용하지 않고 조인만 사용하면 어떻게 될까?
    
    ```sql
    select t
    from Team t join t.members m
    where t.name = '팀A'
    ```
    
    ```sql
    SELECT  T.*
    FROM TEAM TINNER JOIN MEMBER M 
    ON T.ID=M.TEAM_ID
    WHERE T.NAME = '팀A*'
    ```
    
    - JPQL은 결과를 반환할 때 연관관계까지 고려하지 않고, 단지 SELECT 절에 지정한 엔티티만 조회함
    - 반면에 페치 조인을 사용하면 연관된 엔티티도 함께 조회
    
    ```sql
    select t
    from Team t join fetch t.members
    where t.name = '팀A'
    ```
    
    ```sql
    SELECT  T.*, M.*
    FROM TEAM TINNER JOIN MEMBER M
     ON T.ID=M.TEAM_ID
    WHERE T.NAME = '팀A*'
    ```
    
- 페치 조인의 특징과 한계
    - 페치 조인을 사용하면 SQL 한 번으로 연관된 엔티티들을 함께 조회할 수 있어서 SQL 호출 횟수를 줄여 성능을 최적화할 수 있음
    - 글로벌 로딩 전략은 지연 로딩을 사용하고 최적화가 필요하면 페치 조인을 적용하는 것이 효과적
    - 페치 조인을 사용하면 연관된 엔티티를 쿼리 시점에 조회하므로 지연 로딩이 발생하지 않음
    - 준영속 상태에서도 객체 그래프를 탐색할 수 있음
    
    - 페치 조인의 한계
        - 페치조인 대상에는 별칭을 줄 수 없다
            - 별칭을 잘못 사용하면 연관된 데이터 수가 달라져서 데이터 무결성이 깨질 수 있으므로 조심해서 사용해야 함
            - 특히 2차 캐시와 함께 사용할 때 조심해야 하는데, 연관된 데이터 수가 달라진 상태에서 2차 캐시에 저장되면 다른 곳에서 조회할 때도 연관된 데이터 수가 달라지는 문제가 발생할 수 있음
        - 둘 이상의 컬렉션을 페치할 수 없다
            - 구현체에 따라 되기도 하는데 컬렉션*컬렉션 카테시안 곱이 만들어 지므로 주의해야 함
                
                → 카테시안 곱 : From절에 2개 이상의 Table이 있을때 두 Table 사이에 **유효 join 조건을 적지 않았을때,** 해당 테이블에 대한 모든 데이터를 전부 결합하여 Table에 존재하는 행 갯수를 곱한 만큼의 결과값이 반환되는 것이다.
                
        - 컬렉션을 페치 조인하면 페이징 API(setFirstResult, setMaxResults)를 사용할 수 없다
            - 하이버네이트에서 컬렉션을 페치 조인하고 페이징 API를 사용하면 경고 로그를 남기면서 메모리에서 페이징 처리를 함
            - 데이터가 적으면 상관없겠지만 데이터가 많으면 성능 이슈와 메모리 초과 예외가 발생할 수 있어서 위험함
            - 페치 조인은 객체 그래프를 유지할 때 사용하면 효과적
            - 반면에 여러 테이블을 조인해서 엔티티가 가진 모양이 아닌 전혀 다른 결과를 내야 한다면 억지로 페치 조인을 사용하기 보다는 여러 테이블에서 필요한 필드들만 조회해서 DTO로 반환하는 것이 더 효과적일 수 있음
        

### 경로 표현식

- .(점)을 찍어 객체 그래프를 탐색하는 것

```sql
select m.username
from Member m
		join m.team t
		join m.orders o
where t.name = '팀A'
```

→ m.username, m.team, m.orders, t.name이 모두 경로 표현식을 사용한 예

- 경로 표현식의 용어 정리
    - 상태 필드 : 단순히 값을 저장하기 위한 필드
    - 연관 필드 : 연관관계를 위한 필드, 임베디드 타입 포함
        - 단일 값 연관 필드 : @ManyToOne, @OneToOne, 대상이 엔티티
        - 컬렉션 값 연관 필드 : @OneToMany, @ManyToMany, 대상이 컬렉션

```java
@Entity
public class Member {
		@Id @GeneratedValue
		private Logn id;
		
		@Column(name="name")
		private String username; // 상태 필드
		private Integer age; // 상태 필드
	
		@ManyToOne(..)
		private Team team; // 연관 필드(단일 값 연관 필드)

		@OneToMany(..)
		private List<Order> orders; // 연관 필드(컬렉션 값 연관 필드)
```

- 상태 필드 : t.username, t.age
- 단일 값 연관 필드 : m.team
- 컬렉션 값 연관 필드 : m.orders

- 경로 표현식과 특징
    - 상태 필드 경로 : 경로 탐색의 끝이다. 더는 탐색할 수 없다
    - 단일 값 연관 경로 : 묵시적으로 내부 조인이 일어난다. 단일 값 연관 경로는 계속 탐색할 수 있다
    - 컬렉션 값 연관 경로 : 묵시적으로 내부 조인이 일어난다. 더는 탐색할 수 없다. 단 FROM 절에서 조인을 통해 별칭을 얻으면 별칭으로 탐색할 수 있다

- 명시적 조인 : JOIN을 직접 적어주는 것
- 묵시적 조인 : 경로 표현식에 의해 묵시적으로 조인이 일어나는 것. 내부 조인만 할 수 있다

- 경로 탐색을 사용한 묵시적 조인 시 주의사항
    - 항상 내부 조인이다
    - 컬렉션은 경로 탐색의 끝이다. 컬렉션에서 경로 탐색을 하려면 명시적으로 조인해서 별칭을 얻어야 한다
    - 경로 탐색은 주로 SELECT, WHERE 절(다른 곳에서도 사용됨)에서 사용하지만 묵시적 조인으로 인해 SQL의 FROM 절에 영향을 준다

→ 단순하고 성능에 이슈가 없으면 크게 문제가 안되지만 성능이 중요하면 분석하기 쉽도록 묵시적 조인 보다는 명시적 조인을 사용!!

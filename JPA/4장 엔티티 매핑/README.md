# 엔티티 매핑

객체와 테이블 매핑: `@Entity`, `@Table`  
기본키 매핑 : `@Id`  
필드와 컬럼 매핑 : `@Column`  
연관관계 매핑 : `@ManyToOne`, `@JoinColumn`

## @Entity

- JPA를 사용해서 테이블과 매핑할 클래스에 붙이는 어노테이션

### 주의사항

- 기본 생성자는 필수다 (파라미터가 없는 public 또는 protected 생성자) → NoArgsConstructor
    - JPA가 엔티티 객체를 생성할 때 기본 생성자를 사용
    - 자바는 원래 기본생성자를 자동으로 만들지만 사용자가 임의로 생성자를 만든다면 기본생성자도 직접만들어야 한다
- final 클래스, enum, interface, inner 클래스에는 사용할 수 없다
- 저장할 필드에 final을 사용하면 안된다

## @Table

- 엔티티와 매핑할 테이블을 지정

## 다양한 매핑 사용

  1. 회원은 일반 회원과 관리자로 구분해야 한다. 
  2. 회원 가입일과 수정일이 있어야 한다. 
  3. 회원을 설명할 수 있는 필드가 있어야한다 → 이 필드는 길이 제한이 없다

</aside>

- 회원 엔티티

```java
@Entity
@Table(name="MEMBER")
public class Member {

    @Id
    @Column(name = "ID")
    private String id;

    @Column(name = "NAME")
    private String username;

    private Integer age;
		
		@Enumerated(EnumType.STRING)
    private RoleType roleType; 

    @Temporal(TemporalType.TIMESTAMP)
    private Date createdDate;

    @Temporal(TemporalType.TIMESTAMP)
    private Date lastModifiedDate;

    @Lob
    private String description;

		[...]
}

public enum RoleType {
    ADMIN, USER
}
```

- roleType : 자바의 Enum을 사용해서 회원의 타입을 구분, @Enumrated 어노테이션으로 매핑
- createDate, lastModifiedDate : 자바의 날짜타입은 @Temporal을 사용해서 매핑
- description : 길이 제한이 없기 때문에 `VARCHAR` 타입 대신에 `CLOB` 타입으로 저장, @Lob을 사용하면 CLOB, BLOB 타입을 매핑할 수 있음

## 데이터베이스 스키마 자동 생성

- JPA는 데이터베이스 스키마를 자동 생성하는 기능을 지원하고, 데이터베이스 방언을 사용해서 데이터베이스 스키마를 생성

- persistence.xml

```xml
<property name="hibernate.hbm2ddl.auto" value="create" />
```

- 위의 속성을 추가하면 **애플리케이션 실행 시점**에 데이터베이스 테이블을 자동으로 생성
- `<property name="hibernate.show_sql" value="true" />` 로 설정하면 콘솔에 실행되는 테이블 생성 DDL을 출력할 수 있다
- 자동 생성되는 DDL은 지정한 데이터베이스 방언에 따라 달라진다

- hibernate.hbm2ddl.auto 속성

> **개발 단계에 따른 추천 전략**
- 개발 초기 단계 : `create` or `update`
- 초기화 상태로 자동화된 테스트를 진행하는 개발자 환경과 CI 서버 : `create` 또는 `create-drop`
- 테스트 서버 : `update` or `validate`
- 스테이징과 운영 서버 : `validate` or `none`
> 

## DDL 생성기능

- DDL에 제약조건 추가
    - 이름은 필수로 입력되고, 10자를 초과하면 안됨

```java
@Column(name = "NAME", nullable = false, length = 10) //추가 
    private String username;
```

- 유니크 제약조건
    - DDL을 자동생성할 때만 사용되고 JPA의 실행 로직에는 영향을 주지 않음
    - 자동 생성 기능을 사용하지 않고 직접 DDL을 만든다면 사용할 이유가 없음
    - 이 기능을 사용하면 애플리케이션 개발자가 엔티티만 보고도 다양한 제약 조건을 파악할 수 있는 장점이 있음

```java
@Entity
@Table(name="MEMBER", uniqueConstraints = {@UniqueConstraint( //추가
        name = "NAME_AGE_UNIQUE",
        columnNames = {"NAME", "AGE"} )})
public class Member {
```

## 기본 키(Primary Key) 매핑

```java
public class Member {

    @Id
    @Column(name = "ID")
    private String id;
```

- 지금까지는 `@Id` 어노테이션만 사용해서 기본키를 애플리케이션에서 직접 할당 했다

> 직접 할당하는 대신에 데이터베이스가 생성해주는 값을 사용하려면 어떻게 매핑해야 할까?
> 

### JPA가 제공하는 데이터베이스 기본키 생성 전략

- 직접할당 : 기본키를 애플리케이션에서 직접 할당한다
- 자동생성 : 대리키 사용방식
    - IDENTITY : 기본키 생성을 데이터베이스에 위임한다
    - SEQUENCE : 데이터베이스 시퀀스를 사용해서 기본키를 할당한다
    - TABLE : 키생성 테이블을 사용한다

→ 오라클 데이터베이스는 시퀀스를 제공하지만 MySQL을 제공하지 않는 대신 기본 키 값을 자동으로 채워주는 AUTO_INCREMENT 기능을 제공

### 기본 키 직접 할당 전략

- 기본 키를 직접 할당하려면 `@Id`로 매핑하면 된다
- `@Id` 적용 가능 자바타입
    - 자바 기본형
    - 자바 래퍼(Wrapper)형
    - String
    - java.util.Date
    - java.sql.Date
    - java.math.BigDecimal
    - java.math.BigInteger
- em.persist()로 엔티티를 저장하기 전에 애플리케이션에서 기본 키를 직접 할당

### IDENTITY 전략

- 기본 키 생성을 데이터베이스에 위임하는 전략
- 주로 MySQL, PostgreSQL, SQL Server, DB2에서 사용
- 데이터베이스에 값을 저장하고 나서야 기본 키 값을 구할 수 있을 때 사용
- 식별자가 생성되는 경우에는 @GeneratedValue 어노테이션을 사용하고 식별자 생성 전략을 선택해야한다
- GenerationType.IDENTITY 로 지정하면 JPA는 기본 키 값을 얻어오기 위해 데이터베이스를 추가로 조회
    
    → 처음에 지정하는 기본키가 없기 때문에 데이터베이스에 저장후 추가 조회가 필요하다
    


💡 **IDENTITY 전략과 최적화**  
`JDBC3`에 추가된 `Statement.getGeneratedKeys()`를 사용하면 데이터를 저장하면서 동시에 생성된 기본 키값도 얻어 올 수 있다. 하이버네이트는 이 메소드를 사용해서 데이터베이스와 한번만 통신한다!

→ 엔티티가 영속 상태가 되려면 식별자가 반드시 필요한데 IDENTITY 식별자 생성 전략은 엔티티를 데이터베이스에 저장해야 식별자를 구할 수 있으므로, 트랜잭션을 지원하는 `쓰기 지연`이 동작하지 않는다

### SEQUENCE 전략

- `데이터베이스 시퀀스`는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트
- 이 시퀀스를 사용해서 기본키를 생성하는 전략
- 시퀀스를 지원하는 오라클, PostgreSQL, DB2, H2 데이터베이스에서 사용

- 시퀀스 DDL

```sql
CREATE TABLE BOARD (
		ID BIGINT NOT NULL PRIMARY KEY,
		DATA VARCHAR(255)
)

// 시퀀스 생성
CREATE SEQUENCE BOARD_SEQ START WITH 1 INCREMENT BY 1;
```

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE,
								generator = "BOARD_SEQ_GENERATOR")
private Long id;

...
```

- 사용할 데이터베이스 시퀀스를 매핑
- JPA는 시퀀스 생성기를 실제 데이터베이스의 BOARD_SEQ 시퀀스와 매핑
- 사용코드는 IDENTITY 전략과 같지만 내부 동작 방식은 다르다
    - SEQUENCE 전략은 em.persist()를 호출할 때 먼저 데이터베이스 시퀀스를 사용해서 식별자를 조회
    - 조회한 식별자를 엔티티에 할당한 후에 엔티티를 영속성 컨텍스트에 저장
    - 트랜잭션을 커밋해서 플러시가 일어나면 엔티티를 데이터베이스에 저장
    

### TABLE 전략

- 키 생성 전용 테이블을 하나 만들고 여기에 이름과 값으로 사용할 컬럼을 만들어 `데이터베이스 시퀀스`를 흉내내는 전략

- TABLE 전략 키 생성 DDL

```java
create table MY_SEQUENCES(
		sequence_name varchar(255) not null,
		next_val bigint,
		primary key (sequence_name)
)
```

- sequence_name 컬럼을 시퀀스 이름으로 사용하고 next_val 컬럼을 시퀀스 값으로 사용


💡 **TABLE 전략과 최적화**  
TABLE 전략은 값을 조회하면서 SELECT 쿼리를 사용하고 다음 값으로 증가시키기 위해 UPDATE 쿼리를 사용한다. 이 전략은 SEQUENCE 전략과 비교해서 데이터베이스와 한 번 더 통신하는 단점이 있다. 이 전략을 최적화하려면 @TableGenerator.allocationSize를 사용하면 된다. 이 값을 사용해서 최적화하는 방법은 SEQUENCE 전략과 같다.


### AUTO 전략

- GenerationType.AUTO는 선택한 데이터베이스 방언에 따라 IDENTITY, SEQUENCE, TABLE 전략 중 하나를 자동으로 선택
- 오라클 → SEQUENCE, MySQL → IDENTITY
- 데이터베이스를 변경해도 코드를 수정할 필요가 없다는 장점
- 키 생성 전략이 확정되지 않은 개발 초기 단계나 프로토타입 개발 시 편리하게 사용할 수 있다

### 기본 키 매핑 정리

- 영속성 컨텍스트는 엔티티를 식별자 값으로 구분하므로 엔티티를 영속 상태로 만들려면 식별자 값이 반드시 있어야 한다

- 식별자 할당 전략
    - 직접할당 : em.persist()를 호출하기 전에 애플리케이션에서 직접 식별자 값을 할당해야 한다. 식별자 값이 없으면 예외 발생
    - SEQUENCE : 데이터베이스 시퀀스에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장
    - TABLE : 데이터베이스 시퀀스 생성용 테이블에서 식별자 값을 획득한 후 영속성 컨텍스트에 저장
    - IDENTITY :  데이터베이스에 엔티티를 저장해서 식별자 값을 획득한 후 영속성 컨텍스트에 저장 (IDENTITY 전략은 테이블에 데이터를 저장해야 식별자 값을 획득할 수 있다)
    

### 📌 **권장하는 식별자 선택 전략**

**데이터베이스 기본키의 3가지 조건**
1. null 값은 허용하지 않는다
2. 유일해야 한다
3. 변해선 안된다

**테이블의 기본키 전략 2가지**
- **자연키(natural key)**
    → 비즈니스에 의미가 있는 키
    → 예: 주민등록번호, 이메일, 전화번호
- **대리키(surrogate key)**
    → 비즈니스와 관련없는 임의로 만들어진 키 = 대체 키
    → 예 : 오라클 시퀀스, auto_increment, 키생성 테이블 사용

**자연 키보다는 대리 키를 권장한다**

**비즈니스 환경은 언젠가 변한다**

**JPA는 모든 엔티티에 일관된 방식으로 대리 키 사용을 권장한다**

## 필드와 컬럼 매핑 : 레퍼런스

- JPA가 제공하는 필드와 컬럼 매핑용 어노테이션들을 레퍼런스 형식으로 정리
    
    → ***간단히 훑어보고, 필요한 매핑을 사용할 일이 있을 때 찾아서 자세히 읽어보기!***
    | 분류 | 매핑 어노테이션 | 설명 |  
    | --- | --- | --- |  
    | 필드와 컬럼 매핑 | @Column | 컬럼을 매핑한다 |. 
    | ‘’ | @Enumrated | 자바의 enum 타입을 매핑한다 |. 
    | ‘’ | @Temporal | 날짜 타입을 매핑한다 |. 
    | ‘’ | @Lob | BLOB, CLOB 타입을 매핑한다 |. 
    | ‘’ | @Transient | 특정 필드를 데이터베이스에 매핑하지 않는다 |. 
    | 기타 | @Access | JPA가 엔티티에 접근하는 방식을 지정한다 |. 

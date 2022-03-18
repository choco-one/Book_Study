# 7장 고급 매핑

- **상속관계 매핑** : 객체의 상속 관계를 데이터베이스에 어떻게 매핑하는지 다룸
- **@MappedSuperclass** : 등록일, 수정일 같이 여러 엔티티에서 공통으로 사용하는 매핑 정보만 상속받고 싶으면 이 기능을 사용하면 됨
- **복합 키와 식별 관계 매핑** : 데이터베이스의 식별자가 하나 이상일 때 매핑하는 방법을 다루고, 데이터베이스 설계에서 이야기하는 식별 관계와 비식별 관계에 대해서도 다룸
- **조인 테이블** : 테이블은 외래 키 하나로 연관관계를 맺을 수 있지만 연관관계를 관리하는 연결 테이블을 두는 방법도 있으며 이 연결 테이블을 매핑하는 방법
- **엔티티 하나에 여러 테이블 매핑하기** : 보통 엔티티 하나에 테이블 하나를 매핑하지만 엔티티하나에 여러 테이블을 매핑하는 방법

## 상속 관계 매핑

- 관계형 데이터베이스에는 객체지향 언어에서 다루는 `상속`이라는 개념이 없음
- 대신 `슈퍼타입-서브타입 관계(Super-Type Sub-Type Relationship)` 라는 모델링 기법이 `상속`의 개념과 가장 유사
- `슈퍼타입-서브타입 관계(Super-Type Sub-Type Relationship)` 을 실제 물리 모델인 테이블로 구현하는 3가지 방법
    - **각각의 테이블로 변환** : 각각을 모두 테이블로 만들고 조회할 때 조인을 사용한다 → JPA에서는 `조인 전략`이라 함
    - **통합 테이블로 변환** : 테이블을 하나만 사용해서 통합한다 → JPA에서는 단일 테이블 전략이라 함
    - **서브타입 테이블로 변환** : 서브 타입 마다 하나의 테이블을 만든다 → JPA에서는 `구현 클래스마다 테이블전략` 이라 한다
    

### 조인 전략

- 엔티티 각각을 모두 테이블로 만들고 자식 테이블이 부모 테이블의 기본키를 받아서 기본키 + 외래키로 사용하는 전략
- 조회할 때 조인을 자주 사용
- 주의할 점 → 객체는 타입으로 구분할 수 있지만 테이블은 타입의 개념이 없기 때문에 `타입을 구분하는 컬럼`을 추가해야 함

- 조인 전략 매핑

```java
// Item 엔티티
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "DTYPE")
public abstract class Item{
	@Id @GeneratedValue
	@Column(name = "ITEM_ID")
	private Long id;
	
	private String name;
	private int price;
	
	...
}

// Album 엔티티
@Entity
@DiscriminatorValue("A")
public class Album extends Item{
	private String artist;
	...
}

// Movie 엔티티
@Entity
@DiscriminatorValue("M")
public class Movie extends Item{
	private String director;
	private String actor;
	...
}
```

- `@Inheritance(strategy=InheranceTyoe.JOINED)` :  상속 매핑은 부모 클래스에 `@Inheritance` 를 사용해야 함
- `@DiscriminatorValue(name = "DTYPE")` : 부모 클래스에 구분 컬럼을 지정하고 이 컬럼으로 저장된 자식 테이블을 구분할 수 있음
- `@DiscriminatorValue("M")` : 엔티티를 저장할 때 구분 컬럼에 입력할 값을 지정
- 자식 테이블의 기본 키 컬럼명을 변경하고 싶으면 `@PrimaryKeyJoinColumn`을 사용

- **장점**
    - 테이블이 정규화 됨
    - 외래 키 참조 무결성 제약조건을 활용할 수 있음
    - 저장공간을 효율적으로 사용함
- 단점
    - 조회할 때 조인이 많이 사용되므로 성능이 저하될 수 있음
    - 조회 쿼리가 복잡
    - 데이터를 등록할 INSERT SQL을 두 번 실행
- 특징
    - JPA 표준 명세는 구분 컬럼을 사용하도록 하지만 하이버네이트를 포함한 몇 구현체는 구분 칼럼없이도 동작함
    - 관련 어노테이션
        - `@PrimaryKeyJoinColumn`, `@DiscriminatorColumn`, `@DiscriminatorValue`
    

### 단일 테이블 전략

- 이름에서 알 수 있듯 테이블을 하나만 사용하고 `구분 컬럼`으로 어떤 자식 데이터가 저장되었는지 구분하며 조회할 때 조인을 사용하지 않으므로 일반적으로 가장 빠르다
- 주의할 점 → 자식 엔티티가 매핑한 컬럼은 모두 `null` 을 허용해야 한다는 점

- 단일 테이블 전략 매핑

```java
// Item 엔티티
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DisciminatorColumn(name = "DTYPE")
...

// 자식 엔티티
@Entity
@DiscriminatorValue("A")
...

@Entity
@DiscriminatorValue("M")
...
```

- 장점
    - 조인이 필요없어 일반적으로 조회 성능이 빠름
    - 조회 쿼리가 단순함
- 단점
    - 자식 엔티티가 매핑한 컬럼은 모두 null을 허용해야 함
    - 모든것을 저장하는게 단일 테이블이니 테이블이 크며 상황에 따라 조회 성능이 오히려 느려질 수 있음
        
        → 컬럼이 많아서 생기는 문제
        
- 특징
    - 구분 컬럼이 필수 `@DiscriminatorColumn`
    - 구분 컬럼 값(`@DiscriminatorValue`)을 지정하지 않으면 엔티티 이름을 그대로 사용함

### 구현 클래스마다 테이블 전략

- 자식 테이블이 부모 테이블의 모든것을 다 가지고 있는 형태이고 자식 엔티티마다 테이블을 다 만들어 줌
- 일반적으로 추천하지 않는 전략

- 구현 클래스마다 테이블 전략 매핑

```java
// Item 엔티티
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
...

// 자식 엔티티
@Entity
@DiscriminatorValue("A")
...

@Entity
@DiscriminatorValue("M")
...
```

- 장점
    - 서브 타입을 구분해서 처리할 때 효과적
    - not null 제약조건을 사용할 수 있다 → 본인것만 있기 때문
- 단점
    - 여러 자식 테이블을 함께 조회할때 성능이 느림 → SQL에 UNION을 사용
    - 자식 테이블을 통합해서 쿼리하기 힘듦
- 특징
    - 구분 컬럼을 사용하지 않음
    

## @MappedSuperclass

- 위 방법들은 모두 부모 자식 클래스 모두 데이터베이스 테이블과 매핑했는데, 부모 클래스는 테이블과 매핑하지 않고 부모 클래스를 자식 클래스에게 매핑 정보만 제공하고 싶을때 `@MappedSuperclass`를 사용한다
- `@MappedSuperclass`는 추상 클래스와 비슷한데 `@Entity`는 실제 테이블과 매핑되지만,  `@MappedSuperclass`는 그렇지 않고 단순히 매핑 정보를 상속할 목적으로만 사용된다

- `@MappedSuperclass` 매핑

```java
// BaseEntity
@MappedSuperclass
public abstract class BaseEntity{
	@Id @GeneratedValue
	private Long id;
	
	private String name;
	...
}

// Member 엔티티
@Entity
public class Member exteds BaseEntity{
	// ID, NAME 상속
	private String email;
	...
}

// Seller 엔티티
@Entity
public class Seller exteds BaseEntity{
	// ID, NAME 상속
	private String shopName;
	...
}
```

- `BaseEntity`에서 객체들의 공통 매핑 정보를 정의하였고 자식 엔티티들이 이를 상속해 매핑 정보를 받았다
- 여기서 `BaseEntity`는 테이블과 매핑할 필요가 없고 자식 엔티티에게 공통 매핑 정보만 제공한다
- `@AttributeOverrides`를 사용해 부모로부터 받은 매핑 정보를 재정의
- `@AssociationOverrides`를 사용해 연관관계를 재정의

- `@MappedSuperclass` **특징**
    - 테이블과 매핑되지 않고 자식 클래스에 엔티티의 매핑 정보를 상속하기 위해 사용
    - `@MappedSuperclass` 로 지정한 클래스는 엔티티가 아니므로 em.find() 나 JPQL에서 사용할 수 없다
    - 이 클래스를 직접 생성해서 사용할 일은 거의 없으므로 추상 클래스로 만드는 것을 권장

## 복합 키와 식별 관계 매핑

### 식별 관계 VS 비식별 관계

- `식별`과 `비식별`은 기본키로 외래키를 사용하는지의 여부에 따라 갈린다
- `식별` 관계는 부모의 기본키를 자식의 기본키 + 외래키로 사용
- `비식별` 관계는 부모의 기본키는 외래키로만 사용
    - 자식 테이블의 기본키는 부모의 기본키와 관련이 없는 독자적인 키를 사용한다
    - 외래키에 NULL을 허용하는지에 따라 필수적 비식별 관계와 선택적 비식별 관계로 나뉨
        - 필수적 비식별 관계(Mandatory) : 외래키에 NULL을 허용하지 않고, 연관관계를 필수적으로 맺어야 한다
        - 선택적 비식별 관계(Optional) : 외래키에 NULL을 허용하고, 연관관계를 맺을지 말지를 선택할 수 있다
- 최근에는 비식별을 주로 사용하고 꼭 필요한 경우만 식별 관계를 사용

### 복합 키: 비식별 관계 매핑

- JPA는 영속성 컨텍스트에 엔티티를 보관할 떄 엔티티의 식별자를 키로 사용한다 → 구분만 할수있으면 식별자
- 이러한 식별자를 구분하기 위해 equals와 hashCode를 사용해 `동등성(완전히 같은지)` 비교를 한다
- 식별자 필드가 하나면(Id가 하나) 보통 자바의 기본 타입을 쓰는데, 식별자 필드가 2개 이상이면 별도의 식별자 클래스를 만들어 equals와 hashCode를 구현한다

- JPA는 복합키를 지원하기 위해 2가지 방법을 제공한다.
    - `@IdClass` : 관계형 데이터베이스에 가까운 방식
        - 부모 테이블은 **복합 기본 키**를 사용하고 자식은 **복합 외래 키**를 사용한다. (관계형 데이터베이스에서 상속 개념은 없음)
        - 복합키를 매핑하기 위해서는 식별자 클래스를 별도로 만들어야 한다
        - 부모 클래스
        
        ```java
        @Entity
        @IdClass(ParentId.Class)
        public class Parent{
        	@Id
        	@Column(name = "PARENT_ID1")
        	private String id1; // ParentId.id1 과 연결
        	
        	@Id
        	@Column(name = "PARENT_ID2")
        	private String id2; // ParentId.id2 과 연결
        	
        	...
        }
        ```
        
        - 식별자 클래스
        
        ```java
        public class ParentId implements Serializable{
        	private String id1; // Parent.id1 과 연결
        	private String id2; // Parent.id2 과 연결
        	
        	public ParentId() {
        	}
        
        	public ParentId(String id1,String id2) {
        			this.id1=id1;
        			this.id2=id2;
        	}
        	
        	@Override
        	public boolean equals(Object o){...}
        	
        	@Override
        	public int hashCode(){...}
        }
        ```
        
        - 식별자 클래스의 속성명과 엔티티에서 사용하는 속성명이 같아야하고 Serializable 인테페이스를 구현해야 하고, 또한 생성자를 만들어줘야하며 식별자 클래스는 public 이어야 한다
        - 부모의 복합키를 저장할때는 단순히 지정(set)하면 된다
        - 조회할때는 `식별자 클래스`로 호출
        - 자식 클래스는 부모의 기본키가 복합키이므로 외래키도 복합키로 저장
        - 자식 클래스
        
        ```java
        @Entity
        public class Child{
        	@Id
        	private String id;
        	
        	@ManyToOne
        	@JoinColumns({
        		@JoinColumn(name = "PARENT_ID1", 
        				referencedColumnName = "PARENT_ID1"),
        		@JoinColumn(name = "PARENT_ID2", 
        				referencedColumnName = "PARENT_ID2")
        	})
        	private Parent parent;
        }
        ```
        
    
    - `@EmbeddedId` : 객체지향에 가까운 방식
        - 부모 클래스
        
        ```java
        @Entity
        public class Parent{
        	@EmbeddedId
        	private ParentId id;
        	...
        }
        ```
        
        - 식별자 클래스
        
        ```java
        @Embeddable
        public class ParentId implements Serializable{
        	@Column(name = "PARENT_ID1")
        	private String id1;
        	@Column(name = "PARENT_ID2")
        	private String id2;
        	
        	// equals and hashCode 구현
        	...
        }
        ```
        
        - `@EmbeddedId`는 바로 식별자 클래스에 기본키를 직접 매핑
        - Parent는 ParentId 부터 아이디를 받아 사용
        
        ```java
        // Parent 저장
        Parent parent = new Parent();
        ParentId parentId = new ParentId("myid1","myid2");
        parent.setId(parentId);
        em.persist(parent);
        
        // Parent 조회
        ParentId parentId = new ParentId("myId1","myId2");
        Parent parent = em.find(Parent.class,parentId);
        ```
        

- **복합키와 equals(), hashcode()**

```java
ParentId id1 = new ParentId();
id1.setId1("myId1");
id1.setId2("myId2");

ParentId id2 = new ParentId();
id2.setId1("myId1");
id2.setId2("myId2");

id1.equals(id2);
```

- equals는 `동등성 비교` 연산자로 값(인스턴스의 동일성 비교)이 같은지 비교
- 개념적으로 id1과 id2은 equlas가 참이어야하지만 거짓이되어(디폴트로 단순히 `동일성 비교`만 한다) 오버라이딩해줘 적절히 바꿔줘야 참이 된다
- 영속성 컨텍스트는 엔티티의 식별자를 키로 사용해 엔티티를 관리하는데 식별자를 비교할때 equals와 hashCode를 사용
- 동등성이 지켜지지 않으면 엉뚱한 엔티티가 조회되거나 조회를 할 수 없게된다
- 복합키는 필수적으로 equals와 hashCode를 구현해줘야하며 식별자 클래스는 `equals()` 와 `hashCode()`를 구현할때 모든 필드를 사용한다.

- **@IdClass vs @EmbeddedId**
    - 본인 취향에 맞는것을 일관성 있게 사용!
    - `@EmbeddedId` 가 `@IdClass` 와 비교해서 더 객체지향적이고 중복도 없어서 좋아보이긴 하지만 특정 상황에 JPQL이 조금 더 길어질 수 있다
        
        → `@EmbeddedId`에서는 식별자 클래스가 복합키를 전담하기 때문
        

### 복합 키: 식별 관계 매핑

- **@IdClass**
    - 식별 관계에서는 자식은 부모의 외래 키를 자신의 기본키로 사용하기 때문에 자식에 손자까지 있다면 손자는 자식과 부모의 외래 키까지 기본키로 사용해야한다

- @IdClass로 식별관계 매핑하기

```java
// 부모 클래스
@Entity
public class Parent{
	@Id @Column(name = "PARENT_ID")
	private String id;
	...
}

// 자식 클래스
@Entity
@IdClass(ChildId.class)
public class Child{
	@Id
	@ManyToOne
	@JoinColumn(name = "PARENT_ID")
	private Parent parent;
	
	@Id @Column(name = "CHILD_ID")
	private String childId;
	...
}

// 자식 ID (식별자 클래스)
public class ChildId implements Serializable{
	private String parent; // Child.parent 매핑
	private String childId; // Child.childId 매핑
	
	// equals, hashCode
	...
}

// 손자 클래스
@Entity
@IdClass(GrandChildId.class)
public class GrandChild{
	@Id
	@ManyToOne
	@JoinColumns({
		@JoinColumn(name = "PARENT_ID"),
		@JoinCOlumn(name = "CHILD_ID")
	})
	private Child child;
	
	@Id @Column(name = "GRANDCHILD_ID")
	private String id;
	
	...
}

// 손자 ID
public class GrandChildId implements Serializable{
	private ChildId child; // GrandChild.child 매핑
	private String id; // GrandChild.id 매핑
	
	// equals, hashCode
	...
}
```

- 식별 관계는 기본 키와 외래 키를 같이 매핑해야 함
- 식별자 매핑할때 @Id와 연관관계 매핑을 위한 @ManyToOne을 같이 사용

- **@Embedded와 식별 관계**

- @Embedded 식별 관계 매핑하기

```java
// 부모
@Entity
public class Parent{
	@Id @Column(name = "PARENT_ID")
	private String id;
	...
}

// 자식
@Entity
public class Child{
	@EmbeddedId
	private ChildId id;
	
	@MapsId("parentId") // ChildId.parentId 매핑
	@ManyToOne
	@JoinColumn(name = "PARENT_ID")
	public Parent parent;
	
	...
}

// 자식 ID
@Embeddable
public class ChildId implements Serializable{
	private String parentId; // @MapsId("parentId")로 매핑
	
	@Column(name = "CHILD_ID")
	private String id;
	
	// equals, hashCode
	...
}

// 손자
@Entity
public class GrandChild{
	@EmbeddedId
	private GrandChildId id;
	
	@MapId("childId") // GrandCHildId.childId 매핑
	@ManyToOne
	@JoinColumns({
		@JoinColumn(name = "PARENT_ID"),
		@JoinColumn(name = "CHILD_ID")
	})
	private Child child;
	
	...
}

// 손자 ID
@Embeddable
public class GrandChildId implements Serializable{
	private ChildId childId; // @MapsId("childId") 매핑
	
	@Column(name = GRANDCHILD_ID)
	private String id;
	
	...
}
```

- @MapsId는 외래키와 매핑한 연관관계를 기본키에도 매핑하겠다는 뜻이며 속성값은 식별자 클래스의 기본키 필드를 지정하면 된다

### 비식별 관계로 구현

- 비식별 관계 매핑하기

```java
// 부모
@Entity
public class Parent{
	@Id @GeneratedValue
	@Column(name = "PARENT_ID")
	private Long id;
	private String name;
	...
}

// 자식
@Entity
public class Child{
	@Id @GeneratedValue
	@Column(name "CHILD_ID")
	private Long id;
	private String name;
	
	@ManyToOne
	@JoinColumn(name = "PARENT_ID")
	private Parent parent;
	...
}

// 손자
@Entity
public class GrandChild{
	@Id @GeneratedValue
	@Column(name = "GRANDCHILD_ID")
	private Long id;
	private String name;
	
	@ManyToOne
	@JoinColumn(name = "CHILD_ID")
	private Child child;
	...
}
```

- 식별 관계의 복합키를 사용한 코드와 비교하면 매핑도 쉽고 코드도 단순함
- 복합키가 없으므로 복합키 클래스를 만들지 않아도 됨

### 일대일 식별 관계

- 자식 테이블의 기본 키 값으로 부모 테이블의 기본 키 값만 사용
- 부모 테이블의 기본 키가 복합 키가 아니면 자식 테이블의 기본 키는 복합 키로 구성하지 않아도 됨

- 일대일 식별 관계 매핑하기

```java
// 부모
@Entity
public class Board{
	@Id @GeneratedValue
	@Column(name = "BOARD_ID")
	private Long id;
	
	private String title;
	
	@OneToOne(mappedBy = "board")
	private BoardDetail boardDetail;
	...
}

// 자식
@Entity
public class BoardDetail{
	@Id
	private Long boardId;
	
	@MapsId // BoardDetail.boardId 매핑
	@OneToOne
	@JoinColumn(name = "BOARD_ID")
	private Board board;
	
	private String content;
	...
}
```

- 자식이 복합키를 사용하지 않은 부모의 `Id` 를 그대로 사용하는 식별 관계이기 때문에 `@MapsId`는 해당 엔티티의 기본키와 매핑
- 저장할때도 자식이 부모만 매핑해주면 된다

### 식별, 비식별 관계의 장단점

- 데이터베이스 설계 관점에서 보면 다음과 같은 이유로 `식별 관계` 보다는 `비식별 관계`를 선호
    
    
    - 식별 관계에서 부모의 기본 키 컬럼은 하나지만 자식은 2개(부모+자식), 손자는 3개(부모,자식,손자)로 점점 기본 키 컬럼 갯수가 늘어나고, 이는 조인할때 더 복잡해지고 기본키 인덱스가 불필요하게 커진다
    - 식별 관계에서는 2개 이상의 컬럼을 합해 복합 기본키를 만들어야하는 경우가 많다
    - 식별 관계에서 기본 키로 비지니스 의미가 있는 자연 키 컬럼을 조합하는 경우가 많은데 이는 시간이 지남에 따라 비지니스 로직이 바뀌면 전체를 변경하기가 힘들어진다
    - 식별 관계에서 자식 테이블은 부모 테이블의 기본키를 자식 자신의 기본키로 사용해야하므로 테이블 구조가 유연하지 못하다

- 객체 관계 매핑 관점에서는 비식별 관계를 선호
    
    
    - 일대일 관계를 제외하고 식별 관계에서는 2개 이상의 컬럼을 사용해 복합 키를 사용하고, JPA에서 복합키를 사용하기 위해서는 복합 키 클래스를 별도로 만들어줘야하는 번거로움이 생긴다
    - 비식별 관계의 기본 키는 주로 대리키를 사용하는데 JPA에서는 `@GeneratedValue`처럼 쉽게 대리키를 생성할 수 있다
    

## 조인 테이블

데이터베이스 테이블의 연관관계 설계 방법은 크게 2가지다

- 조인 컬럼 사용(외래키를 사용하는 방법)
- 조인 테이블 사용(사이에 테이블을 추가해서 사용하는 방법)

- 조인 컬럼 사용
    - 테이블 간에 관계는 주로 `조인 컬럼`이라 부르는 외래 키 컬럼을 사용해서 관리
    - 두 테이블이 아주 가끔 관계를 맺는다면 외래 키 값 대부분이 `null` 로 저장되는 단점

- 조인 테이블 사용
    - `조인 테이블` 이라는 별도의 테이블을 사용해서 연관관계를 관리
    - 테이블을 따로 추가해야 하는 단점
    - 조인 컬럼에서는 조인을 1번만 사용했는데 조인 테이블에서는 2번의 조인을 사용해야 한다

- 기본적으로 조인 컬럼을 사용하고 필요 시 조인 테이블을 사용
- 객체와 테이블 매핑 시 @JoinColumn과 @JoinTalbe로 매핑이 가능
- 조인 테이블은 주로 다대다 관계를 일대다, 다대일로 풀어낼 때 사용된다
    
    → 일대일, 일대다, 다대일 관계에서도 사용하기도 함
    

### 일대일 조인 테이블

- 일대일 조인 테이블 매핑

```java
// 부모
@Entity
public class Parent{
	@Id @GeneratedValue
	@Column(name = "PARENT_ID")
	private Long id;
	private String name;
	
	@OneToOne
	@JoinTable(name = "PARENT_CHILD", // 매핑할 조인 테이블 이름
		joinColumns = @JoinColumn(name = "PARENT_ID"), // 현재 엔티티를 참조하는 외래 키
		inverseJoinColumns = @JoinColumn(name = "CHILD_ID")) //  반대방향 엔티티를 참조하는 외래 키
	private Child child;
	...
}

// 자식
@Entity
public class Child{
	@Id @GeneratedValue
	@Column(name = "CHILD_ID")
	private Long id;
	private String name;
	...
}
```

### 일대다 조인 테이블

- 일대다 단방향 조인 테이블 매핑

```java
// 부모
@Entity
public class Parent{
	@Id @GeneratedValue
	@Column(name = "PARENT_ID")
	private Long id;
	private String name;
	
	@OneToMany
	@JoinTable(name = "PARENT_CHILD",
		joinColumns = @JoinColumn(name = "PARENT_ID"),
		inverseJoinColumns = @JoinColumn(name = "CHILD_ID"))
	private List<Child> child = new ArrayList<Child>();
}

// 자식
@Entity
public class Child{
	@Id @GeneratedValue
	@Column(name = "CHILD_ID")
	private Long id;
	private String name;
	...
}
```

### 다대일 조인 테이블

- `일대다`를 반대로 해주면 된다
- 다대일 양방향 조인 테이블 매핑

```java
// 부모
@Entity
public class Parent{
	@Id @GeneratedValue
	@Column(name = "PARENT_ID")
	private Long id;
	private String name;
	
	@OneToMany(mappedBy = "parent")
	private List<Child> child = new ArrayList<Child>();
	...
}

// 자식
@Entity
public class Child{
	@Id @GeneratedValue
	@Column(name = "CHILD_ID")
	private Long id;
	private String name;
	
	@ManyToOne(optional = false)
	@JoinTable(name = "PARENT_CHILD",
		joinColumns = @JoinColumn(name = "CHILD_ID"),
		inverseJoinColumns = @JoinColumn(name = "PARENT_ID"))
	private Parent parent;
	...
}
```

### 다대다 조인 테이블

- 두 테이블의 기본키를 이용한 하나의 복한 `유니크 제약조건(기본 키 설정)`을 걸어야한다
- 다대다 조인 테이블 매핑

```java
// 부모
@Entity
public class Parent{
	@Id @GeneratedValue
	@Column(name = "PARENT_ID")
	private Long id;
	private String name;
	
	@ManyToMany
	@JoinTable(name = "PARENT_CHILD",
		joinColumns = @JoinColumn(name = "PARENT_ID"),
		inverseJoinColumns = @JoinColumn(name = "CHILD_ID"))
	private List<Child> child = new ArrayList<Child>();
	...
}

// 자식
@Entity
public class Child{
	@Id @GeneratedValue
	@Column(name = "CHILD_ID")
	private Long id;
	private String name;
	...
}
```

> 조인 테이블에 컬럼을 추가하면 `@JoinTable`  전략을 사용할 수 없다. 대신에 새로운 엔티티를 만들어서 조인 테이블과 매핑해야 한다.
> 

## 엔티티 하나에 여러 테이블 매핑

- 잘 사용하지는 않지만 `@SecondaryTable`을 사용하면 한 엔티티에 여러 테이블을 매핑할 수 있다
- 하나의 엔티티에 여러 테이블 매핑

```java
@Entity
@Table(name = "BOARD") // BOARD 테이블과 매핑
@SecondaryTable(name = "BOARD_DETAIL", // 매핑할 다른 테이블의 이름
	pkJoinColumns = @PrimaryKeyJoinColumn(name = "BOARD_DETAIL_ID")) // 다른 테이블의 기본 키명
public class Board{
	@Id @GeneratedValue
	@Column(name = "BOARD_ID")
	private Long id;
	
	private String title;
	
	@Column(table = "BOARD_DETAIL")
	private String content;
}
```

- 하지만 이 방식보다는 각 테이블마다 엔티티를 만들어서 사용한것이 좋다
    
    → 조회할때 두 테이블을 항상 조회하기 때문에 최적화하기 어렵다

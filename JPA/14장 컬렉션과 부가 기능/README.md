- 컬렉션 : 다양한 컬렉션과 특징  
- 컨버터 : 엔터티의 데이터를 변환해서 데이터베이스에 저장  
- 리스너 : 엔터티에서 발생한 이벤트를 처리  
- 엔터티 그래프 : 엔터티를 조회할 때 연관된 엔터티들을 선택해서 함께 조회  

## 컬렉션

JPA는 자바에서 기본으로 제공하는 Collection, List, Set, Map 컬렉션을 지원한다.  
- @OneToMany, @ManyToMany를 사용해서 일대다나 다대다 엔터티 관계를 매핑할 때  
- @ElementCollection을 사용해서 값 타입을 하나 이상 보관할 때  
위와 같은 상황에서 컬렉션을 사용할 수 있다.  

- Collection : 자바가 제공하는 최상위 컬렉션 / 하이버네이트는 중복을 허용하고 순서를 보장하지 않는다고 가정  
- Set : 중복을 허용하지 않는 컬렉션 / 순서를 보장 x  
- List : 순서가 있는 컬렉션 / 순서를 보장하고 중복 허용  
- Map : Key, Value 구조로 되어 있는 특수한 컬렉션  

### 1. JPA와 컬렉션  
하이버네이트는 엔터티를 영속 상태로 만들 때, 컬렉션 필드를 하이버네이트에서 준비한 컬렉션으로 감싸서 사용한다.  
```java
@Entity
public class Team {
  @Id
  private String id;
  
  @OneToMany
  @JoinColumn
  private Collection<Member> members = new ArrayList<Member>();
  ...
}
```
Team은 members 컬렉션을 필드로 가지고 있고, 다음과 같은 코드로 Team을 영속 상태로 만들 수 있다.  
```java
Team team = new Team();

System.out.println("before persist = " + team.getMembers().getClass());
em.persist(team);
System.out.println("after persist = " + team.getMembers().getClass());
```
=> 출력 결과  
```java
before persist = class java.util.ArrayList
after persist = class org.hibernate.collection.internal.PersistentBag
```
ArrayList 타입이었던 컬렉션이 엔터티를 영속 상태로 만든 직후에 하이버네이트가 제공하는 PersistentBag 타입으로 변경된 것을 확인할 수 있다.  
하이버네이트는 컬렉션을 효율적으로 관리하기 위해 엔터티를 영속 상태로 만들 때 원본 컬렉션을 감싸고 있는 내장 컬렉션을 생성해서 이 내장 컬렉션을 사용하도록 참조를 변경한다.  
그래서 다음과 같이 컬렉션을 사용할 때 즉시 초기화해서 사용하는 것을 권장  
```java
Collection<Member> members = new ArrayList<Member>();
```

|컬렉션 인터페이스|내장 컬렉션|종복 허용|순서 보관|
|---|---|---|---|
|Collection, List|PersistentBag|O|X|
|Set|PersistentSet|X|X|
|List + @OrderColumn|PersistentList|O|O|

### 2. Collection, List  
```java
@Entity
public class Parent {
  @Id @GeneratedValue
  private Long id;
  
  @OneToMany
  @JoinColumn
  private Collection<CollectionChild> collection = new ArrayList<CollectionChild>();
  
  @OneToMany
  @JoinColumn
  private List<ListChild> list = new ArrayList<ListChild>();
  ...
}
```
Collection, List는 종복을 허용하기 때문에 add() 메소드는 내부에서 어떤 비교도 하지 않고 항상 true를 반환한다.  
같은 엔터티가 있는지 찾거나 삭제할 때는 equals() 메소드를 사용한다.  
엔터티를 추가할 때 중복된 엔터티가 있는지 비교하지 않고 저장만 하면 되기 때문에 엔터티를 추가해도 지연 로딩된 컬렉션을 초기화하지 않는다.  

### 3. Set  
```java
@Entity
public class Parent {
  @OneToMany
  @JoinColumn
  private Set<SetChild> set = new HashSet<SetChild>();
  ...
}
```
HashSet은 종복을 허용하지 않기 때문에 add() 메소드는 equals() 메소드로 같은 객체가 있는지 비교하고, 없으면 추가하고 true를 반환 / 있으면 false를 반환한다.    
Hash 알고리즘을 사용하므로 hashcode()도 함께 사용해서 비교한다.  
엔터티를 추가할 때 중복된 엔터티가 있는지 비교해야 하기 때문에 엔터티를 추가할 때 지연 로딩된 컬렉션을 초기화한다.  

### 4. List + @OrderColumn  
List 인터페이스에 @OrderColumn을 추가하면 순서가 있는 특수한 컬렉션으로 인식한다.  
데이터베이스에 순서 값을 저장해서 조회할 때 사용한다.  
```java
@Entity
public class Board {
  @Id @GeneratedValue
  private Long id;
  
  private String title;
  private String content;
  
  @OneToMany(mappedBy = "board")
  @OrderColumn(name = "POSITION")
  private List<Comment> comments = new ArrayList<Comment>();
  ...
}

@Entity
public class Comment {
  @Id @GeneratedValue
  private Long id;
  
  private String comment;
  
  @ManyToOne
  @JoinColumn(name = "BOARD_ID")
  private Board board;
  ...
}
```
순서가 있는 컬렉션은 데이터베이스에 순서 값도 함께 관리한다.  
@OrderColumn의 name 속성에 POSITION이라는 값을 주어서 JPA는 List의 위치 값을 테이블의 POSITION 컬럼에 보관하는데  
Board.commnts 컬렉션은 Board 엔터티에 있지만 테이블의 일대다 관계의 특성상 위치 값은 다(N) 쪽에 저장해야 하므로 실제 POSITON 컬럼은 COMMENT 테이블에 매핑된다.  

@OrderColumn은 실무에서 사용하기에는 단점이 많으므로 개발자가 직접 POSITON 값을 관리하거나 @OrderBy 사용을 권장한다.  

#### @OrderColumn의 단점  
- @OrderColumn을 Board 엔터티에서 매핑하므로 Comment는 POSITON의 값을 알 수 없다. 따라서 Comment를 INSERT할 때, Board.comments의 값을 사용해서 POSITON의 값을 UPDATE 하는 SQL이 추가로 발생한다.  
- List를 변경하면 연관된 많은 위치 값을 변경해야 한다.  
- 중간에 POSITION 값이 없으면 조회한 List에는 null이 보관된다. 따라서 컬렉션을 순회할 때 NullPointerException이 발생한다.  

### 5. @OrderBy  
@OrderBy는 데이터베이스의 ORDER BY절을 사용해서 컬렉션을 정렬하므로 순서용 컬럼을 매핑하지 않아도 된다.  
또한 @OrderBy는 모든 컬렉션에 사용할 수 있다.  
```java
@Entity
public class Team {
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @OneToMany(mappedBy = "team")
  @OrderBy("username desc, id asc")
  private Set<Member> members = new HashSet<Member>();
  ...
}

@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  
  @Column(name = "MEMBER_NAME")
  private String username;
  
  @ManyToOne
  private Team team;
  ...
}
```
@OrderBy의 값은 JPQL의 order by절처럼 엔터티의 필드를 대상으로 한다.  
하이버네이트는 Set에 @OrderBy를 적용해서 결과를 조회하면 순서를 유지하기 위해 HashSet 대신에 LinkedHashSet을 내부에서 사용한다.  

---

## @Converter  
컨버터를 사용하면 엔터티의 데이터를 변환해서 데이터베이스에 저장할 수 있다.  
컨버터 클래스는 @Converter 어노테이션을 사용하고 AttributeConverter 인터페이스를 구현해야 한다.  
또한, 제너릭에 현재 타입과 변환할 타입을 지정해야 한다.  
```java
public interface AttributeConverter<X,Y> {
  public X convertToDatabaseColumn (X attribute);
  public Y convertToEntityAttribute (Y dbData);
}
```
- convertToDatabaseColumn() : 엔터티의 데이터를 데이터베이스 컬럼에 저장할 데이터로 변환  
- convertToEntityAttribute() : 데이터베이스에서 조회한 컬럼 데이터를 엔터티의 데이터로 변환  

또한, 컨버터는 클래스 레벨에도 설정 가능한데 이 때는 attributeName 속성을 사용해서 어떤 필드에 컨버터를 적용할지 명시해야 한다.  

### 1. 글로벌 설정  
모든 Boolean 타입에 컨버터를 적용하려면 @Converter(autoApply = true) 옵션을 적용하면 된다.  
이렇게 설정하면 @Convert를 지정하지 않아도 모든 Boolean 타입에 대해 자동으로 컨버터가 적용된다.  

|속성|기능|기본값|
|---|---|---|
|converter|사용할 컨버터를 지정||
|attributeName|컨버터를 적용할 필드를 지정||
|disableConversion|글로벌 컨버터나 상속받은 컨버터를 사용 X|false|

---

## 리스너  


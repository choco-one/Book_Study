- **컬렉션** : 다양한 컬렉션과 특징  
- **컨버터** : 엔터티의 데이터를 변환해서 데이터베이스에 저장  
- **리스너** : 엔터티에서 발생한 이벤트를 처리  
- **엔터티 그래프** : 엔터티를 조회할 때 연관된 엔터티들을 선택해서 함께 조회  

## 컬렉션

JPA는 자바에서 기본으로 제공하는 Collection, List, Set, Map 컬렉션을 지원한다.  
- `@OneToMany`, `@ManyToMany`를 사용해서 일대다나 다대다 엔터티 관계를 매핑할 때  
- `@ElementCollection`을 사용해서 값 타입을 하나 이상 보관할 때  
위와 같은 상황에서 컬렉션을 사용할 수 있다.  

- **Collection** : 자바가 제공하는 최상위 컬렉션 / 하이버네이트는 중복을 허용하고 순서를 보장하지 않는다고 가정  
- **Set** : 중복을 허용하지 않는 컬렉션 / 순서를 보장 x  
- **List** : 순서가 있는 컬렉션 / 순서를 보장하고 중복 허용  
- **Map** : Key, Value 구조로 되어 있는 특수한 컬렉션  

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
Collection, List는 종복을 허용하기 때문에 `add()` 메소드는 내부에서 어떤 비교도 하지 않고 항상 true를 반환한다.  
같은 엔터티가 있는지 찾거나 삭제할 때는 `equals()` 메소드를 사용한다.  
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
HashSet은 종복을 허용하지 않기 때문에 `add()` 메소드는 `equals()` 메소드로 같은 객체가 있는지 비교하고, 없으면 추가하고 true를 반환 / 있으면 false를 반환한다.    
Hash 알고리즘을 사용하므로 `hashcode()`도 함께 사용해서 비교한다.  
엔터티를 추가할 때 중복된 엔터티가 있는지 비교해야 하기 때문에 엔터티를 추가할 때 지연 로딩된 컬렉션을 초기화한다.  

### 4. List + @OrderColumn  
List 인터페이스에 `@OrderColumn`을 추가하면 순서가 있는 특수한 컬렉션으로 인식한다.  
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
`@OrderColumn`의 name 속성에 POSITION이라는 값을 주어서 JPA는 List의 위치 값을 테이블의 POSITION 컬럼에 보관하는데  
Board.commnts 컬렉션은 Board 엔터티에 있지만 테이블의 일대다 관계의 특성상 위치 값은 다(N) 쪽에 저장해야 하므로 실제 POSITON 컬럼은 COMMENT 테이블에 매핑된다.  

@OrderColumn은 실무에서 사용하기에는 단점이 많으므로 개발자가 직접 POSITON 값을 관리하거나 @OrderBy 사용을 권장한다.  

#### @OrderColumn의 단점  
- `@OrderColumn`을 Board 엔터티에서 매핑하므로 Comment는 POSITON의 값을 알 수 없다. 따라서 Comment를 INSERT할 때, Board.comments의 값을 사용해서 POSITON의 값을 UPDATE 하는 SQL이 추가로 발생한다.  
- List를 변경하면 연관된 많은 위치 값을 변경해야 한다.  
- 중간에 POSITION 값이 없으면 조회한 List에는 null이 보관된다. 따라서 컬렉션을 순회할 때 **NullPointerException**이 발생한다.  

### 5. @OrderBy  
`@OrderBy`는 데이터베이스의 ORDER BY절을 사용해서 컬렉션을 정렬하므로 순서용 컬럼을 매핑하지 않아도 된다.  
또한 `@OrderBy`는 모든 컬렉션에 사용할 수 있다.  
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
`@OrderBy`의 값은 JPQL의 order by절처럼 엔터티의 필드를 대상으로 한다.  
하이버네이트는 Set에 `@OrderBy`를 적용해서 결과를 조회하면 순서를 유지하기 위해 HashSet 대신에 LinkedHashSet을 내부에서 사용한다.  

---

## @Converter  
컨버터를 사용하면 엔터티의 데이터를 변환해서 데이터베이스에 저장할 수 있다.  
컨버터 클래스는 `@Converter` 어노테이션을 사용하고 **AttributeConverter** 인터페이스를 구현해야 한다.  
또한, 제너릭에 현재 타입과 변환할 타입을 지정해야 한다.  
```java
public interface AttributeConverter<X,Y> {
  public X convertToDatabaseColumn (X attribute);
  public Y convertToEntityAttribute (Y dbData);
}
```
- `convertToDatabaseColumn()` : 엔터티의 데이터를 데이터베이스 컬럼에 저장할 데이터로 변환  
- `convertToEntityAttribute()` : 데이터베이스에서 조회한 컬럼 데이터를 엔터티의 데이터로 변환  

또한, 컨버터는 클래스 레벨에도 설정 가능한데 이 때는 attributeName 속성을 사용해서 어떤 필드에 컨버터를 적용할지 명시해야 한다.  

### 1. 글로벌 설정  
모든 Boolean 타입에 컨버터를 적용하려면 `@Converter(autoApply = true)` 옵션을 적용하면 된다.  
이렇게 설정하면 `@Convert`를 지정하지 않아도 모든 Boolean 타입에 대해 자동으로 컨버터가 적용된다.  

|속성|기능|기본값|
|---|---|---|
|converter|사용할 컨버터를 지정||
|attributeName|컨버터를 적용할 필드를 지정||
|disableConversion|글로벌 컨버터나 상속받은 컨버터를 사용 X|false|

---

## 리스너  
### 1. 이벤트 종류  
1. **PostLoad** : 엔터티가 영속성 컨텍스트에 조회된 직후 또는 refresh를 호출한 후  
2. **PrePersist** : `persist()` 메소드를 호출해서 엔터티를 영속성 컨텍스트에 관리하기 직전에 호출 / 식별자 생성 전략을 사용한 경우 엔터티에 식별자는 아직 존재하지 않는다. / 새로운 인스턴스를 merge할 때도 수행  
3. **PreUpdate** : flush나 commit을 호출해서 엔터티를 데이터베이스에 수정하기 직전에 호출  
4. **PreRemove** : `remove()` 메소드를 호출해서 엔터티를 영속성 컨텍스트에서 삭제하기 직전에 호출 / 삭제 명령어로 영속성 전이가 일어날 때도 호출 / orphanRemoval에 대해서는 flush나 commit 시에 호출  
5. **PostPersist** : flush나 commit을 호출해서 엔터티를 데이터베이스에 저장한 직후에 호출 / 식별자 항상 존재 / 식별자 생성 전략이 IDENTITY면 식별자를 생성하기 위해 persist()를 호출하면서 데이터베이스에 해당 엔터티를 저장하므로 호출 직후 바로 PostPersist가 호출된다.  
6. **PostUpdate** : flush나 commit을 호출해서 엔터티를 데이터베이스에 수정한 직후에 호출  
7. **PostRemove** : flush나 commit을 호출해서 엔터티를 데이터베이스에 삭제한 직후에 호출  

### 2. 이벤트 적용 위치  

- **엔터티에 직접 적용**  
  엔터티에 이벤트가 발생할 때마다 어노테이션으로 지정한 메소드가 실행된다.  
  
- **별도의 리스너 등록**  
  리스너는 대상 엔터티를 파라미터로 받을 수 있다.  
  반환 타입은 void로 설정해야 한다.  

- **기본 리스너 사용**  
  모든 엔터티의 이벤트를 처리하려면 `META-INF/orm.xml`에 기본 리스너로 등록하면 된다.  
  여러 리스너를 등록했을 때 이벤트 호출 순서는  
  1. 기본 리스너  
  2. 부모 클래스 리스너  
  3. 리스너  
  4. 엔터티  

**더 세밀한 설정**  
더 세밀한 설정을 위한 어노테이션  
- `javax.persistence.ExcludeDefaultListeners` : 기본 리스너 무시  
- `javax.persistence.ExcludeSuperclassListeners` : 상위 클래스 이벤트 리스너 무시  

---

## 엔터티 그래프  
엔터티를 조회할 때 연관된 엔터티들을 함께 조회하려면 다음처럼 글로벌 fetch 옵션을 `FetchType.EAGER`로 설정한다.  
```java
@Entity
class Order {
  @ManyToOne(fetch=FetchType.EAGER)
  Member member;
  ...
}
```
또는 다음처럼 JPQL에서 페치 조인을 사용하면 된다.  
```sql
select o from Order o join fetch o.member
```
글로벌 fetch 옵션은 어플리케이션 전체에 영향을 주고 변경할 수 없는 단점이 있어서 일반적으로 글로벌 fetch 옵션은 `FetchType.LAZY`를 사용하고 엔터티를 조회할 때 연관된 엔터티를 함께 조회할 필요가 있으면 JPQL의 페치 조인을 사용한다.  

페치 조인을 사용하면 같은 JPQL을 중복해서 작성하는 경우가 많다.  
예를 들어 모두 같은 주문을 조회하는 JPQL이지만 함께 조회할 엔터티에 따라서 다른 JPQL을 사용해야 한다. JPQL이 데이터를 조회하는 기능뿐만 아니라 연관된 엔터티를 함께 조회하는 기능도 제공하기 때문에 JPQL이 두 가지 역할을 모두 수행해서 발생하는 문제다.  
JPA 2.1에 추가된 엔터티 그래프 기능을 사용하면 엔터티를 조회하는 시점에 함께 조회할 연관된 엔터티를 선택할 수 있다.  
=> JPQL은 데이터를 조회하는 기능만 수행하면 되고 연관된 엔터티를 함께 조회하는 기능은 엔터티 그래프를 사용하면 된다.  
엔터티 그래프 기능은 엔터티 조회시점에 연관된 엔터티들을 함께 조회하는 기능이다.  

### 1. Named 엔터티 그래프  
Named 엔터티 그래프는 @NamedEntityGraph로 정의한다.  
- **name** : 엔터티 그래프의 이름을 정의  
- **attributeNodes** : 함께 조회할 속성 선택 / `@NamedAttributeNode`를 사용하고 그 값으로 함께 조회할 속성을 선택하면 된다.  
둘 이상을 정의하려면 `@NamedEntityGraphs`를 사용하면 된다.  

### 2. em.find()에서 엔터티 그래프 사용  
```java
EntityGraph graph = em.getEntityGraph("Order.withMember");

Map hints = new HashMap();
hints.put("javax.persistence.fetchgraph", graph);

Order order = em.find(Order.class, orderId, hints);
```
Named 엔터티 그래프를 사용하려면 정의한 엔터티 그래프를 `em.getEntityGraph("Order.withMember")`를 통해서 찾아오면 된다.  
엔터티 그래프는 JPA의 힌트 기능을 사용해서 동작하는데 힌트의 키로 `javax.persistence.fetchgraph`를 사용하고 힌트의 값으로 찾아온 엔터티 그래프를 사용하면 된다.  

### 3. subgraph  
```java
@NamedEntityGraph(name = "Order.withAll", attributeNodes = {
  @NamedAttributeNode("member"),
  @NamedAttributeNode(value = "orderItems", subgraph = "orderItems")
  },
  subgraphs = @NamedSubgraph(name = "orderItems", attributeNodes = {
    @NamedAttributeNode("item")
  })
)
@Entity
@Table(name = "ORDERS")
public class Order {
  @Id @GeneratedValue
  @Column(name = "ORDER_ID")
  private Long id;
  
  @ManyToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(name = "MEMBER_ID")
  private Member member;
  
  @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
  private List<OrderItem> orderItems = new ArrayList<OrderItem>();
  ...
}

@Entity
@Table(name = "ORDER_ITEM")
public class OrderItem {
  @Id @GeneratedValue
  @Column(name = "ORDER_ITEM_ID")
  private Long id;
  
  @ManyToOne(fetch = FetchType.LAZY)
  @JoinColumn(name = "ITEM_ID")
  private Item item;
  ...
}
```
이 엔터티 그래프는 Order -> Member, Order -> OrderItem, OrderItem -> Item 의 객체 그래프를 함께 조회한다.  
이때 OrderItem -> Item은 Order의 객체 그래프가 아니므로 subgraphs 속성으로 정의해야 한다.  
`@NamedSubgraph`를 사용해서 subgraphs 속성을 정의할 수 있다.  

### 4. JPQL에서 엔터티 그래프 사용  
`em.find()`와 동일하게 힌트만 추가하면 된다.  
```java
List<Order> resultList = 
  em.createQuery("select o from Order o where o.id = :orderId", Order.class)
    .setParameter("orderId", orderId)
    .setHint("javax.persistence.fetchgraph", em.getEntityGraph("Order.withAll"))
    .getResultList();
```

### 5. 동적 엔터티 그래프  
`createEntityGraph()` 메소드를 사용하면 된다.  
```java
public <T> EntityGraph<T> createEntityGraph(Class<T> rootType);  
```

### 6. 엔터티 그래프 정리  
- **ROOT에서 시작**  
  엔터티 그래프는 항상 조회하는 엔터티의 ROOT에서 시작해야 한다.  
  Order 엔터티를 조회하는데 Member부터 시작하는 엔터티 그래프를 사용하면 안된다.  

- **이미 로딩된 엔터티**  
  영속성 컨텍스트에 해당 엔터티가 이미 로딩되어 있으면 엔터티 그래프가 적용되지 않는다.  
  아직 초기화되지 않은 프록시에는 엔터티 그래프가 적용된다.  
  ```java
  Order order1 = em.find(Order.class, orderId); //이미 조회
  hints.put("javax.persistence.fetchgraph", em.getEntityGraph("Order.withMember"));
  Order order2 = em.find(Order.class, orderId, hints);
  ```
  이 경우 조회된 order2에는 엔터티 그래프가 적용되지 않고 처음 조회한 order1과 같은 인스턴스가 반환된다.  

- **fetchgraph, loadgraph의 차이**  
  `javax.persistence.fetchgraph` 힌트를 사용해서 엔터티 그래프를 조회하면 엔터티 그래프에 선택한 속성만 함께 조회한다.  
  `javax.persistence.loadgraph` 속성은 엔터티 그래프에 선택한 속성뿐만 아니라 글로벌 fetch 모드가 `FetchType.EAGER`로 설정된 연관관계도 포함해서 함께 조회한다.  
  


## 다대일

- 일대다 관계의 반대 방향
- 데이터베이스 테이블의 일(1), 다(N) 관계에서 외래 키는 항상 다(N)쪽
- 객체 양방향 관계에서 연관관계의 주인은 항상 다(N)쪽

### 1. 다대일 단방향 [N:1]

- 회원 엔터티
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@ManyToOne
    	@JoinColumn(name = "TEAM_ID")
    	private Team team;
    	
    	//Getter, Setter ...
    	...
    }
    ```
    
- 팀 엔터티
    
    ```java
    @Entity
    public class Team {
    	@Id @GeneratedValue
    	@Column(name = "TEAM_ID")
    	private Long id;
    	
    	private String name;
    	
    	//Getter, Setter ...
    	...
    }
    ```
    

회원은 Member.team으로 팀 엔터티를 참조할 수 있지만, 팀에는 회원을 참조하는 필드가 X

⇒ 다대일 단방향 연관관계

`@JoinColumn(name = "TEAM_ID")` 를 사용해서 Member.team 필드를 TEAM_ID 외래 키와 매핑

⇒ Member.team 필드로 회원 테이블의 TEAM_ID 외래키를 관리

### 2. 다대일 양방향 [N:1, 1:N]

- 회원 엔터티
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@ManyToOne
    	@JoinColumn(name = "TEAM_ID")
    	private Team team;
    	
    	public void setTeam(Team team) {
    		this.team = team;
    		
    		//무한루프에 빠지지 않도록 체크
    		if(!team.getMembers().contains(this)) {
    			team.getMembers().add(this);
    		}
    	}
    }
    ```
    
- 팀 엔터티
    
    ```java
    @Entity
    public class Team {
    	@Id @GeneratedValue
    	@Column(name = "TEAM_ID")
    	private Long id;
    	
    	private String name;
    	
    	@OneToMany(mappedBy = "team")
    	private List<Member> members = new ArrayList<Member>();
    	
    	public void addMember(Member member) {
    		this.members.add(member);
    		if (member.getTeam() != this) {  //무한루프에 빠지지 않도록 체크
    			member.setTeam(this);
    		}
    	}
    }
    ```
    
- **양방향은 외래 키가 있는 쪽이 연관관계의 주인이다**
    - 일대다와 다대일 연관관계는 항상 다(N)에 외래 키가 있음
    - 여기서는 다쪽인 MEMBER 테이블이 외래 키를 가지므로 Member.team이 연관관계의 주인
    - JPA는 외래 키를 관리할 때 연관관계의 주인만 사용하므로 주인이 아닌 Team.members는 조회를 위한 JPQL이나 객체 그래프를 탐색할 때 사용
- **양방향 연관관계는 항상 서로를 참조해야 한다.**
    - 어느 한 쪽만 참조하면 양방향 연관관계가 성립하지 않음
    - 항상 서로 참조하게 하려면 회원의 setTeam(), 팀의 addMember() 와 같은 연관관계 편의 메소드를 작성하는 것이 좋음
    - 편의 메소드는 한 곳에만 작성하거나 양쪽 다 작성할 수 있는데 양쪽에 다 작성하면 무한루프에 빠지므로 주의해야 함

---

## 일대다

- 다대일 관계의 반대 방향
- 엔터티를 하나 이상 참조할 수 있으므로 자바 컬렉션인 Collection, List, Set, Map 중에 하나를 사용해야 한다.

### 1. 일대다 단방향 [1:N]

보통 자신이 매핑한 테이블의 외래 키를 관리하는데, 팀 엔터티의 Team.members로 회원 테이블의 TEAM_ID 외래 키를 관리하는 특이한 모습이 나타난다.

→ 일대다 관계에서 외래 키는 항상 다쪽 테이블에 있는데 다쪽인 Member 엔터티에는 외래 키를 매핑할 수 있는 참조 필드가 없어서

- 팀 엔터티
    
    ```java
    @Entity
    public class Team {
    	@Id @GeneratedValue
    	@Column(name = "TEAM_ID")
    	private Long id;
    	
    	private String name;
    	
    	@OneToMany
    	@JoinColumn(name = "TEAM_ID")  //MEMBER 테이블의 TEAM_ID (FK)
    	private List<Member> members = new ArrayList<Member>();
    	
    	//Getter, Setter ...
    }
    ```
    
- 회원 엔터티
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	//Getter, Setter ...
    }
    ```
    

일대다 단방향 관계를 매핑할 때는 @JoinColumn을 명시해야 한다. 그렇지 않으면 JPA는 연결 테이블을 중간에 두고 연관관계를 관리하는 JoinTable 전략을 기본으로 사용해 매핑한다.

- **일대다 단방향 매핑의 단점**
    
    매핑한 객체가 관리하는 외래 키가 다른 테이블에 있다는 점이다.
    
    본인 테이블에 외래 키가 있으면 엔터티의 저장과 연관관계 처리를 INSERT SQL 한 번으로 끝낼 수 있지만, 다른 테이블에 외래 키가 있으면 연관관계 처리를 위한 UPDATE SQL을 추가로 실행해야 한다.
    
- **일대다 단방향 매핑보다는 다대일 양방향 매핑을 사용하자**
    
    다대일 양방향 매핑은 관리해야 하는 외래 키가 본인 테이블에 있으므로 일대다 단방향 매핑 같은 문제가 발생하지 않는다. 두 매핑의 테이블 모양은 완전히 같으므로 엔터티만 약간 수정하면 된다.
    

### 2. 일대다 양방향 [1:N, N:1]

일대다 양방향 매핑은 존재하지 않는다. 대신 다대일 양방향 매핑을 사용해야 한다.

관계형 데이터베이스의 특성상 일대다, 다대일 관계는 항상 다 쪽에 외래키가 있기 때문에 양방향 매핑에서 @OneToMany는 연관관계의 주인이 될 수 없다.

일대다 단방향 매핑 반대편에 같은 외래 키를 사용하는 다대일 단방향 매핑을 읽기 전용으로 하나 추가하는 방식으로 매핑할 수는 있다.

- 팀 엔터티
    
    ```java
    @Entity
    public class Team {
    	@Id @GeneratedValue
    	@Column(name = "TEAM_ID")
    	private Long id;
    	
    	private String name;
    	
    	@OneToMany
    	@JoinColumn(name = "TEAM_ID")
    	private List<Member> members = new ArrayList<Member>();
    	
    	//Getter, Setter ...
    }
    ```
    
- 회원 엔터티
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@ManyToOne
    	@JoinColumn(name = "TEAM_ID",insertable = false, updatable = false)
    	private Team team;
    
    	//Getter, Setter ...
    }
    ```
    

일대다 단방향 매핑 반대편에 다대일 단방향 매핑을 추가했다.

일대다 단방향 매핑과 같은 TEAM_ID 외래 키 컬럼을 매핑했다. 이러면 둘 다 같은 키를 관리하므로 문제가 발생할 수 있다. 따라서 반대편인 다대일 쪽은 `insertable = false, updatable = false`로 설정해 읽기만 가능하게 했다.

일대다 단방향 매핑이 가지는 단점을 그대로 가지므로 다대일 양방향 매핑을 사용하는 것이 좋다.

---

## 일대일 [1:1]

- 일대일 관계는 그 반대도 일대일 관계다
- 주 테이블이나 대상 테이블 둘 중 어느 곳이나 외래 키를 가질 수 있다.

### 1. 주 테이블에 외래 키

외래 키를 객체 참조와 비슷하게 사용할 수 있어서 객체지향 개발자들이 선호한다.

주 테이블이 외래 키를 가지고 있으므로 주 테이블만 확인해도 대상 테이블과 연관관계가 있는지 알 수 있다.

- 단방향
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@OneToOne
    	@JoinColumn(name = "LOCKER_ID")
    	private Locker locker;
    	...
    }
    
    @Entity
    public class Locker {
    	@Id @GeneratedValue
    	@Column(name = "LOCKER_ID")
    	private Long id;
    	
    	private String name;
    	...
    }
    ```
    
    일대일 관계이므로 객체 매핑에 @OneToOne을 사용했고, 데이터베이스에는 외래 키에 유니크 제약 조건(UNI) 추가했다. 다대일 단방향과 거의 비슷하다.
    
- 양방향
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@OneToOne
    	@JoinColumn(name = "LOCKER_ID")
    	private Locker locker;
    	...
    }
    
    @Entity
    public class Locker {
    	@Id @GeneratedValue
    	@Column(name = "LOCKER_ID")
    	private Long id;
    	
    	private String name;
    
    	@OneToOne(mappedBy = "locker")
    	private Member member;
    	...
    }
    ```
    
    MEMBER 테이블이 외래 키를 가지고 있으므로 Member.locker가 연관관계의 주인이다.
    
    따라서 Locker.member는 mappedBy를 선언해서 연관관계의 주인이 아니라고 설정했다.
    

### 2. 대상 테이블에 외래 키

전통적인 데이터베이스 개발자들이 선호하는 방식으로 테이블 관계를 일대일에서 일대다로 변경할 때 테이블 구조를 그대로 유지할 수 있다.

- 단방향
    
    JPA에서 지원하지 않고 이런 모양으로 매핑할 수 있는 방법도 없다.
    
    단방향 관계를 Locker에서 Member 방향으로 수정하거나, 양방향 관계로 만들고 Locker를 연관관계의 주인으로 설정해야 한다.
    
- 양방향
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@OneToOne(mappedBy = "member")
    	private Locker locker;
    	...
    }
    
    @Entity
    public class Locker {
    	@Id @GeneratedValue
    	@Column(name = "LOCKER_ID")
    	private Long id;
    	
    	private String name;
    
    	@OneToOne
    	@JoinColumn(name = "LOCKER_ID")
    	private Member member;
    	...
    }
    ```
    
    주 엔터티인 Member 엔터티 대신에 대상 엔터티인 Locker를 연관관계의 주인으로 만들어서 LOCKER 테이블의 외래 키를 관리하도록 했다.
    

---

## 다대다 [N:N]

관계형 데이터베이스는 정규화된 테이블 2개로 다대다 관계를 표현할 수 없다.

그래서 보통 다대다 관계를 일대다, 다대일 관계로 풀어내는 연결 테이블을 사용한다.

그런데 객체는 테이블과 다르게 객체 2개로 다대다 관계를 만들 수 있다.

@ManyToMany를 사용하면 다대다 관계를 편리하게 매핑할 수 있다.

### 1. 다대다: 단방향

- 회원
    
    ```java
    @Entity
    public class Member {
    	@Id @GeneratedValue
    	@Column(name = "MEMBER_ID")
    	private Long id;
    	
    	private String username;
    	
    	@ManyToMany
    	@JoinColumn(name = "MEMBER_PRODUCT", 
    		joinColumns = @JoinColumn(name = "MEMBER_ID"), 
    		inverseJoinColumns = @JoinColumn(name = "PRODUCT_ID"))
    	private List<Product> products= new ArrayList<Product>();
    	...
    }
    ```
    
- 상품
    
    ```java
    @Entity
    public class Product {
    	@Id @Column(name = "PRODUCT_ID")
    	private Long id;
    	
    	private String name;
    	...
    }
    ```
    

@ManyToMany와 @JoinTable을 사용해서 연결 테이블을 바로 매핑했다.

따라서 회원과 상품을 연결하는 회원_상품 엔터티 없이 매핑을 완료할 수 있다.

- @JoinTable.name : 연결 테이블을 지정
- @JoinTable.joinColumns : 현재 방향인 회원과 매핑할 조인 컬럼 정보를 지정
- @JoinTable.inverseJoinColumns : 반대 방향인 상품과 매핑할 조인 컬럼 정보를 지정

### 2. 다대다: 양방향

역방향도 @ManyToMany를 사용하고, 양쪽 중 원하는 곳에 mappedBy로 연관관계의 주인을 지정한다.

양방향 연관관계는 연관관계 편의 메소드를 추가해서 관리하는 것이 편하므로 편의 메소드 추가하고 양뱡향 연관관계를 설정한다.

### 3. 다대다: 매핑의 한계와 극복, 연결 엔터티 사용

@ManyToMany를 사용하면 연결 테이블을 자동으로 처리해주므로 도메인 모델이 단순해지고 여러가지로 편리하지만 이 매핑을 실무에서 사용하기에는 한계가 있다.

연결 테이블을 매핑하는 연결 엔터티를 만들고 이곳에 추가한 컬럼들을 매핑해야 한다.

그리고 엔터티 간의 관계도 테이블 관계처럼 다대다에서 일대다, 다대일 관계로 풀어야 한다.

- 복합 기본 키
    - 복합 키는 별도의 식별자 클래스로 만들어야 한다.
    - Serializable을 구현해야 한다.
    - equals와 hashCode 메소드를 구현해야 한다.
    - 기본 생성자가 있어야 한다.
    - 식별자 클래스는 public이다.
    - @IdClass를 사용하는 방법 외에 @EmbeddedId를 사용하는 방법도 있다.
- 식별 관계
    
    부모 테이블의 기본 키를 받아서 자신의 기본 키 + 외래 키로 사용하는 것
    

### 4. 다대다: 새로운 기본 키 사용

추천하는 기본 키 생성 전략은 데이터베이스에서 자동으로 생성해주는 대리 키를 Long 값으로 사용하는 것이다. 이렇게 되면 간편하고 거의 영구적으로 쓸 수 있고 비즈니스에 의존하지 않는다.

그리고 ORM 매핑 시에 복합 키를 만들지 않아도 되므로 간단히 매핑을 완성할 수 있다.

### 5. 다대다 연관관계 정리

- **식별 관계** : 받아온 식별자를 기본 키 + 외래 키로 사용한다.
- **비식별 관계** : 받아온 식별자는 외래 키로만 사용하고 새로운 식별자를 추가한다.

객체 입장에서 보면 비식별 관계를 사용하는 것이 복합 키를 위한 식별자 클래스를 만들지 않아도 되므로 단순하고 편리하게 ORM 매핑을 할 수 있다.

⇒ 식별 관계보다는 비식별 관계 추천

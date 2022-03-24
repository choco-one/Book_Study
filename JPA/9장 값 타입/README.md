JPA의 데이터 타입은 크게 엔터티 타입과 값 타입으로 나뉜다.
> **엔터티 타입**
> - @Entity로 정의하는 객체
> - 식별자를 통해 지속해서 추적 가능

> **값 타입**
> - int, Integer, String처럼 단순히 값으로 사용하는 자바 기본 타입이나 객체
> - 식별자가 없고 숫자나 문자같은 속성만 있으므로 추적 불가

## 기본값 타입
```java
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  private String name;
  private int age;
```

- String, int가 값 타입
- Member 엔터티는 id라는 식별자 값도 가지고 생명주기도 있지만 값 타입인 name, age 속성은 식별자 값도 없고 생명주기도 엔터티에 의존한다.
- 회원 엔터티 인스턴스를 제거하면 name, age 값도 제거된다.

## 임베디드 타입(복합 값 타입)

새로운 값 타입을 직접 정의해서 사용하는 것을 JPA에서는 **임베디드 타입(embedded type)** 이라고 한다.  
*(하이버네이트는 임베디드 타입을 컴포넌트라 한다.)*  
```java
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  //근무 기간
  @Temporal(TemporalType.DATE) java.util.Date startDate;
  @Temporal(TemporalType.DATE) java.util.Date endDate;
  
  //집 주소 표현
  private String city;
  private String street;
  private String zipcode;
  //...
}
```

위와 같이 회원이 상세한 데이터를 그대로 가지고 있는 것은 객체지향적이지 않으며 응집력을 떨어뜨린다.

```java
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded Period workPeriod; //근무 기간
  @Embedded Address homeAddress; //집 주소
  //...
}
```
```java
@Embeddable
public class Period {
  @Temporal(TemporalType.DATE) java.util.Date startDate;
  @Temporal(TemporalType.DATE) java.util.Date endDate;
  //...
  
  public boolean isWork(Date date) {
    // .. 값 타입을 위한 메소드를 정의할 수 있다.
  }
}
```
```java
@Embeddable
public class Address {

  @Column(name="city") // 매핑할 컬럼 정의 가능
  private String city;
  private String street;
  private String zipcode;
  //...
}
```

[근무기간, 집 주소]를 가지도록 임베디드 타입을 사용하면 위와 같이 바꿀 수 있다.  
임베디드 타입은 기본 생성자가 필수다.  
임베디드 타입을 사용하려면 다음 2가지 어노테이션이 필요하다. (둘 중 하나는 생략 가능)  
- `@Embeddable`: 값 타입을 정의하는 곳에 표시
- `@Embedded`: 값 타입을 사용하는 곳에 표시

임베디드 타입을 포함한 모든 값 타입은 엔터티의 생명주기에 의존하므로 엔터티와 임베디드 타입의 관계를 UML로 표현하면 composition 관계가 된다.  

### 1. 임베디드 타입과 테이블 매핑  
- 임베디드 타입은 엔터티의 값이므로 값이 속한 엔터티의 테이블에 매핑한다.  
- 임베디드 타입 덕분에 객체와 테이블을 아주 세밀하게(fine-grained) 매핑하는 것이 가능하다.  
- 잘 설계한 ORM 어플리케이션은 매핑한 테이블의 수보다 클래스의 수가 더 많다.  
- ORM을 사용하지 않고 개발하면 테이블 컬럼과 객체 필드를 대부분 1:1로 매핑한다.  

### 2. 임베디드 타입과 연관관계  
임베디드 타입은 값 타입을 포함하거나 엔터티를 참조할 수 있다.  
```java
@Entity
public class Member {

  @Embedded Address address;          //임베디드 타입 포함
  @Embedded PhoneNumber phoneNumber; //임베디드 타입 포함
  //...
}

@Embeddable
public class Address {
  String street;
  String city;
  String state;
  @Embedded Zipcode zipcode; //임베디드 타입 포함
}

@Embeddable
public class Zipcode {
  String zip;
  String plusFour;
}

@Embeddable
public class PhoneNumber {
  String areaCode;
  String localNumber;
  @ManyToOne PhoneServiceProvider provider; //엔터티 참조
  ...
}

@Entity
public class PhoneServiceProvider {
  @Id String name;
  ...
}
```
- 값 타입인 Address가 값 타입인 Zipcode를 포함
- 값 타입인 PhoneNumber가 엔터티 타입인 PhoneServiceProvider를 참조

### 3. @AttributeOverride: 속성 재정의

임베디드 타입에 정의한 매핑정보를 재정의하려면 엔터티에 `@AttributeOverride`를 사용하면 된다.  
```java
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded Address homeAddress;
  @Embedded Address companyAddress;
}
```
Member 엔터티에 회사 주소를 하나 더 추가했는데 이 때, 테이블에 매핑하는 컬럼명이 중복된다는 문제가 발생한다.  
```java
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  private String name;
  
  @Embedded Address homeAddress;
  @Embedded 
  @AttributeOverrides({
    @AttributeOverride(name="city", column=@Column(name = "COMPANY_CITY")), 
    @AttributeOverride(name="street", column=@Column(name = "COMPANY_STREET")), 
    @AttributeOverride(name="zipcode", column=@Column(name = "COMPANY_ZIPCODE")), 
    Address companyAddress;
}
```
위와 같이 `@AttributeOverrides`를 사용해서 매핑정보를 재정의할 수 있다.  
`@AttributeOverride`를 사용하면 어노테이션을 너무 많이 사용해서 엔터티 코드가 지저분해지는데 다행히 한 엔터티에 같은 임베디드 타입을 중복해서 사용하는 일은 많지 않다.  
아래와 같이 생성된 테이블을 보면 재정의한대로 변경되어 있는 것을 확인할 수 있다.  
```sql
CREATE TABLE MEMBER (
  COMPANY_CITY varchar(255), 
  COMPANY_STREET varchar(255), 
  COMPANY_ZIPCODE varchar(255), 
  city varchar(255), 
  street varchar(255), 
  zipcode varchar(255), 
  ...
)
```

### 4. 임베디드 타입과 null
임베디드 타입이 null이면 매핑한 컬럼 값은 모두 null이 된다.  
```java
member.setAddress(null); //null 입력
em.persist(member);
```
-> 회원 테이블의 주소와 관련된 CITY, STREET, ZIPCODE 컬럼 값은 모두 null이 된다.  

---

## 값 타입과 불변 객체  
### 1. 값 타입 공유 참조  
임베디드 타입 같은 값 타입을 여러 엔터티에서 공유하면 위험하다.  
```java
member1.setHomeAddress(new Address("OldCity"));
Address address = member1.getHomeAddress();

member2.setCity("NewCity"); // 회원1의 address 값을 공유해서 사용
member2.setHomeAddress(address);
```

회원2에 새로운 주소를 할당하려고 회원1의 주소를 그대로 참조해서 사용했는데  
이런 경우에는 회원1의 주소도 "NewCity"로 변경되는 문제가 발생한다.  
-> 회원1과 회원2가 같은 address 인스턴스를 참조하기 때문에 영속성 컨텍스트가 회원1, 회원2 둘 다 city 속성이 변경된 것으로 판단해서 회원1, 회원2 각각 UPDATE SQL을 실행한다.  
=> 이렇게 뭔가를 수정했는데 예상 못한 곳에서 문제가 발생하는 것을 부작용(side effect)라 하고, 이런 부작용을 막으려면 값을 복사해서 사용하면 된다.  

### 2. 값 타입 복사  
```java
member1.setHomeAddress(new Address("OldCity"));
Address address = member1.getHomeAddress();

// 회원1의 address 값을 복사해서 새로운 newAddress 값을 생성
Address newAddress = address.clone();

newAddress.setCity("NewCity");
member2.setHomeAddress(address);
```
-> 회원2의 주소만 "NewCity"로 변경해서 영속성 컨텍스트는 회원2의 주소만 변경된 것으로 판단하고 회원2에 대해서만 UPDATE SQL을 실행한다.  

임베디드 타입처럼 직접 정의한 값 타입은 자바 기본 타입이 아니라 객체 타입이다.  
자바는 기본 타입에 값을 대입하면 값을 복사해서 전달하므로 부작용이 없지만,  
객체에 값을 대입하면 항상 참조 값을 전달하기 때문에 공유 참조로 인한 부작용이 발생할 수 있다.  
문제는 인스턴스를 복사하지 않고 원본의 참조 값을 직접 넘기는 것을 막을 방법이 없다는 것인데  
근본적인 해결책은 객체의 값을 수정하지 못하도록 수정자 메소드를 모두 제거하는 방법이다.  

### 3. 불변 객체  
객체를 불변하게 만들면 값을 수정할 수 없으므로 부작용을 차단할 수 있다.  
따라서 값 타입은 될 수 있으면 **불변 객체(immutable Object)** 로 설계해야 한다.  
불변 객체의 값은 조회할 수 있지만 수정할 수 없으므로 부작용이 발생하지 않는다.  
불변 객체는 생성자로만 값을 설정하고 수정자를 만들지 않는 방법으로 구현할 수 있다.  
*(Integer, String은 자바가 제공하는 대표적인 불변 객체다.)*  
```java
@Embeddable
public class Address {
  private String city;
  
  protected Address() {} //JPA에서 기본 생성자는 필수다.  
  
  //생성자로 초기 값을 설정한다.
  public Address(String city) {this.city = city}
  
  //접근자(Getter)는 노출한다.
  public String getCity() {
    return city;
  }
  
  //수정자(Setter)는 만들지 않는다.
}
```
```java
Address address = member1.getHomeAddress();
//회원1의 주소값을 조회해서 새로운 주소값을 생성
Address newAddress = new Address(address.getCity());
member2.setHomeAddress(newAddress);
```

---

## 값 타입의 비교  
- 동일성 비교 : 인스턴스의 참조 값을 비교, `==` 사용
- 동등성 비교 : 인스턴스의 값을 비교, `equals()` 사용  
값 타입을 비교할 때는 `equals()`를 사용해서 동등성 비교를 해야 한다.  
값 타입의 `equals()` 메소드를 재정의할 때는 보통 모든 필드의 값을 비교하도록 구현한다.  

---

## 값 타입 컬렉션
값 타입을 하나 이상 저장하려면 컬렉션에 보관하고 `@ElementCollection`, `@CollectionTable` 어노테이션을 사용하면 된다.  

### 1. 값 타입 컬렉션 사용  
```java
Member member = new Member();

//임베디드 값 타입
member.setHomeAddress(new Address("통영","몽돌해수욕장","660-123"));

//기본값 타입 컬렉션
member.getFavoriteFoods().add("짬뽕");
member.getFavoriteFoods().add("짜장");
member.getFavoriteFoods().add("탕수육");

//임베디드 값 타입 컬렉션
member.getAddressHistory().add(new Address("서울","강남","123-123"));
member.getAddressHistory().add(new Address("서울","강북","000-000"));

em.persist(member);
```

실제 데이터베이스에 실행되는 INSERT SQL은 다음과 같다.  
- `member`: INSERT SQL 1번
- `member.homeAddress`: 컬렉션이 아닌 임베디드 값 타입이므로 회원테이블을 저장하는 SQL에 포함된다.
- `member.favoriteFoods`: INSERT SQL 3번
- `member.addressHistory`: INSERT SQL 2번

```java
Member member = em.find(Member.class, 1L);

//1. 임베디드 값 타입 수정
member.setHomeAddress(new Address("새로운도시","신도시1","123456"));

//2. 기본값 타입 컬렉션 수정
Set<String> favoriteFoods = member.getFavoriteFoods();
favoriteFoods.remove("탕수육");
favoriteFoods.add("치킨");

//3. 임베디드 값 타입 컬렉션 수정
List<Address> addressHistory = member.getAddressHistory();
addressHistory.remove(new Address("서울","기존 주소","123-123"));
addressHistory.add(new Address("새로운도시","새로운 주소","123-456"));
```
값 타입 컬렉션을 수정하면  
1. **임베디드 값 타입 수정** : homeAddress 임베디드 값 타입은 MEMBER 테이블과 매핑했으므로 MEMBER 테이블만 UPDATE 한다.  
2. **기본값 타입 컬렉션 수정** : 탕수육을 치킨으로 변경하려면 탕수육을 제거하고 치킨을 추가해야 한다. 자바의 String 타입은 수정할 수 없다.  
3. **임베디드 값 타입 컬렉션 수정** : 값 타입은 불변해야 한다. 따라서 컬렉션에서 기존 주소를 삭제하고 새로운 주소를 등록했다. 참고로 값 타입은 equals, hashcode를 꼭 구현해야 한다.  



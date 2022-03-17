## **💫 프로젝트 개발 툴 설치와 프로젝트 불러오기**

<aside>
📌 H2 데이터베이스는 JVM 메모리 안에서 실행되는 **임베디드 모드**와, 실제 데이터베이스처럼 별도의 서버를 띄워 동작하는 **서버 모드**가 있다.

- 임베디드 모드
    - application이 종료된다면 저장, 수정된 Data가 손실(휘발)되어 기본적으로는 영속적이지 않은 방식
    - 메인 메모리에 띄워놓고 사용하기에 데이터 접근이 매우 빠름
- 서버 모드
    - 별도의 프로세스로 DB를 동작시키는 방식
    - 모든 데이터의 처리 흐름이 TCP/IP를 통하여 전송되기 때문에 Embedded 모드보다 상대적으로 느림
</aside>

---

## **💫 라이브러리와 프로젝트 구조**

여기서는 라이브러리 관리 도구로서 메이븐을 사용한다. 그리고 책 전반에 걸쳐서 JPA 구현체로 Hibernate를 사용한다. 핵심 라이브러리로는 `hibernate-core` , `hibernate-entitymanager` , `hibernate-jpa-2.1-api` 를 사용한다.

<aside>
📌 `hibernate-entitymanager` 를 라이브러리로 지정하면, 나머지 2개의 라이브러리도 함께 내려받는다.

</aside>

### **➰ 메이븐과 사용 라이브러리 관리**

<aside>

📌 **메이븐**은 라이브러리 관리 기능과 빌드 기능을 제공함으로써, 자바 애플리케이션의 라이브러리를 관리해주는 도구이다.

- **라이브러리 관리 기능
:** 자바 애플리케이션 개발에 필요한 jar 라이브러리 파일들을 자동으로 내려받고 관리해주는 기능
- **빌드 기능**
: 애플리케이션을 직접 빌드하는 고된 작업이 아닌 표준화된 방법을 제공한다.
</aside>

메이븐 설정 파일(`pom.xml`)을 보면, `<dependencies>` 에 사용할 라이브러리를 지정해주고 있다. `groupId + artifactId + version` 만 명시하여 라이브러리(jar 파일)을 내려받아 추가한다.

---

## **💫 객체 매핑 시작**

아까 생성한 테이블을 토대로, 애플리케이션에서 사용할 회원 클래스를 만든다.

```java
public class Member {

    private String id;

    private String username;

    private Integer age;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

JPA를 사용하려면 먼저 회원 클래스와 회원 테이블을 매핑해야 한다. (**객체와 테이블 간 매핑**) 다음 표를 참고해 둘을 비교하면서 매핑을 수행한다.

위 표를 참고해 JPA가 제공하는 매핑 annotation을 적용한다.

```java
import javax.persistence.*;

@Entity
@Table(name="MEMBER")
public class Member {

    @Id
    @Column(name = "ID")
    private String id;

    @Column(name = "NAME")
    private String username;

    private Integer age;

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

**`@Entity`**

- 해당 클래스를 테이블과 매핑한다고 명시한다. 이 클래스를 엔티티 클래스라고 한다.

**`@Table`**

- 엔티티 클래스에 매핑할 테이블 정보를 알려준다. 위 예시에서는 `name` 속성을 사용해 `Member` entity를 `MEMBER` 테이블에 매핑했다.
- 해당 annotation이 없다면 클래스 이름을 테이블 이름으로 매핑한다. (정확히는 엔티티 이름을 사용한다.)

**`@Id`**

- 엔티티 클래스의 필드를 테이블의 기본 키에 매핑한다. 위 예시에서는 엔티티의 `id` 필드를 테이블의 `ID` 기본 키 컬럼에 매핑했다.
- 이 필드를 식별자 필드라고 한다.

**`@Column`**

- 필드를 컬럼에 매핑한다. 위 예시에서는 `name` 속성을 사용해 `Member` entity의 `username` 필드를 `MEMBER` 테이블의 `NAME` 컬럼에 매핑했다.

**매핑 정보가 없는 필드**

- 매핑 annotation을 생략하면 필드명을 사용해 컬럼명으로 매핑한다. 여기서는 데이터베이스가 대소문자를 구분하기 않는다고 가정하여 `age` 필드가 `age` 컬럼으로 매핑된다.
    - 대소문자를 구분한다면 `@Column(name="AGE")` 로 명시해야 한다.

---

## **💫 `persistence.xml` 설정**

JPA는 `persistence.xml` 을 사용해서 필요한 설정 정보를 관리한다. 이 설정 파일은 `META-INF/persistence.xml` 클래스 패스 경로에 있어야 별도의 설정 없이 JPA가 인식한다.

```xml
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence" version="2.1">
```

- 설정 파일은 `persistence` 로 시작한다. 여기에 XML namespace(ns)와 사용할 버전을 지정한다.

```xml
<persistence-unit name="jpabook">
```

- JPA 설정은 영속성 유닛(`persistence-unit`)이라는 것부터 시작하는데, 일반적으로 연결할 데이터베이스당 하나의 영속성 유닛을 등록한다.
- 그리고 영속성 유닛은 고유한 이름이 필요해 `jpabook` 을 사용했다.

아래에는 JPA 표준 속성과 Hibernate 속성이 차례로 위치한다.

- 이름이 `javax.persistence` 로 시작하는 JPA 표준 속성은 특정 구현체 종속되지 않는다.
- 하지만 `hibernate` 로 시작하는 속성은 Hibernate 전용 속성으로 Hibernate에서만 사용할 수 있다.
    - 가장 중요한 속성은 데이터베이스 방언을 설정하는 `hibernate.dialect` 이다.

### **➰ 데이터베이스 방언**
📌 JPA는 특정 데이터베이스에 종속적이지 않은 기술이다.

따라서 다른 데이터베이스로 손쉽게 교체할 수 있는데, 각 데이터베이스가 제공하는 문법과 함수가 조금씩 다른 문제가 있다.

- 데이터 타입 : `VARCHAR` vs. `VARCHAR2`
- 다른 함수명 : `SUBSTRING()` vs. `SUBSTR()`
- 페이징 처리 : `LIMIT` vs. `ROWNUM`

이처럼 SQL 표준을 지키지 않거나, 특정 데이터베이스만의 고유한 기능을 JPA에서는 **방언**(dialect)이라 한다.

- 당연히 이런 경우가 많아지면, 추후 데이터베이스의 변경을 어렵게 한다.
- 대부분의 JPA 구현체들은 이 문제를 해결하기 위해 다양한 데이터베이스 방언 클래스를 제공한다.

**어떻게 적용이 되느냐?**

- 개발자는 JPA가 제공하는 표준 문법에 맞춰 JPA를 사용한다.
- 특정 데이터베이스에 의존적인 SQL은 데이터베이스 방언이 처리한다.

따라서 애플리케이션 코드의 변경이 아닌 데이터베이스 방언의 변경으로 적용된다!

Hibernate는 대표적으로 `H2` , `오라클 10g` , `MySQL` 데이터베이스 방언을 제공한다. 여기서 H2 데이터베이스를 사용하기에 `dialect` 속성을 `H2Dialect` 로 설정했다.

---

## **💫 애플리케이션 개발**

`JpaMain` 은 애플리케이션을 시작하는 코드이다.

```java
public class JpaMain {

    public static void main(String[] args) {

        //엔티티 매니저 팩토리 생성
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
        EntityManager em = emf.createEntityManager(); //엔티티 매니저 생성

        EntityTransaction tx = em.getTransaction(); //트랜잭션 기능 획득

        try {
            tx.begin(); //트랜잭션 시작
            logic(em);  //비즈니스 로직
            tx.commit();//트랜잭션 커밋

        } catch (Exception e) {
            e.printStackTrace();
            tx.rollback(); //트랜잭션 롤백
        } finally {
            em.close(); //엔티티 매니저 종료
        }

        emf.close(); //엔티티 매니저 팩토리 종료
    }

    public static void logic(EntityManager em) {

        String id = "id1";
        Member member = new Member();
        member.setId(id);
        member.setUsername("지한");
        member.setAge(2);

        //등록
        em.persist(member);

        //수정
        member.setAge(20);

        //한 건 조회
        Member findMember = em.find(Member.class, id);
        System.out.println("findMember=" + findMember.getUsername() + ", age=" + findMember.getAge());

        //목록 조회
        List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
        System.out.println("members.size=" + members.size());

        //삭제
        em.remove(member);
    }
}
```

코드는 크게 3부분으로 나뉜다.

- 엔티티 매니저 설정
- 트랜잭션 관리
- 비즈니스 로직

### **➰ 엔티티 매니저 설정**

흐름은

1. `Persistence` 가 `META-INF/persistence.xml` 에서 설정 정보를 조회한다.
2. 조회한 설정 정보를 가지고 `EntityManageFactory` 를 생성한다.
3. `EntityManagerFactory` 는 `EntityManager` 를 생성한다.

**엔티티 매니저 팩토리 생성**

JPA를 시작하려면 `persistence.xml` 의 설정 정보를 사용해 엔티티 매니저 팩토리를 생성해야 한다. 이때 `Persistence` 클래스를 사용한다.

```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("jpabook");
```

- 설정 파일에서 이름이 `jpabook` 인 영속성 유닛을 찾아 엔티티 매니저 팩토리를 생성한다.
- 이때 설정 파일의 설정 정보를 읽어 JPA를 동작시키기 위한 기반 객체를 만들고 JPA에 따라서는 데이터베이스 커넥션 풀도 생성하므로 **엔티티 매니저 팩토리를 생성하는 하는 비용은 아주 크다.**

📌 "따라서 엔티티 매니저 팩토리는 애플리케이션 전체에서 **딱 1번만 생성**하고, **공유**해서 사용해야 한다."

**엔티티 매니저 생성**

```java
EntityManager em = emf.createEntityManager();
```

- 엔티티 매니저 팩토리에서 엔티티 매니저를 생성한다. JPA이 대부분의 기능은 이 엔티티 매니저가 제공한다.
- 대표적으로 엔티티를 데이터베이스에 **등록/수정/삭제/조회**하는 기능이 있다.
- 엔티티 매니저는 내부에 데이터베이스(데이터베이스 커넥션)를 유지하면서 데이터베이스와 통신한다. 따라서 **가상의 데이터베이스**라고 생각할 수도 있다.

📌 "엔티티 매니저는 데이터베이스 커넥션과 밀접한 관계를 가지므로 **스레드 간 공유하거나 재사용해서는 안된다.**"

**종료**

사용이 끝난 엔티티 매니저와, 애플리케이션이 종료될 때 엔티티 매니저 팩토리를 반드시 종료해야 한다.

```java
em.close();
...

emf.close();
```

### **➰ 트랜잭션 관리**

JPA를 사용하면 항상 트랜잭션 안에서 데이터를 변경해야 한다. 그렇지 않으면 데이터베이스의 일관성을 깨트릴 수 있고 예외가 발생한다.

- 트랜잭션을 시작하려면 엔티티 매니저(`em`)에서 트랜잭션 API를 받아와야 한다.
- 비즈니스 로직이 정상 동작하면 트랜잭션을 커밋하고, 예외가 발생하면 롤백한다.

```java
try {
    tx.begin(); //트랜잭션 시작
    logic(em);  //비즈니스 로직
    tx.commit();//트랜잭션 커밋

} catch (Exception e) {
    e.printStackTrace();
    tx.rollback(); //트랜잭션 롤백
}
```

### **➰ 비즈니스 로직**

비즈니스 로직은 회원 엔티티를 하나 생성한 후 엔티티 매니저를 통해 데이터베이스에 등록, 수정, 삭제, 조회한다.

<aside>
📌 모든 기능이 엔티티 매니저를 통해 수행되어 객체를 저장하는 가상의 데이터베이스처럼 보인다.

</aside>

```java
public static void logic(EntityManager em) {

    String id = "id1";
    Member member = new Member();
    member.setId(id);
    member.setUsername("지한");
    member.setAge(2);

    //등록
    em.persist(member);

    //수정
    member.setAge(20);

    //한 건 조회
    Member findMember = em.find(Member.class, id);
    System.out.println("findMember=" + findMember.getUsername() + ", age=" + findMember.getAge());

    //목록 조회
    List<Member> members = em.createQuery("select m from Member m", Member.class).getResultList();
    System.out.println("members.size=" + members.size());

    //삭제
    em.remove(member);
}
```

**등록**

- 엔티티의 저장은 `persist()` 메소드에 저장할 엔티티를 넘겨주기만 하면 된다.
- 이때 JPA는 회원 엔티티의 매핑 정보(annotation)를 분석해 INSERT SQL을 만들어 데이터베이스에 전달한다.

```sql
INSERT INTO MEMBER (ID, NAME, AGE) VALUES ('id1' , '지한', 2)
```

**수정**

- 수정은 조금 이상해보인다. 엔티티 수정 후 수정된 내용을 반영해주는 `em.update()` 같은 메소드가 필요할 것 같은데, 그저 값만 변경하고 말았다.
- JPA는 어떤 엔티티가 변경되었는지 추적하는 기능이 있다. 따라서 `member.setAge(20)` 처럼 값만 변경하면 UPDATE SQL을 생성해 데이터베이스에 전달한다.

```sql
UPDATE MEMBER SET AGE=20, NAME='지한' WHERE ID='id1'
```

**삭제**

- 엔티티를 삭제하려면 `remove()` 메소드를 사용하면 되고, 이 역시 JPA에서 DELETE SQL을 생성해 데이터베이스에 전달한다.

```sql
DELETE FROM MEMBER WHERE ID='id1'
```

**한 건 조회**

- 한 건 조회를 위해 사용하는 `find()` 메소드는 조회할 엔티티 타입과 `@Id` 로 데이터베이스 테이블의 기본 키와 매핑한 식별자 값으로 엔티티 하나를 조회한다.
- 역시나 SELECT SQL을 생성하고, 조회한 결과 값으로 엔티티를 생성해 반환한다.

```sql
SELECT * FROM MEMBER WHERE ID='id1'
```

### **➰ JPQL**

JPA를 사용하면 개발자는 엔티티 객체 중심으로 개발하고, 데이터베이스에 대한 처리는 JPA가 수행하게 된다.

문제는 **검색 쿼리**다. JPA는 엔티티 객체를 중심으로 개발하므로 검색 시에도 테이블이 아닌 엔티티 객체를 대상으로 검색해야 한다.

그런데 이러한 검색 방식은 데이터베이스의 모든 데이터를 애플리케이션으로 불러와 엔티티 객체로 변경한 다음 검색해야 해서 사실상 불가능하다. 결국 하나 이상의 회원 목록을 조회하는 것과 같은 경우는 검색 조건이 포함된 SQL을 사용해야 한다.

📌 "JPA는 **JPQL**(Java Persistence Query Language)이라는 SQL을 추상화한 쿼리 언어로 이런 문제를 해결한다."

- 문법은 SQL과 거의 유사하지만, 가장 큰 차이점은 다음과 같다.
    - JPQL은 **엔티티 객체를 대상**으로 쿼리한다. 즉 클래스와 필드를 대상으로 쿼리한다.
    - SQL은 **데이터베이스 테이블을 대상**으로 쿼리한다.

```sql
select m from Member m
```

- 여기서 `from Member` 는 회원 엔티티 객체를 말하는 것이지 데이터베이스의 `MEMBER` 테이블을 말하는 것이 아니다.
    - **"JPQL는 데이터베이스 테이블을 전혀 알지 못한다."**
- `em.createQuery(JPQL, 반환 타입)` 메소드를 실행해 퀴리 객체를 생성하고,
- 쿼리 객체의 `getResultList()` 메소드를 호출하여 JPQL을 사용할 수 있다.
- JPA는 JPQL을 분석해 적절한 SQL을 생성해 데이터베이스에 전달한다.
## 트랜잭션과 락

### 1. 트랜잭션과 격리 수준  
트랜잭션은 ACID(Atomicity, Consistency, Isolation, Durability)를 보장해야 한다.  
- 원자성 : 트랜잭션 내에서 실행한 작업들은 하나의 작업인 것처럼 모두 성공 or 모두 실패해야 함  
- 일관성 : 모든 트랜잭션은 일관성 있는 데이터베이스 상태를 유지해야 함  
- 격리성 : 동시에 실행되는 트랜잭션들이 서로에게 영향을 미치지 않도록 해야 함 / 동시성과 관련된 성능 이슈로 인해 격리 수준을 선택할 수 있음  
- 지속성 : 트랜잭션을 성공적으로 끝내면 결과가 항상 기록되어야 함 / 시스템에 문제가 발생해도 데이터베이스 로그 등을 사용해 복구해야 함  

트랜잭션은 원자성, 일관성, 지속성은 보장하지만 트랜잭션 간 격리성을 완벽히 보장하려면 동시성 처리 성능이 매우 나빠진다는 문제점이 있다.  
이런 문제로 인해 ANSI 표준은 트랜잭션의 격리 수준을 4단계로 나누었다.  
- READ UNCOMMITED  
  : 커밋하지 않은 데이터를 읽을 수 있음  
   트랜잭션1이 데이터를 수정하고 있는데 커밋하지 않아도 트랜잭션2가 수정 중인 데이터를 조회할 수 있는 것을 보고 DIRTY READ라고 함  
   DIRTY READ를 허용하는 격리 수준  
- READ COMMITTED  
  : 커밋한 데이터만 읽을 수 있음  
   트랜잭션1이 데이터를 조회 중인데 트랜잭션2가 그 데이터를 수정하고 커밋하면 트랜잭션1이 다시 조회했을 때 수정된 데이터가 조회됨. 이런 반복해서 같은 데이터를 읽을 수 없는 상태를 NON-REPEATABLE READ라고 함  
   DIRTY READ는 허용하지 않지만 NON-REPEATABLE READ는 허용하는 격리 수준  
- REPEATABLE READ  
  : 한 번 조회한 데이터를 반복해서 조회해도 같은 데이터가 조회됨  
   트랜잭션1이 10살 이하의 회원을 조회했는데 트랜잭션2가 5살 회원을 추가하고 커밋하면 트랜잭션1이 다시 10살 이하의 회원을 조회했을 때 회원 하나가 추가된 상태로 조회됨. 이런 반복 조회 시 결과 집합이 달라지는 것을 PHANTOM READ라고 함  
   NON-REPEATABLE READ는 허용하지 않지만 PHANTOM READ는 허용하는 격리 수준  
- SERIALIZABLE  
  : 가장 엄격한 격리 수준  
   동시성 처리 성능이 급격히 떨어질 수 있음  

|격리 수준|DIRTY READ|NON-REPEATABLE READ|PHANTOM READ|
|---|---|---|---|
|READ UNCOMMITTED|o|o|o|
|READ COMMITTED||o|o|
|REPEATABLE READ|||o|
|SERIALIZABLE||||

어플리케이션 대부분은 동시성 처리가 중요하므로 보통 READ COMMITTED 수준을 기본으로 사용한다.  
더 높은 격리 수준이 필요하면 데이터베이스 트랜잭션이 제공하는 잠금 기능을 사용하면 된다.  

### 2. 낙관적 락과 비관적 락 기초  
JPA의 영속성 컨텍스트를 활용하면 트랜잭션이 READ COMMITTED 수준이어도 어플리케이션 레벨에서 REPEATABLE READ가 가능하다.  
(엔터티가 아닌 스칼라 값을 직접 조회하면 영속성 컨텍스트의 관리를 받지 못하므로 REPEATABLE READ가 불가능)  
JPA는 트랜잭션 격리 수준을 READ COMMITTED로 가정한다. 더 높은 격리 수준이 필요하면 낙관적 락과 비관적 락 중 하나를 사용하면 된다.  

- 낙관적 락  
  : 트랜잭션 대부분이 충돌이 발생하지 않는다고 낙관적으로 가정하는 방법 / 데이터베이스가 제공하는 락 기능을 사용하지 않고, JPA가 제공하는 버전 관리 기능을 사용함 / 트랜잭션을 커밋하기 전까지는 트랜잭션의 충돌을 알 수 없음  
- 비관적 락  
  : 트랜잭션의 충돌이 발생한다고 가정하고 우선 락을 거는 방법 / `SELECT FOR UPDATE`구문처럼 데이터베이스가 제공하는 락 기능을 사용  
  
**두 번의 갱신 분실 문제(second lost updates problem)**  
두 번의 갱신 분실 문제는 데이터베이스 트랜잭션의 범위를 넘어선다. 트랜잭션만으로는 문제를 해결할 수 없다.  
- 마지막 커밋만 인정하기 : 사용자 A의 내용은 무시하고 마지막에 커밋한 사용자 B의 내용만 인정  
- 최초 커밋만 인정하기 : 사용자 A가 이미 수정을 완료했으므로 사용자 B가 수정을 완료할 때 오류가 발생  
- 충돌하는 갱신 내용 병합하기 : 사용자 A와 사용자 B의 수정사항을 병합  

기본으로는 마지막 커밋만 인정하기가 사용되지만 상황에 따라 최초 커밋만 인정하기가 더 합리적일 수 있다.  
JPA가 제공하는 버전 관리 기능을 사용해 최초 커밋만 인정하기를 구현할 수 있다.  
충돌하는 갱신 내용 병합하기의 경우에는 개발자가 직접 사용자를 위해 병합 방법을 제공해야 한다.  

### 3. @Version  
JPA가 제공하는 낙관적 락을 사용하려면 `@Version` 어노테이션을 사용해서 버전관리 기능을 추가해야 한다.  
**@Version 적용 가능 타입**  
- Long (long)  
- Integer (int)  
- Short (short)  
- Timestamp
```java
@Entity
public class Board {
  @Id
  private String id;
  private String title;
  
  @Version
  private Integer version;
}
```
버전 관리 기능을 적용하려면 엔터티에 버전 관리용 필드를 하나 추가하고 `@Version`을 붙이면 된다.  
그러면 엔터티를 수정할 때마다 버전이 하나씩 자동으로 증가하고, 엔터티를 수정할 때 조회 시점의 버전과 수정 시점의 버전이 다르면 예외가 발생한다.  
=> 버전 정보를 사용하면 최초 커밋만 인정하기가 적용  

**버전 정보 비교 방법**  
JPA가 버전 정보를 비교하는 방법은 엔터티를 수정하고 트랜잭션을 커밋하면 영속성 컨텍스트를 플러시 하면서 다음과 같은 UPDATE 쿼리를 실행한다.  
이 때 버전을 사용하는 엔터티면 검색 조건에 엔터티의 버전 정보를 추가한다.  
```sql
UPDATE BOARD
SET
  TITLE=?
  VERSION=? (버전 + 1 증가)
WHERE
  ID=?
  AND VERSION=? (버전 비교)
```
데이터베이스 버전과 엔터티 버전이 같으면 데이터를 수정하면서 동시에 버전도 하나 증가시킨다.  
데이터베이스 버전이 이미 증가해서 수정 중인 엔터티 버전과 다르면 WHERE문에서 VERSION 값이 다르므로 이미 버전이 증가한 것으로 판단해 JPA가 예외를 발생시킨다.  
버전은 엔터티의 값을 변경하면 증가하기 때문에 값 타입인 임베디드 타입과 값타입 컬렉션을 수정하면 엔터티의 버전이 증가한다.  
(연관관계 필드는 외래 키를 관리하는 연관관계의 주인 필드를 수정할 때만 버전이 증가)  
`@Version`으로 추가한 버전 관리 필드는 JPA가 직접 관리하므로 개발자가 임의로 수정하면 안된다. (벌크 연산 제외)  
버전 값을 강제로 증가하려면 특별한 락 옵션을 선택해야 함  

### 4. JPA 락 사용  
락은 다음 위치에 적용할 수 있다.  
- `EntityManager.lock()`, `EntityManager.find()`, `EntityManager.referesh()`  
- `Query.setLockMode()` (TypeQuery 포함)  
- `@NamedQuery`  

```java
Board board = em.find(Board.class, id, LockModeType.OPTIMISTIC);
```
조회하면서 즉시 락을 걸 수도 있고,  

```java
Board board = em.find(Board.class, id);
...
em.lock(board, LockModeType.OPTIMISTIC);
```
필요할 때 락을 걸 수도 있다.  
JPA가 제공하는 락 옵션은 `javax.persistence.LockModeType`에 정의되어 있다.  

|락 모드|타입|설명|
|---|---|---|
|낙관적 락|OPTIMISTIC|낙관적 락을 사용|
|낙관적 락|OPTIMISTIC_FORCE_INCREMENT|낙관적 락 + 버전 정보를 강제로 증가|
|비관적 락|PESSIMISTIC_READ|비관적 락, 읽기 락을 사용|
|비관적 락|PESSIMISTIC_WRITE|비관적 락, 쓰기 락을 사용|
|비관적 락|PESSIMISTC_FORCE_INCREMENT|비관적 락 + 버전 정보를 강제로 증가|
|기타|NONE|락을 걸지 않음|
|기타|READ|JPA1.0 호환 기능으로 OPTIMISTIC과 같으므로 OPTIMISTIC을 사용하면 됨|
|기타|WRITE|JPA1.0 호환 기능으로 OPTIMISTIC_FORCE_INCREMENT와 같음|

### 5. JPA 낙관적 락  
JPA가 제공하는 낙관적 락은 버전을 사용하고, 트랜잭션을 커밋하는 시점에 충돌을 알 수 있다는 특징이 있다.  
락 옵션 없이 `@Version`만 있어도 낙관적 락이 적용되지만, 락 옵션을 사용하면 락을 더 세밀하게 제어할 수 있다.  
**낙관적 락에서 발생하는 예외**  
- `javax.persistence.OptimisticLockException`(JPA 예외)  
- `org.hibernate.StaleObjectStateException`(하이버네이트 예외)  
- `org.springframework.orm.ObjectOptimisticLockingFailureException`(스프링 예외 추상화)  

**NONE**  
락 옵션을 적용하지 않아도 엔터티에 @version이 적용된 필드만 있으면 낙관적 락이 적용된다.  
- 용도 : 조회한 엔터티를 수정할 때 다른 트랜잭션에 의해 변경되지 않아야 함 / 조회 시점부터 수정 시점까지를 보장  
- 동작 : 엔터티를 수정할 때 버전을 체크하면서 버전을 증가(UPDATE 쿼리 사용) / 데이터베이스의 버전 값이 현재 버전이 아니면 예외 발생  
- 이점 : second lost updates problem을 예방  

**OPTIMISTIC**  
이 옵션을 추가하면 엔터티를 조히만 해도 버전을 체크한다.  
한 번 조회한 엔터티는 트랜잭션을 종료할 때까지 다른 트랜잭션에서 변경하지 않음을 보장한다.  
- 용도 : 조회한 엔터티는 트랜잭션이 끝날 때까지 다른 트랜잭션에 의해 변경되지 않아야 함 / 조회 시점부터 트랜잭션이 끝날 때까지 조회한 엔터티가 변경되지 않음을 보장  
- 동작 : 트랜잭션을 커밋할 때 버전 정보를 조회해서(SELECT 쿼리 사용) 현재 엔터티 버전과 같은지 검증 / 같지 않다면 예외 발생  
- 이점 : DIRTY READ와 NON-REPEATABLE READ를 방지  

**OPTIMISTIC_FORCE_INCREMENT**  
낙관적 락을 사용하면서 버전 정보를 강제로 증가한다.  
- 용도 : 논리적인 단위의 엔터티 묶음을 관리할 수 있음  
- 동작 : 엔터티를 수정하지 않아도 트랜잭션을 커밋할 때 UPDATE 쿼리를 사용해서 버전 정보를 강제로 증가시킴 / 이때 데이터베이스의 버전이 엔터티의 버전과 다르면 예외 발생 / 추가로 엔터티를 수정하면 수정 시 버전 UPDATE가 발생(=> 총 2번의 버전 증가가 나타날 수 있음)  
- 이점 : 강제로 버전을 증가해서 논리적인 단위의 엔터티 묶음을 버전 관리할 수 있음  

### 6. JPA 비관적 락  
데이터베이스 트랜잭션 락 메커니즘에 의존하는 방법으로 주로 SQL 쿼리에 `SELECT FOR UPDATE` 구문을 사용하면서 시작하고 버전 정보는 사용하지 않는다.  
비관적 락은 주로 PERSSIMISTIC_WRITE 모드를 사용한다.  
- 엔터티가 아닌 스칼라 타입을 조회할 때도 사용할 수 있음  
- 데이터를 수정하는 즉시 트랜잭션 충돌을 감지할 수 있음  

**비관적 락에서 발생하는 예외**  
- `javax.persistence.PessimisticLockException`(JPA 예외)  
- `org.springframework.dao.PessimisticLockingFailureException`(스프링 예외 추상화)  

**PESSIMISTIC_WRITE**  
비관적 락의 일반적 옵션으로 데이터베이스에 쓰기 락을 걸 때 사용한다.  
- 용도 : 데이터베이스에 쓰기 락을 건다.  
- 동작 : 데이터베이스 `SELECT FOR UPDATE`를 사용해서 락을 건다.  
- 이점 : NON-REPEATABLE READ를 방지 / 락이 걸린 row는 다른 트랜잭션이 수정할 수 없음  

**PESSIMISTIC_READ**  
데이터를 반복 읽기만 하고 수정하지 않는 용도로 락을 걸 때 사용하고 일반적으로 잘 사용하지 않는다.  
데이터베이스 대부분은 방언에 의해 PESSIMISTIC_WRITE로 동작한다.  
- MySQL : lock in share mode  
- PostgreSQL : for share  

**PESSIMISTIC_FORCE_INCREMENT**  
비관적 락 중 유일하게 버전 정보를 사용하고 강제로 증가시킨다.  
하이버네이트는 nowait를 지원하는 데이터베이스에 대해서 `for update nowait`옵션을 적용한다.  
- 오라클 : for update nowait  
- PostgreSQL : for update nowait  
- nowait를 지원하지 않으면 for update가 사용  

### 7. 비관적 락과 타임 아웃  
비관적 락을 사용하면 락을 획득할 때까지 트랜잭션이 대기한다.  
무한정 기다릴 수 없으므로 타임아웃 시간을 줄 수 있는데, 타임아웃은 데이터베이스 특성에 따라 동작하지 않을 수 있다.  

---

## 2차 캐시  

### 1. 1차 캐시와 2차 캐시  
영속성 컨텍스트 내부에는 엔터티를 보관하는 장소가 있는데 이를 1차 캐시라고 한다.  
일반적인 웹 어플리케이션 환경은 트랜잭션을 시작하고 종료할 때까지만 1차 캐시가 유효하다. OSIV를 사용해도 클라이언트의 요청이 들어올 때부터 끝날 때까지만 1차 캐시가 유효하다.  
=> 어플리케이션 전체로 보면 데이터베이스 접근 횟수를 획기적으로 줄이지 못함  

하이버네이트를 포함한 대부분의 JPA 구현체들은 어플리케이션 범위의 캐시를 지원하는데 이를 공유 캐시 or 2차 캐시라고 한다.  
2차 캐시를 활용하면 어플리케이션 조회 성능을 향상시킬 수 있다.  

**1차 캐시**  
1차 캐시는 영속성 컨텍스트 내부에 있으며 엔터티 매니저로 조회하거나 변경하는 모든 엔터티는 1차 캐시에 저장된다.  
JPA를 스프링 프레임워크 같은 컨테이너 위에서 실행하면 트랜잭션을 시작할 때 영속성 컨텍스트를 생성하고 트랜잭션을 종료할 때 영속성 컨텍스트도 종료한다.  
OSIV를 사용하면 요청의 시작부터 끝까지 같은 영속성 컨텍스트를 유지한다.  
1. 최초 조회할 때는 1차 캐시에 엔터티가 없으므로  
2. 데이터베이스에서 엔터티를 조회해서  
3. 1차 캐시에 보관하고  
4. 1차 캐시에 보관한 결과를 반환한다.  
5. 이후 같은 엔터티를 조회하면 1차 캐시에 같은 엔터티가 있으므로 데이터베이스를 조회하지 않고 1차 캐시의 엔터티를 그대로 반환한다.  

- 1차 캐시는 같은 엔터티가 있으면 해당 엔터티를 그대로 반환하므로 1차 캐시는 객체 동일성을 보장한다.  
- 1차 캐시는 기본적으로 영속성 컨텍스트 범위의 캐시다.  

**2차 캐시**  
2차 캐시는 어플리케이션 범위의 캐시이므로 어플리케이션을 종료할 때까지 캐시가 유지되고, 분산 캐시나 클러스터링 환경의 캐시는 어플리케이션보다 더 오래 유지될 수도 있다.  
엔터티 매니저를 통해 데이터를 조회할 때 우선 2차 캐시에서 찾고 없으면 데이터베이스에서 찾으므로 데이터베이스 조회 횟수를 획기적으로 줄일 수 있다.  

1. 영속성 컨텍스트는 엔터티가 필요하면 2차 캐시를 조회한다.  
2. 2차 캐시에 엔터티가 없으면 데이터베이스를 조회해서  
3. 결과를 2차 캐시에 보관한다.  
4. 2차 캐시는 자신이 보관하고 있는 엔터티를 복사해서 반환한다.  
5. 2차 캐시에 저장되어 있는 엔터티를 조회하면 복사본을 만들어 반환한다.  

- 2차 캐시는 영속성 유닛 범위의 캐시다.  
- 2차 캐시는 조회한 객체를 그대로 반환하는 것이 아니라 복사본을 만들어 반환한다.  
- 2차 캐시는 데이터베이스 기본 키를 기준으로 캐시하지만 영속성 컨텍스트가 다르면 객체 동일성을 보장하지 않는다.  

### 2. JPA 2차 캐시 기능  
JPA 캐시 표준은 여러 구현체가 공통으로 사용하는 부분만 표준화해서 세밀한 설정을 하려면 구현체에 의존적인 기능을 사용해야 한다.  

**캐시 모드 설정**  
```java
@Cachealbe
@Entity
public class Member {
  @Id @GeneratedValue
  private Long id;
  ...
}
```
2차 캐시를 사용하려면 엔터티에 `javax.persistence.Cacheable` 어노테이션을 사용하면 된다.  
`@Cacheable`은 true, false 값을 설정할 수 있는데 기본값은 true다.  

다음으로 `persistence.xml`에 shard-cache-mode를 설정해서 어플리케이션 전체에 캐시를 어떻게 적용할지 옵션을 설정해야 한다.  

캐시 모드는 `javax.persistence.SharedCacheMode`에 정의되어 있고, 보통 ENABLE_SELECTIVE를 사용한다.  
|캐시 모드|설명|
|---|---|
|ALL|모든 엔터티를 캐시|
|NONE|캐시 사용 X|
|ENABLE_SELECTIVE|Cachebale(true)로 설정된 엔터티만 캐시 적용|
|DISABLE_SELECTIVE|모든 엔터티를 캐시하는데 Cachebale(false)로 설정된 엔터티는 캐시 X|
|UNSPECIFIED|JPA 구현체가 정의한 설정을 따름|

**캐시 조회, 저장 방식 설정**  
```java
em.setProperty("javax.persistence.cache.retrieveMode", CacheRetrieveMode.BYPASS);
```
캐시를 무시하고 데이터베이스를 직접 조회하거나 캐시를 갱신하려면 캐시 조회 모드와 캐시 보관 모드를 사용하면 된다.  
캐시 조회 모드나 보관 모드에 따라 사용할 프로퍼티와 옵션이 다르다.  
- `javax.persistence.cache.retrieveMode` : 캐시 조회 모드 프로퍼티 이름  
- `javax.persistence.cache.storeMode` : 캐시 보관 모드 프로퍼티 이름  
- `javax.persistence.CacheRetrieveMode` : 캐시 조회 모드 설정 옵션  
- `javax.persistence.CacheStoreMode` : 캐시 보관 모드 설정 옵션  

캐시 조회 모드  
```java
public enum CacheRetrieveMode {
  USE,
  BYPASS
}
```
- USE : 캐시에서 조회한다. (기본값)  
- BYPASS : 캐시를 무시하고 데이터베이스에 직접 접근한다.  

캐시 보관 모드  
```java
public enum CacheStoreMode {
  USE,
  BYPASS,
  REFRESH
}
```
- USE : 조회한 데이터를 캐시에 저장 / 조회한 데이터가 이미 캐시에 있으면 캐시 데이터를 최신 상태로 갱신 X / 트랜잭션을 커밋하면 등록 수정한 엔터티도 캐시에 저장 (기본값)  
- BYPASS : 캐시에 저장 X  
- REFRESH : USE 전략에 추가로 데이터베이스에서 조회한 엔터티를 최신 상태로 다시 캐시  

**JPA 캐시 관리 API**  
JPA는 캐시를 관리하기 위한 javax.persistence.Cache 인터페이스를 제공한다.  
```java
Cache cache = emf.getCache();
boolean contains = cache.contains(TestEntity.class, testEntity.getId());
System.out.println("contains = " + contains);
```

Cache 인터페이스의 자세한 기능은 다음과 같다.  
```java
public interface Cache {
  //해당 엔터티가 캐시에 있는지 여부 확인
  public boolean contains(Class cls, Object primaryKey);
  
  //해당 엔터티 중 특정 식별자를 가진 엔터티를 캐시에서 제거
  public void evict(Class cls, Object primaryKey);
  
  //해당 엔터티 전체를 캐시에서 제거
  public void evict(Class cls);
  
  //모든 캐시 데이터 제거
  public void evictAll();
  
  //JPA Cache 구현체 조회
  public <T> T unwrap(Class<T> cls);
}
```

### 3. 하이버네이트와 EHCACHE 적용  
하이버네이트가 지원하는 캐시는 크게 3가지가 있다.  
- 엔터티 캐시 : 엔터티 단위로 캐시 / 식별자로 엔터티를 조회하거나 컬렉션이 아닌 연관된 엔터티를 로딩할 때 사용  
- 컬렉션 캐시 : 엔터티와 연관된 컬렉션을 캐시 / 컬렉션이 엔터티를 담고 있으면 식별자 값만 캐시(하이버네이트 기능)  
- 쿼리 캐시 : 쿼리와 파라미터 정보를 키로 사용해서 캐시 / 결과가 엔터티면 식별자 값만 캐시(하이버네이트 기능)  
JPA 표준에는 엔터티 캐시만 정의되어 있다.  

**환경설정**  
하이버네이트에서 EHCACHE를 사용하려면 hibernate-ehcache 라이브러리를 `pom.xml`에 추가해야 한다.  
EHCACHE는 `ehcache.xml`을 설정 파일로 사용하는데 여기에 캐시를 얼만큼 보관할지, 얼마 동안 보관할지와 같은 캐시 정책을 정의한다. 클래스패스 루트인 `src/main/resources`에 위치  
다음으로 하이버네이트에 캐시 사용정보를 설정해야 하는데 다음과 같은 속성 정보를 `persistence.xml`에 추가하면 된다.  
- hibernate.cache.use_second_level_cache : 2차 캐시를 활성화 / 엔터티 캐시와 컬렉션 캐시 사용 가능  
- hibernate.cache.use_query_cache : 쿼리 캐시를 활성화  
- hibernate.cache.region.factory_class : 2차 캐시를 처리할 클래스 지정 / EHCACHE를 사용하면 `org.hibernate.cache.ehcache.EhCacheRegionFactory`를 적용  
- hibernate.generate_statistics : 이 속성을 true로 설정하면 하이버네이트가 여러 통계 정보를 출력해주고, 캐시 적용 여부를 확인할 수 있다.(성능에 영향을 주므로 개발 환경에서만 주로 사용)  

**엔터티 캐시와 컬렉션 캐시**  
- `javax.persistence.Cacheable` : 엔터티를 캐시하려면 이 어노테이션을 적용하면 된다.  
- `org.hibernate.annotations.Cache` : 이 어노테이션은 하이버네이트 전용으로 캐시와 관련된 세밀한 설정을 할 때 사용하고, 컬렉션 캐시를 적용할 때도 사용한다.  

**@Cache**  
하이버네이트 전용인 `org.hibernate.annotations.Cache` 어노테이션을 사용하면 세밀한 캐시 설정이 가능하다.  
|속성|설명|
|---|---|
|usage|CacheConcurrencyStrategy를 사용해서 캐시 동시성 전략을 설정|
|region|캐시 지역 설정|
|include|연관 객체를 캐시에 포함할지 선택 / all(기본값), non-lazy 옵션을 선택할 수 있음|

여기서 캐시 동시성 전략을 설정할 수 있는 CacheConcurrencyStrategy의 속성은 다음과 같다.  
|속성|설명|
|---|---|
|NONE|캐시 설정 X|
|READ_ONLY|읽기 전용으로 설정해서 등록, 삭제는 가능하지만 수정은 불가능 / 읽기 전용인 불변 객체는 수정되지 않으므로 하이버네이트는 2차 캐시를 조회할 때 객체를 복사하지 않고 원본 객체를 반환|
|NONSTRICT_READ_WRITE|엄격하지 않은 읽고 쓰기 전략으로 동시에 같은 엔터티를 수정하면 데이터 일관성이 깨질 수 있음 / EHCACHE는 데이터를 수정하면 캐시 데이터를 무효화|
|READ_WRITE|읽기 쓰기가 가능하고 READ COMMITTED 정도의 격리 수준을 보장 / EHCACHE는 데이터를 수정하면 캐시 데이터도 같이 수정|
|TRANSACTIONAL|컨테이너 관리 환경에서 사용 가능 / 설정에 따라 REPEATABLE READ 정도의 격리 수준을 보장|

캐시 종류에 따른 동시성 전략 지원 여부는 다음과 같다.  
|Cache|read-only|nonstrict-read-write|read-write|transactional|
|---|---|---|---|---|
|ConcurrentHashMap|yes|yes|yes||
|EHCache|yes|yes|yes|yes|
|Infinispan|yes|||yes|

**캐시 영역**  
캐시를 적용한 코드는 다음 캐시 영역(Cache Region)에 저장된다.  
- 엔터티 캐시 영역 : `jpabook.jpashop.domain.test.cache.ParentMember`  
- 컬렉션 캐시 영역 : `jpabook.jpashop.domain.test.cache.ParentMember.childMembers`  

엔터티 캐시 영역은 기본값으로 [패키지명 + 클래스명]을 사용하고, 컬렉션 캐시 영역은 엔터티 캐시 영역 이름에 캐시한 컬렉션의 필드명이 추가된다.  
캐시 영역을 위한 접두사를 설정하려면 `persistence.xml` 설정에 `hibernate.cache.region_prefix`를 사용하면 된다.  

**쿼리 캐시**  
쿼리 캐시는 쿼리와 파라미터 정보를 키로 사용해서 쿼리 결과를 캐시하는 방법이다.  
```java
em.createQuery("select i from Item i", Item.class)
    .setHint("org.hibernate.cacheable", true)
    .getResultList();
```
```java
@Entity
@NamedQury(
  hints = @QueryHint(name = "org.hibernate.cacheable", value = "true"), 
  name = "Member.findByUsername",
  query = "select m.address from Member m where m.name = :username"
)
public class Member {
  ...
```
쿼리 캐시를 적용하려면 영속성 유닛을 설정에 hibernate.cache.use_query_cache 옵션을 꼭 true로 설정해야 하고, 쿼리 캐시를 적용하려는 쿼리마다 org.hibernate.cacheable을 true로 설정하는 힌트를 주면 된다.  

**쿼리 캐시 영역**  
hibernate.cache.use_query_cache 옵션을 true로 설정해서 쿼리 캐시를 활성화하면 다음 두 캐시 영역이 추가된다.  
- `org.hibernate.cache.internal.StandardQueryCache` : 쿼리 캐시를 저장하는 영역으로 쿼리, 쿼리 결과 집합, 쿼리를 실행한 시점의 타임스탬프를 보관한다.  
- `org.hibernate.cache.spi.UpdateTimestampsCache` : 쿼리 캐시가 유효한지 확인하기 위해 쿼리 대상 테이블의 가장 최근 변경 시간을 저장하는 영역으로 테이블 명과 해당 테이블의 최근 변경된 타임스탬프를 보관한다.  

쿼리 캐시는 캐시한 데이터 집합을 최신 데이터로 유지하려고 쿼리 캐시를 실행하는 시간과 쿼리 캐시가 사용하는 테이블들이 가장 최근에 변경된 시간을 비교한다.  
쿼리 캐시를 적용하고 난 후에 쿼리 캐시가 사용하는 테이블에 조금이라도 변경이 있으면 데이터베이스에서 데이터를 읽어와서 쿼리 결과를 다시 캐시한다.  
쿼리 캐시를 잘 활용하면 극적인 성능 향상이 있지만 빈번하게 변경이 있는 테이블에 사용하면 오히려 성능이 더 저하되기 때문에 수정이 거의 일어나지 않는 테이블에 사용해야 효과를 볼 수 있다.  

**쿼리 캐시와 컬렉션 캐시의 주의점**  
엔터티 캐시를 사용해서 엔터티를 캐시하면 엔터티 정보를 모두 캐시하지만 쿼리 캐시와 컬렉션 캐시는 결과 집합의 식별자 값만 캐시한다.  
따라서 쿼리 캐시와 컬렉션 캐시를 조회하면 안에는 식별자 값만 들어있고, 이 식별자 값을 하나씩 엔터티 캐시에서 조회해서 실제 엔터티를 찾는데 대상 엔터티에 엔터티 캐시를 적용하지 않으면 성능상 심각한 문제가 발생할 수 있다.  
-> 쿼리 캐시나 컬렉션 캐시만 사용하고 엔터티 캐시를 사용하지 않으면 최악의 상황에 결과 집합 수만큼 SQL이 실행된다.  
=> 쿼리 캐시나 컬렉션 캐시를 사용하면 결과 대상 엔터티에는 꼭 엔터티 캐시를 적용해야 한다.  


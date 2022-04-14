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
  : 커밋하지 않은 데이터를 읽을 수 있음 / 트랜잭션1이 데이터를 수정하고 있는데 커밋하지 않아도 트랜잭션2가 수정 중인 데이터를 조회할 수 있는 것을 보고 DIRTY READ라고 함 / DIRTY READ를 허용하는 격리 수준  
- READ COMMITTED  
  : 커밋한 데이터만 읽을 수 있음 / 트랜잭션1이 데이터를 조회 중인데 트랜잭션2가 그 데이터를 수정하고 커밋하면 트랜잭션1이 다시 조회했을 때 수정된 데이터가 조회됨. 이런 반복해서 같은 데이터를 읽을 수 없는 상태를 NON-REPEATABLE READ라고 함 / DIRTY READ는 허용하지 않지만 NON-REPEATABLE READ는 허용하는 격리 수준  
- REPEATABLE READ  
  : 한 번 조회한 데이터를 반복해서 조회해도 같은 데이터가 조회됨 / 트랜잭션1이 10살 이하의 회원을 조회했는데 트랜잭션2가 5살 회원을 추가하고 커밋하면 트랜잭션1이 다시 10살 이하의 회원을 조회했을 때 회원 하나가 추가된 상태로 조회됨. 이런 반복 조회 시 결과 집합이 달라지는 것을 PHANTOM READ라고 함 / NON-REPEATABLE READ는 허용하지 않지만 PHANTOM READ는 허용하는 격리 수준  
- SERIALIZABLE  
  : 가장 엄격한 격리 수준 / 동시성 처리 성능이 급격히 떨어질 수 있음  

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
JPA가 제공하는 낙관적 락을 사용하려면 @Version 어노테이션을 사용해서 버전관리 기능을 추가해야 한다.  
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
버전 관리 기능을 적용하려면 엔터티에 버전 관리용 필드를 하나 추가하고 @Version을 붙이면 된다.  
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
@Version으로 추가한 버전 관리 필드는 JPA가 직접 관리하므로 개발자가 임의로 수정하면 안된다. (벌크 연산 제외)  
버전 값을 강제로 증가하려면 특별한 락 옵션을 선택해야 함  

### 4. JPA 락 사용  
락은 다음 위치에 적용할 수 있다.  
- EntityManager.lock(), EntityManager.find(), EntityManager.referesh()  
- Query.setLockMode() (TypeQuery 포함)  
- @NamedQuery  

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
JPA가 제공하는 락 옵션은 javax.persistence.LockModeType에 정의되어 있다.  

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
락 옵션 없이 @Version만 있어도 낙관적 락이 적용되지만, 락 옵션을 사용하면 락을 더 세밀하게 제어할 수 있다.  
**낙관적 락에서 발생하는 예외**  
- javax.persistence.OptimisticLockException(JPA 예외)  
- org.hibernate.StaleObjectStateException(하이버네이트 예외)  
- org.springframework.orm.ObjectOptimisticLockingFailureException(스프링 예외 추상화)  

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
- javax.persistence.PessimisticLockException(JPA 예외)  
- org.springframework.dao.PessimisticLockingFailureException(스프링 예외 추상화)  

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



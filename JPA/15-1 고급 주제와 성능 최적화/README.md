# 고급 주제와 성능 최적화

## 예외 처리

### JPA 표준 예외 정리

- JPA 표준 예외들은 `javax.persistenceException`의 자식 클래스
- 이 예외 클래스는 `RuntimeException`의 자식
    
    → JPA 예외는 모두 `언체크 예외`
    

- 트랜잭션 롤백을 표시하는 예외
    - 심각한 예외 이므로 복구해선 안됨
    - 트랜잭션을 강제로 커밋해도 트랜잭션이 커밋되지않고 대신에 RollbackException 예외가 발생
    
    | 트랜잭션 롤백을 표시하는 예외 | 설명 |
    | --- | --- |
    | EntityExistException | persist() 호출 시 이미 같은 엔티티가 있으면 발생 |
    | EntityNotFoundException | getReference() 를 호출했는데 실제 사용시 엔티티가 존재하지 않으면 발생 / refresh(),lock()에서도 발생 |
    | OptimisticLockException | 낙관적 락 충돌시 발생 |
    | PessimisticLockException | 비관적 락 충돌시 발생 |
    | RollbackException | commit() 실패 시 발생, 롤백이 표시되어 있는 트랜잭션 커밋 시에도 발생 |
    | TransactionRequiredException | 트랜잭션이 필요할 때 트랜잭션이 없으면 발생, 트랜잭션 없이 엔티티를 변경할 때 주로 발생 |
    - **낙관적 락 :** DB 충돌 상황을 개선할 수 있는 방법 중 2번째인 수정할 때 내가 먼저 이 값을 수정했다고 명시하여 다른 사람이 동일한 조건으로 값을 수정할 수 없게 하는 것
    - **비관적 락 :** 트랜잭션이 시작될 때 `Shared Lock` 또는 `Exclusive Lock`을 걸고 시작하는 방법
- 트랜잭션 롤백을 표시하지 않는 예외
    - 심각한 예외가 아님
    - 개발자가 트랜잭션을 커밋할지 롤백할지 판단
    
    | 트랜잭션 롤백을 표시하지 않는 예외 | 설명 |
    | --- | --- |
    | NoResultException | getSingleResult() 호출 시 결과가 하나도 없을 때 발생 |
    | NonUniqueResultException | getSingleResult() 호출 시 결과가 둘 이상일때 발생 |
    | LockTimeOutException | 비관적 락에서 시간 초과 시 발생 |
    | QueryTimeoutException | 쿼리 실행 시간 초과 시 발생 |

### 스프링 프레임워크의 JPA 예외 변환

- `서비스 계층`에서 `데이터 접근 계층`의 구현 기술에 직접 의존하는 것은 좋은 설계라 할 수 없음
- 서비스 계층에서 JPA의 예외를 직접 사용하면 JPA에 의존하게 됨
- 스프링 프레임워크는 데이터 접근 계층에 대한 예외를 추상화해서 `JPA 예외`를 `스프링 예외`로 변환할 수 있도록 해줌

### 스프링 프레임워크에 JPA 예외 변환기 적용

- JPA 예외를 스프링 프레임워크가 제공하는 추상화된 예외로 변경하려면 `PersistenceExceptionTranslationPostProcessor`를 스프링빈으로 등록하면 됨
- `@Repository` 를 사용한 곳에 예외변환 AOP를 적용해서 JPA 예외를 스프링 프레임워크가 추상화한 예외로 변환해 줌

### 트랜잭션 롤백 시 주의사항

- 트랜잭션을 롤백하는 것은 데이터베이스의 반영사항만 롤백하는 것이지 수정한 자바 객체까지 원상태로 복구해주지는 않음
    
    → 엔티티를 조회해서 수정하는 중에 문제가 있어서 트랜잭션을 롤백하면 `데이터베이스의 데이터`는 원래대로 복구되지만 `객체`는 수정된 상태로 영속성 컨텍스트에 남아 있음
    
- 트랜잭션이 롤백된 영속성 컨텍스트를 그대로 사용하는 것은 위험하기 때문에 `새로운 영속성 컨텍스트를 생성`해서 사용하거나 `EntityManager.clear()를 호출`해서 영속성 컨텍스트를 초기화한 다음에 사용

- 스프링 프레임워크의 영속성 컨텍스트의 범위에 따른 전략
    - 트랜잭션당 영속성 컨텍스트 전략 : 문제가 발생하면 트랜잭션 AOP 종료 시점에 트랜잭션을 롤백하면서 영속성 컨텍스트도 함께 종료되기 때문에 문제 X
    - OSIV(영속성 컨텍스트의 범위를 트랜잭션의 범위보다 넓게 사용해서 여러 트랜잭션이 하나의 영속성 컨택스트를 사용할 때) : 스프링 프레임워크는 영속성 컨텍스트의 범위를 트랜잭션의 범위보다 넓게 설정하면 트랜잭션 롤백시 영속성 컨택스트를 초기화(clear())해서 잘못된 영속성 컨텍스트를 사용하는 문제를 예방

## 엔티티 비교

- 영속성 컨텍스트 내부에는 엔터티 인스턴스를 보관하기 위한 1차 캐시가 있고, 이 1차 캐시는 영속성 컨텍스트와 생명주기를 같이함
- 영속성 컨텍스트를 통해 데이터를 저장하거나 조회하면 1차 캐시에 엔티티가 저장되고, 1차 캐시 덕분에 변경 감지 기능도 동작하고, 이름 그대로 1차캐시로 사용되어서 데이터베이스를 통하지 않고 데이터를 바로 조회할 수도 있음
- 영속성 컨텍스트를 더 정확히 이해하기 위해서는 1차 캐시의 가장 큰 장점인 `애플리케이션 수준의 반복 가능한 읽기`를 이해해야 함
- 같은 영속성 컨텍스트에서 엔티티를 조회하면 항상 같은 엔티티 인스턴스를 반환
    
    → 동등성(equals) 비교 수준이 아니라 정말 주솟값이 같은 인스턴스를 반환
    

### 영속성 컨텍스트가 같을 때 엔티티 비교

![image](https://user-images.githubusercontent.com/31034469/162448492-262ff4f1-f429-4913-a004-801adeea723a.png)

- 영속성 컨텍스트가 같으면 엔티티를 비교할 때 3가지 조건을 모두 만족
    - 동일성(identical) : == 비교가 같다
    - 동등성(equilnalent) : equals() 비교가 같다
    - 데이터베이스 동등성 : `@Id` 인 데이터베이스 식별자가 같다

### 영속성 컨텍스트가 다를 때 엔티티 비교

![image](https://user-images.githubusercontent.com/31034469/162448555-80227005-d36c-4198-a7d1-817d5c87177d.png)

- 영속성 컨텍스트가 다르면 동일성 비교에 실패
    - 동일성(identical) : == 비교가 실패한다
    - 동등성(equilnalent) : equals() 비교가 만족하지만, 단 equals()를 구현해야하고 보통 비즈니스 키로 구현
    - 데이터베이스 동등성 : `@Id` 인 데이터베이스 식별자가 같다

- 엔티티를 비교할 때는 **비즈니스 키를 활용한 동등성 비교를 권장**

## 프록시 심화 주제

- 프록시는 원본 엔티티를 상속받아서 만들어지므로 엔티티를 사용하는 `클라이언트`는 엔티티가 프록시인지 아니면 원본 엔티티인지 구분하지 않고 사용할 수 있음
- 원본 엔티티를 사용하다가 지연 로딩을 하려고 프록시로 변경해도 클라이언트의 비즈니스 로직을 수정하지 않아도 됨

### 영속성 컨텍스트와 프록시

- `엔티티`의 동등성을 비교하려면 비즈니스 키를 사용해서 `equals()` 메소드를 오버라이딩하고 비교하면 된다. 그런데 IDE나 외부 라이브러리를 사용해서 구현한 equals() 메소드로 엔티티를 비교할 때, 비교 대상이 원본 엔티티면 문제가 없지만 `프록시`면 문제가 발생할 수 있다
- 영속성 컨텍스트는 프록시로 조회된 엔티티에 대해서 같은 엔티티를 찾는 요청이 오면 원본 엔티티가 아닌 처음 조회된 프록시를 반환
- 프록시로 조회해도 영속성 컨텍스트는 영속 엔티티의 동일성을 보장한다

### 프록시 타입 비교

- 프록시는 원본 엔티티를 상속 받아서 만들어지므로 프록시로 조회한 엔티티의 타입을 비교할 때는 == 비교를 하면 안되고 대신에 `instanceof`를 사용
- 프록시는 원본 엔티티의 자식 타입이므로 `instanceof` 연산을 사용하면 된다

### 프록시 동등성 비교

- `equals()` 메소드로 엔티티를 비교할 때, 비교 대상이 원본 엔티티면 문제가 없지만 프록시면 문제가 발생할 수도 있음
    - equals() 메소드를 구현할 때는 일반적으로 멤버변수를 직접 비교하지만, 프록시는 실제 데이터를 가지고 있지 않다 → 프록시의 멤버변수에 직접 접근하면 아무값도 조회할 수 없다
- 프록시의 멤버변수에 직접 접근하면 안되고 대신에 접근자 메소드를 사용해야 한다

### 상속관계와 프록시

![image](https://user-images.githubusercontent.com/31034469/162448618-4f009396-95cd-47ad-9e6a-a26242780f65.png)

- 프록시를 부모 타입으로 조회하면 문제가 발생

```java
@Test
public void 부모타입으로_프록시조회() {
	//테스트 데이터 준비
	Book saveBook = new Book();
	saveBook.setName("jpaBook");
	saveBook.setAuthor("kim");
	em.persist(saveBook);

	em.flush();
	em.clear();

	//테스트 시작
	Item proxyItem = em.getReference(Item.class, saveBook.getId());
	System.out.println("proxyItem = " + proxyItem.getClass());

	if (proxyItem instanceof Book) {
		System.out.println("proxyItem instanceof Book");
		Book book = (Book) proxyItem;
		System.out.println("책 저자 = " + book.getAuthor());
	}

		//결과 검증
		Assert.assertFalse(proxyItem.getClass() == Book.class);
		Assert.assertFalse(proxyItem instanceof Book);
		Assert.assertTrue(proxyItem instanceof Item);
}
```

- 그런데 출력 결과를 보면 기대와는 다르게 저자가 출력되지 않은 것을 알 수 있다

![image](https://user-images.githubusercontent.com/31034469/162448661-5ee92574-3e03-4cfc-9730-030e2ca5b0e9.png)

- 프록시를 부모 타입으로 조회하면 부모의 타입을 기반으로 프록시가 생성되는 문제가 있음
- instanceof 연산을 사용할 수 없다
- 하위 타입으로 다운캐스팅을 할 수 없다

# 😭 의존관계 주입 `DI` 를 통한 테스트하기 쉬운 코드 만들기

## ❓ 왜 `DI` 가 필요한가?

- 객체에게 **의존관계**
    - 객체 혼자 모든 일을 처리하기 힘들기 때문에 내가 해야 할 작업을 다른 객체에게 위임하면서 발생
    - 객체 자신이 가지고 있는 책임과 역할을 다른 객체에게 위임하는 순간 발생
- DI 는 객체 간의 의존관계를 어떻게 해결하느냐에 따른 새로운 접근 방식

### Example

```java
import java.util.Calendar;

public class DateMessageProvider {
    public String getDateMessage() {
				Calendar now = Calendar.getInstance();
				int hour = now.get(Calendar.HOUR_OF_DAY);
				
				if(hour < 12)
						return "오전";
				return "오후";
		}
}
```

```java
public class DateMessageProviderTest {
		@Test
		public void 오전() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오전", provider.getDateMessage());
		}

		@Test
		public void 오후() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오후", provider.getDateMessage());
		}
}
```

- 테스트 메소드중 둘 중의 하나는 **반드시 실패**
- 테스트가 모두 성공할 수없는 이유는 `DateMessageProvider` 가 `Calendar` 와 의존관계를 가지는데 테스트를 위해 `Calendar`의 시간을 변경할 수 있는 방법이 없다
- 이와 같은 의존관계를 강하게 결합(tightly coupling)되어 있다고 한다
- 즉, `Calnder` 인스턴스에 대한 생성을 `getDateMessage` 가 결정하는 것이 아니라 `DateMessageProvider` 외부에서 `Calender` 인스턴스를 생성한 후 전달하는 구조로 바꿔야 한다

### getDateMessage() 메소드 인자로 전달하는 방법으로 리팩토링

```java
import java.util.Calendar;

public class DateMessageProvider {
    public String getDateMessage(Calendar now) {
				int hour = now.get(Calendar.HOUR_OF_DAY);
				
				if(hour < 12)
						return "오전";
				return "오후";
		}
}
```

```java
public class DateMessageProviderTest {
		@Test
		public void 오전() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오전", provider.getDateMessage(createCurrentDate(11)));
		}

		@Test
		public void 오후() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오후", provider.getDateMessage(createCurrentDate(13)));
		}

		private Calendar createCurrentDate(int hourOfDay) {
				Calendar now = Calendar.getInstatnce();
				now.set(Calendar.HOUR_OF_DAY, hourOfDay);
				return now;
		}
}
```

- 이처럼 객체 간의 의존관계에 대한 결정권을 `의존관계를 가지는 객체` 가 가지는 것이 아니라 `외부의 누군가`가 담당하도록 맡겨 버림으로써 좀 더 유연한 구조로 개발하는 것을 `DI` 라 한다
- 일반적으로 유연한 구조의 애플리케이션은 변화를 최소화하면서 확장하기도 쉽고, 테스트하기도 쉽다는것을 의미

## 🤬 `DI` 를 적용하면서 쌓이는 불편함

```java
public class QnaService {
    private static QnaService qnaService;

    private QuestionDao questionDao = QuestionDao.getInstance();
    private AnswerDao answerDao = AnswerDao.getInstance();

    private QnaService() {
    }

    public static QnaService getInstance() {
        if (qnaService == null) {
            qnaService = new QnaService();
        }
        return qnaService;
    }

		[...]
}
```

→ `DI` 구조로 변경

```java
public class QnaService {
    private static QnaService qnaService;

    private QuestionDao questionDao;
    private AnswerDao answerDao;

    private QnaService(QuestionDao questionDao, AnswerDao answerDao) {
        this.questionDao = questionDao;
        this.answerDao = answerDao;
    }

    public static QnaService getInstance(QuestionDao questionDao, AnswerDao answerDao) {
        if (qnaService == null) {
            qnaService = new QnaService(questionDao, answerDao);
        }
        return qnaService;
    }

		[...]
}
```

→ `DeleteQuestionController`, `ApiDeleteQuestionController` 에서 컴파일 에러가 발생해 수정

```java
public class DeleteQuestionController extends AbstractController {
    private QnaService qnaService = QnaService.getInstance(QuestionDao.getInstance(), AnswerDao.getInstance());
		[...]
}

public class ApiDeleteQuestionController extends AbstractController {
    private QnaService qnaService = QnaService.getInstance(QuestionDao.getInstance(), AnswerDao.getInstance());
		[...]
}
```

- `QuestionDao` 와 `AnswerDao` 가 데이터베이스와 의존관계
    
    → 의존관계를 가지지 않도록 변경할 수 있어야 데이터베이스에 의존하지 않는 테스트가 가능
    

- DI를 적용해 유연함을 얻을 수 있을지 모르겠지만 추가 작업할 부분이 너무 많다는 느낌
    
    → *DI 는 쓰레기인가?*
    

- DI 의 유연함을 가슴으로 느끼려면 같은 서비스를 일정 기간 유지보수 하면서 사용자의 요구사항이 바뀌고, 새로운 기능을 추가하는 경험을 해봐야 함

- 테스트하기 쉬운코드를 만들기 → DI 를 적용했을 때 가능

## ☺️ 불만 해소하기

1. 싱글톤 패턴을 사용함으로 인해 테스트에 어려움이 있다
2. 테스트를 위해 매번 `Mock` 객체를 만드는 것은 많은 비용이 들고 귀찮은 작업이다

### 싱글톤 패턴을 제거한 DI

- 싱글톤 패턴은 디자인 패턴 중 이해하기도 가장 쉽기 때문에 널리 사용됨
- 하지만 싱글톤 패턴을 사용할 경우 그에 따른 단점도 많음
    - 해당 클래스와 강한 의존관계를 가지기 때문에 테스트하기가 어려움
    - `private` 으로 구현하기 때문에 상속할 수 없다는 단점
    - 객체 지향 설계 원칙에 따라 개발하는 것을 저해하는 요인
    
💡 싱글톤 패턴을 사용하지 않으면서 인스턴스 하나만 유지할 수 있는 다른 해결책!  


컨트롤러는 싱글톤 패턴을 적용하지 않고도 같은 효과 

 → 서블릿 컨테이너가 `DispatcherServlet` 을 초기화 하는 시점에 컨트롤러 인스턴스를 생성한 후 재사용 가능하도록 구현했기 때문

### QnaService 의 QuestionDao 와 AnswerDao를 위의 방법으로 구현하고 테스트

- 싱글 톤 패턴을 위한 코드를 모두 제거하고 생성자를 `public` 으로 변경

```java
public class QnaService {
    private QuestionDao questionDao;
    private AnswerDao answerDao;

    public QnaService(QuestionDao questionDao, AnswerDao answerDao) {
        this.questionDao = questionDao;
        this.answerDao = answerDao;
    }
```

- `QnaService`와 의존관계에 있는 `DeleteQuestionController` 와 `ApiDeleteQuestionController`를 수정

```java
public class DeleteQuestionController extends AbstractController {
    private QnaService qnaService;

    public DeleteQuestionController(QnaService qnaService) {
        this.qnaService = qnaService;
    }
}

public class ApiDeleteQuestionController extends AbstractController {
    private QnaService qnaService;

    public ApiDeleteQuestionController(QnaService qnaService) {
        this.qnaService = qnaService;
    }
}
```

- 위 두 개의 컨트롤러를 생성할 때 QnaService를 DI 로 전달하도록 수정

```java
public class LegacyHandlerMapping implements HandlerMapping {
    private Map<String, Controller> mappings = new HashMap<>();
		
		public void initMapping() {
        QnaService qnaService = new QnaService(JdbcQuestionDao.getInstance(), JdbcAnswerDao.getInstance());
        mappings.put("/qna/delete", new DeleteQuestionController(qnaService));
        mappings.put("/api/qna/deleteQuestion", new ApiDeleteQuestionController(qnaService));
    }
}
```

- JdbcQuestionDao, JdbcAnswerDao 는 아직도 싱글톤 패턴을 사용하므로 적용하지 않도록 수정하고 다른 컨트롤러들에도 컴파일 에러가 발생하는 부분을 모두 찾아 DI로 의존관계를 가지도록 수정

```java
public class LegacyHandlerMapping implements HandlerMapping {
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);
    private Map<String, Controller> mappings = new HashMap<>();

    public void initMapping() {
        QuestionDao questionDao = new JdbcQuestionDao();
        AnswerDao answerDao = new JdbcAnswerDao();
        QnaService qnaService = new QnaService(questionDao, answerDao);

        mappings.put("/", new HomeController(questionDao));
        mappings.put("/qna/show", new ShowQuestionController(questionDao, answerDao));
        mappings.put("/qna/form", new CreateFormQuestionController());
        mappings.put("/qna/create", new CreateQuestionController(questionDao));
        mappings.put("/qna/updateForm", new UpdateFormQuestionController(questionDao));
        mappings.put("/qna/update", new UpdateQuestionController(questionDao));
        mappings.put("/qna/delete", new DeleteQuestionController(qnaService));
        mappings.put("/api/qna/deleteQuestion", new ApiDeleteQuestionController(qnaService));
        mappings.put("/api/qna/list", new ApiListQuestionController(questionDao));
        mappings.put("/api/qna/addAnswer", new AddAnswerController(questionDao, answerDao));
        mappings.put("/api/qna/deleteAnswer", new DeleteAnswerController(answerDao));

        logger.info("Initialized Request Mapping!");
    }
}
```

- MVC 프레임워크를 적용한 사용자 관리기능을 DI 구조로 변경
    
    → Ah! 자바 리플렉션을 활용해 인스턴스를 자동으로 생성하는 방식이다 보니 인스턴스를 생성할 때에 의존관계에 있는 인스턴스를 전달할 방법이 없다!
    
    → 다음 절!
    

### Mockito를 활용한 테스트

- 테스트 코드를 구현하는 것을 큰 부담으로 생각하는 개발자가 많은데 `Mock(가짜)` 클래스까지 구현해야 한다면 큰 부담 → 결과적으로 테스트를 하지 않게됨
- Mock 클래스를 구현하지 않아도 테스트가 가능하도록 지원하는 Mock 테스트 프레임 워크가 있다?!?!
- jMock, EasyMock, Mockito

- 메이븐 설정파일 pom.xml 에 Mockito 에 대한 의존관계를 추가

```xml
<dependency>
			<groupId>org.mockito</groupId>
			<artifactId>mockito-core</artifactId>
			<version>1.10.19</version>
			<scope>test</scope>
		</dependency>
```

- Mockito를 활용해 QnaService의 deleteQuestion() 메소드를 테스트

```java
@RunWith(MockitoJUnitRunner.class)
public class QnaServiceTest {
    @Mock
    private QuestionDao questionDao;
    @Mock
    private AnswerDao answerDao;

    private QnaService qnaService;

    @Before
    public void setup() {
        qnaService = new QnaService(questionDao, answerDao);
    }

    @Test(expected = CannotDeleteException.class)
    public void deleteQuestion_없는_질문() throws Exception {
        when(questionDao.findById(1L)).thenReturn(null);

        qnaService.deleteQuestion(1L, newUser("userId"));
    }

[...]

}
```

- Mokito는 `@Mock` 어노테이션으로 설정한 클래스의 메소드를 호출했을 때 반환값을 지정할 수 있음
- `@Mock` 으로 설정한 클래스의 메소드가 호출되는지의 여부를 verify() 메소드를 통해 검증하는 작업 또한 가능

### DI 보다 우선하는 객체지향 개발

- 현재 시간에 따라 오전/오후를 출력하는 정말 간단한 로직을 테스트하기 위한 코드

```java
public class DateMessageProviderTest {
		@Test
		public void 오전() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오전", provider.getDateMessage(createCurrentDate(11)));
		}

		@Test
		public void 오후() throws Exception {
				DateMessageProvider provider = new DateMessageProvider();
				assertEquals("오후", provider.getDateMessage(createCurrentDate(13)));
		}

		private Calendar createCurrentDate(int hourOfDay) {
				Calendar now = Calendar.getInstatnce();
				now.set(Calendar.HOUR_OF_DAY, hourOfDay);
				return now;
		}
}
```

<aside>
💡 좀 더 객체 지향적인 개발

</aside>

- `Hour` 클래스를 추가

```java
public class Hour {
		private int hour;
			
		public Hour(int hour) {
				this.hour = hour;
		}

		public String getMessage() {
				if (hour < 12) {
						return "오전";
				}
				return "오후";
		}
		
		@Override
		public boolean equals(Object obj) {
				if (this == obj)
						return true;
				if(obj == null)
						return false;
				if(getClass() != obj.getClass())
						return false;
				Hour other = (Hour)obj;
				if(hour != other.hour)
						return false;
				return true;
		}
}				
```

- 위에 대한 테스트

```java
public class DateMessageProviderTest {
		@Test
		public void 오전() throws Exception {
				Hour hour = new Hour(11)
				assertEquals("오전", hour.getMessage());
		}

		@Test
		public void 오후() throws Exception {
				Hour hour = new Hour(16)
				assertEquals("오후", hour.getMessage());
		}
}
```

<aside>
💡 로직 처리를 담당하는 새로운 객체를 추출함으로써 역할을 분리할 수 있으며, 테스트 또한 더 쉽게할 수 있다

</aside>

- “계층형 아키텍처 관점과 객체지향 설계 관점에서 핵심적인 비즈니스 로직을 구현해야 하는 역할은 누가 담당해야 할까?”
    - `Service`에서 처리해야하나? → 이는 서비스 레이어의 역할에 맞지 않다
    - 핵심적인 비즈니스 로직 구현은 도메인 객체가 담당하는 것이 맞다
    - 서비스 레이어의 핵심적인 역할은 도메인 객체들이 비지니스 로직을 구현할 수 있도록 도메인 객체를 조합하거나, 로직처리를 완료했을 때의 상태값을 DAO를 활용해 데이터베이스에 영구 저장하는 등의 역할을 담당해야 한다

- 절차지향적으로 개발하면 서비스 레이어의 복잡도는 점점 더 증가하면서 유지보수와 테스트하기 힘든 상황이 발생한다
- 핵심 객체라 할 수 있는 `도메인 객체`는 사용자가 입력한 데이터를 DAO 에 전달하거나 데이터베이스 데이터를 뷰에 전달하는 역할 밖에 하지 않게 된다
- 지금부터라도 `도메인 객체`에 더 많은 일을 시켜보자!

- `Question` 은 자신이 모든 로직을 처리하지 않고 인자로 전달된 User, Answer 와 협력해 삭제 가능 여부를 판단한다

- `User`, `Answer` 의 구현코드

```java
public class User {
		private String userId;

		[...]

		public boolean isSameUser(String newUserId) {
				return userId.equals(newUserId);
		}
}
```

```java
public class Answer {
		private String writer;
		
		[...]

		public boolean canDelete(User user) {
				return user.isSameUser(this.writer);
		}
}
```

- deleteQuestion() 메소드의 로직이 Question, Answer, User 로 분산된 결과 코드의 복잡도는 많이 낮아졌다

- deleteQuestion() 메소드를 수정

```java
public void deleteQuestion(long questionId, User user) throws CannotDeleteException {
        Question question = questionDao.findById(questionId);
        if (question == null) {
            throw new CannotDeleteException("존재하지 않는 질문입니다.");
        }

        List<Answer> answers = answerDao.findAllByQuestionId(questionId);
        if (question.canDelete(user, answers)) {
            questionDao.delete(questionId);
        }
    }
```

- 객체 지향으로 개발했을 때 얻을 수 있는 장점 중의 하나는 테스트 하기 쉽다는 것이다

- Question 의 canDelete() 메소드에 대한 테스트 코드

```java
public class QuestionTest {
    public static Question newQuestion(String writer) {
        return new Question(1L, writer, "title", "contents", new Date(), 0);
    }

    public static Question newQuestion(long questionId, String writer) {
        return new Question(questionId, writer, "title", "contents", new Date(), 0);
    }

    @Test(expected = CannotDeleteException.class)
    public void canDelete_글쓴이_다르다() throws Exception {
        User user = newUser("javajigi");
        Question question = newQuestion("sanjigi");
        question.canDelete(user, new ArrayList<Answer>());
    }

    @Test
    public void canDelete_글쓴이_같음_답변_없음() throws Exception {
        User user = newUser("javajigi");
        Question question = newQuestion("javajigi");
        assertTrue(question.canDelete(user, new ArrayList<Answer>()));
    }

    @Test
    public void canDelete_같은_사용자_답변() throws Exception {
        String userId = "javajigi";
        User user = newUser(userId);
        Question question = newQuestion(userId);
        List<Answer> answers = Arrays.asList(newAnswer(userId));
        assertTrue(question.canDelete(user, answers));
    }

    @Test(expected = CannotDeleteException.class)
    public void canDelete_다른_사용자_답변() throws Exception {
        String userId = "javajigi";
        List<Answer> answers = Arrays.asList(newAnswer(userId), newAnswer("sanjigi"));
        newQuestion(userId).canDelete(newUser(userId), answers);
    }
}
```

- 의존관계가 없기 때문에 굳이 Mockito와 같은 Mock 프레임워크를 사용할 필요도 없으며 테스트 구현또한 Mockito를 사용할 때 보다 간단하다
- 핵심로직을 도메인 객체에 구현함으로써 로직 구현의 복잡도를 낮추고, 테스트하기 쉬운 코드로 구현할 수있다
- 테스트가 습관화 되어있지 않다면 다른 부분은 하지않더라도 도메인 객체에 대한 테스트 만이라도 진행해야 한다!
- 가장 중요한 로직을 포함하고 있기 때문에 이부분에 대한 로직만이라도 테스트한다면 충분한 효과를 볼 수 있다
- 도메인 객체를 테스트하기 쉬운 코드를 만들려고 고민하다 보면 자연스럽게 좀 더 객체지향적인 코드를 구현할 수 있게 된다

- 새로 구현한 QnaServiceTest

```java
@RunWith(MockitoJUnitRunner.class)
public class QnaServiceTest {
    @Mock
    private QuestionDao questionDao;
    @Mock
    private AnswerDao answerDao;

    private QnaService qnaService;

    @Before
    public void setup() {
        qnaService = new QnaService(questionDao, answerDao);
    }

    @Test(expected = CannotDeleteException.class)
    public void deleteQuestion_없는_질문() throws Exception {
        when(questionDao.findById(1L)).thenReturn(null);

        qnaService.deleteQuestion(1L, newUser("userId"));
    }

    @Test
    public void deleteQuestion_삭제할수_있음() throws Exception {
        User user = newUser("userId");
        Question question = new Question(1L, user.getUserId(), "title", "contents", new Date(), 0) {
            public boolean canDelete(User user, List<Answer> answers) throws CannotDeleteException {
                return true;
            };
        };
        when(questionDao.findById(1L)).thenReturn(question);

        qnaService.deleteQuestion(1L, newUser("userId"));
        verify(questionDao).delete(question.getQuestionId());
    }

    @Test(expected = CannotDeleteException.class)
    public void deleteQuestion_삭제할수_없음() throws Exception {
        User user = newUser("userId");
        Question question = new Question(1L, user.getUserId(), "title", "contents", new Date(), 0) {
            public boolean canDelete(User user, List<Answer> answers) throws CannotDeleteException {
                throw new CannotDeleteException("삭제할 수 없음");
            };
        };
        when(questionDao.findById(1L)).thenReturn(question);

        qnaService.deleteQuestion(1L, newUser("userId"));
    }
}
```

- 관계형 데이터 베이스를 사용하면서 좀더 객체지향적인 개발이 가능하도록 하려면 ORM 프레임워크를 사용할 필요가 있다
- DI 를 적용하는 것도 중요하지만 먼저 객체지향적인 개발을 하도록 연습해야 한다

## 🧪 DI 프레임워크 실습

### 요구사항

- 각 클래스에 대한 인스턴스 생성 및 의존관계 설정을 어노테이션으로 자동화
    - 이미 추가되어있는 @Controller, 서비스는 @Service, DAO 는 @Repository
    - 3개의 설정으로 생성된 각 인스턴스 간의 의존관계는 @Injection 어노테이션을 사용. 
    
💡 `빈`이라는 용어가 등장하면 @Controller, @Service, @Repository 설정에 의해 DI 프레임워크가 자동으로 생성한 인스턴스로 정의. 


- 재귀로 구현한 BeanFactory 코드

```java
public class BeanFactory {
    private static final Logger logger = LoggerFactory.getLogger(BeanFactory.class);

    private Set<Class<?>> preInstanticateBeans;

    private Map<Class<?>, Object> beans = Maps.newHashMap();

    public BeanFactory(Set<Class<?>> preInstanticateBeans) {
        this.preInstanticateBeans = preInstanticateBeans;
    }

    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> requiredType) {
        return (T) beans.get(requiredType);
    }

    public void initialize() {
        for (Class<?> clazz : preInstanticateBeans) {
            if (beans.get(clazz) == null) {
                logger.debug("instantiated Class : {}", clazz);
                instantiateClass(clazz);
            }
        }
    }

    private Object instantiateClass(Class<?> clazz) {
        Object bean = beans.get(clazz);
        if (bean != null) {
            return bean;
        }

        Constructor<?> injectedConstructor = BeanFactoryUtils.getInjectedConstructor(clazz);
        if (injectedConstructor == null) {
            bean = BeanUtils.instantiate(clazz);
            beans.put(clazz, bean);
            return bean;
        }

        logger.debug("Constructor : {}", injectedConstructor);
        bean = instantiateConstructor(injectedConstructor);
        beans.put(clazz, bean);
        return bean;
    }

    private Object instantiateConstructor(Constructor<?> constructor) {
        Class<?>[] pTypes = constructor.getParameterTypes();
        List<Object> args = Lists.newArrayList();
        for (Class<?> clazz : pTypes) {
            Class<?> concreteClazz = BeanFactoryUtils.findConcreteClass(clazz, preInstanticateBeans);
            if (!preInstanticateBeans.contains(concreteClazz)) {
                throw new IllegalStateException(clazz + "는 Bean이 아니다.");
            }

            Object bean = beans.get(concreteClazz);
            if (bean == null) {
                bean = instantiateClass(concreteClazz);
            }
            args.add(bean);
        }
        return BeanUtils.instantiateClass(constructor, args.toArray());
    }
}
```

- @Controller, @Service, @Repository로 설정되어 있는 빈을 찾는 BeanScanner를 구현

```java
public class BeanScanner {
    private Reflections reflections;

    public BeanScanner(Object... basePackage) {
        reflections = new Reflections(basePackage);
    }

    @SuppressWarnings("unchecked")
    public Set<Class<?>> scan() {
        return getTypesAnnotatedWith(Controller.class, Service.class, Repository.class);
    }

    @SuppressWarnings("unchecked")
    private Set<Class<?>> getTypesAnnotatedWith(Class<? extends Annotation>... annotations) {
        Set<Class<?>> preInstantiatedBeans = Sets.newHashSet();
        for (Class<? extends Annotation> annotation : annotations) {
            preInstantiatedBeans.addAll(reflections.getTypesAnnotatedWith(annotation));
        }
        return preInstantiatedBeans;
    }
}
```

- BeanFactory 에 @Controller 로 설정한 빈을 조회할 수 있는 getControllers() 메소드를 추가

```java
public class BeanFactory {
    private static final Logger logger = LoggerFactory.getLogger(BeanFactory.class);

    private Set<Class<?>> preInstanticateBeans;

    private Map<Class<?>, Object> beans = Maps.newHashMap();

    [...]

		public Map<Class<?>, Object> getControllers() {
        Map<Class<?>, Object> controllers = Maps.newHashMap();
        for (Class<?> clazz : preInstanticateBeans) {
            if (clazz.isAnnotationPresent(Controller.class)) {
                controllers.put(clazz, beans.get(clazz));
            }
        }
        return controllers;
    }
}
```

- AnnotationHandlerMapping 에서의 리팩토링

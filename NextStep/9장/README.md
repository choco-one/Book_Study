# **🖲 자체 점검 요구사항**
💡 1. 로컬 개발 환경에 톰캣 서버를 시작하면 서블릿 컨테이너의 초기화 과정이 진행된다. 초기화되는 과정에 대해 설명하라. (프로젝트의 초기화 과정)
- 서블릿 컨테이너는 웹 애플리케이션의 상태를 관리하는 `ServletContext` 를 생성한다.
- `ServletContext` 가 초기화되면, 초기화 이벤트가 발생된다.
- 등록된 `ServletContextListener` 의 콜백 메소드가 호출된다.
    - 여기서는 `public class ContextLoaderListener implements ServletContextListener` 로 `ContextLoaderListener` 가 `ServletContextListener` 를 구현하고 있으므로, 해당 구현부의 `contextInitialized()` 가 호출된다.

```java
ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
populator.addScript(new ClassPathResource("jwp.sql"));
DatabasePopulatorUtils.execute(populator, ConnectionManager.getDataSource());
```

- `contextInitialized()` 메소드로, `jwp.sql` 파일을 실행해 DB 테이블을 초기화한다.
- 서블릿 컨테이너는 클라이언트의 요청을 받을 `DispatcherServlet` 인스턴스를 생성하는데, `loadOnStartup` 속성에 따라 생성 시점이 결정된다.
    - 여기서는 `loadOnStartup` 을 설정해서, 서블릿 컨테이너 시작 시점에 인스턴스를 생성한다.
    - 6장의 설명 참고
- `DispatcherServlet` 의 `init()` 메소드를 실행하여 `RequestMapping` 객체를 생성한다.
    - 생성한 `RequestMapping` 의 `initMapping()` 메소드를 호출해 요청 URL과 `Controller` 인스턴스를 매핑한다.

💡 2. 로컬 개발 환경에 톰캣 서버를 시작한 후 `http://localhost:8080` 으로 접근하면 질문 목록을 확인할 수 있다. URL로 접근해서 질문 목록이 보이기까지의 소스코드의 호출 순서 및 흐름을 설명하라.
- 요청을 처리하기 전, 먼저 `web.filter` 의 `ResourceFilter` 와 `CharacterEncodingFilter` 의 `doFilter()` 메소드가 실행된다. (이 둘은 `Filter` 를 구현)
- 필터를 거친 후, `urlPatterns="/"` 인 `DispatcherServlet` 으로 요청이 전달되어 `service()` 메소드가 호출된다.
    - 요청 URL에 대응하는 `Controller` 객체인 `HomeController` 를 반환한다.
    - 즉, `HomeController` 에게 작업을 위임하여 `HomeController` 의 `execute()` 메소드가 실행된다.
    - 해당 메소드는 `return jspView("home.jsp").addObject("questions", questionDao.findAll());` 를 반환하고, `DispatcherServlet` 는 이를 받아 `view.render(mav.getModel(), req, resp);` 한다. `JspView` 의 `render()` 는 전달된 모델 데이터를 `home.jsp` 에 전달해 HTML을 생성하고 응답한다.

💡 3. 질문 목록은 정상 동작하지만 질문하기 기능은 동작하지 않는다. 이를 구현한다. 질문 추가 로직은 `QuetionDao` 클래스의 `insert()` 메소드를 활용한다. 질문하기 성공 후 질문 목록 페이지(`"/"`)로 이동해야 한다.
- 먼저, 질문하기 버튼의 `href` 를 보면, `/qna/form` 으로 이동하도록 한다.
    - 따라서, `/qna/form` 요청을 처리할 컨트롤러를 생성하고, 이를 `RequestMapping` 에 등록한다.
    - 생성한 컨트롤러는 지금까지와 동일하게 `AbstractController` 를 상속하고, `execute()` 를 구현한다.
    - 이 컨트롤러에서는 `/qna/form.jsp` 를 반환한다.
- `form.jsp` 에서 글쓴이, 제목, 내용을 입력한 후, `/qna/create` 로 POST 요청을 보내도록 한다.
    - 이를 처리할 컨트롤러를 생성하고, 매핑한다.
    - 해당 컨트롤러에서 session으로부터 가져온 `userId` , 사용자 입력 값들(`title` , `contents`)을 사용하여 `Question` 객체를 생성한다.
    - 그리고 생성한 질문 객체를 `QuestionDao` 의 `insert()` 메소드를 통해 DB에 저장한다.
    - `QuestionDao` 의 `insert()` 메소드는 DB에 인자로 받은 질문 객체를 삽입하는 역할을 수행한다.

```java
public Question insert(Question question) {
    JdbcTemplate jdbcTemplate = new JdbcTemplate();
    String sql = "INSERT INTO QUESTIONS (writer, title, contents, createdDate) VALUES (?, ?, ?, ?)";
    PreparedStatementCreator psc = new PreparedStatementCreator() {
        @Override
        public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
            PreparedStatement pstmt = con.prepareStatement(sql);
            pstmt.setString(1, question.getWriter());
            pstmt.setString(2, question.getTitle());
            pstmt.setString(3, question.getContents());
            pstmt.setTimestamp(4, new Timestamp(question.getTimeFromCreateDate()));
            return pstmt;
        }
    };

    KeyHolder keyHolder = new KeyHolder();
    jdbcTemplate.update(psc, keyHolder);
    return findById(keyHolder.getId());
}
```

💡 4. 로그인하지 않은 사용자도 질문하기가 가능하다. 이를 로그인한 사용자만 질문이 가능하도록 수정한다. 이때 글쓴이를 입력하지 않고 로그인한 사용자 정보를 가져와 글쓴이 이름으로 사용하도록 구현한다.
- 질문하기 버튼 클릭 시, 먼저 session으로부터 사용자 정보를 가져와 로그인된 사용자인지 확인하고, 그렇지 않은 경우에는 `/users/loginForm` 으로 보낸다.
- `CreateQuestionController` 에서도 로그인 유무를 확인한 후 사용자가 입력한 글쓴이 정보가 아닌 session으로부터 가져온 `userId` 를 `Question` 객체에 담아 저장한다.

💡 5. 질문 목록에서 제목을 클릭하면 상세보기 화면으로 이동한다. 해당 화면에서 답변 목록이 정적인 HTML로 구현되어 있는데, 이를 DB에 저장되어 있는 답변을 출력하도록 구현한다. 스클립틀릿을 사용하지 않고 JSTL과 EL만으로 구현한다.
- 브랜치 바꿔서 보면 확인할 수 있다!

💡 6. 자바 기반 웹 프로그래밍을 할 경우 한글이 깨진다. 이를 `ServletFilter` 를 활용해 해결할 수 있다. `core.web.filter.CharacterEncodingFilter` 에 annotation 설정을 통해 한글 문제를 해결한다.
- `@WebFilter(value= {"/*"}, initParams=@WebInitParam(name="encoding", value="utf-8"))` annotation을 설정한다.
- `request.setCharacterEncoding(DEFAULT_ENCODING);` 과`response.setCharacterEncoding(DEFAULT_ENCODING);` 으로 request와 response에 대한 한글 인코딩 처리를 수행한다. (*`DEFAULT_ENCODING*= "UTF-8";`)
- `FilterChain` 객체를 사용하여 `chain.doFilter(request, response);` 메소드를 호출한다. 이는 필터작업(한글처리)의 내용이 서버상의 다음 컴포넌트에게 계속 적용되도록 하기 위해 필요하다.

💡 7. `next.web.qna` 패키지의 `ShowController` 는 멀티스레드 상황에서 문제가 발생할 가능성이 있다. 이를 문제가 발생하지 않도록 수정하고, 문제가 되는 이유를 설명하라.
- 클래스의 인스턴스를 생성하는 것과 인스턴스를 더 이상 사용하지 않는 경우 **Garbage Collection**을 통해 메모리에서 해제하는 과정 모두 **비용이 발생**한다.
    - 따라서, 인스턴스를 매번 생성할 필요가 없는 경우, 생성된 인스턴스를 재사용하는 것이 성능 측면에서 더 유리할 수 있다.
    - 이는 **인스턴스가 상태 값을 유지해야 하는지 여부**에 따라 구분된다.
    - 예로, `model` 의 `User` , `Question` , `Answer` 의 경우 클라이언트마다 서로 다른 상태 값을 가지기에, 매 요청마다 인스턴스를 생성하여 처리해야 한다.
    - 하지만, `JdbcTemplate` , 모든 `Dao` , `Controller` 는 매 요청마다 서로 다른 상태 값을 가지지 않아 하나의 인스턴스를 재사용할 수 있다.
- 서블릿은 서블릿 컨테이너 시작 시 인스턴스 하나를 생성해 재사용한다. `RequestMapping` 과 각 컨트롤러의 인스턴스 또한 모두 하나이다.
    - 하지만 서블릿 컨테이너는 **멀티스레드 환경**에서 동작한다. **여러 명의 사용자가 인스턴스 하나를 재사용**한다는 의미이므로, 잘못된 구현으로 인해 버그가 발생할 수 있다. (여러 명의 사용자가 동시에 같은 코드를 실행하는 경우 발생)

`ShowController` 는 인스턴스 하나를 여러 개의 스레드가 공유한다. 먼저 메모리에서 어떻게 동작하는지를 살펴본다.

- JVM은 코드를 실행하기 위해 메모리를 스택과 힙 영역으로 나눠서 관리한다.
    - 스택 영역은 각 메소드가 실행될 때, 메소드의 인자, 지역 변수 등을 관리하는 영역으로, 각 스레드마다 고유한 영역을 가진다.
    - 힙 영역은 클래스의 인스턴스 상태 데이터를 관리하는 영역으로, 스레드끼리 공유할 수 있다.

<img src="https://user-images.githubusercontent.com/33208303/158941284-62fcb070-c141-4e7f-bf51-0f6de0138806.png" width="80%">

- JVM은 각 메소드별로 스택 프레임을 생성한다.
- `ShowController` 의 `execute()` 메소드 실행 시, 해당 메소드에 대한 스택 프레임의 지역 변수 영역의 첫번째 위치에 자기 자신(`this`)에 대한 메모리 위치를 생성한다.
- `ShowController` 의 인스턴스는 힙에 생성되며, `Question` , `List<Answer>` 필드를 가지므로 힙에 생성된 `Question` 과 `List<Answer>` 인스턴스를 가리키는 구조로 실행된다.

위 구조에서, 2개의 스레드가 `execute()` 메소드를 실행하게 되면,

- 첫 번째 스레드가 접근했을 때는 문제가 되지 않는다.
- 하지만, 첫 번째 스레드의 작업이 완료되지 않은 시점에서 두 번째 스레드가 `execute()` 메소드를 실행하게 되면, 메모리 구조가 다음과 같이 변한다.

<img src="https://user-images.githubusercontent.com/33208303/158941293-e16abad0-ea28-42e2-88e3-51161d13bea4.png" width="80%">

- `ShowController` 가 가리키는 필드가 **1번에서 2번으로 변경**되었다.
- 두 번째 스레드는 정상적으로 질문과 답변을 응답으로 받지만, 첫 번째 스레드는 1번 글을 보기 위한 요청을 보냈음에도 두 번째 스레드가 실행한 `execute()` 로 인해 `ShowController` 가 가리키는 필드가 2번으로 변경되어 **2번 질문과 답변을 응답**으로 받게 된다.
- 첫 번째 스레드가 새로고침을 하게 되면 정상적인 응답을 받을 수 있어 버그가 고쳐지는 듯 하지만, 이러한 버그가 다른 중요한 기능에도 적용된다면, 큰 피해를 입을 수 있다.

> "멀티스레드에서 웹 애플리케이션 개발 시 구현하고 있는 코드가 어떻게 동작하는지를 이해하는 것은 매우 중요하다."

이러한 버그는 `Question` 과 `List<Answer>` 를 `ShowController` 의 필드가 아닌 `execute()` 메소드의 지역 변수로 구현함으로써 해결할 수 있다.

```java
public class ShowController extends AbstractController {
    private QuestionDao questionDao = new QuestionDao();
    private AnswerDao answerDao = new AnswerDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Long questionId = Long.parseLong(req.getParameter("questionId"));

        Question question = questionDao.findById(questionId);
        List<Answer> answers = answerDao.findAllByQuestionId(questionId);

        return jspView("/qna/show.jsp").addObject("question", question).addObject("answers", answers);
    }
}
```

<img src="https://user-images.githubusercontent.com/33208303/158941298-a5032e02-b27c-4842-bd97-9e45e9760b3e.png" width="80%">

- 이와 같이 구현하면, `ShowController` 가 `Question` 과 `List<Answer>` 인스턴스에 대한 참조를 가지지 않고 `ShowController` `execute()` 메소드의 **스택 프레임의 지역 변수 영역에서 해당 인스턴스에 대한 참조**를 가진다.
- 2개의 스레드가 접근하더라도, 서로 같은 `ShowController` 인스턴스를 참조하지만 힙 메모리에 생성된 `Question` 과 `List<Answer>` 는 서로 고유한 인스턴스를 참조하게 된다.

> 📌 `QuestionDao` 와 `AnswerDao` 는 모두 상태 값을 가지지 않기에 멀티스레드 환경에서도 문제가 되지 않는다.

💡 8. 상세보기 화면에서 답변하기 기능은 정상적으로 동작한다. 단, 답변 추가 시 전체 답변의 수가 증가하지 않는다. 답변을 추가하는 시점에 질문(`QUESTION` 테이블)의 답변 수(`countOfAnswer`)도 1 증가시킨다.
- 답변을 추가하는 시점에 답변 수를 증가시켜야 하므로, `AddAnswerController` 에서 `answerDao.insert()` 이후 로직이 추가되어야 한다.
- 먼저, `questionDao` 에 `questionId` 를 인자로 받아 질문에 대한 답변 수를 증가시키는 DB 로직을 구현한다.

```java
public void updateCountOfAnswer(long questionId) {
  JdbcTemplate jdbcTemplate = new JdbcTemplate();
  String sql = "UPDATE QUESTIONS SET countOfAnswer=countOfAnswer+1 "
          + "WHERE questionId = ?";

  jdbcTemplate.update(sql, questionId);
}
```

- 구현한 로직을 `AddAnswerController` 에서 사용하도록 한다.

```java
public class AddAnswerController extends AbstractController {
    ...

    private QuestionDao questionDao = new QuestionDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        ...

        Answer savedAnswer = answerDao.insert(answer);
        questionDao.updateCountOfAnswer(savedAnswer.getQuestionId());
        return jsonView().addObject("answer", savedAnswer);
    }
}
```

💡 9. 해당 서비스는 모바일에서도 서비스할 계획이라 API를 추가해야 한다. `/api/qna/list` URL로 접근 시 질문 목록을 JSON 데이터로 조회할 수 있도록 구현한다.
- 먼저, 해당 요청 URL을 처리할 컨트롤러를 생성하고, `RequestMapping` 에서 URL과 함께 매핑한다.
- 생성한 컨트롤러는 `QuestionDao` 의 `findAll()` 결과를 `jsonView()` 로 반환한다.

```java
public class ApiListQuestionController extends AbstractController {
    private QuestionDao questionDao = new QuestionDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        return jsonView().addObject("question", questionDao.findAll());
    }
}
```

💡 10. 상세보기 화면의 답변 목록에서 답변을 삭제해야 한다. 답변 삭제 또한 새로고침 없이 구현이 가능하도록 AJAX로 구현한다.
- `show.jsp` 에서 답변 삭제 버튼 클릭 시 요청을 보낼 URL을 명시하고, 이를 처리할 컨트롤러를 생성한다. (매핑도 수행)
- `webapp/js/scripts.js` 에서 삭제 버튼 클릭 시 발생시킬 이벤트를 명시하고, AJAX를 사용하여 서버에게 요청을 보낸다.

```java
$(".qna-comment").on("click", ".form-delete", deleteAnswer);

function deleteAnswer(e) {
  e.preventDefault();

  var deleteBtn = $(this);
  var queryString = deleteBtn.closest("form").serialize();

  $.ajax({
    type: 'post',
    url: "/api/qna/deleteAnswer",
    data: queryString,
    dataType: 'json',
    error: function (xhr, status) {
      alert("error");
    },
    success: function (json, status) {
      var result = json.result;
      if (result.status) {
        deleteBtn.closest('article').remove();
      }
    }
  });
}
```

- 컨트롤러에서는 요청으로부터 `answerId` 를 꺼내 DB에서 해당 답변을 삭제하고, 결과를 반환한다.
- 반환받은 결과에 따라, `error` or `success` 부분을 실행하여 답변을 삭제한다.

```java
public class DeleteAnswerController extends AbstractController {
    private AnswerDao answerDao = new AnswerDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Long answerId = Long.parseLong(req.getParameter("answerId"));
        try {
            answerDao.delete(answerId);
            return jsonView().addObject("result", Result.ok());
        } catch (DataAccessException e) {
            return jsonView().addObject("result", Result.fail(e.getMessage()));
        }
    }
}
```

> 📌 답변 삭제 시, `countOfAnswers` 를 -1 해주는 로직이 필요할 것 같다.

💡 11. 질문 수정이 가능해야 한다. 이는 글쓴이와 로그인 사용자가 같은 경우에만 가능해야 한다.
- 질문 수정 버튼 클릭 시, 요청을 보낼 URL을 명시하고, 질문 수정 양식으로 넘겨줄 컨트롤러 생성 후 `RequestMapping` 에서 매핑한다.
    - 이때, 해당 질문의 정보를 GET방식으로 URL에 명시하여 전달한다. (`question.questionId`)
- 컨트롤러에서는 글쓴이와 로그인 사용자가 동일한지 검사한다.
    - 이때, `Question` 의 `isSameUser()` 메소드를 구현해 사용하도록 한다.
    - 해당 메소드에는 session으로부터 얻은 로그인 사용자 정보를 인자로 넘겨주고, 이를 받은 메소드는 `writer id`를 가지고, 사용자가 동일한지 검사한다.
- 동일한 경우에만 질문 수정 컨트롤러로 넘겨준다.

```java
public class UpdateFormQuestionController extends AbstractController {
    private QuestionDao questionDao = new QuestionDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        if (!UserSessionUtils.isLogined(req.getSession())) {
            return jspView("redirect:/users/loginForm");
        }

        Long questionId = Long.parseLong(req.getParameter("questionId"));
        Question question = questionDao.findById(questionId);

        if (!question.isSameUser(UserSessionUtils.getUserFromSession(req.getSession()))) {
            throw new IllegalStateException("본인이 작성한 글만 수정할 수 있습니다.");
        }

        return jspView("/qna/update.jsp").addObject("question", question);
    }
}
```

- `/qna/update.jsp` 는 질문의 제목과 내용을 수정할 수 있는 `form` 이 있고, `/qna/update` 로 POST 요청을 보낸다.
- 이를 처리할 컨트롤러 생성 및 매핑을 수행한다.
- 컨트롤러에서는 `questionId` , `title` , `contents` 를 받아 질문 DB에 UPDATE한다.
    - 먼저, `Question` 의 `update()` 메소드로 기존 질문 객체의 정보를 변경하고, `QuestionDao` 의 `update()` 메소드로 `QUESTION` DB에 있는 해당 질문을 UPDATE한다.

```java
public class UpdateQuestionController extends AbstractController {
    private QuestionDao questionDao = new QuestionDao();

    @Override
    public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        if (!UserSessionUtils.isLogined(req.getSession())) {
            return jspView("redirect:/users/loginForm");
        }

        long questionId = Long.parseLong(req.getParameter("questionId"));
        Question question = questionDao.findById(questionId);
        if (!question.isSameUser(UserSessionUtils.getUserFromSession(req.getSession()))) {
            throw new IllegalStateException("다른 사용자가 쓴 글을 수정할 수 없습니다.");
        }

        Question newQuestion = new Question(question.getWriter(), req.getParameter("title"),
                req.getParameter("contents"));
        question.update(newQuestion);
        questionDao.update(question);
        return jspView("redirect:/");
    }
}
```

> "서버측에서 안전한 애플리케이션이 되도록 먼저 구현한 후 클라이언트는 추후에 고려해도 괜찮다. 그만큼 서버측에서 보안에 대한 처리를 중요하게 해야 한다.”

💡 12. 컨트롤러에서 접근하는 `QuestionDao` 와 `AnswerDao` 와 같이 DAO에서 DB 접근 로직을 구현할 때 사용하는 `JdbcTemplate` 은 인스턴스를 여러 개 생성할 필요없다. 인스턴스를 하나만 생성하도록 구현한다.
- 위에서 언급했듯, `Dao` , `Controller` , `JdbcTemplate` 은 매번 인스턴스를 생성하지 않아도 된다. **상태 값을 가지지 않으면서 메소드만 가지는 클래스**이기 때문이다.
- 따라서 인스턴스 하나만 생성해 재사용하도록 해야하는데, 이러한 디자인 패턴을 **싱글톤(Singleton) 패턴**이라고 한다.
    - 구현을 위해서는 먼저 클래스의 기본 생성자를 `private` 접근 제어자로 구현하여 클래스 외부에서 인스턴스를 생성할 수 없도록 한다. (`getInstance()` 와 같은 `static` 메소드를 통해서만 생성이 가능하도록 한다.)

```java
public class JdbcTemplate {
    private static JdbcTemplate jdbcTemplate;
    
    private JdbcTemplate() {}
    
    public static JdbcTemplate getInstance() {
        if (jdbcTemplate == null) {
            jdbcTemplate = new JdbcTemplate();
        }
        return jdbcTemplate;
    }
    ...

}
```

- 위와 같이 싱글톤 패턴을 구현하는 경우, 멀티스레드 환경에서 동시에 `getInstance()` 메소드를 호출해 인스턴스가 하나 이상 생성되는 문제가 발생할 수 있다.
    - 문제가 있지만, 뒤에서 다룰 **의존성 주입(DI)**을 설명하기 위해 위와 같이 구현했다.

> **싱글톤 패턴의 장단점**
> 장점
> - 고정된 메모리 영역을 얻으면서, 인스턴스를 한 번만 생성하기에 메모리 낭비 방지
> - 인스턴스가 전역으로 생성되어, 다른 클래스에서의 접근 및 공유가 용이
> - 이미 생성되어 있는 인스턴스를 사용하여 객체 로딩 시간의 감소
> 
> 단점
> - 싱글톤 인스턴스가 너무 많은 작업을 처리하거나, 많은 데이터를 공유하는 경우 다른 클래스의 인스턴스들 간의 결합도가 너무 높아져 “개방 폐쇄 원칙”을 위배
> - 이로 인해, 수정 및 유지보수의 비용이 증가하고, 멀티스레드 환경에서 동기화 처리가 없다면 인스턴스가 둘 이상 생성될 수 있는 가능성이 존재

```java
JdbcTemplate jdbcTemplate = JdbcTemplate.getInstance();
```

- `JdbcTemplate` 을 사용하는 `Dao` 에서 위와 같이 `JdbcTemplate.getInstance()` 메소드로 인스턴스를 받아 사용한다.

💡 13. 질문 삭제 기능을 구현한다. 삭제가 가능한 경우는 답변이 없는 경우, 질문자와 답변자가 모두 같은 경우이다. 이 기능은 웹과 모바일 앱에서 모두 지원하려고 한다. 웹 브라우저를 통해 접근했을 때는 `JspView` 를 활용해 목록 페이지(`"redirect:/"`)로 이동하고, 모바일 앱을 지원하기 위해 `JsonView` 를 활용해 응답 결과를 JSON으로 전송하려고 한다. 이를 지원하려면 두 개의 컨트롤러가 필요하다. 각 컨트롤러를 구현하고, 중복을 제거한다.
- 먼저, 삭제 버튼 클릭 시 요청을 보낼 URL을 명시하고, 이를 처리할 컨트롤러 생성 및 매핑을 수행한다.
    - 또한, `question.questionId` 를 요청과 함께 전달하여 이를 사용할 수 있도록 한다.

```java
public class DeleteQuestionController extends AbstractController {
  private QnaService qnaService = QnaService.getInstance();

  @Override
  public ModelAndView execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
    if (!UserSessionUtils.isLogined(req.getSession())) {
      return jspView("redirect:/users/loginForm");
    }

    long questionId = Long.parseLong(req.getParameter("questionId"));
    try {
      qnaService.deleteQuestion(questionId, UserSessionUtils.getUserFromSession(req.getSession()));
      return jspView("redirect:/");
    } catch (CannotDeleteException e) {
      return jspView("show.jsp").addObject("question", qnaService.findById(questionId))
              .addObject("answers", qnaService.findAllByQuestionId(questionId))
              .addObject("errorMessage", e.getMessage());
    }
  }
}
```

```java
public class QnaService {
  private static QnaService qnaService;

  private QuestionDao questionDao = new QuestionDao();
  private AnswerDao answerDao = new AnswerDao();

	private QnaService() {}

  public static QnaService getInstance() {
    if (qnaService == null) {
      qnaService = new QnaService();
    }
    return qnaService;
  }

  public Question findById(long questionId) {
    return questionDao.findById(questionId);
  }

  public List<Answer> findAllByQuestionId(long questionId) {
    return answerDao.findAllByQuestionId(questionId);
  }

  public void deleteQuestion(long questionId, User user) throws CannotDeleteException {
    Question question = questionDao.findById(questionId);
    if (question == null) {
      throw new CannotDeleteException("존재하지 않는 질문입니다.");
    }

    if (!question.isSameUser(user)) {
      throw new CannotDeleteException("다른 사용자가 쓴 글을 삭제할 수 없습니다.");
    }

    List<Answer> answers = answerDao.findAllByQuestionId(questionId);
    if (answers.isEmpty()) {
      questionDao.delete(questionId);
      return;
    }

    boolean canDelete = true;
    for (Answer answer : answers) {
      String writer = question.getWriter();
      if (!writer.equals(answer.getWriter())) {
          canDelete = false;
          break;
      }
    }

    if (!canDelete) {
      throw new CannotDeleteException("다른 사용자가 추가한 댓글이 존재해 삭제할 수 없습니다.");
    }

    questionDao.delete(questionId);
  }
}
```

- `QnaService` 서비스를 생성하여 질문과 답변에 대한 처리를 수행한다.
- 여기서는, `deleteQuestion` 에서
    - 인자로 받은 `questionId` 에 해당하는 질문을 찾고,
    - 해당 질문의 작성자와 인자로 받은 `user` 가 동일한지 검사한 후
    - 해당 질문의 답변이 있는지 없는지를 판단하여 없는 경우에만 질문을 삭제하고 반환된다.
    - 그렇지 않은 경우, 질문에 대한 답변 작성자가 사용자 본인인지 아닌지 판단하여 `canDelete` 값이 `false` 가 되면, 추가한 `CannotDeleteException` 예외를 던지도록 한다.
- 추가적으로, 모바일 지원을 위해 `ApiDeleteQuestionController` 를 생성하고, 매핑한다.
    - 이는 `jspView()` 가 아닌 `jsonView()` 를 반환하도록 한다.

💡 14. 13번 문제를 구현할 때 단위 테스트가 가능하도록 한다. Dao를 사용하는 모든 컨트롤러 클래스는 DB가 설치되어 있어야 하며, 테이블까지 생성되어 있는 상태에서만 테스트가 가능하다. DB가 존재하지 않는 상태에서도 위 로직을 단위 테스트하려고 한다.
- 13번 문제는 질문 삭제 기능을 웹과 모바일 모두에 제공하기 위해 2개의 `Controller` 구현 시 발생하는 중복을 어떻게 제거할 것인가에 대한 문제이다.
    - 이 요구사항을 만족하도록 `DeleteQuestionController` , `ApiDeleteQuestionController` 클래스를 구현했다.
    - 원래 구현대로라면, `questionId` 를 가져와 질문을 먼저 조회하고, 작성자와 수정하려는 사용자가 동일한지 검사하고, 해당 질문에 대한 답변을 조회한 뒤, 답변의 작성자까지 다 확인해야하는 과정 모두 동일하고, 응답할 뷰(`JspView` , `JsonView`)만 다르다.
    - 이에 대한 중복을 제거해야 한다.

방법은 2가지가 있다.

- 이 두 클래스에 대한 부모 클래스를 추가해 중복 로직을 부모 클래스로 이동한 후 상속을 통해 중복을 제거한다.
- 컨트롤러가 DAO와 의존관계를 통해 DB 접근 로직을 제거했듯 새로운 클래스를 추가해 로직 처리를 위임하여 중복을 제거한다. → **"조합"(composition)**
    - 상속의 경우 **부모 클래스에 변경이 발생하면 자식 클래스에도 영향**이 생기기 때문에 상속보다는 조합을 통해 중복을 제거할 것을 추천한다.

자바 진영에서는 이러한 중복 제거와 컨트롤러 역할 분리 등의 목적으로 **Service**(또는 Manager)라는 클래스를 추가해 담당하도록 구현한다. 따라서 위 문제에서는 `QnaService` 라는 클래스를 추가한다.

```java
public class QnaService {
  private static QnaService qnaService;

  private QuestionDao questionDao = new QuestionDao();
  private AnswerDao answerDao = new AnswerDao();

  private QnaService() {}

  public static QnaService getInstance() {
    if (qnaService == null) {
      qnaService = new QnaService();
    }
    return qnaService;
  }

  public Question findById(long questionId) {
    return questionDao.findById(questionId);
  }

  public List<Answer> findAllByQuestionId(long questionId) {
    return answerDao.findAllByQuestionId(questionId);
  }

  public void deleteQuestion(long questionId, User user) throws CannotDeleteException {
    Question question = questionDao.findById(questionId);
    if (question == null) {
      throw new CannotDeleteException("존재하지 않는 질문입니다.");
    }

    if (!question.isSameUser(user)) {
      throw new CannotDeleteException("다른 사용자가 쓴 글을 삭제할 수 없습니다.");
    }

    List<Answer> answers = answerDao.findAllByQuestionId(questionId);
    if (answers.isEmpty()) {
      questionDao.delete(questionId);
      return;
    }

    boolean canDelete = true;
    for (Answer answer : answers) {
      String writer = question.getWriter();
      if (!writer.equals(answer.getWriter())) {
          canDelete = false;
          break;
      }
    }

    if (!canDelete) {
      throw new CannotDeleteException("다른 사용자가 추가한 댓글이 존재해 삭제할 수 없습니다.");
    }

    questionDao.delete(questionId);
  }
}
```

- 이 클래스 또한 상태 값을 가지지 않아 **싱글톤 패턴**으로 구현한다.
- `deleteQuestion()` 가 `DeleteQuestionController` 와 `ApiDeleteQuestionController` 클래스에서의 중복 코드를 구현한 로직이다.
    - 이렇게 컨트롤러에서의 복잡하고 중복된 로직을 서비스로 위임했기에, 컨트롤러에서는 정상적으로 삭제가 되는 경우와 에러가 발생하는 경우에 따른 처리만 구현할 수 있게 된다.
    - 이때, `CannotDeleteException` 이라는 컴파일타임 Exception을 `throw` 한다.
        - 이와 같이 사용자에게 예외처리를 통해 에러 메세지를 전달하거나, 다른 작업을 하도록 유도할 필요가 있는 경우 런타임 Exception보다 컴파일타임 Exception이 적합하다.

> **계층형 아키텍처**
> 컨트롤러, 서비스, DAO 구조로 웹 애플리케이션을 개발하는 것
> <img src="https://user-images.githubusercontent.com/33208303/158941304-9bc2d1cb-4178-41fa-b0e6-911bedbb6ccd.png" width="50%">
> 
> 질문 삭제를 담당하는 클래스들을 **계층형 아키텍처**로 나타내면 다음과 같다.
> 
> <img src="https://user-images.githubusercontent.com/33208303/158941301-a52cd31f-7951-4e7a-91c1-37b5a2b198b1.png" width="50%">
> 
> 최근에는 DB 접근 로직을 담당하는 클래스 이름을 DAO 대신 Repository로 구현하는 경우가 많아지고 있다. Domain 객체에 해당하는 클래스는 Question & Answer이다.

💡 15. `RequestMapping` 코드를 보면, 컨트롤러 추가마다 요청 URL과 컨트롤러를 추가해야 하는 불편함이 있다. 서블릿과 같이 annotation을 활용해 설정을 추가하고 서버가 시작할 때 자동으로 매핑이 되도록 개선한다.
- `@Controller` 설정을 통한 새로운 MVC는 다음 장에서 구현한다.
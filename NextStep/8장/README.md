# 8️⃣  AJAX를 활용해 새로고침 없이 데이터 갱신하기

## AJAX 활용해 답변 추가, 삭제 실습

- `브라우저`가 `서버`에서 HTML 응답을 받아 처리하는 과정
    - HTML 응답을 받으면 브라우저는 먼저 HTML을 라인 단위로 읽어내려가면서 서버에 재요청이 필요한 부분(CSS, javascript, Image 등)을 찾아 서버에 다시 요청을 보냄
    - 서버에서 자원을 다운로드 하면서 HTML DOM 트리를 구성
    - 서버에서 CSS 파일을 다운로드하면 앞에서 생성한 HTML DOM 트리에 CSS 스타일을 적용한 후 모니터 화면에 그리게 됨

### 답변 추가하기

- 사용자가 답변하기 버튼을 클릭하는 단계에서 시작
- 사용자가 입력한 데이터를 서버로 전송
- 서버는 사용자가 입력한 데이터를 데이터베이스에 저장
- 저장한 데이터를 클라이언트에 JSON 형태로 전송
- 클라이언트는 서버가 응답한 JSON 데이터를 HTML로 변환해 사용자의 화면에 출력

`scripts.js`

- 답변하기 버튼(".answerWrite input[type=submit]")에 클릭 이벤트가 발생하면 addAnswer() 함수를 실행
- 성공하면 onSuccess()함수를 호출하면서 응답 데이터를 전달받음
- 실패하면 onError() 함수를 호출하면서 실패 원인을 전달받는 방식으로 동작한다

```jsx
$(".answerWrite input[type=submit]").click(addAnswer);

function addAnswer(e) {
    e.preventDefault(); // submit이 자동으로 동작하는 것을 막는다
    // form 데이터들을 자동으로 묶어준다
    var queryString = $("form[name=answer]").serialize();

    $.ajax({
        type : 'post',
        url : '/api/qna/addAnswer',
        data : queryString,
        dataType : 'json',
        error: onError,
        success : onSuccess,
    });
}
```

`AddAnswerController`

- 클라이언트의 요청을 처리하는 서버 코드
- 응답을 할 때 `HTML` 이 아닌 `JSON` 형태로 데이터만 전달

```java
public class AddAnswerController implements Controller {
    private static final Logger log = LoggerFactory.getLogger(AddAnswerController.class);

    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Answer answer = new Answer(req.getParameter("writer"), req.getParameter("contents"), Long.parseLong(req.getParameter("questionId")));
        log.debug("answer : {}", answer);

        AnswerDao answerDao = new AnswerDao();
        Answer savedAnswer = answerDao.insert(answer);
        ObjectMapper mapper = new ObjectMapper();

        resp.setContentType("application/json;charset=UTF-8");

        PrintWriter out  = resp.getWriter();
        out.print(mapper.writeValueAsString(savedAnswer));
        return null;
    }
}
```

### 답변 삭제하기

- 모든 삭제 링크에 대해 클릭 이벤트 발생시 함수를 호출하도록 구현
    
    `$(".qna-comment").on("click", ".form-delete", deleteAnswer);`
    
- 클라이언트는 서버 응답 status 값이 true 일 경우 HTML 에서 해당 답변의 HTML 을 삭제

`scripts.js`

```jsx
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
      if (json.status) {
        deleteBtn.closest('article').remove();
      }
    }
  });
}
```

`DeleteAnswerController`

- 클라이언트의 요청을 처리하는 서버 코드
- 응답을 할 때 `HTML` 이 아닌 `JSON` 형태로 데이터만 전달

```java
public class DeleteAnswerController implements Controller {
    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Long answerId = Long.parseLong(req.getParameter("answerId"));
        AnswerDao answerDao = new AnswerDao();

        answerDao.delete(answerId);

        ObjectMapper mapper = new ObjectMapper();
        resp.setContentType("application/json;charset=UTF-8");
        PrintWriter out = resp.getWriter();
        out.print(mapper.writeValueAsString(Result.ok()));
        return null;
    }
}
```

## MVC 프레임워크 요구사항 2단계

```java
public class DeleteAnswerController implements Controller {
    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        Long answerId = Long.parseLong(req.getParameter("answerId"));
        AnswerDao answerDao = new AnswerDao();

        answerDao.delete(answerId);

        ObjectMapper mapper = new ObjectMapper();
        resp.setContentType("application/json;charset=UTF-8");
        PrintWriter out = resp.getWriter();
        out.print(mapper.writeValueAsString(Result.ok()));
        return null;
    }
}
```

- 위 코드의 문제점 2 가지
    - `JSON` 으로 응답을 보내는 경우 이동할 `JSP` 페이지가 없다보니 불필요하게 null 을 반환
        
        → 컨트롤러에서 응답할 뷰가 `JSP` 하나에서 `JSP` 와 `JSON` 두가지로 증가했기 때문
        
    - 자바 객체를 `JSON` 으로 변환하고 응답하는 부분에서 중복이 발생한다
        
        → 중복 코드를 별도의 메소드로 분리한 후 `Abstract JsonController` 와 같은 부모 클래스를 만들어 중복을 해결할 수 있음
        

### 요구사항 분리

- View를 추상화한 인터페이스 추가
    
    ```java
    public interface View {
        void render(HttpServletRequest request, HttpServletResponse response) throws Exception;
    }
    ```
    

- View를 구현하는 JspView와 JsonView를 생성해 각 View 성격에 맞도록 구현
    
    ```java
    public class JspView implements View {
        private static final String DEFAULT_REDIRECT_PREFIX = "redirect:";
    
        private String viewName;
    
        public JspView(String viewName) {
            if (viewName == null) {
                throw new NullPointerException("viewName is null. 이동할 URL을 추가해 주세요.");
            }
            this.viewName = viewName;
        }
    
        @Override
        public void render(HttpServletRequest req, HttpServletResponse resp)
                throws Exception {
            if (viewName.startsWith(DEFAULT_REDIRECT_PREFIX)) {
                resp.sendRedirect(viewName.substring(DEFAULT_REDIRECT_PREFIX.length()));
                return;
            }
    
            RequestDispatcher rd = req.getRequestDispatcher(viewName);
            rd.forward(req, resp);
        }
    }
    ```
    
    ```java
    public class JsonView implements View {
        @Override
        public void render(HttpServletRequest req, HttpServletResponse resp) throws Exception {
            ObjectMapper mapper = new ObjectMapper();
            resp.setContentType("application/json;charset=UTF-8");
            PrintWriter out = resp.getWriter();
            out.print(mapper.writeValueAsString(createModel(req)));
        }
        
        private Map<String, Object> createModel(HttpServletRequest req) {
            Enumeration<String> names = req.getAttributeNames();
            Map<String, Object> model = new HashMap<>();
    
            while (names.hasMoreElements()) {
                String name = names.nextElement();
                model.put(name, req.getAttribute(name));
            }
            
            return model;
        }
    }
    ```
    

- Controller 인터페이스의 반환값을 String 에서 View 로 변경
    
    ```java
    public interface Controller {
        View execute(HttpServletRequest request, HttpServletResponse response) throws Exception;
    }
    ```
    

- 각 Controller에서 String 대신 JspView 또는 JsonView 중 하나를 사용하도록 변경
    
    ```java
    public class HomeController extends AbstractController {
        
        @Override
        public View execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
            QuestionDao questionDao = new QuestionDao();
            req.setAttribute("questions", questionDao.findAll());
            return new JspView("home.jsp");
        }
    }
    ```
    

- DispatcherServlet에서 String 대신 View 인터페이스를 사용하도록 수정
    
    ```java
    @WebServlet(name = "dispatcher", urlPatterns = "/", loadOnStartup = 1)
    public class DispatcherServlet extends HttpServlet {
        private static final long serialVersionUID = 1L;
        private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);
    
        private RequestMapping rm;
    
        @Override
        public void init() throws ServletException {
            rm = new RequestMapping();
            rm.initMapping();
        }
    
        @Override
        protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            String requestUri = req.getRequestURI();
            logger.debug("Method : {}, Request URI : {}", req.getMethod(), requestUri);
    
            Controller controller = rm.findController(req.getRequestURI());
            try {
                View view = controller.execute(req, resp);
                view.render(req, resp);
            } catch (Throwable e) {
                logger.error("Exception : {}", e);
                throw new ServletException(e.getMessage());
            }
        }
    }
    ```
    

- 모델 데이터에 대한 추상화 작업
    
    ```java
    public class ModelAndView {
        private View view;
        private Map<String, Object> model = new HashMap<>();
    
        public ModelAndView(View view) {
            this.view = view;
        }
    
        public ModelAndView addObject(String attributeName, Object attributeValue) {
            model.put(attributeName, attributeValue);
            return this;
        }
    
        public Map<String, Object> getModel() {
            return Collections.unmodifiableMap(model);
        }
    
        public View getView() {
            return view;
        }
    }
    ```
    

- 경우의 수가 하나가 아닌 2개 이상이 발생하는 경우 if/else를 통해 해결할 수 있지만 그보다는 `인터페이스`를 통해 문제를 해결하는 것이 좀 더 확장 가능하고 깔끔한 코드를 구현할 수 있음

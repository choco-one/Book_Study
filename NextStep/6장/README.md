# **🧔 서블릿/JSP로 회원관리 기능 다시 개발하기**
## **👲 서블릿/JSP 복습**
### **회원가입**

`/user/form.html` 을 그대로 사용한다. 사용자 입력 데이터를 추출한 후 DB에 추가하는 방식이다.

```java
@WebServlet("/user/create")
public class CreateUserServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger log = LoggerFactory.getLogger(CreateUserServlet.class);

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        User user = new User(req.getParameter("userId"), req.getParameter("password"), req.getParameter("name"),
                req.getParameter("email"));
        log.debug("user : {}", user);
        DataBase.addUser(user);
        resp.sendRedirect("/user/list");
    }
}
```

- 회원가입을 완료한 후 사용자 목록 출력을 위해 `"/user/list"` 로 리다이렉트한다.

### **사용자 목록**

회원가입할 때 저장한 사용자 목록을 조회한 후 JSP에 `"users"` 라는 이름으로 전달한다. 이전 장에서는 `ListUserController` 에서 동적으로 `StringBuilder`를 이용하여 HTML을 생성했지만 여기서는 JSP파일로 위임한다.

```java
@WebServlet("/user/list")
public class ListUserServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        req.setAttribute("users", DataBase.findAll());
        RequestDispatcher rd = req.getRequestDispatcher("/user/list.jsp");
        rd.forward(req, resp);
    }
}
```

- 서블릿도 동적으로 HTML을 생성하기 위해서는 5장의 `ListUserController` 와 같은 방식으로 구현해야 한다.
- 이와 같은 서블릿의 한계를 극복하기 위해 JSP가 등장했다.

### **JSP**

정적인 HTML은 그대로 두고, 동적으로 변경되는 부분만을 구현한다. **Java Server Page**라는 이름답게 자바 구문을 그대로 사용할 수 있다. (**스크립틀릿(scriptlet)**이라고 하는 `<% %>` 내에 사용)

하지만 웹 애플리케이션 요구사항의 복잡도가 증가하면서 **많은 로직**이 구현되다보니 **JSP의 유지보수가 어려워지는 문제**가 발생했다.

이를 해결하기 위해 **JSTL(JavaServer Pages Standard Tag Library)과 EL(Expression Language)가 등장**했고, JSP의 복잡도를 낮추기 위해 **MVC 패턴을 적용한 프레임워크가 등장**했다.

```jsx
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core"%>

...

<c:forEach items="${users}" var="user" varStatus="status">
    <tr>
        <th scope="row">${status.count}</th>
        <td>${user.userId}</td>
        <td>${user.name}</td>
        <td>${user.email}</td>
        <td><a href="#" class="btn btn-success" role="button">수정</a>
        </td>
    </tr>
</c:forEach>

...
```

- **JSTL과 EL**을 활용하면 JSP에서 자바 구문을 완전히 제거할 수 있다.
    - 사실 이를 위해서는 JSP가 출력할 데이터(ex. `users`)를 전달해줄 **컨트롤러가 필요**하다. 즉, **MVC 패턴 기반으로 개발**해야 자바 구문을 완전히 제거할 수 있다는 것이다.
    - 여기서는 사용자 목록 조회 후 JSP에 전달했던 `ListUserServlet` 이 MVC 패턴에서 컨트롤러 역할을 수행한다.

## **🤶 개인정보수정 실습**

개인정보수정 기능에 대한 구현을 시작한다. 수정 화면은 회원가입 화면인 `/user/form.html` 을 재사용한다.

`list.jsp`

```html
<li><a href="../user/updateForm.jsp" role="button">개인정보수정</a></li>
```

- 먼저, 개인정보수정 버튼 클릭 시, `../user/updateForm.jsp` 로 이동하도록 지정

`updateForm.jsp`

```html
<div class="container" id="main">
    <div class="col-md-6 col-md-offset-3">
        <div class="panel panel-default content-main">
            <form name="question" method="post" action="/user/update">
                <input type="hidden" name="userId" value="${user.userId}" />
                <div class="form-group">
                    <label>사용자 아이디</label>
                    ${user.userId}
                </div>
                <div class="form-group">
                    <label for="password">비밀번호</label>
                    <input type="password" class="form-control" id="password" name="password" placeholder="Password">
                </div>
                <div class="form-group">
                    <label for="name">이름</label>
                    <input class="form-control" id="name" name="name" placeholder="Name" value="${user.name}">
                </div>
                <div class="form-group">
                    <label for="email">이메일</label>
                    <input type="email" class="form-control" id="email" name="email" placeholder="Email" value="${user.email}">
                </div>
                <button type="submit" class="btn btn-success clearfix pull-right">개인정보수정</button>
                <div class="clearfix" />
            </form>
        </div>
    </div>
</div>
```

- `/user/update` 로 POST 요청을 보내는 `form` 태그 생성

`UpdateUserServlet.java`

```java
@WebServlet(value = {"/user/update", "/user/updateForm"})
public class UpdateUserServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger log = LoggerFactory.getLogger(CreateUserServlet.class);

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        RequestDispatcher rd = req.getRequestDispatcher("/user/updateForm.jsp");
        rd.forward(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        User user = DataBase.findUserById(req.getParameter("userId"));

        User updateUser = new User(req.getParameter("userId"), req.getParameter("password"), req.getParameter("name"),
                req.getParameter("email"));
        log.debug("updateUser : {}", updateUser);
        user.update(updateUser);
        resp.sendRedirect("/user/list");
    }
}
```

- `/user/updateForm` 로의 GET 요청과, `/user/update` 로의 POST 요청을 처리하기 위해 `value` 사용 (The URL patterns of the servlet)
- `userId` 를 이용해 DB에서 사용자를 조회하고 입력받은 값으로 `user.update` 수행

> 💡 왜 GET과 POST를 받는 URL을 다르게 하는지?

`model/User.java`

```java
public void update(User updateUser) {
    this.password = updateUser.password;
    this.name = updateUser.name;
    this.email = updateUser.email;
}
```

- `userId` 를 제외한 사용자 정보 update를 위한 메소드

## **👳‍♀️ 로그인/로그아웃 기능 실습**

현재 상태가 로그인 상태이면 상단 메뉴가 **로그아웃**, **개인정보수정**이 나타나야 하며, 로그아웃 상태이면 **로그인**, **회원가입**이 나타나야 한다.

`LoginController.java`

```java
@WebServlet(value = { "/users/login", "/users/loginForm" })
public class LoginController extends HttpServlet {
    private static final long serialVersionUID = 1L;

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        forward("/user/login.jsp", req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String userId = req.getParameter("userId");
        String password = req.getParameter("password");
        User user = DataBase.findUserById(userId);

        if (user == null) {
            req.setAttribute("loginFailed", true);
            forward("/user/login.jsp", req, resp);
            return;
        }

        if (user.matchPassword(password)) {
            HttpSession session = req.getSession();
            session.setAttribute(UserSessionUtils.USER_SESSION_KEY, user);
            resp.sendRedirect("/");
        } else {
            req.setAttribute("loginFailed", true);
            forward("/user/login.jsp", req, resp);
        }
    }

    private void forward(String forwardUrl, HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        RequestDispatcher rd = req.getRequestDispatcher(forwardUrl);
        rd.forward(req, resp);
    }
}
```

- `/user/login`, `/user/loginForm` 에 대한 응답을 처리한다.
- **GET 요청**의 경우, `/user/login.jsp` 를 반환한다.
- **POST 요청**의 경우, 입력된 id를 이용해 DB에 존재하는 사용자인지 조회하고, 비밀번호까지 일치하면 `HttpSession` 으로 session에 사용자를 추가한다.
    - `USER_SESSION_KEY` : 저장하는 `user` 를 식별하기 위한 키
    - 나중에 암호화를 하게 된다면, 
    ID를 먼저 조회해서 DB에 저장된 PWD를 가져오고, 
    사용자 입력 값을 암호화한 값과의 비교를 수행

`include/~.jspf`

```java
<c:if test="${not empty sessionScope.user}">
    <ul class="nav dropdown-menu">
        <li><a href="/users/profile?userId=${sessionScope.user.userId}"><i class="glyphicon glyphicon-user" style="color:#1111dd;"></i> Profile</a></li>
        <li class="nav-divider"></li>
        <li><a href="#"><i class="glyphicon glyphicon-cog" style="color:#dd1111;"></i> Settings</a></li>
    </ul>
</c:if>

<c:choose>
<c:when test="${not empty sessionScope.user}">
<li><a href="/users/logout" role="button">로그아웃</a></li>
<li><a href="/users/updateForm?userId=${sessionScope.user.userId}" role="button">개인정보수정</a></li>
</c:when>
<c:otherwise>
<li><a href="/users/loginForm" role="button">로그인</a></li>
<li><a href="/users/form" role="button">회원가입</a></li>
</c:otherwise>
</c:choose>
```

- **JSP**에서 session에 로그인한 사용자 여부를 체크하고 **JSTL**에서 분기문 처리하여 로그인한 사용자와 그렇지 않은 사용자가 사용할 수 있는 기능을 다르게 처리한다.

`LogoutController.java`

```java
@WebServlet("/users/logout")
public class LogoutController extends HttpServlet {
    private static final long serialVersionUID = 1L;

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        HttpSession session = req.getSession();
        session.removeAttribute(UserSessionUtils.USER_SESSION_KEY);
        resp.sendRedirect("/");
    }
}
```

- `HttpSession` 을 날려버리는 것으로 로그아웃을 구현한다.

## **💂‍♀️ 회원 목록 및 개인정보 수정 보안 강화 실습**

**로그인한 사용자만이 사용자 목록 조회**가 가능해야 하며, **개인정보 수정은 자신의 정보만** 수정 가능해야 한다.

`ListUserController.java`

```java
if (!UserSessionUtils.isLogined(req.getSession())) {
    resp.sendRedirect("/users/loginForm");
    return;
}
```

- session에 로그인한 사용자가 있는지 확인한다.
    - 로그인한 사용자가 없는 경우, `loginForm`으로 리다이렉션한다.

`UpdateUserController.java`

```java
if (!UserSessionUtils.isSameUser(req.getSession(), user)) {
    throw new IllegalStateException("다른 사용자의 정보를 수정할 수 없습니다.");
}
```

- 정보 수정 시, 수정하려는 사용자가 현재 로그인한 본인인지 확인한다.

`UserSessionUtils.java`

```java
public class UserSessionUtils {
    public static final String USER_SESSION_KEY = "user";

    public static User getUserFromSession(HttpSession session) {
        Object user = session.getAttribute(USER_SESSION_KEY);
        if (user == null) {
            return null;
        }
        return (User) user;
    }

    public static boolean isLogined(HttpSession session) {
        if (getUserFromSession(session) == null) {
            return false;
        }
        return true;
    }

    public static boolean isSameUser(HttpSession session, User user) {
        if (!isLogined(session)) {
            return false;
        }

        if (user == null) {
            return false;
        }

        return user.isSameUser(getUserFromSession(session));
    }
}
```

- `getUserFromSession` : session을 인자로 받아 session으로부터 `user` attribute를 추출한다.
- `isLogined` : `getUserFromSession` 메소드를 호출하여 그 반환값으로 로그인한 사용자가 있는지 확인한다.
- `isSameUser` : `isLogined` 메소드 호출로 session에 로그인한 사용자가 있는지와 인자로 받은 `user` 의 `null` 여부를 확인하여 `boolean` 값을 반환한다. ??? (왜 인자가 1개만 넘겨주지)

---

# **👨‍🎓 세션(`HttpSession`) 요구사항 및 실습**

이전 장에서 언급했듯 **HTTP는 무상태 프로토콜**이다. 하지만 로그인과 같이 상태를 유지할 필요가 있는 요구사항이 발생한다. 이와 같은 경우 사용할 수 있는 방법이 "쿠키 헤더를 사용하는 것"이다.

`"Set-Cookie"` 헤더를 통해 쿠키를 생성하면, 이후 발생하는 모든 요청에 `"Set-Cookie"` 로 추가한 값을 "Cookie" 헤더로 전달하는 방식이다.

하지만 쿠키를 사용하는 데 **보안상 취약하다**는 문제점이 있다.

왜냐면, 누구나 브라우저 개발자 도구나 HTTP 분석 도구를 활용하여 HTTP 요청 및 응답 헤더를 확인할 수 있기 때문이다. 따라서, **쿠키를 통해 중요한 개인정보를 전달하는 것은 보안상 적합하지 않다.**

**Session의 등장**

쿠키의 보안상 단점을 보완하기 위해 **Session**이 등장했다. 이는 상태 값으로 유지하고 싶은 정보를 클라이언트단에 저장하는 것이 아닌 **서버단에 저장**한다.

- 서버에 저장 후, 각 클라이언트마다 고유한 아이디를 발급해 이 아이디를 `"Set-Cookie"` 헤더를 통해 전달한다.
    - HTTP에서 상태를 유지하는 방법은 **쿠키**밖에 없기에, 결국은 상태 값 유지를 위한 **값 전달**에는 쿠키를 사용한다.

> "세션은 HTTP의 쿠키를 기반으로 동작한다."

## **👧 요구사항**

서블릿에서 지원하는 `HttpSession` API의 일부를 지원해야 한다. 구현할 메소드는 `getId()`, `setAttribute(String name, Object value)`, `getAttribute(String name)`, `removeAttribute(String name)`, `invalidate()` 이다.

- `String getId()` : 현재 session에 할당되어 있는 고유한 session id를 반환
- `void setAttribute(String name, Object value)` : 현재 session에 `value` 인자로 전달되는 객체를 `name` 인자 이름으로 저장
- `Object getAttribute(String name)` : 현재 session에 `name` 인자로 저장되어 있는 객체 값을 찾아 반환
- `void removeAttribute(String name)` : 현재 session에 `name` 인자로 저장되어 있는 객체 값을 삭제
- `void invalidate()` : 현재 session에 저장되어 있는 모든 값을 삭제

## **🧒 요구사항 분리 및 힌트**

- 클라이언트와 서버 간 주고 받을 고유한 아이디를 생성해야 한다. 고유한 아이디는 쉽게 예측할 수 없어야 한다. 그렇지 않다면, 쿠키 값을 조작해 다른 사용자처럼 속일 수 있기 때문이다.
- 생성한 고유한 아이디를 쿠키를 통해 전달한다. (`”Set-Cookie”` 헤더)
- 서버 측에서 모든 클라이언트의 session 값을 관리하는 저장소 클래스를 추가한다. (`Map<String, String>`, **Key**는 앞에서 생성한 고유한 아이디)
- 클라이언트별 session 데이터를 관리할 수 있는 클래스(`HttpSession`)를 추가한다.
    - 해당 클래스는 위 5개의 메소드를 모두 구현하고, 상태 데이터를 저장할 `Map<String, Object>` 가 필요하다.

---

# **👩‍💻 세션(`HttpSession`) 구현**

## **🤴 고유한 아이디 생성**

session에서 사용할 고유한 아이디를 생성해 보자. 랜덤으로 임의의 값을 생성할 수 있지만, JDK에서 제공하는 `UUID` 클래스를 활용한다. 먼저 `UUIDTest` 클래스를 추가해 어떤 형태로 생성되는지 확인해본다.

`UUID.randomUUID()` 를 출력해보면 `5736e178-a06e-422b-84be-2309a3741eff` 와 같은 형태로 값이 생성된다.

## **👶 쿠키를 활용해 아이디 전달**

클라이언트가 처음 접근하는 경우, 해당 클라이언트가 사용할 **session 아이디를 생성**하고 이를 **쿠키로 전달**한다. 이후 요청부터는 상태 값 공유를 위해 전달해준 session 아이디를 사용한다.

`RequestHandler` 클래스에 session 아이디가 존재하는지 여부를 판단한 후 없는 경우 session 아이디를 새로 발급한다. session 아이디는 `JSESSIONID` 로 전달한다.

```java
public class RequestHandler extends Thread {
    ...

    public void run() {
        log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
                connection.getPort());

        try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
            HttpRequest request = new HttpRequest(in);
            HttpResponse response = new HttpResponse(out);

            if (getSessionId(request.getHeader("Cookie")) == null) {
                response.addHeader("Set-Cookie", "JSESSIONID=" + UUID.randomUUID());
            }
            
            ...

        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private String getSessionId(String cookieValue) {
        Map<String, String> cookies = HttpRequestUtils.parseCookies(cookieValue);
        return cookies.get("JSESSIONID");
    }
    ...

}
```

- `getSessionId` 메소드는 request 헤더의 `Cookie` 값을 파싱하여 `JSESSIONID` 값을 반환한다.
    - 있으면, `JSESSIONID=5736e178-a06e-422b-84be-2309a3741eff` 같은 형태
    - session 아이디가 없는 경우, 새로 발급한다.

`ListUserController` 에서 로그인 여부 확인을 위해 `Cookie` 헤더 값을 활용하는 부분이 증가하므로 이를 관리하는 `HttpCookie` 클래스를 추가하는 리팩토링을 실시한다.

```java
public class HttpCookie {
    private Map<String, String> cookies;

    HttpCookie(String cookieValue) {
        cookies = HttpRequestUtils.parseCookies(cookieValue);
    }

    public String getCookie(String name) {
        return cookies.get(name);
    }
}
```

```java
public class HttpRequest {
    ...

    public HttpCookie getCookies() {
			return new HttpCookie(getHeader("Cookie"));
		}
}
```

- 이후 `HttpRequest` 에서 `HttpCookie` 에 접근할 수 있는 메소드를 추가한다.

이제 `RequestHandler` 클래스는 새로 추가한 `HttpCookie` 클래스의 메소드를 사용하도록 변경할 수 있다.

```java
if (request.getCookies().getCookie("JSESSIONID") == null) {
    response.addHeader("Set-Cookie", "JSESSIONID=" + UUID.randomUUID());
}
```

## **👨‍🦱 모든 클라이언트의 세션 데이터에 대한 저장소 추가**

서버는 **다수의 클라이언트 session을 지원**해야 한다. 이를 위해서는 session을 관리할 수 있는 **저장소**가 필요하다.

- 모든 session을 매번 생성하는 것이 아니라 **한 번 생성 후 재사용**해야 한다. (`static` 으로 `Map` 생성해 (id, session)을 쌍으로 저장)

```java
public class HttpSessions {
    private static Map<String, HttpSession> sessions = new HashMap<>();
    
    public static HttpSession getSession(String id) {
        HttpSession session = sessions.get(id);
        
        if (session == null) {
            session = new HttpSession(id);
            sessions.put(id, session);
            return session;
        }
        
        return session;
    }
    
    static void remove (String id) {
        sessions.remove(id);
    }
}
```

## **👵 클라이언트별 세션 저장소 추가**

서블릿에서 session 데이터에 접근할 때 사용한 클래스인, 각 클라이언트별 session을 담당할 `HttpSession` 클래스를 추가한다.

```java
public class HttpSession {
    private Map<String, Object> values = new HashMap<>();
    
    private String id;
    
    public HttpSession(String id) {
        this.id = id;
    }
    
    public String getId() {
        return id;
    }
    
    public void setAttribute(String name, Object value) {
        values.put(name, value);
    }
    
    public Object getAttribute(String name) {
        return values.get(name);
    }
    
    public void removeAttribute(String name) {
        values.remove(name);
    }
    
    public void invalidate() {
        HttpSessions.remove(id);
    }
}
```

- `HttpRequest` 에서 클라이언트에 해당하는 `HttpSession` 에 접근할 수 있도록 메소드를 추가한다.

```java
public HttpSession getSession() {
    return HttpSessions.getSession(getCookies().getCookie("JSESSIONID"));
}
```

session 추가를 완료했으므로, `logined=true` 와 같은 쿠키 값을 추가하는 것이 아닌 `User` 객체를 추가해 로그인 여부를 판단하도록 코드를 변경하자.

```java
public class LoginController extends AbstractController {
    @Override
    public void doPost(HttpRequest request, HttpResponse response) {
        User user = DataBase.findUserById(request.getParameter("userId"));
        if (user != null) {
            if (user.login(request.getParameter("password"))) {
                HttpSession session = request.getSession();
                session.setAttribute("user", user);
                response.sendRedirect("/index.html");
            } else {
                response.sendRedirect("/user/login_failed.html");
            }
        } else {
            response.sendRedirect("/user/login_failed.html");
        }
    }
}
```

session에 `User` 객체를 추가했으므로, `ListUserController` 에서 로그인 여부 판단 코드로 변경해줘야 한다.

```java
public class ListUserController extends AbstractController {
    @Override
    public void doGet(HttpRequest request, HttpResponse response) {
        if (!isLogined(request.getSession())) {
            response.sendRedirect("/user/login.html");
            return;
        }
        ...

    }

    private static boolean isLogined(HttpSession session) {
        Object user = session.getAttribute("user");
        if (user == null) {
            return false;
        }
        return true;
    }
}
```

session을 활용하면, 클라이언트와 서버 간 상태 공유를 위해 전달하는 데이터는 session 아이디 뿐이다.

- 예측할 수 없도록 생성하는 것은 보안측면에서 중요하다.

쿠키는 보안 강화를 위해 

- `domain` : 쿠키에 접근 가능한 도메인을 지정하는 속성
- `path` : 웹서버의 특정 URL을 지정하는 속성
- `max-age` : 쿠키를 얼마나 유지할 것인지를 지정하는 속성 (초 단위)
- `expires` : max-age와 동일한 역할, 유효한 날짜를 지정하는 속성
- `secure` : https통신에서만 쿠키에 접근하도록 지정하는 속성

속성을 사용할 수 있다.

> 단순히 session만 사용한다고 해서 보안 문제가 해결되는 것은 아니다.

---

# **👨‍✈️ MVC 프레임워크 요구사항 1단계**

MVC 패턴이 사용되기 전, 대부분의 웹 애플리케이션 개발은 JSP에 대부분의 로직을 포함하고 있었다. 하지만 이는 초기 개발 속도는 빠르지만 **유지보수 비용이 증가**하는 문제를 가지고 있었다. 따라서 유지보수 비용을 줄이기 위해 **MVC(Model, View, Controller) 패턴 기반**으로 웹 애플리케이션을 개발하는 방향으로 발전했다.

- JSP에 집중되었던 로직을 `Model`, `View`, `Controller` 3개의 역할로 분리 개발 (like 삼권분립)

**MVC 패턴 기반 개발 시 요청과 응답 흐름**

<img src="https://user-images.githubusercontent.com/33208303/158940954-8b29b986-3019-4ec8-bcb9-37574c618079.png" width="80%">

**MVC 패턴 기반 개발 시 요청과 응답 흐름 + 소스코드**

<img src="https://user-images.githubusercontent.com/33208303/158940959-c4ea361a-fe76-40d3-b4b9-c95d9df8bbe7.png" width="80%">

MVC 패턴을 적용하는 경우 기존과 다른 점은 

**"클라이언트 요청이 처음 진입하는 부분이 컨트롤러"**라는 것이다.

- 기존에는 뷰에 해당하는 JSP가 요청의 첫 진입부였다.
- MVC 패턴을 적용하면 **대부분의 로직은 컨트롤러와 모델이 담당**하고, 뷰에 해당하는 **JSP**는 **컨트롤러에서 전달한 데이터를 출력하는 로직만**을 포함한다.
    - 따라서 JSP의 복잡도는 낮아진다.

> 프레임워크는 개발자 간의 역량 차이가 있더라도 MVC 패턴을 적용해 **일관성 있는 코드**를 구현하도록 강제한다.
> 

**프레임워크와 라이브러리**

모두 애플리케이션 개발에서 발생하는 중복 코드를 제거해 **재사용성을 높임**으로써 **개발 속도를 빠르게 하기 위한 목적**을 가지지만, 가장 큰 차이로는

- **프레임워크**는 특정 패턴 기반으로 개발하도록 강제하는 역할이고,
- **라이브러리**는 강제하는 부분이 없다.

## **👩‍💼 요구사항**

: "MVC 패턴을 지원하는 프레임워크를 구현하는 것”

MVC 패턴을 지원하는 기본적인 구조는 **모든 요청을 하나의 서블릿이 받은 후 요청 URL에 따라 분기 처리하는 방식**으로 구현하면 된다.

MVC 패턴은 사용자의 최초 진입 지점이 **컨트롤러**이다. 따라서 JSP(뷰) 직접 접근하지 않도록 해야 한다. 아래는 MVC 프레임워크를 구현했을 때의 클래스 다이어그램이다.

<img src="https://user-images.githubusercontent.com/33208303/158940961-e96c2bd1-e0f1-4297-b01a-20f9a10e4770.png" width="80%">

- 모든 요청은 `DispatcherServlet` 이 받은 후 요청 URL에 따라 해당 컨트롤러에 작업을 위임한다.
    - `@WebServlet` 으로 URL 매핑 시, `urlPatterns="/"` 와 같이 설정하여 모든 요청 URL이 `DispatcherServlet` 으로 연결되도록 한다.
- CSS, JS, Image와 같은 정적 자원은 컨트롤러가 필요없다. 하지만 위와 같은 매핑 방식은 정적 자원에 대한 요청까지 `DispatcherServlet` 으로 매핑되어버린다.
    - 이를 위해 정적 자원을 처리하는 서블릿 필터를 추가한다. (`core.web.filter.ResourceFilter`)

> **서블릿 필터(Servlet Filter)**
서블릿 실행 전 또는 후에 특정 작업을 하고자 할 때 사용한다. 예로는 인증 필터, 로깅 및 감시 필터, 암호화 필터 등이 있다.

---

# **👼 MVC 프레임워크 구현 1단계**

클라이언트 요청에 대한 처리를 담당하는 부분을 **`Controller` 라는 인터페이스로 추상화**할 수 있다.

```java
public interface Controller {
    String execute(HttpServletRequest req, HttpServletResponse resp)
        throws Exception;
}
```

지금까지 `HttpServlet` 을 상속해 구현한 서블릿 코드들을 `Controller` 인터페이스를 상속해 구현하도록 변경한다.

```java
@WebServlet("/users")
public class ListUserController implements Controller {

    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        if (!UserSessionUtils.isLogined(req.getSession())) {
            return "redirect:/users/loginForm";
        }

        req.setAttribute("users", DataBase.findAll());
        return "/user/list.jsp";
    }
}
```

- 서블릿에서 JSP로 이동할 때 구현해야 하는 중복 코드가 제거되었다.

특별한 로직 구현이 필요 없이 뷰에 대한 이동만을 담당하는 경우 컨트롤러 생성이 불합리하다고 판단해, 뷰에 대한 이동만을 담당하는 컨트롤러를 생성한다. (`ForwardController`)

```java
public class ForwardController implements Controller {
    private String forwardUrl;

    public ForwardController(String forwardUrl) {
        this.forwardUrl = forwardUrl;
        if (forwardUrl == null) {
            throw new NullPointerException("forwardUrl is null. 이동할 URL을 입력하세요.");
        }
    }

    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        return forwardUrl;
    }
}
```

모든 서블릿들이 `Controller` 인터페이스를 상속하도록 구현한 후, 요청 URL과 컨트롤러 매핑을 담당하는 `RequestMapping` 을 생성한다.

```java
public class RequestMapping {
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);
    private Map<String, Controller> mappings = new HashMap<>();

    void initMapping() {
        mappings.put("/", new HomeController());
        mappings.put("/users/form", new ForwardController("/user/form.jsp"));
        mappings.put("/users/loginForm", new ForwardController("/user/login.jsp"));
        mappings.put("/users", new ListUserController());
        mappings.put("/users/login", new LoginController());
        mappings.put("/users/profile", new ProfileController());
        mappings.put("/users/logout", new LogoutController());
        mappings.put("/users/create", new CreateUserController());
        mappings.put("/users/updateForm", new UpdateFormUserController());
        mappings.put("/users/update", new UpdateUserController());

        logger.info("Initialized Request Mapping!");
    }

    public Controller findController(String url) {
        return mappings.get(url);
    }

    void put(String url, Controller controller) {
        mappings.put(url, controller);
    }
}
```

- 발생하는 모든 요청 URL과 서비스를 담당할 컨트롤러를 연결하여 `Map` 으로 저장한다.

마지막 작업은 클라이언트의 모든 요청을 받아 URL에 해당하는 컨트롤러로 작업을 위임하고, 실행된 결과 페이지로 이동하는 작업이다. 이는 `DispatcherServlet` 에서 구현한다. (클래스 다이어그램에서 클라이언트의 요청을 받는 지점)

```java
@WebServlet(name = "dispatcher", urlPatterns = "/", loadOnStartup = 1)
public class DispatcherServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);
    private static final String DEFAULT_REDIRECT_PREFIX = "redirect:";

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

        Controller controller = rm.findController(requestUri);
        try {
            String viewName = controller.execute(req, resp);
            move(viewName, req, resp);
        } catch (Throwable e) {
            logger.error("Exception : {}", e);
            throw new ServletException(e.getMessage());
        }
    }

    private void move(String viewName, HttpServletRequest req, HttpServletResponse resp)
            throws ServletException, IOException {
        if (viewName.startsWith(DEFAULT_REDIRECT_PREFIX)) {
            resp.sendRedirect(viewName.substring(DEFAULT_REDIRECT_PREFIX.length()));
            return;
        }

        RequestDispatcher rd = req.getRequestDispatcher(viewName);
        rd.forward(req, resp);
    }
}
```

- `urlPatterns = "/"` 로 매핑하여 모든 요청 URL이 `DispatcherServlet` 으로 매핑되도록 한다.
    - 일반적으로 모든 요청 URL을 매핑한다고 하면 `"/*"` 으로 생각할 수도 있다. 틀리진 않았지만, 이렇게 하게 되면 모든 JSP에 대한 요청 또한 매핑되어 JSP에 대한 요청이 정상적으로 처리되지 않는다.
- `"/"` 매핑은 매핑되어 있는 서블릿, JSP 요청이 아닌 JS, CSS, Images와 같은 요청을 처리하도록 설계되었다. 톰캣 서버 기본 설정에서, `"/"` 설정은 `"default"` 라는 이름을 가지는 서블릿을 매핑해 정적 자원을 처리하도록 구현되어 있다.
    - 해당 설정을 `DispatcherServlet` 에서 재정의하여 JSP에 대한 요청을 제외한 모든 요청을 담당하도록 구현했다.
- `"default"` 서블릿에서 처리하는 정적 자원에 대한 처리는 `DispatcherServlet` 으로 요청되기 전에 `ResourceFilter` 에서 처리하도록 구현한다.

**주의할 점**

- `HomeController` 에서 `"/"` 로 매핑한 후 `localhost:8080` 요청의 경우, 웹 자원을 관리하는 `webapp` 디렉토리에 `index.jsp` 가 존재하면 `HomeController` 가 아닌 `index.jsp` 로 요청이 처리된다.
    - 이는 `path` 가 없는 경우 처리를 담당하는 기본 파일이 `index.jsp` 로 설정되어 있기 때문이다.
    - 이러한 이유로 `HomeController` 에서 이동할 뷰의 이름이 `home.jsp` 이다.

**`loadOnStartup` 속성**

- 서블릿 인스턴스를 생성하는 시점과 초기화를 담당하는 `init()` 메소드를 어느 시점에 호출할 것인가를 결정하는 설정
    - **이를 하지 않은 경우** 서블릿 인스턴스 생성과 초기화는 서블릿 컨테이너가 시작을 완료한 후 **클라이언트의 요청이 최초로 발생하는 시점에 진행**된다.
    - **설정한 경우 서블릿 컨테이너가 시작하는 시점에 서블릿 인스턴스 생성과 초기화가 진행**된다.
- 설정의 숫자 값으로 서블릿의 초기화 순서를 결정한다. (낮은 순으로 먼저 진행)

**`move()` 메소드**

- 각 서블릿에서 서블릿과 JSP 사이를 이동하기 위해 구현한 모든 중복 코드를 담당한다.

**프론트 컨트롤러(front controller) 패턴**

- 각 컨트롤러의 앞에 모든 요청을 받아 각 컨트롤러에 작업을 위임하는 방식으로 구현하는 것

**MVC 프레임워크 기반으로 요청에서 응답까지의 흐름**

- 클라이언트의 요청 발생
- `ServletFilter`(`ResourceFilter`) 에서의 선처리
- `DispatcherServlet` 으로 요청 전달
- `RequestMapping` 에 url을 전달하고 해당 처리를 맡을 `Controller` 를 반환
- `Controller` 가 작업을 처리
- 응답을 보내기 전 `ServletFilter` 에서의 후처리
- 클라이언트로 응답을 전송

---

# **👨 쉘 스크립트를 활용한 배포 자동화**

원격 서버에 톰캣 서버를 설치하고, 지금까지 구현한 소스코드를 배포한다. 수동으로 배포하는 작업을 쉘 스크립트를 활용해 자동화하는 과정까지 진행한다.

## **👴 요구사항**

- 구현한 기능을 개발 서버에 톰캣 서버를 설치한 후 배포한다.
- 톰캣 로그 파일을 통해 서버의 정상적인 실행을 확인한다.
- 쉘 스크립트를 만들어 배포를 자동화한다.

`TOMCAT_HOME/webapps/ROOT` 에 배포할 서비스를 컴파일하고 패키징(mvn clean package)하여 만든 `target` 디렉토리의 하위 디렉토리를 이동시켜 서버를 시작한다

- 이때, `pom.xml` 에서 사용하는 패키징 방식에 따라 패키징을 수행한다.
    - JAR : 동작에 톰캣이 필요없는 application으로 `bin` 내의 `.class` 파일들을 묶는 것
    - WAR : 톰캣 위에서 동작하는 application으로 `webapp` 내의 파일들을 묶는 것

# **📕 출처**

- **자바 웹 프로그래밍 Next Step : 하나씩 벗겨가는 양파껍질 학습법** - 박재성
- [https://dololak.tistory.com/543](https://dololak.tistory.com/543)
- [쿠키와 document.cookie](https://ko.javascript.info/cookie)
- [쿠키 속성들](https://velog.io/@y1andyu/%EC%BF%A0%ED%82%A4-%EC%86%8D%EC%84%B1%EB%93%A4)
- [Maven project 와 War, Jar파일 차이](https://hoon93.tistory.com/15)
- [DAO, DTO 차이](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=godwings&logNo=221048712980)
- [[Servlet] 서블릿 필터 (Filter)](https://m.blog.naver.com/PostView.naver?isHttpsRedirect=true&blogId=adamdoha&logNo=221665607853)
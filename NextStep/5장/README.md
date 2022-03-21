# 웹 서버 리팩토링, 서블릿 컨테이너와 서블릿의 관계

- 실무에서는 프로젝트의 성격과 규모에 따라 다르지만 메소드 수준까지 철저한 설계를한 후 개발을 시작하거나 대략적인 클래스 설계를 한 후에 진행하는 경우도 있음
- 하지만 대부분의 프로젝트는 요구사항이 명확하지 않고, 프로젝트를 진행하면서 요구사항이 변경되는 경우가 많아 초반의 설계대로 개발하는 경우는 거의 없음
- 설계는 반드시 필요하고 한번만 해야 한다는 생각을 가질 필요가 없음
- 한 번의 작업으로 끝내야 하는 것이 아니라 애플리케이션을 개발하고 배포해 운영하는 동안 끊임없이 진행해야하는 것이 설계
- 지속적인 설계와 구현을 잘 할 수  있는 방법이 지속적인 리팩토링
- **리팩토링은 설계를 개선하기 위한 일련의 활동**

## HTTP 웹 서버 리팩토링 실습

### 🔎 리팩토링할 부분 찾기

- 나쁜 냄새가 나는 코드를 찾을 수 있는 능력
    - 어떤 기준을 가지고 하기보다는 직관에 의존해 진행
    - 직관을 키우려면 좋은 코드, 나쁜 코드 가리지 말고 다른 개발자가 구현한 많은 코드를 읽어야 한다
- 자신이 구현한 코드에 대해 지속적으로 의도적인 리팩토링

### ❓ 리팩토링

### 1️⃣ 요청 데이터를 처리하는 로직을 별도의 클래스로 분리한다

`HttpRequest.java` : 클라이언트 요청 데이터를 담고 있는 `InputStream` 을 생성자로 받아 (HTTP 메소드, URL, Header, 본문)을 분리하는 class

```java
public class HttpRequest {
    // getMethod(), getPath(), getHeader(), getParameter();
    private static final Logger log = LoggerFactory.getLogger(HttpRequest.class);

    private Map<String, String> headers = new HashMap<>();
    private Map<String, String> params = new HashMap<>();
    private RequestLine requestLine;

    public HttpRequest(InputStream in) {
        try {
            BufferedReader br = new BufferedReader(new InputStreamReader(in, "UTF-8"));
            String line = br.readLine();

            if (line == null)
                return;

            requestLine = new RequestLine(line);

            line = br.readLine();
            while (line != null && !line.equals("")) {
                log.debug("header : {}", line);
                String[] tokens = line.split(":");
                headers.put(tokens[0].trim(), tokens[1].trim());
                line = br.readLine();
            }

            if ("POST".equals(getMethod())) {
                String body = IOUtils.readData(br, Integer.parseInt(headers.get("Content-Length")));
                params = HttpRequestUtils.parseQueryString(body);
            } else {
                params = requestLine.getParams();
            }

        } catch (IOException io) {
            log.error(io.getMessage());
        }
    }

    public String getMethod() {
        return requestLine.getMethod();
    }

    public String getPath() {
        return requestLine.getPath();
    }

    public String getHeader(String name) {
        return headers.get(name);
    }

    public String getParameter(String name) {
        return params.get(name);
    }
}
```

> 코드를 구현하다가 생긴 궁금증 : `setHeader()` 에서 while문을 돌 때, null 처리를 안하면 안되는가? 다른 사람의 코드를 비교했을 때, null 처리를 한 사람은 없다 → 다같이 안됐다~!
> 

### 2️⃣ 응답 데이터를 처리하는 로직을 별도의 클래스로 분리한다

`HttpResponse.java` : 응답 헤더 정보를 `Map<String,String>` 으로 관리

```java
public class HttpResponse {
    private static final Logger log = LoggerFactory.getLogger(HttpResponse.class);

    private DataOutputStream dos = null;
    private Map<String, String> headers = new HashMap<>();

    public HttpResponse(OutputStream out) {
        dos = new DataOutputStream(out);
    }

    public void addHeader(String key, String value) {
        headers.put(key, value);
    }

    public void forward(String url) {
        try {
            byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());

            if(url.endsWith(".css")){
                headers.put("Content-Type", "text/css");
            } else if (url.endsWith(".js")) {
                headers.put("Content-Type", "application/javascript");
            } else {
                headers.put("Content-Type", "text/html;charset=utf-8");
            }

            headers.put("Content-Length", body.length + "");
            response200Header(body.length);
            responseBody(body);

        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    public void forwardBody(String body) {
        byte[] contents = body.getBytes();

        headers.put("Content-Type", "text/html;charset-utf-8");
        headers.put("Content-Length", contents.length + "");
        response200Header(contents.length);
        responseBody(contents);
    }

    public void sendRedirect(String redirectUrl) {
        try {
            dos.writeBytes("HTTP/1.1 302 Found \r\n");
            processHeaders();
            dos.writeBytes("Location: " + redirectUrl + "\r\n");
            dos.writeBytes("\r\n");
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private void response200Header(int lengthOfBodyContent) {
        try {
            dos.writeBytes("HTTP/1.1 200 OK \r\n");
            processHeaders();
            dos.writeBytes("\r\n");
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private void responseBody(byte[] body) {
        try {
            dos.write(body, 0, body.length);
            dos.writeBytes("\r\n");
            dos.flush();
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private void processHeaders() {
        try {
            Set<String> keys = headers.keySet();
            for (String key : keys) {
                dos.writeBytes(key + ": " + headers.get(key) + " \r\n");
            }
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }
}
```

### 🧪 테스트 코드 기반의 개발

1. 클래스에 버그가 있는지를 빨리 찾아 구현할 수 있음
    
    → 클래스에 대한 테스트를 마친 후 사용한다면 수동 테스트 횟수는 급격하게 줄어듦
    
2. 디버깅하기가 쉬움
    
    → 클래스에 대한 단위 테스트를 하는 것은 결과적으로 디버깅을 좀 더 쉽고 빠르게 할 수 있기 때문에 개발 생산성을 높여줌
    
3. 테스트 코드가 있기 때문에 마음 놓고 리팩토링을 할 수 있음
    
    → 리팩토링을 하면 지금까지 했던 테스트를 다시 반복해야 하는데 테스트 코드가 이미 존재한다면 리팩토링을 완료한 후 테스트를 한번 실행하기만 하면 됨
    

- **상수값이 연관성을 가지는 경우 `enum` 을 사용하기 적합하다**
- **객체를 최대한 활용하는 연습을 하는 첫번째는 객체에서 값을 꺼낸 후 로직을 구현하려고 하지말고**
    
    **`값을 가지고 있는 객체에 메시지롤 보내 일을 시키도록` 연습**
    
- **객체를 최대한 활용했는데 복잡도가 증가하고 책임이 점점 많아진다는 느낌이 드는 순간 `새로운 객체`를 추가**

### 3️⃣ 다형성을 활용해 클라이언트 요청 URL에 대한 분기 처리를 제거한다

- 새로운 기능이 추가되거나 수정사항이 발생하더라도 변화의 범위를 최소화하도록 설계를 개선

`Controller.java` : `RequestHandler` 의 분기문에서 분리했던 메소드의 구현 class를 위한 인터페이스

```java
package controller;

import http.HttpRequest;
import http.HttpResponse;

public interface Controller {
    void service(HttpRequest request, HttpResponse response);
}
```

`RequestMapping` : 각 요청 URL과 URL에 대응하는 `Controller`를 연결

```java
public class RequestMapping {
    private static Map<String, Controller> controllers = new HashMap<>();

    static {
        controllers.put("/user/create", new CreateUserController());
        controllers.put("/user/login", new LoginController());
        controllers.put("/user/list", new ListUserController());
    }

    public static Controller getController(String requestUrl) {
        return controllers.get(requestUrl);
    }
}
```

### ❗️ HTTP 웹 서버의 문제점

- HTTP 요청과 응답 헤더, 본문 처리와 같은 데 시간을 투자함으로써 정작 중요한 로직을 구현하는 데 투자할 시간이 상대적으로 적다
- 동적인 HTML을 지원하는 데 한계가 있다. 동적으로 HTML을 생성할 수 있지만 많은 코딩량을 필요로 한다
- 사용자가 입력한 데이터가 서버를 재시작하면 사라진다. 사용자가 입력한 데이터를 유지하고 싶다

### 🤜🏻 서블릿 컨테이너, 서블릿/JSP를 활용한 문제해결

- `서블릿` : HTTP의 클라이언트 요청과 응답에 대한 표준을 정해 놓은 것
- `서블릿 컨테이너` : `서블릿 표준에 대한 구현`을 담당하고 있으며 앞에서 구현한 웹서버가 서블릿 컨테이너의 역할
- `서블릿 컨테이너와 서블릿의 동작 방식`
    - 서블릿 컨테이너는 서버가 시작할 때, 서블릿 인스턴스를 생성해, 요청 URL과 서블릿 인스턴스를 연결해 놓는다
    - 클라이언트에서 요청이 오면 요청 URL 에 해당하는 서블릿을 찾아 서블릿에 모든 작업을 위임한다
    - 서블릿 컨테이너는 서버를 시작할 때 클래스패스에 있는 클래스 중 `HttpServlet`을 상속하는 클래스를 찾은 후 `@WebServlet` 애노테이션의 값을 읽어 요청 `URL`과 `서블릿`을 연결하는 `Map`을 생성한다
    - `Map`에 `서블릿`을 추가하고, `요청 URL` 에 대한 `서블릿`을 찾아 서비스하는 역할을 `서블릿 컨테이너`가 담당
    - 서블릿 컨테이너의 중요한 역할 :  서블릿 클래스의 인스턴스 생성, 요청 URL과 서블릿 인스턴스 매핑, 클라이언트 요청에 해당하는 서블릿을 찾은 후 서블릿에 작업을 위임하는 역할, 서블릿과 관련한 초기화와 소멸 작업을 담당
    

- `서블릿의 생명주기` : 서블릿의 생성, 초기화, 서비스, 소멸의 전체 과정
- 서블릿은 서블릿 컨테이너가 시작할 때 한번 생성되면 모든 스레드가 같은 인스턴스를 재사용한다
    
    → 멀티스레드가 인스턴스 하나를 공유하면서 발생하는 문제와 이에 대한 해결방법은 이후에 살펴봄!

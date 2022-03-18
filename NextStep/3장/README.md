# 🚊 서비스 요구사항
## **🚒 질문 목록 화면**

- 회원가입
- 로그인
- 로그아웃
- 개인정보 수정
- 질문하기
- 각 질문 제목을 클릭하여 상세보기 화면으로 이동
    - 답변 추가, 질문과 답변의 수정/삭제

---

# **🚲 로컬 개발 환경 구축**

💡 자바 8버전 & Intellij 통합 개발 환경
- [GitHub - slipp/web-application-server: 웹 애플리케이션 서버 실습을 위한 뼈대](https://github.com/slipp/web-application-server)
- [터미널에서 git clone 및 eclipse로 프로젝트 import](https://www.youtube.com/watch?v=5hjYe_PggJI)

---

# **🚍 원격 서버에 배포**

💡 애자일 프로세스의 접근 방식
: 현 시점에 가장 가치있는, 동작하는 소프트웨어를 만드는 것을 원칙으로 하는 방식이다.

로컬 개발 환경을 구축한 후 바로 실습 단계를 진행할 수 있지만 실습을 진행하기 전에 먼저 HTTP 웹 서버를 원격 서버에 배포해보자. 이러한 경험을 통해 터미널 환경에서 작업하는 것에 익숙해지도록 한다.

https://opentutorials.org/module/1946 참고하여 AWS에 대한 회원가입, 우분투 운영체제 설치, SSH를 통한 접근을 진행할 수 있다.

## **🚇 요구사항**

로컬 개발 환경에 설치한 HTTP 웹 서버를 물리적으로 떨어져 있는 원격 서버에 배포해 정상적으로 동작하는지 테스트한다.

---

# **🚄 웹 서버 실습**

## **💺 실습 환경 세팅 및 소스코드 분석**

실습으로 진행할 HTTP 웹 서버는 앞의 로컬 개발 환경에서 세팅한 Git 저장소의 **master** 브랜치에서 시작하면 된다.

실습 HTTP 웹 서버의 핵심이 되는 코드는 webserver 패키지의 `WebServer`, `RequestHandler` 클래스이다.

- `WebServer` 클래스는 웹 서버를 시작하고, 사용자의 요청이 있을 때까지 대기 상태에 있다가 사용자 요청이 있을 경우 해당 요청을 `RequestHandler` 클래스에 위임하는 역할을 수행한다.

```java
// in WebServer.java
try (ServerSocket listenSocket = new ServerSocket(port)) {
  log.info("Web Application Server started {} port.", port);

  // 클라이언트가 연결될때까지 대기한다.
  Socket connection;
  while ((connection = listenSocket.accept()) != null) {
      RequestHandler requestHandler = new RequestHandler(connection);
      requestHandler.start();
  }
}
```

- 사용자 요청이 발생할 때까지 대기 상태에 있도록 지원하는 역할은 자바에 포함되어 있는 `ServerSocket`클래스가 담당한다.
- 사용자 요청이 발생하는 순간 클라이언트와 연결을 담당하는 Socket 을 `RequestHandler`에 전달하면서 새로운 스레드를 실행하는 방식으로 **멀티스레드 프로그래밍**을 지원한다.

```java
// in RequestHandler.java
package webserver;

import java.io.DataOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class RequestHandler extends Thread {
    private static final Logger log = LoggerFactory.getLogger(RequestHandler.class);

    private Socket connection;

    public RequestHandler(Socket connectionSocket) {
        this.connection = connectionSocket;
    }

    public void run() {
        log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
                connection.getPort());

        try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
            // TODO 사용자 요청에 대한 처리는 이 곳에 구현하면 된다.
            DataOutputStream dos = new DataOutputStream(out);
            byte[] body = "Hello World".getBytes();
            response200Header(dos, body.length);
            responseBody(dos, body);
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private void response200Header(DataOutputStream dos, int lengthOfBodyContent) {
        try {
            dos.writeBytes("HTTP/1.1 200 OK \r\n");
            dos.writeBytes("Content-Type: text/html;charset=utf-8\r\n");
            dos.writeBytes("Content-Length: " + lengthOfBodyContent + "\r\n");
            dos.writeBytes("\r\n");
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }

    private void responseBody(DataOutputStream dos, byte[] body) {
        try {
            dos.write(body, 0, body.length);
            dos.flush();
        } catch (IOException e) {
            log.error(e.getMessage());
        }
    }
}
```

- `RequestHandler` 클래스는 Thread를 상속하고, **사용자의 요청에 대한 처리와 응답에 대한 처리를 담당하는 클래스**이다.
- 앞으로의 모든 실습은 `RequestHandler` 클래스의 `run()` 메소드에서 구현할 수 있다.
    - 해당 메소드의 복잡도가 증가하는 경우, 새로운 클래스 또는 메소드로 분리하는 방식으로 리팩토링을 수행할 수 있다.
- `InputStream` : 클라이언트(웹 브라우저)에서 서버로 요청을 보낼 때 전달되는 데이터를 담당
- `OutputStream` : 서버에서 클라이언트에 응답을 보낼 때 전달되는 데이터를 담당
    - 웹 서버 입장에서 이해!

전체적인 흐름은,

- `브라우저` 와 `WebServer의 RequestHandler` 는 `Socket` 이라는 통로로 통신한다.
- 해당 통로의 `InputStream` & `OutputStream` 수단을 통해 데이터를 주고 받는다.

> "프로그래밍 실행 중 발생하는 로그 메세지를 주의 깊게 살펴보는 것은 좋은 습관 중 하나이다."

## **🛵 실습 요구사항**

**요구사항 1 - index.html 응답하기**

: 현재 HTTP 웹 서버에 접속하면 무조건 "Hello World" 라는 문구만 뜬다. 이를 `http://localhost:8080/index.html` 로 접속했을 때, `index.html` 파일을 응답할 수 있도록 한다.

```java
try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
  // TODO 사용자 요청에 대한 처리는 이 곳에 구현하면 된다.
  BufferedReader br = new BufferedReader(new InputStreamReader(in));

  String line = br.readLine();
  if (line == null) {return ;}
  String[] tokens = line.split(" ");

  while (!"".equals(line)) {
      line = br.readLine();
  }

  byte[] body = Files.readAllBytes(new File("./webapp" + tokens[1]).toPath());
  // byte[] body = "Hello World".getBytes();

  DataOutputStream dos = new DataOutputStream(out);
  response200Header(dos, body.length);
  responseBody(dos, body);
} catch (IOException e) {
  log.error(e.getMessage());
}
```

- `InputStream` , 즉 클라이언트가 보내는 요청을 한 줄 단위로 읽기 위해 `BufferedReader` 를 사용한다.
- `null` 인 경우는 `return`
- `line` 을 **" "** 단위로 잘라 저장하고, `GET /index.html HTTP/1.1` 같은 형식이므로 `token[1]` 에 요청하는 파일의 정보가 담긴다.
- 해당 파일 데이터를 `byte[]` 로 읽어 `response` 에 전달한다.

**요구사항 2 - GET 방식으로 회원가입하기**

: 회원가입 메뉴를 클릭하면 `http://localhost:8080/user/form.html` 로 이동하면서 회원가입이 가능하다. 회원가입을 수행하면 다음과 같은 형태로 사용자 입력값이 서버에 전달된다.

```bash
/user/create?userId=javajigi&password=password&name=JaeSung&email=javajigi%40slipp.net
```

HTML과 URL을 비교하고 사용자 입력 값을 파싱해 `model.User` 클래스에 저장한다.

```java
if (url.startsWith("/user/create")) {
    int index = url.indexOf("?");
    if (index != -1) {
        Map<String, String> map = HttpRequestUtils.parseQueryString(url.substring(index+1));
        User user = new User(map.get("userId"), map.get("password"), map.get("name"), map.get("email"));
        System.out.println(user);
    }
}
```

- URL이 `/user/create` 로 시작하는 경우, 회원가입 요청이다.
- 쿼리 문자열을 (이름, 값) 과 같은 형태로 분석하는 `parseQueryString` 을 이용하여 파라미터로부터 사용자 입력 값을 파싱한다.
- 파싱한 값을 `User` 객체에 담는다.

**요구사항 3 - POST 방식으로 회원가입하기**

: `http://localhost:8080/user/form.html` 파일의 `form` 태그 메소드를 **`POST`** 로 수정한 후 회원가입을 구현한다.

```java
else if (tokens[0].equals("POST")) {
    while(!line.equals("")) {
        line = br.readLine();
        if (line.contains("Content-Length")) {
            String[] content = line.split(":");
            contLength = Integer.parseInt(content[1].trim());
        }
    }
    String body = IOUtils.readData(br, contLength);
    Map<String, String> map = HttpRequestUtils.parseQueryString(body);
    User user = new User(map.get("userId"), map.get("password"), map.get("name"), map.get("email"));
    System.out.println(user);
}
```

- `/user/create` 분기에서 걸러지고, `GET` or `POST` 이므로 한 번 더 분기를 나눈다.
- HTTP header와 body 사이에 **빈 공백이 존재**한다고 했으므로, 그 전까지 HTTP 요청을 읽으면서, `Content-Length` 값을 탐색한다.
    - 이때, `Content-Length: 55` 와 같은 형식이라 `content[1]` 의 첫번째 자리에 공백이 포함되므로 `trim()` 을 이용해 이를 제거한다.
- `IOUtils.readData` 에 `reader` 와 `size` 를 주고 데이터를 읽고 `GET` 과 동일하게 사용자 입력 값을 파싱한다.

**요구사항 4 - 302 status code 적용**

: 회원가입을 완료한 후 `/index.html` 로 이동하고 싶다.

```java
response302Header(dos, "/index.html");

...

private void response302Header(DataOutputStream dos, String url) {
    try {
        dos.writeBytes("HTTP/1.1 302 Redirect \r\n");
        dos.writeBytes("Location: " + url + " \r\n");
        dos.writeBytes("\r\n");
    } catch (IOException e) {
        log.error(e.getMessage());
    }
}
```

- **302** code는 3XX Redirection 클래스에 속하는 HTTP 상태코드이다.
    - 클라이언트를 지정된 위치로 이동시키거나 참조하게 하는 등의 동작을 수행한다.
    - https://en.wikipedia.org/wiki/HTTP_302, https://nsinc.tistory.com/168

**요구사항 5 - 로그인하기**

: 로그인 메뉴를 클릭하면 `~/user/login.html` 로 이동해 로그인할 수 있다. 성공하면 `/index.html` 로 이동하고, 그렇지 않으면 `/user/login_failed.html` 로 이동해야 한다.

앞에서 회원가입한 사용자로 로그인할 수 있어야 한다. 로그인 성공 시, 쿠키를 활용해 로그인 상태를 유지할 수 있어야 한다. 로그인이 성공할 경우 요청 헤더의 **Cookie 헤더 값**이 `logined=true`, 실패할 경우 `logined=false` 로 전달되어야 한다.

```java
else if (url.equals("/user/login")) {
    if (tokens[0].equals("POST")) {
        while(!line.equals("")) {
            line = br.readLine();
            log.debug("Header : {}", line);
            if (line.contains("Content-Length")) {
                String[] content = line.split(":");
                contLength= Integer.parseInt(content[1].trim());
            }
        }
        String body = IOUtils.readData(br, contLength);
        Map<String, String> map = HttpRequestUtils.parseQueryString(body);
        User user = DataBase.findUserById(map.get("userId"));
        log.debug("User : {}", user);
        if (user == null) {
            responseFail(dos, "/user/login_failed.html");
            return;
        }
        if (user.getPassword().equals(map.get("password"))) {
            response302Logined(dos);
        } else {
            responseFail(dos, "/user/login_failed.html");
            return;
        }
    }
}

private void response302Logined(DataOutputStream dos) {
    try {
        dos.writeBytes("HTTP/1.1 302 Redirect \r\n");
        dos.writeBytes("Location: /index.html \r\n");
        dos.writeBytes("Set-Cookie: logined=true \r\n");
        dos.writeBytes("\r\n");
    } catch (IOException e) {
        log.error(e.getMessage());
    }
}

private void responseFail(DataOutputStream dos, String url) throws IOException {
    byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());

    response200Header(dos, body.length);
    responseBody(dos, body);
}
```

- `/user/login` 에 **POST**를 보내는 경우, HTTP body를 읽어 사용자 입력 값을 추출한다.
- DB에 저장되어 있는 값과 비교한 후, 각 경우에 맞게 response를 전달한다.

**요구사항 6 - 사용자 목록 출력**

: 사용자가 로그인 상태인 경우 `http://localhost:8080/user/list` 로 접근했을 때 사용자 목록을 출력한다. 로그인하지 않은 상태라면 `login.html` 로 이동한다.

```java
else if (url.equals("/user/list")) {
    if (tokens[0].equals("GET")) {
        while(!line.equals("")) {
            line = br.readLine();
            log.debug("Header : {}", line);
            if (line.contains("Cookie")) {
                login = isLogin(line);
            }
        }
        if (!login) {
            responseFail(dos, "/user/login.html");
            return;
        }
        Collection<User> users = DataBase.findAll();
        StringBuilder sb = new StringBuilder();
        sb.append("<h1>회원 목록</h1>");
        for (User user : users) {
            sb.append("<p>" + user.getUserId() + "</p>");
            sb.append("<p>" + user.getName() + "</p>");
            sb.append("<p>" + user.getEmail() + "</p>");
        }
        sb.append("<h2>목록 끝</h2>");
        byte[] body = sb.toString().getBytes();
        response200Header(dos, body.length);
        responseBody(dos, body);
    }
}
```

- `Cookie` 값을 가져와 로그인한 사용자인지 판단한다.
    - 로그인하지 않은 경우 `login.html` 을 전달한다.
    - `Database.findAll()` 로 회원가입된 모든 사용자를 가져온다.
    - `StringBuilder` 를 이용해 출력할 HTML code를 작성하고, 이를 response한다.

> **String은 불변 객체**이다. 2개의 String 객체가 있을 때, 두 객체를 더하는 연산은 새로운 String을 생성한다. 이 때, **메모리 할당과 해제를 발생**시켜 더하는 연산이 많아질수록 성능적으로 좋지 않다.
따라서, 문자열을 더할 때 새로운 객체를 생성하는 것이 아닌 기존의 데이터에 더하는 방식을 사용하는 **StringBuilder** 클래스가 생겨났다.

**요구사항 7 - CSS 지원하기**

: CSS 파일을 지원하도록 구현한다.

```java
else if (url.endsWith(".css")) {
    byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());
    response200CssHeader(dos, body.length);
    responseBody(dos, body);
}

...

private void response200CssHeader(DataOutputStream dos, int length) {
    try {
        dos.writeBytes("HTTP/1.1 200 OK \r\n");
        dos.writeBytes("Content-Type: text/css\r\n");
        dos.writeBytes("Content-Length: " + length + "\r\n");
        dos.writeBytes("\r\n");
    } catch (IOException e) {
        log.error(e.getMessage());
    }
}
```

- HTTP 요청의 header에 `./css/style.css` 와 같이 날라오기에, `endsWith()` 메소드를 사용해 이를 추출한다.
- `Content-Type` 이 `text/css` 로 달라진 것을 확인할 수 있다.

---

# **🚀 추가 학습 자료**

## **🚦 Git과 Github**

**Git**

: 최근에 가장 많이 사용하는 버전 관리 시스템(Version Control System) 의 한 종류, VCS에 대한 기본 기능을 제공하는 도구이다.

**Github**

: Git이 제공하는 기능과 더불어 개발자들이 유용하게 사용할 수 있는 추가적인 기능을 제공하는 웹 애플리케이션이다.

- 참고
  - [Learn Git Branching](http://pcottle.github.io/learnGitBranching)
  - [Sourcetree | Free Git GUI for Mac and Windows](http://www.sourcetreeapp.com)
    
## **🚥 빌드 도구 메이븐**

### **빌드 도구**

: 프로젝트와 관련한 설정을 관리하면서 소스코드(프로덕션, 테스트)에 대한 컴파일, 컴파일을 위해 필요한 라이브러리 관리, 테스트, 배포를 위한 패킹 작업 등의 작업을 자동화할 수 있도록 지원하는 도구

- 프로젝트 디렉토리 구조와 의존성 라이브러리를 관리하므로, 프로젝트를 통합 개발 도구 프로젝트로 변환하는 것도 가능하다.
- 웹 애플리케이션 개발에서 발생하는 단순, 반복적인 작업을 자동화할 수 있다.
- 간단한 배포 작업까지 수행할 수 있다.

### **Maven (메이븐)과 Gradle (그래들)**

**메이븐**

: 자바 진영에서 더 오랫 동안 사용해온 빌드 도구로, 설정 파일을 **XML**로 작성한다.

- **XML**을 사용하기에 프로젝트의 크기가 커질수록 내용이 길어지고 가독성이 떨어진다.

```jsx
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<groupId>org.nhnnext</groupId>
	<artifactId>web-application-server</artifactId>
	<version>1.0</version>
	<packaging>jar</packaging>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
	</properties>

	<dependencies>
		<!-- unit testing -->
		<dependency>
			<groupId>junit</groupId>
			<artifactId>junit</artifactId>
			<version>4.11</version>
			<scope>test</scope>
		</dependency>

		<dependency>
			<groupId>com.google.guava</groupId>
			<artifactId>guava</artifactId>
			<version>18.0</version>
		</dependency>

		<!-- logger -->
		<dependency>
			<groupId>ch.qos.logback</groupId>
			<artifactId>logback-classic</artifactId>
			<version>1.1.2</version>
		</dependency>
	</dependencies>

	<build>
		<finalName>web-application-server</finalName>
		<sourceDirectory>src/main/java</sourceDirectory>
		<testSourceDirectory>src/test/java</testSourceDirectory>
		<testOutputDirectory>target/test-classes</testOutputDirectory>

		<resources>
			<resource>
				<directory>src/main/resources</directory>
			</resource>
		</resources>

		<plugins>
			<plugin>
				<artifactId>maven-eclipse-plugin</artifactId>
				<version>2.9</version>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<version>3.1</version>
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
					<encoding>utf-8</encoding>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-dependency-plugin</artifactId>
				<version>2.4</version>
				<executions>
					<execution>
						<id>copy-dependencies</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```

**그래들**

: 최근 인기도가 높아지는 빌드 도구로, `그루비` 라는 언어를 기반으로 설정 파일을 관리하여 설정 파일에 대한 유연성도 높고, 코딩량도 메이븐의 **XML**에 비해 적다.

- 빌드 속도가 메이븐에 비해 10~100배가량 빠르다.
- 길이와 가독성 측면에서 효율적이다.

두 빌드 도구의 기본 개념은 비슷한 점이 많아, 하나의 빌드 도구만 익히면 다른 빌드 도구에 대한 학습 비용도 낮다.

## **🚧 디버깅을 위한 로깅(logging)**

개발자는 다음과 같은 목적으로 수많은 메세지를 출력한다.

- 애플리케이션이 정상적으로 동작하는지 확인하기 위한 목적
- 애플리케이션에 문제가 발생했을 때 원인을 파악하기 위한 디버깅 목적

이때, 친숙한 System.out.println() 으로 출력하는데, 이는 **애플리케이션의 성능을 저하**시키는 원인이다.

- 이를 이용해 디버깅 메세지를 출력하면 파일로 메세지가 출력하게 되는데, 파일에 메세지를 출력하는 작업은 상당한 비용이 발생한다.

### **logging 라이브러리**

이와 같은 단점을 보완하기 위해 **logging 라이브러리**가 등장했다. 자바 진영에서 많이 사용하는 로깅 라이브러리는 Logback 이다.

자바 진영에는 많은 로깅 라이브러리 구현체가 존재하는데, 더 좋은 구현체가 등장할 때마다 전체 소스코드에서 로깅 라이브러리 구현 부분을 수정하는 어려움이 있다. 이를 해결하기 위해 SLF4J 라는 라이브러리를 활용해 로깅 API에 대한 창구를 일원화했다.

- 즉, 자바 소스코드는 SLF4J 라이브러리를 사용해 디버깅 메세지를 남기면 실제로 디버깅 메세지를 출력하는 구현체는 Log4J, Logback 이 담당하는 방식이다.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger log = LoggerFactory.getLogger(RequestHandler.class);
```

- 해당 프로젝트는 로깅 구현체로 Logback 라이브러리를 사용한다. 하지만 이를 직접 사용하지 않고 SLF4J 를 import하고 있다.
- Logback 라이브러리에 대한 구현체는 메이븐 설정 파일인 pom.xml 에서 설정하고 있다.

### **로그 레벨**

로깅 라이브러리는 메세지 출력 여부를 **로그 레벨**을 통해 관리한다.

- `TRACE` = `log.trace()`, `DEBUG` = `log.debug()`, `INFO` = `log.info()`, `WARN` = `log.warn()`, `ERROR` = `log.error()`
- TRACE < DEBUG < INFO < WARN < ERROR 순으로 높아진다.
    - 레벨이 높을 수록 출력되는 메세지는 적어지고, 낮을 수록 더 많은 로깅 레벨이 출력된다.
- 각 메세지에 대한 로그 레벨은 로깅 메세지를 구현할 때 결정된다.

### **로그 메세지 생성**

로그 메세지 출력 시 다음과 같은 메세지 구현 방식이 일반적이다.

```java
log.debug("New Client Connect! Connected IP : " + connection.getInetAddress() + ", Port : " + conneciton.getPort());
```

- 하지만 위와 같은 방식은 로그 레벨이 INFO 나 WARN 인 경우 굳이 해당 메소드에 인자 전달을 위해 문자열을 더하는 부분이 실행될 필요가 없다. (아까 위에서 StringBuilder 를 사용하는 이유와 동일)
- SLF4J 는 이러한 단점 보완을 위해 동적인 메세지를 구현하는 별도의 메소드를 제공한다.

```java
log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(), conneciton.getPort());
```

- 위 방식으로 debug() 메소드에서 로그 레벨에 따라 메세지를 더할 필요가 있는지의 여부를 판단하게 된다.

Logback 의 로그 레벨과 메세지 형식에 대한 설정 파일은 logback.xml 이며, 이를 통해 출력되는 로그 메세지의 패턴을 변경할 수 있다.

> Log4j, SLF4J 라이브러리별 템플릿 또한 존재해, 단순 반복적으로 발생하는 코드를 추가하여 개발 생산성을 높이는 데 도움이 될 수 있다.
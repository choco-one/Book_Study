# 🚩 HTTP 웹 서버 구현

## 요구사항 1 - index.html 응답하기

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
     connection.getPort());

	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
	  BufferedReader br = new BufferedReader(new InputStreamReader(in, "UTF-8"));
	  String line = br.readLine();
    log.debug("request line: {}", line);

    if (line == null) {
	    return;
    }

		String[] tokens = line.split(" ");

    while (!line.equals("")) {
	    line = br.readLine();
      log.debug("header : {}", line);
		}

		DataOutputStream dos = new DataOutputStream(out);
    byte[] body = Files.readAllBytes(new File("./webapp" + tokens[1]).toPath());
    response200Header(dos, body.length);
    responseBody(dos, body);
	}
[...]
```

**콘솔 화면**

```java
16:49:11.032 [DEBUG] [Thread-0] [webserver.RequestHandler] - New Client Connect! Connected IP : /0:0:0:0:0:0:0:1, Port : 63062
16:49:11.032 [DEBUG] [Thread-1] [webserver.RequestHandler] - New Client Connect! Connected IP : /0:0:0:0:0:0:0:1, Port : 63063
16:49:11.917 [DEBUG] [Thread-0] [webserver.RequestHandler] - line: GET /index.html HTTP/1.1
16:49:11.917 [DEBUG] [Thread-0] [webserver.RequestHandler] - line: Host: localhost:8080
16:49:11.917 [DEBUG] [Thread-0] [webserver.RequestHandler] - line: Connection: keep-alive
[...]
```

- 클라이언트로부터 2개의 요청이 발생하며, 서로 다른 port로 연결
- 서버는 각 요청에 대응하는 Thread를 생성해서 동시에 실행
- 요청의 첫 번째 라인은 `GET /index.html HTTP/1.1` 의 형태
- 첫 번째 라인을 제외한 나머지 요청 데이터는 `<필드 이름>: <필드 값>` 형태
- 요청의 마지막은 빈 문자열(`””`)로 구성

> HTML 요청을 한번 보내더라도 그 HTML 응답 내용에 CSS, 자바스크립트, 이미지 등의 자원이 포함되어 있으면 서버에 해당 자원을 다시 요청하고, 여러 번의 요청과 응답을 주고 받음
> 

### HTTP 규약

```python
# 요청 라인
POST /user/create HTTP/1.1
# 요청 헤더
HOST: localhost:8080
Connection-Length: 59
Content-Type: application/x-wwww-form-urlencoded
Accept: */*
# 헤더와 본문 사이의 빈 공백 라인

# 요청 본문
userId=test&password=password
```

- 요청 라인(Request Line)
    
    요청 데이터의 첫 번째 라인으로, `HTTP-메소드 URI HTTP-버전` 으로 구성
    
    - HTTP-메소드
        
        요청의 종류
        
    - URI
        
        클라이언트가 서버에 유일하게 식별할 수 있는 요청 자원의 경로
        
    - HTTP-버전
        
        현재 요청의 HTTP 버전, 주로 HTTP/1.1 사용
        
- 요청 헤더(Request Headers)
    
    `<필드이름>: <필드 값>` 쌍으로 구성
    
- 상태 라인(Status Line)
    
    서버에서 클라이언트로의 응답 또한 요청과 비슷하게 구성되어 있는데, 첫 번째 라인이 상태 라인으로 다름
    
    `HTTP-버전 상태코드 응답구문` 으로 구성
    

## 요구사항 2 - GET 방식으로 회원가입하기

웹 브라우저는 HTML form 태그 구현에 따라 요청 라인을 생성해 서버에 요청을 보냄

`GET /user/create?usesrId=test&password=password&name=anonymous&email=test%40abc.net HTTP/1.1`

위의 요청 라인 예시에서 GET은 form 태그 method 속성 값이고, 요청 URI는 action 속성 값

요청 URI에서 `/user/create`는 **경로(path)**, 물음표 뒤에 매개변수는 **쿼리 스트링(query string)**이라고 부름

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]

		String url = tokens[1];
		if (url.startsWith("/user/create")) {
			int index = url.indexOf("?");
			String queryString = url.substring(index+1);
      Map<String, String> params = HttpRequestUtils.parseQueryString(queryString);
      User user = new User(params.get("userId"), params.get("password"), params.get("name"), params.get("email"));
      log.debug("User: {}", user);
		} else {
				DataOutputStream dos = new DataOutputStream(out);
		    byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());
		    response200Header(dos, body.length);
		    responseBody(dos, body);
		}
		[...]
```

아직 응답을 보내지 않아서 “수신된 데이터 없음" 이라는 에러 메세지가 출력됨

> **GET 방식의 문제점**
1. 사용자가 입력한 데이터가 브라우저 URL 입력창에 노출됨 (보안이 중요한 경우 **POST** 방식 선호)
2. 요청 라인 길이에 제한이 있음
> 

## 요구사항 3 - POST 방식으로 회원가입하기

POST 방식으로 데이터를 전달하기 위해 form.html의 form 태그 method 속성을 get에서 post로 수정

`POST /user/create HTTP/1.1` 형식으로 요청 라인이 바뀌고, 

쿼리 스트링은 HTTP 요청의 본문(body)를 통해 전달됨. 

(본문 데이터의 길이는 Content-Length라는 필드 이름으로 헤더에 전달)

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]
		String[] tokens = line.split(" ");
    int contentLength = 0;
    while (!line.equals("")) {
			 log.debug("header: {}", line);
       line = br.readLine();
       if (line.contains("Content-Length")) {
	       contentLength = getContentLength(line);
       }
		}

		String url = tokens[1];
		if ("/user/create".equals(url)) {
			String body = IOUtils.readData(br, contentLength);
      Map<String, String> params = HttpRequestUtils.parseQueryString(body);
      User user = new User(params.get("userId"), params.get("password"), params.get("name"), params.get("email"));
      log.debug("User: {}", user);
		} 
		[...]
}

private int getContentLength(String line) {
	String[] headerTokens = line.split(":");
	return Integer.parseInt(headerTokens[1].trim());
}
[...]
```

- HTTP는 요청 형태에 따라 GET, POST, HEAD, PUT, DELETE 등 여러 메소드를 지원함
- 하지만 기본으로 GET, POST만 주로 사용함
    - Why? <form> 태그가 지원하는 method 속성이 GET, POST 밖에 없기 때문에-
    - 나머지 메소드는 서버와의 비동기 통신을 담당하는 AJAX에서 사용

> **GET, POST 둘 중에 무엇을 사용할까?**
**GET**은 서버에 존재하는 데이터를 조회하는 역할로만 사용
**POST**는 데이터의 상태를 변경하는 작업을 할 때 주로 사용
> 

## 요구사항 4 - 302 status code 적용

간단한 방법

```java
String url = tokens[1];
if ("/user/create".equals(url)) {
	[...]
	log.debug("User: {}", user);
	url = "/index.html";
}
```

- 요청 URL 값을 “/index.html” 로 변경하면 첫 화면을 보여줄 수 있음
    - 새로고침을 누르면 회원가입 요청을 다시 보내는 문제점이 발생
        
        → 브라우저가 이전 요청 정보를 유지하고 있기 때문에
        
    - 302 상태코드를 활용해서 해결 가능

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]

		String url = tokens[1];
		if ("/user/create".equals(url)) {
			String body = IOUtils.readData(br, contentLength);
      Map<String, String> params = HttpRequestUtils.parseQueryString(body);
      User user = new User(params.get("userId"), params.get("password"), params.get("name"), params.get("email"));
      log.debug("User: {}", user);
			DataOutputStream dos = new DataOutputStream(out);
      response302Header(dos, "/index.html");
		} 
		[...]
}

private void response302Header(DataOutputStream dos, String url) {
	try {
	  dos.writeBytes("HTTP/1.1 302 Redirect \r\n");
    dos.writeBytes("Location: " + url + "\r\n");
    dos.writeBytes("\r\n");
  } catch (IOException e) {
    log.error(e.getMessage());
  }
}
[...]
```

- 200 상태 코드를 이용하면 회원가입 요청을 처리한 후, index.html 파일을 읽어 응답을 보내는 방식으로
    
    클라이언트와 서버 간의 요청과 응답이 한 번씩만 발생
    
- 302 상태코드를 이용하면 회원가입 요청을 처리하고 /index.html로 302 응답하고,
    
    다시 /index.html 요청해서 파일을 읽은 뒤 200 응답을 하는 방식으로
    
    클라이언트와 서버 간의 요청과 응답이 두 번 발생
    

> **2XX** : 성공. 클라이언트가 요청한 동작을 수신하여 이해했고 승낙했으며 성공적으로 처리
**3XX** : 리다이렉션. 클라이언트는 요청을 마치기 위해 추가 동작이 필요함
**4XX** : 오류. 클라이언트에 오류가 있음
**5XX** : 서버 오류. 서버가 유효한 요청을 명백하게 수행하지 못했음
> 

## 요구사항 5 - 로그인하기

### **무상태 프로토콜**

HTTP는 클라이언트와 서버 간의 요청, 응답이 끝나면 연결을 끊는다.

(HTTP 1.1 부터 한번 맺은 연결을 재사용하긴 하지만, 각 요청간의 상태 데이터를 공유할 수는 없다.)

→ 따라서, 서버가 앞에서 클라이언트가 한 행동을 기억할 수 없다.

⇒ 이를 해결하기 위해 쿠키(Cookie)가 사용됨

### 쿠키

서버에서 요청에 대한 응답 헤더에 Set-Cookie로 결과 값을 저장해서 전송

클라이언트는 응답 헤더에 Set-Cookie가 존재하는 경우, Set-Cookie 값을 읽어서

서버에 보내는 요청 헤더의 Cookie 헤더 값으로 다시 전송

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]

		String url = tokens[1];
		if ("/user/create".equals(url)) {
			[...]
      User user = new User(params.get("userId"), params.get("password"), params.get("name"), params.get("email"));
      DataBase.addUser(user);
		} else if (url.equals("/user/login")) {
	    String body = IOUtils.readData(br, contentLength);
      Map<String, String> params = HttpRequestUtils.parseQueryString(body);
      User user = DataBase.findUserById(params.get("userId"));
      if (user == null) {
	      responseResource(out, "/user/login_failed.html");
      }
      if (user.getPassword().equals(params.get("password"))) {
	      DataOutputStream dos = new DataOutputStream(out);
        response302LoginSuccessHeader(dos);
      } else {
        responseResource(out, "/user/login_failed.html");
      }
    } else {
	    responseResource(out, url);
    }
		[...]
}

private void responseResource(OutputStream out, String url) throws IOException {
	DataOutputStream dos = new DataOutputStream(out);
  byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());
  response200Header(dos, body.length);
  responseBody(dos, body);
}

private void response302LoginSuccessHeader(DataOutputStream dos) {
	try {
	  dos.writeBytes("HTTP/1.1 302 Redirect \r\n");
    dos.writeBytes("Location: /index.html" + "\r\n");
    dos.writeBytes("Set-Cookie: logined=true \r\n");
    dos.writeBytes("\r\n");
  } catch (IOException e) {
    log.error(e.getMessage());
  }
}
[...]
```

회원가입한 사용자를 DataBase 클래스를 통해서 불러오고, 

로그인이 성공하면 응답 헤더에 Set-Cookie 헤더 값으로 logined=true를 전달

`Cookie: logined=true`

## 요구사항 6 - 사용자 목록 출력

Cookie의 헤더 값을 활용해 현재 요청을 보내고 있는 클라이언트가 로그인을 한 상태인지 판단

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]
		String[] tokens = line.split(" ");
		boolean logined = false;
    while (!line.equals("")) {
			 log.debug("header: {}", line);
       line = br.readLine();
       if (line.contains("Cookie")) {
	       logined = isLogin(line);
       }
		}
		
		String url = tokens[1];
		if ("/user/create".equals(url)) {
			[...]
		} else if (url.equals("/user/login")) {
			[...]
		} else if (url.equals("/user/list")) {
	    if (!logined) {
	      responseResource(out, "/user/login.html");
        return;
      }
      Collection<User> users = DataBase.findAll();
      StringBuilder sb = new StringBuilder();
      sb.append("<table border='1'>");
      for (User user : users) {
	      sb.append("<tr>");
        sb.append("<td>" + user.getUserId() + "</td>");
        sb.append("<td>" + user.getName() + "</td>");
        sb.append("<td>" + user.getEmail() + "</td>");
        sb.append("</tr>");
      }
      sb.append("</table>");
      byte[] body = sb.toString().getBytes();
      DataOutputStream dos = new DataOutputStream(out);
      response200Header(dos, body.length);
      responseBody(dos, body);
		}
[...]

private boolean isLogin(String line) {
	String[] headerTokens = line.split(":");
  Map<String, String> cookies = HttpRequestUtils.parseCookies(headerTokens[1].trim());
  String value = cookies.get("logined");
  if (value == null) {
		return false;
	}
  return Boolean.parseBoolean(value);
}
```

쿠키의 경우에는 서버가 전달하는 쿠키 정보를 클라이언트에 저장하기 때문에 보안 이슈가 있음.

⇒ 보안을 강화하기 위해 쿠키와 비슷하지만 서버에 저장하는 세션을 활용하기도 함

## 요구사항 7 - CSS 지원하기

브라우저는 응답 받은 후, Content-Type 헤더 값을 통해 응답 본문에 포함되어 있는 컨텐츠가 어떤 컨텐츠인지를 판단함.

요청 URL의 확장자가 css인 경우에 Content-Type 헤더 값을 text/css로 응답을 보내줘야 CSS가 제대로 적용

```java
public void run() {
	log.debug("New Client Connect! Connected IP : {}, Port : {}", connection.getInetAddress(),
	  connection.getPort());
	
	try (InputStream in = connection.getInputStream(); OutputStream out = connection.getOutputStream()) {
		[...]
	
		String url = tokens[1];
		if ("/user/create".equals(url)) {
			[...]
		} else if (url.endWith(".css")) {
			DataOutputStream dos = new DataOutputStream(out);
      byte[] body = Files.readAllBytes(new File("./webapp" + url).toPath());
      response200CssHeader(dos, body.length);
      responseBody(dos, body);
		}
		[...]
}

private void response200CssHeader(DataOutputStream dos, int lengthOfBodyContent) {
	try {
	  dos.writeBytes("HTTP/1.1 200 OK \r\n");
    dos.writeBytes("Content-Type: text/css\r\n");
    dos.writeBytes("Content-Length: " + lengthOfBodyContent + "\r\n");
    dos.writeBytes("\r\n");
	} catch (IOException e) {
    log.error(e.getMessage());
  }
}
```

**메타데이터**

: 실제 데이터가 아닌 데이터에 대한 정보를 담고 있는 헤더 정보들

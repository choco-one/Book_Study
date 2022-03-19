# 📚회원 데이터를 DB에 저장하기 실습

## 1. 실습 코드 리뷰 및 JDBC 복습

### 테이블 생성 jwp.sql 파일

```sql
DROP TABLE IF EXISTS USERS;

CREATE TABLE USERS ( 
	userId          varchar(12)		NOT NULL, 
	password		varchar(12)		NOT NULL,
	name			varchar(20)		NOT NULL,
	email			varchar(50),	
  	
	PRIMARY KEY               (userId)
);

INSERT INTO USERS VALUES('admin', 'password', '자바지기', 'admin@slipp.net');
```

- 회원 정보에 대한 테이블 생성 스크립트
- 서버가 시작하는 시점에 이 파일을 이용해 테이블 초기화
- `src/main/resources` 디렉토리 아래 `jwp.sql` 파일로 구현되어 있음

### 초기화를 위한 ContextLoaderListener 클래스

```java
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.core.io.ClassPathResource;
import org.springframework.jdbc.datasource.init.DatabasePopulatorUtils;
import org.springframework.jdbc.datasource.init.ResourceDatabasePopulator;

import core.jdbc.ConnectionManager;

@WebListener
public class ContextLoaderListener implements ServletContextListener {
    private static final Logger logger = LoggerFactory.getLogger(ContextLoaderListener.class);

    @Override
    public void contextInitialized(ServletContextEvent sce) {
        ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
        populator.addScript(new ClassPathResource("jwp.sql"));
        DatabasePopulatorUtils.execute(populator, ConnectionManager.getDataSource());

        logger.info("Completed Load ServletContext!");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {
    }
}
```

톰캣 서버가 시작할 때, 초기화할 수 있도록 `ContextLoaderListener` 클래스에 구현되어 있다.

- `ContextLoaderListener`가 `ServletContextListener` 인터페이스를 구현하고 있으며, `@WebListener` 애노테이션이 설정되어 있기 때문에 서블릿 컨테이너를 시작하는 과정에서 `contextInitialized()` 메소드를 호출해 초기화 작업을 진행한다.
- `ServletContextListener`에 대한 초기화는 서블릿 초기화보다 먼저 진행되기 때문에, 웹 애플리케이션 전체에 영향을 미치는 초기화가 필요한 경우에 활용할 수 있다.

### DAO(Data Access Object)

데이터베이스에 대한 접근 로직 처리를 담당하는 객체를 별도로 분리해 구현하는 패턴으로, 현재 일반적인 패턴으로 널리 활용되고 있다.

```java
import java.sql.Connection;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;

import core.jdbc.ConnectionManager;
import next.model.User;

public class UserDao {
    public void insert(User user) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = "INSERT INTO USERS VALUES (?, ?, ?, ?)";
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, user.getUserId());
            pstmt.setString(2, user.getPassword());
            pstmt.setString(3, user.getName());
            pstmt.setString(4, user.getEmail());

            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }

    public void update(User user) throws SQLException {
        // TODO 구현 필요함.
    }

    public List<User> findAll() throws SQLException {
        // TODO 구현 필요함.
        return new ArrayList<User>();
    }

    public User findByUserId(String userId) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = "SELECT userId, password, name, email FROM USERS WHERE userid=?";
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, userId);

            rs = pstmt.executeQuery();

            User user = null;
            if (rs.next()) {
                user = new User(rs.getString("userId"), rs.getString("password"), rs.getString("name"),
                        rs.getString("email"));
            }

            return user;
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (pstmt != null) {
                pstmt.close();
            }
            if (con != null) {
                con.close();
            }
        }
    }
}
```

- 사용자 데이터를 추가하고, 사용자 아이디에 해당하는 사용자 데이터를 조회하는 기능만 구현해놓은 `UserDao`
- 아래 `CreateUserController` 코드에서 보이는 것처럼 사용할 수 있다.

```java
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import core.db.DataBase;
import core.mvc.Controller;
import next.model.User;

public class CreateUserController implements Controller {
    private static final Logger log = LoggerFactory.getLogger(CreateUserController.class);

    @Override
    public String execute(HttpServletRequest req, HttpServletResponse resp) throws Exception {
        User user = new User(req.getParameter("userId"), req.getParameter("password"), req.getParameter("name"),
                req.getParameter("email"));
        log.debug("User : {}", user);

        DataBase.addUser(user);
        return "redirect:/";
    }
}
```

### 회원 목록 실습

회원 전체 목록을 조회하는 `findAll()` 메소드를 구현해야 함.

```java
public List<User> findAll() throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = "SELECT userId, password, name, email FROM USERS";
            pstmt = con.prepareStatement(sql);

            rs = pstmt.executeQuery();

            User user = null;
            ArrayList<User> users = new ArrayList<User>();
            if (rs.next()) {
                user = new User(rs.getString("userId"), rs.getString("password"), rs.getString("name"),
                        rs.getString("email"));
                users.add(user);
            }

            return users;
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (pstmt != null) {
                pstmt.close();
            }
            if (con != null) {
                con.close();
            }
        }
    }
```

### 개인정보 수정 실습

UPDATE SQL문을 활용해 개인정보 수정 기능 추가

```java
public void update(User user) throws SQLException {
        // TODO 구현 필요함.
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, user.getPassword());
            pstmt.setString(2, user.getName());
            pstmt.setString(3, user.getEmail());
            pstmt.setString(4, user.getUserId());

            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
```

---

## 2. DAO 리팩토링 실습

현재 UserDao 코드에는 많은 중복 코드가 존재하기 때문에 리팩토링이 필요하다.

⇒ 변화가 없는 부분(공통 라이브러리 부분), 변화가 발생하는 부분(개발자가 구현할 부분)으로 나눠야 함

⇒ SQL 쿼리, 쿼리에 전달할 인자, SELECT 구문의 경우 조회한 데이터를 추출하는 3가지만 구현하면 됨

---

SQLException으로 인해 생기는 불필요하게 많은 try/catch 절 때문에 소스코드의 가독성이 떨어지기 때문에 이 점도 리팩토링이 필요하다.

> ***컴파일타임 Exception과 런타임 Exception을 사용해야 되는 가이드라인***
- API를 사용하는 모든 곳에서 이 예외를 처리해야 하는가? 예외가 반드시 메소드에 대한 반환 값이 되어야 하는가? 
  → Yes? 컴파일타임 Exception을 사용
- API를 사용하는 소수 중 이 예외를 처리해야 하는가?
  → Yes? 런타임 Exception으로 구현 
  (API를 사용하는 모든 코드가 Exception을 catch하도록 강제하지 않는 것이 좋음)
- 무엇인가 큰 문제가 발생했는가? 이 문제를 복구할 방법이 없는가?
  → Yes? 런타임 Exception으로 구현
  (Exception을 catch하더라도 에러에 대한 정보를 통보 받는 것 외에 할 수 있는 것이 없음)
- 아직도 불명확한가?
  → Yes? 런타임 Exception으로 구현
  (Exception에 대해 문서화하고 API를 사용하는 곳에서 Exception에 대한 처리를 결정하도록)
> 

### 요구사항 1

INSERT, UPDATE 쿼리 메소드 중복 제거 작업

```java
public void insert(User user) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = createQueryForInsert();
            pstmt = con.prepareStatement(sql);
            setValuesForInsert(user, pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }

    private void setValuesForInsert(User user, PreparedStatement pstmt) throws SQLException {
        pstmt.setString(1, user.getUserId());
        pstmt.setString(2, user.getPassword());
        pstmt.setString(3, user.getName());
        pstmt.setString(4, user.getEmail());
    }

    private String createQueryForInsert(){
        return "INSERT INTO USERS VALUES (?, ?, ?, ?)";
    }
```

- Extract Method 리팩토링을 통해 위 코드처럼 변화가 발생하는 부분과 변화가 발생하지 않는 부분을 분리할 수 있다.
- 위 코드는 insert에 대한 Extract Method 리팩토링이고, update에 대한 부분도 같은 방식으로 진행

### 요구사항 2

분리한 메소드 중 변화가 발생하지 않는 부분(공통 라이브러리로 구현할 코드)을 새로운 클래스로 추가하고 이동

```java
public class InsertJdbcTemplate {
    public void insert(User user, UserDao userDao) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = userDao.createQueryForInsert();
            pstmt = con.prepareStatement(sql);
            userDao.setValuesForInsert(user, pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
}
```

- 공통 라이브러리로 구현하는 부분을 분리하기 위해 새로운 InsertJdbcTemplate 클래스를 추가하고,
- UserDao의 insert() 메소드를 InsertJdbcTemplate으로 이동

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        InsertJdbcTemplate jdbcTemplate = new InsertJdbcTemplate();
        jdbcTemplate.insert(user, this);
    }
[...]
```

- UserDao의 insert() 메소드가 InsertJdbcTemplate의 insert() 메소드를 호출하도록 변경하고,
- createQueryForInsert(), setValuesForInsert()는 InsertJdbcTemplate의 insert() 메소드가 접근 가능하도록 private에서 default로 접근제어자 변경
- update() 메소드도 같은 방식으로 리팩토링

> ❓ 왜 default??? public으로 하면 안되나??
→ default는 현재 패키지에서만 접근 가능하지만, public은 모든 패키지에서 접근 가능하다.
    필요없는 곳에서 쓰이지 않게, 안전하게 하기 위해 default 접근제어자를 사용한다고 생각
> 

### 요구사항 3

InsertJdbcTemplate과 UpdateJdbcTemplate의 UserDao에 대한 의존관계를 끊는다.

```java
public abstract class InsertJdbcTemplate {
    public void insert(User user) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = createQueryForInsert();
            pstmt = con.prepareStatement(sql);
            setValuesForInsert(user, pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
    
    abstract String createQueryForInsert();
    abstract void setValuesForInsert(User user, PreparedStatement pstmt) throws SQLException;
}
```

- InsertJdbcTemplate이 UserDao에 의존관계를 가지지 않도록 추상 클래스로 바꿔준다.
- createQueryForInsert(), setValuesForInsert() 메소드도 추상 메소드로 리팩토링해준다.

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        InsertJdbcTemplate jdbcTemplate = new InsertJdbcTemplate() {
            void setValuesForInsert(User user, PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }

            String createQueryForInsert(){
                return "INSERT INTO USERS VALUES (?, ?, ?, ?)";
            }
        };
        jdbcTemplate.insert(user);
    }
[...]
```

- InsertJdbcTemplate이 추상 클래스이기 때문에 바로 생성할 수 없다.
    
    > **추상 클래스의 인스턴스를 생성하는 방법**
    1. 상속하는 새로운 클래스를 추가
    2. 이름을 갖지 않는 익명 클래스 추가
    > 
- 익명 클래스를 사용해 UserDao의 insert() 메소드 구현
- 이와 같이 반복적으로 발생하는 중복 로직을 상위 클래스가 구현하고 변화가 발생하는 부분만 추상 메소드로 만들어 구현하도록 하는 디자인 패턴을 템플릿 메소드(Template Method) 패턴이라고 한다.
- 같은 방법으로 update도 리팩토링

### 요구사항 4

InsertJdbcTemplate과 UpdateJdbcTemplate 중 하나만 사용하도록 리팩토링

```java
public abstract class JdbcTemplate {
    public void update(User user) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = createQuery();
            pstmt = con.prepareStatement(sql);
            setValues(user, pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
    abstract String createQuery();
    abstract void setValues(User user, PreparedStatement pstmt) throws SQLException;
}
```

- 메소드 이름들을 createQuery(), setValues()로 Rename 리팩토링 진행
- 두 개의 클래스로 분리할 필요가 없어졌기 때문에 클래스 이름을 JdbcTemplate으로 바꾸고, 메소드 이름은 update()를 사용하도록 리팩토링

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(User user, PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }

            String createQuery(){
                return "INSERT INTO USERS VALUES (?, ?, ?, ?)";
            }
        };
        jdbcTemplate.update(user);
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(User user, PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getPassword());
                pstmt.setString(2, user.getName());
                pstmt.setString(3, user.getEmail());
                pstmt.setString(4, user.getUserId());
            }

            String createQuery(){
                return "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
            }
        };
        jdbcTemplate.update(user);
    }
[...]
```

- UserDao의 insert(), update() 메소드가 JdbcTemplate을 사용하도록 리팩토링
- InsertJdbcTemplate은 사용하는 곳이 없기 때문에 클래스 삭제

### 요구사항 5

JdbcTemplate과 User의 의존관계를 끊는다.

```java
public abstract class JdbcTemplate {
    public void update() throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = createQuery();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
    abstract String createQuery();
    abstract void setValues(PreparedStatement pstmt) throws SQLException;
}
```

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }

            String createQuery(){
                return "INSERT INTO USERS VALUES (?, ?, ?, ?)";
            }
        };
        jdbcTemplate.update();
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getPassword());
                pstmt.setString(2, user.getName());
                pstmt.setString(3, user.getEmail());
                pstmt.setString(4, user.getUserId());
            }

            String createQuery(){
                return "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
            }
        };
        jdbcTemplate.update();
    }
[...]
```

- user를 인자로 전달하지 않아도 되기 때문에 User에 대한 의존 관계를 모두 끊어준다.

```java
public abstract class JdbcTemplate {
    public void update() throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            String sql = createQuery();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
    public void update2(String sql) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }
    abstract String createQuery();
    abstract void setValues(PreparedStatement pstmt) throws SQLException;
}
```

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }

            String createQuery(){
                return "INSERT INTO USERS VALUES (?, ?, ?, ?)";
            }
        };
        String sql = "INSERT INTO USERS VALUES (?, ?, ?, ?)";
        jdbcTemplate.update2(sql);
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getPassword());
                pstmt.setString(2, user.getName());
                pstmt.setString(3, user.getEmail());
                pstmt.setString(4, user.getUserId());
            }

            String createQuery(){
                return "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
            }
        };
        String sql = "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
        jdbcTemplate.update2(sql);
    }
[...]
```

- SQL 쿼리는 굳이 추상 메소드를 통해 전달하지 않고, JdbcTemplate의 update() 메소드 인자로 전달하는 게 사용성 측면에서 더 좋을 것이다.
- 컴파일 에러가 발생하지 않도록 천천히 리팩토링을 하기 위해 기존 update() 메소드는 유지한 상태로 새로운 update2(String sql)을 추가한다.
- 오류가 없는 것을 확인하면 필요없는 update(), createQuery() 메소드를 삭제하고, update2() 메소드 이름을 update()로 Rename 리팩토링한다.

### 요구사항 6

JdbcTemplate과 같은 방법으로 SelectJdbcTemplate을 생성해 반복 코드를 분리

```java
public abstract class SelectJdbcTemplate {
    public List query(String sql) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);

            rs = pstmt.executeQuery();

            List<Object> result = new ArrayList<>();
            if (rs.next()) {
                result.add(mapRow(rs));
            }

            return result;
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (pstmt != null) {
                pstmt.close();
            }
            if (con != null) {
                con.close();
            }
        }
    }

    public Object queryForObject(String sql) throws SQLException {
        List result = query(sql);
        if (result.isEmpty()) {
            return null;
        }
        return result.get(0);
    }

    abstract void setValues(PreparedStatement pstmt) throws SQLException;
    abstract Object mapRow(ResultSet rs) throws SQLException;
}
```

- 조회한 데이터를 자바 객체로 변환하는 부분인 mapRow() 메소드를 추상 메소드로 추가해 구현
- userId로 조회하는 경우를 위해서 queryForObject() 메소드 구현

```java
public List<User> findAll() throws SQLException {
        SelectJdbcTemplate jdbcTemplate = new SelectJdbcTemplate() {
            @Override
            void setValues(PreparedStatement pstmt) throws SQLException {

            }

            @Override
            Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS";
        return (List<User>)jdbcTemplate.query(sql);
    }

    public User findByUserId(String userId) throws SQLException {
        SelectJdbcTemplate jdbcTemplate = new SelectJdbcTemplate() {
            @Override
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userId);
            }

            @Override
            Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS WHERE userId=?";
        return (User)jdbcTemplate.queryForObject(sql);
    }
```

- UserDao의 findAll(), findByUserId() 메소드들이 SelectJdbcTemplate을 사용하도록 구현

### 요구사항 7

JdbcTemplate과 SelectJdbcTemplate을 한 개의 클래스만을 제공하도록 리팩토링

```java
public abstract class JdbcTemplate {
    public void update(String sql) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }

    public List query(String sql) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            setValues(pstmt);

            rs = pstmt.executeQuery();

            List<Object> result = new ArrayList<>();
            if (rs.next()) {
                result.add(mapRow(rs));
            }

            return result;
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (pstmt != null) {
                pstmt.close();
            }
            if (con != null) {
                con.close();
            }
        }
    }

    public Object queryForObject(String sql) throws SQLException {
        List result = query(sql);
        if (result.isEmpty()) {
            return null;
        }
        return result.get(0);
    }
    abstract Object mapRow(ResultSet rs) throws SQLException;
    abstract void setValues(PreparedStatement pstmt) throws SQLException;
}
```

- JdbcTemplate과 SelectJdbcTemplate 클래스를 통합

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }
            Object mapRow(ResultSet rs) throws SQLException {
                return null;
            }
        };
        String sql = "INSERT INTO USERS VALUES (?, ?, ?, ?)";
        jdbcTemplate.update(sql);
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getPassword());
                pstmt.setString(2, user.getName());
                pstmt.setString(3, user.getEmail());
                pstmt.setString(4, user.getUserId());
            }
            Object mapRow(ResultSet rs) throws SQLException {
                return null;
            }
        };
        String sql = "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
        jdbcTemplate.update(sql);
    }

    public List<User> findAll() throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            @Override
            void setValues(PreparedStatement pstmt) throws SQLException {

            }

            @Override
            Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS";
        return (List<User>)jdbcTemplate.query(sql);
    }

    public User findByUserId(String userId) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate() {
            @Override
            void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userId);
            }

            @Override
            Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS WHERE userId=?";
        return (User)jdbcTemplate.queryForObject(sql);
    }
}
```

- UserDao의 insert(), update() 메소드에서 mapRow() 메소드를 구현하도록 수정해야하는 불필요한 문제점이 생김

### 요구사항 8

위와 같이 SelectJdbcTemplate 클래스로 통합했을 때의 문제점을 찾아보고 해결하기 위한 방법을 찾아본다.

```java
public interface PreparedStatementSetter {
    void setValues(PreparedStatement pstmt) throws SQLException;
}
```

```java
public interface RowMapper {
    Object mapRow(ResultSet rs) throws SQLException;
}
```

- 위 문제점 해결을 위해 각각의 추상 메소드를 인터페이스를 통해 분리

```java
public class JdbcTemplate {
    public void update(String sql, PreparedStatementSetter pss) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            pss.setValues(pstmt);
            pstmt.executeUpdate();
        } finally {
            if (pstmt != null) {
                pstmt.close();
            }

            if (con != null) {
                con.close();
            }
        }
    }

    public List query(String sql, PreparedStatementSetter pss, RowMapper rowMapper) throws SQLException {
        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            pss.setValues(pstmt);

            rs = pstmt.executeQuery();

            List<Object> result = new ArrayList<>();
            if (rs.next()) {
                result.add(rowMapper.mapRow(rs));
            }

            return result;
        } finally {
            if (rs != null) {
                rs.close();
            }
            if (pstmt != null) {
                pstmt.close();
            }
            if (con != null) {
                con.close();
            }
        }
    }

    public Object queryForObject(String sql, PreparedStatementSetter pss, RowMapper rowMapper) throws SQLException {
        List result = query(sql, pss, rowMapper);
        if (result.isEmpty()) {
            return null;
        }
        return result.get(0);
    }
}
```

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getUserId());
                pstmt.setString(2, user.getPassword());
                pstmt.setString(3, user.getName());
                pstmt.setString(4, user.getEmail());
            }
        };
        String sql = "INSERT INTO USERS VALUES (?, ?, ?, ?)";
        jdbcTemplate.update(sql, pss);
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, user.getPassword());
                pstmt.setString(2, user.getName());
                pstmt.setString(3, user.getEmail());
                pstmt.setString(4, user.getUserId());
            }
        };
        String sql = "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
        jdbcTemplate.update(sql, pss);
    }

    public List<User> findAll() throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {

            }
        };
        RowMapper rowMapper = new RowMapper() {
            @Override
            public Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS";
        return (List<User>)jdbcTemplate.query(sql, pss, rowMapper);
    }

    public User findByUserId(String userId) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userId);
            }
        };
        RowMapper rowMapper = new RowMapper() {
            @Override
            public Object mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS WHERE userId=?";
        return (User)jdbcTemplate.queryForObject(sql, pss, rowMapper);
    }
}
```

- JdbcTemplate과 UserDao에서도 PreparedStatementSetter, RowMapper 인터페이스를 활용하도록 리팩토링

> 왜? findAll을 위한 JdbcTemplate의 query 메소드에서 PreparedStatementSetter는 왜 필요????
⇒
> 

### 요구사항 9

SQLException을 런타임 Exception으로 변환해 throw하도록 한다. 

Connection, PreparedStatement 자원 반납을 close() 메소드를 사용하지 말고 try-with-resources 구문을 적용해 해결한다.

```java
public class DataAccessException extends RuntimeException{
    private static final long serialVersionUID = 1L;

    public DataAccessException() {
        super();
    }

    public DataAccessException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace) {
        super(message, cause, enableSuppression, writableStackTrace);
    }

    public DataAccessException(String message, Throwable cause) {
        super(message, cause);
    }

    public DataAccessException(String message) {
        super(message);
    }

    public DataAccessException(Throwable cause) {
        super(cause);
    }
}
```

- RuntimeException을 상속하는 새로운 Exception 추가

```java
public class JdbcTemplate {
    public void update(String sql, PreparedStatementSetter pss) throws DataAccessException {
        Connection con = null;
        PreparedStatement pstmt = null;
        try {
            con = ConnectionManager.getConnection();
            pstmt = con.prepareStatement(sql);
            pss.setValues(pstmt);
            pstmt.executeUpdate();
        } catch (SQLException e) {
            throw new DataAccessException(e);
        } finally {
            if (pstmt != null) {
                try {
                    pstmt.close();
                } catch (SQLException e) {
                    throw new DataAccessException(e);
                }
            }
            if (con != null) {
                try {
                    con.close();
                } catch (SQLException e) {
                    throw new DataAccessException(e);
                }

            }
        }
    }
[...]
```

- JdbcTemplate에서 위와 같이 리팩토링하여 SQLException을 처리하지 않게 바꿀 수 있음
- 하지만, finally 절의 복잡도가 너무 높아지는 문제점 생김

```java
public class JdbcTemplate {
    public void update(String sql, PreparedStatementSetter pss) throws DataAccessException {
        try (Connection conn = ConnectionManager.getConnection();
            PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pss.setValues(pstmt);
            pstmt.executeUpdate();
        } catch (SQLException e) {
            throw new DataAccessException(e);
        }
    }
[...]
```

- 자원을 활용한 후 반납하기 위한 목적으로 사용하는 close() 메소드 호출 부분들을 자바 7 버전부터 제공하는 `java.io.AutoClosable` 인터페이스를 통해 해결할 수 있다.
- `AutoClosable` 인터페이스를 구현하는 클래스는 try-with-resources 구문을 활용해 자원을 반납할 수 있다.

### 요구사항 10

SELECT문의 경우 조회한 데이터를 캐스팅하는 부분이 있다.

캐스팅하지 않고 구현하도록 개선

```java
public interface RowMapper<T> {
    T mapRow(ResultSet rs) throws SQLException;
}
```

- 제너릭 문법을 사용하도록 RowMapper 수정

```java
public <T> List<T> query(String sql, PreparedStatementSetter pss, RowMapper<T> rowMapper) throws SQLException {
        ResultSet rs = null;
        try (Connection con = ConnectionManager.getConnection();
             PreparedStatement pstmt = con.prepareStatement(sql)) {
            pss.setValues(pstmt);

            rs = pstmt.executeQuery();

            List<T> result = new ArrayList<>();
            if (rs.next()) {
                result.add(rowMapper.mapRow(rs));
            }

            return result;
        } finally {
            if (rs != null) {
                rs.close();
            }
        }
    }

    public <T> T queryForObject(String sql, PreparedStatementSetter pss, RowMapper<T> rowMapper) throws SQLException {
        List<T> result = query(sql, pss, rowMapper);
        if (result.isEmpty()) {
            return null;
        }
        return result.get(0);
    }
```

```java
public List<User> findAll() throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {

            }
        };
        RowMapper<User> rowMapper = new RowMapper<User>() {
            @Override
            public User mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS";
        return jdbcTemplate.query(sql, pss, rowMapper);
    }

    public User findByUserId(String userId) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userId);
            }
        };
        RowMapper<User> rowMapper = new RowMapper<User>() {
            @Override
            public User mapRow(ResultSet rs) throws SQLException {
                return new User(
                        rs.getString("userId"),
                        rs.getString("password"),
                        rs.getString("name"),
                        rs.getString("email"));
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS WHERE userId=?";
        return jdbcTemplate.queryForObject(sql, pss, rowMapper);
    }
```

- 제너릭을 사용하도록 JdbcTemplate, UserDao의 메소드들 수정

### 요구사항 11

각 쿼리에 전달할 인자를 PreparedStatementSetter를 통해 전달할 수도 있지만 자바의 가변인자를 통해 전달할 수 있는 메소드를 추가한다.

```java
public class JdbcTemplate {
    public void update(String sql, Object... parameters) throws DataAccessException {
        try (Connection conn = ConnectionManager.getConnection();
            PreparedStatement pstmt = conn.prepareStatement(sql)) {
            for (int i = 0; i < parameters.length; i++) {
                pstmt.setObject(i + 1, parameters[i]);
            }
            pstmt.executeUpdate();
        } catch (SQLException e) {
            throw new DataAccessException(e);
        }
    }
[...]
```

- 가변인자를 활용하도록 JdbcTemplate 수정

```java
public class UserDao {
    public void insert(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        String sql = "INSERT INTO USERS VALUES (?, ?, ?, ?)";
        jdbcTemplate.update(sql, user.getUserId(), user.getPassword(), user.getName(), user.getEmail());
    }

    public void update(User user) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        String sql = "UPDATE USERS SET password = ?, name = ?, email = ? WHERE userId = ?";
        jdbcTemplate.update(sql, user.getPassword(), user.getName(), user.getEmail(), user.getUserId());
    }
[...]
```

- UserDao도 가변인자 활용하도록 수정

### 요구사항 12

UserDao에서 PreparedStatementSetter, RowMapper 인터페이스를 구현하는 부분을 JDK 8에서 추가한 람다 표현식을 활용하도록 리팩토링

```java
public User findByUserId(String userId) throws SQLException {
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        PreparedStatementSetter pss = new PreparedStatementSetter() {
            @Override
            public void setValues(PreparedStatement pstmt) throws SQLException {
                pstmt.setString(1, userId);
            }
        };
        String sql = "SELECT userId, password, name, email FROM USERS WHERE userId=?";
        return jdbcTemplate.queryForObject(sql, pss, (ResultSet rs) -> {
            return new User(
                    rs.getString("userId"),
                    rs.getString("password"),
                    rs.getString("name"),
                    rs.getString("email"));
        });
 }
```

```java
@FunctionalInterface
public interface RowMapper<T> {
    T mapRow(ResultSet rs) throws SQLException;
}
```

- 람다 표현식을 사용하기 위해 인터페이스에 `@FunctionalInterface` 어노테이션 추가

# 10장

# MVC 프레임워크 요구사항 3단계

- 새로운 기능이 추가될 때마다 매번 컨트롤러를 추가하는 것이 아니라 메소드를 추가하는 방식
- 요청 URL을 매핑할 때 HTTP 메소드(GET, POST, PUT, DELETE 등)도 매핑에 활용할 수 있도록
- 어노테이션 기반으로 MVC 프레임워크를 구현한 후, 레거시 MVC 프레임워크와 어노테이션 기반의 새로운 MVC 프레임워크를 통합하는 방식으로 점진적인 리팩토링을 진행
- 자바 리플렉션을 활용해 @Controller 어노테이션이 설정되어 있는 클래스를 찾은 후, @RequestMapping 설정에 따라 요청 URL과 메소드를 연결하도록 구현할 수 있다.

## 1. 자바 리플렉션

### 1-1. 자바 리플렉션 API 활용해 클래스 정보 출력하기

자바 리플렉션을 활용해 Question 클래스의 모든 필드, 생성자, 메소드 정보를 출력한다.

```java
@Test
    public void showClass() {
        Class<Question> clazz = Question.class;
        Field[] fields = clazz.getDeclaredFields();
        Constructor[] constructors = clazz.getConstructors();
        Method[] methods = clazz.getMethods();
        logger.debug(clazz.getName());
        for (Field field : fields) {
            logger.debug(String.valueOf(field));
        }
        for (Constructor constructor : constructors) {
            logger.debug(String.valueOf(constructor));
        }
        for (Method method : methods) {
            logger.debug(String.valueOf(method));
        }
    }
```

- field의 경우에는 public이 아닌 private이기 때문에 getFields()가 아닌 getDeclaredFields()를 사용해서 필드 정보를 가져온다.

### 1-2. “test”로 시작하는 메소드 실행하기

자바 리플렉션을 활용해 core.ref.Junit3Test에서 test로 시작하는 모든 메소드를 실행

```java
@Test
    public void run() throws Exception {
        Class<Junit3Test> clazz = Junit3Test.class;
        Method[] methods = clazz.getMethods();
        for (Method method : methods) {
            if (method.getName().startsWith("test")) {
                method.invoke(clazz.newInstance());
            }
        }
    }
```

- startsWith()로 메소드 이름이 “test”로 시작하는지 확인한 후, method.invoke(clazz.newInstance())로 실행

### 1-3. @MyTest 어노테이션으로 설정된 메소드 실행하기

자바 리플렉션을 활용해 core.ref.Junit4Test에서 @MyTest 어노테이션이 설정되어 있는 모든 메소드를 실행

```java
@Test
    public void run() throws Exception {
        Class<Junit4Test> clazz = Junit4Test.class;
        Method[] methods = clazz.getMethods();
        for (Method method : methods) {
            if (method.isAnnotationPresent(MyTest.class)) {
                method.invoke(clazz.newInstance());
            }
        }
    }
```

- method.isAnnotationPresent(MyTest.class)를 이용해 @MyTest 어노테이션이 설정되어 있는 메소드를 찾아서 실행

### 1-4. 생성자가 있는 클래스의 인스턴스 생성하기

자바 리플렉션을 활용해 User 클래스의 인스턴스를 생성한다.

```java
@Test
    public void newInstanceWithConstructorArgs() throws InvocationTargetException, InstantiationException, IllegalAccessException {
        Class<User> clazz = User.class;
        logger.debug(clazz.getName());

        Constructor[] constructors = clazz.getDeclaredConstructors();
        for (Constructor constructor : constructors) {
            constructor.newInstance("test", "password", "test", "test@abc");
        }
    }
```

- 기본 생성자가 없는 경우, getDeclaredConstructors()를 통해 Constructor를 찾고
- constructor.newInstance(Object... args)로 인스턴스 생성

### 1-5. private 필드에 접근하기

리플렉션 API를 통해 클래스의 private 필드에 접근해 값을 전달할 수 있다.

```java
@Test
    public void privateFieldAccess() throws IllegalAccessException {
        Class<Student> clazz = Student.class;
        logger.debug(clazz.getName());

        Student student = new Student();
        Field[] fields = clazz.getDeclaredFields();
        for (Field field : fields) {
            field.setAccessible(true);
            if (String.valueOf(field.getType()).equals("int")) {
                field.setInt(student, 10);
            }
            else {
                field.set(student, "test");
            }
        }
    }
```

- Student 인스턴스를 직접 생성한 후 필드에 값을 할당하는 방식
- getDeclaredField 메소드를 이용해 private 필드를 찾고
- field.setAccessible(true)로 설정해서 private 필드에 접근
- field.set(student, “test”)로 private 필드에 값을 할당 / int 타입이면 field.setInt(student, 10)

---

## 2. MVC 프레임워크 구현 3단계

### 2-1. @Controller 어노테이션 설정 클래스 스캔

```java
public class ControllerScanner {
    private static final Logger log = LoggerFactory.getLogger(ControllerScanner.class);

    private Reflections reflections;

    public ControllerScanner(Object... basePackage) {
        reflections = new Reflections(basePackage);
    }

    public Map<Class<?>, Object> getControllers() {
        Set<Class<?>> preInitiatedControllers = reflections.getTypesAnnotatedWith(Controller.class);
        return instantiateControllers(preInitiatedControllers);
    }

    Map<Class<?>, Object> instantiateControllers(Set<Class<?>> preInitiatedControllers) {
        Map<Class<?>, Object> controllers = Maps.newHashMap();
        try {
            for (Class<?> clazz : preInitiatedControllers) {
                controllers.put(clazz, clazz.newInstance());
            }
        } catch (InstantiationException | IllegalAccessException e) {
            log.error(e.getMessage());
        }

        return controllers;
    }
}
```

- @Controller 어노테이션이 설정되어 있는 모든 클래스를 찾고,
- 찾은 클래스에 대한 인스턴스를 생성해 Map<Class<?>, Object>에 추가

### 2-2. @RequestMapping 어노테이션 설정을 활용한 매핑

어노테이션 기반 매핑을 담당할 AnnotationHandlerMapping 클래스를 추가한 후 초기화

```java
public class HandlerKey {
    private String url;
    private RequestMethod requestMethod;

    public HandlerKey(String url, RequestMethod requestMethod) {
        this.url = url;
        this.requestMethod = requestMethod;
    }

    @Override
    public String toString() {
        return "HandlerKey [url=" + url + ", requestMethod=" + requestMethod + "]";
    }
}
```

- @RequestMapping 어노테이션이 가지고 있는 URL과 HTTP 메소드 정보를 가짐

```java
public class HandlerExecution {
    private static final Logger logger = LoggerFactory.getLogger(HandlerExecution.class);

    private Object declaredObject;
    private Method method;

    public HandlerExecution(Object declaredObject, Method method) {
        this.declaredObject = declaredObject;
        this.method = method;
    }

    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response) throws Exception {
        try {
            return (ModelAndView) method.invoke(declaredObject, request, response);
        } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
            logger.error("{} method invoke fail. error message : {}", method, e.getMessage());
            throw new RuntimeException(e);
        }
    }
}
```

- HandlerExecution은 자바 리플렉션에서 메소드를 실행하기 위해 필요한 정보를 가진다.
- 실행할 메소드가 존재하는 클래스의 인스턴스 정보와 실행할 메소드 정보(java.reflect.Method)를 가져야 한다.

```java
public class AnnotationHandlerMapping {
    private static final Logger logger = LoggerFactory.getLogger(AnnotationHandlerMapping.class);

    private Object[] basePackage;

    private Map<HandlerKey, HandlerExecution> handlerExecutions = Maps.newHashMap();

    public AnnotationHandlerMapping(Object... basePackage) {
        this.basePackage = basePackage;
    }

    public void initialize() {
        ControllerScanner controllerScanner = new ControllerScanner(basePackage);
        Map<Class<?>, Object> controllers = controllerScanner.getControllers();
        Set<Method> methods = getRequestMappingMethods(controllers.keySet());
        for (Method method : methods) {
            RequestMapping rm = method.getAnnotation(RequestMapping.class);
            logger.debug("register handlerExecution : url is {}, method is {}", rm.value(), method);
            handlerExecutions.put(createHandlerKey(rm),
                    new HandlerExecution(controllers.get(method.getDeclaringClass()), method));
        }

        logger.info("Initialized AnnotationHandlerMapping!");
    }

    private HandlerKey createHandlerKey(RequestMapping rm) {
        return new HandlerKey(rm.value(), rm.method());
    }

    @SuppressWarnings("unchecked")
    private Set<Method> getRequestMappingMethods(Set<Class<?>> controlleers) {
        Set<Method> requestMappingMethods = Sets.newHashSet();
        for (Class<?> clazz : controlleers) {
            requestMappingMethods
                    .addAll(ReflectionUtils.getAllMethods(clazz, ReflectionUtils.withAnnotation(RequestMapping.class)));
        }
        return requestMappingMethods;
    }
}
```

- ControllerScanner를 통해 찾은 @Controller 클래스의 메소드 중 RequestMapping 어노테이션이 설정되어 있는 모든 메소드를 찾는다.
- Map<HandlerKey, HandlerExecution>을 통해 HandlerKey와 HandlerExecution을 연결한다.

### 2-3. 클라이언트 요청에 해당하는 HandlerExecution 반환

```java
public class AnnotationHandlerMapping {
    private static final Logger logger = LoggerFactory.getLogger(AnnotationHandlerMapping.class);

    private Object[] basePackage;

    private Map<HandlerKey, HandlerExecution> handlerExecutions = Maps.newHashMap();

    [...]

    public HandlerExecution getHandler(HttpServletRequest request) {
        String requestUri = request.getRequestURI();
        RequestMethod rm = RequestMethod.valueOf(request.getMethod().toUpperCase());
        logger.debug("requestUri : {}, requestMethod : {}", requestUri, rm);
        return handlerExecutions.get(new HandlerKey(requestUri, rm));
    }
}
```

- 클라이언트 요청 정보를 전달하면 요청에 해당하는 HandlerExecution을 반환하는 메소드 구현

### 2-4. DispatcherServlet과 AnnotationHandlerMapping 통합

```java
public interface HandlerMapping {
    Object getHandler(HttpServletRequest request);
}
```

- RequestMapping과 AnnotationHandlerMapping을 모두 지원하면서 통합 관리하기 위해 HandlerMapping 인터페이스 추가

```java
public class LegacyHandlerMapping implements HandlerMapping {
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);
    private Map<String, Controller> mappings = new HashMap<>();

        [...]

    @Override
    public Object getHandler(HttpServletRequest request) {
        return mappings.get(request.getRequestURI());
    }
}
```

- RequestMapping 클래스를 LegacyHandlerMapping으로 이름 변경한 후 HandlerMapping 인터페이스 구현
- AnnotationHandlerMapping도 HandlerMapping 인터페이스 구현

```java
@WebServlet(name = "dispatcher", urlPatterns = "/", loadOnStartup = 1)
public class DispatcherServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);

    private List<HandlerMapping> mappings = Lists.newArrayList();
    private List<HandlerMapping> handlerAdapters = Lists.newArrayList();

    @Override
    public void init() throws ServletException {
        LegacyHandlerMapping lhm = new LegacyHandlerMapping();
        lhm.initMapping();
        AnnotationHandlerMapping ahm = new AnnotationHandlerMapping("next.controller");
        ahm.initialize();

        mappings.add(lhm);
        mappings.add(ahm);
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        String requestUri = req.getRequestURI();
        logger.debug("Method : {}, Request URI : {}", req.getMethod(), requestUri);

        Object handler = getHandler(req);
        if (handler == null) {
            throw new IllegalArgumentException("존재하지 않는 URL입니다.");
        }

        try {
            ModelAndView mav = execute(handler, req, resp);
            if (mav != null) {
                View view = mav.getView();
                view.render(mav.getModel(), req, resp);
            }
        } catch (Throwable e) {
            logger.error("Exception : {}", e);
            throw new ServletException(e.getMessage());
        }
    }

    private Object getHandler(HttpServletRequest req) {
        for (HandlerMapping handlerMapping : mappings) {
            Object handler = handlerMapping.getHandler(req);
            if (handler != null) {
                return handler;
            }
        }
        return null;
    }

    private ModelAndView execute(Object handler, HttpServletRequest req, HttpServletResponse resp) throws Exception {
        if (handler instanceof Controller) {
            return ((Controller)handler).execute(req, resp);
        } else {
            return ((HandlerExecution)handler).handle(req, resp);
        }
    }
}
```

- HandlerMapping을 List로 관리하면서 요청 URL과 HTTP 메소드에 해당하는 컨트롤러를 찾아 컨트롤러가 존재할 경우 컨트롤러에게 작업을 위임하도록 구현

---

## 3. 인터페이스가 다른 경우 확장성 있는 설계

- 어플리케이션을 개발하다보면 역할은 같은데 서로 다른 인터페이스를 사용함으로써 통합하기 힘든 상황이 발생 → 좀 더 유연한 구조를 지원하려면 인터페이스 하나로 강제하는 것은 바람직하지 않음
- 서로 다른 인터페이스를 통합하다보면 캐스팅을 해야 하는 상황이 발생

```java
private ModelAndView execute(Object handler, HttpServletRequest req, HttpServletResponse resp) throws Exception {
        if (handler instanceof Controller) {
            return ((Controller)handler).execute(req, resp);
        } else {
            return ((HandlerExecution)handler).handle(req, resp);
        }
    }
```

- 새로운 컨트롤러 유형이 추가될 경우 else if 절이 추가되는 구조로 되어있음
- 메소드 구현 로직이 **컨트롤러의 인스턴스가 무엇인지를 판단하는 부분**과 **해당 컨트롤러로 캐스팅한 후 컨트롤러를 실행하는 부분**으로 나뉨

```java
public interface HandlerAdapter {
    boolean supports(Object handler);

    ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
}
```

- HandlerAdapter 인터페이스로 추상화

```java
public class ControllerHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof Controller;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        return ((Controller) handler).execute(request, response);
    }
}
```

```java
public class HandlerExecutionHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        return handler instanceof HandlerExecution;
    }

    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        return ((HandlerExecution) handler).handle(request, response);
    }
}
```

- 각 컨트롤러에 대한 HandlerAdapter를 구현

```java
@WebServlet(name = "dispatcher", urlPatterns = "/", loadOnStartup = 1)
public class DispatcherServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);

    private List<HandlerMapping> mappings = Lists.newArrayList();
    private List<HandlerAdapter> handlerAdapters = Lists.newArrayList();

    @Override
    public void init() throws ServletException {
        [...]
        handlerAdapters.add(new ControllerHandlerAdapter());
        handlerAdapters.add(new HandlerExecutionHandlerAdapter());
    }

	[...]

    private ModelAndView execute(Object handler, HttpServletRequest req, HttpServletResponse resp) throws Exception {
        for (HandlerAdapter handlerAdapter : handlerAdapters) {
            if (handlerAdapter.supports(handler)) {
                return handlerAdapter.handle(req, resp, handler);
            }
        }
        return null;
    }
}
```

- DispatcherServlet 리팩토링
- 새로운 컨트롤러가 추가되더라도 HandlerAdapter 구현체만 구현한 후 DispatcherServlet의 HandlerAdapter 목록에 추가하면 된다.

---

## 4. 배포 자동화를 위한 쉘 스크립트 개선

### 4-1. 심볼릭 링크 기반으로 배포하는 배포 스크립트

```bash
#!/bin/bash

REPOSITORIES_DIR=~/repositories/jwp-basic
TOMCAT_HOME=~/tomcat
RELEASE_DIR=~/releases/jwp-basic

cd $REPOSITORIES_DIR
pwd
git pull
mvn clean package

C_TIME=$(date +%s)

mv $REPOSITORIES_DIR/target/jwp-basic $RELEASE_DIR/$C_TIME
echo "deploy source $RELEASE_DIR/$C_TIME directory"

$TOMCAT_HOME/bin/shutdown.sh

rm -rf $TOMCAT_HOME/webapps/ROOT
ln -s $RELEASE_DIR/$C_TIME $TOMCAT_HOME/webapps/ROOT

$TOMCAT_HOME/bin/startup.sh

tail -500f $TOMCAT_HOME/logs/catalina.out
```

### 4-2. 원복 스크립트

```bash
#!/bin/bash

RELEASE_DIR=~/releases/jwp-basic
TOMCAT_HOME=~/tomcat

RELEASES=$(ls -1tr $RELEASE_DIR)
echo "releases : $RELEASES"
REVISIONS=(${RELEASES//\n/})

if [ "${#REVISIONS[@]}" -lt 2]; then
	echo "release source length more than 2"
else
	echo "rollback directory : ${REVISIONS[1]}"

	$TOMCAT_HOME/bin/shutdown.sh
	
	rm -rf $TOMCAT_HOME/webapps/ROOT
	ln -s $RELEASE_DIR/${REVISIONS[1]} $TOMCAT_HOME/webapps/ROOT
	
	$TOMCAT_HOME/bin/startup.sh
	
	tail -500f $TOMCAT_HOME/logs/catalina.out
fi
```

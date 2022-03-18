# **🎿 필드와 setter 메소드에 @Inject 기능 추가**
## **⛷ 요구사항**

`@Inject` 를 활용해 DI함으로써 빈 간의 의존관계를 쉽게 연결할 수 있다. 하지만 현재는 생성자를 통해서만 DI가 가능하다. 따라서 생성자 이외에 **필드, `setter` 메소드**를 통해서도 DI할 수 있도록 기능을 추가한다.

## **🏂 1단계 힌트 - 클래스 설계**

생성자, 필드, `setter` 메소드, 총 3가지의 경우의 수가 있다. 지금까지 여러 경우의 수가 있는 경우 **추상화 과정을 통해 인터페이스를 추가**한 후 각 경우에 대한 구현을 수행했다.

`BeanFactory` 의 역할은 빈을 추가하고, 조회하는 역할만 남기고 생성자를 활용해 DI를 하고 인스턴스 생성은 `ConstructorInjector` 라는 이름으로 분리한다.

- `Injector` 라는 인터페이스를 추가한다. 이는 생성자, 필드, `setter` 메소드 DI에 대한 추상 메소드로 `inject()` 를 가진다.
- 생성자, 필드, `setter` 메소드에 대한 DI를 담당하는 클래스를 구현한다. 이는 `Injector` 인터페이스를 구현한다.
    - 3개의 구현체를 구현하는 과정에서 발생하는 중복 코드들은 `Injector` 인터페이스와 구현체 사이에 `AbstractInjector` 와 같은 추상 클래스로 제거한다.

## **🪂 2단계 힌트**

생성자 구현체(`ConstructorInjector`)는 `@Inject` 가 설정되어 있는 생성자를 가지는 클래스를 찾는다.

- 생성자의 인자에 해당하는 빈이 `BeanFactory` 에 등록되어 있는지 확인해보고 등록되어 있지 않다면 빈 인스턴스를 생성하여 DI한다.

필드 구현체(`FieldInjector`)와 `setter` 메소드 구현체(`SetterInjector`) 또한 같은 방식으로 구현한다.

- 필드 구현체의 경우 클래스에 `@Injector`가 설정되어 있는 모든 필드를 찾는다.
- 필드 타입(클래스)에 해당하는 빈이 등록되어 있는지 확인해보고 등록되어 있지 않다면 빈 인스턴스를 생성하여 DI한다.

## **🏋️‍♀️ 중복 제거를 위한 힌트**

`@Injector` 가 설정되어 있는 위치가 다르다는 점을 제외하면, 3개의 구현체의 로직 처리 과정은 동일하다.

- 이와 같이 로직이 같은 경우 **템플릿 메소드 패턴**을 적용할 수 있다.
- 부모 클래스인 `AbstractInjector` 는 로직 구현을 담당하고, 3개의 하위 클래스(구현체)는 각 클래스마다 다른 부분만을 구현한다.

---

# **🏋️ 필드와 setter 메소드 @Inject 구현**

`ConstructorInjector` 는 이전에 구현한 `BeanFactory` 의 `instantiateClass()` , `instantiateConstructor()` 메소드와 같다.

- 다른 점은 빈을 저장하기 위한 `Map` 을 직접 가지지 않고 `BeanFactory` 를 통해 접근하도록 한다는 것이다.
- `ConstructorInjector` 의 `instantiateClass()` 는 앞으로 추가할 `FieldInjector` , `SetterInjector` 도 필요하므로, 이들의 중복 제거를 위한 클래스인 `AbstractInjector` 에서 구현하도록 한다.

```java
public abstract class AbstractInjector implements Injector {
    private BeanFactory beanFactory;

    public AbstractInjector(BeanFactory beanFactory) {
        this.beanFactory = beanFactory;
    }

    @Override
    public void inject(Class<?> clazz) {
        instantiateClass(clazz);
        Set<?> injectedBeans = getInjectedBeans(clazz);
        for (Object injectedBean : injectedBeans) {
            Class<?> beanClass = getBeanClass(injectedBean);
            inject(injectedBean, instantiateClass(beanClass), beanFactory);
        }
    }

    abstract Set<?> getInjectedBeans(Class<?> clazz);

    abstract Class<?> getBeanClass(Object injectedBean);

    abstract void inject(Object injectedBean, Object bean, BeanFactory beanFactory);

    private Object instantiateClass(Class<?> clazz) {
        Class<?> concreteClass = findBeanClass(clazz, beanFactory.getPreInstanticateBeans());
        Object bean = beanFactory.getBean(concreteClass);
        if (bean != null) {
            return bean;
        }

        Constructor<?> injectedConstructor = BeanFactoryUtils.getInjectedConstructor(concreteClass);

        if (injectedConstructor == null) {
            bean = BeanUtils.instantiate(concreteClass);
            beanFactory.registerBean(concreteClass, bean);
            return bean;
        }

        bean = instantiateConstructor(injectedConstructor);
        beanFactory.registerBean(concreteClass, bean);
        return bean;
    }

    private Object instantiateConstructor(Constructor<?> constructor) {
        Class<?>[] pTypes = constructor.getParameterTypes();
        List<Object> args = Lists.newArrayList();

        for (Class<?> clazz : pTypes) {
            Class<?> concreteClazz = BeanFactoryUtils.findConcreteClass(clazz, beanFactory.getPreInstanticateBeans());
            Object bean = beanFactory.getBean(concreteClazz);
            if (bean == null) {
                bean = instantiateClass(concreteClazz);
            }
            args.add(bean);
        }
        return BeanUtils.instantiateClass(constructor, args.toArray());
    }

    private Class<?> findBeanClass(Class<?> clazz, Set<Class<?>> preInstanticateBeans) {
        Class<?> concreteClazz = BeanFactoryUtils.findConcreteClass(clazz, preInstanticateBeans);
        if (!preInstanticateBeans.contains(concreteClazz)) {
            throw new IllegalStateException(clazz + "는 Bean이 아니다.");
        }
        return concreteClazz;
    }
}
```

- `BeanFactory` 의 `private` 자원들에 대한 접근을 위해 `BeanFactory` 에 선언한 `getBean()` , `getPreInstanticateBeans()` , `registerBean()` 메소드를 활용한다.

```java
public class ConstructorInjector extends AbstractInjector {
    public ConstructorInjector(BeanFactory beanFactory) {
        super(beanFactory);
    }

    @Override
    Set<?> getInjectedBeans(Class<?> clazz) {
        return Sets.newHashSet();
    }

    @Override
    Class<?> getBeanClass(Object injectedBean) {
        return null;
    }

    @Override
    void inject(Object injectedBean, Object bean, BeanFactory beanFactory) {
    }
}
```

- `ConstructorInjector` 는 구현 로직이 모두 `AbstractInjector` 로 이동했으므로 간단하다.

```java
public class FieldInjector extends AbstractInjector {
    private static final Logger log = LoggerFactory.getLogger(FieldInjector.class);

    public FieldInjector(BeanFactory beanFactory) {
        super(beanFactory);
    }

    @Override
    Set<?> getInjectedBeans(Class<?> clazz) {
        return BeanFactoryUtils.getInjectedFields(clazz);
    }

    @Override
    Class<?> getBeanClass(Object injectedBean) {
        Field field = (Field)injectedBean;
        return field.getType();
    }

    @Override
    void inject(Object injectedBean, Object bean, BeanFactory beanFactory) {
        Field field = (Field) injectedBean;
        try {
            field.setAccessible(true);
            field.set(beanFactory.getBean(field.getDeclaringClass()), bean);
        } catch (IllegalAccessException | IllegalArgumentException e) {
            log.error(e.getMessage());
        }
    }
}
```

```java
public class SetterInjector extends AbstractInjector {
    private static final Logger log = LoggerFactory.getLogger(SetterInjector.class);

    public SetterInjector(BeanFactory beanFactory) {
        super(beanFactory);
    }

    @Override
    Set<?> getInjectedBeans(Class<?> clazz) {
        return BeanFactoryUtils.getInjectedMethods(clazz);
    }

    @Override
    Class<?> getBeanClass(Object injectedBean) {
        Method method = (Method) injectedBean;
        Class<?>[] paramTypes = method.getParameterTypes();
        if (paramTypes.length != 1) {
            throw new IllegalStateException("DI할 메소드 인자는 하나여야 합니다.");
        }
        return paramTypes[0];
    }

    @Override
    void inject(Object injectedBean, Object bean, BeanFactory beanFactory) {
        Method method = (Method) injectedBean;
        try {
            method.invoke(beanFactory.getBean(method.getDeclaringClass()), bean);
        } catch (IllegalAccessException | IllegalArgumentException | InvocationTargetException e) {
            log.error(e.getMessage());
        }
    }
}
```

템플릿 메소드 패턴을 적용해 로직 중복을 제거하여 하위 클래스의 구현이 매우 간단해졌다. 이제 `BeanFactory` 가 방금까지 구현한 3개의 `Injector` 를 사용하도록 리팩토링한다.

```java
public class BeanFactory {
    private static final Logger logger = LoggerFactory.getLogger(BeanFactory.class);

    private Set<Class<?>> preInstanticateBeans;

    private Map<Class<?>, Object> beans = Maps.newHashMap();

    private List<Injector> injectors;

    public BeanFactory(Set<Class<?>> preInstanticateBeans) {
        this.preInstanticateBeans = preInstanticateBeans;

        injectors = Arrays.asList(
                new FieldInjector(this),
                new SetterInjector(this),
                new ConstructorInjector(this)
        );
    }

    public void initialize() {
        for (Class<?> clazz : preInstanticateBeans) {
            if (beans.get(clazz) == null) {
                logger.debug("instantiated Class : {}", clazz);
                inject(clazz);
            }
        }
    }

    private void inject(Class<?> clazz) {
        for (Injector injector : injectors) {
            injector.inject(clazz);
        }
    }
    ...

}
```

- `BeanFactory` 생성자 호출 시, 구현한 `Injector` 들을 초기화해주고, `initialize()` 메소드 호출로, 초기화한 `Injector` 들에 대한 DI를 수행한다.

---

# **🏋️‍♂️ @Inject 개선**

`BeanFactory` 와 `AbstractInjector` 코드를 보면, 중복을 제거하기는 했지만 소스 코드의 이해도가 떨어진다. 또한, 현재 모든 빈과 관련한 정보는 `BeanFactory` 가 관리하는데, 빈 인스턴스 생성과 주입을 `Injector` 구현 클래스가 담당하는 구조여서 `BeanFactory` 에게 일을 시키는 것이 아닌 **빈 정보 조회**가 계속 발생한다.

- 즉, `BeanFactory` 객체의 활용도가 떨어지는 구조이다.
- 활용도를 높이기 위해 빈 인스턴스 생성과 주입 작업은 `BeanFactory` 가, 현재 빈 클래스의 상태 정보를 별도의 클래스로 추상화(`BeanDefinition`)해 관리하게끔 한다.
- 설정된 annotation에 있는 클래스를 조회하는 역할을 하는 `BeanScanner` 클래스의 이름도 `ClasspathBeanDefinitionScanner` 로 rename한다.
    - 이는 annotation이 설정된 클래스를 조회한 후 `BeanDefinition` 을 생성해 `BeanFactory` 에 전달한다. 이때, `ClasspathBeanDefinitionScanner` 와 `BeanFactory` 간 강한 의존관계를 가지지 않도록 설계한다. 
    (`BeanDefinition` 을 `BeanFactory` 에 전달하는 부분에서만 발생하도록 구현, `BeanDefinition` 을 저장하는 인터페이스(`BeanDefinitionRegistry`)를 추가하여 의존관계를 느슨하게!)
    - 즉, `BeanFactory` 는 **`BeanDefinition` 을 저장하는 저장소 역할**과 **`BeanDefinition` 을 활용해 빈 인스턴스 생성, 의존관계 주입을 담당하는 역할**로 나뉜다.

```java
public interface BeanDefinitionRegistry {
    void registerBeanDefinition(Class<?> clazz, BeanDefinition beanDefinition);
}
```

```java
public class ClasspathBeanDefinitionScanner {
    private final BeanDefinitionRegistry beanDefinitionRegistry;

    public ClasspathBeanDefinitionScanner(BeanDefinitionRegistry beanDefinitionRegistry) {
        this.beanDefinitionRegistry = beanDefinitionRegistry;
    }

    @SuppressWarnings("unchecked")
    public void doScan(Object... basePackages) {
        Reflections reflections = new Reflections(basePackages);
        Set<Class<?>> beanClasses = getTypesAnnotatedWith(reflections, Controller.class, Service.class, Repository.class);
        for (Class<?> clazz : beanClasses) {
            beanDefinitionRegistry.registerBeanDefinition(clazz, new BeanDefinition(clazz));
        }
    }

    @SuppressWarnings("unchecked")
    private Set<Class<?>> getTypesAnnotatedWith(Reflections reflections, Class<? extends Annotation>... annotations) {
        Set<Class<?>> preInstantiatedBeans = Sets.newHashSet();
        for (Class<? extends Annotation> annotation : annotations) {
            preInstantiatedBeans.addAll(reflections.getTypesAnnotatedWith(annotation));
        }
        return preInstantiatedBeans;
    }
}
```

- 이로써, 각 객체의 역할을 분리했다.
    - `ClasspathBeanDefinitionScanner` 는 클래스패스에서 빈을 조회(`getTypesAnnotatedWith()`)하는 역할을 담당하고,
    - 조회한 빈 정보를 `BeanDefinition` 을 생성해 `beanDefinitionRegistry` 에 전달(`beanDefinitionRegistry.registerBeanDefinition(clazz, new BeanDefinition(clazz));`)
    - `BeanDefinitionRegistry` 구현체가 `BeanDefinition` 의 저장소 역할을 담당한다.

`BeanFactory` 는 빈 인스턴스를 생성하고, 의존관계 주입을 위해서는 `BeanDefinition` 정보가 필요하다.

- 이는 `BeanFactory` 가 `BeanDefinitionRegistry` 구현체로 `BeanDefinition` 정보를 관리하도록 구현한다.

```java
public class BeanFactory implements BeanDefinitionRegistry {
    private static final Logger logger = LoggerFactory.getLogger(BeanFactory.class);
    ...

    private Map<Class<?>, BeanDefinition> beanDefinitions = Maps.newHashMap();
    ...

    public void initialize() {
        for (Class<?> clazz : getBeanClasses()) {
            getBean(clazz);
        }
    }

    public Set<Class<?>> getBeanClasses() {
        return beanDefinitions.keySet();
    }

    @Override
    public void registerBeanDefinition(Class<?> clazz, BeanDefinition beanDefinition) {
        logger.debug("register bean : {]", clazz);
        beanDefinitions.put(clazz, beanDefinition);
    }
}
```

`BeanFactory` 와 `ClasspathBeanDefinitionScanner` 의 의존관계에 대한 연결은 이 두 클래스를 활용하는 곳에서 담당한다. 지금까지는 `AnnotationHandlerMapping` 이 이를 담당한다.

```java
public class AnnotationHandlerMapping implements HandlerMapping {
    private static final Logger logger = LoggerFactory.getLogger(AnnotationHandlerMapping.class);

    private Object[] basePackage;

    private Map<HandlerKey, HandlerExecution> handlerExecutions = Maps.newHashMap();

    public AnnotationHandlerMapping(Object... basePackage) {
        this.basePackage = basePackage;
    }

    public void initialize() {
        BeanFactory beanFactory = new BeanFactory();
        ClasspathBeanDefinitionScanner scanner = new ClasspathBeanDefinitionScanner(beanFactory);
        scanner.doScan(basePackage);
        beanFactory.initialize();
        ...

    }
    ...

}
```

`BeanFactory` 와 `ClasspathBeanDefinitionScanner` 의 의존관계 또한 DI를 활용해 구현하여 유연성을 확보했다.

- `ClasspathBeanDefinitionScanner` 는 단순히 클래스 패스에서 빈을 조회한 후 `BeanDefinition` 을 생성해 `BeanDefinitionRegistry` 의 `registerBeanDefinition()` 메소드로 전달하는 역할을 담당한다.
- 하지만, 이 경우 **객체 간의 DI를 담당할 코드가 필요**하다는 단점이 있다.
    - 객체 간의 의존관계를 연결하지 않은 채로 API를 제공하면, 이를 사용하는 개발자는 일일이 DI를 해줘야 한다.
- 이러한 단점 보완을 위해 **객체 간의 DI를 담당하는 새로운 객체를 추가**한다. (`ApplicationContext`)

```java
public class ApplicationContext {
    private BeanFactory beanFactory;
    
    public ApplicationContext(Object... basePackages) {
        beanFactory = new BeanFactory();
        ClasspathBeanDefinitionScanner scanner = new ClasspathBeanDefinitionScanner(beanFactory);
        scanner.doScan(basePackages);
        beanFactory.initialize();
    }
    
    public <T> T getBean(Class<T> clazz) {
        return beanFactory.getBean(clazz);
    }
    
    public Set<Class<?>> getBeanClasses() {
        return beanFactory.getBeanClasses();
    }
}
```

- `BeanFactory` 초기화 과정을 `ApplicationContext` 로 옮기고, `BeanFactory` 로 직접 접근하던 API를 `ApplicationContext` 를 통해 접근하도록 리팩토링했다.
    - 이때, `BeanFactory` 가 담당하던 `getControllers()` 메소드의 역할이 적합하지 않아 이를 `AnnotationHandlerMapping` 으로 이동하고 구현을 위해 필요한 기능만 `ApplicationContext` 를 통해 접근하도록 구현했다.

```java
public class AnnotationHandlerMapping implements HandlerMapping {
    ...

    public void initialize() {
        ApplicationContext ac = new ApplicationContext(basePackage);
        Map<Class<?>, Object> controllers = getControllers(ac);
        ...

    }

    private Map<Class<?>, Object> getControllers(ApplicationContext ac) {
        Map<Class<?>, Object> controllers = Maps.newHashMap();
        for (Class<?> clazz : ac.getBeanClasses()) {
            Annotation annotation = clazz.getAnnotation(Controller.class);
            if (annotation != null) {
                controllers.put(clazz, ac.getBean(clazz));
            }
        }
        return controllers;
    }
}
```

- 객체 간의 의존관계 연결을 담당하도록 구현하여 더 이상 `BeanFactory` 를 사용하는 개발자가 의존관계때문에 머리 아파하지 않아도 된다.

빈 클래스 정보를 담고 있는 `BeanDefinition` 은 생성자로 전달되는 클래스에서 `@Inject` 가 어떻게 설정되어 있는지에 따라 `InjectType` 을 결정한다. (`enum` 사용)

- 이때, `InjectType` 에서는 필드와 `setter` 메소드를 활용한 `@Inject` 는 동일하게 취급하는데, 이는 내부 구현의 차이가 없기 때문이다.

빈 클래스 정보를 활용해 빈 인스턴스를 생성하고 의존관계 주입을 담당하는 `BeanFactory` 는 빈 클래스의 의존관계와 관련한 정보 처리를 모두 `BeanDefinition` 에게 위임하여 담당할 책임이 줄었다.

---

# **🤼‍♀️ 설정 추가를 통한 유연성 확보**

지금까지 구현한 코드를 비판적인 시각으로 분석했을때, 아래와 같은 개선사항들이 보인다.

- `DispatcherServlet` 의 `init()` 메소드를 보면 `"next"` 라는 패키지 이름이 하드 코딩되어 있다. 기본 패키지명을 외부에서 전달할 수 있도록 구현한다.
- `JdbcTemplate` 이 아직까지 싱글톤 패턴으로 구현되어 있다. 이를 DI 프레임워크의 빈으로 등록해 관리함으로써, 싱글톤 패턴을 제거한다.
    - 이때, 각 레이어에 명확히 일치하지 않는 빈을 지원하는 annotation을 추가할 필요가 있다.
- DB Connection을 생성하는 부분도 `static` 으로 구현되어 있다. 또한 DB 설정 정보도 하드 코딩으로 관리되어 특정 DB에 종속되는 구조이다. 이러한 구조를 벗어나고, **Connection Pooling을** 지원하기 위해 `Connection` 대신 `javax.sql.DataSource` 인터페이스에 의존관계를 가지도록 개선한다.

세번째 이슈는 `DataSource` 구현체로 Apache Commons에서 제공하는 DBCP 라이브러리를 활용해 DB 설정한 후 빈으로 등록하면 해결이 가능하다.

- 지금까지는 모두 직접 구현한 클래스를 빈으로 등록했기에 annotation 설정이 가능했다.
- 하지만 DBCP 구현체의 경우 외부 라이브러리이기때문에 수정이 불가능하다. 외부 라이브러리 사용을 위해 구현체를 상속해 빈 설정을 하면 해결이 가능하지만, 매번 이러한 과정을 거치는 것은 효율적이지 못하다.
- 따라서, 첫번째와 두번째 이슈부터 개선하도록 한다.

## **🤼 ServletContainerInitializer를 활용해 web.xml 없이 웹 개발하기**

- **첫번째 이슈**

: 서블릿에 인자를 전달함으로써 해결이 가능하다.

서블릿 컨테이너의 경우, `web.xml` 설정을 통해 서블릿에 인자를 전달할 수 있다. 하지만 서블릿 3.0 이후 버전에서는 `web.xml` 을 사용하지 않고도 설정이 가능한데, 이는 `ServletContainerInitializer` 인터페이스를 구현함으로써 가능하다.

```java
public interface WebApplicationInitializer {
    void onStartup(ServletContext servletContext) throws ServletException;
}
```

위 인터페이스에 대한 구현체를 구현한다. (`MyServletContainerInitializer`)

```java
@HandlesTypes(WebApplicationInitializer.class)
public class MyServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext) throws ServletException {
        List<WebApplicationInitializer> initializers = new LinkedList<>();

        if (webAppInitializerClasses != null) {
            for (Class<?> waiClass : webAppInitializerClasses) {
                try {
                    initializers.add((WebApplicationInitializer) waiClass.newInstance());
                } catch (Throwable ex) {
                    throw new ServletException("WebApplicationInitializer class를 인스턴스화하는데 실패했습니다.", ex);
                }
            }
        }
        
        if (initializers.isEmpty()) {
            servletContext.log("classpath에서 감지된 Spring WebApplicationInitializer types가 없습니다.");
            return;
        }
        
        for(WebApplicationInitializer initializer : initializers) {
            initializer.onStartup(servletContext);
        }
    }
}
```

- 서블릿 컨테이너는 클래스패스에 존재하는 클래스 중 `WebApplicationInitializer` 인터페이스를 구현하는 모든 구현체를 찾아 `MyServletContainerInitializer` 의 `onStartup()` 메소드 인자로 전달한다.
- 해당 메소드는 서블릿 컨테이너에 전달한 `WebApplicationInitializer` 의 `onStartup()` 메소드를 실행한다.

> `WebApplicationInitializer` 인터페이스를 구현함으로써 서블릿 컨테이너의 초기화 과정을 확장할 수 있게 되었다.
> 

`MyServletContainerInitializer` 의 `onStartup()` 메소드 실행을 위해서는 서블릿 컨테이너가 이 클래스를 인식하는 것이 먼저이다. 이를 위한 설정은 클래스패스의 `META-INF/services` 에 `javax.servlet.ServletContainerInitializer` 파일을 생성해, `core.web.MyServletContainerInitializer` 을 추가한다.

이제 `web.xml` 없이도 개발이 가능한 상태가 되었으니, 하드코딩으로 구현되어 있던 패키지 이름을 `WebApplicationInitializer` 에 대한 구현체를 만들어 구현 가능하다.

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {
    private static final Logger log = LoggerFactory.getLogger(MyWebApplicationInitializer.class);

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet("next"));
        dispatcher.setLoadOnStartup(1);
        dispatcher.addMapping("/");

        log.info("Start MyWebApplication Initializer");
    }
}
```

- `MyWebApplicationInitializer` 에서 `DispatcherServlet` 을 직접 생성해 등록(`LoadOnStartup` , `Mapping` 속성 또한 등록) `DispatcherServlet` 의 `@WebServlet` annotation 설정은 더이상 필요없다.

```java
public class DispatcherServlet extends HttpServlet {
    private static final long serialVersionUID = 1L;
    private static final Logger logger = LoggerFactory.getLogger(DispatcherServlet.class);

    private List<HandlerMapping> mappings = Lists.newArrayList();
    private List<HandlerAdapter> handlerAdapters = Lists.newArrayList();
    private Object[] basePackages;
    
    public DispatcherServlet(Object... basePackages) {
        this.basePackages = basePackages;
    }
    
    @Override
    public void init() throws ServletException {
        AnnotationHandlerMapping ahm = new AnnotationHandlerMapping(basePackages);
        ahm.initialize();
        mappings.add(ahm);
        handlerAdapters.add(new HandlerExecutionHandlerAdapter());
    }
    ...

}
```

- `init()` 메소드에서 `AnnotationHandlerMapping` 과 직접적인 의존관계를 가지는 것을 확인할 수 있다. 이또한 DI를 통해 의존관계를 느슨하게 가지도록 변경해 확장 가능하도록 개선할 수 있다.
    - 생성자에 `HandlerMapping` 을 인자로 전달할 수 있도록 리팩토링한다.

```java
public class DispatcherServlet extends HttpServlet {
    ...

    private HandlerMapping hm;

    public DispatcherServlet(HandlerMapping hm) {
        this.hm = hm;
    }

    @Override
    public void init() throws ServletException {
        mappings.add(hm);
        handlerAdapters.add(new HandlerExecutionHandlerAdapter());
    }
    ...

}
```

- `DispatcherServlet` 이 어느 `HandlerMapping` 구현체와 의존관계를 가질 것인지는 `DispatcherServlet` 를 생성하는 `MyApplicationInitializer` 가 아래와 같이 결정한다.
    
    ```java
    public class MyWebApplicationInitializer implements WebApplicationInitializer {
        private static final Logger log = LoggerFactory.getLogger(MyWebApplicationInitializer.class);
    
        @Override
        public void onStartup(ServletContext servletContext) throws ServletException {
            AnnotationHandlerMapping ahm = new AnnotationHandlerMapping("next");
            ahm.initialize();
            ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet(ahm));
            ...
    
        }
    }
    ```
    

## **🤼‍♂️ @Component annotation 지원**

**두번째 이슈**

: `ClasspathBeanDefinition` 클래스에 `@Component` annotation 설정만 추가하면 해결이 가능하다.

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Component {
    String value() default "";
}
```

이제 추가한 annotation을 `ClasspathBeanDefinitionScanner` 의 `doScan()` 메소드로 찾을 수 있도록 추가한다.

```java
public class ClasspathBeanDefinitionScanner {
    public void doScan(Object... basePackages) {
        Reflections reflections = new Reflections(basePackages);
        Set<Class<?>> beanClasses = getTypesAnnotatedWith(reflections, Controller.class, Service.class, Repository.class, Component.class);
        for (Class<?> clazz : beanClasses) {
            beanDefinitionRegistry.registerBeanDefinition(clazz, new BeanDefinition(clazz));
        }
    }
}
```

이제 추가한 annotation을 활용해 `JdbcTemplate` 을 빈으로 등록 가능하도록 설정 후 각 DAO에서 `JdbcTemplate` 을 DI로 사용하도록 리팩토링한다.

```java
@Component
public class JdbcTemplate {
    ...

}

@Repository
public class JdbcQuestionDao implements QuestionDao {
    private JdbcTemplate jdbcTemplate;
    
    @Inject
    public JdbcQuestionDao(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    ...

}

@Repository
public class JdbcAnswerDao implements AnswerDao {
    private JdbcTemplate jdbcTemplate;

    @Inject
    public JdbcAnswerDao(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    ...

}
```

서버를 시작하면 `JdbcTemplate` 이 `"next"` 패키지가 아닌 `"core"` 패키지 아래에 있어 인식하지 못해 에러가 발생한다.

- `MyWebApplicationInitializer` 에서 이를 인식하도록 `AnnotationHandlerMapping` 에 `"core"` 를 추가해 전달한다.

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {
    private static final Logger log = LoggerFactory.getLogger(MyWebApplicationInitializer.class);

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        AnnotationHandlerMapping ahm = new AnnotationHandlerMapping("core", "next");
        ...

    }
}
```

---

# 5. 외부 라이브러리 클래스를 빈으로 등록하기

## 5-1. 요구사항

- 설정 파일에 빈 인스턴스를 생성하는 메소드를 구현해 놓고 어노테이션으로 설정
- DI 프레임워크는 이 설정 파일을 읽어 **BeanFactory**에 빈으로 저장하여 **ClasspathBeanDefinitionScanner**를 통해 등록한 빈과 같은 저장소에서 관리
- 설정 파일은 자바 클래스로 하고 모든 설정은 어노테이션을 통해 관리
- 별도의 설정 파일을 만들면서 **ClasspathBeanDefinitionScanner**에서 사용할 기본 패키지 설정도 설정 파일에서 할 수 있도록 관리

```java
@Configuration
@ComponentScan({ "next", "core" })
public class MyConfiguration {
    @Bean
    public DataSource dataSource() {
        BasicDataSource ds = new BasicDataSource();
        ds.setDriverClassName("org.h2.Driver");
        ds.setUrl("jdbc:h2:~/jwp-basic;AUTO_SERVER=TRUE");
        ds.setUsername("sa");
        ds.setPassword("");
        return ds;
    }

    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

- `@Configuration`으로 자바 클래스가 설정 파일이라고 표시
- `@Bean` 어노테이션으로 각 메소드에서 생성하는 인스턴스가 **BeanFactory**에 빈으로 등록하라고 설정
- **ClasspathBeanDefinitionScanner**에서 사용할 기본 패키지에 대한 설정을 하드 코딩했는데 설정 파일에서 `@ComponentScan`으로 설정할 수 있도록 하는 것이 목표
- `@Configuration` 설정 파일을 통해 등록한 빈과 **ClasspathBeanDefinitionScanner**를 통해 등록한 빈 간에도 DI가 가능하도록 하는 것도 목표

## 5-2. 1단계 힌트

위 요구사항을 테스트할 수 있는 단위 테스트를 먼저 구현

**AnnotatedBeanDefinitionReaderTest**라는 테스트 클래스를 만들고 앞의 설정 파일을 참고해 테스트용 설정 파일을 만든다.

```java
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface ComponentScan {
    String[] value() default {};
    String[] basePackages() default {};
}
```

- `@ComponentScan`에 배열을 값으로 전달할 수 있도록 코드 추가

- 다음으로 `@Configuration` 설정 파일을 읽어 **BeanFactory**에 **BeanDefinition**을 등록하는 역할을 하는 새로운 **AnnotatedBeanDefinitionReader** 클래스를 추가해야 함.
    - 메소드 반환 클래스에 해당하는 **BeanDefinition**을 생성한 후, **BeanFactory**에 등록
    - **BeanDefinition**을 상속하는 **AnnotatedBeanDefinition**을 추가한 후 메소드 정보까지 같이 전달
    - **BeanFactory**의 `getBean()` 메소드는 인자로 전달되는 빈 클래스에 해당하는 **BeanDefinition**이 **AnnotatedBeanDefinition** 인스턴스일 경우 기존과 다른 방식으로 빈 인스턴스를 생성

**ApplicationContext**가 **ClasspathBeanDefinitionScanner**에서 사용할 기본 패키지 정보를 받는 것이 아니라 `@Configuration`에 설정되어 있는 설정 파일을 인자로 받도록 수정한 후 `@ComponentScan` 값을 구해 **ClasspathBeanDefinitionScanner**로 전달하고 **AnnotatedBeanDefinitionReader**를 통해 설정 파일에 등록된 `@Bean`을 **BeanFactory**에 등록하도록 구현해야 함

## 5-3. 2단계 힌트

### 5-3-1. 단위 테스트 코드 만들기

```java
@Configuration
public class ExampleConfig {
    @Bean
    public DataSource dataSource() {
        BasicDataSource ds = new BasicDataSource();
        ds.setDriverClassName("org.h2.Driver");
        ds.setUrl("jdbc:h2:~/jwp-basic;AUTO_SERVER=TRUE");
        ds.setUsername("sa");
        ds.setPassword("");
        return ds;
    }
}
```

- src/test/java에 테스트용 설정 파일 추가

```java
public class AnnotatedBeanDefinitionReaderTest {
    @Test
    public void register_simple() {
        BeanFactory beanFactory = new BeanFactory();
        AnnotatedBeanDefinitionReader abdr = new AnnotatedBeanDefinitionReader(beanFactory);
        abdr.register(ExampleConfig.class);
        beanFactory.initialize();

        assertNotNull(beanFactory.getBean(DataSource.class));
    }
}
```

- 위 설정 파일을 읽어 **BeanFactory**에 등록한 후 `getBean()`을 통해 빈을 찾았을 때 **null**이 아닌지 여부를 판단하는 테스트 추가

### 5-3-2. BeanFactory에 BeanDefinition 추가

```java
public static Set<Method> getBeanMethods(Class<?> clazz, Class<? extends Annotation> annotation) {
        return getAllMethods(clazz, withAnnotation(annotation));
}
```

- `@Configuration` 설정 파일에서 `@Bean`으로 설정되어 있는 메소드를 먼저 찾음

```java
public class AnnotatedBeanDefinition extends BeanDefinition {
    private Method method;

    public AnnotatedBeanDefinition(Class<?> clazz, Method method) {
        super(clazz);
        this.method = method;
    }

    public Method getMethod() {
        return method;
    }
}
```

- 빈을 생성하려면 `@Bean`이 설정되어 있는 메소드 정보가 필요
- 기존의 **BeanDefinition**을 상속하는 **AnnotatedBeanDefinition**을 추가

### 5-3-3. 빈 인스턴스 생성 및 의존관계 주입

**BeanFactory**에 **BeanDefinition**을 추가했으니 빈 인스턴스 생성과 의존관계를 주입해야 함.

**BeanFactory**의 `getBean()` 메소드에서 **AnnotatedBeanDefinition**을 고려해 개발

```java
public <T> T getBean(Class<T> clazz) {
        Object bean = beans.get(clazz);
        if (bean != null) {
            return (T) bean;
        }

        BeanDefinition beanDefinition = beanDefinitions.get(clazz);
        if (beanDefinition != null && beanDefinition instanceof AnnotatedBeanDefinition) {
            bean = createAnnotatedBean(beanDefinition);
						beans.put(clazz, bean);
		        return (T)bean;
        }
[...]
```

- 위와 같이 구현한 후 앞에서 추가한 테스트가 성공하는지 확인
- 설정 파일에 대한 테스트가 모두 성공하면 **ClasspathBeanDefinitionScanner**와의 통합 테스트를 진행 → **ClasspathBeanDefinitionScanner**에서 추가한 빈과 **AnnotatedBeanDefinitionReader**를 통해 추가한 빈 사이에 DI가 가능한지 테스트할 수 있는 빈을 추가한 후 테스트를 진행

### 5-3-4. ApplicationContext가 설정 파일을 사용하도록 리팩토링

```java
private Object[] findBasePackages(Class<?>[] annotatedClasses) {
        List<Object> basePackages = Lists.newArrayList();
        for (Class<?> annotatedClass : annotatedClasses) {
            ComponentScan componentScan = annotatedClass.getAnnotation(ComponentScan.class);
            if (componentScan == null) {
                continue;
            }
            for (String basePackage : componentScan.value()) {
                log.info("Component Scan basePackage : {}", basePackage);
            }
            basePackages.addAll(Arrays.asList(componentScan.value()));
        }
        return basePackages.toArray();
    }
```

- **ApplicationContext**에서 `@Configuration` 설정 파일을 생성자 인자로 전달받을 수 있도록 수정하고 **ClasspathBeanDefinitionScanner**에서 사용할 기본 패키지 정보를 `@ComponentScan`에서 찾음
- **ClasspathBeanDefinitionScanner**와 **AnnotatedBeanDefinitionReader**를 같이 사용하도록 통합

### 5-3-5. AnnotationHandlerMapping에 ApplicationContext를 DI

**AnnotationHandlerMapping**에서 **ApplicationContext**를 직접 생성하고 있는데 **ApplicationContext**도 **DI**하도록 수정

```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {
    private static final Logger log = LoggerFactory.getLogger(MyWebApplicationInitializer.class);

    @Override
    public void onStartup(ServletContext servletContext) throws ServletException {
        ApplicationContext ac = new AnnotationConfigApplicationContext(MyConfiguration.class);
        AnnotationHandlerMapping ahm = new AnnotationHandlerMapping(ac);
        ahm.initialize();
        [...]
    }
}
```

- **ApplicationContext**라는 인터페이스를 추출하고
- 기존 **ApplicationContext** 클래스를 **AnnotationConfigApplicationContext**로 이름 변경

---

# 6. 초기화 기능 추가

필요없는 코드를 정리하는 작업을 하기위해 **ConnectionManager** 클래스를 제거하면 컴파일 에러가 발생한다.

**BeanFactory**에서 관리하는 **DataSource**를 사용하도록 변경하려면 **ContextLoaderListener**도 빈으로 등록해야 하는데 빈 등록을 하면서 초기화 작업이 필요하다.

→ **BeanFactory**에서 빈 인스턴스를 생성하고 **DI**가 끝난 후 초기화 작업을 진행하도록 기능 추가

```java
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
public @interface PostConstruct {
}
```

- 초기화가 필요한 메소드에 설정할 수 있도록 `@PostConstruct` 어노테이션 추가

```java
@Component
public class DBInitializer {
    private static final Logger log = LoggerFactory.getLogger(DBInitializer.class);

    @Inject
    private DataSource dataSource;

    @PostConstruct
    public void initialize() {
        ResourceDatabasePopulator populator = new ResourceDatabasePopulator();
        populator.addScript(new ClassPathResource("jwp.sql"));
        DatabasePopulatorUtils.execute(populator, dataSource);

        log.info("Completed Load ServletContext!");
    }
}
```

- 데이터베이스 초기화를 담당하는 **ContextLoaderListener**를 **DBInitializer**로 이름 변경
- `@PostConstruct`를 설정한 메소드가 존재하면 메소드를 실행해 초기화할 수 있도록 지원

```java
public class BeanFactory implements BeanDefinitionRegistry {
    public <T> T getBean(Class<T> clazz) {
				[...]
        beanDefinition = beanDefinitions.get(concreteClazz.get());
        bean = inject(beanDefinition);
        beans.put(concreteClazz.get(), bean);
        initialize(bean, concreteClazz.get());
        return (T) bean;
    }

    private void initialize(Object bean, Class<?> beanClass) {
        Set<Method> initializeMethods = BeanFactoryUtils.getBeanMethods(beanClass, PostConstruct.class);
        if (initializeMethods.isEmpty()) {
            return;
        }
        for (Method initializeMethod : initializeMethods) {
            log.debug("@PostConstruct Initialize Method : {}", initializeMethod);
            BeanFactoryUtils.invokeMethod(initializeMethod, bean,
                    populateArguments(initializeMethod.getParameterTypes()));
        }
    }
}
```

- **BeanFactory**의 `getBean()`에서 `@PostConstruct` 설정 메소드를 찾아 실행

---

# 7. 인터페이스, DI, DI 컨테이너

지금까지 MVC 프레임워크와 DI 프레임워크를 구현하는 과정은 유연성과 확장성이 필요한 경우 인터페이스를 추가하고 이 인터페이스를 구현하는 구현체를 추가하는 방식으로 확장했다.

스프링 프레임워크가 버전을 업그레이드하면서 하위 버전을 지원할 수 있는 이유 또한 잘 설계한 인터페이스에 있다.

```java
public interface BeanDefinitionReader {
    void loadBeanDefinitions(Class<?>... annotatedClasses);
}
```

- BeanDefinition을 생성하기 위해 어노테이션 설정만을 지원하는 것이 아니라 XML, Property 설정을 지원하도록 확장할 수 있다.

```java
public interface BeanDefinitionRegistry {
    void registerBeanDefinition(Class<?> clazz, BeanDefinition beanDefinition);
}
```

- BeanDefinitionReader의 변화에 영향을 받지 않으면서 독립적으로 확장하는 것이 가능

```java
public interface BeanFactory {
    Set<Class<?>> getBeanClasses();
    <T> T getBean(Class<T> clazz);
    void clear();
}
```

```java
public interface BeanDefinition {
    Constructor<?> getInjectConstructor();
    Set<Field> getInjectFields();
    Class<?> getBeanClass();
    InjectType getResolvedInjectMode();
}
```

- BeanDefinition, BeanFactory 또한 인터페이스 기반으로 확장 가능

확장 필요한 부분을 인터페이스로 구현하고 각 인터페이스 사이의 연결은 구현 클래스가 아닌 인터페이스를 통해서만 의존관계를 가지도록 연결함으로써 유연한 구조를 가질 수 있음.

서로 다르게 변화하는 구현 클래스 사이의 연결은 이 클래스의 연결을 담당하는 factory 클래스를 통해 담당하도록 구현함으로써 개발자의 편의성 확보 가능

```java
public interface ApplicationContext {
    <T> T getBean(Class<T> clazz);
    Set<Class<?>> getBeanClasses();
}
```

- 대표적인 팩토리 클래스인 ApplicationContext 또한 구현 클래스의 연결 조합에 따라 다양한 조합이 가능하기 때문에 인터페이스를 추가한 후 확장 가능하도록 구현 가능

```java
public class AnnotationConfigApplicationContext implements ApplicationContext {
    private static final Logger log = LoggerFactory.getLogger(AnnotationConfigApplicationContext.class);

    private DefaultBeanFactory beanFactory;

    public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
        Object[] basePackages = findBasePackages(annotatedClasses);
        beanFactory = new DefaultBeanFactory();
        BeanDefinitionReader abdr = new AnnotatedBeanDefinitionReader(beanFactory);
        abdr.loadBeanDefinitions(annotatedClasses);

        if (basePackages.length > 0) {
            ClasspathBeanDefinitionScanner scanner = new ClasspathBeanDefinitionScanner(beanFactory);
            scanner.doScan(basePackages);
        }
        beanFactory.preInstantiateSinglonetons();
    }

    private Object[] findBasePackages(Class<?>[] annotatedClasses) {
        [...]
    }

    @Override
    public <T> T getBean(Class<T> clazz) {
        return beanFactory.getBean(clazz);
    }

    @Override
    public Set<Class<?>> getBeanClasses() {
        return beanFactory.getBeanClasses();
    }
}
```

- ApplicationContext 인터페이스에 대한 구현 클래스는 어노테이션 기반으로 DI 컨테이너를 생성할 경우, AnnotationConfigApplicationContext라는 이름으로 확장 가능
- AnnotationConfigApplicationContext는 각 인터페이스에 대한 구현체를 조합하는 역할만 담당하고, 어노테이션 설정 기반의 DI 컨테이너에 대한 실질적인 작업은 BeanFactory에 모두 위임한다.

---

# 8. 웹 서버 도입을 통한 서비스 운영

서비스에 접속하는 사용자가 없는 경우에는 톰캣 서버를 종료하고 소스코드를 배포하는 방식으로 운영해도 되지만 사용자가 많아질 경우 아파치, nignx와 같은 웹서버를 도입해 톰캣 서버와 통합 운영해야 한다.

## 8-1. 요구사항

웹 서버를 추가해 톰캣 서버와 연결한 후 소스코드 배포 중 점검 페이지를 보여주도록 개선

## 8-2. 1단계 힌트

- nginx를 설치하고 톰캣 서버와 연결
- nginx 설정 파일로 점검용 페이지를 위한 설정과 톰캣 서버와의 연결을 담당하는 설정 2개 구현
- 위 과정을 수동으로 진행해 성공하면 다음 단계는 쉘 스크립트를 통해 자동화

## 8-3. 2단계 힌트

### 8-3-1. nginx 설치

- nginx 설치
- ps -ef | grep nginx 명령을 통해 nginx가 실행 상태인지 확인
- curl http://localhost 명령을 실행해 nginx 환영 메세지를 포함하는 HTML 응답하는지 확인
- 웹 브라우저에서 http://{서버 IP 주소} 로 접근 (접근되지 않으면 80 포트에 대한 방화벽 설정을 해제)

### 8-3-2. nginx와 톰캣 서버 연결

- /etc/nginx/sites-available 디렉토리에 jwp-basic.conf와 같은 이름으로 파일 생성 후, 내용 설정
- /etc/nginx/sites-enabled 디렉토리의 default 심볼릭 링크 삭제
- /etc/nginx/sites-enabled 디렉토리에 앞에서 생성한 설정 파일을 심볼릭 링크로 설정
- sudo nginx -s reload 명령을 실행해 nginx 재시작
- 웹 브라우저에서 http://{서버 IP 주소} 로 접근

### 8-3-3. 점검 페이지 준비 및 수동으로 점검 페이지 변경

- 점검 페이지로 사용할 HTML, CSS, 이미지 준비
- /etc/nginx/sites-available 디렉토리의 default 설정 파일을 jwp-basic-pm.conf와 같은 이름으로 복사
- 복사한 설정 파일의 root 설정을 점검 페이지를 위한 디렉토리로 변경
- /etc/nginx/sites-enabled 디렉토리의 심볼릭 링크 설정을 삭제하고 앞에서 추가한 설정 파일에 심볼릭 링크 설정
- sudo nginx -s reload 명령을 실행해 nginx 재시작
- 웹 브라우저에서 http://{서버 IP 주소} 로 접근

### 8-3-4. 쉘 스크립트를 통한 배포 자동화

- deploy.sh 설정 파일에 다음 내용 추가
- /etc/nginx/sites-enabled 디렉토리의 심볼릭 링크 설정을 삭제하고 /etc/nginx/sites-available 디렉토리의 점검용 설정 파일로 심볼릭 링크 다시 설정
- sudo nginx -s reload 명령을 실행해 nginx 재시작
- 배포 완료하고 톰캣 서버가 정상인지 확인한 후, nignx 설정을 점검 페이지에서 서비스 연결로 변경하기 위해 쉘 스크립트 작성
- deploy-finish.sh라는 이름의 새로운 쉘 스크립트 추가
- /etc/nginx/sites-enabled 디렉토리의 심볼릭 링크 설정을 삭제하고 /etc/nginx/sites-available 디렉토리의 서비스용 설정 파일로 심볼릭 링크 다시 설정
- sudo nginx -s reload 명령을 실행해 nginx 재시작
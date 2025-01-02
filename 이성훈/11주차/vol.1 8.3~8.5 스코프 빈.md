# 1.3~ 끝까지

## 1.3 프로로타입과 스코프

기본적으로 스프링 빈은 싱글톤

= 애플리케이션 컨텍스트마다 한 개만 만들어진다.

= 하나의 빈을 여러개의 빈에서 DI 하더라도 매번 동일한 오브젝트가 주입된다.

= 따라서 변하는 상태를 저장하는 값은 저장하지 않아야 한다.

만약, 싱글톤이 아닌 다른 방법으로 만들어 사용해야 한다면?

→ 프로토타입 빈 or 스코프 빈을 사용할 수 있다.

## 프로토타입 스코프

컨테이너에게 **빈을 요청할 때마다**(getBean() 메소드를 통한 요청뿐만 아니라 @Autowired 같은 DI 선언도 해당) 매번 새로운 오브젝트를 생성해준다.

<일반적인 빈의 주기>

- 코드가 아닌 컨테이너가 빈의 생명주기를 관리함(IoC)
- 즉, 빈의 생성과 의존관계 주입, 초기화(메소드 호출), DI와 DL을 통한 사용, 제거 등 모든 생명주기를 **컨테이너가 관리**한다.

<프로토타입 빈>

- 빈을 제공하고 나면 컨테이너는 더 이상 빈 오브젝트를 관리하지 않는다.
- 즉, 컨테이너가 초기 생성 시에만 관여하고 DI 한 후에는 더 이상 신경을 쓰지 않는다.

### **1. 언제 사용할까?**

**1번 상황**

예를 들어, 다음과 같이 ServiceRequest 라는 빈이 있는데, Customer 라는 ‘값이 매번 달라지는’ 변수가 있다고 하자.

```java
public class ServiceRequest {
	**Customer customer;**
	String productNo;
}
```

위와 같은 경우 ServiceRequest라는 빈은 호출될 때마다 새로 만들어져야 한다.

**2번 상황**

또 여기서, 만약 CustomerDao를 통해 DB와 직접 연결을 하고 싶다면 어떻게 해야할까?
(CustomerDao는 이미 스프링 컨테이너에 의해 관리되는 싱글톤 빈이다.)

```java
public class ServiceRequest {
	**Customer customer;**
	String productNo;
	@Autowired 
	CustomerDao customerDao;
}
```

위에서 봤던 1번 상황과 2번 상황을 고려했을 때,

**매번 새로운 오브젝트가 필요**하면서 DI를 통해 다른 빈을 사용할 수 있어야 할 때

프로토타입 빈으로 해결할 수 있다.

### 2. 어떻게 사용할까?

```java
@Component
**@Scope("prototype")**
public class ServiceRequest {
	**Customer customer;**
	String productNo;
}
```

위와 같이, @Scope 어노테이션을 통해 프로토타입 빈으로 지정할 수 있다.

```java
<bean id="serviceRequest" class="...ServiceRequest" scope="prototype">
```

또는, xml을 이용한다면 위와 같이 작성하여 프로토타입 빈으로 지정할 수 있다.

**주의:::: 프로토타입으로 지정된 빈을 다음과 같이 DI를 통해 사용해서는 안된다.**

```java
@Autowired ServiceRequest serviceRequest;

public void serviceRequestFormSubmit(HttpServletRequest request) {
	this.serviceRequest.setCustomerNo(request.getParameter("custno"));
	...
}
```

**DI 작업은 빈 오브젝트가 처음 만들어질 때 단 한번만 진행되기 때문에, 프로토타입 빈으로 지정하였더라도 DI를 한다면 오직 단 한번만 생성된다.**

따라서 프로토타입 빈은 DL 전략을 사용해야 한다.

DL 방법은 다음과 같이 여러 방식이 존재한다.

- ApplicationContext 혹은 BeanFactory를 DI 받은 후에, getBean() 메소드를 호출하는 방법
    
    → 스프링의 API가 일반 애플리케이션 코드에서 사용된다는 사실이 불편할 수 있다.
    
    → 너무 무겁다.
    
- ObjectFactory 또는 ObjectFactoryCreatingFactoryBean을 사용
    
    → 팩토리 메서드 패턴을 사용함으로써 getBean() 과 같은 로우 레벨의 메소드를 사용하지 않을 수 있다.
    
    → 빈을 새로 추가해야함
    
    → 위 방법보다 테스트에서 사용하기에 용이하다.
    
- ServiceLocatorFactoryBean 팩토리 인터페이스를 만들어 사용
    
    → 스프링 프레임워크의 인터페이스를 애플리케이션에서 사용하는 것이 마음에 들지 않을 때 적용해볼 수 있다.
    
    → 빈을 새로 추가해야함
    
- 메소드 주입 (메소드 코드 자체를 주입하는 것을 말함)
    
    → 스프링의 <lookup-method>라는 태그와 더불어 추상 클래스 및 추상 메소드를 이용함
    
    → 위 두 가지 단점을 보완함
    
    → 하지만 단위 테스트를 많이 작성할 것이라면 더 불편할 수 있음
    
- Provider<T> + @Inject  함께 사용
    
    → 스프링이 자동으로 Provider를 구현한 오브젝트를 생성하여 주입함
    

## 스코프

### 스코프의 종류

스프링이 기본적으로 제공하는 스코프는 다음과 같다.

- 요청 스코프
    
    하나의 웹 요청 안에서 만들어지고 해당 요청이 끝날 때 제거된다.
    
    각 요청별로 독립적인 빈이 만들어지기 때문에 빈 오브젝트 내에 상태 값을 저장해둬도 안전하다.
    
    주요 용도는 애플리케이션 코드에서 생성한 정보를 프레임워크 레벨의 서비스나 인터셉터에 전달하는 것, 혹은 그 반대로 전달하여 이용하는 것
    
- 세션 스코프/글로벌세션 스코프
    
    HTTP 세션과 같은 존재 범위를 갖는 빈으로 만들어주는 스코프
    
    브라우저를 닫거나 세션 타임이 종료될 때까지 유지
    
- 애플리케이션 스코프
    
    서블릿 컨텍스트에 저장되는 빈 오브젝트
    
    싱글톤 스코프와 비슷한 존재 범위를 갖는다.
    
    웹 애플리케이션 밖에서 더 오랫동안 존재하는 컨텍스트도 있고, 더 짧게 존재하는 서블릿 레벨의 컨텍스트도 있기 때문에 존재
    
    상태를 갖지 않거나, 상태가 있다고 하더라도 읽기전용으로 만들거나, 멀티스레드 환경에서 안전하도록 만들어야 함
    

위 세 가지 스코프는 프로토타입 빈과 마찬가지로 한 개 이상의 빈 오브젝트가 생성된다는 점은 같지만, 컨테이너가 정확하게 언제 새로운 빈이 만들어지고 사용될지 파악한다는 점이 다르다.

스코프 빈을 DI 방식으로 사용하려면 별도의 스코프 프록시를 통해 적용 가능하다.

(하지만, 주입되는 빈의 스코프를 모르면 코드를 이해하기 어려울 수 있다는 점 유의해야함)

또한, 스프링이 기본적으로 제공하는 스코프 외에도 임의의 스코프를 만들 수도 있다.

## 빈 설정 메타정보

### 빈 이름

XML에서의 빈 태그

- id
    
    빈의 식별자로서 사용, id는 생략도 가능하긴 함
    
    단 하나만 부여할 수 있음
    
- name
    
    id와 달리 한 번에 여러 개의 이름을 지정할 수 있음
    

애노테이션에서의 빈 이름

- @Component(”myuserService”)
- @Named(”myUserService”)
- @Bean(name=”myUserService”, “userService”)

### 초기화 메서드

빈 오브젝트가 생성되고 DI 까지 마친 다음에 실행되는 메서드

- xml에서 init-method 태그를 사용
- @PostConstruct
- @Bean(init-method=”~~~”)

### 제거 메소드

- DisposableBean 인터페이스를 구현하여 destory() 함수를 구현
- xml에서 destroy-method 태그를 사용
- @PreDestory
- @Bean(destroyMethod)

### 팩토리 빈과 팩토리 메소드

**팩토리 빈** 생성자 대신 오브젝트를 생성해주는 코드의 도움을 받아서 빈 오브젝트를 생성하는 것

- FactoryBean 인터페이스 사용
- 스태틱 팩토리 메소드
- 인스턴스 팩토리 메소드
- @Bean

## 스프링 3.1의 IoC 컨테이너와 DI

### 빈의 역할과 구분

- 애플리케이션 로직 빈
    
    애플리케이션 로직을 담고 있는 주요 클래스의 오브젝트가 빈으로 등록된 것
    
- 애플리케이션 인프라 빈
    
    DAO와 같이, 애플리케이션 로직이 동작하기 위해 참여하는 빈
    
- 컨테이너 인프라 빈
    
    DefaultAdvisorAutoProxyCreator 등과 같이 스프링 컨테이너의 기능을 확장해서 빈의 등록과 생성, 관계설정, 초기화 등의 작업에 참여하는 빈
    
    @Configuration
    
    → @Configuration/@Bean은 스프링 컨테이너가 기본적으로 제공하는 기능이 아니므로, XML을 사용한다면 <context:annotation-config />를 사용해야함
    
    → @ComponentScan을 통해 컨테이너 인프라 빈 등록 가능
    
    | 버전 | 애플리케이션 로직 빈 | 애플리케이션 인프라 빈 | 컨테이너 인프라 빈 |
    | --- | --- | --- | --- |
    | 스프링 1.x | <bean> | <bean> | <bean> |
    | 스프링 2.0 | <bean> | <bean> | 전용 태그 |
    | 스프링 2.5 | <bean>, 빈 스캔 | <bean> | 전용 태그 |
    | 스프링 3.0 | <bean>, 빈 스캔, 자바 코드 | <bean>,  자바 코드 | 전용 태그 |
    | 스프링 3.1 | <bean>, 빈 스캔, 자바 코드 | <bean>.  자바 코드 | 전용 태그, 자바 코드 |

- @ComponentScan
    
    패키지 이름 혹은 마커 인터페이스(basePackageClasses=” “)를 사용하여 기준 설정
    
- @Import
    
    다른 @Configuration 클래스를 빈 메타정보에 추가할 때 사용
    
- @ImportResource
    
    xml 파일을 지정하여 빈 설정을 가져올 수 있음
    
- @EnableTransactionManagement
    
    AOP 관련 빈을 등록할 수 있음
    

### 런타임 환경 추상화와 프로파일

개발 환경, 테스트 환경, 배포 환경마다 빈 설정정보가 다를 경우, 환경에 맞게 적용하기 위함

```java
spring:
  profiles:
    include:
      - db
      - security
    group:
      test:
        - db-local
        - security-test
      dev:
        - db-dev
        - security-dev
      prod:
        - db-prod
        - security-prod
    active: dev
```
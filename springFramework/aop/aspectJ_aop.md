# AspectJ - aop

## Aspect-Oriented Programming in Java

본 글은 Markus Voelter에 의해 작성된 글 중 일부이다. 원문은 AOP 기본 개념, Xerox  

PARC에 의해 구현된 Java의 AOP 확장 버전인 AspectJ 소개, Metaclass 프로그래밍과의 비교 등 총 3 파트로 구성되어 있으며, 번역문은 이 중 첫 번째 파트만 커버한다. 참고로 원문의 AspectJ 관련 코드는 상당히 오래된 문법에 기반하여 현재의 그것과 많은 차이를 보인다 

## Introduction 

최근 몇 년에 걸쳐 객체지향 프로그래밍(Object-Oriented Programming, OOP)은 절차적 방법론을 거의 완벽히 대체하며 프로그래밍 방법론의 새 주류로 떠오르게 되었다. 객체지향적 방식의 가장 큰 이점 중 하나는 소프트웨어 시스템이 여러 개의 독립된 클래스들의 집합으로 구성된다는 것이다. 이들 각각의 클래스들은 잘 정의된 고유 작업을 수행하게 되고, 그 역할 또한 명백히 정의되어 있다. 객체지향 어플리케이션에서는 어플리케이션이 목표한 동작을 수행하기 위해 이런 클래스들이 서로 유기적으로 협력하게 된다. 하지만 시스템의 어떤 기능들은 특정 한 클래스가 도맡아 처리할 수 없다. 이들은 시스템 전체를 들쑤시며 해당 코드들을 여러 클래스들에 흩뿌려 놓는다. 이런 현상을 횡단적(cross-cutting)이라 표현한다. 분산 어플리케이션에서의 락킹(locking, 동기화) 문제, 예외 처리, 로깅 등이 그 예이다. 물론 필요한 모든 클래스들에 관련 코드를 집어 넣으면 해결될 문제이다. 하지만 이런 행위는 각각의 클래스는 잘 정의된(well-defined) 역할만을 수행한다 

는 기본 원칙에 위배된다. 이런 상황이 바로 Aspect-Oriented Programming (AOP)이 잉태된 원인이 되었다.  

AOP에서는 aspect라는 새로운 프로그램 구조를 정의해 사용한다. 이는 쉽게 struct, class, interface 등과 같이 특정한 용도의 구조라 생각하면 된다. Aspect 내에는 프로그램의 여러 모듈들에 흩어져 있는 기능(하나의 기능이 여러 모듈에 흩어져 있음을 뜻한다)을 모아 정의하게 된다. 전체적으로, 어플리케이션의 각각의 클래스는 자신에게 주어진 기능만을 수행하고, 추가된 각 aspect들이 횡단적인 행위(기능)들을 모아 처리하며 전체 프로그램을 이루는 형태가 만들어진다. 

 ### AOP Basics  

 이해를 돕기 위해, 그리고 설명을 쉽게 하기 위해 예를 들어가며 AOP 개념을 설명토록 하겠다. 어플리케이션의 여러 스레드들이 하나의 데이터를 공유하는 상황을 가정해보자. 공유 데이터는 Data라는 객체(Data 클래스의 인스턴스)로 캡슐화되어 있다. 서로 다른 여러 클래스의 인스턴스들이 하나의 Data 객체를 사용하고 있으나 이 공유 데이터에 접근할 수 있는 객체는 한 번에 하나씩이어야만 한다. 그렇다면 어떤 형태이건 동기화 메커니즘이 도입되어야 할 것이다. 즉, 어떤 한 객체가 데이터를 사용중이라면 Data 객체는 잠겨(lock)져야 하며, 사용이 끝났을 때 해제(unlock)되어야 한다. 전통적인 해결책은 공유 데이터를 사용하는 모든 클래스들이 하나의 공통 부모 클래스("worker" 라 부르도록 하자)로부터 파생되는 형태이다. worker 클래스에는 lock()과 unlock() 메소드를 정의하여 작업의 시작과 끝에 이 메소드를 호출토록 하면 된다. 하지만 이런 형태는 다음과 문제들을 파생시킨다. 

 - 공유 데이터를 사용하는 메소드는 상당히 주의해서 작성되어야 한다. 동기화 코드를 잘못 삽입하면 데드락(dead-lock)이 발생하거나 데이터 영속성이 깨질 수 있다. 또한 메소드 내부는 본래의 기능과 관련 없는 동기화 관련 코드들로 더럽혀질 것이다. 

 - Java와 같은 단일 상속 모델에서는 worker를 만든다는 것이 불가능할 수 있다. 어떤 클래스들은 이미 다른 클래스들로부터 확장되었을 수도 있기 때문이다. 이는 특히 클래스 계층 구조 설계가 마무리된 후, 뒤늦게 동기화의 필요성을 깨달았을 때 흔히 발생한다. 동기화를 신경 쓰지 않은 범용 클래스 라이브러리를 통해 공유 데이터에 접근하려 하는 경우가 한 예가 될 수 있다. 

 - 재활용성(reusability)이 감소된다. worker 클래스는 동기화가 필요치 않는 클래스나 심지어 다른 동기화 메커니즘이 필요한 상황에서도 사용하길 원할 수 있다. 동기화 관련 코드를 삽입함으로써 worker 클래스는 특정 어플리케이션에 종속적인 클래스로 전락하게 된다. 

우리가 앞서 가정한 어플리케이션에서 동기화 개념은 다음과 같은 속성들을 갖는다. 

- 동기화는 worker 클래스에 할당된 최우선 작업이 아니다. 

- 동기화 메커니즘은 worker 클래스의 최우선 작업과 독립적이다. 

- 한 객체에 대한 동기화 관련 코드가 시스템 전체에 횡단적으로 존재한다. 다수의 클래스와 더 많은 수의 메소드들이 이 동기화 메커니즘에 영향 받는다. 

AOP에서는 이런 형태의 문제를 해결하기 위해 새로운 형태의 접근 방법을 제기하고 있다. AOP는 새로 도입된 프로그램 구조를 통해 시스템에 횡단되어 있는 기능들을 정의해 처리하도록 했다. 이 새로운 구조를 우리는 aspect라 부른다. 

우리의 예에 Lock이라는 aspect를 도입해보자. Lock aspect는 다음과 같은 역할이 할당될 것이다. 

- Data 객체를 사용하는 클래스들을 위해 lock 및 unlock 메커니즘을 제공한다(lock(), unlock()). 

- Data 객체를 수정하는 모든 메소드들이 수행 전에 lock()을 호출하고, 수행 후에는 unlock()을 호출함을 보장한다. 

- 이상의 기능을 Data 객체를 사용하는 클래스의 자바 소스를 변경하지 않고 투명하게 수행한다. 


## Aspect는 또 어떤 일들을 수행할 수 있을까? 

특정 메소드(ex. 객체 생성 과정 추적) 호출을 로깅할 경우 aspect가 도움이 될 수 있다. 기존 방법대로라면 log() 메소드를 만들어 놓은 후, 자바 소스에서 로깅을 원하는 메소드를 찾아 log()를 호출하는 형태를 취해야할 것이다. 여기서 AOP를 사용하면 원본 자바 코드를 수정할 필요 없이 원하는 위치에서 원하는 로깅을 수행할 수 있다. 이런 작업 모두는 aspect라는 외부 모듈에 의해 수행된다. 

또 다른 예로 예외 처리가 있다. Aspect를 이용해 여러 클래스들의 산재된 메소드들에 영향을 주는 catch() 조항(clause)을 정의해 어플리케이션 전체에 걸친 지속적이고 일관적으로 예외를 처리할 수 있다. 

### 6.1.1. AOP 개념  

몇몇 중심적인 AOP개념을 명시함으로써 시작해보자. 이 개념들은 Spring에 종속적인 개념이 아니다. 운 나쁘게도 AOP전문용어는 특히 직관적이지 않다. 어쨌든 Spring이 그 자신의 전문용어를 사용했다면 좀더 혼란스러울것이다. 

- Aspect: 다중 객체에 영향을 주는 concern의 모듈화. 트랜잭션 관리는 J2EE애플리케이션의 crosscutting concern의 좋은 예제이다. Spring AOP에서, aspect는 정규 클래스나 @Aspect 어노테이션(@Aspect 스타일)으로 주석처리된 정규 클래스를 사용하여 구현된다. 

- Joinpoint: 메소드 수행이나 예외를 다루는 것과 같은 프로그램의 수행기간 동안의 point. Spring AOP에서, joinpoint는 언제나 메소드 호출을 나타낸다. Join point정보는 org.aspectj.lang.JoinPoint 타입의 파라미터를 선언하여 advice에서 사용가능하다. 

- Advice: 특정 joinpoint에서 aspect에 의해 획득되는 액션. advice의 다른 타입은 "around," "before" 과 "after" advice를 포함한다. advice 타입은 밑에서 언급된다. Spring을 포함한 많은 AOP프레임워크는 인터셉터로 advice를 모델화하고 joinpoint "주위(around)"로 인터셉터의 묶음(chain)을 유지한다. 

- Pointcut: join point에 일치하는 속성. advice는 pointcut표현과 관련있고 pointcut에 일치하는 join point에서 수행된다(예를 들어, 어떤 이름을 가진 메소드의 수행). pointcut표현에 의해 일치하는 join point의 개념은 AOP에 집중된다. Spring은 디폴트로 AspectJ pointcut언어를 사용한다. 

- Introduction: (중간(inter)-타입 선언으로 알려진) 타입에서 추가적인 메소드나 필드의 선언. Spring은 프록시화된 객체를 위해 새로운 인터페이스(와 관련 구현물)를 소개한다. 예를 들어, 간단한 캐시를 위해, IsModified 인터페이스를 구현하는 bean을 만들기 위한 소개(introduction)를 사용한다. 

- 대상 객체: 객체는 하나이상의 aspect에 의해 충고된다. 또한 advised 객체를 참조한다. Spring AOP가 런타임 프록시를 사용하여 구현되기 때문에, 이 객체는 언제나 프록시화된 객체가 될것이다. 

- AOP 프록시: aspect 규칙(advise메소드수행과 기타등등)을 구현하기 위하여 AOP프레임워크에 의해 생성되는 객체. Spring에서. AOP프록시는 JDK 동적 프록시나 CGLIB 프록시가 될것이다. 노트 : 프록시 생성은 스키마-기반과 Spring 2.0에서 소개된 aspect선언의 @AspectJ스타일의 사용자에게 명백하다. 

- Weaving: 다른 애플리케이션 타입이나 advised객체를 생성하기 위한 객체를 가지는 aspect 연결. 이것은 컴파일 시점(예를 들어, AspectJ 컴파일러를 사용하여), 로그시점 또는 런타임에 수행될수 있다. 다른 Java AOP프레임워크처럼 Spring은 런타임시 직조(weaving)를 수행한다. 


### Advice 타입

- Before advice: joinpoint전에 수행되는 advice. 하지만 joinpoint를 위한 수행 흐름 처리(execution flow proceeding)를 막기위한 능력(만약 예외를 던지지 않는다면)을 가지지는 않는다. 
- After returning advice: joinpoint이 일반적으로 예를 들어 메소드가 예외를 던지는것없이 반환된다면 완성된 후에 수행되는 advice
- After throwing advice: 메소드가 예외를 던져서 빠져나갈때 수행되는 advice 
- After (finally) advice: join point를 빠져나가는(정상적이거나 예외적인 반환) 방법에 상관없이 수행되는 advice. 
- Around advice: 메소드 호출과 같은 joinpoint주위(surround)의 advice. 이것은 가장 강력한 종류의 advice이다. Around advice는 메소드 호출 전후에 사용자 정의 행위를 수행할수 있다. 그것들은 joinpoint를 처리하거나 자기 자신의 반환값을 반환함으로써 짧게 수행하거나 예외를 던지는 것인지에 대해 책임을 진다. 

Around advice는 가장 일반적인 종류의 advice이다. Nanning Aspects와 같은 대부분의 인터셉션-기반의 AOP프레임워크는 오직 around advice만을 제공한다. 

AspectJ처럼 Spring이 advice타입의 모든 범위를 제공하기 때문에 우리는 요구되는 행위를 구현할수 있는 최소한의 강력한 advice타입을 사용하길 권한다. 예를 들어 당신이 메소드의 값을 반환하는 캐시만을 수정할 필요가 있다면 around advice가 같은것을 수행할수 있다고하더라도 around advice보다 advice를 반환한 후에 구현하는게 더 좋다. 대부분 특정 advice타입을 사용하는것은 잠재적으로 적은 에러를 가지는 간단한 프로그래밍 모델을 제공한다. 예를 들어 당신은 around advice를 위해 사용되는 JoinPoint의 proceed()메소드를 호출할 필요가 없고 나아가 그것을 호출하는것을 실패할수도 있다. 

Spring 2.0에서, 모든 advice파라미터는 정적으로 타입화된다. 그래서 객체 배열보다는 적절한 타입(예를 들면, 메소드 수행의 반환값의 타입)의 advice파라미터를 가지고 작업한다. 

pointcut에 의해 일치하는 join point의 개념은 오직 인터셉션만을 제공하는 예전 기술과 구분되는 AOP의 핵심이다. pointcut은 OO구조의 단독으로 대상화되도록 해준다. 예를 들어, 선언적인 트랜잭션 관리를 제공하는 around advice는 다중 객체에 걸쳐있는 메소드에 적용될수 있다(서비스 레이어내 모든 비니지스 작동과 같은). 


그럼 우선 포인트 컷을 정의하는 방법부터 보자.

### 포인트컷(Pointcut) 정의하기 

포인트컷은 결합점(Join points)을 지정하여 충고(Advice)가 언제 실행될지를 지정하는데 사용된다. Spring AOP는 Spring 빈에 대한 메소드 실행 결합점만을 지원하므로, Spring에서 포인트컷은 빈의 메소드 실행점을 지정하는 것으로 생각할 수 있다. 

다음 예제는 egovframework.rte.fdl.aop.sample 패키지 하위의 Sample 명으로 끝나는 클래스의 모든 메소드 수행과 일치할 'targetMethod' 라는 이름의 pointcut을 정의한다. 

@Aspect 

public class AspectUsingAnnotation { 

   ... 

   @Pointcut("execution(public * egovframework.rte.fdl.aop.sample.*Sample.*(..))") 

   public void targetMethod() { 

       // pointcut annotation 값을 참조하기 위한 dummy method 

   } 

   ... 

} 

### 포인트컷 지정자(Designators) 

Spring에서 포인트컷 표현식에 사용될 수 있는 지정자는 다음과 같다. 포인트컷은 모두 public 메소드를 대상으로 한다. 
- execution: 메소드 실행 결합점(join points)과 일치시키는데 사용된다. 
- within: 특정 타입에 속하는 결합점을 정의한다. 
- this: 빈 참조가 주어진 타입의 인스턴스를 갖는 결합점을 정의한다. 
- target: 대상 객체가 주어진 타입을 갖는 결합점을 정의한다. 
- args: 인자가 주어진 타입의 인스턴스인 결합점을 정의한다. 
- @target: 수행중인 객체의 클래스가 주어진 타입의 어노테이션을 갖는 결합점을 정의한다. 
- @args: 전달된 인자의 런타입 타입이 주어진 타입의 어노테이션을 갖는 결합점을 정의한다. 
- @within: 주어진 어노테이션을 갖는 타입 내 결합점을 정의한다. 
- @annotation: 결합점의 대상 객체가 주어진 어노테이션을 갖는 결합점을 정의한다. 

### 포인트컷 표현식 조합하기 

포인트컷 표현식은 '&&', '||' 그리고 '!' 를 사용하여 조합할 수 있다. 

  @Pointcut("execution(public * *(..))") 

   private void anyPublicOperation() {} 

 

   @Pointcut("within(com.xyz.someapp.trading..*)") 

   private void inTrading() {} 

 

   @Pointcut("anyPublicOperation() && inTrading()") 

   private void tradingOperation() {} 

### 포인트컷 정의 예제

Spring AOP에서 자주 사용되는 포인트컷 표현식의 예를 살펴본다. 

Pointcut | 선택된 Joinpoints 
---------|------------------

execution(public * *(..)) |public 메소드 실행 
execution(* set*(..)) |이름이 set으로 시작하는 모든 메소드명 실행 
execution(* set*(..)) |이름이 set으로 시작하는 모든 메소드명 실행 
execution(* com.xyz.service.AccountService.*(..)) |AccountService 인터페이스의 모든 메소드 실행 
execution(* com.xyz.service.*.*(..)) |service 패키지의 모든 메소드 실행 
execution(* com.xyz.service..*.*(..)) |service 패키지와 하위 패키지의 모든 메소드 실행 
within(com.xyz.service.*) |service 패키지 내의 모든 결합점 (클래스 포함) 
within(com.xyz.service..*) |service 패키지 및 하위 패키지의 모든 결합점 (클래스 포함) 
this(com.xyz.service.AccountService) |AccountService 인터페이스를 구현하는 프록시 개체의 모든 결합점 
target(com.xyz.service.AccountService) |AccountService 인터페이스를 구현하는 대상 객체의 모든 결합점 
args(java.io.Serializable) |하나의 파라미터를 갖고 전달된 인자가 Serializable인 모든 결합점 
@target(org.springframework.transaction.annotation.Transactional) |대상 객체가 @Transactional 어노테이션을 갖는 모든 결합점 
@within(org.springframework.transaction.annotation.Transactional) |대상 객체의 선언 타입이 @Transactional 어노테이션을 갖는 모든 결합점 
@annotation(org.springframework.transaction.annotation.Transactional) |실행 메소드가 @Transactional 어노테이션을 갖는 모든 결합점 
@args(com.xyz.security.Classified) |단일 파라미터를 받고, 전달된 인자 타입이 @Classified 어노테이션을 갖는 모든 결합점 
bean(accountRepository) |“accountRepository” 빈 
!bean(accountRepository) |“accountRepository” 빈을 제외한 모든 빈 
bean(*) |모든 빈 
bean(account*) |이름이 'account'로 시작되는 모든 빈 
bean(*Repository) |이름이 “Repository”로 끝나는 모든 빈 
bean(accounting/*) |이름이 “accounting/“로 시작하는 모든 빈 
bean(*dataSource) || bean(*DataSource) |이름이 “dataSource” 나 “DataSource” 으로 끝나는 모든 빈 콘솔 로그 출력을 보면 충고(Advice)가 적용되는 순서는 다음과 같다. 

- @Before 
- @Around (대상 메소드 수행 전) 
- 대상 메소드 
- @Around (대상 메소드 수행 후) 
- @After(finally) 
- @AfterReturning 

예제는 class에 annotation을 이용한 logging 전략과, 
execution을 이용한 logging 전략을 이용한다.
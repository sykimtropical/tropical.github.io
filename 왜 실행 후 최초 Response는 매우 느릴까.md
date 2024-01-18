
> 💡키워드 <br>
> • Spring boot Warm up<br>
> • cold start
<br><br>
# 현상
Tomcat, Spring Boot 등의 어플리케이션을 실행 후 최초로 REST API 요청 시 Response Time의 딜레이가 많이 긴것을 확인할 수 있다.

<br><br>

# 이유
JVM 프로세스가 지연로딩 방식을 기반으로 하기 때문에 최초의 한번 요청할때 로드되는 과정 때문에 지연시간이 발생한다.

로딩항목은 아래와 같다.
1. Bootstrap Class Loading : Java코드와 `java.lang.Object` 와 같은 **필수 클래스**를 메모리에 로드
2. Extension Class Loading : `java.ext.dirs` 경로에 있는 모든 JAR 파일을 로드 (개발자가  수동으로 JAR 파일을 추가한 경우)
3. Application Class Loading : 어플리케이션 클래스 경로에 있는 **모든 클래스**를 로드

위 클래스를 모두 직접 로딩시켜주면 cold start 현상을 없앨 수 있는데, 직접 로딩시켜주기엔 어려움이 있으니 
최초 요청을 boot시 실행되도록 하여 지연로딩이 완료될 수 있도록 하면 된다.

# 해결방법
Spring Boot에서는 어플리케이션이 최초 로딩 된 후 한번 REST API를 호출하는 방식으로 해결한다. (워머 등록)
또한 1시간 이상 IDLE 상태에 빠져있다가 새로운 요청이 들어오면 동일하게 지연로딩으로 인해 딜레이가 생기므로 
scheduler를 등록하여 주기적으로 API를 요청하여 깨우는 방법도 사용된다.


>이 현상은 어플리케이션이 실행 된 후 최초 한번 또는 
>아주 오랜 시간동안 요청이 전혀 없었을 경우(1시간동안 IDLE 상태에 빠지면)에만 발생하기 때문에 꼭 해결해야하는 문제는 아니다. 
>그러나 MSA 구조와 같이 AutoScaling이 활발하고, 통신이 잦은 서비스에서는 필수적으로 해결하는 것 같다.


# 유의사항
1. 외부 REST API 만 호출하면 되는지?
2. DB 쿼리도 한번 수행을 해야 하는지?



# 적용 결과

비즈니스 로직 : api Request -> cmm request | table select > back request | table select > response
## 로컬환경

적용 전
1. cmm : 380ms
2. back : 395ms
3. total response : 1043ms

적용 후
1. cmm : 37ms
2. back : 80ms
3. total response : 152ms


## 운영환경

적용 전
1차 6486ms
2차 8299ms
3차 6993ms

적용 후
1차 2102ms
2차 1798ms
3차 1405ms

=> 적용 후에도 여전히 2초대를 보이며 느리지만 적용 전에 비해 훨씬 빨라진 걸 확인 할 수 있다.
=> warm up 을 적용했음에도 2번째 3번째 요청과는 응답 속도가 차이 나는 것을 확인 할 수 있다. 이유가 뭘까?



# 적용방법

전제조건
* `/actuator/health`  호출 시 warmup이 끝나야 UP 상태로 함께 변경 해 주어야 한다.

ApplicationWarmer.java - 해당 interface를 상속받은 구현체의 warmup() 메서드를 실행 할 예정
```java

// warm up 시 동작할 내용이 있다면 해당 인터페이스를 상속받아 내용을 구현하면 된다.  
public interface ApplicationWarmer {  
    void warmup();  
}
```


ApplicationHealthIndicator.java - `/actuator/health` 호출 시 warmup 의 상태를 확인 후 상태(UP/DOWN)를 반환한다.
```java
  
  
/**  
 * ApplicationHealthIndicatorConfig.java 에서 bean으로 등록 된다.  
 * /actuator/health 가 request 되면 doHealthCheck() 메서드가 실행된다.  
 * */
@Slf4j  
public class ApplicationHealthIndicator extends AbstractHealthIndicator {  
  
    private final ApplicationWarmerChecker warmer;  
  
    public ApplicationHealthIndicator(ConfigurableApplicationContext context, ApplicationWarmerChecker warmer) {  
       this.warmer = warmer;  
       context.addApplicationListener(this.warmer);  
    }  
  
    @Override  
    protected void doHealthCheck(Health.Builder builder) throws Exception {  
       log.info("request from /actuator/health");  
  
       if (this.warmer.checkWarmup()) {  
          builder.up();  
       } else {  
          builder.outOfService();  
       }  
  
    }  
}
```


ApplicationWarmerChecker.java - 실제 warmer의  실행 여부를 반환하며, 실행되지 않았을 경우 실행 후 결과를 반환한다.
```java
  
  
/**  
 * ApplicationListener<ApplicationReadyEvent> 를 상속받음으로 trigger 포인트가 두개가 된다.  
 * 1. 어플리케이션이 ready 상태가 되면 이벤트 실행  
 * 2. 앞서 ApplicationHealthIndicatorConfig 에서 등록한 ApplicationHealthIndicator 에서 /actuator/health requst가 요청 될 때 마다 실행  
 *  
 * 포인트는 2곳이나 checkWarmup() 에서 synchronized 로 lock을 걸어두어 한번만 실행된다.  
 * */
@Slf4j  
@Component  
@RequiredArgsConstructor  
public class ApplicationWarmerChecker implements ApplicationListener<ApplicationReadyEvent> {  
  
  
    private final Object lockObject = new Object();  
    private final AtomicBoolean completed = new AtomicBoolean(false);  
  
    /* ApplicationWarmer 인터페이스를 상속받은 모든 구현 객체를 가져온다. */  
    private final List<ApplicationWarmer> applicationWarmers;  
  
    public boolean checkWarmup() {  
       synchronized (this.lockObject) {  
          if (!completed.get()) {  
             this.execute();  
          }  
          return completed.get();  
       }  
    }  
  
    private void execute() {  
       log.info("application warmup start");  
       // 아래 warmup(ApplicationWarmer) 메서드를 실행 시킨다.  
       Optional.ofNullable(applicationWarmers)  
          .orElseGet(Collections::emptyList)  
          .forEach(this::warmup);  
       this.completed.set(true);  
       log.info("application warmup completed");  
    }  
  
    private void warmup(ApplicationWarmer warmer) {  
       try {  
          var className = warmer.getClass().getSimpleName();  
          log.info("{} - warmup start", className);  
          warmer.warmup();   // 여기서 ApplicationWarmer를 상속받은 모든 구현 객체의 warmup() 메서드가 실행된다.  
          log.info("{} - warmup end", className);  
       } catch (Exception ex) {  
          log.warn("ApplicationWarmer is Failed - " + ex.getMessage());  
       }  
    }  
  
  
    @Override  
    public void onApplicationEvent(ApplicationReadyEvent event) {  
       checkWarmup();  
    }  
}
```


ApplicationHealthIndicatorConfig.java - actuator health 상태를 모니터링 하기 위해 configuration 등록한다.
```java
  
  
/**  
 * warm up 을 위한 configuration 객체  
 * */  
@Slf4j  
@Configuration  
@RequiredArgsConstructor  
public class ApplicationHealthIndicatorConfig {  
  
    /**  
     * /actuator/health 가 request 될 때 이벤트를 받는 객체  
     * */  
    private final ApplicationWarmerChecker warmer;  
  
    /**  
     * ApplicationWarmerChecker 객체를 baen으로 등록해준다.  
     * */    @Bean  
    ApplicationHealthIndicator applicationHealthIndicator(ApplicationContext context) {  
       var configurableApplicationContext = (ConfigurableApplicationContext)context;  
       return new ApplicationHealthIndicator(configurableApplicationContext, warmer);  
    }  
  
}
```


ApplicationWarmerImpl.java - ApplicationWarmer 인터페이스를 상속받은 실제 warmer 작업 내용을 여기 서술한다.
```java
  
/**  
 * 실제 warmup() 호출 시 실행 될 내용을 기술한다.  
 * */
@Slf4j  
@Component  
@RequiredArgsConstructor  
public class ApplicationWarmerImpl implements ApplicationWarmer{  
  
  
    private final JPAQueryFactory queryFactory;  
    private QAutoUms qAutoUms = new QAutoUms("autoUms");  
  
    private final WarmupClient warmupClient;  
  
    @Override  
    public void warmup() { // 실제 warmup 내용을 구현  
       log.info("ApplicationWarmerImpl warmup start");  
  
       // DB 쿼리 실행 (실행 결과 의미 없음)  
      
  
       // REST API 호출 (실행 결과 의미 없음)  
      
  
    }  
}
```


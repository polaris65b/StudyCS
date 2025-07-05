# @Transactional 고급 주제

## 개요

`@Transactional` 어노테이션의 고급 기능들을 다루는 문서입니다. 트랜잭션 이벤트 처리, 분산 트랜잭션, 마이크로서비스 환경에서의 트랜잭션 관리 등 복잡한 시나리오를 다룹니다.

## 1. 트랜잭션 이벤트 처리

### 1.1 트랜잭션 이벤트 리스너
```java
@Component
public class TransactionEventListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(TransactionEvent event) {
        // 트랜잭션 커밋 후 실행
        log.info("트랜잭션 커밋 완료: {}", event.getMethodName());
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleAfterRollback(TransactionEvent event) {
        // 트랜잭션 롤백 후 실행
        log.error("트랜잭션 롤백 발생: {}", event.getMethodName());
    }
    
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void handleBeforeCommit(TransactionEvent event) {
        // 트랜잭션 커밋 전 실행
        log.info("트랜잭션 커밋 전: {}", event.getMethodName());
    }
}
```

### 1.2 커스텀 트랜잭션 이벤트
```java
public class UserCreatedEvent {
    private final User user;
    private final LocalDateTime createdAt;
    
    public UserCreatedEvent(User user) {
        this.user = user;
        this.createdAt = LocalDateTime.now();
    }
    
    // getters...
}

@Service
@Transactional
public class UserService {
    
    private final ApplicationEventPublisher eventPublisher;
    
    public User createUser(User user) {
        User savedUser = userRepository.save(user);
        
        // 트랜잭션 이벤트 발행
        eventPublisher.publishEvent(new UserCreatedEvent(savedUser));
        
        return savedUser;
    }
}
```

## 2. 트랜잭션 동기화

### 2.1 TransactionSynchronizationManager 사용
```java
@Service
public class TransactionSynchronizationService {
    
    @Transactional
    public void methodWithSynchronization() {
        // 트랜잭션 동기화 등록
        TransactionSynchronizationManager.registerSynchronization(
            new TransactionSynchronizationAdapter() {
                @Override
                public void afterCommit() {
                    // 트랜잭션 커밋 후 실행할 작업
                    log.info("트랜잭션 커밋 후 작업 실행");
                }
                
                @Override
                public void afterCompletion(int status) {
                    // 트랜잭션 완료 후 실행 (성공/실패 관계없이)
                    if (status == TransactionSynchronization.STATUS_COMMITTED) {
                        log.info("트랜잭션 성공");
                    } else {
                        log.info("트랜잭션 실패");
                    }
                }
            }
        );
        
        // 비즈니스 로직 실행
        userRepository.save(new User("John"));
    }
}
```

## 3. 분산 트랜잭션

### 3.1 JTA (Java Transaction API) 사용
```java
@Service
public class DistributedTransactionService {
    
    @Transactional(transactionManager = "jtaTransactionManager")
    public void distributedOperation() {
        // 여러 데이터베이스에 걸친 트랜잭션
        userRepository.save(new User("John"));
        orderRepository.save(new Order("ORDER-001"));
        paymentRepository.save(new Payment("PAY-001"));
    }
}
```

### 3.2 XA 트랜잭션 설정
```yaml
# application.yml
spring:
  datasource:
    xa:
      properties:
        xaDataSourceClassName: com.mysql.cj.jdbc.MysqlXADataSource
        url: jdbc:mysql://localhost:3306/testdb
        user: root
        password: password
        serverTimezone: UTC
```

## 4. 마이크로서비스 환경에서의 트랜잭션

### 4.1 Saga 패턴
```java
@Service
public class OrderSagaService {
    
    @Transactional
    public void createOrderWithSaga(Order order) {
        try {
            // 1단계: 주문 생성
            Order savedOrder = orderRepository.save(order);
            
            // 2단계: 재고 확인
            inventoryService.reserveInventory(order.getItems());
            
            // 3단계: 결제 처리
            paymentService.processPayment(order.getPaymentInfo());
            
            // 4단계: 배송 준비
            shippingService.prepareShipping(order);
            
        } catch (Exception e) {
            // 실패 시 보상 트랜잭션 실행
            compensateOrder(order);
            throw e;
        }
    }
    
    @Transactional
    public void compensateOrder(Order order) {
        // 보상 트랜잭션
        inventoryService.releaseInventory(order.getItems());
        paymentService.refundPayment(order.getPaymentInfo());
        shippingService.cancelShipping(order);
    }
}
```

### 4.2 분산 트랜잭션 모니터링
```java
@Component
public class DistributedTransactionMonitor {
    
    private final MeterRegistry meterRegistry;
    
    public DistributedTransactionMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @EventListener
    public void monitorDistributedTransaction(TransactionEvent event) {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        // 분산 트랜잭션 지표 수집
        Counter.builder("distributed.transaction.total")
            .tag("service", "order-service")
            .register(meterRegistry)
            .increment();
        
        sample.stop(Timer.builder("distributed.transaction.duration")
            .tag("service", "order-service")
            .register(meterRegistry));
    }
}
```

## 5. 트랜잭션 보안

### 5.1 트랜잭션 권한 검증
```java
@Service
public class SecureTransactionService {
    
    @Transactional
    @PreAuthorize("hasRole('ADMIN')")
    public void adminOnlyOperation() {
        // 관리자만 접근 가능한 트랜잭션
        userRepository.deleteAll();
    }
    
    @Transactional(readOnly = true)
    @PreAuthorize("hasRole('USER')")
    public List<User> userReadOperation() {
        // 일반 사용자 읽기 전용 트랜잭션
        return userRepository.findAll();
    }
}
```

### 5.2 트랜잭션 감사 로그
```java
@Aspect
@Component
public class TransactionAuditAspect {
    
    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object auditTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().getName();
        String username = SecurityContextHolder.getContext().getAuthentication().getName();
        
        log.info("트랜잭션 시작 - 메서드: {}, 사용자: {}, 시간: {}", 
            methodName, username, LocalDateTime.now());
        
        try {
            Object result = joinPoint.proceed();
            log.info("트랜잭션 성공 - 메서드: {}, 사용자: {}", methodName, username);
            return result;
        } catch (Exception e) {
            log.error("트랜잭션 실패 - 메서드: {}, 사용자: {}, 예외: {}", 
                methodName, username, e.getMessage());
            throw e;
        }
    }
}
```

## 6. 트랜잭션 디버깅

### 6.1 트랜잭션 로그 설정
```xml
<!-- logback-spring.xml -->
<logger name="org.springframework.transaction" level="DEBUG"/>
<logger name="org.springframework.orm.jpa" level="DEBUG"/>
<logger name="org.hibernate.SQL" level="DEBUG"/>
<logger name="org.hibernate.type.descriptor.sql.BasicBinder" level="TRACE"/>
```

### 6.2 트랜잭션 상태 모니터링
```java
@Component
public class TransactionDebugger {
    
    @EventListener
    public void debugTransaction(TransactionEvent event) {
        log.debug("=== 트랜잭션 디버그 정보 ===");
        log.debug("메서드: {}", event.getMethodName());
        log.debug("읽기 전용: {}", event.isReadOnly());
        log.debug("격리 수준: {}", event.getIsolationLevel());
        log.debug("전파 속성: {}", event.getPropagation());
        log.debug("타임아웃: {}초", event.getTimeout());
        log.debug("롤백 예외: {}", event.getRollbackFor());
        log.debug("실행 시간: {}ms", event.getDuration());
        log.debug("================================");
    }
}
```

## 7. 성능 최적화 고급 기법

### 7.1 배치 처리 최적화
```java
@Service
public class BatchProcessingService {
    
    @Transactional
    public void batchInsertUsers(List<User> users) {
        // 배치 크기 설정으로 성능 최적화
        int batchSize = 100;
        
        for (int i = 0; i < users.size(); i += batchSize) {
            int endIndex = Math.min(i + batchSize, users.size());
            List<User> batch = users.subList(i, endIndex);
            
            userRepository.saveAll(batch);
            
            // 배치마다 플러시하여 메모리 관리
            entityManager.flush();
            entityManager.clear();
        }
    }
    
    @Transactional(readOnly = true)
    public List<User> batchReadUsers(List<Long> userIds) {
        // 배치 조회로 성능 최적화
        return userRepository.findAllById(userIds);
    }
}
```

### 7.2 커넥션 풀 최적화
```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

## 8. 트랜잭션 테스트

### 8.1 통합 테스트
```java
@SpringBootTest
@Transactional
class TransactionIntegrationTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void testTransactionRollback() {
        // 테스트 메서드가 끝나면 자동으로 롤백
        User user = new User("TestUser");
        userService.createUser(user);
        
        assertThat(userRepository.findById(user.getId())).isPresent();
    }
    
    @Test
    @Rollback(false)
    void testTransactionCommit() {
        // 롤백하지 않고 커밋
        User user = new User("TestUser");
        userService.createUser(user);
        
        assertThat(userRepository.findById(user.getId())).isPresent();
    }
}
```

### 8.2 트랜잭션 격리 수준 테스트
```java
@SpringBootTest
class TransactionIsolationTest {
    
    @Test
    void testDirtyRead() {
        // READ_UNCOMMITTED 격리 수준 테스트
        CountDownLatch latch = new CountDownLatch(2);
        
        Thread thread1 = new Thread(() -> {
            @Transactional(isolation = Isolation.READ_UNCOMMITTED)
            public void dirtyReadTest() {
                // 미커밋 데이터 읽기 가능
                List<User> users = userRepository.findAll();
                latch.countDown();
            }
        });
        
        Thread thread2 = new Thread(() -> {
            @Transactional
            public void updateWithoutCommit() {
                userRepository.save(new User("DirtyUser"));
                // 커밋하지 않고 종료
                latch.countDown();
            }
        });
        
        thread1.start();
        thread2.start();
        latch.await();
    }
}
```

## 9. 결론

고급 트랜잭션 기능들을 활용하면 복잡한 비즈니스 로직을 안전하고 효율적으로 처리할 수 있습니다. 특히 분산 환경이나 마이크로서비스 아키텍처에서는 이러한 고급 기능들이 필수적입니다.

### 핵심 포인트
1. **이벤트 기반 처리**: 트랜잭션 이벤트를 활용한 비동기 처리
2. **분산 트랜잭션**: JTA와 XA를 활용한 다중 데이터베이스 처리
3. **보안 강화**: 권한 기반 트랜잭션 제어
4. **모니터링**: 상세한 트랜잭션 모니터링 및 디버깅
5. **테스트**: 다양한 시나리오에 대한 철저한 테스트

## 참고 자료
- [@Transactional 기본 개념](./@Transactional_기본개념.md)
- [@Transactional 성능 최적화](./@Transactional_성능최적화.md)
- [@Transactional 실무 적용](./@Transactional_실무적용.md)
- Spring Framework 고급 가이드
- 분산 트랜잭션 패턴 
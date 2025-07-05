# @Transactional 성능 최적화

## 개요

`@Transactional` 어노테이션의 성능 최적화는 Spring 애플리케이션의 전체적인 성능에 큰 영향을 미칩니다. 특히 `readOnly` 속성과 다양한 최적화 기법을 통해 데이터베이스 리소스를 효율적으로 사용할 수 있습니다.

## 1. readOnly 속성 상세 분석

### 1.1 readOnly = true의 효과

#### 1.1.1 성능 최적화
```java
@Service
@Transactional(readOnly = true)
public class OptimizedQueryService {
    
    public List<User> findAllUsers() {
        // 데이터베이스 최적화:
        // - 읽기 전용 락만 사용
        // - 롤백 로그 생성 안함
        // - 메모리 사용량 감소
        return userRepository.findAll();
    }
    
    public User findUserById(Long id) {
        // CPU 사용량 감소:
        // - 락 관리 오버헤드 없음
        // - 트랜잭션 ID 할당 최소화
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

#### 1.1.2 동시성 개선
```java
@Service
@Transactional(readOnly = true)
public class ConcurrentService {
    
    public void concurrentReadOperations() {
        // 여러 스레드가 동시에 실행해도
        // 읽기 전용 락만 사용하므로 충돌 없음
        List<User> users = userRepository.findAll();
        
        // 긴 처리 시간이어도 다른 읽기 트랜잭션과 충돌 없음
        processUsers(users);
    }
}
```

### 1.2 readOnly 사용 시 주의사항

#### 1.2.1 금지되는 작업들
```java
@Service
@Transactional(readOnly = true)
public class ProblematicService {
    
    public void invalidOperations() {
        // ❌ 다음 작업들은 예외 발생
        
        // INSERT 작업
        userRepository.save(new User("John"));
        
        // UPDATE 작업
        userRepository.updateName(1L, "NewName");
        
        // DELETE 작업
        userRepository.deleteById(1L);
        
        // 엔티티 상태 변경
        User user = userRepository.findById(1L).orElse(null);
        user.setName("NewName"); // ❌ 예외 발생
        userRepository.save(user); // ❌ 예외 발생
    }
}
```

#### 1.2.2 올바른 사용법
```java
@Service
public class CorrectService {
    
    @Transactional(readOnly = true)
    public List<User> findAllUsers() {
        // ✅ 읽기 전용 작업만 수행
        return userRepository.findAll();
    }
    
    @Transactional
    public User createUser(User user) {
        // ✅ 쓰기 작업은 일반 트랜잭션 사용
        return userRepository.save(user);
    }
    
    @Transactional
    public User updateUser(Long id, String newName) {
        // ✅ 쓰기 작업은 일반 트랜잭션 사용
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        user.setName(newName);
        return userRepository.save(user);
    }
}
```

## 2. 성능 비교 및 벤치마크

### 2.1 readOnly vs 일반 트랜잭션 비교

#### 2.1.1 코드 비교
```java
// 일반 트랜잭션
@Transactional
public List<User> findAllUsers() {
    return userRepository.findAll();
}

// 읽기 전용 트랜잭션
@Transactional(readOnly = true)
public List<User> findAllUsersReadOnly() {
    return userRepository.findAll();
}
```

#### 2.1.2 성능 테스트
```java
@SpringBootTest
class TransactionPerformanceTest {
    
    @Autowired
    private UserService userService;
    
    @Test
    void comparePerformance() {
        int iterations = 1000;
        
        // 읽기 전용 트랜잭션 테스트
        long readOnlyStart = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            userService.findAllUsersReadOnly();
        }
        long readOnlyTime = System.nanoTime() - readOnlyStart;
        
        // 일반 트랜잭션 테스트
        long normalStart = System.nanoTime();
        for (int i = 0; i < iterations; i++) {
            userService.findAllUsers();
        }
        long normalTime = System.nanoTime() - normalStart;
        
        log.info("읽기 전용 트랜잭션 평균 시간: {} ns", readOnlyTime / iterations);
        log.info("일반 트랜잭션 평균 시간: {} ns", normalTime / iterations);
        log.info("성능 향상률: {}%", 
            ((double)(normalTime - readOnlyTime) / normalTime) * 100);
    }
}
```

### 2.2 리소스 사용량 비교

| 항목 | 일반 트랜잭션 | 읽기 전용 트랜잭션 | 개선율 |
|------|---------------|-------------------|--------|
| 메모리 사용량 | 100% | 60-70% | 30-40% 감소 |
| CPU 사용량 | 100% | 70-80% | 20-30% 감소 |
| 데이터베이스 락 | 쓰기 락 준비 | 읽기 락만 | 락 오버헤드 감소 |
| 롤백 로그 | 생성 | 생성 안함 | 100% 감소 |
| 동시성 | 제한적 | 높음 | 상당한 개선 |

### 2.3 실제 벤치마크 결과 예시

#### 2.3.1 대용량 데이터 조회 테스트
```java
@SpringBootTest
class LargeDataBenchmarkTest {
    
    @Test
    void benchmarkLargeDataQuery() {
        // 100만 건 데이터 조회 테스트
        
        // 읽기 전용 트랜잭션
        long readOnlyStart = System.currentTimeMillis();
        List<User> readOnlyResult = userService.findLargeDataSetReadOnly();
        long readOnlyTime = System.currentTimeMillis() - readOnlyStart;
        
        // 일반 트랜잭션
        long normalStart = System.currentTimeMillis();
        List<User> normalResult = userService.findLargeDataSet();
        long normalTime = System.currentTimeMillis() - normalStart;
        
        log.info("=== 대용량 데이터 조회 벤치마크 ===");
        log.info("읽기 전용 트랜잭션: {} ms", readOnlyTime);
        log.info("일반 트랜잭션: {} ms", normalTime);
        log.info("성능 향상: {} ms ({}%)", 
            normalTime - readOnlyTime, 
            ((double)(normalTime - readOnlyTime) / normalTime) * 100);
    }
}
```

## 3. 성능 최적화 고급 기법

### 3.1 배치 처리 최적화
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

### 3.2 커넥션 풀 최적화
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

### 3.3 메모리 사용량 비교
```java
@Component
public class MemoryUsageMonitor {
    
    @EventListener
    public void monitorMemoryUsage(TransactionEvent event) {
        Runtime runtime = Runtime.getRuntime();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        
        if (event.isReadOnly()) {
            log.info("읽기 전용 트랜잭션 - 메모리 사용량: {} MB", usedMemory / 1024 / 1024);
        } else {
            log.info("일반 트랜잭션 - 메모리 사용량: {} MB", usedMemory / 1024 / 1024);
        }
    }
}
```

## 4. 동시성 처리 차이

### 4.1 일반 트랜잭션 (동시성 문제 가능)
```java
@Service
public class ProblematicService {
    
    @Transactional
    public void problematicMethod() {
        // 여러 스레드가 동시에 실행할 때
        // 락 경합(lock contention) 발생 가능
        List<User> users = userRepository.findAll();
        
        // 긴 처리 시간으로 인한 락 유지
        Thread.sleep(1000);
        
        // 다른 트랜잭션들이 대기해야 함
    }
}
```

### 4.2 읽기 전용 트랜잭션 (동시성 문제 최소화)
```java
@Service
public class OptimizedService {
    
    @Transactional(readOnly = true)
    public void optimizedMethod() {
        // 여러 스레드가 동시에 실행해도
        // 읽기 전용 락만 사용하므로 충돌 없음
        List<User> users = userRepository.findAll();
        
        // 긴 처리 시간이어도 다른 읽기 트랜잭션과 충돌 없음
        Thread.sleep(1000);
        
        // 다른 읽기 트랜잭션들이 대기하지 않음
    }
}
```

## 5. 실제 프로덕션 환경에서의 차이

### 5.1 모니터링 지표
```java
@Component
public class ProductionMonitor {
    
    @EventListener
    public void monitorTransactionPerformance(TransactionEvent event) {
        if (event.isReadOnly()) {
            // 읽기 전용 트랜잭션 지표
            incrementCounter("readonly.transactions");
            recordLatency("readonly.latency", event.getDuration());
        } else {
            // 일반 트랜잭션 지표
            incrementCounter("normal.transactions");
            recordLatency("normal.latency", event.getDuration());
        }
    }
}
```

### 5.2 성능 개선 효과
```java
@Service
public class PerformanceReportService {
    
    public PerformanceReport generateReport() {
        return PerformanceReport.builder()
            .readOnlyTransactionCount(getReadOnlyCount())
            .normalTransactionCount(getNormalCount())
            .averageReadOnlyLatency(getReadOnlyLatency())
            .averageNormalLatency(getNormalLatency())
            .performanceImprovement(calculateImprovement())
            .build();
    }
}
```

## 6. 성능 문제 해결

### 6.1 N+1 문제 해결
```java
@Service
public class NPlusOneSolutionService {
    
    @Transactional(readOnly = true)
    public List<UserWithOrders> getUsersWithOrders() {
        // ❌ N+1 문제 발생
        List<User> users = userRepository.findAll();
        users.forEach(user -> {
            List<Order> orders = orderRepository.findByUser(user);
            user.setOrders(orders);
        });
        
        // ✅ 해결 방법: JOIN FETCH 사용
        return userRepository.findAllWithOrders();
    }
}
```

### 6.2 대용량 데이터 처리
```java
@Service
public class LargeDataProcessingService {
    
    @Transactional
    public void processLargeDataset() {
        // ✅ 배치 처리로 메모리 사용량 제어
        int batchSize = 1000;
        int offset = 0;
        
        while (true) {
            List<User> batch = userRepository.findBatch(offset, batchSize);
            if (batch.isEmpty()) break;
            
            processBatch(batch);
            offset += batchSize;
            
            // 배치마다 플러시하여 메모리 관리
            entityManager.flush();
            entityManager.clear();
        }
    }
}
```

## 7. 베스트 프랙티스

### 7.1 적절한 사용 시나리오

#### 7.1.1 readOnly = true 사용 시
- ✅ 단순 조회 작업
- ✅ 통계 데이터 조회
- ✅ 리포트 생성
- ✅ 대시보드 데이터 조회
- ✅ 복잡한 조회 로직
- ✅ 데이터 검증 (읽기만)

#### 7.1.2 일반 트랜잭션 사용 시
- ✅ 데이터 생성 (INSERT)
- ✅ 데이터 수정 (UPDATE)
- ✅ 데이터 삭제 (DELETE)
- ✅ 복합 작업 (읽기 + 쓰기)
- ✅ 비즈니스 로직 처리

### 7.2 설계 패턴

#### 7.2.1 CQRS 패턴 적용
```java
// Command (쓰기 작업)
@Service
@Transactional
public class UserCommandService {
    public User createUser(User user) { /* ... */ }
    public User updateUser(Long id, User user) { /* ... */ }
    public void deleteUser(Long id) { /* ... */ }
}

// Query (읽기 작업)
@Service
@Transactional(readOnly = true)
public class UserQueryService {
    public List<User> findAllUsers() { /* ... */ }
    public User findUserById(Long id) { /* ... */ }
    public List<User> findUsersByAge(int age) { /* ... */ }
}
```

#### 7.2.2 계층별 트랜잭션 관리
```java
// Repository 레이어
@Repository
public class UserRepository extends JpaRepository<User, Long> {
    // 기본적으로 트랜잭션 없음
    // Service 레이어에서 트랜잭션 관리
}

// Service 레이어
@Service
@Transactional(readOnly = true)
public class UserService {
    // 클래스 레벨에서 읽기 전용 설정
    // 특정 메서드에서만 @Transactional로 오버라이드
}
```

## 8. 성능 최적화 체크리스트

- [ ] 읽기 작업에 readOnly = true 적용
- [ ] 배치 처리 구현
- [ ] N+1 문제 해결
- [ ] 커넥션 풀 최적화
- [ ] 인덱스 최적화
- [ ] 캐싱 전략 수립
- [ ] 모니터링 지표 설정
- [ ] 로그 레벨 조정

## 참고 자료
- [@Transactional 기본 개념](./@Transactional_기본개념.md)
- [@Transactional 실무 적용](./@Transactional_실무적용.md)
- [@Transactional 고급 주제](./@Transactional_고급주제.md)
- Spring Boot 성능 가이드 
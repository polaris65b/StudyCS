# @Transactional 실무 적용

## 개요

실무에서 `@Transactional` 어노테이션을 효과적으로 활용하기 위한 다양한 패턴과 적용 사례를 다룹니다. 실제 비즈니스 시나리오를 기반으로 한 구체적인 예제들을 제공합니다.

## 1. 실제 사용 패턴

### 1.1 서비스 레이어 분리 패턴

#### 1.1.1 읽기 전용 서비스
```java
@Service
@Transactional(readOnly = true)
public class UserQueryService {
    
    private final UserRepository userRepository;
    
    public UserQueryService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public List<User> findAllUsers() {
        return userRepository.findAll();
    }
    
    public User findUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    public List<User> findUsersByAge(int age) {
        return userRepository.findByAge(age);
    }
    
    public long countUsers() {
        return userRepository.count();
    }
    
    public List<User> findUsersByNameContaining(String name) {
        return userRepository.findByNameContaining(name);
    }
}
```

#### 1.1.2 쓰기 전용 서비스
```java
@Service
@Transactional
public class UserCommandService {
    
    private final UserRepository userRepository;
    
    public UserCommandService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    public User updateUser(Long id, User userDetails) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        
        user.setName(userDetails.getName());
        user.setEmail(userDetails.getEmail());
        user.setAge(userDetails.getAge());
        
        return userRepository.save(user);
    }
    
    public void deleteUser(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        userRepository.delete(user);
    }
    
    public void updateUserStatus(Long id, UserStatus status) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        user.setStatus(status);
        userRepository.save(user);
    }
}
```

### 1.2 복합 트랜잭션 패턴

```java
@Service
public class ComplexTransactionService {
    
    private final UserRepository userRepository;
    private final OrderRepository orderRepository;
    
    public ComplexTransactionService(UserRepository userRepository, 
                                   OrderRepository orderRepository) {
        this.userRepository = userRepository;
        this.orderRepository = orderRepository;
    }
    
    @Transactional
    public Order createOrderWithUser(Long userId, Order order) {
        // 사용자 조회 (읽기 전용이지만 같은 트랜잭션에서)
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        // 주문 생성 (쓰기 작업)
        order.setUser(user);
        return orderRepository.save(order);
    }
    
    @Transactional(readOnly = true)
    public OrderSummary getOrderSummary(Long userId) {
        // 읽기 전용 트랜잭션으로 성능 최적화
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        List<Order> orders = orderRepository.findByUser(user);
        
        return OrderSummary.builder()
            .userName(user.getName())
            .totalOrders(orders.size())
            .totalAmount(orders.stream()
                .mapToDouble(Order::getAmount)
                .sum())
            .build();
    }
}
```

## 2. 실무 적용 사례

### 2.1 이커머스 시스템
```java
@Service
public class EcommerceTransactionService {
    
    @Transactional
    public Order processOrder(OrderRequest request) {
        // 1. 재고 확인 및 예약
        inventoryService.reserveInventory(request.getItems());
        
        // 2. 주문 생성
        Order order = orderRepository.save(new Order(request));
        
        // 3. 결제 처리
        Payment payment = paymentService.processPayment(request.getPaymentInfo());
        order.setPayment(payment);
        
        // 4. 배송 정보 생성
        Shipping shipping = shippingService.createShipping(order);
        order.setShipping(shipping);
        
        return orderRepository.save(order);
    }
    
    @Transactional(readOnly = true)
    public OrderSummary getOrderSummary(Long userId) {
        // 읽기 전용으로 성능 최적화
        List<Order> orders = orderRepository.findByUserId(userId);
        
        return OrderSummary.builder()
            .totalOrders(orders.size())
            .totalAmount(orders.stream()
                .mapToDouble(Order::getTotalAmount)
                .sum())
            .build();
    }
}
```

### 2.2 금융 시스템
```java
@Service
public class BankingTransactionService {
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void transferMoney(Long fromAccountId, Long toAccountId, BigDecimal amount) {
        // 가장 높은 격리 수준으로 데이터 일관성 보장
        
        Account fromAccount = accountRepository.findById(fromAccountId)
            .orElseThrow(() -> new AccountNotFoundException(fromAccountId));
        
        Account toAccount = accountRepository.findById(toAccountId)
            .orElseThrow(() -> new AccountNotFoundException(toAccountId));
        
        // 잔액 확인
        if (fromAccount.getBalance().compareTo(amount) < 0) {
            throw new InsufficientBalanceException();
        }
        
        // 이체 처리
        fromAccount.withdraw(amount);
        toAccount.deposit(amount);
        
        accountRepository.save(fromAccount);
        accountRepository.save(toAccount);
        
        // 거래 내역 기록
        transactionRepository.save(new Transaction(fromAccount, toAccount, amount));
    }
}
```

### 2.3 소셜 미디어 시스템
```java
@Service
public class SocialMediaTransactionService {
    
    @Transactional
    public Post createPostWithMedia(PostRequest request) {
        // 포스트 생성
        Post post = postRepository.save(new Post(request));
        
        // 미디어 파일 처리
        for (MediaFile mediaFile : request.getMediaFiles()) {
            String uploadedUrl = mediaService.uploadFile(mediaFile);
            post.addMedia(new PostMedia(uploadedUrl, mediaFile.getType()));
        }
        
        // 해시태그 처리
        for (String hashtag : request.getHashtags()) {
            post.addHashtag(hashtagRepository.findOrCreate(hashtag));
        }
        
        // 알림 발송
        notificationService.notifyFollowers(post.getAuthor(), post);
        
        return postRepository.save(post);
    }
    
    @Transactional(readOnly = true)
    public FeedResponse getFeed(Long userId, int page, int size) {
        // 읽기 전용으로 피드 조회 성능 최적화
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException(userId));
        
        Page<Post> posts = postRepository.findFeedByUser(user, PageRequest.of(page, size));
        
        return FeedResponse.builder()
            .posts(posts.getContent())
            .hasNext(posts.hasNext())
            .totalElements(posts.getTotalElements())
            .build();
    }
}
```

## 3. 모니터링과 디버깅

### 3.1 트랜잭션 모니터링
```java
@Component
public class TransactionMonitor {
    
    @EventListener
    public void handleTransactionEvent(TransactionEvent event) {
        log.info("트랜잭션 이벤트: {}", event.getMethodName());
        log.info("읽기 전용: {}", event.isReadOnly());
        log.info("격리 수준: {}", event.getIsolationLevel());
        log.info("전파 속성: {}", event.getPropagation());
        log.info("실행 시간: {} ms", event.getDuration());
    }
}
```

### 3.2 성능 지표 수집
```java
@Component
public class PerformanceMetrics {
    
    private final MeterRegistry meterRegistry;
    
    public PerformanceMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @EventListener
    public void recordTransactionMetrics(TransactionEvent event) {
        if (event.isReadOnly()) {
            Counter.builder("transaction.readonly")
                .description("읽기 전용 트랜잭션 수")
                .register(meterRegistry)
                .increment();
        } else {
            Counter.builder("transaction.normal")
                .description("일반 트랜잭션 수")
                .register(meterRegistry)
                .increment();
        }
    }
}
```

## 4. 트러블슈팅 가이드

### 4.1 일반적인 문제들

#### 4.1.1 트랜잭션이 적용되지 않는 경우
```java
// ❌ 문제가 있는 코드
@Service
public class ProblematicService {
    
    public void methodWithoutTransaction() {
        // @Transactional이 적용되지 않음
        userRepository.save(new User("John"));
    }
    
    @Transactional
    public void methodWithTransaction() {
        methodWithoutTransaction(); // 내부 호출로 인해 트랜잭션 적용 안됨
    }
}

// ✅ 해결 방법
@Service
public class CorrectService {
    
    @Autowired
    private CorrectService self; // 자기 자신을 주입
    
    @Transactional
    public void methodWithTransaction() {
        self.methodWithoutTransaction(); // 프록시를 통한 호출
    }
    
    @Transactional
    public void methodWithoutTransaction() {
        userRepository.save(new User("John"));
    }
}
```

#### 4.1.2 데드락 해결
```java
@Service
public class DeadlockPreventionService {
    
    @Transactional
    public void updateUsersInOrder(List<Long> userIds) {
        // ID 순서대로 정렬하여 데드락 방지
        List<Long> sortedIds = userIds.stream()
            .sorted()
            .collect(Collectors.toList());
        
        for (Long userId : sortedIds) {
            User user = userRepository.findById(userId)
                .orElseThrow(() -> new UserNotFoundException(userId));
            user.setStatus(UserStatus.ACTIVE);
            userRepository.save(user);
        }
    }
}
```

### 4.2 성능 문제 해결

#### 4.2.1 N+1 문제 해결
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

#### 4.2.2 대용량 데이터 처리
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

## 5. 베스트 프랙티스

### 5.1 적절한 사용 시나리오

#### 5.1.1 readOnly = true 사용 시
- ✅ 단순 조회 작업
- ✅ 통계 데이터 조회
- ✅ 리포트 생성
- ✅ 대시보드 데이터 조회
- ✅ 복잡한 조회 로직
- ✅ 데이터 검증 (읽기만)

#### 5.1.2 일반 트랜잭션 사용 시
- ✅ 데이터 생성 (INSERT)
- ✅ 데이터 수정 (UPDATE)
- ✅ 데이터 삭제 (DELETE)
- ✅ 복합 작업 (읽기 + 쓰기)
- ✅ 비즈니스 로직 처리

### 5.2 설계 패턴

#### 5.2.1 CQRS 패턴 적용
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

#### 5.2.2 계층별 트랜잭션 관리
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

## 6. 트랜잭션 테스트

### 6.1 통합 테스트
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

### 6.2 트랜잭션 격리 수준 테스트
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

## 7. 결론 및 체크리스트

### 7.1 트랜잭션 설계 체크리스트

- [ ] 적절한 전파 속성 선택 (REQUIRED, REQUIRES_NEW 등)
- [ ] 격리 수준 설정 (READ_COMMITTED, SERIALIZABLE 등)
- [ ] 읽기 전용 트랜잭션 활용 (readOnly = true)
- [ ] 타임아웃 설정 (timeout 속성)
- [ ] 롤백 조건 정의 (rollbackFor, noRollbackFor)
- [ ] 서비스 레이어 분리 (읽기/쓰기)
- [ ] 예외 처리 전략 수립
- [ ] 성능 모니터링 구현
- [ ] 테스트 코드 작성
- [ ] 보안 고려사항 적용

### 7.2 성능 최적화 체크리스트

- [ ] 읽기 작업에 readOnly = true 적용
- [ ] 배치 처리 구현
- [ ] N+1 문제 해결
- [ ] 커넥션 풀 최적화
- [ ] 인덱스 최적화
- [ ] 캐싱 전략 수립
- [ ] 모니터링 지표 설정
- [ ] 로그 레벨 조정

### 7.3 운영 체크리스트

- [ ] 트랜잭션 모니터링 설정
- [ ] 알림 설정 (타임아웃, 데드락 등)
- [ ] 백업 및 복구 전략
- [ ] 장애 대응 매뉴얼
- [ ] 성능 테스트 수행
- [ ] 보안 감사 수행

## 참고 자료
- [@Transactional 기본 개념](./@Transactional_기본개념.md)
- [@Transactional 성능 최적화](./@Transactional_성능최적화.md)
- [@Transactional 고급 주제](./@Transactional_고급주제.md)
- Spring Boot 실무 가이드 
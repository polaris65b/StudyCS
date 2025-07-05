# @Transactional 어노테이션 기본 개념

## 개요

`@Transactional` 어노테이션은 Spring Framework에서 제공하는 선언적 트랜잭션 관리 방식으로, 메서드나 클래스에 트랜잭션 동작을 정의할 수 있게 해줍니다.

## 1. @Transactional 기본 개념

### 1.1 정의
- **선언적 트랜잭션**: 코드에 직접 트랜잭션 로직을 작성하지 않고 어노테이션으로 정의
- **AOP 기반**: Spring AOP를 사용하여 트랜잭션 로직을 자동으로 적용
- **자동 관리**: 트랜잭션 시작, 커밋, 롤백을 자동으로 처리

### 1.2 기본 사용법
```java
@Service
public class UserService {
    
    @Transactional
    public User createUser(User user) {
        // 트랜잭션이 자동으로 시작됨
        User savedUser = userRepository.save(user);
        // 메서드가 정상 종료되면 자동으로 커밋됨
        return savedUser;
    }
}
```

## 2. @Transactional 속성들

### 2.1 propagation (전파 속성)

#### 2.1.1 기본값: REQUIRED
```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodWithRequired() {
    // 기존 트랜잭션이 있으면 참여, 없으면 새로 생성
}
```

#### 2.1.2 다른 전파 속성들
```java
@Service
public class TransactionalService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodWithRequiresNew() {
        // 항상 새로운 트랜잭션 생성
    }
    
    @Transactional(propagation = Propagation.SUPPORTS)
    public void methodWithSupports() {
        // 기존 트랜잭션이 있으면 참여, 없으면 트랜잭션 없이 실행
    }
    
    @Transactional(propagation = Propagation.NOT_SUPPORTED)
    public void methodWithNotSupported() {
        // 트랜잭션 없이 실행
    }
    
    @Transactional(propagation = Propagation.MANDATORY)
    public void methodWithMandatory() {
        // 반드시 기존 트랜잭션이 있어야 함
    }
    
    @Transactional(propagation = Propagation.NEVER)
    public void methodWithNever() {
        // 트랜잭션이 있으면 예외 발생
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void methodWithNested() {
        // 중첩 트랜잭션 생성 (지원하는 DB만)
    }
}
```

### 2.2 isolation (격리 수준)

#### 2.2.1 기본값: DEFAULT
```java
@Transactional(isolation = Isolation.DEFAULT)
public void methodWithDefaultIsolation() {
    // 데이터베이스 기본 격리 수준 사용
}
```

#### 2.2.2 다른 격리 수준들
```java
@Service
public class IsolationService {
    
    @Transactional(isolation = Isolation.READ_UNCOMMITTED)
    public void methodWithReadUncommitted() {
        // 가장 낮은 격리 수준 - Dirty Read 허용
    }
    
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public void methodWithReadCommitted() {
        // 커밋된 데이터만 읽기 - Dirty Read 방지
    }
    
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public void methodWithRepeatableRead() {
        // 반복 읽기 시 같은 결과 보장 - Phantom Read 방지
    }
    
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void methodWithSerializable() {
        // 가장 높은 격리 수준 - 모든 동시성 문제 방지
    }
}
```

### 2.3 timeout (타임아웃)

```java
@Service
public class TimeoutService {
    
    @Transactional(timeout = 30)
    public void methodWithTimeout() {
        // 30초 후 타임아웃 발생
    }
    
    @Transactional(timeout = 60)
    public void longRunningMethod() {
        // 60초 후 타임아웃 발생
        // 긴 처리 시간이 필요한 작업
    }
}
```

### 2.4 readOnly (읽기 전용)

#### 2.4.1 기본값: false
```java
@Transactional(readOnly = false)
public void methodWithReadWrite() {
    // 읽기/쓰기 가능한 트랜잭션
}
```

#### 2.4.2 읽기 전용 트랜잭션
```java
@Service
public class ReadOnlyService {
    
    @Transactional(readOnly = true)
    public List<User> findAllUsers() {
        // 읽기 전용 트랜잭션 - 성능 최적화
        return userRepository.findAll();
    }
    
    @Transactional(readOnly = true)
    public User findUserById(Long id) {
        // 읽기 전용 트랜잭션
        return userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
}
```

### 2.5 rollbackFor (롤백 조건)

```java
@Service
public class RollbackService {
    
    @Transactional(rollbackFor = Exception.class)
    public void methodWithRollbackForException() {
        // 모든 예외에 대해 롤백
    }
    
    @Transactional(rollbackFor = {UserNotFoundException.class, ValidationException.class})
    public void methodWithSpecificRollback() {
        // 특정 예외들에 대해서만 롤백
    }
    
    @Transactional(noRollbackFor = IllegalArgumentException.class)
    public void methodWithNoRollbackFor() {
        // 특정 예외에 대해서는 롤백하지 않음
    }
}
```

## 3. 데이터베이스별 동작 방식

### 3.1 MySQL
```sql
-- 일반 트랜잭션
START TRANSACTION;
SELECT * FROM users WHERE id = 1;
-- 데이터베이스가 쓰기 가능한 상태로 락을 유지
-- 롤백 로그 준비
COMMIT;

-- 읽기 전용 트랜잭션
START TRANSACTION READ ONLY;
SELECT * FROM users WHERE id = 1;
-- 읽기 전용 락만 사용
-- 롤백 로그 생성 안함
COMMIT;
```

### 3.2 PostgreSQL
```sql
-- 일반 트랜잭션
BEGIN;
SELECT * FROM users WHERE id = 1;
-- MVCC 스냅샷 생성
-- 쓰기 락 준비
-- 트랜잭션 ID 할당
COMMIT;

-- 읽기 전용 트랜잭션
BEGIN READ ONLY;
SELECT * FROM users WHERE id = 1;
-- 읽기 전용 스냅샷만 생성
-- 락 없음
-- 최소한의 트랜잭션 오버헤드
COMMIT;
```

### 3.3 Oracle
```sql
-- 일반 트랜잭션
SET TRANSACTION;
SELECT * FROM users WHERE id = 1;
-- 트랜잭션 ID 할당
-- Undo 로그 준비
COMMIT;

-- 읽기 전용 트랜잭션
SET TRANSACTION READ ONLY;
SELECT * FROM users WHERE id = 1;
-- 읽기 전용 모드
-- 최소한의 리소스 사용
COMMIT;
```

## 4. 주의사항과 제한사항

### 4.1 readOnly 제한사항
```java
@Service
@Transactional(readOnly = true)
public class LimitedService {
    
    public void problematicMethod() {
        // ❌ 다음 작업들은 예외 발생
        userRepository.save(new User());           // INSERT
        userRepository.deleteById(1L);            // DELETE
        userRepository.updateName(1L, "NewName"); // UPDATE
        
        // ❌ 엔티티 상태 변경
        User user = userRepository.findById(1L).orElse(null);
        user.setName("NewName"); // 예외 발생
    }
}
```

### 4.2 전파 속성 주의사항
```java
@Service
public class PropagationService {
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodWithRequiresNew() {
        // 주의: 기존 트랜잭션과 완전히 분리됨
        // 롤백이 서로 영향을 주지 않음
    }
    
    @Transactional(propagation = Propagation.NESTED)
    public void methodWithNested() {
        // 주의: 모든 데이터베이스가 지원하지 않음
        // MySQL의 경우 SAVEPOINT 사용
    }
}
```

## 참고 자료
- [@Transactional 성능 최적화](./@Transactional_성능최적화.md)
- [@Transactional 실무 적용](./@Transactional_실무적용.md)
- [@Transactional 고급 주제](./@Transactional_고급주제.md)
- Spring Framework 공식 문서 
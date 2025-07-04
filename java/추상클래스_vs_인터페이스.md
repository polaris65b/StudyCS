# 추상 클래스(Abstract Class) vs 인터페이스(Interface)

## 1. 추상 클래스
- **정의**: 공통적인 속성과 기능을 정의, 일부 메서드는 구현 가능
- **특징**: 단일 상속, 필드/생성자/구현 메서드 가질 수 있음

## 2. 인터페이스
- **정의**: 구현 객체가 따라야 할 명세(기능의 집합)
- **특징**: 다중 구현 가능, 상수/추상 메서드만 가짐(Java 8 이후 default/static 메서드 가능)

## 3. 차이점 및 사용 시점
| 구분         | 추상 클래스           | 인터페이스                |
|--------------|----------------------|---------------------------|
| 상속/구현    | extends(단일 상속)   | implements(다중 구현)     |
| 목적         | 공통 속성/기능 제공  | 규약(명세) 제공           |
| 상태/필드    | 가질 수 있음         | 상수만 가능               |
| 생성자       | 가질 수 있음         | 없음                      |
| 메서드 구현  | 일부 구현 가능       | (Java 8+) default/static  |

- **추상 클래스**: 공통 상태/기능이 필요할 때, 계층 구조 설계
- **인터페이스**: 역할/기능의 명세, 다양한 구현체 필요할 때

## 4. 예시 코드
```java
abstract class Animal { abstract void sound(); }
interface Flyable { void fly(); }
class Bird extends Animal implements Flyable {
    void sound() { System.out.println("짹짹"); }
    public void fly() { System.out.println("난다"); }
}
```

## 5. 면접 포인트
- "추상 클래스와 인터페이스의 차이점은?"
- "실무에서 각각 언제 사용했는가?" 
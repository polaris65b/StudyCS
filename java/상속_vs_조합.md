# 상속(Inheritance) vs 조합(Composition)

## 1. 상속(Inheritance)
- **정의**: 기존 클래스(부모)의 속성과 기능을 새로운 클래스(자식)가 물려받는 것
- **장점**: 코드 재사용, 계층적 구조, 다형성의 기반 제공
- **단점**: 강한 결합, 유연성 저하, 부모 변경 시 자식에 영향

## 2. 조합(Composition)
- **정의**: 객체가 다른 객체를 자신의 필드로 포함(참조)하여 기능을 사용하는 것
- **장점**: 느슨한 결합, 유연성, 런타임에 객체 교체 가능
- **단점**: 설계가 복잡해질 수 있음

## 3. 언제 어떤 것을 선택할까?
- **상속**: is-a 관계(진짜 하위 타입), 공통 기능이 명확할 때
- **조합**: has-a 관계, 유연한 구조, 변경/확장에 강한 설계가 필요할 때(실무에서는 조합을 더 선호)

## 4. 예시 코드
```java
// 상속
class Animal { void eat() {} }
class Dog extends Animal { void bark() {} }

// 조합
class Engine { void start() {} }
class Car {
    private Engine engine = new Engine();
    void drive() { engine.start(); }
}
```

## 5. 면접 포인트
- "상속보다는 조합을 우선 고려하라"(Effective Java)
- 상속 남용의 문제점, 조합의 장점 설명

---

## 6. 실무에서의 조합 활용 예시: 전략 패턴(Strategy Pattern)
```java
interface PaymentStrategy { void pay(int amount); }
class CardPayment implements PaymentStrategy {
    public void pay(int amount) { System.out.println("카드 결제: " + amount); }
}
class CashPayment implements PaymentStrategy {
    public void pay(int amount) { System.out.println("현금 결제: " + amount); }
}
class Order {
    private PaymentStrategy paymentStrategy;
    public Order(PaymentStrategy paymentStrategy) { this.paymentStrategy = paymentStrategy; }
    public void process(int amount) { paymentStrategy.pay(amount); }
}
// 사용
Order order = new Order(new CardPayment());
order.process(10000);
```

## 7. 상속 남용의 실제 문제점
- 부모 클래스의 변경이 자식 클래스에 예기치 않은 영향을 미침 (fragile base class problem)
- 불필요한 메서드 상속, 계층 구조가 복잡해짐
- 테스트 및 유지보수 어려움

## 8. 참고 자료
- [Effective Java, Item 18: 상속보다는 조합을 사용하라](https://book.naver.com/bookdb/book_detail.naver?bid=14767498)
- [Oracle Java Tutorials - Composition](https://docs.oracle.com/javase/tutorial/java/IandI/composition.html)
- GoF 디자인 패턴 
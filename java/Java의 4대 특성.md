# Java의 객체지향 4대 특성

## 1. 캡슐화 (Encapsulation)
- **정의**: 데이터(필드)와 이를 처리하는 메서드를 하나의 단위(클래스)로 묶고, 외부에서 직접 접근하지 못하도록 제한하는 것.
- **특징**: 접근 제어자(private, protected, public 등)를 사용하여 데이터 보호 및 정보 은닉.
- **장점**: 데이터 무결성 보장, 유지보수성 향상, 모듈화 용이
- **실무 예시**:  
  - DTO, Entity 등에서 필드를 private으로 선언하고, public getter/setter로 접근 제어  
  - Lombok의 @Getter, @Setter 사용  
  - 불변 객체(Immutable Object) 설계 시 setter를 제공하지 않음
- **면접 질문**:  
  - "캡슐화와 정보 은닉의 차이는?"  
  - "Getter/Setter를 무조건 만들어야 할까?"  
  - "캡슐화가 잘못된 설계의 예시는?"
- **예시 코드**:
  ```java
  public class Person {
      private String name; // 외부 직접 접근 불가
      public String getName() { return name; }
      public void setName(String name) { this.name = name; }
  }
  ```

## 2. 상속 (Inheritance)
- **정의**: 기존 클래스(부모, 슈퍼클래스)의 속성과 기능을 새로운 클래스(자식, 서브클래스)가 물려받는 것.
- **특징**: 코드 재사용성 증가, 계층적 구조 형성, 다형성의 기반 제공
- **장점**: 공통 기능의 재사용, 유지보수성 향상
- **단점/주의점**:  
  - 강한 결합(부모 변경 시 자식에 영향)  
  - 다중 상속 불가(Java는 클래스 다중 상속을 지원하지 않음, 인터페이스로 가능)
  - "상속보다는 조합(Composition)을 우선 고려" (Effective Java)
- **실무 예시**:  
  - Spring의 Controller, Service 계층 구조  
  - JPA Entity 상속 구조
- **면접 질문**:  
  - "상속과 조합의 차이점은?"  
  - "상속을 남용하면 어떤 문제가 생길 수 있나?"
- **예시 코드**:
  ```java
  class Animal {
      void eat() { System.out.println("먹는다"); }
  }
  class Dog extends Animal {
      void bark() { System.out.println("짖는다"); }
  }
  // Dog는 eat()와 bark() 모두 사용 가능
  ```

## 3. 다형성 (Polymorphism)
- **정의**: 하나의 인터페이스(메서드, 객체 등)가 여러 형태로 동작할 수 있는 성질.
- **특징**: 오버라이딩(동적 다형성), 오버로딩(정적 다형성), 부모 타입 참조 변수로 자식 객체 참조 가능
- **장점**: 유연한 코드 설계, 확장성, 유지보수성 향상
- **실무 예시**:  
  - List, Set, Map 등 컬렉션 프레임워크에서 인터페이스 타입으로 구현체 교체  
  - 전략 패턴, 템플릿 메서드 패턴 등 디자인 패턴에서 활용
- **면접 질문**:  
  - "다형성이란 무엇인가?"  
  - "오버라이딩과 오버로딩의 차이점은?"  
  - "다형성이 실무에서 어떻게 활용되는가?"
- **예시 코드**:
  ```java
  class Animal {
      void sound() { System.out.println("동물 소리"); }
  }
  class Cat extends Animal {
      void sound() { System.out.println("야옹"); }
  }
  Animal a = new Cat();
  a.sound(); // "야옹" 출력 (동적 바인딩)
  ```

## 4. 추상화 (Abstraction)
- **정의**: 불필요한 세부 사항은 숨기고, 필요한 부분만을 추출하여 모델링하는 것.
- **특징**: 추상 클래스(abstract class), 인터페이스(interface)로 구현
- **장점**: 복잡성 감소, 설계의 유연성, 확장성
- **실무 예시**:  
  - JDBC, JPA 등에서 인터페이스를 통한 구현체 교체  
  - 서비스 계층에서 인터페이스로 비즈니스 로직 추상화
- **면접 질문**:  
  - "추상 클래스와 인터페이스의 차이점은?"  
  - "추상화가 왜 중요한가?"  
  - "실제 프로젝트에서 추상화를 어떻게 적용했는가?"
- **예시 코드**:
  ```java
  abstract class Shape {
      abstract void draw(); // 추상 메서드
  }
  class Circle extends Shape {
      void draw() { System.out.println("원을 그린다"); }
  }
  ```

## [비교 표]

| 특성     | 주요 키워드         | 장점/의의                | 실무 예시/면접 포인트                |
|----------|--------------------|--------------------------|--------------------------------------|
| 캡슐화   | 정보 은닉, 접근제어 | 데이터 보호, 모듈화       | DTO, Entity, Getter/Setter           |
| 상속     | 코드 재사용, 계층   | 유지보수, 다형성 기반     | Controller 계층, JPA Entity          |
| 다형성   | 오버라이딩, 인터페이스 | 유연성, 확장성           | 컬렉션, 디자인 패턴, 전략 패턴        |
| 추상화   | 인터페이스, 추상클래스 | 복잡성 감소, 설계 유연성 | JPA, JDBC, 서비스 계층 인터페이스    |
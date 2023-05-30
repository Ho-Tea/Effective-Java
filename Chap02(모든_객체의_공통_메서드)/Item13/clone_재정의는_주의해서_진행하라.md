# clone 재정의는 주의해서 진행하라
- `Cloneable`은 복제해도 되는 클래스임을 명시하는 용도의 믹스인 인터페이스이다
  - 하지만, `clone`메서드가 선언된 곳이 `Cloneable`이 아닌 `Object`이다
- `clone`메서드를 잘 동작하게끔 해주는 **구현방법과 언제 그렇게 해야하는지**를 알아보자
  - **(얕은복사와 깊은복사를 주의하자)**

## Cloneable 인터페이스
  - `Object`의 `protected` 메서드인 `clone`의 동작 방식을 결정한다
  - `Cloneable`을 구현한 클래스의 인스턴스에서 `clone`을 호출하면 그 객체의 필드들을 하나하나 복사한 객체를 반환한다
    - 그렇지 않다면(구현하지 않았다면), `CloneNotSupportedException`을 던진다
  - **해당 인터페이스를 구현했다는 것은 복제가 가능하다는 것을 의미한다.**


## clone()
  - ### 필드들만 복사(얕은복사)해서 새로운 객체(깊은복사)를 만든다
    ``` java
    public class Cloneable {

    static class Maruta {

        int age = 0;

        void setAge(int age){
            this.age = age;
        }

    }

    static class Example implements java.lang.Cloneable {

        Example(){
            ex2 = new Maruta();
        }
        Maruta ex2;
        @Override
        protected Object clone() throws CloneNotSupportedException {
            return super.clone();
        }
    }

    @Test
    void 객체_내부의_레퍼런스_객체는_얕은_복사가_이루어진다() throws CloneNotSupportedException {
        Example ex = new Example();
        Example result = (Example)ex.clone();
        ex.ex2.setAge(10);

        Assertions.assertEquals(result.ex2.age, ex.ex2.age);
      }

    }
    ```
  - 동작
    - `Cloneable`을 구현했는지 확인 → 안되었으면 `CloneNotSupportedException` 발생
    - 객체를 복제한 새 객체를 반환(구현은 JVM마다 다를 수 있다)
  - 규약
    - `x.clone() != x` : 새로운 객체여야 한다.
    - `x.clone().getClass() == x.getClass()` : 같은 타입(클래스)여야 한다.
    - `x.clone().equals(x)` : 필드가 모두 같아야 한다.

## clone() 상황에 따른 구현 방법들
  - **불변 객체(객체의 필드가 모두 불변의 경우)**
    ```java
    public class ChessPiece implements Cloneable {
    
    private final Role role; //enum
    private final Color color; //enum
    
    @Override
    public ChessPiece clone() throws CloneNotSupportedException {
        return (ChessPiece) super.clone();
    }
    // ...
    }
    ```
    - ### 공변 반환 타입 (jdk 1.5 ~)
      ``` java
      class Parent {
        protected Parent createNewOne() {
          return new Parent();
        }
      }

      class Child extends Parent {
      // 부모 클래스로부터 재정의하였으나 반환형을 자식 클래스로 변경할 수 있다.
        @Override public Child createNewOne() {
          return new Child();
        }
      }
      ```
      - 부모 클래스의 반환 타입(`Object`)을 자식 클래스의 타입(`ChessPiece`)으로 변경한다.
      - `clone()`을 완벽히 재정의한 클래스를 상속했다면, 형변환에서 절대 실패하지 않을 것이다.(당연)

  - **가변 객체 (얕은복사와 깊은복사를 주의)**
    ``` java
    public class Board implements Cloneable{
      private final Map<Square, Piece> board;
      private Side turn;
    
      // ...

      // 재정의한 메서드는 CloneNotSupportedException을 없애야 한다
      // 검사예외를 던지지 않아야 이 메서드를 사용하기 편하다
      @Override
      public Board clone() {  
        return (Board) super.clone();
      }
    }
    ```
    - 이렇게 `clone()`을 재정의 하게 되면
    - `Board board`와 `board.clone()` 은 다른 객체지만, 필드의 참조주소는 같다.
    - `board.clone()`의 상태의 변화가 `board`의 상태를 변화하게 된다.

  - **고수준의 메서드 사용 (제일 겉면에 있는 메서드)등등...**

  -------------

 ## 핵심정리
  - `Cloneable`을 구현하는 모든 클래스는 `clone`을 재정의해야 한다
  - 접근제한자는 `public`으로, 반환 타입은 `클래스 자신`으로 변경한다
  - 가장 먼저 `super.clone`을 호출 한 후 필요한 필드를 적절히 수정
  - 너무 복잡하다...
  - ### Cloneable을 구현하지 않은 상황에서는 복사생성자와 복사 팩터리라는 더 나은 객체 복사 방식을 제공할 수 있다(이걸 쓰자)
  
## 복사 생성자
  - 자신과 같은 클래스의 인스턴스를 인수로 받는 생성자
    ``` java
    public Yum(Yum yum){...};
    ```
## 복사 팩터리
  - 복사생성자를 모방한 정적 팩터리
    ``` java
    public static Yum newInstance(Yum yum){...};
    ```

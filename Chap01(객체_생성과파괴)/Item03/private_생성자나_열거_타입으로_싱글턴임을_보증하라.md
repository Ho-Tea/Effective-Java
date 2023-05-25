## private 생성자나 열거타입으로 싱글턴임을 보증하라
- 싱글턴이란 인스턴스를 오직 하나만 생성할 수 있는 클래스를 의미한다
- 클래스를 싱글턴으로 만들면 이를 사용하는 클라이언트를 테스트하기가 어려워질 수 있다
  <br> 싱글턴 인스턴스를 가짜(`mock`)구현으로 대체할 수 없기 때문이다


-------------
- ### public static final 필드 방식의 싱글턴
  - 장점:
     1. 해당 클래스가 싱글턴임이 API에 명백히 드러난다
     2. 간결하다 
``` java
public class Elvis {
  public static final Elvis INSTANCE = new Elvis();
  private Elvis() {...}

  public void leaveTheBuilding() {...}
}
// 싱글턴이 보장되지만 하나의 예외가 존재
// 권한이 있는 클라이언트는 리플렉션 API인 AccessibleObject.setAccessible을 사용해 private 생성자를 호출할 수 있다
// 이러한 공격을 방어하려면 생성자를 수정하여 두번째 객체가 생성되려 할 때 예외를 던지게하면 된다
```



- ### 정적 팩터리 방식의 싱글턴
  - 장점:
    1. API를 바꾸지 않고도 싱글턴이 아니게끔 변경할 수 있다
    2. 정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있다
    3. 정적팩터리의 메서드참조를 공급자(`supplier`)로 사용할 수 있다는 점이다<br> `Elvis::getInstance`를 `Supplier<Elvis>`로 사용하는 방식이다 
        - Supplier를 쓰는 이유
          - **T의 하위 타입 인스턴스를 반환할 수 있다**
            ``` java
            public class AdminUser extends User { }

            ...

            //given
            Supplier<User> supplier = ( ) -> AdminUser.of("관리자 아이디", "관리자 이름")

            //when
            User user = supplier.get();

            //then
            assertTrue(user instanceOf AdminUser);
            assertEquals(user.getId(), "관리자 아이디");
            assertEquals(user.getName(), "관리자 이름");
            ```
          - **Lazy Evaluation**(**=불필요한 연산을 피한다**)을 적용할 수 있다
            ``` java
            private static void negativeTest(int number, Supplier<String> expressionSupplier){
              if(number > 0>){
                System.out.println("hi" + expressionSupplier.get());
              }else{
                System.out.println("nope");
              }
              
            }

            private static String expressionProcessing() {
              try{
                TimeUnit.SECONDS.sleep(3);
              }catch(InterruptedException e) {
                e.printStackTrace();
              }
              return "gold-egg";
            }
            public static void main(String[] args){
              negativeTest(1, () -> expressionProcessing());
              negativeTest(-1, () -> expressionProcessing()); //불필요한 연산을 하지 않는다
            }
            ```


``` java
public class Elvis {
  private static final Elvis INSTANCE = new Elvis();
  private Elvis() {...}
  public static Elvis getInstance() {return INSTANCE; }

  public void leaveTheBuilding() {...}
}
// 앞서 말한 리플렉션 예외는 똑같이 적용된다
```

- 둘 중 하나의 방식으로 만든 싱글턴 클래스를 직렬화하려면 단순히 `Serializable`을 구현한다고 선언하는 것만으로는 부족하다<br> 직렬화된 인스턴스를 역직렬화할 때마다 새로운 인스턴스가 만들어 지기 때문에 `readResolve` 메서드를 제공해야 한다

--------------

- ## 핵심정리
- ### Enum 사용(제일 좋다)
  - 더 간결하다
  - 직렬화가 가능하다
  - **리플렉션 공격을 완벽히 막아준다**
  - 상속이 안된다는 단점 존재.
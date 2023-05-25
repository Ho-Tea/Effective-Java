## finalizer와 cleaner 사용을 피하라
  - 자바는 두가지 **객체 소멸자**를 제공한다
    - `finalizer`
      - 예측할수 없고, 상황에 따라 위험하다
      - 나름의 쓰임새가 있지만 쓰지말아야하는 API
    - `cleaner`
      - `finalizer`보다는 덜 위험하지만, 예측할 수 없다
      - 일반적으로 불필요하다
    
  - ### 즉시 수행된다는 보장이 없다
    - 실행되기까지 얼마나 걸릴지 알 수 없다
    - **즉, finalizer와 cleaner로는 제때 실행되어야 하는 작업은 절대 할 수 없다**
    - 클래스에 finalizer를 달아두면 그 인스턴스의 자원 회수가 제멋대로 지연될 수 있다

  - ### 수행시점 뿐 아니라 수행 여부조차 보장하지 않는다
    - 상태를 영구적으로 수정하는 작업에서는 절대 `finalizer`나 `cleaner`에 의존해서는 안 된다.
    - `finalizer` 동작중 발생한 예외는 무시되며, 처리할 작업이 남았더라도 그 순간 종료된다<br> 잡지 못한 예외때문에 해당 객체는 자칫 마무리가 덜 된 상태로 남을 수 있다

  - ### 심각한 성능 문제도 동반한다
    - 가비지 컬렉터가 수거하는데 12ns, finalizer를 사용하면 550ns 경과
  
  - ### finalizer를 사용한 클래스는 finalizer 공격에 노출되어 심각한 보안 문제를 일으킬 수 있다
    - **생성이나 직렬화 과정에서 에외가 발생하면 생성되다 만 객체에서 악의적인 하위 클래스의 `finalizer`가 수행될 수 있게 된다.**
    ``` java
    public class Account {
      private String name;

      public Account(String name){
        this.name = name;
        if(this.name.equals("푸틴")){
          throw new IllegalArgumentException("푸틴은 안돼~~")
        }
      }
      public void transfer(int amount , String to){
        System.out.println("transfer %d from %s to %s",amount, this.name, to);
      }
    }

    //Test
    void 일반사람(){
      Account account = new Account("일반사람");
      account.transfer(100,"juho");
    } // 통과

    void 푸틴(){
      Account account = new Account("푸틴");
      account.transfer(100,"juho")
    } // 막힌다 하지만 보낼수 있다
    // finalizer 공격으로!
    ```
    ``` java

    // 이게 바로 finalizer 공격
    public class BrokenAccount extends Account {
      public BrokenAccount(String name){
        super(name);
      }
      @Override
      protected void finalize() throws Throwable{
        this.trnasfer(1000000,"juho");
      }
    }


    //Test
    void 일반사람(){
      Account account = new Account("일반사람");
      account.transfer(100,"juho");
    } // 통과

    void 푸틴(){
      Account account = null;
      try{
        account = new BrokenAccount("푸틴");
      } catch(Exception e){
        System.out.println("푸틴은 안되는데??");
      }
      System.gc();  //가비지 컬렉터를 실행
      Thread.sleep(3000L); // 가비지 컬렉터가 다 실행되기까지 살짝 기다리는 것
    } 
    ```

-------------
  - ## AutoCloseable을 구현하자!
    - 클라이언트에서 인스턴스를 다 쓰고 나면 close 메서드를 호출하면 된다
    - **그러면 cleaner와 finalizer는 대체 어디에 쓰지?**
      - 자원의 소유자가 `close`메서드를 호출하지 않는것에 대비한 **안전망 역할**
        - ex) `FileInputStream`, `FileOutputStream` 등
      - **네이티브 피어**와 연결된 객체
        > 네이티브 피어란 일반 자바 객체가 네이티브 메서드를 통해 기능을 위임한 네이티브 객체를 말한다
        - 네이티브 피어는 자바 객체가 아니니 가비지 컬렉터는 그 존재를 알지 못한다
        - 그 결과, 자바 피어를 회수할 때 네이티브 객체까지 회수하지 못한다
    ``` java
    //cleaner를 안전망으로 활용하는 AutoCloseable클래스

    //방 자원을 수거하기 전에 반드시 청소해야한다고 가정

    public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // 청소가 필요한 자원, 절대 Room을 참조해서는 안된다!
    private static class State implements Runnable { 
        int numJunkPiles; //방안의 쓰레기 수

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        @Override
        public void run() {  
          // **colse가 호출되거나, GC가 Room을 수거해갈 때 run() 호출**
            System.out.println("Room Clean");
            numJunkPiles = 0;
        }
    }
    // 방의상태 cleanable과 공유한다
    private final State state;

    // cleanable객체 수거 대상이 되면 방을 청소한다
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);  
        // State는 Runnable을 구현하고 그안의 run 메서드는 cleanable에 의해 딱 한번만 호출될 것이다

    }

    @Override
    public void close() {
        cleanable.clean();
    }
    }
    ```
    - `run`메서드가 호출되는 상황
      - `Room`의 `close`메서드를 호출 할 때
        - `close`메서드에서 `Cleanable`의 `clean`을 호출하면 이 메서드 안에서 `run`을 호출한다
      - 가비지 컬렉터가 `Room`을 회수할 때까지 클라이언트가 `close`를 호출하지 않는다면, `cleaner`가 `State`의 `run`메서드를 호출시킬 것이다
    - **`State`인스턴스는 절대로. Room인스턴스를 참조해서는 안된다**
      - `Room`인스턴스를 참조할 경우 **순환참조**가 생겨 가비지 컬렉터가 `Room`인스턴스를 회수해갈 기회가 오지 앟는다
      - `State`가 **정적 중첩 클래스인 이유**가 이것이다
      - **정적이 아닌 중첩 클래스**인 경우에는 자동으로 바깥객체의 참조를 갖게 되기 때문에
        - 람다 역시 바깥객체의 참조를 갖기 쉬우니 조심하자
    - 위 코드는 단지 `cleaner`를 안전망으로 쓴 경우이고 (`close`메서드를 실행시키지 않는 다는 가정하에) **실행될지 안될지 예측할 수 없다**
      ``` java
      public static void main(final String[] args) {
        new Room(8);
        System.out.println("방 쓰레기 생성~~");
      }
      
      ```
    - `try-with-resources`블록으로 감쌌다면 자동청소는 전혀 필요하지 않다
      ``` java
      public static void main(final String[] args) {
       try (Room myRoom = new Room(8)) {
          System.out.println("방 쓰레기 생성~~");
        }
      }
      ```

-------------

- ## 핵심정리
  - **cleaner, finalizer를 사용하지 말자**
  - **try-with-resource를 사용하자 + AutoCloseable을 구현하자**
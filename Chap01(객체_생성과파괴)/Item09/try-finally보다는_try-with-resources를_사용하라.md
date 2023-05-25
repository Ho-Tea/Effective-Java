## try-finally보다는 try-with-resources를 사용하라

- ### try-finally를 쓰지 말아야 하는 이유
  - 자바 라이브러리 에서는 `close`메서드를 호출해 직접 닫아주어야 하는 자원이 많다
    - `InutStream`, `OutputStream`, `java.sql.connection`등이 좋은 예
  - 전통적으로 자원이 제대로 닫힘을 보장하는 수단으로 `try-finally`가 쓰였다
  ``` java
  static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try{
      return br.readLine();
    }finally{
      br.close();
    }
  }
  
  // 그렇다면 만약 close를 여러 번 호출해야 하는 상황이 오면 어떻게 될까?
  //자원이 둘 이상이되면 지저분해 진다
  public static void inputAndWriteString() throws IOException {
    BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    try {
        BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(System.out));
        try {
             String line = br.readLine();
            bw.write(line);
        } finally {
            bw.close();
        }
    } finally {
        br.close();
    }
  }

  ```
  - 사실 가장 큰 문제는 예외를 추적하기 힘들어진다는 것이다
  - `firstLineOfFile` 메서드의 try 블록을 실행하던 도중 기기에 문제가 생긴다면 <br> `readLine`이 정상적으로 실행되지 못하고 예외를 던지게 되고, <br>같은 이유로 `finally` 블록의 `close` 메서드도 예외를 던지게 된다.
  - 만약 이 예외들을 `catch`해서 상위 메서드에서 예외 정보를 체크해본다면,<br> `finally` 블록에서 터진 예외가 `try` 블록에서 생긴 
  **예외를 집어 삼켜서** `finally` 블록의 예외만 체크하게 된다.
  - `try` 블록에서 터진 예외로 인해 `finally` 블록에서 예외가 발생했음에도 불구하고 최초 원인인 예외를 체크하지 못하게 되는 것이다. 
  - 물론 적절한 코드를 통해 최초 원인 예외를 체크할 수는 있지만, 코드가 너무 더러워지기 때문에 **추천하는 방법은 아니다.**
  ``` java
  //Test
  public class Application {

    public static void main(String[] args) {
        try {
            check();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void check() {
        try {
            throw new IllegalArgumentException();
        } finally {
            throw new NullPointerException();
            // 예외는 관련 없이 임의로 선택했다.
        }
    }
  }

  //Result
  java.lang.NullPointerException 
    at Application.check(Application.java:14)  
    at Application.main(Application.java:4
  ```
  - 발생하는 예외를 main의 `catch` 블록에서 체크해서 `printStackTrace`를 호출했는데, 14번 라인, <br>즉, `finally` 블록에서 던져진 `NullPointerException`을 `catch` 하고 `try` 블록의 `IllegalArgumentException`은 무시된 것을 볼 수 있다.
  - **정리하자면 try-finally는 가독성을 해칠 가능성이 높으며, 예외 처리 로직을 작성하기에 결점이 존재한다.**

- ### try-with-resources를 써야하는 이유
  - 이 구조를 사용하려면 해당 자원이 `AutoCloseable`인터페이스를 구현해야 한다
    > 자바 라이브러리와 서드파티 라이브러리들의 수많은 클래스와 인터페이스가 이미 AutoCloseable을 구현하거나 확장해 두었다

  - **닫아야 하는 자원을 뜻하는 클래스를 작성한다면 AutoCloseable을 반드시 구현하자**

  ``` java
  public static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
  }
  ```
  - `firstLineOfFile` 메서드의 `readLine`과 `close` 모두에서 예외가 발생하는 경우,<br> `close`(물론 코드 상으로는 보이지 않지만) 호출 시 발생하는 예외는 숨겨지고 `readLine`의 예외가 기록된다.
  - 이렇게 숨겨진 예외는 무시되는 것이 아니라, `suppressed` 상태가 되어 stackTrace 시 숨겨졌다는 메시지로 출력된다. 
    > suppressed 상태가 된 예외는 자바 7부터 도입된 getSuppressed 메서드를 통해 가져와서 사용할 수 있다
  
  ``` java
  //Test
  public class Application {

    public static void main(String[] args) {
        try {
            check();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void check() throws Exception {
        try (IllegalArgumentExceptionThrower thrower = new IllegalArgumentExceptionThrower()) { //AutoCloseable을 구현하지 않은 객체는 컴파일 오류가 난다
            throw new NullPointerException();
        }
    }

    static class IllegalArgumentExceptionThrower implements AutoCloseable {
        @Override
        public void close() throws Exception {
            throw new IllegalArgumentException();
        }
    }
  }

  //Result

  java.lang.NullPointerException  
    at Application.check(Application.java:17)
    at Application.main(Application.java:9)
    Suppressed: java.lang.IllegalArgumentException
        at Application$IllegalArgumentExceptionThrower.close(Application.java:24)
        at Application.check(Application.java:16)
        ... 1 more
  ```
  - 직접 throw 한 `NullPointerException이` `catch`되어 출력되며,<br> 이 때 close 메서드에서 던지는 `IllegalArgumentException`은 `Suppressed`: 태그 뒤로 출력되는 것을 볼 수 있다.
  

- ### try-with-resources + catch
  ``` java
  public static String firstLineOfFile(String path) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return "IOException 발생";
    }
  }
  ```
  - catch문도 같이 사용 가능하다
-------------

- ## 핵심정리
  - **꼭 회수해야 하는 자원을 다룰 때는 `try-finally`말고, `try-with-resources`를 사용하자**
  - **디버깅하기 더 쉽게 에러를 추적할 수 있다**
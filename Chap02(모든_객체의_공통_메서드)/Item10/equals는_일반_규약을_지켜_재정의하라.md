# equals는 일반 규칙을 지켜 재정의하라
> **equals 메서드는 재정의하기 쉬워보이지만 곳곳에 함정이 도사리고 있어서 자칫하면 끔찍한 결과를 초래한다**


- ## **재정의**하지 말아야 할 상황들
  - **각 인스턴스가 본질적으로 고유하다**
    - `Thread`가 좋은 예(`Object`의 `equals`메서드는 이러한 클래스에 딱 맞게 구현되었다)
  - **인스턴스의 '논리적 동치성'을 검사할 일이 없다**
  - **상위 클래스에서 재정의한 equals가 하위클래스에서도 딱 들어맞는다**
  - **클래스가 private이거나 package-private이고 equlas메서드를 호출할 일이 없다**



- ## **재정의** 해야할 상황들
  - **논리적 동치성을 확인해야하는데 상위 클래스의 equlas가 논리적 동치성을 비교하도록 재정의 되지 않았을 때**
    - 주로 **값 클래스**들이 여기에 해당
  - **값 클래스**라고 해도 `Enum`과 같은 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제클래스라면 재정의하지 않아도 된다

---------------

- ## equals 메서드를 재정의할때는 반드시 일반규약을 따라야 한다
  - ### 반사성 : `null`이 아닌 모든 참조 값 x에 대해, `x.equals(x)는 true다`
    - 객체가 자기 자신과 같아야 한다는 뜻
  - ### 대칭성 : `null`이 아닌 모든 참조 값 x, y에 대해 `x.equals(y)` 가 `true`면 `y.equals(x)`도 `true`이다
    ``` java
    public final class CaseInsensitiveString {

    private final String str;
    
    public CaseInsensitiveString(String str) {
        this.str = Objects.requireNonNull(str);
    }
    
    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString) {
            return str.equalsIgnoreCase(((CaseInsensitiveString) o).str);
        }
    
        if (o instanceof String) { // 한 방향으로만 작동한다.
            return str.equalsIgnoreCase((String) o);
        }
        return false;
      }
    }

    void symmetryTest() {
      CaseInsensitiveString caseInsensitiveString = new CaseInsensitiveString("Test");
      String test = "test";
      System.out.println(caseInsensitiveString.equals(test)); // true
      System.out.println(test.equals(caseInsensitiveString)); // false
    }
    ```
  - ### 추이성 : `null`이 아닌 모든 참조 값 x, y, z에 대해 `x.equals(y)`가 `true`이고, `y.equals(z)`도 `true`면, `x.equlas(z)`도 `true`다
    ``` java
    // Point
    public class Point {

    private final int x;
    private final int y;
    
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) {
            return false;
        }
        Point p = (Point) o;
        return this.x == p.x && this.y == p.y;
      }
    }


    // ColorPoint
    public class ColorPoint extends Point {

    private final Color color;

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) {
            return false;
        }

        // o가 일반 Point이면 색상을 무시햐고 x,y 정보만 비교한다.
        if (!(o instanceof ColorPoint)) {
            return o.equals(this);
        }

        // o가 ColorPoint이면 색상까지 비교한다.
        return super.equals(o) && this.color == ((ColorPoint) o).color;
      }
    }

    // Result
    void transitivityTest() {
    ColorPoint a = new ColorPoint(2, 3, Color.RED);
    Point b = new Point(2, 3);
    ColorPoint c = new ColorPoint(2, 3, Color.BLUE);

    System.out.println(a.equals(b)); // true
    System.out.println(b.equals(c)); // true
    System.out.println(a.equals(c)); // false
    }

    ```
    - ### 무한재귀에 빠질 수도 있다
      ``` java
      public class SmellPoint extends Point {
        private final Smell smell;

        @Override
        public boolean equals(Object o) {
          if (!(o instanceof Point)) {
            return false;
        }

          // o가 일반 Point이면 색상을 무시햐고 x,y 정보만 비교한다.
          if (!(o instanceof SmellPoint)) {
            return o.equals(this);
          }

          // o가 ColorPoint이면 색상까지 비교한다.
          return super.equals(o) && this.smell == ((SmellPoint) o).smell;
        }
      }

      void infinityTest() {
        Point cp = new ColorPoint(2, 3, Color.RED);
        Point sp = new SmellPoint(2, 3, Smell.SWEET);

        System.out.println(cp.equals(sp));
      }


      -> Result : StackOverFlow
      ```

    - ### (해결방안..?) 추이성을 지키기 위해서 Point의 equals를 각 클래스들을 `getClass`를 통해서 같은 구체 클래스일 경우에만 비교하도록 하면 어떨까?
      - -> 동작은 하지만, **리스코프 치환원칙을 위배**

        > 이는 모든 객체 지향 언어의 동치관계에서 나타나는 근본적인 문제이며, <br> 객체 지향적 추상화의 이점을 포기하지 않는 한 불가능하다.

    - ### (해결방안..!) 상속보다는 구성을 이용하라
      ``` java
      public class ColorPoint2 {

        private Point point;
        private Color color;

        public ColorPoint2(int x, int y, Color color) {
          this.point = new Point(x, y);
          this.color = Objects.requireNonNull(color);
        }

        public Point asPoint() {
          return this.point;
        }

        @Override
        public boolean equals(Object o) {
          if (!(o instanceof ColorPoint)) {
            return false;
          }
          ColorPoint cp = (ColorPoint) o;
          return this.point.equals(cp) && this.color.equals(cp.color);
        }
      }
      ```

      
  - ### 일관성 : `null`이 아닌 모든 참조 값 x, y에 대해, `x.equals(y)`를 반복해서 호출하면 항상 `true`를 반환하거나 항상 `false`를 반환한다
    ``` java
    @Test
    void consistencyTest() throws MalformedURLException {
      URL url1 = new URL("www.xxx.com");
      URL url2 = new URL("www.xxx.com");

      System.out.println(url1.equals(url2)); // 항상 같지 않다.
    }
    ```
    - java.net.URL 클래스는 URL과 매핑된 `host의 IP주소`를 이용해 비교하기 때문에 같은 도메인 주소라도<br> 나오는 IP정보가 달라질 수 있기 때문에 반복적으로 호출할 경우 결과가 달라질 수 있다.

    - 따라서 이런 문제를 피하려면 **equals는 항시 메모리에 존재하는 객체만을 사용한 결정적 계산을 수행해야 한다.**
  - ### null-아님 : `null`이 아닌 모든 참조 값 x에 대해, `x.equals(null)`은 `false`다
    - 명시적으로 `null`검사를 할 필요는 없고 `instanceof`를 이용하자
    ``` java
    if (!(o instanceof Point)) {
        return false;
    }
    ```


- ## 좋은 equals 메서드 재정의 방법
  ``` java
  @Override
  public boolean equals(final Object o) {
    // 1. == 연산자를 사용해 입력이 자기 자신의 참조인지 확인한다.
    if (this == o) {
        return true;
    }

    // 2. instanceof 연산자로 입력이 올바른 타입인지 확인한다.
    if (!(o instanceof Point)) {
        return false;
    }

    // 3. 입력을 올바른 타입으로 형변환 한다.
    final Piece piece = (Piece) o;

    // 4. 입력 개체와 자기 자신의 대응되는 핵심 필드들이 모두 일치하는지 하나씩 검사한다.
    
    // float와 double을 제외한 기본 타입 필드는 ==를 사용한다.
    return this.x == p.x && this.y == p.y;
    
    // 필드가 참조 차입이라면 equals를 사용한다.
    return str1.equals(str2);
    
    // null 값을 정상 값으로 취급한다면 Objects.equals로 NullPointException을 예방하자.
    return Objects.equals(Object, Object);
  }
  ```

------------------

## 핵심정리
  - 꼭 필요한 경우가 아니면 재정의 하지 말자
  - **equals를 재정의 할때는 hashcode도 반드시 재정의하자**
  - 너무 복잡하게 해결하려 들지 말자
  - 핵심필드를 빠짐없이 위의 5가지 규약을 지키며 재정의하자
  - **equals의 매개변수 입력을 Object가 아닌 타입으로는 선언하지 말자**
# int 상수 대신 열거 타입을 사용하라

## 결론

- 안전하고 강력한 **Enum**을 사용하자
- **각 상수를 특정 데이터와 연결짓거나 상수마다 다르게 동작하게 할 때는 필요하다**
- 드물게 **하나의 메서드가 상수별로 다르게 동작**해야 할 때도 있다
이런 열거타입에서는 **`switch`문(상수별 메서드 구현을 하지 않은) 대신에 상수별 메서드 구현을 사용하자**
**열거 타입 상수 일부가 같은 동작을 공유**한다면 
**전략 열거 타입 패턴을 사용하자**

## 열거 타입(`Enum`)

- 열거타입 자체는 클래스이며, 
상수 하나당 자신의 인스턴스를 하나씩 만들어 `public static final` 필드로 공개한다
- 밖에서 접근할 수 있는 생성자를 제공하지 않으므로 **final**이다
- 인스턴스들은 딱 하나씩만 존재함이 보장된다

> **열거타입은 인스턴스 통제, 싱글턴이다**
> 

### 장점

- 임의의 메서드나 필드를 추가할 수 있다
- 임의의 인터페이스를 구현하게 할 수 있다
- Object 메서드들을 높은 품질로 구현해 놓았다
- Comparable과 Serializable을 구현했다

### 특징

- 열거타입은 근본적으로 불변이라 모든 필드는 `final`이여야 한다
- 필드를 `public`으로 선언해도 되지만, `private`으로 두고 별도의 `public` 접근자 메서드를 두는게 낫다

### Enum의 상수별 메서드 구현

> **상수별 메서드 구현을 상수별 데이터와 결합한 경우도 존재**
> 

```java
public enum Operation {
    PLUS("+"){
        public double apply(double x, double y) {
            return x + y;
        }}
		MINUS("-"){
				public double apply(double x, double y) {
            return x - y;
        }};
    private final String symbol;
    Operation(String symbol){
        this.symbol = symbol;
    }
    public abstract double apply(double x, double y);
}
```

- 열거 타입 상수끼리 코드를 **공유하기 어렵다는 단점**이 존재

### 해결법

- 계산 코드를 모두 상수에 중복해서 넣는다

```java
public enum Operation {
    PLUS("+"){
        public double apply(double x, double y) {
            return x + y;
        }},
    SUPER_PLUS("++"){
        public double apply(double x, double y) {
            return x + y;
        }},
		MINUS("-"){
        public double apply(double x, double y) {
            return x - y;
        }};
    private final String symbol;
    Operation(String symbol){
        this.symbol = symbol;
    }
    public abstract double apply(double x, double y);
}

Operation operation = Operation.PLUS;
operation.apply(2.5, 3.5);
```

- 계산 코드를 나누어 각각을 도우미 메서드로 작성한다음 각 상황에 맞게 적절하게 출력한다

```java
public enum Operation {
    PLUS("+", () -> applyPlus()),
    SUPER_PLUS("++", () -> applyPlus()),
    MINUS("-", () -> applyMinus());

    private final String symbol;
    private Supplier supplier;
    private static double x;
    private static double y;

    public void setValue(double x, double y){
        this.x = x;
        this.y = y;
    }

    Operation(String symbol, Supplier s) {
        this.symbol = symbol;
        this.supplier = s;
    }

    public static double applyPlus(){
        return applyPlusDetail(x, y);
    }

    public static double applyPlusDetail(double x , double y) {
        return x+y;
    }
    public static double applyMinus(){
        return applyMinusDetail(x,y);
    }

    public static double applyMinusDetail(double x , double y) {
        return x-y;
    }
}

@Test
    void testOperation(){
        Operation operation = Operation.PLUS;
        operation.setValue(1.5, 2.5);
        assertThat(operation.supplier.get()).isEqualTo(4.0);

   }
```

<aside>

📌 **위의 두가지 상황 모두 코드가 장황해져 가독성이 크게 떨어지고 오류발생가능성이 높아진다**

</aside>

### 전략 열거 타입 패턴

> **가장 깔끔한 상수별 메서드 구현**
> 

```java
public enum PayrollDay {
    MONDAY(PayType.WEEKDAY), TUESDAY(PayType.WEEKDAY),
    WEDNESDAY(PayType.WEEKDAY), THURSDAY(PayType.WEEKDAY),
    FRIDAY(PayType.WEEKDAY),
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);

    private final PayType payType;

    PayrollDay(PayType payType) {
        this.payType = payType;
    }

    double pay(double hoursWorked, double payRate) {
        return payType.pay(hoursWorked, payRate);
    }

    // 내부 정책 enum타입
    private enum PayType {
        WEEKDAY {
            double overtimePay(double hours, double payRate) {
                return hours <= HOURS_PER_SHIFT ? 0 : (hours - HOURS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            double overtimePay(double hours, double payRate) {
                return hours * payRate / 2;
            }
        };
        private static final int HOURS_PER_SHIFT = 8;

        abstract double overtimePay(double hours, double payRate);

        double pay(double hoursWorked, double payRate) {
            double basePay = hoursWorked * payRate;
            return basePay + overtimePay(hoursWorked, payRate);
        }
    }
}
```

**switch**문은 열거타입의 상수별 동작을 **구현하는 데 적합하지 않지만,**

> **기존 열거 타입에 상수별 동작을 혼합해 넣을 때는 `switch`문이 좋은 선택이 될 수 있다**
> 
- 기존 열거타입에 동작을 혼합해 넣어야 하는 경우에는 메서드를 하나 만들고 그 안에 `switch`문을 구성하자
```java
public static Operation inverse(Operation op){
	switch(op) {
		case PLUS : return Operation.MINUS;
		case MINUS : return Opertaion.PLUS;

		default : throw new AssertionError("알 수 없는 연산");
	}
}
```
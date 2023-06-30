# 멤버 클래스는 되도록 static으로 만들라

**중첩 클래스**의 종류는 

- 정적 멤버 클래스
- 멤버 클래스
- 익명 클래스
- 지역 클래스

이 중 첫번째를 제외한 나머지는 **내부클래스**에 해당한다

<img src = 1.png>

<aside>

📌 멤버 클래스는  **static** 클래스로 선언하자!
**static**이 아닌 내부 인스턴스 클래스는 외부와 연결이 되어 있어 
'외부 참조'를 갖게되어 메모리를 더 먹고, 느리며, 또한 
GC 대상에서 제외되는 여러 문제점을 일으키기 때문이다.

</aside>

## 정적 멤버 클래스

- 다른 클래스 안에 선언되고, 바깥 클래스의 private멤버에도 접근할 수 있다는 점만 제외하고는 일반 클래스와 똑같다
- 다른 정적 멤버와 똑같은 접근 규칙을 적용받는다
→ `private`으로 선언하면 바깥 클래스에서만 접근할 수 있는 식
- 열거 타입도 암시적 **static**

<img src = 2.png>

- 정적 멤버 클래스와 비정적 멤버 클래스의 구문상 차이는 단지 **static**이 붙어 있고 없고 뿐이지만, 의미상 차이는 의외로 크다
- 바깥 클래스가 표현하는 객체의 한 부분(구성요소)일 때 사용
- `example`: **Map의 Entry**
    - Map과 연관
    - Entry의 `getKey()`, `getValue()` 등의 메소드를 직접 사용 X
    - Map 안의 Entry는 interface다.
    Map을 구현하는 구현체인 HashMap에서 해당 interface를 implements하여 새로운 클래스를 정의한다.

<img src = 3.png>

<aside>
📌 **비정적 멤버 클래스의 인스턴스는 바깥 클래스의 인스턴스와 암묵적으로 연결된다**

</aside>

- 따라서 비정적 멤버 클래스의 인스턴스 메서드에서 정규화된 `this`를 사용해 바깥 인스턴스의 메서드를 호출하거나 바깥 인스턴스의 참조를 가져올 수 있다.
    - **정규화된 `this` (클래스명.this)** : 바깥 인스턴스의 메서드를 호출하거나 바깥 인스턴스의 참조를 가져올 수 있다
        
        <img src = 4.png>
        

### 따라서 멤버클래스는,

- 중첩클래스의 인스턴스가 바깥 인스턴스와 독립적으로 존재할 수 있다면 **정적 멤버클래스**로 만들어야 한다
    - **비정적 멤버 클래스는 바깥 인스턴스 없이는 생성할 수 없기 때문이다**

---

## 비정적 멤버 클래스

- **static**이 붙지 않은 멤버 클래스
- 비정적 멤버 클래스의 인스턴스 메서드에서 `클래스명.this`를 사용해 바깥 인스턴스의 메서드나 참조 가져올 수 있음
- 바깥 인스턴스 없이는 생성 불가<br>
(👉 비정적 멤버 클래스의 인스턴스는 바깥 클래스의 인스턴스와 암묵적인 연결)

```java
public class TestClass {
    private String name = "yeonlog";

    public class PublicSample {
        public void printName() {
            // 바깥 클래스의 private 멤버 가져오기
            System.out.println(name);
        }

        public void callTestClassMethod() {
            // 바깥 클래스의 메소드 호출하기
            TestClass.this.testMethod();
        }
    }

    public PublicSample createPublicSample() {
        return new PublicSample();
    }

    public void testMethod() {
        System.out.println("hello world");
    }
}
```

<img src = 5.png>

- 바깥 클래스의 인스턴스 메서드에서 비정적 멤버 클래스의 생성자 호출
- **바깥 클래스 인스턴스.new 멤버클래스()** 호출
    - 위 두 방법으로 바깥 클래스 - 비정적 멤버 클래스의 관계 생성됨
    - 관계 정보는 비정적 멤버 클래스의 인스턴스 내에서 **메모리 공간을 차지**

```java
TestClass test = new TestClass();
PublicSample publicSample1 = test.createPublicSample(); // 바깥 클래스의 인스턴스 메서드에서 생성자 호출
PublicSample publicSample2 = test.new PublicSample(); // 바깥 인스턴스 클래스.new 멤버클래스()
```

### 멤버 클래스에서 바깥 인스턴스에 접근할 일이 없다면 static을 붙이자.

**static을 생략하면 바깥 인스턴스로의 숨은 외부 참조를 갖게된다
그로인해,** 

- **바깥 인스턴스 - 멤버 클래스 관계**를 위한 시간과 공간 소모 **(참조를 저장하려면)**
- `Garbage Collection`이 바깥 클래스의 인스턴스 수거 불가 -> 메모리 누수 발생
- 참조가 눈에 보이지 않아 개발이 어려움

### 그럼 비정적 멤버 클래스는 언제 쓰는가?

👉 어댑터 정의 시 자주 쓰임.

= **어떤 클래스의 인스턴스를 감싸서 다른 클래스의 인스턴스처럼 보이게 하는 뷰로 사용**

`example` - Map 인터페이스의 구현체

```java
👉 keySet(), entrySet(), values()가 반환하는 자신의 [컬렉션 뷰]를 구현할 때 활용

public class HashMap<K, V> extends AbstractMap<K, V>
        implements Map<K, V>, Cloneable, Serializable {

    final class EntrySet extends AbstractSet<Map.Entry<K, V>> {
        // size(), clear(), contains(), remove(), ...
    }

    final class KeySet extends AbstractSet<K> {
        // size(), clear(), contains(), remove(), ...
    }

    final class Values extends AbstractCollection<V> {
        // size(), clear(), contains(), remove(), ...
    }
}
```

---

## 익명 클래스

- **이름이 없는 클래스**
- 바깥 클래스의 멤버가 아님
- **쓰이는 시점에 선언 + 인스턴스 생성**

### 익명 클래스의 제약 사항

- 비정적인 문맥에서 사용될 때만 바깥 클래스의 인스턴스 참조 가능
    - **static으로 선언된 메소드에서는 static만 접근이 가능하다.**
- 정적 문맥에서 **static final 상수 외의 정적 멤버 갖기 불가능**
- 선언 지점에서만 **인스턴스 생성 가능**
- `instanceof` 검사 및 클래스 이름이 필요한 작업 수행 불가
    - 선언과 동시에 인스턴스 생성하고 더이상 사용하지 않음. **클래스 이름이 없음.**
- 인터페이스 구현 및 다른 클래스의 상속 X
- 짧지 않으면 가독성 ↓

```java
public class Calculator {
    private int x;
    private int y;

    public Calculator(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public int plus() {
        Operator operator = new Operator() {
            private static final String COMMENT = "더하기"; // 상수
            // private static int num = 10; // 상수 외의 정적 멤버는 불가능

            @Override
            public int plus() {
                // Calculator.plus()가 static이면 x, y 참조 불가 -> 비정적인 문맥에서만 사용가능
                return x + y;
            }
        };
        return operator.plus();
    }
}

interface Operator {
    int plus();
}
```

### 언제 사용하는가?

- 즉석에서 작은 함수 객체나 처리 객체를 만드는 데 주로 사용

👉 람다 등장 이후로 람다가 이 역할을 대체

```java
List<Integer> list = Arrays.asList(10, 5, 6, 7, 1, 3, 4);

// 익명 클래스 사용
Collections.sort(list, new Comparator<Integer>() {
    @Override
    public int compare(Integer o1, Integer o2) {
        return Integer.compare(o1, o2);
    }
});

// 람다 도입 후
Collections.sort(list, Comparator.comparingInt(o -> o));
```

- 정적 팩터리 메소드 구현 시 사용

```java
static List<Integer> intArrayAsList(int[] a) {
    Objects.requiredNonNull(a);
    
    return new AbstracktList<>() {
        @Override public Integer get(int i) {
            return a[i];
        }
    }
}
```

---

## 지역 클래스

- 지역 변수를 선언할 수 있는 곳이면 어디서든 선언 가능
- 유효 범위는 지역 변수와 동일
- 지역클래스가 비정적 멤버클래스처럼 바깥클래스를 참조할 수 있는지 **정규화된 `this`**를 통해?
→ **가능**
    
  <img src = 6.png>
    

### 각 클래스와의 공통점

- `멤버 클래스`: 이름이 있고 반복해서 사용 가능
- `익명 클래스`: 비정적 문맥에서 사용될 때만 바깥 인스턴스 참조 가능
- `익명 클래스`: 정적 멤버를 가질 수 없음
- `익명 클래스`: 가독성을 위해 짧게 작성해야 함

```java
public class TestClass {
    private int number = 10;

    public TestClass() {
    }

    public void foo() {
        // 지역변수처럼 선언
        class LocalClass {
            private String name;
            // private static int staticNumber; // 정적 멤버 가질 수 없음

            public LocalClass(String name) {
                this.name = name;
            }

            public void print() {
                // 비정적 문맥에선 바깥 인스턴스를 참조 가능
                // foo()가 static이면 number에서 컴파일 에러
                System.out.println(number + name);
            }
        }
        LocalClass localClass1 = new LocalClass("local1"); // 이름이 있고
        LocalClass localClass2 = new LocalClass("local2"); // 반복해서 사용 가능
    }
}
```

---

## 결론

- 멤버 클래스: 메소드 밖에서 사용해야 하거나 너무 긴 경우
    - **정적**: 바깥 인스턴스 참조 X
    - **비정적**: 바깥 인스턴스 참조 O
- 익명 클래스: 한 메서드 안에서만 사용 + 인스턴스 생성 시점이 단 한 곳 + 해당 타입으로 쓰기 적합한 클래스나 인터페이스 이미 존재
**비정적 문맥에서 사용될 때만 바깥 인스턴스 참조 가능**
- 지역 클래스: 그 외. (잘 쓰이지 않음)
**비정적 문맥에서 사용될 때만 바깥 인스턴스 참조 가능**
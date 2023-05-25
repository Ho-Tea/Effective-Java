# equals를 재정의하려거든 hashCode도 재정의하라
> hashCode : 객체의 주소값을 변환하여 생성한 객체의 고유한 정수값
- **equals를 재정의한 클래스 모두에서 hashCode도 재정의해야 한다**
- 그렇지 않으면, `hashCode` 일반 규약을 어기게 되어 해당 클래스의 인스턴스를 HashMap이나<br> HashSet같은 컬렉션의 원소로 사용할 때 문제를 일으킬 것이다

- ## Object 명세
  - (1) `equals` 비교에 사용되는 정보가 변경되지 않았다면, 애플리케이션이 실행되는 동안 <br>그 객체의 `hashCode` 메서드는 몇 번을 호출해도 일관되게 항상 같은 값을 반환해야 한다
    - 단, 애플리케이션을 다시 실행한다면 이 값이 달라져도 상관없다
  - (2) **`equals(Object)`가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 똑같은 값을 반환해야 한다**
  - (3) `equals(Object)`가 두 객체를 다르다고 판단했더라도, 두 객체의 hashCode가 서로 다른 값을 반환할 필요는 없다
    - 단, 다른 객체에 대해서는 다른 값을 반환해야 해시테이블의 성능이 좋아진다

- ## 크게문제가 되는것은 2번째 조항
  - 즉, 논리적으로 같은 객체는 같은 해시코드를 반환해야 한다.
    ``` java
    Map<PhoneNumber, String> m = new HashMap<>();
    m.put(new PhoneNumber(707, 867, 5309), "제니");
    // 이 코드 다음에 

    m.get(new PhoneNumber(707, 867, 5309)); //를 실행하면 null 반환
    ```
    - `PhoneNumber`클래스는 hashCode를 재정의 하지 않았기 때문에 논리적 동치인<br> 두 객체가 서로 다른 해시코드를 반환하여 두번째 규약을 지키지 못한다

- ## 좋은 해시코드 재정의 요령
  - int result = c
  - c 의 계산 방식
    - 기본 타입 필드(String, Integer 등)라면 Type.hashCode(필드)
    - 참조 타입 필드(기본 타입 이외의 타입)라면 재귀적으로 호출, 계산이 복잡해진다면 표준형을 만들어 `hashCode` 호출
      - `ex) getLottoMachine().getLottoTicket().getLotto().getLottoNumber()`
    - 배열 필드라면 핵심 원소에 대해 위의 1, 2 방식으로 해시코드 계산하여 비교, 모든 원소가 핵심 원소라면 `Arrays.hashCode` 이용

  - result = 31 * result + c 
    > 31인 이유는 홀수이면서 소수여서...등

  ``` java
  public class Address {
    
    private String 도시;
    private String 구;
    private String 동;
    
    @Override
    public int hashCode() {
        int result = String.hashCode(도시);
        result = 31 * result + String.hashCode(구);
        result = 31 * result + String.hashCode(동);
        return result;
    }
  }
  // 해시함수 제작요령이 최첨단은 아니지만 충분히 훌륭하다
   @Override public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
   }
  // Objects클래스는 정적 메서드 hash를 이용해 위의 과정을 단 한줄로 간소화 할 수 있다
  // 하지만, 느리다
  ```

-------------
## 핵심정리
- 성능을 높인답시고 해시코드를 계산할 때 핵심필드를 생략해서는 안된다
- hashCode가 반환하는 값의 생성규칙을 API사용자에게 자세히 공표하지 말자
  - **자율성이 높아진다**
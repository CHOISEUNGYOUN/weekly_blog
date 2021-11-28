# 경쟁상태와 데이터 레이스(Race Condition vs. Data Race)

경쟁상태는 이벤트의 실행 시점이나 순서가 프로그램의 정확성에 영향을 미치는 경우 문제가 된다. 보통 경쟁상티를 일으키는 조건으로 외부적인 타이밍이나 비결정적인 순서를 꼽는다. 대표적인 예는 컨텍스트 스위칭, OS에서 보내는 신호, 멀티프로세스에서 일어나는 메모리 운영, 하드웨어 간섭이 있다.

데이터 레이스는 프로그램 상에서 한 메모리에 두번의 접근이 발생할 때 발생하는데 그 조건은 아래와 같다:
1. 동일한 메모리 주소를 가리키는 경우
2. 두개의 쓰레드에서 동시에 실행되는 경우
3. 읽기 작업이 아닌 경우
4. 동기화 된 작업이 아닌 경우

이외에도 다른 정의가 있지만 이정도만 서술해도 큰 문제가 되지 않는다. 해당 설명은 Microsoft Research 에서 근무하고 있는 Sebastian Burckhardt의 말을 인용한 것이다. 해당 내용 중에서 두가지 내용은 이해하는데 조금의 노력이 필요하다. **"동시에 실행"** 된다는 말은 한번에 하나의 작업만 수행하도록 하는 잠금장치와 같은 역할을 하는 것이 없다는 의미이다. **"동기화된 작업이 아닌 경우"** 라는 말은 프로그램이 잠금역할을 하는 특별한 메모리 보호장치가 있는데 이것이 동기화 되지 않았음을 의미한다.

실제로는 경쟁상태와 데이터 레이스에 꽤 많은점에서 공통점이 있다. 많은 경생상태 조건들은 데이터 레이스 때문에 발생했고 많은 데이터 레이스는 경쟁상태를 유발한다는 것이다. 다시말해 경쟁상태는 데이터 레이스 없이 발생 할 수 있고 경쟁상태 조건이 아닌 데이터 레이스 또한 발생 할 수 있다는 것이다. 이를 좀 더 쉽게 이해하기 위해 두 은행에서 돈을 송금한다고 생각해보자.

```js
transfer1 (amount, account_from, account_to) {
  if (account_from.balance < amount) return NOPE;
  account_to.balance += amount;
  account_from.balance -= amount;
  return YEP;
}
```
당연히 알고 있겠지만 이 코드는 실제로 돈을 송금하는 코드가 아니다. 하지만 계좌의 잔고가 음수인 경우 송금이 되거나 돈을 차감하는 일이 없어야 함을 이 예시를 통해 직관적으로 이해 할수 있다. 외부에서 동기화를 시키지 않고 여러 쓰레드를 생성하게 된다면 이 함수는 데이터 레이스(여러 쓰레드가 동시에 계좌 잔고를 갱신하려고 시도)와 경쟁상태(수평적인 컨텍스트 조건에서 계좌에 돈을 추가 및 차감을 실행.) 모두 야기시킬 것이다. 이런 상태를 아래와 같이 수정 보완할 수 있다.

```js
transfer2 (amount, account_from, account_to) {
  atomic {
    bal = account_from.balance;
  }
  if (bal < amount) return NOPE;
  atomic {
    account_to.balance += amount;
  }
  atomic {
    account_from.balance -= amount;
  }
  return YEP;
}
```
여기서 "원자적 조건"은 해당 언어의 런타임에서 수행된다. 아마 단순히 `atomic` 시작 블록에서 쓰레드 뮤텍스를 획득하여 블록 종료와 함께 해제한다거나 특정 트랜잭션 기능을 사용한다거나 작업 실행 중 간섭할 수 없도록 특정 장치를 구현했을 수도 있다. 어찌됐든 해당 코드의 블록들이 원자적으로 실행되기만 한다면 어떤 방법을 사용했는지는 상관없긴 하다.

`transfer2` 함수의 경우 다중 쓰레드에서 호출 되는 경우 데이터 레이스는 발생하지 않는다. 하지만 동기화 되지도 않고 계좌에서 금액을 추가 및 차감을 시키는 경쟁상태 조건을 가진 멍청한 함수가 되었다. 기술적 관점에서 `transfer2` 함수의 문제는 다른 쓰레드가 불변 상태가 아닌 메모리 상태를 본다는 것이다.

불변성을 유지해주기 위해선 좀 더 나은 잠금 전략을 구사해야 한다. 원자 시맨틱이 해당 블록의 원자 영역 종료 지점까지 도달할때 까지 유지되어야 한다면 아래의 방법은 좀 디테일함이 떨이진다.

```js
transfer3 (amount, account_from, account_to) {
  atomic {
    if (account_from.balance < amount) return NOPE;
    account_to.balance += amount;
    account_from.balance -= amount;
    return YEP;
  }
}
```
이 함수는 데이터 레이스와 경쟁상태 조건 모두 해결했다. 여기서 데이터 레이스는 발생하지만 경쟁상태는 없는 조건으로 만들어 보면 어떨까? 아래의 코드를 살펴보자.

```js
transfer4 (amount, account_from, account_to) {
  account_from.activity = true;
  account_to.activity = true;
  atomic {
    if (account_from.balance < amount) return NOPE;
    account_to.balance += amount;
    account_from.balance -= amount;
    return YEP;
  }
}
```
이 예시에서 볼수 있듯이 해당 계좌에서 특정 작업이 이루어진다는 표식을 남겨두었다. 해당 표식이 데이터 레이스 조건을 가지고 있는 것이 위험할까? 딱히 그러진 않을것이다. 예를 들어 저녁에는 모든 계좌에서 진행되고 있는 쓰레드를 종료시키고 직접 검사가 필요하다고 표식이 남겨진 계좌 중 무작위로 10개의 계좌를 선택 할 수도 있다. 이런 경우 데이터 레이스는 완전히 해가 되는 조건이 아니다.

모든 조건을 한번 표로 정리하자면 다음과 같다.

|                    |데이터 레이스인 경우|데이터 레이스가 아닌 경우|
|--------------------|---------------|-------------------|
|경쟁 상태 조건인 경우    |transfer1      |transfer2          |
|경쟁 상태 조건이 아닌 경우|transfer4      |transfer3          |

우리가 살펴보았던 예시중 주목해야 될 것은 `transfer2` 함수와 `transfer4` 함수이다. 해당 예시에서 데이터 레이스를 방지하는 것은 동시 정확성을 보장해주지도 충족시키지도 못하는 아주 미미한 요소인 것을 확인 할 수 있었다. 그런데도 왜 데이터 레이스를 고려해야할까? 여기에는 몇가지 이유들이 존재한다. 첫번째로 데이터 레이스를 방지한 프로그램은 어떤 약한 메모리 모델을 가지고 있더라도 이로부터 독립적이라는 것이다. 두번째는 데이터 레이스는 C 나 C++ 에서는 정의되지 않은 행위이나 다른 언어에서는 복잡한 시멘틱 요소들로 구성되어 있다는 것이다. 셋째로 데이터 레이스는 원자적으로 찾기 쉽다는 것이다. 데이터 레이스는 메모리 접근을 모니터링하기만 해도 확인 할 수 있다. 앱의 시멘틱 요소에 대한 지식이 없어도 알 수 있다. 이에 반해 경쟁상태는 앱 레벨의 불변 조건들과 함께 맞물려 있기 때문에 분석하기 위해선 해당 조건들을 좁힐 필요가 있다.


이 글의 내용은 분명하지만 이 내용을 분명히 더 잘 알고 있어야 하는 사람들이(예를 들자면 동시성 정확성을 연구하고 있는 사람들) 데이터 레이스와 경쟁상태에 대한 정의를 혼동하고 있는 것을 알게 되었다. 더 심각한건 사람들이 이미 두 단어에 대한 기본 개념을 완벽하게 이해하고 있음에도 경쟁상태를 데이터 레이스와 혼동하여 사용하는 것을 종종 발견한다. 이 글을 쓴 나 자신조차도 말이다.

Original Source:
[Race Condition vs. Data Race](https://blog.regehr.org/archives/490)
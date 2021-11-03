# Chapter 14 Fear of the Unknown (모르는 것에 대한 두려움)

## 목표 - 누락된 값을 구분하기
값이 없는 데이터가 존재하는 것은 어쩔 수 없는 일이다. 모든 컬럼의 값을 지정하여 저장하거나 아무런 의미 없는 데이터를 선언하는것이 일반적이다. SQL에서는 이런 데이터를 `NULL` 키워드로 지원한다. 

null을 잘 이해하고 사용한다면 유용하게 쓸 수 있는데 SQL에서는 주로 아래와 같은 용도로 활용된다.
* 재직중인 근로자의 퇴직일과 같이 아직 일어나진 않으나 필요한 컬럼 값인 경우.
* 차량의 연비 측정과 같이 지금 당장 계산 할 수 없는 값의 컬럼이 필요한 경우.
* `2009-12-32`일과 같이 올바르지 않은 변수 값으로 함수를 실행할때 결과값으로 리턴하는 경우.
* `OUTER JOIN`을 사용했을 때 해당 컬럼에 데이터가 존재하지 않는 경우.

## 안티패턴 - NULL을 일반 값처럼 사용
많은 개발자들이 이런 null 에 대한 사용 예를 잘못 이해하고 있어 무방비 상태로 노출되곤 한다. 다른 수많은 언어들과는 다르게 SQL에서는 null을 0이나 false, "" 와는 다르게 인식하고 처리한다. 이는 SQL뿐만 아니라 다른 여러 데이터베이스에서도 동일하다. 하지만 Oracle과 Sybase 에서는 빈 문자열과 동일하게 취급한다. 이런 null 의 특성은 여러가지 다른 현상을 유발한다.

### Null을 표현식에서 활용하기
어떤 개발자들은 null 값에 사칙연산을 적용하는 경우가 있다. 예를 들어 `hour` 라는 컬럼에 아무런 값도 없는 경우 기본값으로 10을 입력하기 위해 빈 컬럼에 10을 더하는 경우가 있다.

```sql
SELECT hours + 10 FROM Bugs;
```
하지만 위 쿼리는 null을 반환한다. 위에서 언급했듯이 null은 0이랑 동일한 데이터 타입이 아니다. 또한 null은 빈 문자열과 동일하지도 않다. 표준 SQL에서는 어떤 문자열 값이라도 null과 병합한다면 null을 반환하게끔 설계되어있다. (Oracle과 Sybase는 예외)

Null은 false 와도 동일하지 않다. AND, OR 나 NOT 과 같은 Boolean 표현식을 사용하더라도 null은 null을 반환한다.

### 컬럼값이 null인 데이터 찾기
아래 쿼리는 `assigned_to` 컬럼의 데이터가 123인 데이터만 반환한다.
```sql
SELECT * FROM Bugs WHERE assigned_to = 123;
```
이런 논리로 생각했을 때 바로 아래 쿼리는 `assgined_to` 가 123이 아닌 모든 로우를 리턴한다고 생각할 것이다.
```sql
SELECT * FROM Bugs WHERE NOT (assigned_to = 123);
```
하지만 이 쿼리는 `assigned_to`가 null인 로우 데이터는 반환하지 않는다. 앞서 말했듯이 null은 null이기 때문에 어떠한 비교문에도 true로 동작하지 않는다. 만약 null인 값과 null이 아닌 모든 값들을 찾고 싶다면 아래와 같이 쿼리를 작성해야한다.

```sql
SELECT * FROM Bugs WHERE assigned_to = NULL;
SELECT * FROM Bugs WHERE assigned_to <> NULL;
```
`WHERE` 문은 항상 조건이 참인 경우에만 결과값을 반환한다. 하지만 `NULL` 값은 항상 거짓이기 때문에 이렇게 `NULL`에 대한 비교문을 따로 작성해야만 한다.

### Null을 쿼리 변수값으로 사용하기
변수를 사용하는 SQL 표현식(함수나 프로시져)에서 null을 일반 값으로 취급하는 경우 null을 활용하기 어려워진다.
```sql
SELECT * FROM Bugs WHERE assigned_to = ?;
```
정수와 같이 일반적으로 예측 할 수 있는 값을 변수로 사용하지 않고 `NULL`을 사용하는 것은 SQL 표준에서 권장하지 않는다.

### 문제 회피하기
많은 개발자들이 null을 다루는 것 자체를 까다로워해서 null 자체를 데이터베이스에 허용하지 않는 경우가 종종 발견된다. 예를 들어 "unknown" 이나 "inapplicable"을 기본값으로 적용한다. 이런 경우 우리가 예측하지 못한 또다른 변수들이 발생하게 된다.

```sql
CREATE TABLE Bugs (
  bug_id SERIAL PRIMARY KEY,
  -- other columns
  assigned_to BIGINT UNSIGNED NOT NULL,
  hours NUMERIC(9,2) NOT NULL,
  FOREIGN KEY (assigned_to) REFERENCES Accounts(account_id)
);
```
작성중...

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter14 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

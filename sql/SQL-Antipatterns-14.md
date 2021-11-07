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

위 DDL예시 에서 `hours` 컬럼에 null 대신 -1을 아무의미 없는 값으로 사용한다고 가정하자. 
```sql
INSERT INTO Bugs (assigned_to, hours) VALUES (-1, -1);
```

`hours` 컬럼은 정수이고 시간을 셀때 음수는 존재하지 않기 때문에 -1을 활용할 수는 있다. 하지만 문제는 `SUM()` 이나 `AVG()`와 같은 통계 함수를 활용하여 지표를 계산할때 문제가 발생한다. 이런 경우 null을 제외하기 위해 `<>`을 사용했던 것과 동일하게 -1을 제외시켜야만 한다.

```sql
SELECT AVG( hours ) AS average_hours_per_bug FROM Bugs
WHERE hours <> -1;
```
이런 경우 뿐만 아니라 다양한 경우에서 null을 회피하기 위해 다른 아무의미 없는 데이터를 기본값으로 설정하고 그 값을 제외하는 비효율적인 작업들을 통계 함수를 사용할 때 반드시 해야만 한다. null을 활용한다면 단지 null이 아닌 데이터만 불러오면 되기 때문에 규약상 더 깔끔하다. 즉, 없는 컬럼 데이터는 null로 선언하는것이 옳은 방식이다.

## 어떻게 안티패턴을 구분하는가
* `assigned_to` 또는 다른 컬럼에 값이 없는 로우를 불러오려면 어떻게 해야하는지 고민하는 경우<br>
동등 연산자로 null 값을 구분할 수 없다. `IS NULL`을 활용해서 null인 데이터를 확인해야한다.
* 몇몇 사용자들의 성명 데이터는 비어있다고 어플리케이션에서 나타나는데, 사실 그들의 이름이 데이터베이스에 존재하는 경우.<br>
이 문제는 null이 포함된 데이터를 문자열로 합치려고 시도하기 때문에 발생하는 문제이다.
* 프로젝트를 수행한 총 시간 값의 통계를 내는데 시간 오차가 발생하는 경우.<br>
null 값이 포함된 데이터를 걸러내지 못해 발생하는 문제이다. 앞서 말했듯이 null은 동등 연산자로 비교할 수 없다.
* `unknown`이라는 특정 문자값을 알수없는 데이터의 기본값으로 지정할 수 없어 다른 데이터로 변환하여 마이그레이션 해야하는 경우.<br>
이는 null 대신에 다른 값을 알수없는 값의 기본값으로 설정하여 발생하는 문제이다. 이런 경우 null을 활용하는 것이 최선이다.

대부분의 경우 null을 피하기 위해 다른 값을 설정하는 것 보다 null 자체를 활용하는 방법을 아는 것이 더 효과적이다.

## 안티패턴 사용이 정당화 되는 경우
null을 사용하는 것 자체는 안티패턴이 아니다. null을 일반적인 값과 동일하게 사용하는 것이 안티패턴이다.

null을 일반적인 데이터로 취급을 하는 경우는 다른 외부 형식 데이터를 들여오는 경우이다. 예를들어 CSV 파일을 읽으려고 하는 경우 해당 CSV에는 모든 컬럼의 데이터가 텍스트로 명시되어 있어야 한다. 이 경우 `\N`을 사용하여 해당 컬럼의 값이 null이라는 것을 명시적으로 표현해줘야 한다.

이와 비슷한 경우로 사용자가 입력한 데이터가 null로 바로 표현되지 않는 경우도 있다. 예를 들어 `MS .NET 2.0`에서는 `ConvertEmptyStringToNull` 이라는 웹 인터페이스가 존재하는데 이 기능은 빈 문자열 값을 자동으로 null 변환해준다.

마지막으로 null은 빈 값을 명시적으로 구분해줘야 하는 경우 제대로 활용 할 수 없다. 예를 들어 해당 버그 케이스에 `assigned_to` 컬럼이 비어 있는 경우 담당자가 지정되지 않은 건지 담당자가 퇴사를 한 것인지 구분할 수 없다.

## 해결책 - Null을 고유값으로 활용하기
null로 인해 발생하는 대부분의 문제들은 SQL의 3차 논리 체계를 제대로 이해하지 못해 발생한다. 대부분의 개발자들은 true와 false만 존재하는 2차 논리에 익숙해져 있어서 해당 개념이 낯설 것이다. 그렇기에 우리는 SQL에서 null을 어떻게 다뤄야 하는지 이해하고 사용하는 방법을 배워야 한다.

### 스칼라 표현식에서의 Null
스칼라 표현식에서 null은 다음과 같이 동작한다.
|표현식            |예상     |실제 |이유                                      |
|----------------|--------|----|-----------------------------------------|
|NULL = 0        |TRUE    |NULL|null은 0이 아니다.                          |
|NULL = 12345    |FALSE   |NULL|특정되지 않은 값과 비교를 하는 경우 null로 반환된다.|
|NULL <> 12345   |TRUE    |NULL|특정되지 않은 값과 비교를 하는 경우 null로 반환된다.|
|NULL + 12345    |12345   |NULL|null은 0이 아니다.                          |
|NULL || ’string’|’string’|NULL|null은 빈 문자열이 아니다.                    |
|NULL = NULL     |TRUE    |NULL|특정되지 않은 값들 끼리 비교할 수 없다.           |
|NULL <> NULL    |FALSE   |NULL|특정되지 않은 값들 끼리 비교할 수 없다.           |

### 불리언 표현식에서의 Null
불리언 표현식에서 null은 다음과 같이 동작한다.
|표현식          |예상  |실제  |이유                                   |
|--------------|-----|-----|--------------------------------------|
|NULL AND TRUE |FALSE|NULL |null은 거짓이 아니다.                     |
|NULL AND FALSE|FALSE|FALSE|어떠한 참의 값과 false를 AND 비교하면 거짓이다.|
|NULL OR FALSE |FALSE|NULL |null은 거짓이 아니다.                     |
|NULL OR TRUE  |TRUE |TRUE |어떠한 참의 값과 false를 OR 비교하면 참이다.  |
|NOT (NULL)    |TRUE |NULL |null은 거짓이 아니다.                     |

여기서 볼 수 있듯이 null은 참도 거짓도 아니다. `NOT (NULL)` 문항은 다른 null 값을 반환 할 뿐이다.

### null 값을 찾는 경우
null 값을 불리언 표현식으로 비교할 수 없기 때문에 전통적인 SQL에서는 `IS NULL` 또는 `IS NOT NULL`을 사용한다.
```sql
SELECT * FROM Bugs WHERE assigned_to IS NULL;
SELECT * FROM Bugs WHERE assigned_to IS NOT NULL;
```

추가로 SQL-99 표준에서는 다른 방법으로 `IS DISTINCT FROM`을 제시하고 있다. 이는 일반적인 부등 연산자인 `<>`와 동일하게 동작한다.
```sql
SELECT * FROM Bugs WHERE assigned_to IS NULL OR assigned_to <> 1;
SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM 1;
```

쿼리의 파라미터로 사용한다면 아래와 같이 사용 할 수 있다.
```sql
SELECT * FROM Bugs WHERE assigned_to IS DISTINCT FROM ?;
```


> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter14 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

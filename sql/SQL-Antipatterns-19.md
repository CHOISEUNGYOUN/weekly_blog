# Chapter 19 Implicit Columns (명시적이지 않은 컬럼들)

## 목표 - 타이핑 덜 하기
개발자들은 대게 타이핑 하는것을 좋아하지 않는다. 아이러니하게도 이것이 그들의 선택한 직업의 주된 일임에도 말이다. 개발자들이 타이핑을 많이 해야되는 대표적인 작업은 모든 컬럼명을 명시하는 SQL 쿼리를 작성하는 것이다.

```sql
SELECT bug_id, date_reported, summary, description, resolution,
  reported_by, assigned_to, verified_by, status, priority, hours
FROM Bugs;
```

이런 귀찮음 때문에 많은 개발자들이 당연하게 와일드카드(`*`)를 쿼리 작성시 활용한다. `*` 기호는 모든 컬럼을 의미하기 때문에 암시적으로 모든 컬럼의 값을 조회하게 된다. 이는 쿼리문을 간결하게 만들어준다.

```sql
SELECT * FROM Bugs;
```
이와 비슷하게 `INSERT` 문을 작성할때도 컬럼명을 암시적으로 가리키는 기능이 있다. 이는 바로 컬럼에 해당되는 값들을 순차적으로 나열하여 작성하는 경우 컬럼명 없이도 데이터를 추가해주는 기능이다.

```sql
# 컬럼명을 전부 나열하는 경우
INSERT INTO Accounts (account_name, first_name, last_name, email,
  password, portrait_image, hourly_rate) VALUES
  ('bkarwin', 'Bill', 'Karwin', 'bill@example.com', SHA2('xyzzy'), NULL, 49.95);

# 컬럼명을 나열하지 않는 경우
INSERT INTO Accounts VALUES (DEFAULT,
  'bkarwin', 'Bill', 'Karwin', 'bill@example.com', SHA2('xyzzy'), NULL, 49.95);
```

## 안티패턴 - 잘못된 길로 인도하는 지름길
와일드카드와 컬럼명을 명시하지 않고 로우를 추가하는 것이 타이핑을 덜 하게 해주는 역할을 해주지만 이를 남발하면 여러가지 문제점을 낳는다.

### 리팩토링의 어려움
`Bugs` 테이블에 `date_due` 라는 컬럼을 스케쥴 관리 차원에서 추가한다고 가정해보자.
```sql
ALTER TABLE Bugs ADD COLUMN date_due DATE;
```
이 경우 컬럼명 없이 `INSERT` 구문을 선언한 경우 새로운 컬럼에 대한 데이터를 추가하지 않았기 때문에 에러를 발생시킬 것이다.
```sql
INSERT INTO Bugs VALUES (DEFAULT, CURDATE(), 'New bug', 'Test T987 fails...',
  NULL, 123, NULL, NULL, DEFAULT, 'Medium', NULL);

-- SQLSTATE 21S01: Column count doesn't match value count at row 1
```
컬럼명 암시적 선언을 통한 로우 추가를 하는 경우 반드시 모든 로우에 순서대로 값을 넣어줘야 한다. 컬럼이 변경되면 에러를 발생하기도 하고 심지어 잘못된 데이터를 바꿔 넣는 실수를 할 수도 있다.

와일드카드를 사용하는 경우 컬럼명을 모르기 때문에 컬럼이 선언된 순서대로 데이터를 반환한다. 이 경우 조심해야될 점은 컬럼의 순서가 변경되거나 중간에 삭제하는 경우 순서가 달라지기 때문에 문제가 발생한다는 점이다.

```sql
ALTER TABLE Bugs DROP COLUMN verified_by;
```
이런 사소한 문제 때문에 에러가 전체 어플리케이션으로 전파되어 서비스가 중단되는 문제를 발생시킬수도 있다.

### 숨겨진 비용
와일드카드를 사용하는 쿼리는 성능과 확장성 측면에서도 좋지 않다. 더 많은 컬럼을 호출하면 할수록 더 많은 데이터가 데이터베이스 서버와 어플리케이션 서버내 네트워크에 전송되게 된다.

당신의 앱 서버는 한번에 많은 쿼리를 동시에 호출하고 있을 것이다. 이 경우 각 쿼리들은 같은 네트워크 대역폭 내에서 각각 호출 될 것이다. 기가바이트 네트워크를 사용하고 있더라도 수천개의 로우를 수백개가 되는 앱 클라이언트에서 동시에 호출하게 된다면 금방 포화상태가 될 것이다.

Active Record 와 같은 객체 관계 매핑(ORM)에서는 와일드카드를 기본으로 테이블의 로우 데이터를 호출한다. ORM이 해당 기능을 오버라이딩 하는 기능을 제공함에도 많은 개발자들은 이를 크게 신경쓰지 않고 기본값을 활용하고 있다.

### 요청한대로 반환
많은 개발자들이 와일드카드를 사용할때 특정 컬럼만 제외하는 기능이 없는지 궁금해하곤 한다. 아마 데이터베이스에서 값을 호출 할 때 와일드카드를 사용하면서도 필요없는 데이터를 제외하고 싶은 마음에서 나온 질문일 것이다. 하지만 SQL에는 그러한 기능이 존재하지 않는다. 필요한 컬럼값만 반환하고 싶다면 컬럼명을 명시하는것 말고는 방법이 없다.


## 어떻게 안티패턴을 구분하는가

## 안티패턴 사용이 정당화 되는 경우

## 해결책 - 명시적으로 컬럼명을 입력하기

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter19 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

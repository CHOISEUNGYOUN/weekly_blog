# Chapter 5 Keyless Entity (키가 없는 데이터 집합)

## 간결한 BD 아키텍쳐 구성하기

RDBMS 디자인은 각각의 테이블에 관계를 이어주는 것이라고 해도 과언이 아니다. 여기서 DB 설계와 운영 측면에서 항상 빠지지 않는 핵심요소는 참조 무결성(Referential Integrity)이다. 이 참조 무결성을 보장하기 위해서 RDBMS는 외래키로 테이블간에 연결을 맺을 수 있도록 제약 조건을 제공하고 있다.

하지만 몇몇 개발자들은 참조 무결성을 보장하는 제약조건을 사용하는 것을 선호하지 않는데, 이유는 다음과 같다.

* 제약조건 때문에 데이터 추가, 변경, 삭제 시 충돌이 발생하는 것 때문에 불편하다고 생각하는.
* 유연한 형태의 DB 디자인을 구성했기 때문에 참조 무결성을 100% 보장할 수 없는 경우.(NOSQL이 이에 해당.)
* DB에서 생성하는 외래키 인덱스가 DB 성능에 영향을 미친다고 생각하는 경우.
* 외래키를 지원하지 않는 DB를 사용. (예: Sqlite 레거시 버전, MySQL을 MyISAM 엔진으로 사용하는 경우.)
* 외래키를 사용할 줄 모르는 경우.

## 안티패턴 - 제약조건을 제외시키기.

외래키를 걸지않는 DB 디자인이 얼핏보면 더 간단하고 유연하면서도 성능이 좋아보이지만 그 이면에는 다른 문제가 도사리고 있다. 제약조건이 없기 때문에 데이터 무결성을 보장하기 위해서 개발자가 더욱 더 설계에 공을 들여야 한다는 문제이다.

### 완전무결한 코드의 필요성
많은 개발자들이 참조 무결성을 보장하기 위한 방법으로 데이터 관계성을 100% 만족하는 완벽한 코드를 어플리케이션 단에서 작성하는것을 제시한다. 열을 삭제/수정 할 때마다 자식 테이블에 제대로 반영되었는지 확인해야한다. 다시말해, 실수하지 마라는 것이다. 아래 예시는 제약조건이 없는 테이블에서 데이터를 추가/삭제하는 스크립트이다.

```sql
# 계정 조회
SELECT account_id FROM Accounts WHERE account_id = 1;

# 버그 테이블에 데이터 추가
INSERT INTO Bugs (reported_by) VALUES (1);

# 버그 조회
SELECT bug_id FROM Bugs WHERE reported_by = 1;

# 계정 삭제
DELETE FROM Accounts WHERE account_id = 1;
```

조건대로 순차적으로 실행된다면 아무런 문제가 없을 것이다. 하지만 이때 다른 요청이 들어와서 `account_id = 1` 을 참조하는 새로운 버그 데이터를 생성한다면 어떻게 될까? 아무런 제약 조건이 없기 때문에 등록 되어 버릴것이다. 이를 방지하기 위해선 위의 작업이 실행되는 동안 DB 는 잠겨 있어야 하는데, 이는 확장성과 동시성 측면에서 아주 안좋은 영향을 미친다.

### 데이터 오염 확인

제약 조건이 없는 경우 개발자가 직접 유효하지 않은 데이터를 제거하는 스크립트를 작성하고 주기적으로 실행시켜줘야 한다. 아래 예시는 `BugStatus` 테이블을 참조하는 `Bugs` 테이블의 상태값을 확인하는 쿼리이다.

```sql
SELECT b.bug_id, b.status
FROM Bugs b LEFT OUTER JOIN BugStatus s
  ON (b.status = s.status)
WHERE s.status IS NULL;
```

이처럼 모든 관계성을 지닌 테이블마다 이러한 쿼리 스크립트를 작성해서 주기적으로 실행시켜줘야 할 것이다. 거기에 만약 아래와 같은 데이터가 들어있다면 어떻게 할 것인가?
```sql
UPDATE Bugs SET status = DEFAULT WHERE status = 'BANANA';
```

이런 경우가 생기면 아무리 데이터 정제를 하는 스크립트가 잘 작성되어 있다고 하더라도 걸러내기 힘들 것이다.

### 내 실수가 아니야!

데이터베이스에 접근하는 모든 코드가 완벽하다는 것은 거의 있을 수 없는 일이다. 데이터베이스에 접근하는 다양한 함수들이 있을건데 이를 한번에 100% 충족시키는 코드를 작성한다는 것 또한 불가능에 가깝다. 특정 SQL문 때문에 참조 무결성이 깨질 수도 있다. 데이터베이스의 일관성을 유지하려면 제약조건이 있는것이 좋을지 없는것이 좋을지 한번쯤 생각 해볼 수 있다.

### 업데이트의 딜레마

많은 개발자들이 외래키 제약조건을 사용하는 것을 꺼리는데 이는 데이터 업데이트 하는 절차가 까다롭기 때문이다. 예를 들어 다른 열과 연결되어 있는 열을 삭제하려고 하면 다음과 같이 실행해야 한다.

```sql
DELETE FROM BugStatus WHERE status = 'BOGUS'; -- ERROR!

DELETE FROM Bugs WHERE status = 'BOGUS';

DELETE FROM BugStatus WHERE status = 'BOGUS'; -- retry succeeds
```

이처럼 데이터를 삭제하기 위해선 여러번의 스크립트를 실행해야 한다. 특히 문제가 되는것은 자식 열을 수정하려고 할 때 이다. 부모 열을 업데이트 하지 않으면 자식 열을 업데이트 할 수 없기 때문이다. 결국 수정을 하려면 부모 열 자식 열 모두 한번에 업데이트를 해야한다. 따로따로 스크립트를 실행하는것은 불가능하다. 이러한 제약 조건 때문에 몇몇의 개발자들은 외래키 제약조건을 사용하는 것을 포기한다.


## 어떻게 안티패턴을 구분하는가

* 한쪽의 테이블에는 존재하지만 다른 테이블에는 존재하지 않는 데이터를 조회하는 경우. <br>
(대게 부모와의 연결이 끊어진 열을 수정하거나 삭제하려는 경우이다.)
* 테이블에 삽입하면서 다른 테이블에 어떤 값이 있는지를 확인하는 빠른 방법이 있는지 고민하는 경우.<br>
(부모 값이 있는지 확인하기 위함임. 이런 경우 외래키로 연결하여 관계를 맺는 것이 가장 효율적이다.)
* 외래키는 데이터베이스 성능을 떨어뜨린다고 생각하는 경우.<br>
(위에 안티패턴에서 열거했듯이 이런 경우 외래키를 사용하지 않음으로써 성능을 더 저하시킨다.)

## 안티패턴이 허용되는 경우

* 외래키 제약조건이 없는 데이터베이스를 사용하는 경우.(MyISAM 엔진이나 SQLite 3.6.19 버전 이하)
* 유연한 형태의 모델링을 구현해야 하는 경우

## 해결책 - 제약조건 선언하기

제약조건을 설정함으로써 참조 무결성을 보장 할 수 있다. 이는 더 큰 문제를 일으키기 전에 사전에 예방하는 것과 같다.
```sql
CREATE TABLE Bugs (
  -- . . .
  reported_by BIGINT UNSIGNED NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'NEW',
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
  FOREIGN KEY (status) REFERENCES BugStatus(status)
);
```
이와 동시에 외래키 제약조건을 사용하면 참조 무결성을 보장하기 위해 부차적으로 스크립트를 작성하여 실행 시킨다던지 참조하는 테이블을 찾아 id를 직접 비교하는 등의 작업을 덜 수 있다.


### 여러 테이블 변경 지원

외래키를 사용하면 `cascading updates` 라는 기능 또한 사용 가능하다.

```sql
CREATE TABLE Bugs (
  -- . . .
  reported_by BIGINT UNSIGNED NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'NEW',
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
    ON UPDATE CASCADE
    ON DELETE RESTRICT,
  FOREIGN KEY (status) REFERENCES BugStatus(status)
    ON UPDATE CASCADE
    ON DELETE SET DEFAULT
);
```
이 기능을 사용하면 부모 테이블에서 수정/삭제가 일어날때 자식 테이블에도 이러한 업데이트를 함께 반영 시킬 수 있다.

## 결론
외래키를 사용하는 것이 좀 더 손이 많이 가는 작업이라고 오해하지만, 결과적으로 안티패턴 챕터에서 보았던 경우들을 생각해 본다면 이는 개발자의 시간을 더욱 더 아껴주는 것이라는 것을 알수 있다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter5 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

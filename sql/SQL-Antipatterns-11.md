# Chapter 11 31 Flavors (31가지 맛)

## 목표 - 특정 값들만 허용하도록 제한하기
컬럼값에 지정된 값들만 추가할 수 있도록 제약을 거는 것은 유용하다. 허용되지 않는 값들을 제한하면 컬럼에 들어있는 데이터를 사용하기도 훨씬 간편하다. 예를 들어 `Bug`라는 테이블에 `status` 컬럼에는 `NEW`, `IN PROGRESS`, `FIXED` 와 기타 등등의 정해진 데이터만 들어간다고 가정하자. 이 경우 우리는 각 `status` 에 따라 버그 데이터를 관리 할 수 있을 것이다. 또한 지정되지 않는 값들을 허용하지 않음으로써 관리측면에서 효율성도 가져다 줄 수 있다.

```sql
INSERT INTO Bugs (status) VALUES ('NEW'); -- OK
INSERT INTO Bugs (status) VALUES ('BANANA'); -- Error!
```

## 안티패턴 - 컬럼 선언에 특정 값만 사용 할 수 있도록 선언하기(ENUM)
많은 사람들이 특정 데이터 값들만 허용 시키기 위해 `CHECK` 함수를 사용하여 DDL에 다음과 같이 컬럼에 직접 제약조건을 선언한다.

```sql
CREATE TABLE Bugs (
  -- other columns
  status VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
);
```

위 조건과 비슷하게 MySQL에서는 `ENUM` 데이터 타입을 지원해준다.
```sql
CREATE TABLE Bugs (
  -- other columns
  status ENUM('NEW', 'IN PROGRESS', 'FIXED'),
);
```

위 DDL처럼 `ENUM` 함수를 사용하면 문자열 데이터를 인자로 넘겨주더라도 DB 내부에서는 서수(ordinal number)로 저장해준다. 이로 인해 DB 내에 저장되는 값은 가벼워진다. 또한 조회를 할 시 해당 값은 문자열 알파벳 순서가 아닌 DDL에 선언된 문자열의 순서대로 조회된다. 다른 방법으로 도메인이나 사용자 정의 타입을 사용하는 방법이 있다. 이 방법은 Postgresql에서 사용되는 방법으로 사용자가 미리 지정한 값만 허용하도록 제한 하는 방법이다. 하지만 많은 RDBMS에서 지원하는 시스템은 아니다.

어찌됐든 이 방법을 사용하면 선언시 추가한 문자열 이외에는 데이터를 추가 할 수 없게된다. 이러한 방식은 몇가지 단점이 있는데 그 이야기를 지금부터 하고자 한다.


### 중간값이 뭐였지?
현재 사용되고 있는 모든 상태값을 불러오기 위해선 많은 사람들은 보통 아래와 같은 쿼리를 실행 할 것이다.

```sql
SELECT DISTINCT status FROM Bugs;
```
만약에 `Bugs` 테이블에 상태가 `NEW`인 로우만 존재한다면 어떻게 될까? 이 경우 `NEW`만 리턴 할 것이다. 이처럼 `ENUM`을 사용하게 되면 현재 사용되고 있는 상태값은 오직 선언했던 DDL을 통해서만 확인이 가능하다.

```sql
SELECT column_type
FROM information_schema.columns
WHERE table_schema = 'bugtracker_schema' -- schema title
  AND table_name = 'bugs'
  AND column_name = 'status';
```

### 새로운 맛 추가하기
가장 일반적인 테이블 수정은 허용값을 추가하거나 수정하는 경우이다. `ENUM`의 경우 따로 추가하거나 제거할 수 있는 방법을 제공해주는 함수가 없다. 오직 테이블 메타데이터 수정을 통해서만 값 변경이 가능하다. 여기서 유의해야 할 점은 특정 DB 브랜드들은 이러한 메타데이터 수정을 빈 테이블이 아닌 경우 허용하지 않는다는 것이다. 이 경우 흔히 말하는 `ETL` 즉, 추출하고 변경해서 다시 불러와야한다. 이 작업은 복잡하면서도 많은 자원을 소모하게된다. 앞선 챕터에서 한번 언급했듯이 이렇게 되면 DB Locking 이 발생해서 실제 운영하는 어플리케이션에 직접적인 영향을 미치게 된다.


### 오래된 맛은 절대 사라지지 않는다
위 문제보다 더 큰 문제는 한번 선언한 `ENUM` 기본값은 삭제가 불가능하다. 위와 비슷하지만 더 큰 문제로써 한번 값을 변경하게 되면 기존에 등록된 모든 로우에 영향을 미치기 때문에 삭제하는 것은 거의 불가능하다.


### 마이그레이션이 어려움
허용값 제약조건, 도메인, 그리고 사용자 정의 타입은 각 SQL 브랜드마다 지원하는 방식이 다르다. 특히 `ENUM` 타입은 MySQL 에서만 메인으로 다루어지는 기능이다. 만약 다른 브랜드의 DB로 변경해야 되는 경우 해당 제약조건은 DB 마이그레이션을 더욱 힘들게 한다.

## 어떻게 안티패턴을 구분하는가
위에서 언급한 문제들은 모두 상태값이 가변적인 경우 발생한다. 만약 `ENUM`을 사용하고 싶다면 해당 값들이 한번 선언하면 변경되지 않는지 다시한번 생각해봐야 한다. 그렇지 않으면 아래와 같은 불상사가 발생한다.

* 새로운 상태값을 추가하기 위해선 항상 DB를 오프라인 상태에서 변경해야 되는 경우.<br>
  안티패턴이 발생하는 대표적인 사례이다. 이러한 변화가 생기더라도 서비스 중단이 일어나선 안된다.
* 상태값은 항상 이 값들 중 하나일 것이라고 근거없이 확신하는 경우.<br>
  근거없는 확신은 항상 예상하지 못한 문제를 불러일으킨다.
* 특정 리스트 값들이 특정 DB에서는 정확하게 일치하지 않는 경우.<br>
  여러 DB 브랜드를 섞어 사용하는 경우 충분히 발생 할 수 있다.

## 안티패턴 사용이 정당화 되는 경우
앞에서 잠깐 언급했듯이 `ENUM`으로 선언한 상태값들이 불변적인 경우 큰 문제를 야기시키진 않는다. 메타데이터를 확인하기 위해선 여전히 어려운 쿼리를 실행해야하지만 동일한 브랜드의 DB에서 불변속성(예: on/off) 들을 다루는 경우 사용해도 괜찮다.


## 해결책 - 데이터에 특정 값 제시하기
가장 안전한 해결책은 색인 테이블을 만들어 참조하는 것이다.

```sql
CREATE TABLE BugStatus (
  status VARCHAR(20) PRIMARY KEY
);

INSERT INTO BugStatus (status) VALUES ('NEW'), ('IN PROGRESS'), ('FIXED');

CREATE TABLE Bugs (
-- other columns
  status VARCHAR(20),
  FOREIGN KEY (status) REFERENCES BugStatus(status)
  ON UPDATE CASCADE
);
```
위와 같이 사용하는 경우 상태값은 항상 `BugStatus`에 존재하는 로우 값들만 사용이 가능하다. 이는 `ENUM`을 사용하여 선언하는 것과 동일한 효과를 냄과 동시에 상태값의 유연성 또한 확보할 수 있다.

### 허용값 조회하기
허용값을 조회하는 방법은 `ENUM`에서 DDL을 조회하는것과는 다르게 단순히 `BugStatus` 테이블을 조회함으로써 해결 할 수 있다.

```sql
SELECT status FROM BugStatus ORDER BY status;
```

### 색인 테이블 값 변경하기
색인 테이블 값을 추가하는 경우 단순히 새로운 로우만 색인 테이블에 추가하면 된다.
```sql
INSERT INTO BugStatus (status) VALUES ('DUPLICATE');
```

상태값을 삭제해야하는 경우 아래처럼 소프트 틸리트를 함으로써 손쉽게 해결 가능하다.
```sql
UPDATE BugStatus SET status = 'INVALID' WHERE status = 'BOGUS';
``` 

### 더이상 사용하지 않는 값 지원
색인값이 참조되고 있는 경우 삭제가 불가능하다. 하지만 위 경우처럼 삭제 대신 임의 값을 넣어 해결 할 수 있다.

### 마이그레이션의 용이함
`ENUM`이나 기타 다른 사용자 정의 타입과는 다르게 이 방법은 모든 DB 브랜드에서 공통으로 사용되는 방법이다. 따라서 DB 간 마이그레이션을 해야하는 경우 별다른 문제 없이 작업을 할 수 있다.


전반적으로 대규모 프로젝트를 진행해야하는 경우 특정 DB에서 지원하는 기능을 사용하기 보다는 공통으로 통용되는 방법을 사용하는 것을 권장하는 느낌을 받았다. 하지만 ORM을 사용하는 경우 프레임워크 수준에서 어느정도 호환성을 보장해주기 때문에 크게 문제되지 않는다고 생각한다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter11 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.
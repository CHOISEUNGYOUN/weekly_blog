# Chapter 6 Entity-Attribute-Value (엔티티-속성-값 모델)

## EAV 모델이란?
EAV(Entity-Attribute-Value) 모델은 효율적인 공간 활용을 위해 속성 단위(SQL에서는 행)로 데이터로 구성하는 모델링이다. 이 방식은 확장성 측면에서 장점을 가지고 있다. SQL에서 구현하면 새로운 형태의 데이터를 추기하기 위해 테이블에 행을 추가 할 필요없이 따로 구현된 속성 테이블에 데이터 타입과 값을 저장만 하면 된다는 이점을 가지고 있다. 이는 DB 유연성 측면에서 굉장한 이점을 보여준다. 하지만 잘못 사용하게 되면 SQL의 많은 장점을 활용하지 못하는 경우가 발생한다.

## 안티패턴 - 제네릭 속성 테이블 사용
예시로 `Issues`, `IssuesAttributes`라는 테이블을 EAV 스타일에 맞추어 구성해보았다.
```sql

CREATE TABLE Issues (
  issue_id SERIAL PRIMARY KEY
);

INSERT INTO Issues (issue_id) VALUES (1234);

CREATE TABLE IssueAttributes (
  issue_id BIGINT UNSIGNED NOT NULL,
  attr_name VARCHAR(100) NOT NULL,
  attr_value VARCHAR(100),
  PRIMARY KEY (issue_id, attr_name),
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);

INSERT INTO IssueAttributes (issue_id, attr_name, attr_value)
  VALUES
    (1234, 'product', '1'),
    (1234, 'date_reported', '2009-06-01'),
    (1234, 'status', 'NEW'),
    (1234, 'description', 'Saving does not work'),
    (1234, 'reported_by', 'Bill'),
    (1234, 'version_affected', '1.0'),
    (1234, 'severity', 'loss of functionality'),
    (1234, 'priority', 'high');
```

위 처럼 모델링을 하게 되면 다음과 같은 이점들을 기대 할 수 있다.
* 두 테이블 모두 컬럼 갯수를 최소한으로만 생성했다.
* 새로운 속성이 필요하더라도 행을 추가할 필요가 없다.
* 경우에 따라 필요없는 행이 생기는데, 이런 걱정을 할 필요가 없다.

이처럼 장점만 있는 것 같지만, SQL에서 사용하다보면 많은 문제가 발생한다.

## 요소 쿼리하기
예를들어 매일 건별로 버그리포트를 조회한다고 가정해보자. 평범한 테이블 구조에서는 아래와 같이 조회 할 수 있을 것이다.

```sql
SELECT issue_id, date_reported FROM Issues;
```
하지만, EAV 구조에서는 속성값 별로 조회해야하기 때문에 아래처럼 비효율이 발생한다.
```sql
SELECT issue_id, attr_value AS "date_reported"
FROM IssueAttributes
WHERE attr_name = 'date_reported';
```

## 데이터 무결성 보장의 어려움
EAV 방식은 데이터 무결성 관점에서도 많은 문제를 안고있다. 

### 필수 요소값 추가 불가능
예시의 `Issues` 테이블의 행 값들은 모두 `IssueAttributes` 테이블에 존재한다. SQL 자체에서 Issue 속성별로 필수값을 지정해서 사용 할 수 없다.

### SQL 데이터 타입 사용불가
위와 비슷한 문제이다. `IssueAttributes` 테이블의 요소로 관리되기 때문에 특정 데이터타입을 지정 할 수 없다. 종종 이런 문제를 해결하기위해 각 테이터타입에 해당되는 컬럼을 만들어서 사용하는 경우도 있다.

```sql
SELECT issue_id, COALESCE(attr_value_date, attr_value_datetime,
  attr_value_integer, attr_value_numeric, attr_value_float,
  attr_value_string, attr_value_text) AS "date_reported"
FROM IssueAttributes
WHERE attr_name = 'date_reported';
```

하지만 이런 방법은 오히려 사용하지 않는 컬럼 갯수만 늘려 쿼리를 더욱 어렵게 만든다.


### 참조 무결성 사용불가
기존 데이터베이스 모델이서는 외래키로 참조제약을 걸 수 있다. 하지만 EAV 모델링 방식에서는 사용할 수 없다. 이는 외래키가 테이블의 전체를 참조하여 제약조건을 부여하기 때문이다.

### 속성에 영구적인 이름을 부여할 수 없음
행이 존재하지 않는 형태이기 때문에 테이블 조회할때마다 컬럼명을 임의로 만들어줘야 한다.(`AS`를 매번 쿼리 할때마다 선언해줘야 함.) 이는 조회 할 때마다 행 이름이 바뀔 수 있다는 것이다.

### 행의 재구성
위와 마찬가지로 행이 존재하지 않기 때문에 행을 매번 테이블 조회할때마다 재구성해야한다. 아래와 같이 매번 `OUTRER JOIN`을 이용해서 각 조건에 맞는 속성값을 지정하여 테이블을 재구성 할 수 있다.

```sql
SELECT i.issue_id,
  i1.attr_value AS "date_reported",
  i2.attr_value AS "status",
  i3.attr_value AS "priority",
  i4.attr_value AS "description"
FROM Issues AS i
  LEFT OUTER JOIN IssueAttributes AS i1
    ON i.issue_id = i1.issue_id AND i1.attr_name = 'date_reported'
  LEFT OUTER JOIN IssueAttributes AS i2
    ON i.issue_id = i2.issue_id AND i2.attr_name = 'status'
  LEFT OUTER JOIN IssueAttributes AS i3
    ON i.issue_id = i3.issue_id AND i3.attr_name = 'priority';
  LEFT OUTER JOIN IssueAttributes AS i4
    ON i.issue_id = i4.issue_id AND i4.attr_name = 'description';
WHERE i.issue_id = 1234;
```
이 방식은 요소 속성이 늘어나면 늘어날 수록 더욱 더 많은 `OUTER JOIN`문이 필요하게 되기 때문에 복잡도가 올라간다.

작성중...

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter6 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

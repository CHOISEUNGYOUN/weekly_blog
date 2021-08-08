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

## 어떻게 안티패턴을 구분하는가
만약 이런 말들을 하는 동료 개발자가 있다면 EAV 안티패턴을 활용하고 있을 것이다.

* EAV 구조는 메타데이터 변경 없이도 확장성이 좋은 형태이기 때문에 종은 구조 <br>
SQL은 이런 형태의 데이터 구조를 제대로 지원하지 않기 때문에 반드시 이 방법이 옳은지 생각해보아야 한다.<br>
* 쿼리에서 JOIN이 최대 몇번까지 사용할 수 있는지 고민하고 있는 경우.<br>
JOIN을 최대치까지 사용하는 경우는 극히 드물다. EAV 패턴을 사용하고 있으면 어쩔 수 없이 수많은 JOIN을 할 수 밖에 없다.<br>
* 외부 소프트웨어 서비스를 사용하고 있는데 EAV 패턴을 사용하고 있어 어려움을 겪는 경우<br>
많은 턴키 프로젝트들은 커스터마이징의 용이성 때문에 EAV 패턴을 많이 사용하고 있다. 이는 해당 툴을 사용하기 위해 복잡하고 실용성 없는 쿼리를 사용하게끔 강요한다.

## 안티패턴이 허용되는 경우
SQL에서 EAV 패턴 사용을 권장하는 경우는 드물다. 그래도 필요한다면 제한적으로 사용하는 것이 좋다. 진짜 필요하다면 비관계형 데이터베이스 도입을 진지하게 고려해보아야 한다. 

## 대안 - 제약조건 선언하기

### 단일 테이블 상속
가장 간단한 방법은 관련된 모든 타입을 하나의 테이블에 선언하는 것이다.
```sql
CREATE TABLE Issues (
  issue_id SERIAL PRIMARY KEY,
  reported_by BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED,
  priority VARCHAR(20),
  version_resolved VARCHAR(20),
  status VARCHAR(20),
  issue_type VARCHAR(10), -- BUG or FEATURE
  severity VARCHAR(20), -- only for bugs
  version_affected VARCHAR(20), -- only for bugs
  sponsor VARCHAR(50), -- only for feature requests
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
  FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

위처럼 간단하게 설계 하면 값이 있는 경우 해당 값을 넣고 없는 경우 `NULL` 처리하면 된다. 하지만 새로운 타입이 필요할 때 마다 행을 추가해야 한다는 점, 테이블 당 행 갯수 제한이 있다는 문제가 있다. 그리고 어떤 서브타입 보유에 대한 기준을 가지고 있는 메타데이터가 없기 때문에 조회 할 시 구분하기 힘들다. 이런 형태의 모델링은 서브타입의 종류가 적고 그에 따른 속성이 적은 경우에 유용하다. 그리고 사용 할 시 `Active Record` 패턴으로 테이블에 접근하는 것이 필요하다.

### 구상 테이블 상속
또 다른 방법으로 서브타입별로 테이블을 나누는 것이다.

```sql
CREATE TABLE Bugs (
  issue_id SERIAL PRIMARY KEY,
  reported_by BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED,
  priority VARCHAR(20),
  version_resolved VARCHAR(20),
  status VARCHAR(20),
  severity VARCHAR(20), -- only for bugs
  version_affected VARCHAR(20), -- only for bugs
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
  FOREIGN KEY (product_id) REFERENCES Products(product_id)
);

CREATE TABLE FeatureRequests (
  issue_id SERIAL PRIMARY KEY,
  reported_by BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED,
  priority VARCHAR(20),
  version_resolved VARCHAR(20),
  status VARCHAR(20),
  sponsor VARCHAR(50), -- only for feature requests
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
  FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```
이런 방식은 각 서브타입 테이블 별로 필요한 행만 추가해서 관리 할 수 있다. 또 다른 장점으로는 서브타입이 추가 될 때마다 에티블만 추가하면 되기에 위에 예시로 들었던 단일 테이블 상속처럼 행을 일일이 추가할 필요도 없다.

하지만 이 방식에도 문제가 있다. 먼저 서브타입 테이블들을 관리 해 줄 수 있는 공통 테이블이 없다. 공통 요소들이 추가되면 모든 서브타입 테이블에도 행을 추가해야한다. 마지막으로 서브타입에 상관없이 테이블을 한꺼번에 조회하기도 까다롭다. 만약 조회하려고 시도한다면 다음과 같은 쿼리를 해야한다.

```sql
CREATE VIEW Issues AS
  SELECT b.*, 'bug' AS issue_type
  FROM Bugs AS b
    UNION ALL
  SELECT f.*, 'feature' AS issue_type
  FROM FeatureRequests AS f;
```

이 구조는 모든 서브타입을 한번에 조회하는 경우가 드문 경우 효율적이다.

### 클래스 테이블 상속
이 방법은 객체지향 스타일로 테이블들을 구성하는 방식이다.
```sql
CREATE TABLE Issues (
  issue_id SERIAL PRIMARY KEY,
  reported_by BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED,
  priority VARCHAR(20),
  version_resolved VARCHAR(20),
  status VARCHAR(20),
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
  FOREIGN KEY (product_id) REFERENCES Products(product_id)
);

CREATE TABLE Bugs (
  issue_id BIGINT UNSIGNED PRIMARY KEY,
  severity VARCHAR(20),
  version_affected VARCHAR(20),
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);

CREATE TABLE FeatureRequests (
  issue_id BIGINT UNSIGNED PRIMARY KEY,
  sponsor VARCHAR(50),
  FOREIGN KEY (issue_id) REFERENCES Issues(issue_id)
);
```
위 예시처럼 기초가 되는 테이블을 만든 다음 공통 요소들을 선언한 뒤, 각각 속성별로 필요한 요소들을 담은 테이블들을 만들어 외래키로 제약조건을 부여하는 방식이다. 이러한 일대일 관계는 메타데이터 기반으로 연결되고 테이블의 열은 반드시 유일해야 하기 때문에 부모 테이블의 ID가 고유키가 된다. 이 방식은 모든 서브타입들을 한번에 조회하기 용이하다.

```sql
SELECT i.*, b.*, f.*
FROM Issues AS i
  LEFT OUTER JOIN Bugs AS b USING (issue_id)
  LEFT OUTER JOIN FeatureRequests AS f USING (issue_id);
```

그리고 기초가 되는 테이블의 열 또한 일대일 관계로 고유하기 때문에 어떤 서브타입을 의미하는지 알 필요가 없다. 이 디자인은 모든 서브타입들을 한번에 조회해야 할 때 유용하다.

### 반정형 데이터
일정하지 않은 형태의 데이터 타입을 지원해야한다면 `BLOB` 행을 추가하여 관리하는 것도 방법이다.
```sql
CREATE TABLE Issues (
  issue_id SERIAL PRIMARY KEY,
  reported_by BIGINT UNSIGNED NOT NULL,
  product_id BIGINT UNSIGNED,
  priority VARCHAR(20),
  version_resolved VARCHAR(20),
  status VARCHAR(20),
  issue_type VARCHAR(10), -- BUG or FEATURE
  attributes TEXT NOT NULL, -- all dynamic attributes for the row
  FOREIGN KEY (reported_by) REFERENCES Accounts(account_id),
  FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```
`JSON`이나 `XML`과 같은 데이터를 저장할 때 용이하다. 하지만 이런 형식의 데이터는 SQL에서 지원되는 기능이 제한적이기 때문에 어플리케이션에서 추가적인 데이터 처리를 하여 사용하는것이 좋다.

## 결론
EAV 형식은 얼핏 보았을 때 document나 graph 형식의 데이터 구조로 생각된다. 이런 방식은 SQL에서 지원하는 기능이 제한적이기 때문에 필요하다면 다른 DB를 도입하는 것이 더 옳다고 생각한다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter6 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

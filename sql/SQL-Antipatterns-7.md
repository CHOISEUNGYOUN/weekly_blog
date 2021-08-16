# Chapter 7 Polymorphic Associations (다형성 관계 테이블)

아래의 DDL 처럼 한 컬럼에 두개의 테이블을 동시에 참조하는 방식은 SQL에서 지원하지 않는다.

```sql
CREATE TABLE Comments (
  comment_id SERIAL PRIMARY KEY,
  bug_id BIGINT UNSIGNED NOT NULL,
  author_id BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME NOT NULL,
  comment TEXT NOT NULL,
  FOREIGN KEY (author_id) REFERENCES Accounts(account_id),
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
  FOREIGN KEY (issue_id) REFERENCES Bugs(issue_id) OR FeatureRequests(issue_id)
);
```

## 목표 - 여러 부모 테이블 참조하기

```sql
CREATE TABLE Comments (
  comment_id SERIAL PRIMARY KEY,
  issue_type VARCHAR(20), -- "Bugs" or "FeatureRequests"
  issue_id BIGINT UNSIGNED NOT NULL,
  author BIGINT UNSIGNED NOT NULL,
  comment_date DATETIME,
  comment TEXT,
  FOREIGN KEY (author) REFERENCES Accounts(account_id)
);
```

위 예시는 `Comments` 테이블의 `issue_type`에 따라 `Bugs` 테이블을 참조할 것인지 `FeatureRequests` 테이블을 참조할 것인지 지정하는 것을 목표로 한다. 이러한 관계를 다형성 관계라고 칭한다.

![img](imgs/SQL-Antipatterns-7_1.png)


## 안티패턴 - 이중목적 외래키 사용
다형성 관계를 만들기 위해선 `issue_id`라는 컬럼을 추가해야한다. 이 컬럼의 값은 예시에서 언급된 `Bugs` 또는 `FeatureRequests` 중 어느 테이블을 참조 할 것인지 결정한다. 여기서 문제점은 `issue_id`에 외래키 선언이 빠져있다는 점이다. 앞선 쳅터에서 설명했듯이 외래키 제약조건은 반드시 다른 테이블을 지정해서 참조해야하기 때문에 참조 무결성을 보장하지 못한다.

### 다형성 관계 테이블 조회하기
`Comments` 테이블을 `issue_id`와 `issue_type`에 따라 구분되고 있기 때문에 각기 다른 부모 테이블의 코멘트를 조회하려면 `issue_type`에 부여한 값을 정확하게 지정하고 조회해야만 한다.

```sql
SELECT *
FROM Bugs AS b JOIN Comments AS c
  ON (b.issue_id = c.issue_id AND c.issue_type = 'Bugs')
WHERE b.issue_id = 1234;
```

위 코드가 잘 동작하는것 처럼 보이지만 `Bugs` 테이블과 `FeatureRequests` 테이블을 함께 조회하려는 경우 문제가 발생한다.

```sql
SELECT *
FROM Comments AS c
  LEFT OUTER JOIN Bugs AS b
    ON (b.issue_id = c.issue_id AND c.issue_type = 'Bugs')
  LEFT OUTER JOIN FeatureRequests AS f
    ON (f.issue_id = c.issue_id AND c.issue_type = 'FeatureRequests');
```

위 쿼리의 결과값은 다음과 같다.

|c.comment_id|c.issue_type|c.issue_id|c.comment|b.issue_id|f.issue_id|
|------------|------------|----------|---------|----------|----------|
|6789|Bugs|1234|It crashes!|1234|NULL|
|9876|Feature...|2345|Great idea!|NULL|2345|

위처럼 컬럼을 추가하여 결과값을 도출해야하는데 보다시피 매칭되지않는 로우 데이터는 `NULL`을 반환하고 있다.


### 객체지향 스타일이 아닌 경우
이 예제에서 `Bugs`와 `FeatureRequests` 테이블 모두 모델과 연관된 서브타입이다. 다형성 관계 모델은 각 부모 테이블이 완전히 관계가 없는 경우 사용될 수도 있다. 예를 들어 이커머스 데이터베이스에서는 `Users` 테이블과 `Orders` 테이블이 `Addresses` 테이블과 관계를 형성하고 있다.

```sql
CREATE TABLE Addresses (
  address_id SERIAL PRIMARY KEY,
  parent VARCHAR(20), -- "Users" or "Orders"
  parent_id BIGINT UNSIGNED NOT NULL,
  address TEXT
);
```

이런 경우 `Addresses` 테이블은 다형성 관계를 가지고 있다. 여기에 `Billings` 와 `Shippings` 에 대한 다형성 관계 또한 추가하게 된다면 아래와 같은 모습이 될 것이다.

```sql
CREATE TABLE Addresses (
  address_id SERIAL PRIMARY KEY,
  parent VARCHAR(20), -- "Users" or "Orders"
  parent_id BIGINT UNSIGNED NOT NULL,
  users_usage VARCHAR(20), -- "billing" or "shipping"
  orders_usage VARCHAR(20), -- "billing" or "shipping"
  address TEXT
);
```
이런 경우 `Addresses` 테이블의 복잡도가 한없이 높아지게 된다.



> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter7 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

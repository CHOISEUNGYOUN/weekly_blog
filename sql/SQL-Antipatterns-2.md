## Chapter 2 Jaywalking

### Jaywalking 이란?
SQL 모델링에서 ,(콤마)로 분리해서 배열 형식으로 데이터를 저장하는 방식을 Jaywalking으로 말하며 일종의 안티패턴으로 취급한다.
그에 대한 예제는 아래와 같다.

```sql
CREATE TABLE Products (
    product_id SERIAL PRIMARY KEY,
    product_name VARCHAR(1000),
    account_id VARCHAR(100), -- comma-separated list
    -- . . .
);

INSERT INTO Products (product_id, product_name, account_id)
VALUES (DEFAULT, 'Visual TurboBuilder', '12,34');
```

위 예제처럼 테이블을 구성하고 데이터를 관리하면 한 테이블에서 다른 테이블의 여러 id들을 저장하고 관리 할 수 있기 때문에 편리하고 성능이 뛰어 날 것이라고 착각하곤 한다. 하지만 이는 사실이 아니다. 그에 대한 이유는 다음과 같다.

1. 단순한 비교연산자로 필요한 데이터를 불러 올 수 없음.
이런 방식은 아래 예시처럼 복잡한 정규표현식으로 비교 연산을 해야한다.
```sql
SELECT * FROM Products WHERE account_id REGEXP '[[:<:]]12[[:>:]]';
```
지금은 단순해 보이지만 여러 id를 가져와야 하거나 `JOIN` 을 해야된다면 더욱 복잡해진다.
```sql
SELECT * FROM Products AS p JOIN Accounts AS a
    ON p.account_id REGEXP '[[:<:]]' || a.account_id || '[[:>:]]'
WHERE p.product_id = 123;
```
그리고 SQL 프로그램(예: MySQL, MSSQL, PostgreSQL 등) 마다 지원하는 정규표현식 방식도 다르니 구현하는 방식도 매번 달라져야한다.

2. 특정 id값을 업데이트 하는 경우 중복 쿼리를 작성해야하고 id 를 순차적으로 저장 할 수 없음.
예시의 `account_id` 컬럼에 새로운 id 를 저장하기 위해선 기존의 데이터를 불러 온 뒤, 거기에 구분자와 함께 새로운 id를 추가하여 저장해야 한다. 삭제의 경우 더욱 복잡한 경우로 구분자 기준으로 분리 한 뒤 모든 값을 조회 한 뒤 해당되는 값을 삭제하고 구분자로 합쳐 다시 저장해야한다. 
```sql
UPDATE Products
    SET account_id = account_id || ',' || 56
WHERE product_id = 123;
```

```php
<?php
$stmt = $pdo->query(
  "SELECT account_id FROM Products WHERE product_id = 123");
$row = $stmt->fetch();
$contact_list = $row['account_id'];

// change list in PHP code
$value_to_remove = "34";
$contact_list = split(",", $contact_list);
$key_to_remove = array_search($value_to_remove, $contact_list);
unset($contact_list[$key_to_remove]);
$contact_list = join(",", $contact_list);
$stmt = $pdo->prepare(
  "UPDATE Products SET account_id = ?
   WHERE product_id = 123");
$stmt->execute(array($contact_list));
```
3. `VARCHAR` 타입이기 때문에 문자열 데이터를 숫자로 형변환 하여 검증해야함.
아래 예시처럼 불특정 문자열이 들어오는 것을 방지하기 위한 방어 로직을 반드시 세워야 한다.
```sql
INSERT INTO Products (product_id, product_name, account_id)
VALUES (DEFAULT, 'Visual TurboBuilder', '12,34,banana');
```
4. 선언한 `VARCHAR` 의 길이 제한에 따라 저장 할 수 있는 id 수량이 정해진다.
만약에 `account_id` 컬럼에 `VARCHAR(30)` 으로 하게된다면 아래와 같은 불상사가 발생 할 것이다.
```sql
UPDATE Products SET account_id = '10,14,18,22,26,30,34,38,42,46'
WHERE product_id = 123;
UPDATE Products SET account_id = '101418,222630,343842,467790'
WHERE product_id = 123;
```
이렇게 id 길이마다 저장 할 수 있는 id 수량이 정해지는것은 정말 어처구니 없을 것이다.

5. SQL에서 기본적으로 지원해주는 통계 쿼리를 사용하기 어렵다.
`COUNT`, `SUM`, `AVG` 같은 통계 쿼리는 기본적으로 row 기반으로 동작하기 때문에 이상한 꼼수(?)를 사용해서 통계 계산을 해야한다.
```sql
SELECT product_id, LENGTH(account_id) - LENGTH(REPLACE(account_id, ',', '')) + 1
    AS contacts_per_product
FROM Products;
```

6. 구분자가 포함된 데이터가 저장 될 수도 있다.
기본값으로 ,(콤마)를 구분자로 사용하는데, 만약 콤마가 포함된 데이터가 들어오면 어떻게 할 것인가? 이런 경우 정규표현식으로도 완벽하게 걸러 낼 수 없을 것이다. 보통 콤마가 아닌 다른 특수문자를 구분자로 사용하긴 한다. (대표적으로 세미콜론(;)) 그렇다고 해서 무조건 사용자정의 구분자가 저장되지 않으라는 법은 없지 않는가?

### 최선책
가장 좋은 방법은 정규화를 통해 중계 테이블을 생성하여 관리하는 것이다. 이런 방식을 사용하면, 기본적으로 SQL에서 지원하는 쿼리 방식을 온전히 사용 할 수도 있고, 데이터 무결성도 지킬 수 있다.
```sql
CREATE TABLE Contacts (
    product_id BIGINT UNSIGNED NOT NULL,
    account_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (product_id, account_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id),
    FOREIGN KEY (account_id) REFERENCES Accounts(account_id)
);
INSERT INTO Contacts (product_id, accont_id)
VALUES (123, 12), (123, 34), (345, 23), (567, 12), (567, 34);
```

하지만 정규화가 항상 옳은 것은 아니다. 예시에 나온 `product` 테이블 처럼 무한정으로 제품 라인업이 늘어나는 것이 아니라 제품군 처럼 한정된 카테고리, 개별 아이템을 필수적으로 조회 해야 하는 것이 아닌 경우에는 성능상의 효율성을 얻기 위해 구분자로 구분된 데이터를 저장 할 수 있다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter2 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.
> 
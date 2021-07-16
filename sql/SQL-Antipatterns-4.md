# Chapter 4 ID Required

## 고유키(Primary Key)

고유키는 전통적으로 테이블을 구성할 때 가장 중요한 요소 중 하나이다. 테이블 데이터의 단일성을 보장해줌과 동시에 다른 테이블과 연걸할 때 사용하는 외래키로써 사용되기 때문이다.

종종 개발자들이 고유키에 이메일이나 전화번호, 주민등록번호등을 사용하는데, 이는 잘못된 행위이다. 이들 데이터 유형은 100% 중복되지 않을거란 보장이 없기 때문이다. 그렇다면 어떤 값을 고유키로 사용해야될까? RDBMS에서는 가상키(id)를 제공함으로써 이러한 니즈를 충족시켜준다. 가상키를 생성하는 기능은 각 벤더사마다 다음과 같이 제공해주고 있다.

|기능|지원 데이터베이스|
|---|------------|
|AUTO_INCREMENT|MySQL|
|GENERATOR|Firebird, InterBase|
|IDENTITY|DB2, Derby, Microsoft SQL Server, Sybase|
|ROWID|SQLite|
|SEQUENCE|DB2, Firebird, Informix, Ingres, Oracle , PostgreSQL|
|SERIAL|MySQL, PostgreSQL|

## 안티패턴
책, 기술 아티클, 각종 프레임워크에서 고유키는 다음과 같이 정의하고 있다.
1. 고유키 컬럼 이름은 `id`로 칭한다.
2. 데이터 타입은 32비트 또는 64비트 정수이다.
3. 고유값은 자동 생성된다.

SQL로 스크립트를 작성하면 다음과 같다.
```sql
CREATE TABLE Bugs (
    id SERIAL PRIMARY KEY,
    description VARCHAR(1000),
    -- . . .
);
```
하지만 이런 컨벤션을 따르는 경우 몇가지 문제가 발생한다.

### 필요 없는 고유키를 생성할 수도 있다.

```sql
CREATE TABLE Bugs (
    id SERIAL PRIMARY KEY,
    bug_id VARCHAR(10) UNIQUE,
    description VARCHAR(1000),
-- . . .
);
INSERT INTO Bugs (bug_id, description, ...)
  VALUES ('VIS-078', 'crashes on save', ...);
```
위 예시처럼 버그리포트에 약속된 고유키 규칙이 있다면 id는 그저 쓸모없는 데이터가 되어버린다.

### 중복열 생성에 대한 위험
중계 테이블에 id를 고유키로 지정하는 경우 중복된 데이터가 생성될 수도 있음.

```sql
CREATE TABLE BugsProducts (
    id SERIAL PRIMARY KEY,
    bug_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
INSERT INTO BugsProducts (bug_id, product_id)
    VALUES (1234, 1), (1234, 1), (1234, 1); -- 중복값 허용.
```
중복값을 허용하지 않는다면 외래키를 복합키로 걸어 중복 방지를 하는것이 옳다. 이 경우 id는 필요없는 정보가 된다.

```sql
CREATE TABLE BugsProducts (
    id SERIAL PRIMARY KEY,
    bug_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    UNIQUE KEY (bug_id, product_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);
```

### id 라는 컬럼명에서 오는 모호함
모든 테이블에서 id를 고유키로 지정하다 보면 간혹 테이블을 `JOIN` 할 시, 혼동이 발생할 수도 있다. 이런 경우를 방지하기 위해서는 id를 의미있는 명칭으로 부여하는 것이 좋다고 한다.
> __요악을 하고있는 나는 개인적으로 반대한다. 특히나 ORM을 사용한다면 더더욱 이런 걱정을 할 필요 없다고 생각한다.__

```sql
SELECT b.id, a.id
FROM Bugs b
JOIN Accounts a ON (b.assigned_to = a.id)
WHERE b.status = 'OPEN';
```

### `USING` 쿼리 옵션을 사용하지 못함
`USING` 쿼리 옵션은 `JOIN` 절에서 `ON` 대신 비교를 수행할 수 있는 토큰이다.
```sql
SELECT * FROM Bugs JOIN BugsProducts USING (bug_id);
```
만약 테이블에 `bug_id` 로 고유키를 명시한 것이 아니라 일반 `id` 를 사용하고 있다면 다음과 같이 조금은(?) 비효율적으로 캐스팅 해야한다.
```sql
SELECT * FROM Bugs AS b JOIN BugsProducts AS bp ON (b.id = bp.bug_id);
```

### 예외케이스
특정 ORM은 설정보다 규칙을 중요시한다. 즉, 룰에 맞추어 사용하는 것을 권장한다. 이런 경우 대부분 `id`를 고유키로 사용하고 있기 때문에 해당 프레임워크의 규칙을 따르는것이 좋다.

## 대안

### 고유키에 의미있는 명칭을 부여하자
아래 예시의 `Bugs` 테이블에 있는 `bug_id` 처럼 의미있는 고유키를 부여 하면 좀 더 가독성이 좋다.

```sql
CREATE TABLE Bugs (
    bug_id BIGINT UNSIGNED NOT NULL,
    -- . . .
    reported_by BIGINT UNSIGNED NOT NULL,
    FOREIGN KEY (reported_by) REFERENCES Accounts(account_id)
);
```

### ORM에서 가상키를 직접 설정하자
ORM 자체적으로 자동 생성되는 가상키에 위의 규칙을 적용할 수도 있다. 아래 예시는 루비온레일즈에서 가상키를 직접 설정하는 방법이다.
```rb
class Bug < ActiveRecord::Base
  set_primary_key "bug_id"
end
```

### 중계테이블에선 복합키를 사용하자
대부분의 경우 중계테이블에 가상키를 생성할 이유가 없다. 대신 외래키에 복합키 설정을 하면 중복열 생성을 방지 할 수 있다.

```sql
CREATE TABLE BugsProducts (
    bug_id BIGINT UNSIGNED NOT NULL,
    product_id BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (bug_id, product_id),
    FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id),
    FOREIGN KEY (product_id) REFERENCES Products(product_id)
);

INSERT INTO BugsProducts (bug_id, product_id)
  VALUES (1234, 1), (1234, 2), (1234, 3);

INSERT INTO BugsProducts (bug_id, product_id)
  VALUES (1234, 1); -- error: duplicate entry
```

### 결론
사실 이번 챕터는 정리하면서 의문이 가는 포인트들이 있었다. 아무래도 보편적인 SQL을 설명하려다 보니 이런 내용들이 나온것이라고 이해하고 있다. 일반적인 백엔드를 구축한다고 생각했을때, SQL을 직접 구성하는 것 보다 ORM을 사용하여 규칙을 지키면 이런 문제들이 좀 덜 하지 않을까 하는 생각이 든다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter4 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.
> 
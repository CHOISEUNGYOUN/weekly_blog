# Chapter 17 Poor Man’s Search Engine (미련한 사람의 검색 엔진)

## 목표 - Full-text 검색
텍스트를 저장하고 있는 모든 어플리케이션은 텍스트 내 단어 또는 구절 단위로 검색을 하는 기능이 필요하다. 요즘엔 더더욱 텍스트 데이터를 활용하는 경우가 많으며 이와 동시에 원하는 패턴을 더 빨리 찾을 수 있도록 요구되는 추세이다. 특히 웹 애플리케이션의 경우 고성능, 확장성이 둘다 만족하는 솔루션을 제공해야한다.

SQL의 가장 기본적인 원칙은 컬럼내 존재하는 컬럼이 원자적이라는 것이다. 이는 즉, 각기 다른 값들을 비교 할 수 있지만 해당 조건을 수행하려면 컬럼 내 모든 값들을 비교해야 한다는 것이다. 결국 부분 문자열을 SQL에서 비교하는 것은 비효율적이거나 부정확할 수 밖에 없다.

그럼에도 불구하고 부분 문자열을 긴 문자열에서 비교하고 조건에 일치하는 긴 문자열이 있는지 확인하는 기능은 반드시 필요하다. SQL에서는 이를 어떻게 해결해야될까?

## 안티패턴 - 술어문을 사용하여 패턴 매칭하기
SQL에서는 문자열 술어문 패턴 매칭을 제공하고 있으며 이는 대부분의 개발자들이 가장 먼저 생각하고 적용해보는 방법이다. 이 조건을 수행해주는 키워드가 바로 `LIKE`이다. `LIKE` 는 와일드카드인 `%` 를 이용하여 문자열을 비교 탐색한다.

```sql
SELECT * FROM Bugs WHERE description LIKE '%crash%';
```
SQL 표준은 아니지만 정규표현식 또한 많은 데이터베이스에서 지원해주고 있다. 정규표현식은 기본적으로 패턴 매칭된 문자열을 찾아 반환해주기 때문에 와일드카드가 필요하지 않는다.

```sql
SELECT * FROM Bugs WHERE description REGEXP 'crash';
```

이러한 패턴 매칭 방식은 굉장히 효율적이지 못한 성능을 가져다준다. 이 방식은 일반적으로 사용되는 인덱스의 이점을 누릴수 없기 때문에 각 로우별로 비교대조 작업이 이뤄져야한다. 문자열 컬럼을 비교대조 하는 작업은 굉장히 많은 자원을 소모하기 때문에 전체 테이블을 검색하여 원하는 로우를 추출하는 작업은 굉장히 비싼 작업이 된다.

이러한 패턴 매칭의 또 다른 문제점은 의도하지 않은 결과값이 나올수도 있다는 점이다.

```sql
SELECT * FROM Bugs WHERE description LIKE '%one%';
```
위 예시는 `one` 이라는 단어를 포함한 텍스트를 추출하기 위해서 작성되었지만 의도와는 다르게 `money`, `prone`, `lonely`와 같이 의도되지 않은 단어들도 함께 매칭하여 반환 시킬 것이다. `LIKE`의 와일드카드로는 이런 문제를 해결 할 순 없지만 정규표현식에서는 `word boundary` 를 지정하여 이러한 문제를 해결 할 수 있다.

```sql
SELECT * FROM Bugs WHERE description REGEXP '[[:<:]]one[[:>:]]';
```

이런 방식을 통해 원하는 패턴만 매칭하도록 지정할 순 있지만 성능상으로는 아주 나쁜 선택이 된다.

## 어떻게 안티패턴을 구분하는가
* `LIKE` 를 사용하는 표현식의 와일드카드 사이에 변수를 지정하는 것을 고민하는 경우.<br>
개발자가 패턴 매칭을 활용한 검색 기능을 고민하는 경우 흔히 볼 수 있는 모습이다.
* 정규표현식을 활용하여 여러 단어를 한꺼번에 찾지만 특정 단어를 포함시키지 않거나 특정 형태의 단어만 찾도록 고민하는 경우<br>
* 몇가지 데이터만 추가되었을 뿐인데 우리 앱의 검색기능이 현저하게 느려진 경우<br>
데이터의 크기가 커지면 커질수록 패턴 매칭 방식은 현저한 성능저하를 일으킨다.

## 안티패턴 사용이 정당화 되는 경우
여기서 언급한 안티패턴은 SQL에서 정상적으로 동작하는 쿼리이다. 또한 이는 직관적이기 때문에 사용하기가 쉽다. 이는 충분히 고려해볼만한 점이다.

성능이 물론 중요하긴 하지만 자주 실행되지 않는 쿼리를 최적화 하기 위해 많은 자원을 투입하여 개발하는 것 또한 좋은 방법은 아니다. 자주 사용되지 않는 쿼리를 최적화 하기 위해 인덱스를 활용하여 기능 구현 하는 것은 비효율적인 쿼리를 실행하는 것 만큼 비효율적인 작업이다. 또한 특별한 목적을 위해 이런 쿼리를 작성한다면 인덱스를 활용하는 것이 장점을 가져다 줄지 또한 생각해보아야 한다.

패턴 매칭 연산을 사용하면 복잡한 쿼리 작성을 하기 어렵긴 하지만 간단한 경우라면 별 힘들이지도 않고도 기능을 구현할 수 있다.

## 해결책 - 각 상황에 맞는 툴을 사용하기
검색을 최적화하고 싶다면 검색 최적화를 도와주는 도구를 사용하는것이 가장 올바른 방법이다. 여기서 몇가지 툴들을 언급하고자 한다.

### 각 DB별 확장 기능
대부분의 데이터베이스에서 검색기능을 활용하기 위한 특별한 확장 기능들을 제공하고 있다. 본인이 사용하고 있는 데이터베이스 브랜드에 맞추어 사용하면 검색 기능을 최적화 할 수 있다.

#### MySQL: Full-text 인덱스
MySQL에서는 MyISAM 엔진을 사용하는 경우 full-text 인덱스 기능을 사용하여 최적화 할 수 있다. 이 기능은 `CHAR`, `VARCHAR`, `TEXT`와 같은 문자열 형태의 컬럼에서만 사용 할 수 있다.

```sql
ALTER TABLE Bugs ADD FULLTEXT INDEX bugfts (summary, description);
```
이렇게 지정한 인덱스를 `MATCH`를 활용하여 패턴 매칭을 할 수 있다. 이 때 반드시 `FULLTEXT` 인덱스가 생성된 컬럼을 지정해야지만 사용이 가능하다.

```sql
SELECT * FROM Bugs WHERE MATCH(summary, description) AGAINST ('crash');
```

MySQL 4.1 이상 버전에서는 여기에서 한발 더 나아가서 불리언 형식의 패턴 매칭을 활용 할 수 있다.
```sql
SELECT * FROM Bugs WHERE MATCH(summary, description)
AGAINST ('+crash -save' IN BOOLEAN MODE);
```

#### PostgreSQL: Text Search
PostgreSQL 8.3 에서는 텍스트를 검색 가능한 어휘 단위로 변환하고 이를 패턴 매칭 시켜주는 기능을 제공한다. 이런 기능을 활용하고 싶다면 기존 텍스트 컬럼 이외에도 `TSVECTOR` 데이터 타입을 설정하여 저장해야 한다. 이 경우 `TSVECTOR` 컬럼을 기존 텍스트 컬럼과 항상 동기화 시켜줘야한다. PostgreSQL에서는 이를 처리하기 위한 빌트인 트리거 프로시져를 제공한다.
```sql
CREATE TRIGGER ts_bugtext BEFORE INSERT OR UPDATE ON Bugs
FOR EACH ROW EXECUTE PROCEDURE
  tsvector_update_trigger(ts_bugtext, 'pg_catalog.english', summary, description);
```
반드시 표준화된 역 인덱스(GIN)를 `TSVECTOR` 컬럼에 생성해야함을 잊으면 안된다.

```sql
CREATE INDEX bugs_ts ON Bugs USING GIN(ts_bugtext);
```

위 작업을 거치고나면 `@@` 토큰을 활용하여 문자열 패턴 매칭을 통한 검색을 할 수 있다.
```sql
SELECT * FROM Bugs WHERE ts_bugtext @@ to_tsquery('crash');
```

#### SQLite: Full-Text Search (FTS)
표준 SQLite는 효율적인 full-text 검색 기능을 지원하지 않지만 선택 확장기능을 활용하면 가상 테이블을 활용하여 문자열 검색을 할 수 있다. 해당 익스텐션은 FTS1, FTS2, FTS3 세가지 버전이 있다.

FTS 확장 기능의 경우 SQLite의 기본 내장기능은 아니지만 필요에 따라 세가지 버전 중 한가지를 활성화하여 사용 할 수 있다.
```sql
TCC += -DSQLITE_CORE=1
TCC += -DSQLITE_ENABLE_FTS3=1
```
위와 같이 FTS 기능을 활성화 한다면 검색 기능을 활용 할 수 있는 가상 테이블을 생성할 수 있다. 이때 어떠한 데이터타입, 제약조건, 컬럼에 부여된 옵션값들이 무시된다.

```sql
CREATE VIRTUAL TABLE BugsText USING fts3(summary, description);
```
다른 테이블로부터 텍스트를 인덱싱을 하려고 한다면 반드시 해당 데이터를 복사하여 가상 테이블에 추가해야한다. FTS 가상테이블은 항상 `docid`로 불리는 고유기를 가지고 있기 때문에 해당 고유키를 활용하여 원본 테이블과 연관관계를 맺을 수 있다.

```sql
INSERT INTO BugsText (docid, summary, description)
  SELECT bug_id, summary, description FROM Bugs;
```
여기까지 하였다면 아래와 같이 `MATCH` 술어문을 활용하여 검색기능을 사용 할 수 있다.
```sql
SELECT b.* FROM BugsText t JOIN Bugs b ON (t.docid = b.bug_id)
WHERE BugsText MATCH 'crash';
```
해당 패턴 매칭은 불리언 표현식도 사용이 가능하다.
```sql
SELECT * FROM BugsText WHERE BugsText MATCH 'crash -save';
```

### 써드 파티 검색 엔진 사용
데이터베이스 브랜드에 상관없이 검색 기능을 구현하고 싶다면 SQL과 별개로 동작하는 검색 엔진을 찾아서 활용하는것이 좋다.
(책에서 소개해주는 검색엔진에 대한 설명은 생략하겠다.)

### 직접 검색 엔진 구성하기
써드파티 소프트웨어를 사용하는것을 원하지 않거나 검색엔진만을 위한 의존성을 설치하는 것을 원하지 않는다면 직접 구현해보는 것 또한 방법이 될 수 있다. 대신 효율적이면서 데이터베이스 독립적으로 구현해야 할 것이다. 이 섹션에서는 역색인(inverted index)이라고 불리는 디자인을 구성할 예정이다. 기본적으로 역색인은 검색할만한 단어들의 집합이다. 다대다 관계를 통해 해당 인덱스들은 관련성 있는 단어들과 텍스트들을 연관관계를 맺어줄 것이다. 예를 들면 `crash`라는 단어는 수많은 버그 메세지에서 나타날 수 있는 단어이다. 또한 각 버그들은 다른 많은 키워드들과 매칭 될 수 있다. 이번 섹션에서는 이런 경우들에 대해 대응 할 수 있는 역색인을 어떻게 디자인 할 수 있는지 보여주려고 한다.

우선 사용자들이 검색할만한 키워드의 집합인 `Keywords` 테이블과 `Bugs` 테이블과 `Keywords` 테이블을 이어줄 수 있는 `BugsKeywords` 중계 테이블을 선언한다.

```sql
CREATE TABLE Keywords (
  keyword_id SERIAL PRIMARY KEY,
  keyword VARCHAR(40) NOT NULL,
  UNIQUE KEY (keyword)
);

CREATE TABLE BugsKeywords (
  keyword_id BIGINT UNSIGNED NOT NULL,
  bug_id BIGINT UNSIGNED NOT NULL,
  PRIMARY KEY (keyword_id, bug_id),
  FOREIGN KEY (keyword_id) REFERENCES Keywords(keyword_id),
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);
```

그 다음엔 `BugsKeywords` 테이블에 주어진 버그와 매칭 될 수 있는 모든 단어들을 로우로 추가한다. `LIKE`나 정규표현식을 활용하여 매칭되는 부분 문자열들을 
추출하여 추가하면 된다. 이 경우에는 우리가 안티패턴이라고 언급했던 방식과 별반 다르지 않는 성능을 보여주지만 부분 문자열을 추출하는 순간에만 사용하면 되기 때문에 효율적으로 사용할 수 있다. 결과 데이터를 중계테이블에 저장한 이후에는 동일한 단어를 훨씬 더 빠르게 검색 할 수 있다.

이 다음으로는 저장 프로시져를 만들어서 주어진 단어를 더 효율적으로 검색할 수 있도록 구현하면 된다. 이미 검색한 적이 있는 단어라면 해당 쿼리는 더 빨리 검색을 수행하게 되는데 이는 `BugsKeywords` 테이블에 이미 한번이라도 검색 요청한 단어들이 저장되어 있기 때문이다. 만약 한번도 검색되지 않은 단어라면 단어 리스트에서 있는지 전체 검색 한 다음 추가하는 절차를 만들어야 한다.

```sql
CREATE PROCEDURE BugsSearch(keyword VARCHAR(40))
BEGIN
  SET @keyword = keyword;

  # step1
  PREPARE s1 FROM 'SELECT MAX(keyword_id) INTO @k FROM Keywords
    WHERE keyword = ?';
  EXECUTE s1 USING @keyword;
  DEALLOCATE PREPARE s1;

  IF (@k IS NULL) THEN
    # step2
    PREPARE s2 FROM 'INSERT INTO Keywords (keyword) VALUES (?)';
    EXECUTE s2 USING @keyword;
    DEALLOCATE PREPARE s2;
    # step3
    SELECT LAST_INSERT_ID() INTO @k;

    # step4
    PREPARE s3 FROM 'INSERT INTO BugsKeywords (bug_id, keyword_id)
      SELECT bug_id, ? FROM Bugs
      WHERE summary REGEXP CONCAT(''[[:<:]]'', ?, ''[[:>:]]'')
        OR description REGEXP CONCAT(''[[:<:]]'', ?, ''[[:>]]'')';
    EXECUTE s3 USING @k, @keyword, @keyword;
    DEALLOCATE PREPARE s3;
  END IF;

  # step5
  PREPARE s4 FROM 'SELECT b.* FROM Bugs b
    JOIN BugsKeywords k USING (bug_id)
    WHERE k.keyword_id = ?';
  EXECUTE s4 USING @k;
  DEALLOCATE PREPARE s4;
END
```

1. 사용자가 정의한 키워드를 검색한다. `Keywords.keyword_id` 고유키 값을 반환하거나 해당 키워드가 검색 된 적이 없다면 `null`을 반환한다.
2. 해당 키워드를 찾지 못한경우 새로운 단어로 추가한다.
3. 새로 추가된 키워드의 고유키 값을 반환한다.
4. 중계테이블에 새로 추가된 키워드와 매칭되는 `Bugs` 로우를 찾아 해당 고유키들을 추가한다.
5. 위 작업을 거치고 나면 해당 키워드의 고유키와 관계를 가지고 있는 `Bugs` 로우들을 찾을 수 있다.

해당 프로시져를 실행하면 우리가 원하는 결과를 얻을 수 있게 된다. 이 프로시져를 통해 버그 테이블에서 해당 키워드와 매칭되는 로우를 찾는 작업과 새 키워드를 추가하는 작업을 수행함과 동시에 이미 한번이라도 검색 된적이 있는 키워드는 바로 중계 테이블을 통해 결과를 볼수 있게 되었다.

```sql
CALL BugsSearch('crash');
```

또 하나 해줘야 할 작업은 트리거를 정의하여 새로운 버그가 추가될 때 마다 중계 테이블에 해당 버그를 등록해주는 작업을 해줘야한다. 만약 버그 데이터를 수정하는 작업이 필요하다면 수정작업에도 동일하게 트리거를 적용해줘야 한다.

```sql
CREATE TRIGGER Bugs_Insert AFTER INSERT ON Bugs
FOR EACH ROW
BEGIN
  INSERT INTO BugsKeywords (bug_id, keyword_id)
    SELECT NEW.bug_id, k.keyword_id FROM Keywords k
    WHERE NEW.description REGEXP CONCAT('[[:<:]]', k.keyword, '[[:>:]]')
        OR NEW.summary REGEXP CONCAT('[[:<:]]', k.keyword, '[[:>:]]');
END
```

이렇게 구성하면 사용자가 매번 검색할 때마다 자연스립게 키워드가 추가되기 때문에 손으로 직접 키워드 리스트를 만들어서 추가할 필요가 없어진다. 다시 말해 초기에 단어 등록할 때 성능 저하만 감수하면 효율적으로 검색을 할 수 있게 도와줄 수 있게된다.

필자는 역색인 디자인을 통해 검색 시스템을 구현해서 현재까지 사용하고 있다. 여기에 `num_searches` 컬럼을 `Keywords` 테이블에 추가하여 각 단어들이 얼마나 자주 검색되는지도 확인하고 있다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter17 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

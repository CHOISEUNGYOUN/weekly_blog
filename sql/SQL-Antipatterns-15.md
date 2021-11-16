# Chapter 15 Ambiguous Groups (애매한 그룹)

## 목표 - 누락된 값을 구분하기
SQL에서 `GROUP BY` 쿼리를 배우는 단계에 있는 많은 개발자들은 해당 기능을 활용하여 로우를 통합하고 집계를 내는것을 좋아한다. 이 강력한 기능을 잘 활용하면 아무리 복잡해보이는 보고서라도 단 몇줄의 코드만으로도 결과를 출력해 낼 수 있다.

예를들어 각 제품에서 가장 최근에 보고된 버그를 데이터베이스에서 찾으려고 한다면 다음과 같이 조회하면 된다.

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

여기서 `bug_id`를 함께 조회하려는 경우 대부분 아래와 같은 쿼리를 이용하여 조회하려 할 것이다.
```sql
SELECT product_id, MAX(date_reported) AS latest, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

하지만 이 쿼리를 실행하면 에러가 발생하거나 정확하지 않은 결과를 리턴한다. 이는 SQL을 사용하는 대부분의 개발자들이 혼란스러워 하는 요소이다.

이 챕터의 주제는 가장 최근에 보고된 버그 그룹을 찾는 것 뿐만 아니라 해당 로우의 다른 값들을 추출 할 수 있는 방법을 알아보고자 한다.

## 안티패턴 - 그룹핑 되지 않은 컬럼 값들을 참조하기

이 안티패턴의 원인은 SQL에서 그룹핑이 어떻게 동작하는지 잘 이해하지 못해 발생하기 때문에 그룹핑이 어떻게 동작하는지 이해하는 것이 중요하다.

### 단일 결과값 원칙
각 그룹의 로우들은 해당 컬럼의 동일한 값 또는 `GROUP BY` 로 지정한 컬럼의 값들이 나온다. 예를들어 아래 예시의 쿼리는 `product_id` 별로 오직 하나의 로우 값만 출력한다.

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

쿼리로 선택된 컬럼 리스트는 반드시 그룹별로 하나의 로우만 반환되어야 한다. 이를 단일 결과의 원칙이라고 한다. `GROUP BY`로 선택된 컬럼은 얼마나 많은 로우 데이터가 매칭되던 상관없이 그룹별로 하나의 값만 반환하도록 보장된다.

`MAX()` 함수 또한 각 그룹별로 하나의 값만 반환하도록 설계되어있다. 즉, 그룹핑된 데이터중 가장 높은 값의 데이터만 출력된다는 이야기이다.

하지만 데이터베이스는 선택된 다른 리스트의 값들이 이런 조건에 해당되는지 보장하지는 않는다. 즉, 그룹핑된 데이터 중 다른 컬럼의 데이터는 항상 동일하지 않다는 이야기이다.
```sql
# MySQL에서는 해당 쿼리를 실행 할 수 없음!
SELECT product_id, MAX(date_reported) AS latest, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```
위 예시에서는 `BugsProducts` 조인 테이블에 한 `product_id` 당 여러 `bug_id`가 맺어져 있기 때문에 `product_id` 그룹에 해당되는 `bug_id` 여러 고유키 값들이 존재한다. 해당 그룹핑 쿼리는 `product_id` 기준으로 로우의 갯수를 통합하기 때문에 모든 `bug_id` 값들을 가질 수 없다.

### 내 뜻대로 동작하는 쿼리
개발자들이 잘못 이해하고 있는 상식 중 하나는 SQL이 `MAX()` 함수를 다른 컬럼에서 사용하고 있기 때문에 어떤 `bug_id` 값을 선택하려고 하는지 이해하고 있다고 착각하는 것이다. 대부분의 사람들이 해당 쿼리가 최대값을 불러오기 때문에 다른 컬럼의 값 또한 자연스럽게 그 최대값에 해당되는 로우의 컬럼 값들을 불러온다고 잘못 이해하고 있다. 이런 잘못된 컨셉들을 아래 예시에서 짚어보고자 한다.

* 두가지 버그 데이터가 동일한 `date_reported` 컬럼 값을 가지고 있고 해당 값이 그룹핑 된 값들 중 가장 큰 값인 경우 어떤 `bug_id`를 반환하는지 모르는 경우.
* `MAX()` 나 `MIN()` 과 같은 두가지 통계 함수를 사용하여 조회하는 경우 각 함수에 대한 두가지 로우 값을 반환할 것으로 예상하지만 어떤 `bug_id`를 반환하는지 모르는 경우.

```sql
SELECT product_id, MAX(date_reported) AS latest,
  MIN(date_reported) AS earliest, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```
* 집계 함수를 사용했을 때 어떤 로우 데이터도 조건에 만족하지 못하는 경우 어떤 `bug_id`를 반환하는지 모르는 경우.
```sql
SELECT product_id, SUM(hours) AS total_project_estimate, bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

위 경우들이 왜 단일 결과의 원칙이 필요한지 잘 설명해주고 있다. 모든 쿼리들이 이런 모호한 결과를 반환하는 것은 아니나 대부분의 경우 부정확하게 동작한다. 이런 모호성을 무시하고 동작시키면 앱 전체의 신뢰성을 낮추는 결과를 초리핸다.

## 어떻게 안티패턴을 구분하는가
대부분의 데이터베이스 제품에서 이런 단일 결과의 원칙을 무시하는 쿼리들은 즉각적으로 에러를 반환하도록 설계되어 있다. 아래 메세지들이 각 데이터베이스에서 반환하는 에러 메세지이다.

* Firebird 2.1:
```sql
Invalid expression in the select list (not contained in either an
aggregate function or the GROUP BY clause)
```
* IBM DB2 9.5:
```sql
An expression starting with "BUG_ID" specified in a SELECT clause,
HAVING clause, or ORDER BY clause is not specified in the GROUP BY
clause or it is in a SELECT clause, HAVING clause, or ORDER BY clause
with a column function and no GROUP BY clause is specified.
```
* Microsoft SQL Server 2008:
```sql
Column 'Bugs.bug_id' is invalid in the select list because it is not
contained in either an aggregate function or the GROUP BY clause.
```
* MySQL 5.1 (`ONLY_FULL_GROUP` SQL 모드 옵션을 비활성화 한 경우)
```sql
'bugs.b.bug_id' isn't in GROUP BY
```
* Oracle 10.2:
```sql
not a GROUP BY expression
```
* PostgreSQL 8.3:
```sql
column "bp.bug_id" must appear in the GROUP BY clause or be
used in an aggregate function
```

SQLite나 MySQL에서는 이런 모호한 컬럼에 대한 결과값으로 예상하지 못한 이상한 데이터 값을 반환할 수도 있다. MySQL에서는 그룹핑된 데이터중 가장 첫번째로 저장된 데이터를 반환한다. SQLite 에서는 이와 반대로 가장 최근의 데이터를 반환한다. 이 두 케이스 모두 문서에 기술되어 있지 않고 이런식으로 동작한다고 보장하지도 않기 때문에 버전별로 상이할 수 있다.

## 안티패턴 사용이 정당화 되는 경우
위에서 언급했듯이 MySQL이나 SQLite는 단일 결과 원칙에 해당되지 않는 컬럼에 대해 신뢰성이 보장되는 결과값을 반환하지 못한다. 이런 특성을 잘 이해하면 아래와 같은 쿼리로 응용할 수 있다.

```sql
SELECT b.reported_by, a.account_name
FROM Bugs b JOIN Accounts a ON (b.reported_by = a.account_id)
GROUP BY b.reported_by;
```
이 쿼리의 `account_name` 컬럼은 원칙적으로 `GROUP BY` 에 의해 지정되지도 집계 함수의 인자로 사용되지 않았기 때문에 단일 결과 원칙에 위배된다. 그럼에도 각 그룹당 하나의 `account_name` 을 반환한다. 해당 그룹은 `Accounts` 테이블의 외래키인 `Bugs.reported_by`을 기반으로 만들어진다. 결과적으로 `Accounts` 테이블과 일대일로 상응하는 그룹 데이터가 생성된다.

다시 말해 `reported_by`의 데이터를 인지하고 있으면 `account_name`의 값을 마치 `Accounts` 테이블의 고유키를 통해 조회하는 것과 동일한 결과를 가져 올수 있다는 것이다.

이런 명확한 관계를 함수 종속성이라고 일컫는다. 가장 흔한 예시는 테이블의 고유키와 속성들이다. 여기서 `account_name`은 `account_id`에 대한 함수 종속적인 관계를 갖는다. 테이블의 고유키를 가지고 그룹핑 쿼리를 실행하는 경우 이 그룹핑 데이터는 해당 테이블의 각 로우 데이터를 모두 반환하기 때문에 같은 테이블의 다른 컬럼 값 또한 그룹 당 하나의 동일한 데이터를 반환한다.

`Bugs.reported_by` 또한 `Accounts` 테이블의 고유키 값을 참조하고 있기 때문에 `Accounts` 테이블과 비슷한 함수 종속적 관계를 가지고 있다. `reported_by` 컬럼을 기준으로 그룹핑을 하는 경우 `Accounts` 테이블의 함수 종속적인 관계 때문에 모호하지 않은 쿼리 결과값을 반환하게 된다.

하지만 대부분의 데이터베이스 제품에서는 이런 쿼리에 대해 에러를 반환한다. 이는 함수 종속적인 관계를 알아내는게 그리 어려운 일도 아니기도 하고 이런 동작 자체가 SQL 표준에 위배되기 때문이다. 그럼에도 MySQL이나 SQLite를 사용하고 있다면 이런 함수 종속적인 관계를 인지하고 모호한 관계를 만들지 않도록 조심해야 한다.

## 해결책 - 컬럼을 모호하게 사용하지 않기

### 함수 종속인 컬럼만 조회하기
가장 직관적인 해결책은 쿼리에서 모호한 컬럼을 제외하는 것이다.

```sql
SELECT product_id, MAX(date_reported) AS latest
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

이 쿼리는 가장 최근에 발생한 버그에 대한 `bug_id`를 제공해주진 않지만 각 제품당 발생한 가장 최근의 버그를 반환한다. 이 방식은 가장 간단하면서도 명확하다.

### 상호연관된 서브쿼리 사용하기
상호 연관된 서브쿼리는 외부쿼리를 참조하기 때문에 외부쿼리를 통해 조회된 각 로우마다 다른 결과를 반환한다. 이를 통해 각 제품 당 발생한 가장 최근의 버그 데이터를 조회할 수 있다. 서브쿼리에서 아무것도 반환하지 않는다면 외부쿼리에서 조회된 버그 데이터가 가장 최근에 발생한 버그가 된다.

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
WHERE NOT EXISTS
  (SELECT * FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
   WHERE bp1.product_id = bp2.product_id
    AND b1.date_reported < b2.date_reported);
```
이 방법은 코드 측면에서 간단하고 가독성이 좋은 해결 방법이다. 하지만 상호연관된 서브쿼리들을 각 로우당 한번씩 외부쿼리로 비교해야되기 때문에 성능적인 측면에서 최선의 방법은 아니다.

### 유도 테이블 사용하기
`product_id`와 이에 해당되는 가장 최근의 버그의 보고일을 담은 임시결과를 유도 테이블로써 구성하여 해결할수도 있다. 해당 결과를 테이블과 비교 조회하면 결과로 각 제품당 가장 최근에 발생한 버그 값만 불러올 수 있다.

```sql
SELECT m.product_id, m.latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 USING (bug_id)
  JOIN (SELECT bp2.product_id, MAX(b2.date_reported) AS latest
    FROM Bugs b2 JOIN BugsProducts bp2 USING (bug_id)
    GROUP BY bp2.product_id) m
  ON (bp1.product_id = m.product_id AND b1.date_reported = m.latest);
```
|product_id|latest    |bug_id|
|----------|----------|------|
|1         |2010-06-01|2248  |
|2         |2010-02-16|3456  |
|2         |2010-02-16|5150  |
|3         |2010-01-01|5678  |

위 결과에서 볼 수 있듯이 각 제품당 가장 최근에 발생한 버그와 해당 고유키를 반환하고 있다. 각 제품당 하나의 버그만 조회하고 싶다면 아래와 같이 그룹핑을 하면 된다.

```sql
SELECT m.product_id, m.latest, MAX(b1.bug_id) AS latest_bug_id
FROM Bugs b1 JOIN
  (SELECT product_id, MAX(date_reported) AS latest
   FROM Bugs b2 JOIN BugsProducts USING (bug_id)
   GROUP BY product_id) m
  ON (b1.date_reported = m.latest)
GROUP BY m.product_id, m.latest;
```
|product_id|latest    |bug_id|
|----------|----------|------|
|1         |2010-06-01|2248  |
|2         |2010-02-16|5150  |
|3         |2010-01-01|5678  |

유도 테이블을 사용하는 방법은 위에서 제시한 상호연관된 서브쿼리를 사용하는것 보다 확장성이 좋다. 유도 테이블은 상호 연관 관계가 아니기 때문에 대부분의 데이터베이스에서는 서브쿼리를 한번만 실행해주면 된다. 하지만 데이터베이스에 반드시 임시로 출력한 결과를 임시 테이블로써 저장해둬야하기 때문에 성능 측면에서 가장 좋은 해결책이라고 볼 순 없다.

### JOIN 사용하기
JOIN을 사용하여 존재하지 않는 로우에 대한 값들을 대조할수도 있다. 이런 방식의 병합을 외부 병합(OUTER JOIN)이라고 부른다. 로우 대조시 데이터가 존재하지 않는 경우 해당 로우의 컬럼값에 `null`이 모두 선언된다. 이를 통해 병합을 할때 매칭된 데이터가 없음을 인지할 수 있다.

```sql
SELECT bp1.product_id, b1.date_reported AS latest, b1.bug_id
FROM Bugs b1 JOIN BugsProducts bp1 ON (b1.bug_id = bp1.bug_id)
LEFT OUTER JOIN (Bugs AS b2 JOIN BugsProducts AS bp2 ON (b2.bug_id = bp2.bug_id))
  ON (bp1.product_id = bp2.product_id AND (b1.date_reported < b2.date_reported
    OR b1.date_reported = b2.date_reported AND b1.bug_id < b2.bug_id))
WHERE b2.bug_id IS NULL;
```

|product_id|latest    |bug_id|
|----------|----------|------|
|1         |2010-06-01|2248  |
|2         |2010-02-16|5150  |
|3         |2010-01-01|5678  |

위 쿼리는 직관성이 떨어지기 때문에 이해하는데 시간이 필요할 것이다. 이해하기만 한다면 이런 방식이 유용함을 깨닫게 될 것이다.

이러한 방식은 다량의 세트 데이터를 다룰때 유용하다. 보기에는 복잡하고 어려운 개념으로 보이지만 서브쿼리를 활용하여 해결하는 방식보다 확장성 측면에서 훨씬 더 용이하다. 성능 측면에서도 다른 해결 방식보다 더 좋은 방법이기도 하다.

### 다른 컬럼에 집계 함수 사용하기
단일 결과의 원칙을 응용하여 집계함수를 활용하여 병합을 할 수도 있다.

```sql
SELECT product_id, MAX(date_reported) AS latest,
  MAX(bug_id) AS latest_bug_id
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```
이 방법은 가장 최근에 발생한 `bug_id`의 `date_reported` 값이 정확한 경우 사용 할 수 있다. 다시 말해 버그 데이터가 시간 순서대로 쌓이는 경우에 사용할 수 있다.

### 각 그룹에 대해 모든 값을 연결하기
다른 집계 함수를 사용하여 단일 결과 원칙을 위배하지 않고 해결 할 수도 있다. MySQL과 SQLite에서는 `GROUP_CONCAT()` 함수를 지원하는데 이는 그룹에 해당되는 모든 컬럼 값을 하나로 병합해준다. 해당 컬럼은 기본적으로 콤마(`,`)로 분리하도록 설계되어 있다.

```sql
SELECT product_id, MAX(date_reported) AS latest
  GROUP_CONCAT(bug_id) AS bug_id_list,
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

|product_id|latest    |bug_id_list   |
|----------|----------|--------------|
|1         |2010-06-01|1234,2248     |
|2         |2010-02-16|3456,4077,5150|
|3         |2010-01-01|5678,8063     |

이 결과값은 어떤 `bug_id`가 가장 최근에 발생한 버그인지 구분하지는 못한다. 또다른 문제점은 이 방식은 표준 SQL에 기재되어 있는 방식이 아니라는 점이다. 이 말은 다른 데이터베이스 제품에서는 이러한 방식을 적용 할 수 없다는 것이다. 다른 제품에서 종종 이런 집계를 구현하기 위해 사용자정의 함수를 구현 할 수 있게 지원하는 경우도 있다. 아래의 예시는 PostgreSQL의 사용자정의 함수 예시이다.

```sql
CREATE AGGREGATE GROUP_ARRAY (
  BASETYPE = ANYELEMENT,
    SFUNC = ARRAY_APPEND,
    STYPE = ANYARRAY,
  INITCOND = '{}'
);

SELECT product_id, MAX(date_reported) AS latest
  ARRAY_TO_STRING(GROUP_ARRAY(bug_id), ',') AS bug_id_list,
FROM Bugs JOIN BugsProducts USING (bug_id)
GROUP BY product_id;
```

다른 데이터베이스에서는 이런 사용자정의 함수 기능을 제공하지 않기 때문에 프로시져를 활용하여 반복문을 실행하여 직접 병합을 시켜줘야 한다. 이 방식은 그룹당 추가 컬럼 데이터가 필요한 경우 단일 결과 원칙을 위배하지 않고 해결할 수 있는 방법이다.


> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter15 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

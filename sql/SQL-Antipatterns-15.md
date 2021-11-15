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
작성중...

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter15 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

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

작성중...

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter15 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

# Chapter 18 Spaghetti Query (스파게티 쿼리)

## 목표 - 쿼리 사이즈 줄이기
많은 개발자들이 쿼리를 작성하면서 가장 많이 고민하는 부분을 꼽자면 "이걸 어떻게 한번의 쿼리로 뽑아내지?" 일 것이다. 개발자들은 기본적으로 SQL 쿼리 자체가 어렵고 복잡하며 자원 소모가 많은 작업으로 인식하고 있기 때문에 하나로 해결 할 수 있는 일을 두개 이상의 쿼리를 실행하는 것은 두배로 나쁜 일이라고 인식한다. 한마디로 한번 이상의 SQL 쿼리로 문제를 해결하는 것 자체를 염두에 두지 않는다는 것이다.

개발자들이 해당 작업의 복잡도를 낮출 수는 없지만 간단하게 해결하려는 경향이 있다. 목표는 항상 "아름다우면서도 효율적"으로 문제를 푸는 것으로 생각하기 때문에 대게 어려운 문제를 하나의 쿼리로 풀려고 한다.

## 안티패턴 - 복잡한 문제를 한번에 해결하기
SQL은 매우 자원소모가 심한 언어이다. 따라서 하나의 쿼리나 선언으로 해결하는 것이 좋다. 하지만 이 의미가 항상 한 문제를 한 줄의 코드로 풀어라는 의미는 아니다. 우리는 다른 언어를 사용해서 문제를 해결할때도 이런식으로 접근하지 않는다.

### 의도하지 않은 제곱
모든 결과를 한 쿼리로 뽑아내는 과정에서 발생하는 실수는 곱집합이 발생하는 것이다. 이는 두 테이블을 조인하여 합칠때 아무런 조건이 없는 경우 발생한다. 아래 예시를 실행시키면 이런 현상이 발생하는 것을 이해할 수 있다.
```sql
SELECT p.product_id,
  COUNT(f.bug_id) AS count_fixed,
  COUNT(o.bug_id) AS count_open
FROM BugsProducts p
LEFT OUTER JOIN Bugs f ON (p.bug_id = f.bug_id AND f.status = 'FIXED')
LEFT OUTER JOIN Bugs o ON (p.bug_id = o.bug_id AND o.status = 'OPEN')
WHERE p.product_id = 1
GROUP BY p.product_id;
```
![img](imgs/SQL-Antipatterns-18_1.png)

작성중...

## 어떻게 안티패턴을 구분하는가

## 안티패턴 사용이 정당화 되는 경우

## 해결책 - 분할 정복하기

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter18 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

# Chapter 16 Random Selection (무작위 선택)

## 목표 - 샘플 로우 데이터 가져오기
우리는 앱을 개발하면서 종종 다음과 같은 이유로 무작위 로우를 출력해야 하는 비즈니스 로직을 만들어야 한다.

* 광고나 뉴스 하이라이트와 같이 매번 변경되는 컨텐츠를 노출시키는 경우.
* 데이터의 일부분만 검수하는 경우.
* 상담가능한 상담원에게 인바운드 전화를 넘기는 경우.
* 테스트 데이터를 생성하는 경우.

이런 경우 모든 데이터를 어플리케이션에 로딩하는 것 보다 샘플링 하여 불러 오는 것이 효율적이다. 이번 쳅터의 목표는 어떻게 효율적으로 데이터 샘플링을 할 수 있는지에 대해 다뤄보고자 한다.

## 안티패턴 - 무작위로 데이터 정렬하기
무작위 로우를 뽑는 가장 일반적인 SQL 트릭으로 무작위로 로우를 정렬한 다음 가장 첫번째 데이터를 뽑는 방식이 있다. 이 방식은 직관적이고 쉽게 적용 할 수 있다.
```sql
SELECT * FROM Bugs ORDER BY RAND() LIMIT 1;
```
이 방법이 가장 인기가 많긴 하지만 큰 단점도 있다. 이 단점을 이해하기 위해서는 기존에 사용하고 있던 정렬 방식, 즉 컬럼 값을 비교하고 오름차순 또는 내림차순으로 정렬하는 방식과 비교해보아야 한다. 이런 방식의 정렬은 여러번 실행 하더라도 동일한 결과를 출력하기 때문에 반복 가능하다. 또한 인덱스를 활용하기 때문에 빠르게 호출한다. 이는 기본적으로 인덱스가 이미 해당 컬럼을 정렬된 상태의 값들을 가지고 있기 때문이다.

```sql
SELECT * FROM Bugs ORDER BY date_reported;
```
정렬 기준이 로우 별로 무작위 값을 반환 한다면 해당 값이 다른 로우 값 또한 무작위로 선택되기 때문에 어떤 값이 크고 작은지 또한 무작위가 되어버린다. 결국 이 값은 각 로우마다 연관성이 전혀 없는 데이터셋을 반환한다. 그렇기 때문에 해당 쿼리를 실행할 때 마다 다른 값을 반환하게 된다. 여기까지는 우리가 원하는 진짜 무작위 데이터를 뽑는 가설에 충족하기 때문에 좋은 결과로 생각할 수도 있다.

`RAND()`와 같은 비결정론적 표현식을 정렬에 사용한다는 것은 인덱스를 활용하여 정렬을 할 수 없다는 것과 동일하다. 무작위 함수를 도울 수 있는 인덱스 같은 것은 없기 때문이다.

이것이 쿼리 성능을 저하시키는 원인이다. 인덱스를 사용하지 않기 때문에 정렬하는데 최적의 속도를 내지 못하기 때문이다. 인덱스를 사용하지 않게되면 데이터베이스 내에서 매번 실행 될 때마다 직접 정렬을 수행한다. 이를 테이블 `순차접근(full table scan)` 이라고 부르며 이 과정에서 모든 결과를 임시 테이블에 저장하고 직접 일일이 로우를 바꿔가며 정렬을 수행한다. 이런 순차접근 정렬 방식은 인덱스를 사용한 정렬 방식보다 훨씬 느리고 데이터셋이 커지면 커질수록 그 차이는 더 극심해진다.

또 다른 문제점은 이렇게 자원이 많이 소모되는 프로세스를 수행하지만 대부분의 데이터셋들이 버려진다는 점이다. 이는 대게 첫번째 로우를 제외하곤 필요없는 데이터이기 때문이다.

두가지 문제 모두 적은 데이터셋을 조회하는 경우 눈에 띄지 않기 때문에 개발 및 테스트 단계에서 잡아내기 어렵다. 이는 결국 데이터 양이 많아질수록 전체 데이터베이스의 성능을 저하시키는 원인이 된다.

## 어떻게 안티패턴을 구분하는가
안티패턴으로 살펴본 이 테크닉은 직관적이고 블로그 같은데 흔히 볼 수 있기 때문에 많은 개발자들이 애용하는 패턴이다. 아래의 예시와 같은 문제를 당신의 동료가 겪고 있다면 이는 이 안티패턴을 적용하고 있다는 증거이다.

* SQL에서 무작위 로우 값을 추출하는 것은 엄청 느리다고 말하는 경우.<br>
무작위 샘플 데이터를 추출하는 이 쿼리는 적은 데이터를 대상으로 하는 경우 별 문제없이 잘 동작하지만 데이터가 커지면 커질수록 점점 느려진다. 이 경우 데이터베이스 튜닝이나 인덱싱 또는 캐싱으로 해결 할 수 없다.
* 모든 로우중에서 하나의 로우만 무작위로 뽑기 위해 앱 메모리 용량을 늘리려고 하는 경우.<br>
단 하나의 무작위 로우 값을 뽑아내기 위해 모든 로우 데이터를 무작위로 정렬하는 것은 엄청난 낭비이다. 이외에도 서비스를 운영하다보면 자연스럽게 데이터베이스 크기 또한 앱에서 다룰 수 있는 메모리 용량보다 커진다.
* 무작위로 뽑아내는 로우 값이 무작위가 아니라 특정 로우만 더 자주 반환하는 경우<br>
실제로 무작위로 뽑아낸 숫자는 빈 고유키 값들 때문에 100% 일치하지 않는다.

## 안티패턴 사용이 정당화 되는 경우
무작위로 뽑아내는 것이 비효율적으로 보여도 데이터셋 크기 자체가 작은 경우 최상의 성능을 보장해주진 않지만 그럭저럭 쓸만하다. 예를 들어 무작위 함수를 사용하여 개발자에게 버그를 할당하는 비즈니스 로직은 큰 문제가 되지 않는다. 적어도 개발자를 10만명 정도 고용하지 않는한 말이다. 또 다른 예시로 미국의 50개주 중에서 무작위로 하나의 주를 선택하는 로직 또한 데이터셋 자체가 적고 변할 가능성이 없기 때문에 이 함수를 사용해도 별 문제가 되지 않는다.

## 해결책 - 테이블 전체 정렬 피하기
무작위로 정렬하는 방식은 순차접근과 자원이 매우 많이 소모되는 직접 정렬 방식의 전형적인 예이다. SQL로 비즈니스 로직을 구성하는 경우 이런 비효율적인 쿼리들을 반드시 관리해야한다. 이런 경우 최적화 시킬 수 없는 쿼리이기 때문에 다른 접근법을 취해야한다. 지금부터 다른 접근법을 통한 해결방법을 제시하고자 한다.

### 1 과 최대값 중 하나의 무작위 키 값을 선별하기
이 방법은 단순히 고유키 값중 하나의 키를 무작위로 선별하는 방식이다.
```sql
SELECT b1.*
FROM Bugs AS b1
JOIN (SELECT CEIL(RAND() * (SELECT MAX(bug_id) FROM Bugs)) AS rand_id) AS b2
  ON (b1.bug_id = b2.rand_id);
```
이 방법은 고유키가 1부터 순차적으로 존재한다고 가정 할 때 잘 동작한다. 다시 말해 1부터 가장 최근의 고유키 사이에 빈 값이 없어야 한다는 것이다. 만약 중간에 빈 값이 존재하면 무작위로 뽑힌 숫자가 존재하지 않는다고 인식하기 때문에 어떤 로우도 반환하지 않게 된다. 따라서 이 방법은 고유키가 순차적으로 존재할때만 사용이 가능하다.

### 다음으로 큰 키 값 선별하기
이 방법은 앞선 방법과 비슷하지만 고유키 사이에 공백이 있는 경우 무작위 숫자보다 큰 고유키 중 가장 첫번째 로우 값을 반환한다.
```sql
SELECT b1.*
FROM Bugs AS b1
JOIN (SELECT CEIL(RAND() * (SELECT MAX(bug_id) FROM Bugs)) AS bug_id) AS b2
WHERE b1.bug_id >= b2.bug_id
ORDER BY b1.bug_id
LIMIT 1;
```

이 방법은 무작위 숫자값이 고유키로 존재하지 않는 경우를 해결해주지만 특정 공백 사이에 존재하는 로우 값을 더 많이 노출시키는 문제를 야기시킨다. 이는 무작위 값은 반드시 동일한 확률로 나타나야한다는 규칙을 위배하게 된다. 이 방법은 고유키 사이에 공백이 있지만 모든 로우 데이터를 동일한 확률로 노출시키지 않아도 되는 경우 사용 할 수 있다.

### 모든 키 값의 목록을 불러 온 후 하나만 무작위로 선택하기
어플리케이션 백엔드에서 모든 고유키 값을 불러온 후 하나의 고유키만 선택하는 방법도 있다. 아래의 예시는 PHP로 구성한 코드이다.
```php
<?php
$bug_id_list = $pdo->query("SELECT bug_id FROM Bugs")->fetchAll();

$rand = random( count($bug_id_list) );
$rand_bug_id = $bug_id_list[$rand]["bug_id"];
$stmt = $pdo->prepare("SELECT * FROM Bugs WHERE bug_id = ?");
$stmt->execute( array($rand_bug_id) );
$rand_bug = $stmt->fetch();
```
이 방법은 테이블 정렬을 하지않고도 무작위로 로우를 선별할 수 있지만 두가지 문제를 가지고 있다.
* 모든 `bug_id`를 불러오는 것 자체가 엄청난 양의 데이터를 반환하는 작업이다. 이는 앱에서 지원하는 메모리 자원을 훨씬 뛰어넘어 에러를 유발시킬수 있다.
* 이 방법은 모든 고유키를 먼저 불러온 다음 무작위로 로우를 뽑아내야하기 때문에 반드시 두번의 쿼리 실행이 필요하다. 만약 해당 쿼리가 복잡하고 자원을 많이 소모한다면 문제가 된다.

이 방법은 데이터셋이 적절한 수준인 경우 사용해볼 수 있다. 또한 연속적이지 않은 로우 셋을 불러오는데 용이하다.

### Offset을 활용하여 무작위 로우 선별하기
위와 유사하지만 또 다른 방법으로 `COUNT` 통계 함수를 활용해서 무작위 숫자를 만든 뒤 `OFFSET`을 설정하여 해당 영역의 로우 데이터만 뽑아내는 방식도 있다.
```php
<?php
$rand = "SELECT ROUND(RAND() * (SELECT COUNT(*) FROM Bugs))";
$offset = $pdo->query($rand)->fetch(PDO::FETCH_ASSOC);

$sql = "SELECT * FROM Bugs LIMIT 1 OFFSET :offset";
$stmt = $pdo->prepare($sql);
$stmt->execute( $offset );
$rand_bug = $stmt->fetch();
```
이 방법은 SQL 표준이 아닌 `LIMIT` 절을 사용해야되기 때문에 MySQL, PostgreSQL, SQLite에서만 사용이 가능하다. Oracle, Microsoft SQL, IBM DB2에서는 `ROW_NUMBER()` 함수를 활용해서 동일한 효과를 낼 수 있다. 예를 들어 Oracle에서는 아래와 같이 적용해 볼 수 있다.
```php
<?php
$rand = "SELECT 1 + MOD(ABS(dbms_random.random()),
(SELECT COUNT(*) FROM Bugs)) AS offset FROM dual";
$offset = $pdo->query($rand)->fetch(PDO::FETCH_ASSOC);

$sql = "WITH NumberedBugs AS (
SELECT b.*, ROW_NUMBER() OVER (ORDER BY bug_id) AS RN FROM Bugs b
) SELECT * FROM NumberedBugs WHERE RN = :offset";
$stmt = $pdo->prepare($sql);
$stmt->execute( $offset );
$rand_bug = $stmt->fetch();
```
이 방법은 고유키가 연속으로 존재하지 않고 모든 데이터가 균일하게 노출되어야 하는 경우 사용하면 좋다.

### 특정 데이터베이스만의 고유 솔루션 사용하기
무작위 데이터를 뽑아내기 위해 각 데이터베이스마다 각기 다른 솔루션을 제공하고 있다. Microsoft SQL에서는 `TABLESAMPLE`이라는 절을 제공한다.
```sql
SELECT * FROM Bugs TABLESAMPLE (1 ROWS);
```
Oracle은 살짝 다르지만 유사한 `SAMPLE` 절을 지원한다.
```sql
SELECT * FROM (SELECT * FROM Bugs SAMPLE (1)
ORDER BY dbms_random.value) WHERE ROWNUM = 1;
```
이는 각 데이터베이스마다 제공해주는 방법이 다르기 때문에 해당 데이터베이스의 문서를 잘 읽고 적용해야된다.

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter16 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.
# Chapter 8 Multicolumn Attributes (다중 컬럼 속성)

## 목표 - 여러 값 저장하기
이 패턴은 2장에서 언급한 무단횡단 패턴이 한 컬럼에 여러 값을 저장하는 것과는 반대로 비슷하거나 동일한 속성값을 여러 컬럼을 만들어 한 테이블에 저장하는 패턴이다.

여기서 예시로 `Bugs` 테이블에 `tag` 라는 컬럼을 만들어 해당 버그가 어떤 카테고리로 분류되는지 구분시켜준다. 여기서 문제는 하나의 버그가 여러 태그 속성을 가지도록 해야한다는 것이다.

## 안티패턴 - 여러개의 컬럼 생성하기

위 설계 조건에 따라 `Bugs` 테이블에 여러 태그를 설정할 수 있도록 `tag` 컬럼을 3개 추가했다.

```sql
CREATE TABLE Bugs (
  bug_id SERIAL PRIMARY KEY,
  description VARCHAR(1000),
  tag1 VARCHAR(20),
  tag2 VARCHAR(20),
  tag3 VARCHAR(20)
);
```

이런식의 테이블을 구성하면 아래처럼 태그가 관리된다.

|bug_id|description|tag1|tag2|tag3|
|------|-----------|----|----|----|
|1234|Crashes while saving|crash|NULL|NULL|
|3456|Increase performance|printing|performance|NULL|
|5678|Support XML|NULL|NULL|NULL|

### 특정 값 찾기
위와 같이 관리되고 있는 테이블에서 태그에 해당되는 로우 값을 찾으려면 다음과 같은 처리를 해줘야 한다.

```sql
SELECT * FROM Bugs
WHERE tag1 = 'performance'
  OR tag2 = 'performance'
  OR tag3 = 'performance';
```

만약 여기서 `performance`와 `printing` 태그가 달린 로우를 찾으려고 한다면 더욱 복잡해진다.

```sql
SELECT * FROM Bugs
WHERE (tag1 = 'performance' OR tag2 = 'performance' OR tag3 = 'performance')
  AND (tag1 = 'printing' OR tag2 = 'printing' OR tag3 = 'printing');
```

일반적인 방법이 아니지만 아래와 같이 조회 할 수도 있다.
```sql
SELECT * FROM Bugs
WHERE 'performance' IN (tag1, tag2, tag3)
  AND 'printing' IN (tag1, tag2, tag3);
```

### 값 추가 및 삭제
이 구조는 여러 컬럼중에서 어떤 컬럼만 사용할 것인지 보장되지 않기 때문에 안전한 구조가 아니다. 또한 업데이트 하는 과정에서 데이터 충돌이 발생 할 수 있기 때문에 위험하다. 이를 방지하기 위해선 아래와 같이 복잡한 쿼리를 실행해야 한다.
```sql
# 일치하는 태그 값 NULL 변환
UPDATE Bugs
SET tag1 = NULLIF(tag1, 'performance'),
    tag2 = NULLIF(tag2, 'performance'),
    tag3 = NULLIF(tag3, 'performance')
WHERE bug_id = 3456;

# 태그별 중복 값 추가 방지
UPDATE Bugs
SET tag1 = CASE
      WHEN 'performance' IN (tag2, tag3) THEN tag1
      ELSE COALESCE(tag1, 'performance') END,
    tag2 = CASE
      WHEN 'performance' IN (tag1, tag3) THEN tag2
      ELSE COALESCE(tag2, 'performance') END,
    tag3 = CASE
      WHEN 'performance' IN (tag1, tag2) THEN tag3
      ELSE COALESCE(tag3, 'performance') END
WHERE bug_id = 3456;
```
`NULLIF`함수는 조건을 만족하면 해당 컬럼의 값을 `NULL`로 변환시켜 줄 것이다. 그 다음에 작성한 쿼리는 `CASE`문에서 중복되는 태그 없이 업데이트 할 수 있도록 보장 해주는 쿼리문이다. 이런 방식의 관리는 `tag` 컬럼이 늘어나면 늘어날 수록 더 많이 반복해야된다.

### 유일성 보장
여러 태그 컬럼에서 동일한 태그 데이터가 들어가는 것을 방지하기 어렵다.
```sql
INSERT INTO Bugs (description, tag1, tag2, tag3)
  VALUES ('printing is slow', 'printing', 'performance', 'performance');
```

### 세트 값이 늘어남에 따른 데이터 처리
추가해야할 태그가 늘어남에 따라 자연스럽게 관리해야 할 컬럼 수가 늘어나는 구조이다. 초기에 셋업을 잘 해놓은 경우는 괜찮을 순 있지만 중간에 필요에 의해 컬럼을 추가해야 하는 경우 아래와 같은 문제가 발생한다.

* 새로운 컬럼을 추가하는 경우 기존에 존재하는 모든 로우를 스캔해서 업데이트를 해야하므로 그동안 DB 잠김이 발생한다.
* DB타입에 따라 전체 데이터를 복사 한 뒤 새로운 컬럼 추가 및 기존 테이블 삭제가 일어나기 때문에 데이터 양이 많으면 많을수록 시간이 오래걸린다.
* 컬럼을 추가 한 뒤 모든 SQL문을 확인하고 그에 맞게 수정해야 한다.

## 어떻게 안티패턴을 구분하는가
사용자 인터페이스나 문서에 여러개의 값을 할당할 수 있지만 최대 갯수가 제한되어 있는 속성이 있다면 다중 컬럼 속성을 사용하고 있는 것이다.
다른 단서로는 아래의 경우가 있다.

* 태그를 최대 몇개까지 지원해야 하는지 고민하는 경우<br>
다중 값 속성을 위해 얼마나 많은 컬럼을 정의해야 할 지 고민하고 있다는 증거이다.
* 여러 다른 컬럼을 한꺼번에 조회 하려는 경우<br>
주어진 값을 여러 컬럼에서 동시에 살펴보아야 한다면 이는 그 값이 하나의 논리속성으로 저장되어야 하는데 그렇지 못함을 의미한다.

## 안티패턴 사용이 정당화 되는 경우
각 컬럼에 들어갈 데이터의 유형이 정해져 있고 컬럼 별로 의미하는 바가 정확하게 정의되어 있다면 이는 효율적으로 사용될 수 있다. 여기서 태그의 예시로 들자면 첫번째 컬럼에는 버그와 연관되어 있는 계정, 두번째는 버그 보고자, 세번째는 버그 픽스를 위해 배정된 개발자, 네번째에는 QA를 진행할 엔지니어 등, 다 동일한 계정 속성으로 보이지만 그 역할이 저마다 다르다. 즉, 조건과 상황에 맞추어 사용한다면 충분히 용인 될 수 있다.

## 해결책 - 종속 테이블 생성
[2장 무단횡단](./SQL-Antipatterns-2.md)에서 언급했듯이 최고의 해결방법은 종속 테이블을 생성하는 것이다. 이번 장에서는 일대다 관계의 테이블을 생성하여 해결하는 것을 권장한다. 

```sql
CREATE TABLE Tags (
  bug_id BIGINT UNSIGNED NOT NULL
  tag VARCHAR(20),
  PRIMARY KEY (bug_id, tag),
  FOREIGN KEY (bug_id) REFERENCES Bugs(bug_id)
);

INSERT INTO Tags (bug_id, tag)
  VALUES (1234, 'crash'), (3456, 'printing'), (3456, 'performance');
```
하나의 내용에 연관된 모든 태그가 한 컬럼안에 있으므로 조회/수정/삭제가 용이해진다.

```sql
# 단일 태그 관련 버그 조회
SELECT * FROM Bugs JOIN Tags USING (bug_id)
WHERE tag = 'performance';

# 여러 태그 동시에 만족하는 버그 조회
SELECT * FROM Bugs
  JOIN Tags AS t1 USING (bug_id)
  JOIN Tags AS t2 USING (bug_id)
WHERE t1.tag = 'printing' AND t2.tag = 'performance';

# 태그 추가
INSERT INTO Tags (bug_id, tag) VALUES (1234, 'save');

# 태그 삭제
DELETE FROM Tags WHERE bug_id = 1234 AND tag = 'crash';
```

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter8 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

# Chapter 12 31 Flavors (31가지 맛)

## 목표 - 특정 값들만 허용하도록 제한하기
컬럼값에 지정된 값들만 추가할 수 있도록 제약을 거는 것은 유용하다. 허용되지 않는 값들을 제한하면 컬럼에 들어있는 데이터를 사용하기도 훨씬 간편하다. 예를 들어 `Bug`라는 테이블에 `status` 컬럼에는 `NEW`, `IN PROGRESS`, `FIXED` 와 기타 등등의 정해진 데이터만 들어간다고 가정하자. 이 경우 우리는 각 `status` 에 따라 버그 데이터를 관리 할 수 있을 것이다. 또한 지정되지 않는 값들을 허용하지 않음으로써 관리측면에서 효율성도 가져다 줄 수 있다.

```sql
INSERT INTO Bugs (status) VALUES ('NEW'); -- OK
INSERT INTO Bugs (status) VALUES ('BANANA'); -- Error!
```

## 안티패턴 - 컬럼 선언에 특정 값만 사용 할 수 있도록 선언하기(ENUM)
많은 사람들이 특정 데이터 값들만 허용 시키기 위해 `CHECK` 함수를 사용하여 DDL에 다음과 같이 컬럼에 직접 제약조건을 선언한다.

```sql
CREATE TABLE Bugs (
  -- other columns
  status VARCHAR(20) CHECK (status IN ('NEW', 'IN PROGRESS', 'FIXED'))
);
```

위 조건과 비슷하게 MySQL에서는 `ENUM` 데이터 타입을 지원해준다.
```sql
CREATE TABLE Bugs (
  -- other columns
  status ENUM('NEW', 'IN PROGRESS', 'FIXED'),
);
```

위 DDL처럼 `ENUM` 함수를 사용하면 문자열 데이터를 인자로 넘겨주더라도 DB 내부에서는 서수(ordinal number)로 저장해준다. 이로 인해 DB 내에 저장되는 값은 가벼워진다. 또한 조회를 할 시 해당 값은 문자열 알파벳 순서가 아닌 DDL에 선언된 문자열의 순서대로 조회된다. 다른 방법으로 도메인이나 사용자 정의 타입을 사용하는 방법이 있다. 이 방법은 Postgresql에서 사용되는 방법으로 사용자가 미리 지정한 값만 허용하도록 제한 하는 방법이다. 하지만 많은 RDBMS에서 지원하는 시스템은 아니다.

어찌됐든 이 방법을 사용하면 선언시 추가한 문자열 이외에는 데이터를 추가 할 수 없게된다. 이러한 방식은 몇가지 단점이 있는데 그 이야기를 지금부터 하고자 한다.

작성중...

<!-- ### 중간값이 뭐지?

### 새로운 맛 추가하기

### 오래된 맛은 절대 사라지지 않는다

### 마이그레이션이 어려움


## 어떻게 안티패턴을 구분하는가


## 안티패턴 사용이 정당화 되는 경우


## 해결책 - 데이터에 특정 값 제시하기 -->


> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter11 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

# Chapter 9 Metadata Tribbles (메타데이터 트리블)

## 목표 - 확장성 보장하기
어떤 데이터베이스를 사용하더라도 데이터 양이 많아지면 쿼리의 성능이 저하되기 마련이다. 원하는 쿼리의 결과값이 수천개의 로우만 반환 하더라도 테이블의 크기가 커질수록 일어나는 자연스러운 현상이다. 테이블 인덱싱을 통해 조회 속도 향상을 기대 할 수 있지만, 데이터 크기가 문제는 여전히 남아있다. 이번 주제는 데이터 크기가 증가하더라도 균일한 쿼리 성능을 보장하는 데이터베이스 구조를 구축하는 것이다.

## 안티패턴 - 테이블 또는 컬럼 클론하기
적은 로우를 가진 테이블을 조회 할 때 쿼리 실행 속도가 더 빠르다는 사실은 SQL을 사용하는 개발자라면 익히 알고 있는 상식이다. 하지만 이런 상식이 때때로 아래 두가지 형태의 잘못된 모델링을 설계하는 원인이 되기도 한다.
* 하나의 테이블을 비슷한 이름을 가진 여러 작은 테이블로 분할 하여 관리.
* 하나의 컬럼에 속한 데이터를 비슷한 이름을 가진 여러 작은 컬럼으로 분리하여 관리.

하지만 이러한 방식은 항상 하나의 결과를 취하기 위해 여러가지 문제를 발생시킨다. 

### 테이블 복제
데이터를 복제한 각각의 테이블에 분산 저장하기 위해서는 동일한 메타데이터를 가진 테이블을 특정 네이밍 규칙에 따라 생성해줘야한다. 아래의 예시는 `date_reported` 컬럼의 년도 정보에 따른 데이터 복제 생성 및 데이터 저장 DDL이다.

```sql
# 테이블 복제
CREATE TABLE Bugs_2008 ( . . . );
CREATE TABLE Bugs_2009 ( . . . );
CREATE TABLE Bugs_2010 ( . . . );

# 데이터 저장
INSERT INTO Bugs_2010 (..., date_reported, ...) VALUES (..., '2010-06-01', ...);
INSERT INTO Bugs_2011 (..., date_reported, ...) VALUES (..., '2011-02-20', ...);
```
위 처럼 `date_reported`의 년도 기준에 맞추어 각 해당되는 테이블에 데이터를 추가해야만 한다.

### 데이터 무결성 관리
실수로 `Bugs_2009` 테이블에 2010년도 데이터가 추가되면 어떻게 방지 할 수 있을까? 사실 이 방식에서는 제약 조건을 자동으로 선언 할 수 있는 방법은 없다. 다만 `CHECK` 제약조건을 아래 예시처럼 각 테이블에 선언 함으로써 무결성을 어느정도 보장 할 수 있다.

```sql
CREATE TABLE Bugs_2009 (
  -- other columns
  date_reported DATE CHECK (EXTRACT(YEAR FROM date_reported) = 2009)
);
CREATE TABLE Bugs_2010 (
  -- other columns
  date_reported DATE CHECK (EXTRACT(YEAR FROM date_reported) = 2010)
);
```


작성중...

> 이 글은 [SQL Antipatterns - by Bill Karwin](https://pragprog.com/titles/bksqla/sql-antipatterns/) 영문 원본의 Chapter9 를 요약한 글입니다. 자의적인 해석이 들어 간 것을 참고하셨으면 좋겠습니다.

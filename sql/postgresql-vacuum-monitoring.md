# PostgreSQL VACUUM 프로세스 모니터링하기

베큐밍(Vacuuming)은 건강하고 효율적으로 PostgreSQL 데이터베이스를 관리를 하기위해서는 반드시 필요한 요소이다. [autovacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html#AUTOVACUUM)이 설정되어 있다면 데이터베이스 자체적으로 vacuum이 관리되고 있기 때문에 언제 어떻게 동작시켜야 할 지 특별히 고민할 필요가 없다. 하지만 데이터가 계속 업데이트 되거나 삭제된다면 vacuum 스케쥴 자체가 해당 변화에 맞추어 대응이 되지 않는다. 최악인 경우에는 vacuum 프로세스 자체가 제대로 동작하지 않아 데이터베이스 퍼포먼스와 자원 관리에 치명적인 영향을 미치는 사이드 이펙트들이 발생한다. 몇 가지 주요 PostgreSQL 메트릭 및 이벤트를 모니터링하면 vacuum 프로세스가 예상대로 진행되고 있는지 확인하는 데 도움이 된다.

이 글은 PostgreSQL의 vacuum 프로세스가 왜 중요한지에 대한 몇가지 이유와 해당 VACUUM 프로세스로 인해 성능이 저해되는 몇가지 문제들을 파악하고 해결하는 방법을 알아보고자 한다.

## PostgreSQL vacuum에 대한 이해
PostgreSQL은 다중 버전 동시성 제어(multi-version concurrency control, MVCC - 동시 접근을 허용하는 데이터베이스에서 동시성을 제어하기 위해 사용하는 방법)를 사용하여 데이터가 일관성 있게 유지되고 동시성이 높게 요구되는 환경에서도 접근 가능한지 보장을 해주고 있다. 각 트랜잭션은 시작된 시점의 데이터베이스에 대한 자체 스냅샷에서 작동하므로 오래된 데이터를 즉시 삭제할 수 없다. 대신 해당 스냅샷의 데이터에 dead row 라고 표식을 남기는데, 해당 데이터는 주기적으로 돌아가는 vacuum 프로세스에서 반드시 제거되어야 한다. MVCC와 vacuum에 대한 더 자세한 내용은 [PostgreSQL monitoring guide](https://www.datadoghq.com/blog/postgresql-monitoring)을 참고하기 바란다.

Vacuum은 데이터베이스 성능 및 아래 예시와 같은 상황에 대비해 리소스를 최적화 시켜준다.

- 새로운 데이터를 저장하기 위해 dead rows에 마킹을 하여 불필요한 디스크 사용을 방지하며 순차검색 속도를 높여준다. 이는 순차검색이 dead rows를 생략하지 못하기 때문이다. [시각화 맵](https://www.postgresql.org/docs/current/storage-vm.html)을 업데이트하여 오래되거나 이미 삭제된 데이터가 포함되지 않도록 유지시켜준다. 또한 인덱스 검색 속도도 향상시켜준다.(힙에 존재하는 데이터에 접근하지 않고 인덱스에서 조회할때 마킹된 데이터를 정리해준다.)

- [트랜잭션 ID wraparound 실패](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)를 방지하는데, 이 문제는 데이터 유실 또는 새로운 트랜잭션 생성을 중단시키는 현상이 발생한다.

PostgreSQL의 내장 autovacuum 기능에는 수동 vacuum 작업보다 유용한 또 다른 기능이 있는데, 이는 주기적으로 `ANALYZE` 프로세스를 실행하여 자주 업데이트 되는 테이블들의 최신 통계를 가져와서 해당 테이블들을 접근하는 쿼리들을 최적화 시켜주는 것이다. autovacuum 자체가 원래 주기적으로 데이터베이스 내 전체 테이블에 `VACUUM`을 실행하여 위에서 언급한 유지보수 기능을 담당하고 있지만 `VACUUM` 프로세스 자체가 잘 돌아가고 있는지 확인하기 위해서는 몇가지 주요한 메트릭과 이벤트들이 있기 때문에 모니터링 하기 위해서 반드시 알아두어야 한다.
## 모니터링을 위한 PostgreSQL VACUUM 메트릭
`VACUUM`이 잘 돌아가고 있는지 확인하기 위해서 아래와 같은 지표들을 모니터링 해야한다.

- [dead rows](#dead-rows)
- [테이블 디스크 사용량](#테이블-디스크-사용량)
- [마지막으로 VACUUM 또는 AUTOVACUUM을 실시한 시각](#마지막으로-(auto)vacuum-을-수행한-시각)
- [수동/야간 VACUUM 이벤트](#VACUUM과-메트릭-상관관계)

이미 `VACUUM` 관련하여 문제들을 겪고 있다면 바로 [`VACUUM` 이 돌지 않는 이유에 대한 해결책](#VACUUM과-관련된-일반적인-문제-파악하기) 섹션으로 넘어가서 해결책을 찾아보자.

## Dead rows
PostgreSQL은 `pg_stat_user_tables` 라는 뷰 테이블을 제공해주고 있는데 이 뷰는 각 테이블(`relname`)과 해당 테이블에 얼마나 많은 죽은 로우(`n_dead_tup`)들이 발생했는지 현황을 보여준다.

```sql
SELECT relname, n_dead_tup FROM pg_stat_user_tables;

  relname  | n_dead_tup
-----------+------------
 blog_joke |    3780
 ```

각 테이블, 특히 자주 업데이트 되는 테이블의 죽은 로우 갯수를 추적하게 되면 VACUUM 프로세스가 디스크 공간을 재사용 할 수 있도록 주기적으로 돌고 있는지 확인하는데 도움이 된다.
![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-1.jpeg)

## 테이블 디스크 사용량
각 테이블에서 사용되고 있는 디스크 용량을 추적하는 것은 중요한데 이는 시간이 경과함에 따라 데이터 변경에 대한 쿼리 성능을 측정함으로써 VACUUM과 관련된 잠재적 문제를 확인하는데 유용하기 때문이다. 종종 디스크 사용량 증가는 사용자의 예상범위 내에서 이뤄져야 한다. 예를 들어 최근에 많은 양의 새로운 데이터를 테이블에 추가하게 되면 다른 테이블에 예상치 못한 디스크 증가가 일어날 수 있게 되는데 이는 해당 테이블의 VACUMM 프로세스에 문제가 발생했음을 예상 할 수 있다.

Vacuuming은 사용되지 않는 로우들을 재사용할 수 있도록 표시를 남기는데 이는 곧 VACUUM이 주기적으로 동작하지 않으면 새로 추가되는 데이터는 죽은 로우를 제거함으로써 생긴 디스크 공간을 사용하는 것이 아니라 추가적으로 디스크 공간을 사용함을 의미한다.

아래 쿼리는 어떤 테이블이 가장 많은 디스크를 사용하고 있는지 보여주고 있다.
```sql
SELECT
       relname AS "table_name",
       pg_size_pretty(pg_table_size(C.oid)) AS "table_size"
FROM
       pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C.relnamespace)
WHERE nspname NOT IN ('pg_catalog', 'information_schema') AND nspname !~ '^pg_toast' AND relkind IN ('r')
ORDER BY pg_table_size(C.oid)
DESC LIMIT 1;

   table_name    | table_size
-----------------+------------
 blog_joke       | 162 MB
(1 row)
```
![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-2.jpeg)

이를 통해 데이터베이스에서 가장 자주 업데이트가 일어나는 테이블의 살아있는 로우, 죽은 로우 및 디스크 사용량의 추이를 시각화해서 봄으로써 해당 메트릭이 예상한대로 흘러가는지 파악 할 수 있게 도와준다.

예를 들어 최근에 3만개의 죽은 로우가 발생한 테이블이 있다고 가정하자. VACUUM 프로세스를 실행하여 죽은 로우의 용량을 차후에 사용할 수 있도록 표시를 남기면 새로운 로우를 테이블에 추가하더라도 테이블 크기(디스크 사용량) 자체에 큰 변화가 없음을 알 수 있을 것이다.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-3.jpeg)

위 그래프에서 볼 수 있듯이 1만개의 로우가 테이블에 추가되었지만 해당 테이블의 디스크 사용량은 얼마 늘지 않은 것을 알 수 있다. 이는 vaccum 프로세스가 3만개의 죽은 로우들을 청소하고 저장가능한 공간으로 표식을 남겼기 때문이다.

만약 특정 테이블이 예상하지 못한 사용량 증가를 불러일으킨다면 VACUUM 프로세스가 제대로 동작하지 않았을 것이라고 추측할 수 있다. 이를 확인하기 위해 각 테이블의 마지막 vacuum 작업 시각을 질의해서 확인 해볼 수 있다.

## 마지막으로 (auto)vacuum 을 수행한 시각
내장 `pg_stat_user_tables` 뷰에서 마지막으로 vacuum 또는 autovacuum 프로세스를 성공적으로 동작한 시간을 확인할 수 있다.
```sql
SELECT relname, last_vacuum, last_autovacuum FROM pg_stat_user_tables;


               relname               |          last_vacuum          |        last_autovacuum
-------------------------------------+-------------------------------+-------------------------------
 blog_joke                           | 2018-01-23 18:03:28.498505-05 | 2018-01-18 14:56:43.060002-05
```

## VACUUM과 메트릭 상관관계
데이터베이스 내 특정 테이블들이 지속적으로 업데이트 된다면 autovacuum에 추가적으로 수동 vacuum 커맨드를 실행하여 보충해주는 것이 합리적일 것이라고 판단할 것이다. (또는 `vacuumdb` 등과 같은 커맨드를 트래픽이 적은 시간에 스케쥴링하여 실행 할 수도 있다. Postgres 9.6부터 `pg_stat_progress_vacuum` 라는 뷰가 추가되었는데 이를 통해 현재 실행중인 vacuum 상태를 확인 할 수 있다.)

이 커맨드를 추적하고 모니터링 툴에서 결과를 보기를 원한다면 PostgreSQL 데이터베이스내 다른 메트릭과 상관관계를 지정하고 vacuum 스케쥴을 `/frequency` 에서 필요한 만큼 조절 가능하다. 예를 들면 [Datadog 파이썬 라이브러리인 `dogwrap`](https://github.com/DataDog/datadogpy)을 사용하여 [psql의 VACUUM 커맨드를 래핑하여](https://docs.datadoghq.com/developers/guide/dogwrap/) vacuum 활동을 추적 할 수 있다.

```sh
dogwrap -n "Vacuuming my_table" -k $API_KEY --submit_mode all "psql -d <DATABASE> -c 'vacuum verbose my_table'"
```
위 커맨드를 사용하면 `VACUUM VERBOSE`의 결과값을 Datadog에 전송하여 아래 이미지와 같이 vacuum되는 테이블의 키 메트릭을 그래프로 표현해준다.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-4.jpeg)

위 그래프는 vacuum 이후의 테이블을 보여주는데 여기서 볼 수 있듯이 제거된 죽은 로우의 갯수를 보여주지만 테이블의 디스크 사이즈는 줄어들지 않음을 확인 할 수 있다. 이는 예상된 결과인데 왜나하면 `VACUUM`을 수행하면서 기록된 사용되지 않는 로우의 디스크 용량은 추후 데이터 저장이 필요한 경우 사용할 수 있지만 디스크 공간을 [특정 경우](https://www.postgresql.org/docs/current/routine-vacuuming.html)를 제외하곤 시스템에 돌려주지 않는다. 디스크 공간을 운영 시스템에 반환하기를 원한다면 [VACUUM FULL](https://www.postgresql.org/docs/current/routine-vacuuming.html) 커맨드를 실행해야 한다. 하지만 이 커맨드는 해당 테이블에 배타적 잠금이 필요한 높은 I/O를 수행하는 작업이다. `VACUUM FULL`의 이러한 리소스 요구조건 때문에 PostgreSQL에서는 보통 정기적 autovacuum을 더 나은 데이터베이스 유지보수 작업 선택지로 추천한다.

## VACUUM과 관련된 일반적인 문제 파악하기

메트릭에서 PostgreSQL 테이블 중에서 VACUUM이 제대로 수행되지 않는 테이블이 발견되면 몇가지 세팅값과 데이터베이스 메트릭을 참고하여 어떤 문제가 발생했는지 조사 해 볼 수 있다. 이번 섹션에서는 데이터베이스가 PostgreSQL 문서에서 권장하는 빌트인 autovacuum 기능을 사용한다고 가정하고 한번 알아보고자 한다.

해당 문제는 아래와 같은 몇가지 원인으로 인해 autovacuum 프로세스가 지연되거나 전혀 실행되지 않는다. 문제는 아래와 같다.

<!-- no toc -->
- [데이터베이스에서 autovacuum 프로세스가 비활성화 되어 있는 경우](#autovacuum이-잘-수행되고-있는가?)
- [특정 테이블에서만 autovacuum 프로세스가 비활성화 되어있는 경우](#특정-테이블에서만-autovacuum-프로세스가-비활성화-되어있는-경우)
- [Autovacuum 설정이 업데이트 속도를 따라가지 못하는 경우](#Autovacuum-설정이-업데이트-속도를-따라가지-못하는-경우)
- [잠금 충돌](#VACUUM-프로세스가-잠금-충돌되는-경우)
- [오랜 기간동안 열려있는 트랜잭션](#오랜-기간동안-열려있는-트랜잭션)


### autovacuum이 잘 수행되고 있는가?
autovacuum이 주기적으로 잘 수행되고 있지만 `last_autovacuum` ([위 쿼리 참조](#마지막으로-(auto)vacuum-을-수행한-시각))이 예상과 다르게 나온다면 autovacuum 프로세스가 동작하고 있는지 아래 커맨드를 사용하여 확인 해볼 수 있다.

```sh
ps -axww | grep autovacuum
```

여기서 autovacuum 프로세스가 동작하는지 여부에 따라 다음 두가지 조치를 취할 수 있다. Autovacuum 프로세스가 동작하지 않는다면 [Autovacuum 프로세스가 동작하지 않는 경우](#Autovacuum-프로세스가-동작하지-않는-경우) 를 참고하고 아니라면 아래 따로 언급할 섹션을 참고하자.

### Autovacuum 프로세스가 동작하지 않는 경우
Autovacuum이 동작하지 않는 것 처럼 보인다면 PostgreSQL 세팅에서 autovacuum 플래그가 `on` 으로 되어있는지 확인해보아야 한다. 기본값으로 autovacuum은 이미 `on` 처리 되어있지만 검증 차원에서 한번 더 확인해보자.

```sql
SELECT name, setting FROM pg_settings WHERE name='autovacuum';
    name    | setting
------------+---------
 autovacuum | on
(1 row)
```

세팅값에서 autovacuum이 켜져 있지만 프로세스가 서버에서 동작하지 않는 경우 [statistics collector(통계 수집기)](https://www.postgresql.org/docs/current/monitoring-stats.html)의 문제일수도 있다. Autovacuum은 얼마나 자주 동작해야하는지 여부를 통계 수집기를 통해 의존한다. 기본적으로 통계 수집기는 이미 켜져 있어야하지만 만약 이게 꺼져 있다면 통계 수집기는 autovacuum 데몬 프로세스에 실시간 데이터베이스 활동에 관한 정보를 전달할 수 없다.

통계 수집기가 켜져있는지 여부는 **postgresql.conf** 설정 파일의 "런타임 통계" 섹션에서 `track_counts` 가 켜져있는지 확인하거나 아래와 같이 질의해서 확인할 수 있다.

```sql
SELECT name, setting FROM pg_settings WHERE name='track_counts';

     name     | setting
--------------+---------
 track_counts | on
(1 row)
```

`track_counts`가 꺼져있다면 통계 수집기는 각 테이블의 죽은 로우들의 갯수를 업데이트하지 않게 된다. 이는 autovacuum 데몬 프로세스가 해당 테이블에 autovacuum을 언제 돌릴것인지 판단하는 지표이기 때문에 당연히 동작하지 않게 된다.

이 경우 [참고 링크](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE)에 나와있는데로 PostgreSQL 설정 파일을 수정하여 `autovacuum`과 `track_counts`를 활성화 시킬 수 있다.

### 특정 테이블에서만 autovacuum 프로세스가 비활성화 되어있는 경우
데이터베이스에서 autovacuum 프로세스가 활성화 되어있는 것 처럼 보이지만 VACUUM이 실행되었어야 할 테이블에 프로세스가 동작하지 않았다면 autovacuum이 특정 설정으로 인해 특정 테이블에서만 수행되지 않았을 가능성이 높다.

이 경우 `pg_class`를 질의하여 해당 테이블에 autovacuum이 활성화 되어있는지 확인 할 수 있다.

```sql
SELECT reloptions FROM pg_class WHERE relname='my_table';
         reloptions
----------------------------
 {autovacuum_enabled=false}
(1 row)
```
이 예시에서는 `my_table` 에만 autovacuum이 비활성화 되어있는 것으로 확인된다. 이 경우 아래 업데이트 쿼리를 실행하여 다시 활성화 시켜줘야한다.

```sql
ALTER TABLE my_table SET (autovacuum_enabled = true);
```

또한 `reloptions` 질의를 모든 테이블에 실행하여 확인 할 수 있다.

```sql
SELECT relname, reloptions FROM pg_class;
```
다른 트러블슈팅 단계를 하기 전에 반드시 모든 테이블에 autovacuum이 활성화 되어있는지 확인을 해보아야한다.

### Autovacuum 설정이 업데이트 속도를 따라가지 못하는 경우
데이터베이스 전체에 autovacuum 설정이 활성화 되어있어도 VACUUM 프로세스가 원하는 주기에 충분히 동작하지 않는 것처럼 보인다면 기본 설정을 변경하고 싶을 것이다. Autovacuum 데몬 프로세스는 VACUUM 과 ANALYZE 커맨드를 자동실행하기 위해 몇가지 환경 설정을 해줘야 한다. 이러한 설정들을 보고싶다면 `pg_settings`에 다음과 같이 질의하면 된다.
```sql
SELECT * from pg_settings where category like 'Autovacuum';

                name                 |  setting  | unit |  category  |                                        short_desc                                         | extra_desc |  context   | vartype | source  | min_val |  max_val   | enumvals | boot_val  | reset_val | sourcefile | sourceline
-------------------------------------+-----------+------+------------+-------------------------------------------------------------------------------------------+------------+------------+---------+---------+---------+------------+----------+-----------+-----------+------------+------------
 autovacuum                          | on        |      | Autovacuum | Starts the autovacuum subprocess.                                                         |            | sighup     | bool    | default |         |            |          | on        | on        |            |
 autovacuum_analyze_scale_factor     | 0.1       |      | Autovacuum | Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples. |            | sighup     | real    | default | 0       | 100        |          | 0.1       | 0.1       |            |
 autovacuum_analyze_threshold        | 50        |      | Autovacuum | Minimum number of tuple inserts, updates, or deletes prior to analyze.                    |            | sighup     | integer | default | 0       | 2147483647 |          | 50        | 50        |            |
 autovacuum_freeze_max_age           | 200000000 |      | Autovacuum | Age at which to autovacuum a table to prevent transaction ID wraparound.                  |            | postmaster | integer | default | 100000  | 2000000000 |          | 200000000 | 200000000 |            |
 autovacuum_max_workers              | 3         |      | Autovacuum | Sets the maximum number of simultaneously running autovacuum worker processes.            |            | postmaster | integer | default | 1       | 8388607    |          | 3         | 3         |            |
 autovacuum_multixact_freeze_max_age | 400000000 |      | Autovacuum | Multixact age at which to autovacuum a table to prevent multixact wraparound.             |            | postmaster | integer | default | 10000   | 2000000000 |          | 400000000 | 400000000 |            |
 autovacuum_naptime                  | 60        | s    | Autovacuum | Time to sleep between autovacuum runs.                                                    |            | sighup     | integer | default | 1       | 2147483    |          | 60        | 60        |            |
 autovacuum_vacuum_cost_delay        | 20        | ms   | Autovacuum | Vacuum cost delay in milliseconds, for autovacuum.                                        |            | sighup     | integer | default | -1      | 100        |          | 20        | 20        |            |
 autovacuum_vacuum_cost_limit        | -1        |      | Autovacuum | Vacuum cost amount available before napping, for autovacuum.                              |            | sighup     | integer | default | -1      | 10000      |          | -1        | -1        |            |
 autovacuum_vacuum_scale_factor      | 0.2       |      | Autovacuum | Number of tuple updates or deletes prior to vacuum as a fraction of reltuples.            |            | sighup     | real    | default | 0       | 100        |          | 0.2       | 0.2       |            |
 autovacuum_vacuum_threshold         | 50        |      | Autovacuum | Minimum number of tuple updates or deletes prior to vacuum.                               |            | sighup     | integer | default | 0       | 2147483647 |          | 50        | 50        |            |
(11 rows)
```
`setting` 컬럼은 현재 설정된 값을 보여주며 `boot_val` 컬럼은 해당 설정의 기본값을 보여준다.(PostgreSQL에 아무런 설정을 하지 않았다면 해당 값을 사용한다.) 해당 설정에 대한 모든 상세한 설명은 [여기 문서](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html)에서 확인 할 수 있다.

위 설정을 좀 더 들여다 보면 autovacuum 데몬의 동작 주기를 결정하는 요소들을 확인 할 수 있다.

- `autovacuum_vacuum_threshold` (50, 기본값)
- `autovacuum_vacuum_scale_factor`: (0.2, 기본값)
- 예상 로우 갯수([`pg_class.reltuples`](https://www.postgresql.org/docs/current/catalog-pg-class.html) 기반)
Autovacuum 데몬은 해당 변수를 autovacuum 임계치에 지정하기 위해 공식에 반영한다.(VACUUM 프로세스를 자동으로 트리거 하는 죽은 로우 갯수를 모니터링 한다.)

```sql
autovacuuming threshold = autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor * estimated number of rows in the table)
```

위 설정값을 조정하면 autovacuum이 충분히 주기적으로 돌아가는지 보장 할 수 있게 된다. 예를 들어 `autovacuum_vacuum_scale_factor` 값을 줄이게 되면 autovacuum 프로세스를 좀 더 자주 트리거하게 된다. 이처럼 다른 설정값들을 변경하는 경우 더 큰 스케일의 환경에 적용하기 전에 충분한 테스트와 모니터링을 통해 어떤 영향을 미치는지 확인해야 한다.

또 다른 설정값은 `log_autovacuum_min_duration`인데, 이 설정은 autovacuum 프로세스가 특정 시간(밀리세컨드)을 초과하는 경우 해당 활동을 로그로 남기는 역할을 한다. 이 설정은 느리게 수행되는 autovacuum 프로세스에 대해 가시성을 확보하여 특정 세팅값을 변경하여 성능을 효율화 시키는데 도움을 준다.

### VACUUM 프로세스가 잠금 충돌되는 경우
Autovacuum 설정을 확인했을 때 정확하게 세팅되어 있다면 VACUUM이 테이블 배타적 잠금(Exclusive lock, write lock이라고도 불린다. 어떤 트랜잭션에서 데이터를 변경하고자 할 때(ex . 쓰고자 할 때) 해당 트랜잭션이 완료될 때까지 해당 테이블 혹은 레코드(row)를 다른 트랜잭션에서 읽거나 쓰지 못하게 잠금을 거는 트랜잭션)과 같은 충돌이 발생하여 중단이 될 수도 있다. 해당 테이블에 VACUUM 작업을 재개하기 위해서 VACUUM 프로세스는 `SHARE UPDATE EXCLUSIVE` 잠금 권한을 획득해야 한다. 이 권한은 같은 종류의 다른 잠금과 충돌이 일어나는 경우(두 트랜잭션이 동시에 `SHARE UPDATE EXCLUSIVE` 잠금을 가지고 있을 수 없음) 이에 대해 접근하기 위함이다. 해당 권한을 획득하면 `SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE` 을 가지게 된다. 결국 특정 트랜잭션이 앞에 언급한 잠금 중 하나가 테이블에 발생하면 해당 VACUUM은 잠금이 풀리기 전까지 실행 되지 않기 때문에 SHARE UPDATE EXCLUSIVE 잠금 권한을 획득해야 한다.

하나의 `psql` 세션에서 많은 양의 로우를 업데이트하고 테이블 내 `ACCESS EXCLUSIVE` 잠금이 필요한 `ALTER TABLE` 커맨드를 실행해보자.

```sql
BEGIN;
UPDATE something SET number = number + 1;
ALTER TABLE something RENAME COLUMN number TO id;
```

이제 다른 `psql` 세션을 열어 테이블 `VACUUM`을 동작시키려고 시도해보자.

```
VACUUM something;
```

이 경우에는 vacuum 프로세스가 실행되지 않는데 이는 테이블 잠금 권한을 획득하지 못했기 때문이다. 첫번째 트랜잭션에서 이미 `ACCESS EXCLUSIVE` 잠금을 테이블에 걸었기 때문에 VACUUM을 실행하기 위해 `SHARE UPDATE EXCLUSIVE` 잠금과 충돌되기 때문이다.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-5.jpeg)

`htop`이나 다른 실시간 프로세스 모니터링 툴에서 VACUUM 프로세스가 대기 상태인것을 확인 할 수 있다.

```sh
ps -ef | grep 'waiting'

postgres 17358 25147  0 11:15 ?        00:00:00 postgres: emily djangodogz 127.0.0.1(37844) VACUUM waiting
```
![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-6.jpeg)

PostgreSQL 로그에서 vacuum 프로세스가 왜 대기 상태인지 좀 더 자세히 확인할 수 있다. 이는 아래 사진에 보이듯이 필요한 테이블 잠금 권한을 획득하지 못했기 때문이다.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-7.jpeg)

특정 트랜잭션이 vacuum 프로세스와 충돌을 일으키게 되는 테이블 레벨 배타적 잠금으로 잠기게 되기 때문에 DB 클라이언트에서 트랜잭션을 필요 이상으로 열어두지 않도록 하는 것이 좋다. 이 경우 트랜잭션을 커밋하지 않고 열린상태로 유지하게 되면 테이블은 절대 vacuum 되지 않기 때문이다.

### 오랜 기간동안 열려있는 트랜잭션
MVCC 또다른 부작용은 하나 또는 그 이상의 트랜잭션이 오래된 버전의 데이터를 아직 필요로 하는 경우(예: 하나 또는 그 이상의 열린 트랜잭션이 데이터가 업데이트 또는 사용되지 않게 표식 되기 이전에 오래된 스냅샷의 데이터를 가져와 사용하는 경우) vacuum 프로세스가 죽은 로우들을 제대로 제거하지 못한다는 것이다. 그렇기 때문에 트랜잭션은 필요 이상으로 오래 유지되거나 아이들(idle) 되지 않는 대신 정확한 시간 내에 커밋되고 종료되는 것이 중요하다.

`pg_stat_activity` 뷰에 질의하면 어떤 커넥션이 [“트랜잭션 아이들”](https://www.postgresql.org/docs/current/monitoring-ps.html) 상태인지 확인 할 수 있는데 이 상태는 세션을 맺었지만 아무런 동작을 하지 않음을 의미한다.

```sql
SELECT xact_start, state, usename FROM pg_stat_activity;


          xact_start           |        state        | usename
-------------------------------+---------------------+----------
 2018-01-24 15:59:18.651181-05 | idle in transaction | emily
```
기본적으로 PostgreSQL은 자동으로 현재 열린 세션들을 추적할 수 있도록 정보들을 제공하지만 위 쿼리의 출력이 비활성화 된 것으로 표시된다면 `postgresql.conf` 파일에서 [`track_activities` 설정을 활성화](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE) 해야한다.

해당 쿼리 결과값을 자동으로 수집하여 모니터링 플랫폼에 제공 할 수 있는데 이렇게 하면 해당 메트릭을 시간 경과에 따라 그래프로 볼 수 있고 많은 양의 트랜잭션이 idle 상태로 있음을 감지하면 알림을 보낼 수 있다. Datadog으로 이러한 설정을 하고 싶다면 [Part 3](https://www.datadoghq.com/blog/collect-postgresql-data-with-datadog)을 참고하여 설정하면 된다.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-8.jpeg)

트랜잭션의 상태를 예의주시 함으로써 idle 트랜잭션이 테이블의 VACUUM을 하지 못하도록 방해하는 동작을 방지 할 수 있다. PostgreSQL 9.6 에서 [idle_in_transaction_session_timeout](https://www.postgresql.org/docs/10/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT) 이라는 설정값이 추가되었는데 이를 이용하면 해당 설정 시간(밀리세컨드로 표시) 이상의 세션이 살아있다면 자동으로 종료시켜준다. 또한 `statement_timeout` 설정을 추가하여 세션별 제한 시간을 둘 수도 있다. 이 설정은 오랜 시간동안 열려있는 트랜잭션을 일정 시간이 지나면 자동으로 종료시켜버리가 때문에 VACUUM 프로세스가 테이블 내 idle 상태인 트랜잭션 때문에 제대로 수행되지 않는 문제를 미연에 방지할 수 있다.

## PostgreSQL VACUUM 모니티링과 그 이외에 추가 내용
PostgreSQL VACUUM 프로세스는 건강하고 효율적인 데이터베이스를 유지보수 하기위한 한가지 조건이다. 데이터베이스 상태와 성능을 포괄적으로 관리 할 수 있는 뷰가 필요하다면 다른 환경들과 마찬가지로 모든 데이터베이스 인스턴스에 주요 메트릭, 분산된 요청 추적 및 로그들을 모니터링 해야한다.

Original Source:
[Monitoring PostgreSQL VACUUM processes](https://www.datadoghq.com/blog/postgresql-vacuum-monitoring/)

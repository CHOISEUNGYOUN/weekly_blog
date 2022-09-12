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
- [테이블 디스크 사용량](#table-disk-usage)
- [마지막으로 VACUUM 또는 AUTOVACUUM을 실시한 시각](#last-time-autovacuum-ran)
- [수동/야간 VACUUM 이벤트](#correlating-vacuums-with-metrics)

이미 `VACUUM` 관련하여 문제들을 겪고 있다면 바로 [`VACUUM` 이 돌지 않는 이유에 대한 해결책](#investigating-common-vacuum-related-issues) 섹션으로 넘어가서 해결책을 찾아보자.

## Dead rows
PostgreSQL offers a `pg_stat_user_tables` view that provides a breakdown of each table (`relname`) and how many dead rows (`n_dead_tup`) are in that table:

```sql
SELECT relname, n_dead_tup FROM pg_stat_user_tables;

  relname  | n_dead_tup
-----------+------------
 blog_joke |    3780
 ```
Tracking the number of dead rows in each table—particularly tables that are frequently updated—will help you determine if VACUUM processes are effectively removing them periodically so that their disk space can be reused.
![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-1.jpeg)


PostgreSQL live rows in Datadog
## Table disk usage
Tracking the amount of disk space used by each table is important because it enables you to gauge expected changes in query performance over time—but it can also help you detect potential vacuuming-related issues. Sometimes increased disk usage may align with your expectations—for example, if you’ve recently added a lot of new data to that table. But if you see an unexpected increase in any particular table’s disk usage, it may indicate that there is a problem with vacuuming that table.

Vacuuming helps mark outdated rows as available for reuse, so if VACUUMs are not running regularly, then newly added data will use additional disk space, instead of reusing the disk space taken up by dead rows.

The following query shows you the table that is using the most disk space in your database:
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

It may be helpful to graph the number of live rows, dead rows, and disk usage across each of the most constantly updated tables in your database, to see if the metrics align with your expectations.

For example, let’s say you have a table that currently has about 30,000 dead rows. You run a VACUUM process to mark the dead rows as available for future storage. Then, if you add new rows to the table, you shouldn’t see much (if any) increase in that table’s size (disk usage).

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-3.jpeg)

Indeed, the graphs indicate that even though we inserted about 10,000 new rows into our table, the amount of disk being used by that table did not increase much, because the vacuuming process cleaned up about 30,000 dead rows and marked them as available for storage.

If you see the size of any PostgreSQL tables increasing unexpectedly, VACUUM processes may not be executing properly on that table. To confirm if that’s the case, we can query the database to determine the last time each of our tables was vacuumed.

## Last time (auto)vacuum ran
The built-in view pg_stat_user_tables enables you to find out the last time a vacuuming or autovacuuming process successfully ran on each of your tables:
```sql
SELECT relname, last_vacuum, last_autovacuum FROM pg_stat_user_tables;


               relname               |          last_vacuum          |        last_autovacuum
-------------------------------------+-------------------------------+-------------------------------
 blog_joke                           | 2018-01-23 18:03:28.498505-05 | 2018-01-18 14:56:43.060002-05
```

## Correlating VACUUMs with metrics
If certain tables in your database are constantly updated, you may find that it makes sense to supplement autovacuuming with manual VACUUM commands (or schedule them using something like `vacuumdb`) during low-traffic periods of time. As of version 9.6, PostgreSQL also includes VACUUM progress reporting via a view called `pg_stat_progress_vacuum`, which you can use to track the real-time status of your VACUUM commands.

If you track this command and its ensuing output in a monitoring platform, you can correlate it with other metrics across your PostgreSQL database and tweak your vacuuming schedule/frequency as needed. For example, you could track vacuuming activity by using the [Datadog Python library’s `dogwrap`](https://github.com/DataDog/datadogpy) utility to [wrap your psql VACUUM command](https://docs.datadoghq.com/developers/guide/dogwrap/):

```sh
dogwrap -n "Vacuuming my_table" -k $API_KEY --submit_mode all "psql -d <DATABASE> -c 'vacuum verbose my_table'"
```
This sends the output of the VACUUM VERBOSE command to Datadog, which you can overlay on graphs that display key metrics from the vacuumed table.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-4.jpeg)

PostgreSQL VACUUM monitoring in DatadogAfter running a VACUUM process on a table (overlaid in purple on each graph), the number of dead rows in that table dropped to 0, but the table's disk usage (table size) remained the same.
It looks like after we vacuumed this table, the number of dead rows dropped, but the size (disk usage) of the table did not decrease. This is expected behavior, because VACUUMs mark outdated rows as available for future data storage, but they rarely (except in [certain cases](https://www.postgresql.org/docs/current/routine-vacuuming.html)) return disk space to the system. If you actually need to return the disk space to your operating system, you’ll have to execute a [VACUUM FULL](https://www.postgresql.org/docs/current/routine-vacuuming.html)). However, this is typically an I/O-intensive process that requires an exclusive lock on the table, which can block queries and degrade performance. Because of the resource requirements of VACUUM FULL, PostgreSQL generally recommends regular autovacuuming as a better database maintenance option.

## Investigating common VACUUM-related issues
If metrics indicate that your VACUUMs are not running properly across your PostgreSQL tables, you can investigate the cause of the issue by querying a few settings and metrics across your database. In this section, we will assume that you are using PostgreSQL’s built-in autovacuuming feature, as recommended in the PostgreSQL documentation.

There are a variety of possible reasons why autovacuuming processes could be stalling, or unable to run at all, including:

- [The autovacuum process is disabled on your database](#is-autovacuum-running)
- [The autovacuum process is disabled on one or more tables](#autovacuum-may-be-disabled-on-certain-tables)
- [Autovacuuming settings aren’t keeping pace with updates](#autovacuuming-settings-arent-keeping-pace-with-updates)
- [Lock conflicts](#vacuums-are-running-into-lock-conflicts)
- [Long-running open transactions](#long-running-open-transactions)

### Is autovacuum running?
If you expect autovacuuming to run regularly, but the time of the `last_autovacuum` ([queried above](#last-time-autovacuum-ran)) does not match your expectations, check that the autovacuum process is running:

```sh
ps -axww | grep autovacuum
```

We have two possible paths we could investigate, depending on whether the process is actually running or not. Read [the next section](#autovacuum-process-is-not-running) if autovacuum is not running; otherwise, skip ahead to our other suggested solutions.

### Autovacuum process is not running
If it doesn’t look like autovacuum is running, we should verify that it’s actually been enabled in our PostgreSQL settings. By default, autovacuuming should already be turned on, but let’s double check:

```sql
SELECT name, setting FROM pg_settings WHERE name='autovacuum';
    name    | setting
------------+---------
 autovacuum | on
(1 row)
```

If it looks like autovacuuming has been enabled in your settings, but the process is not running on your server, it could be due to a problem with the [statistics collector](https://www.postgresql.org/docs/current/monitoring-stats.html). Autovacuuming relies on the statistics collector to determine when and how often it should run. By default, the statistics collector should already be enabled, but if it has been disabled, the statistics collector won’t able to provide the autovacuum daemon with information about real-time database activity.

You can check if the statistics collector is enabled by consulting the “Runtime Statistics” section of your **postgresql.conf** configuration file to see if track_counts is on, or by running the following query:

```sql
SELECT name, setting FROM pg_settings WHERE name='track_counts';

     name     | setting
--------------+---------
 track_counts | on
(1 row)
```

If `track_counts` is off, the statistics collector won’t update the count of the number of dead rows for each table, which is the value that the autovacuum daemon checks in order to determine when and where it needs to run.

You can enable `autovacuuming` and `track_counts` by editing these settings in your PostgreSQL configuration file, [as described in the documentation](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE).

### Autovacuum may be disabled on certain tables
If it looks like the autovacuum process is running in your database, but it hasn’t launched on a table that received a lot of updates that should have triggered a VACUUM, it’s possible that autovacuuming was disabled at some point on the table(s) in question.

We can query `pg_class` to check if autovacuum is enabled on this table:

```sql
SELECT reloptions FROM pg_class WHERE relname='my_table';
         reloptions
----------------------------
 {autovacuum_enabled=false}
(1 row)
```
In this example, it looks like autovacuuming is disabled on `my_table`. We can re-enable it by running:

```sql
ALTER TABLE my_table SET (autovacuum_enabled = true);
```

You can also view the `reloptions` for every table in your database by querying:

```sql
SELECT relname, reloptions FROM pg_class;
```
Before proceeding with the other troubleshooting steps listed below, make sure that autovacuum has been enabled on all of your desired tables.

### Autovacuuming settings aren’t keeping pace with updates
If autovacuuming is enabled everywhere throughout your database, but you believe that it is not triggering VACUUM processes on your tables frequently enough, you may want to tweak the default configuration settings. The autovacuum daemon relies on a number of configuration settings to determine when it should automatically run VACUUM and ANALYZE commands on your databases. You can view the current and default settings by querying `pg_settings`:

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
The `setting` column shows the currently configured value, while the `boot_val` column shows the default value for this setting (which is what PostgreSQL will use if you don’t specify otherwise). You can find the full descriptions of these settings [in the documentation](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html).

More specifically, let’s focus on the key factors that determine how frequently the autovacuum daemon will run a VACUUM command on any given table:

- `autovacuum_vacuum_threshold` (50, by default)
- `autovacuum_vacuum_scale_factor`: (0.2, by default)
- the estimated number of rows in the table (based on [`pg_class.reltuples`](https://www.postgresql.org/docs/current/catalog-pg-class.html))
The autovacuum daemon plugs these variables into a formula to determine the autovacuuming threshold (the number of dead rows that would automatically trigger a VACUUM process):

```sql
autovacuuming threshold = autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor * estimated number of rows in the table)
```

Adjusting these settings can help ensure that autovacuuming is running frequently enough to keep up with demand. For example, reducing the `autovacuum_vacuum_scale_factor` can cause the autovacuuming process to trigger VACUUMs more frequently. As with any other configuration change, make sure to thoroughly test and observe the impacts of any changes before you implement them on a larger scale.

Another informative setting is `log_autovacuum_min_duration`, which will log any autovacuuming activity after the process exceeds this amount of time (measured in milliseconds). This can help provide more visibility into slow autovacuum processes so that you can determine if you need to tweak certain settings to optimize performance.

### VACUUMs are running into lock conflicts
If you’ve already ensured that your autovacuuming settings are configured correctly, your VACUUMs could be stalling due to conflicting, exclusive locks on tables. In order to run on a table, a VACUUM process needs to acquire a SHARE UPDATE EXCLUSIVE lock, which conflicts with other locks of the same kind (two transactions cannot hold a SHARE UPDATE EXCLUSIVE lock at the same time), as well as the following lock modes: SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, and ACCESS EXCLUSIVE. Therefore, if any transactions hold one of these locks on a table, VACUUM cannot execute on that table until the other lock is released, so that it can acquire the SHARE UPDATE EXCLUSIVE lock that it needs.

In one `psql` session, let’s update a bunch of rows, and then issue an ALTER TABLE command, which requires an ACCESS EXCLUSIVE lock on the table:

```sql
BEGIN;
UPDATE something SET number = number + 1;
ALTER TABLE something RENAME COLUMN number TO id;
```

Now, let’s open another `psql` session and try running a VACUUM on the table:

```
VACUUM something;
```

In this case, the vacuuming process stalls because it cannot obtain a lock on the table. Our first transaction (which is still open) already holds an ACCESS EXCLUSIVE lock on the table, which conflicts with the SHARE UPDATE EXCLUSIVE lock that a VACUUM requires in order to run.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-5.jpeg)

In `htop`, or any other real-time process-monitoring tool, we can see that the VACUUM process is waiting:

```sh
ps -ef | grep 'waiting'

postgres 17358 25147  0 11:15 ?        00:00:00 postgres: emily djangodogz 127.0.0.1(37844) VACUUM waiting
```
![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-6.jpeg)

The PostgreSQL logs tell us even more details about why the vacuuming process was waiting—it was unable to obtain the necessary lock on the table:

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-7.jpeg)

Because certain transactions may hold exclusive table-level locks that conflict with the vacuuming process, it’s generally a good idea to ensure that clients aren’t leaving transactions open longer than needed—in this case, if we were to keep this transaction open without committing it, this table would never get vacuumed.

### Long-running open transactions
Another side effect of MVCC is that the vacuuming process cannot clean up dead rows if one or more transactions still need to access the outdated version of the data (e.g., one or more open transactions are operating on a snapshot of the data that was taken before the data was updated/became outdated). Therefore, it’s important to make sure that transactions are committed/closed instead of staying open longer than necessary, or idle.

Querying the `pg_stat_activity` view can help confirm if any connections are in a state of [“idle in transaction,”](https://www.postgresql.org/docs/current/monitoring-ps.html) meaning they have begun a session but are not actively doing any work:

```sql
SELECT xact_start, state, usename FROM pg_stat_activity;


          xact_start           |        state        | usename
-------------------------------+---------------------+----------
 2018-01-24 15:59:18.651181-05 | idle in transaction | emily
```
By default, PostgreSQL will automatically track information about currently open sessions, but if the output of the query above shows that the state is disabled, you need to [enable the `track_activities` setting](https://www.postgresql.org/docs/current/config-setting.html#CONFIG-SETTING-CONFIGURATION-FILE) in your postgresql.conf file.

You can automatically collect the results of this query and forward them to a monitoring platform, which enables you to graph this metric over time and set up alerts to detect when a large number of transactions have gone idle. For more details on how to set this up in Datadog, see [Part 3](https://www.datadoghq.com/blog/collect-postgresql-data-with-datadog) of our PostgreSQL monitoring guide.

![img](/sql/imgs/postgresql-vacuum-monitoring/postgresql-vacuum-monitoring-8.jpeg)


Keeping an eye on the state of your transactions can help ensure that idle transactions aren’t preventing VACUUMs from running on tables. In version 9.6, a setting called [idle_in_transaction_session_timeout](https://www.postgresql.org/docs/10/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT) was added, which automatically terminates any session that surpasses this number of milliseconds. You can also set a `statement_timeout` that’s limited to a specific session. This setting determines how long to wait before automatically terminating long-running transactions, which can help prevent the scenario where VACUUM processes are having trouble running on tables with idle transactions.

## Monitoring PostgreSQL VACUUMs and more
PostgreSQL VACUUM processes are just one aspect of maintaining a healthy, efficient database. In order to gain a comprehensive view of your database’s health and performance, you’ll need to monitor key metrics, distributed request traces, and logs from all of your database instances—as well as the rest of your environment.

As your application updates or deletes data in a PostgreSQL database, Datadog can help you correlate all of those queries with performance indicators (e.g., request latency) and infrastructure metrics (e.g., dead rows and disk usage). In the flame graph below, you can see that this request spent almost 30 percent of its time accessing the PostgreSQL service. You can also inspect individual queries and correlate them with host-level metrics and logs to get deep visibility into database health and the overall performance of your application.

Original Source:
[Monitoring PostgreSQL VACUUM processes](https://www.datadoghq.com/blog/postgresql-vacuum-monitoring/)

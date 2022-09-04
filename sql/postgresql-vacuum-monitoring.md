# Monitoring PostgreSQL VACUUM processes

Vacuuming is a necessary aspect of maintaining a healthy and efficient PostgreSQL database. If you have [autovacuuming](https://www.postgresql.org/docs/current/routine-vacuuming.html#AUTOVACUUM) configured, you usually don’t need to think about how and when to execute PostgreSQL VACUUMs at all—the whole process is automatically handled by the database. However, if you are constantly updating or deleting data, vacuuming schedules may not be able to keep up with the pace of those changes. Even worse, vacuum processes may not be running at all, which can lead to a number of side effects that negatively impact database performance and resource usage. Monitoring a few key PostgreSQL metrics and events will help you ensure that vacuum processes are proceeding as expected.

This article will provide some background on why vacuuming is important in PostgreSQL, and explore a few ways to investigate and resolve issues that prevent VACUUMs from running efficiently.

## PostgreSQL vacuuming overview
PostgreSQL uses multi-version concurrency control (MVCC) to ensure that data remains consistent and accessible in high-concurrency environments. Each transaction operates on its own snapshot of the database at the point in time it began, which means that outdated data cannot be deleted right away. Instead, it is marked as a dead row, which must be cleaned up through a routine process known as vacuuming. For more information about MVCC and vacuuming, read our [PostgreSQL monitoring guide](https://www.datadoghq.com/blog/postgresql-monitoring).

Vacuuming helps optimize database performance and resource usage by:

- marking dead rows as available to store new data, which helps prevent unnecessary disk usage, and also helps speed up sequential scans, which cannot skip over dead rows.
updating a [visibility map](https://www.postgresql.org/docs/current/storage-vm.html), which keeps track of pages that don’t contain any outdated/deleted data. This helps speed up index-only scans (by making it apparent when you can serve the data directly from the index, without having to access the data from the heap).

- [preventing transaction ID wraparound failure](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND), which could lead to data loss, or bring the database to a halt by blocking any new transactions.

PostgreSQL’s built-in autovacuuming feature provides another benefit over manual vacuuming: It periodically runs an ANALYZE process to gather the latest statistics about frequently updated tables, which enables the query planner to optimize its plans. Although autovacuuming is designed to periodically execute VACUUM commands across your databases in order to carry out the maintenance tasks listed above, you should still monitor a handful of key metrics and events to ensure that your VACUUM processes aren’t running into any hiccups along the way.

## PostgreSQL VACUUM metrics to monitor
In order to ensure that VACUUMs are running smoothly across your databases, you should monitor:

- [dead rows](#dead-rows)
- [table disk usage](#table-disk-usage)
- [the last time a VACUUM or AUTOVACUUM ran](#last-time-autovacuum-ran)
- manual/nightly VACUUM events

If you already know that VACUUMs are experiencing issues, skip straight down to the suggested solutions to find out why your VACUUMs may not be running.

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
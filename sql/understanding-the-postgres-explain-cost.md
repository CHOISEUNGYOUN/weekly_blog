# PostgreSQL EXPLAIN – 쿼리 비용(Query Cost)란 무엇인가?
## PostgreSQL EXPLAIN 비용 이해하기
`EXPLAIN` 은 Postgres 쿼리의 성능을 측정하는데 아주 유용한 명령어이다. 실행하면 PostgreSQL 쿼리 실행계획서가 실행 계획을 반한한다. 이 `EXPLAIN` 명령어는 테이블을 참조하는 쿼리문이 인덱스 또는 순차 검색을 하는지 자세하게 설명한다.

`EXPLAIN`을 사용하여 검토를 할 때 가장 먼저 눈여겨 볼 점은 비용 통계인데 이것이 어떤 의미인지, 어떻게 계산되는지, 어떻게 사용되는지 의문을 갖는것은 자연스러운 현상이다.

간단히 말해서 PostgreSQL 실행계획서는 해당 쿼리가 얼마나 많은 시간을 요구하는지(임의 값 기준) 시작 비용과 각 실행 단계별 총 비용을 합하여 계산한다. 추가 내용은 아래에서 설명하겠다. 쿼리를 실행하는데 여러 옵션들이 함께 사용되는 경우 옵션들 중에서 가장 비용이 저렴한 선택지를 고르게 된다.

## 비용을 측정하는 단위는 무엇인가?
비용은 임의 값 기준으로 산정된다. 이를 밀리세컨드(ms) 나 다른 시간 단위로 계산된다고 오해하는데, 이는 틀린 말이다. 비용 단위는 기본적으로 순차적으로 페이지를 읽는데 걸리는 시간을 1.0 으로 일컫는다(`seq_page_cost`). 각 로우가 수행될때 마다 `cpu_tuple_cost`가 0.01 이 추가되며 비순차적으로 페이지를 읽게되면 4.0 `random_page_cost`가 추가된다. 이와 비슷한 상수들이 많이 존재하지만 모두 설정 가능하다. 마지막에 언급한 단위가 지금 사용하고 있는 하드웨어에서 특히 보편적으로 사용되는 단위이다. 이에 대한 내용은 나중에 좀 더 다루려고 한다.

## 시작 비용
`cost=`뒤에 바로 보이는 숫자는 흔히 "시작 비용(startup cost)" 이라고 불린다. 이는 첫번째 로우를 가져오는데 걸리는 시간을 의미한다. 이와 같이 쿼리 수행의 시작 비용은 서브쿼리에서 실행하는 비용을 포함한다.

순차 검색에서는 첫번째 로우를 바로 가져오기 때문에 시작 비용이 0에 가깝다. 정렬을 수행할 때에는 시작 비용이 좀 더 올라가는데 그 이유는 첫번째 로우를 가져오기 전에 정렬 작업을 거치기 때문이다.

예시를 들기위해 먼저 1000개의 유저명을 가진 테이블을 생성해보자.

```postgresql
CREATE TABLE users (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username text NOT NULL);
INSERT INTO users (username)
SELECT 'person' || n
FROM generate_series(1, 1000) AS n;
ANALYZE users;
```
이제 몇가지 작업을 포함한 실행 계획을 살펴보자.

```postgresql
EXPLAIN SELECT * FROM users ORDER BY username;
 
|QUERY PLAN                                                    |
|--------------------------------------------------------------+
|Sort  (cost=66.83..69.33 rows=1000 width=17)                  |
|  Sort Key: username                                          |
|  ->  Seq Scan on users  (cost=0.00..17.00 rows=1000 width=17)|
```

예상했듯이 위 쿼리 계획에서는 순차 검색에 대한 실행 예상 비용이 `0.00`, 비용으로 `66.83`이 출력된다.

## Total costs
The second cost statistic, after the startup cost and the two dots, is known as the “total cost”. This is an estimate of how long it will take to return all the rows.

Let’s look at that example query plan again:

```postgresql
|QUERY PLAN                                                    |
|--------------------------------------------------------------+
|Sort  (cost=66.83..69.33 rows=1000 width=17)                  |
|  Sort Key: username                                          |
|  ->  Seq Scan on users  (cost=0.00..17.00 rows=1000 width=17)|
```

We can see that the total cost of the `Seq Scan` operation is `17.00`. For the `Sort` operation is `69.33`, which is not much more than its startup cost (as expected).

Total costs usually include the cost of the operations preceding them. For example, the total cost of the `Sort` operation above includes that of the `Seq Scan`.

An important exception is `LIMIT` clauses, which the planner uses to estimate whether it can abort early. If it only needs a small number of rows, the conditions for which are common, it may calculate that a simpler scan choice is cheaper (likely to be faster).

For example:

```postgresql
EXPLAIN SELECT * FROM users LIMIT 1;
 
|QUERY PLAN                                                    |
|--------------------------------------------------------------+
|Limit  (cost=0.00..0.02 rows=1 width=17)                      |
|  ->  Seq Scan on users  (cost=0.00..17.00 rows=1000 width=17)|
```

As you can see, the total cost reported on the `Seq Scan` node is still `17.00`, but the full cost of the `Limit` operation is reported to be `0.02`. This is because the planner expects that it will only have to process 1 row out of 1000, so the cost, in this case, is estimated to be 1000th of the total.

## How the costs are calculated
In order to calculate these costs, the Postgres query planner uses both constants (some of which we’ve already seen) and metadata about the contents of the database. The metadata is often referred to as “statistics”.

Statistics are gathered via `ANALYZE` (not to be confused with the `EXPLAIN` parameter of the same name), and stored in `pg_statistic`. They are also refreshed automatically as part of autovacuum.

These statistics include a number of very useful things, like roughly the number of rows each table has, and what the most common values in each column are.

Let’s look at a simple example, using the same query data as before:

```postgresql
EXPLAIN SELECT count(*) FROM users;
 
|QUERY PLAN                                                   |
|-------------------------------------------------------------+
|Aggregate  (cost=19.50..19.51 rows=1 width=8)                |
|  ->  Seq Scan on users  (cost=0.00..17.00 rows=1000 width=0)|
```

In our case, the planner’s statistics suggested the data for the table was stored within 7 pages (or blocks), and that 1000 rows would be returned. The cost parameters `seq_page_cost`, `cpu_tuple_cost`, and `cpu_operator_cost` were left at their defaults of `1`, `0.01`, and `0.0025` respectively.

As such, the Seq Scan total cost was calculated as:

```postgresql
|Total cost of Seq Scan
|= (estimated sequential page reads * seq_page_cost) + (estimated rows returned * cpu_tuple_cost)
|= (7 * 1) + (1000 * 0.01)
|= 7 + 10.00
|= 17.00
```

And for the Aggregate as:

```postgresql
|Total cost of Aggregate
|= (cost of Seq Scan) + (estimated rows processed * cpu_operator_cost) + (estimated rows returned * cpu_tuple_cost)
|= (17.00) + (1000 * 0.0025) + (1 * 0.01) 
|= 17.00 + 2.50 + 0.01
|= 19.51
```

## How the planner uses the costs
Since we know Postgres will pick the query plan with the lowest total cost, we can use that to try to understand the choices it has made. For example, if a query is not using an index that you expect it to, you can use settings like `enable_seqscan` to massively discourage certain query plan choices. By this point, you shouldn’t be surprised to hear that settings like this work by increasing the costs!
Row numbers are an extremely important part of cost estimation. They are used to calculate estimates for different join orders, join algorithms, scan types, and more. Row cost estimates that are out by a lot can lead to cost estimation being out by a lot, which can ultimately result in a suboptimal plan choice being made.

## Using EXPLAIN ANALYZE to get a query plan
When you write SQL statements in PostgreSQL, the `ANALYZE` command is key to optimizing queries, making them faster and more efficient. In addition to displaying the query plan and PostgreSQL estimates, the `EXPLAIN ANALYZE` option performs the query (be careful with `UPDATE` and `DELETE`!), and shows the actual execution time and row count number for each step in the execution process. This is necessary for monitoring SQL performance.

You can use `EXPLAIN ANALYZE` to compare the estimated number of rows with the actual rows returned by each operation.

Let’s look at an example, using the same data again:

```postgresql
|QUERY PLAN
|-------------------------------------------------------------------------+
|Sort (cost=66.83..69.33 rows=1000 width=17) (actual time=20.569..20.684 rows=1000 loops=1)
|  Sort Key: username
|  Sort Method: quicksort  Memory: 102kB
|  ->  Seq Scan on users (cost=0.00..17.00 rows=1000 width=17) (actual time=0.048..0.596 rows=1000 loops=1)
|Planning Time: 0.171 ms
|Execution Time: 20.793 ms
```

We can see that the total execution cost is still 69.33, with the majority of that being the Sort operation, and 17.00 coming from the Sequential Scan. Note that the query execution time is just under 21ms.

### Sequential scan vs. Index Scan
Now, let’s add an index to try to avoid that costly sort of the entire table:

```postgresql
|​​CREATE INDEX people_username_idx ON users (username);
|
|EXPLAIN ANALYZE SELECT * FROM users ORDER BY username;
|
|QUERY PLAN
|-------------------------------------------------------------------------+
|Index Scan using people_username_idx on users  (cost=0.28..28.27 rows=1000 width=17) (actual time=0.052..1.494 rows=1000 loops=1)
|Planning Time: 0.186 ms
|Execution Time: 1.686 ms
```

As you can see, the query planner has now chosen an Index Scan, since the total cost of that plan is 28.27 (lower than 69.33). It looks that the index scan was more efficient than the sequential scan, as the query execution time is now just under 2ms.

## Helping the planner estimate more accurately
We can help the planner estimate more accurately in two ways:

1. Help it gather better statistics
2. Tune the constants it uses for the calculations

The statistics can be especially bad after a big change to the data in a table. As such, when loading a lot of data into a table, you can help Postgres out by running a manual `ANALYZE` on it. Statistics also do not persist over a major version upgrade, so that’s another important time to do this.

Naturally, tables also change over time, so tuning the autovacuum settings to make sure it runs frequently enough for your workload can be very helpful.

If you’re having trouble with bad estimates for a column with a skewed distribution, you may benefit from increasing the amount of information Postgres gathers by using the `ALTER TABLE SET STATISTICS` command, or even the `default_statistics_target` for the whole database.

Another common cause of bad estimates is that, by default, Postgres will assume that two columns are independent. You can fix this by asking it to gather correlation data on two columns from the same table via extended statistics.

On the constant tuning front, there are a lot of parameters you can tune to suit your hardware. Assuming you’re running on SSDs, you’ll likely at minimum want to tune your setting of `random_page_cost`. This defaults to 4, which is 4x more expensive than the `seq_page_cost` we looked at earlier. This ratio made sense on spinning disks, but on SSDs it tends to penalize random I/O too much. As such a setting closer to 1, or between 1 and 2, might make more sense. At ScaleGrid, we default to 1.

## Can I remove the costs from query plans?
For many of the reasons mentioned above, most people leave the costs on when running EXPLAIN. However, should you wish, you can turn them off using the COSTS parameter.

```postgresql
EXPLAIN (COSTS OFF) SELECT * FROM users LIMIT 1;
 
|QUERY PLAN             |
|-----------------------+
|Limit                  |
|  ->  Seq Scan on users|
```

## Conclusion
To re-cap, the costs in query plans are Postgres’ estimates for how long an SQL query will take, in an arbitrary unit.

It picks the plan with the lowest overall cost, based on some configurable constants and some statistics it has gathered.

Helping it estimate these costs more accurately is very important to help it make good choices, and keep your queries performant.

Original Source:
[Understanding the Postgres EXPLAIN cost](https://scalegrid.io/blog/postgres-explain-cost/)
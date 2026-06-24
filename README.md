# Assignment2 ADB
## 1. The issues that were identified.

This report provides a detailed performance diagnostic and optimization strategy for the top candidate queries identified via `pg_stat_statements`. Each query analysis breaks down execution plan metrics, identifies hidden bottlenecks, and provides precise, production-ready engineering decisions.

First five candidetes for optimization by mean_exec_time with this query(filtered with calls because there was no big mean time with calls less than 3):

```sql
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC, mean_exec_time desc, calls desc
LIMIT 100;
```

1.

```sql
SELECT
                customer_id,
                event_type,
                COUNT(*) AS events_count,
                MAX(event_time) AS last_event_time
            FROM customer_events_wide
            WHERE event_time >= NOW() - INTERVAL $1
            GROUP BY customer_id, event_type
            ORDER BY events_count DESC
            LIMIT $2
```

explain analyze result

```sql
Limit  (cost=12487.39..12487.89 rows=200 width=27) (actual time=82.373..82.401 rows=200 loops=1)
  ->  Sort  (cost=12487.39..12495.68 rows=3316 width=27) (actual time=82.372..82.393 rows=200 loops=1)
        Sort Key: (count(*)) DESC
        Sort Method: top-N heapsort  Memory: 39kB
        ->  Finalize GroupAggregate  (cost=11901.71..12344.07 rows=3316 width=27) (actual time=78.639..81.333 rows=3627 loops=1)
"              Group Key: customer_id, event_type"
              ->  Gather Merge  (cost=11901.71..12280.97 rows=2994 width=27) (actual time=78.605..80.430 rows=3674 loops=1)
                    Workers Planned: 2
                    Workers Launched: 2
                    ->  Partial GroupAggregate  (cost=10901.68..10935.37 rows=1497 width=27) (actual time=61.829..62.337 rows=1225 loops=3)
"                          Group Key: customer_id, event_type"
                          ->  Sort  (cost=10901.68..10905.43 rows=1497 width=19) (actual time=61.801..61.859 rows=1234 loops=3)
"                                Sort Key: customer_id, event_type"
                                Sort Method: quicksort  Memory: 126kB
                                Worker 0:  Sort Method: quicksort  Memory: 57kB
                                Worker 1:  Sort Method: quicksort  Memory: 59kB
                                ->  Parallel Seq Scan on customer_events_wide  (cost=0.00..10822.73 rows=1497 width=19) (actual time=0.039..50.340 rows=1234 loops=3)
                                      Filter: (event_time >= (now() - '7 days'::interval))
                                      Rows Removed by Filter: 65432
Planning Time: 0.131 ms
Execution Time: 82.452 ms

```

The database executes a Parallel Seq Scan across the entire customer_events_wide table. For a 7-day lookback window, it processes and subsequently discards **65,432 rows** via a filter condition, spending 50.340 ms just on physical disk reads.

**Indexing over Partitioning:** At the current data volume, implementing table partitioning for `customer_events_wide` introduces unnecessary architectural complexity and query planner overhead. An index is highly efficient and perfectly adequate for the current scale. However, as historical event data grows into millions of rows, implementing range partitioning on `event_time` (e.g., daily or weekly partitions) should be revisited as a future-proof strategy to maintain optimal index sizes and enable instant data pruning.

### Final Engineering Decisions

1. **In the future we should implement Table Partitioning:** Since this is a high-volume event/log table, partition `customer_events_wide` by range on the `event_time` column (e.g., daily or monthly partitions). This ensures the engine completely ignores historical chunks of data and scans only the active time-slice. 
2. **Deploy a Tailored Composite Index:** To eliminate the sorting overhead and accelerate the filter, create a composite B-Tree index:

| **Metric** | **Before Optimization** | with composite index |
| --- | --- | --- |
| **Execution Time** | 82.452 ms | 17.821 ms |
| **Rows Filtered (Waste)** | 65,432 | - |

2.

```sql
SELECT COUNT(*) FROM order_items WHERE quantity > $1
```

explain analyze result

```sql
Finalize Aggregate  (cost=6099.37..6099.38 rows=1 width=8) (actual time=29.763..29.776 rows=1 loops=1)
  ->  Gather  (cost=6099.26..6099.37 rows=1 width=8) (actual time=29.758..29.773 rows=2 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        ->  Partial Aggregate  (cost=5099.26..5099.27 rows=1 width=8) (actual time=20.287..20.287 rows=1 loops=2)
              ->  Parallel Seq Scan on order_items  (cost=0.00..4939.93 rows=63731 width=0) (actual time=0.011..18.361 rows=53912 loops=2)
                    Filter: (quantity > 7)
                    Rows Removed by Filter: 126080
Planning Time: 0.645 ms
Execution Time: 30.189 ms

```

 The engine falls back to a Parallel Seq Scan on order_items, processing 180,000 records total and removing **126,080 rows** via the Filter: (quantity > 7).

the amount of filtered rows depends on quantity number placed in where condition, when it is 7 50% can be filtered, when it is 1 only 5%.  But because of the fact that database will not jump between index table and data retrieving, and the index cover all what we need for query the fact of the row that meet condition.

### Final Engineering Decisions

1. **Deploy a Dedicated B-Tree Index:** When the index ON order_items (quantity) is in place, PostgreSQL can completely bypass loading the underlying table data blocks from disk. Instead, it will perform an `Index Only Scan`, traversing the lean index structure to isolate and count the target pointers directly, converting a `30 ms` operation into a near-instantaneous sub-millisecond lookup.

| **Metric** | **Before Optimization** | with index |
| --- | --- | --- |
| **Execution Time** | 30.189 ms | `10.853 ms` |
| **Rows Filtered (Waste)** | 126,080 | 0 |

It is important that if the `quantity > 7` condition returns a large percentage of the table (e.g., >30%), the PostgreSQL planner may intentionally bypass the index. In such high-selectivity scenarios, reverting to a Sequential Scan is structurally cheaper than incurring the random I/O overhead associated with massive index lookups.

3.

```sql
SELECT COUNT(*)
            FROM customers c
            JOIN orders o ON o.customer_id = c.customer_id
            JOIN customer_events_wide e ON e.customer_id = c.customer_id
            WHERE c.status IN ($1, $2)
              AND e.event_time >= NOW() - INTERVAL $3
```

explain analyze:

```sql
Finalize Aggregate  (cost=12906.04..12906.05 rows=1 width=8) (actual time=52.490..53.924 rows=1 loops=1)
  ->  Gather  (cost=12905.82..12906.03 rows=2 width=8) (actual time=52.261..53.920 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate  (cost=11905.82..11905.83 rows=1 width=8) (actual time=39.493..39.495 rows=1 loops=3)
              ->  Nested Loop  (cost=661.63..11897.45 rows=3348 width=0) (actual time=39.491..39.492 rows=0 loops=3)
                    ->  Hash Join  (cost=661.34..11514.90 rows=531 width=8) (actual time=39.491..39.492 rows=0 loops=3)
                          Hash Cond: (e.customer_id = c.customer_id)
                          ->  Parallel Seq Scan on customer_events_wide e  (cost=0.00..10849.33 rows=1611 width=4) (actual time=0.028..27.027 rows=1230 loops=3)
                                Filter: (event_time >= (now() - '7 days'::interval))
                                Rows Removed by Filter: 65437
                          ->  Hash  (cost=579.00..579.00 rows=6587 width=4) (actual time=11.776..11.776 rows=6587 loops=3)
                                Buckets: 8192  Batches: 1  Memory Usage: 296kB
                                ->  Seq Scan on customers c  (cost=0.00..579.00 rows=6587 width=4) (actual time=0.053..3.523 rows=6587 loops=3)
                                      Filter: (status = 'inactive'::text)
                                      Rows Removed by Filter: 13413
                    ->  Index Only Scan using idx_orders_customer_id on orders o  (cost=0.29..0.66 rows=6 width=4) (never executed)
                          Index Cond: (customer_id = c.customer_id)
                          Heap Fetches: 0
Planning Time: 0.463 ms
Execution Time: 54.650 ms
```

need the index on the event_time or partitioning in customer_events_wide table and on status in customers table

so much time spent on the parallel scan on customer_events_wide e to filter by event_time 27 milliseconds, so we should create index on this column or/and partition on that column. 

on status scanning database spend only 3 milliseconds, so patitioning or/and index also can speed up.

### Final Engineering Decisions

1. **Optimize Filter Anchors:** Create supporting indexes on the filtering predicates of the outer tables to prevent them from reading unneeded disk pages:

idx_customers_status ON customers (status);
idx_customer_events_time ON customer_events_wide (event_time);

| **Metric** | **Before Optimization** | with cover index from query1 | with index on status |
| --- | --- | --- | --- |
| **Execution Time** | 54.650 ms | `5.339 ms` | `6.484 ms` |
| **Rows Filtered (Waste)** | 78,850 *(Total)* | `13413` | 0 |

so because of the optimisation by index idx_customer_events_time_group there is no need to create another index idx_customer_events_time proposed by me before, it gives no optimisation in time.

4.

```sql
SELECT *
            FROM orders
            WHERE delivery_city LIKE $1
              AND status = $2
```

explain analyze

```sql
Seq Scan on orders  (cost=0.00..3023.00 rows=1764 width=49) (actual time=0.033..12.105 rows=1736 loops=1)
  Filter: ((delivery_city ~~ 'North%'::text) AND (status = 'shipped'::text))
  Rows Removed by Filter: 118264
Planning Time: 0.106 ms
Execution Time: 12.206 ms
```

The database uses a full `Seq Scan` to inspect all `120,000` rows within `orders`. It extracts a mere `1,736` rows while discarding **118,264 records** via the filter. Reading the entire table for a result set representing less than 1.5% of the total volume is highly inefficient.

### Final Engineering Decisions

1. **Deploy a Pattern-Optimized Composite Index:** Standard B-Tree indexes cannot properly interpret prefix wildcard matching (`LIKE 'North%'`) under default non-C locales. A dedicated operator class must be specified:

```sql
CREATE INDEX idx_orders_status_city ON orders (status, delivery_city varchar_pattern_ops);
```

Placing `status` first provides an exact-match equality anchor (`=`). Appending `delivery_city` with `varchar_pattern_ops` allows the B-Tree structure to map preﬁx ranges natively, completely eliminating the full `Seq Scan` and reducing block access to only matching rows.

| **Metric** | **Before Optimization** | **After Optimization** |
| --- | --- | --- |
| **Execution Time** | 12.206 ms | 1.992 ms |
| **Rows Filtered (Waste)** | 118,264 | 0 |

5.

```sql
SELECT
                p.category,
                COUNT(*) AS items_sold,
                SUM(oi.quantity * oi.unit_price) AS revenue
            FROM order_items oi
            JOIN products p ON oi.product_id = p.product_id
            GROUP BY p.category
            ORDER BY revenue DESC
```

explain analyze result:

```sql
Sort  (cost=8681.40..8681.41 rows=5 width=47) (actual time=24.951..25.908 rows=0 loops=1)
  Sort Key: (sum(((oi.quantity)::numeric * oi.unit_price))) DESC
  Sort Method: quicksort  Memory: 25kB
  ->  Finalize GroupAggregate  (cost=8680.65..8681.34 rows=5 width=47) (actual time=24.940..25.896 rows=0 loops=1)
        Group Key: p.category
        ->  Gather Merge  (cost=8680.65..8681.23 rows=5 width=47) (actual time=24.939..25.895 rows=0 loops=1)
              Workers Planned: 1
              Workers Launched: 1
              ->  Sort  (cost=7680.64..7680.65 rows=5 width=47) (actual time=16.926..16.927 rows=0 loops=2)
                    Sort Key: p.category
                    Sort Method: quicksort  Memory: 25kB
                    Worker 0:  Sort Method: quicksort  Memory: 25kB
                    ->  Partial HashAggregate  (cost=7680.52..7680.58 rows=5 width=47) (actual time=16.747..16.748 rows=0 loops=2)
                          Group Key: p.category
                          Batches: 1  Memory Usage: 24kB
                          Worker 0:  Batches: 1  Memory Usage: 24kB
                          ->  Hash Join  (cost=66.00..5033.58 rows=211755 width=17) (actual time=16.745..16.746 rows=0 loops=2)
                                Hash Cond: (oi.product_id = p.product_id)
                                ->  Parallel Seq Scan on order_items oi  (cost=0.00..4410.55 rows=211755 width=14) (actual time=0.010..6.895 rows=179992 loops=2)
                                ->  Hash  (cost=41.00..41.00 rows=2000 width=11) (actual time=0.426..0.426 rows=2000 loops=2)
                                      Buckets: 2048  Batches: 1  Memory Usage: 104kB
                                      ->  Seq Scan on products p  (cost=0.00..41.00 rows=2000 width=11) (actual time=0.021..0.153 rows=2000 loops=2)
Planning Time: 1.141 ms
Execution Time: 26.171 ms

```

The query is designed to calculate historical metrics across 100% of rows in both `order_items` (`179,992` records) and `products` (`2,000` records). Standard indexes cannot improve this query because every row must be processed.

### Final Engineering Decisions

1. **Implement a Materialized View:** Because the runtime is directly tied to total data volume, standard indexes offer no remedy. The computation should be offloaded into a cache. To maintain database availability, the view must be updated asynchronously using `REFRESH MATERIALIZED VIEW CONCURRENTLY`, scheduled strictly during off-peak hours.

| **Metric** | **Before Optimization** | select materealized view |
| --- | --- | --- |
| **Execution Time** | 26.171 ms | 0.024 ms |
| **Rows Filtered (Waste)** | 0 *(100% scan)* | 0 |

6.

```sql
SELECT
                c.customer_id,
                c.full_name,
                COUNT(o.order_id) AS orders_count,
                SUM(o.total_amount) AS revenue
            FROM customers c
            JOIN orders o ON c.customer_id = o.customer_id
            WHERE c.status = $1
            GROUP BY c.customer_id, c.full_name
            ORDER BY revenue DESC
            LIMIT $2
```

explain analyze result:

```sql
Limit  (cost=3953.43..3953.48 rows=20 width=58) (actual time=16.016..16.018 rows=0 loops=1)
  ->  Sort  (cost=3953.43..3969.90 rows=6587 width=58) (actual time=16.015..16.016 rows=0 loops=1)
        Sort Key: (sum(o.total_amount)) DESC
        Sort Method: quicksort  Memory: 25kB
        ->  HashAggregate  (cost=3695.82..3778.16 rows=6587 width=58) (actual time=15.992..15.993 rows=0 loops=1)
              Group Key: c.customer_id
              Batches: 1  Memory Usage: 217kB
              ->  Hash Join  (cost=661.34..3399.40 rows=39522 width=28) (actual time=15.970..15.972 rows=0 loops=1)
                    Hash Cond: (o.customer_id = c.customer_id)
                    ->  Seq Scan on orders o  (cost=0.00..2423.00 rows=120000 width=14) (actual time=0.017..5.781 rows=120000 loops=1)
                    ->  Hash  (cost=579.00..579.00 rows=6587 width=18) (actual time=3.268..3.269 rows=6587 loops=1)
                          Buckets: 8192  Batches: 1  Memory Usage: 388kB
                          ->  Seq Scan on customers c  (cost=0.00..579.00 rows=6587 width=18) (actual time=0.234..2.778 rows=6587 loops=1)
                                Filter: (status = 'inactive'::text)
                                Rows Removed by Filter: 13413
Planning Time: 0.384 ms
Execution Time: 16.303 ms

```

The engine performs a full `Seq Scan` across `120,000` records in `orders` (`5.781 ms`), blindly scanning historical system data just to match records for a small subset of inactive users.

 Finding inactive clients (`status = 'inactive'`) removes **13,413 rows** via a sequential scan on `customers`.

### Final Engineering Decisions

1. **Enforce Foreign Key & Dimension Indexes:** To shift the planner's strategy away from full table scans and towards a precise, highly selective execution path, implement indexes:

idx_customers_status ON customers (status)

idx_orders_customer_id ON orders (customer_id);

This pair of indexes shifts the optimal join strategy from a broad `Hash Join` to a targeted `Nested Loop Join`. The database will first pinpoint the subset of target customers via `idx_customers_status`, and then perform swift, pinpoint index lookups into `orders` via the foreign key index, avoiding the need to process all 120,000 orders.

| **Metric** | **Before Optimization** | with status index | with index customer_id |
| --- | --- | --- | --- |
| **Execution Time** | 16.303 ms | 15.322 ms | 15.322 ms |
| **Rows Filtered (Waste)** | 133,413 *(Total)* | 0 | 0 |

Although the planner currently chooses a hash join due to the high sampling percentage (index tipping point), the idx_orders_customer_id index has been retained as it is necessary for scalability and ad hoc OLTP queries.

There are also DML queries that give load on DB:

```sql
UPDATE test_lock SET val = $1 WHERE id = $2;
```

!Screenshot 2026-06-24 at 16.08.44.png

```sql
UPDATE customer_events_wide
SET attr_01 = $1 || NOW()::TEXT,
attr_02 = $2,
attr_03 = $3,
attr_04 = $4
WHERE customer_id = $5
```

!Screenshot 2026-06-24 at 16.09.17.png

```sql
UPDATE customers
SET phone = $1 || NOW()::TEXT
WHERE customer_id = $2
```

!Screenshot 2026-06-24 at 16.10.00.png

```sql
INSERT INTO customer_events_wide
(
customer_id, event_type, event_time, source, campaign,
device, browser, os, ip_address, page_url, referrer,
utm_source, utm_medium, utm_campaign,
attr_01, attr_02, attr_03, attr_04, attr_05,
attr_06, attr_07, attr_08, attr_09, attr_10
)
VALUES (
$1, $2, $3::timestamp, $4, $5,
$6, $7, $8, $9, $10, $11,
$12, $13, $14,
$15, $16, $17, $18, $19,
$20, $21, $22, $23, $24
)
```

!Screenshot 2026-06-24 at 16.10.27.png

```sql
INSERT INTO order_items
(order_id, product_id, quantity, unit_price)
VALUES ($1, $2, $3, $4)
```

!Screenshot 2026-06-24 at 16.11.14.png

```sql
INSERT INTO orders
(customer_id, order_date, status, total_amount, payment_method, delivery_city)
VALUES ($1, $2::timestamp, $3, $4, $5, $6)
```

!Screenshot 2026-06-24 at 16.12.37.png

### Issue 1: High-Frequency Single-Row Operations (Write Amplification)

**Empirical Evidence (`pg_stat_statements`):**

- `INSERT INTO order_items` -> 720,107 calls
- `UPDATE customers` -> 1,176,818 calls

**Analysis:**

Row-by-row execution in loops creates severe network and I/O bottlenecks due to constant Write-Ahead Log (WAL) disk flushes. Furthermore, the massive volume of single-row `UPDATE` queries causes heavy MVCC overhead, forcing PostgreSQL to continuously write new data pages and rewrite indexes.

**Resolution Strategy:**

1. **Application-Level Batching:** Refactor the codebase to utilize Bulk Inserts (grouping 1,000+ rows per statement). This cuts network round-trips and WAL sync operations by 99.9%.
2. **Asynchronous Commit:** Apply `SET synchronous_commit = off;` for ingestion pipelines. This allows PostgreSQL to acknowledge writes from RAM instantly without waiting for physical disk flushes, massively accelerating bulk inserts (viable if a sub-second data loss during a hard server crash is acceptable).
3. **Enable HOT Updates:** Execute `ALTER TABLE customers SET (fillfactor = 90);`. Leaving 10% free space on data pages allows PostgreSQL to update rows "in place" without rewriting indexes, drastically reducing I/O overhead.

### Issue 2: `Idle in Transaction` Bottlenecks & Lock Contention

- **Empirical Evidence (`pg_stat_activity`):**

| **pid** | **usename** | **state** | **query** |
| --- | --- | --- | --- |
| 42893 | darynaburdak | idle in transaction | `UPDATE customers SET status = 'active' WHERE customer_id = 3;` |
| 43096 | darynaburdak | idle in transaction | `UPDATE customers SET city = city WHERE customer_id = 4;` |
- **Root Cause Analysis:** Transactions opened by user `darynaburdak` issued `UPDATE` statements but remained uncommitted (`idle in transaction`). While in this state, PostgreSQL indefinitely holds an exclusive row-level lock (`RowExclusiveLock`) on `customer_id = 3` and `customer_id = 4`. This completely blocks any concurrent application threads trying to modify these customers, spiking queue duration. Furthermore, it holds back the global `xmin` horizon, preventing `VACUUM` from purging dead tuples and causing severe table bloat.
- **Resolution Strategy:** 1. Strict application-layer control ensuring all code blocks wrapped in transactions guarantee a `COMMIT` or `ROLLBACK` in a `finally` block.
2. Implement database-level safety rails: `SET idle_in_transaction_session_timeout = '10s';` to automatically kill abandoned connections and release acquired locks.

### Issue 3: Live Lock Contention and Table-Level Blocking (Bonus Section Evidence)

**Detection Methodology & Empirical Evidence:**

Monitoring active sessions via `pg_stat_activity` combined with `pg_blocking_pids()` exposed an explicit block contention scenario on the `orders` table during concurrent workload execution.

*Lock Tree Mapping:*

- **Blocking PID 22705** (User: darynaburdak) executed: `LOCK TABLE orders IN SHARE ROW EXCLUSIVE MODE`
- **Blocked PID 22708** (User: darynaburdak) executed: `SET lock_timeout = '20s'; ...`

!Screenshot 2026-06-21 at 16.39.03.png

*Active Wait State Metrics:*

- `wait_event_type`: Lock
- `wait_event`: relation (Table-level lock constraint)
- `running_for`: 11.11 seconds (Active accumulated delay)

**Root Cause Analysis:**

The severe latency spike was caused by Transaction PID 22705 acquiring a heavy `SHARE ROW EXCLUSIVE` lock on the entire `orders` table. It is crucial to note that such high-level table locks do not occur naturally from standard DML operations (like regular `INSERT` or `UPDATE` queries). This explicit `LOCK TABLE` command was deliberately executed by the load-generator script to simulate a severe contention scenario for testing purposes.

Because of this intentional roadblock, the operational process (PID 22708) attempting normal DML operations was forced into a synchronous wait state, experiencing an artificial latency spike of over 11 seconds. This introduces an immediate risk of connection pool exhaustion and transaction timeout exceptions once the 20-second threshold is crossed.

**Resolution Strategy:**

1. **Ban Explicit Table Locks:** Completely avoid explicit table-level locking (`LOCK TABLE`) within application workflows.
2. **Use Fine-Grained Locking:** Shift to low-impact row-level locks (`SELECT ... FOR UPDATE`) or application-level optimistic lock controls.
3. **Minimize Lock Duration:** Decouple heavy transaction initializations from the active DB connection lifecycle. Ensure all transactions altering mission-critical tables are short-lived, committing immediately after execution to keep lock hold durations strictly under sub-millisecond limits.

## To check:

**Ensure Visibility Map Accuracy for Index-Only Scans:** While the new B-Tree index enables a highly efficient Index-Only Scan, its actual runtime performance relies entirely on an up-to-date **Visibility Map**. Given the massive volume of concurrent `INSERT` operations on the `order_items` table (over 720,000 calls observed), the default global `autovacuum` schedule will lag behind. An outdated Visibility Map forces PostgreSQL to perform expensive **Heap Fetches** to verify MVCC row visibility, completely negating the performance benefits of the Index-Only Scan.

To prevent this we must configure aggressive, table-specific autovacuum thresholds:

```sql
ALTER TABLE order_items SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_vacuum_insert_scale_factor = 0.05
);
```

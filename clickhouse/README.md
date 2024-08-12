# cluster 啟動
```
docker-compose up -d
```

# cluster 設定
包含
- 資料 + 副本 (node1)
- 資料 + 副本 (node2)
- 資料 + 副本 (node3)
- 資料 + 副本 (node3)
- clickhouse-keeper-01
- clickhouse-keeper-02
- clickhouse-keeper-03

## clickhouse-server

### remote_server.xml (群集設定)
- 設定群集分片與副本  
- 一個分片建議建立2個以上副本,以維持高可用 (4node - 2shard - 4 replica)

### use-keeper.xml (keeper設定)
- keeper用於仲裁資料如何分配與取得

## clickhouse-keeper

## DDL操作集群(ON CLUSTER)

### 0. 啟動 clickhouse-client
```shell
clickhouse-client
```

### 1. 檢查cluster是否有建起來
```
SHOW CLUSTERS
```

```
result: 會出現建立的force_cluster

   ┌─cluster───────┐
1. │ default       │
2. │ force_cluster │
   └───────────────┘
```

### 2. 檢查cluster主機狀態
``` sql
SELECT *
FROM system.clusters
WHERE cluster = 'force_cluster';
```

### 3. 在集群建立資料庫 "db1"
``` sql
CREATE DATABASE IF NOT EXISTS db1 ON CLUSTER force_cluster;
```
> 建議用IF NOT EXISTS , ON CLUSTER 會在集群的所有節點執行CREATE DATABASE

```
result: 會在集群的兩個實體建立"db1"

   ┌─host──────────┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
1. │ clickhouse-01 │ 9000 │      0 │       │                   1 │                0 │
2. │ clickhouse-02 │ 9000 │      0 │       │                   0 │                0 │
   └───────────────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘
```

### 4. 在集群建立資料表 "table1"

``` sql
CREATE TABLE IF NOT EXISTS db1.table1 ON CLUSTER force_cluster
(
    `id` UInt64,
    `column1` String
)
ENGINE = MergeTree
ORDER BY id
```
### 5. 在子節點建立資料
```sql
--- 在 node 1 insert
INSERT INTO db1.table1 (id, column1) VALUES (1, 'abc');

--- 在 node 2 insert
INSERT INTO db1.table1 (id, column1) VALUES (2, 'def');

--- 連線到子節點, 可以查詢到各子節點的資料
```
### ６.建立分散式表(Distributed),從節點取得資料
``` sql
CREATE TABLE IF NOT EXISTS db1.table1_dist ON CLUSTER force_cluster
(
    `id` UInt64,
    `column1` String
)
ENGINE = Distributed('force_cluster', 'db1', 'table1', rand())
```
### 7.查詢分散式表
```
SELECT * FROM db1.table1_dist;
```
```
result:集群節點的所有資料

   ┌─id─┬─column1─┐
1. │  2 │ def     │
   └────┴─────────┘
   ┌─id─┬─column1─┐
2. │  1 │ abc     │
   └────┴─────────┘

```

## 新增Table欄位
1. 新增群集所有欄位
``` sql
ALTER TABLE table1 ON CLUSTER force_cluster ADD COLUMN IF NOT EXISTS created_date DateTime;
```

2. 更新分布表
``` sql
ALTER TABLE table1_dist ON CLUSTER force_cluster ADD COLUMN IF NOT EXISTS created_date DateTime;
```

## 寫入Distributed表
1. 建立一個分布表 , sharding_key , 用年月 , 相同年月的資料會放到同一個shard內
``` sql
CREATE TABLE IF NOT EXISTS db1.table1_ym_dist ON CLUSTER force_cluster
AS db1.table1
ENGINE = Distributed('force_cluster', 'db1', 'table1', toYYYYMM(created_date))
```
2. 寫入資料到分布表
``` sql
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (1, 'aaa' , '2024-08-01');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (1, 'aaa' , '2024-09-02');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (1, 'aaa' , '2024-08-03');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (2, 'bbb' , '2024-08-01');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (2, 'bbb' , '2024-09-02');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (2, 'bbb' , '2024-11-03');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (3, 'bbb' , '2024-08-14');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (3, 'bbb' , '2024-07-14');
INSERT INTO db1.table1_ym_dist (id, column1, created_date) VALUES (3, 'bbb' , '2024-09-14');

3. 檢查各節點資料 , 相同年月資料會在同一個節點
``` sql
SELECT * FROM db1.table1 order by created_date

   ┌─id─┬─column1─┬────────created_date─┐
1. │  2 │ 2       │ 1970-01-01 00:00:00 │
2. │  1 │ aaa     │ 2024-08-01 00:00:00 │
3. │  2 │ bbb     │ 2024-08-01 00:00:00 │
4. │  1 │ aaa     │ 2024-08-03 00:00:00 │
5. │  3 │ bbb     │ 2024-08-14 00:00:00 │
6. │  2 │ bbb     │ 2024-11-03 00:00:00 │
   └────┴─────────┴─────────────────────┘

```

## 複製

複製會在shard內實現  
2shard (shard1 , shard2) , 每個shard有2個replica , 當寫入資料到clickhouse-03 , 03會同步資料到04 , (01,02) 不會有資料 , 如果要把全部的資料都取出 , 可以再建立Distributed來讀取

- shard1
  - replica1 (clickhouse-01) [shard1內同步]
  - replica2 (clickhouse-02) [shard1內同步]
- shard2
  - replica3 (clickhouse-03) [shard2內同步]
  - replica4 (clickhouse-04) [shard2內同步]

### 1. 測試clickhouse keeper正常運行
``` shell
# 在任一keeper 上運行
echo mntr | nc clickhouse-keeper-01 9181
```
``` shell
# result

zk_version      v24.5.3.5-stable-e0eb66f8e1724d8339dd69310d290bbfdd144c83
zk_avg_latency  1
zk_max_latency  6
zk_min_latency  0
zk_packets_received     66
zk_packets_sent 66
zk_num_alive_connections        1
zk_outstanding_requests 0
zk_server_state leader # 表示為leader , 其他keeper會是follower
zk_znode_count  13
zk_watch_count  1
zk_ephemerals_count     0
zk_approximate_data_size        2179
zk_key_arena_size       0
zk_latest_snapshot_size 0
zk_open_file_descriptor_count   83
zk_max_file_descriptor_count    1048576
zk_followers    2
zk_synced_followers     2
```

### 2. 建立Replicated Table
```sql
CREATE TABLE IF NOT EXISTS db1.table2 ON CLUSTER force_cluster
(
    EventDate DateTime,
    Flag String,
    UserID UInt32
)
ENGINE = ReplicatedMergeTree
PARTITION BY toYYYYMM(EventDate)
ORDER BY (EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID); -- 隨機採樣設定
```

### 3. 在其中一個node寫入1筆資料到table2
```sql
INSERT INTO db1.table2(EventDate, Flag, UserID) VALUES ('2024-01-02', 'btn-click' , 10002);

SELECT * FROM db1.table2;
```

### 4. Replicated 和 Distributed 並用
```sql
--- 建立分布表, 對應到前一步建立的table2複製表
CREATE TABLE IF NOT EXISTS db1.table2_dist ON CLUSTER force_cluster
(
    EventDate DateTime,
    Flag String,
    UserID UInt32
) ENGINE = Distributed(force_cluster, db1, table2, rand());
```


---

# 參考
https://clickhouse.com/docs/en/architecture/horizontal-scaling
https://clickhouse.com/docs/en/engines/table-engines/special/distributed
https://clickhouse.com/docs/en/sql-reference/statements/alter/update
https://altinity.com/blog/2018-5-10-circular-replication-cluster-topology-in-clickhouse





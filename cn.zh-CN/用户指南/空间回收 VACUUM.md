# 空间回收 VACUUM {#concept_265480 .concept}

## 背景信息 {#section_bgy_mxh_thb .section}

表中的数据被删除或更新后（UPDATE/DELTE），物理存储层面并不会直接删除数据，而是标记这些数据不可见，所以会在数据页中留下很多“空洞”，在读取数据时，这些“空洞”会随数据页一起加载，拖慢数据扫描速度，需要定期回收删除的空间。

## 空间回收方法 {#section_tl4_xwh_thb .section}

使用`VACUUM`命令，可以对表进行重新整理，回收空间，以便获取更好的数据读取性能。VACUUM命令如下：

```
VACUUM [FULL] [FREEZE] [VERBOSE] [table];
```

VACUUM 会在页内进行整理，VACUUM FULL会跨数据页移动数据。 VACUUM执行速度更快， VACUUM FULL执行地更彻底，但会请求排他锁。建议定期对系统表进行VACUUM（每周一次）。

## 使用建议 {#section_jbb_5xh_thb .section}

什么情况下做VACUUM？

-   不锁表回收空间，只能回收部分空间。
-   频率：对于有较多实时更新的表，每天做一次。
-   如果更新是每天一次批量进行的，可以在每天批量更新后做一次。
-   对系统影响：不会锁表，表可以正常读写。会导致CPU、I/O使用率增加，可能影响查询的性能。

什么情况下做VACUUM FULL？

-   锁表，通过重建表回收空间,可回收所有空洞空间。对做了大量更新后的表，建议尽快执行VACUUM FULL。
-   频率：至少每周执行一次。如果每天会更新几乎所有数据，需要每天做一次。
-   对系统影响：会对正在进行vacuum full的表锁定，无法读写。会导致CPU、I/O使用率增加。建议在维护窗口进行操作。

## 查询需要执行VACUUM的表 {#section_pyt_fyh_thb .section}

AnalyticDB for PostgreSQL提供了一个gp\_bloat\_diag视图，统计当前页数和实际需要页数的比例。通过analyze table来收集统计信息之后，查看该视图。

```
gpadmin=# SELECT * FROM gp_toolkit.gp_bloat_diag;
bdirelid | bdinspname | bdirelname | bdirelpages | bdiexppages |                bdidiag                
----------+------------+------------+-------------+-------------+---------------------------------------
    21488 | public     | t1         |          97 |           1 | significant amount of bloat suspected
(1 row)
```

其结果只包括发生了中度或者显著膨胀的表。当实际页面数和预期页面的比率超过4但小于10时，就会报告为中度膨胀。当该比率超过10时就会报告显著膨胀。对于这些表，可以考虑进行VACUUM FULL来回收空间。

## VACUUM FREEZE的使用 {#section_fk5_jyh_thb .section}

AnalyticDB for PostgreSQL执行的所有事务都有唯一的事务ID\(XID\)，XID是单调递增的，上限是20亿。

随着数据库执行事务的增多，为防止XID超过极限，在XID接近xid\_stop\_limit-xid\_warn\_limit\(默认500000000\)时， ADB for PG会对执行事务的sql返回warning信息，提醒用户：

```
WARNING: database "database_name" must be vacuumed within number_of_transactions transactions
```

用户可通过手动执行VACUUM FREEZE当前database来缩小XID。

如果忽略这个warning信息，在XID继续增长到超过xid\_stop\_limit\(默认1000000000\)时， ADB for PG会拒绝新的事务执行，并返回报错信息：

```
FATAL: database is not accepting commands to avoid wraparound data loss in database "database_name"
```

此时，您需要通过提交工单联系阿里云工程师来解决此问题。


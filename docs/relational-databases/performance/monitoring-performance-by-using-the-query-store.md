---
title: "Monitoring Performance By Using the Query Store"
description: The SQL Server Query Store provides insight on query plan choice and performance. Query Store captures history of queries, plans, and runtime statistics.
ms.custom: ""
ms.date: 03/15/2022
ms.prod: sql
ms.prod_service: "database-engine, sql-database"
ms.reviewer: ""
ms.technology: performance
ms.topic: conceptual
helpviewer_keywords: 
  - "Query Store"
  - "Query Store, described"
author: WilliamDAssafMSFT
ms.author: wiassaf
monikerRange: "=azuresqldb-current||>=sql-server-2016||>=sql-server-linux-2017||=azuresqldb-mi-current||=azure-sqldw-latest"
---
# Monitoring performance by using the Query Store

[!INCLUDE [SQL Server ASDB, ASDBMI, ASA Dedicated Only](../../includes/applies-to-version/sqlserver2016-asdb-asdbmi-asa-dedicated-pool-only.md)]

The [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] Query Store feature provides you with insight on query plan choice and performance. It simplifies performance troubleshooting by helping you quickly find performance differences caused by query plan changes. Query Store automatically captures a history of queries, plans, and runtime statistics, and retains these for your review. It separates data by time windows so you can see database usage patterns and understand when query plan changes happened on the server. You can configure query store using the [ALTER DATABASE SET](../../t-sql/statements/alter-database-transact-sql-set-options.md) option.

For information about operating the Query Store in Azure [!INCLUDE[ssSDS](../../includes/sssds-md.md)], see [Operating the Query Store in Azure SQL Database](best-practice-with-the-query-store.md#Insight).

> [!IMPORTANT]
> If you are using Query Store for just in time workload insights in [!INCLUDE[sssql16-md](../../includes/sssql16-md.md)], plan to install the performance scalability fixes in [KB 4340759](https://support.microsoft.com/help/4340759) as soon as possible.

## <a name="Enabling"></a> Enabling the Query Store

 Query Store is not enabled by default for new SQL Server and Azure Synapse Analytics databases, and is enabled by default for new Azure SQL Database databases.

### Use the Query Store Page in [!INCLUDE[ssManStudioFull](../../includes/ssmanstudiofull-md.md)]

1. In Object Explorer, right-click a database, and then select **Properties**.

   > [!NOTE]
   > Requires at least version 16 of [!INCLUDE[ssManStudio](../../includes/ssmanstudio-md.md)].

2. In the **Database Properties** dialog box, select the **Query Store** page.

3. In the **Operation Mode (Requested)** box, select **Read Write**.

### Use Transact-SQL Statements

Use the `ALTER DATABASE` statement to enable the query store for a given database. For example:

```sql
ALTER DATABASE <database_name>
SET QUERY_STORE = ON (OPERATION_MODE = READ_WRITE);
```

In Azure Synapse Analytics, enable the Query Store without additional options, for example:

```sql
ALTER DATABASE <database_name>
SET QUERY_STORE = ON;
```

For more syntax options related to the Query Store, see [ALTER DATABASE SET Options &#40;Transact-SQL&#41;](../../t-sql/statements/alter-database-transact-sql-set-options.md).

> [!NOTE]
> Query Store cannot be enabled for the `master` or `tempdb` databases.

> [!IMPORTANT]
> For information on enabling Query Store and keeping it adjusted to your workload, refer to [Best Practice with the Query Store](../../relational-databases/performance/best-practice-with-the-query-store.md#Configure).

## <a name="About"></a> Information in the Query Store

Execution plans for any specific query in [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] typically evolve over time due to a number of different reasons such as statistics changes, schema changes, creation/deletion of indexes, etc. The procedure cache (where cached query plans are stored) only stores the latest execution plan. Plans also get evicted from the plan cache due to memory pressure. As a result, query performance regressions caused by execution plan changes can be non-trivial and time consuming to resolve.

Since the Query Store retains multiple execution plans per query, it can enforce policies to direct the Query Processor to use a specific execution plan for a query. This is referred to as plan forcing. Plan forcing in Query Store is provided by using a mechanism similar to the [USE PLAN](../../t-sql/queries/hints-transact-sql-query.md) query hint, but it does not require any change in user applications. Plan forcing can resolve a query performance regression caused by a plan change in a very short period of time.

> [!NOTE]
> Query Store collects plans for DML Statements such as SELECT, INSERT, UPDATE, DELETE, MERGE, and BULK INSERT.
>
> Query Store does not collect data for natively compiled stored procedures by default. Use [sys.sp_xtp_control_query_exec_stats](../../relational-databases/system-stored-procedures/sys-sp-xtp-control-query-exec-stats-transact-sql.md) to enable data collection for natively compiled stored procedures.

**Wait stats** are another source of information that helps to troubleshoot performance in the [!INCLUDE[ssde_md](../../includes/ssde_md.md)]. For a long time, wait statistics were available only on instance level, which made it hard to backtrack waits to a specific query. Starting with [!INCLUDE[ssSQL17](../../includes/sssql17-md.md)] and [!INCLUDE[ssSDSfull](../../includes/sssdsfull-md.md)], Query Store includes a dimension that tracks wait stats. The following example enables the Query Store to collect wait stats.

```sql
ALTER DATABASE <database_name>
SET QUERY_STORE = ON ( WAIT_STATS_CAPTURE_MODE = ON );
```

Common scenarios for using the Query Store feature are:

- Quickly find and fix a plan performance regression by forcing the previous query plan. Fix queries that have recently regressed in performance due to execution plan changes.
- Determine the number of times a query was executed in a given time window, assisting a DBA in troubleshooting performance resource problems.
- Identify top *n* queries (by execution time, memory consumption, etc.) in the past *x* hours.
- Audit the history of query plans for a given query.
- Analyze the resource (CPU, I/O, and Memory) usage patterns for a particular database.
- Identify top n queries that are waiting on resources.
- Understand wait nature for a particular query or plan.

The Query Store contains three stores:

- a **plan store** for persisting the execution plan information.
- a **runtime stats store** for persisting the execution statistics information.
- a **wait stats store** for persisting wait statistics information.

The number of unique plans that can be stored for a query in the plan store is limited by the **max_plans_per_query** configuration option. To enhance performance, the information is written to the stores asynchronously. To minimize space usage, the runtime execution statistics in the runtime stats store are aggregated over a fixed time window. The information in these stores is visible by querying the Query Store catalog views.

The following query returns information about queries and plans in the Query Store.

```sql
SELECT Txt.query_text_id, Txt.query_sql_text, Pl.plan_id, Qry.*
FROM sys.query_store_plan AS Pl
INNER JOIN sys.query_store_query AS Qry
    ON Pl.query_id = Qry.query_id
INNER JOIN sys.query_store_query_text AS Txt
    ON Qry.query_text_id = Txt.query_text_id;
```

## <a name="Regressed"></a> Use the Regressed Queries feature

After enabling the Query Store, refresh the database portion of the Object Explorer pane to add the **Query Store** section.

![SQL Server 2016 Query Store tree in SSMS Object Explorer](../../relational-databases/performance/media/objectexplorerquerystore.PNG "SQL Server 2016 Query Store tree in SSMS Object Explorer") ![SQL Server 2017 Query Store tree in SSMS Object Explorer](../../relational-databases/performance/media/objectexplorerquerystore_sql17.PNG "SQL Server 2017 Query Store tree in SSMS Object Explorer")

> [!NOTE]
>  For Azure Synapse Analytics, Query Store views are available under **System Views** in the database portion of the Object Explorer pane.

Select **Regressed Queries** to open the **Regressed Queries** pane in [!INCLUDE[ssManStudioFull](../../includes/ssmanstudiofull-md.md)]. The Regressed Queries pane shows you the queries and plans in the query store. Use the drop-down boxes at the top to filter queries based on various criteria: **Duration (ms)** (Default), CPU Time (ms), Logical Reads (KB), Logical Writes (KB), Physical Reads (KB), CLR Time (ms), DOP, Memory Consumption (KB), Row Count, Log Memory Used (KB), Temp DB Memory Used (KB), and Wait Time (ms).

Select a plan to see the graphical query plan. Buttons are available to view the source query, force and unforce a query plan, toggle between grid and chart formats, compare selected plans (if more than one is selected), and refresh the display.

![SQL Server 2016 Regressed Queries in SSMS Object Explorer](../../relational-databases/performance/media/objectexplorerregressedqueries.PNG "SQL Server 2016 Regressed Queries in SSMS Object Explorer")

To force a plan, select a query and plan, then select **Force Plan**. You can only force plans that were saved by the query plan feature and are still retained in the query plan cache.

## <a name="Waiting"></a> Finding waiting queries

Starting with [!INCLUDE[ssSQL17](../../includes/sssql17-md.md)] and [!INCLUDE[ssSDSfull](../../includes/sssdsfull-md.md)], wait statistics per query over time are available in Query Store.

In Query Store, wait types are combined into **wait categories**. The mapping of wait categories to wait types is available in [sys.query_store_wait_stats &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-wait-stats-transact-sql.md#wait-categories-mapping-table).

Select **Query Wait Statistics** to open the **Query Wait Statistics** pane in [!INCLUDE[ssManStudioFull](../../includes/ssmanstudiofull-md.md)] v18 or higher. The Query Wait Statistics pane shows you a bar chart containing the top wait categories in the Query Store. Use the drop-down at the top to select an aggregate criteria for the wait time: avg, max, min, std dev, and **total** (default).

![SQL Server 2017 Query Wait Statistics in SSMS Object Explorer](../../relational-databases/performance/media/query-store-waits.PNG "SQL Server 2017 Query Wait Statistics in SSMS Object Explorer")

Select a wait category by clicking on the bar and a detail view on the selected wait category displays. This new bar chart contains the queries that contributed to that wait category.

![SQL Server 2017 Query Wait Statistics detail view in SSMS Object Explorer](../../relational-databases/performance/media/query-store-waits-detail.PNG "SQL Server 2017 Query Wait Statistics detail view in SSMS Object Explorer")

Use the drop-down box at the top to filter queries based on various wait time criteria for the selected wait category: avg, max, min, std dev, and **total** (default). Select a plan to see the graphical query plan. Buttons are available to view the source query, force, and unforce a query plan, and refresh the display.

**Wait categories** are combining different wait types into buckets similar by nature. Different wait categories require a different follow-up analysis to resolve the issue, but wait types from the same category lead to very similar troubleshooting experiences, and providing the affected query on top of waits would be the missing piece to complete most such investigations successfully.

Here are some examples how you can get more insights into your workload before and after introducing wait categories in Query Store:

|Previous experience|New experience|Action|
|-|-|-|
|High RESOURCE_SEMAPHORE waits per database|High Memory waits in Query Store for specific queries|Find the top memory consuming queries in Query Store. These queries are probably delaying further progress of the affected queries. Consider using MAX_GRANT_PERCENT query hint for these queries, or for the affected queries.|
|High LCK_M_X waits per database|High Lock waits in Query Store for specific queries|Check the query texts for the affected queries and identify the target entities. Look in Query Store for other queries modifying the same entity, which are executed frequently and/or have high duration. After identifying these queries, consider changing the application logic to improve concurrency, or use a less restrictive isolation level.|
|High PAGEIOLATCH_SH waits per database|High Buffer IO waits in Query Store for specific queries|Find the queries with a high number of physical reads in Query Store. If they match the queries with high IO waits, consider introducing an index on the underlying entity, in order to do seeks instead of scans, and thus minimize the IO overhead of the queries.|
|High SOS_SCHEDULER_YIELD waits per database|High CPU waits in Query Store for specific queries|Find the top CPU consuming queries in Query Store. Among them, identify the queries for which high CPU trend correlates with high CPU waits for the affected queries. Focus on optimizing those queries - there could be a plan regression, or perhaps a missing index.|

## <a name="Options"></a> Configuration Options

For the available options to configure Query Store parameters, see [ALTER DATABASE SET options (Transact-SQL)](../../t-sql/statements/alter-database-transact-sql-set-options.md#query-store).

Query the `sys.database_query_store_options` view to determine the current options of the Query Store. For more information about the values, see [sys.database_query_store_options](../../relational-databases/system-catalog-views/sys-database-query-store-options-transact-sql.md).

For examples about setting configuration options using [!INCLUDE[tsql](../../includes/tsql-md.md)] statements, see [Option Management](#OptionMgmt).

> [!NOTE]
> For Azure Synapse Analytics, the Query Store can be enabled as on other platforms but additional configuration options are not supported. 

## <a name="Related"></a> Related Views, Functions, and Procedures

View and manage Query Store through [!INCLUDE[ssManStudio](../../includes/ssmanstudio-md.md)] or by using the following views and procedures.

### Query Store Functions

Functions help operations with the Query Store.

:::row:::
    :::column:::
        [sys.fn_stmt_sql_handle_from_sql_stmt &#40;Transact-SQL&#41;](../../relational-databases/system-functions/sys-fn-stmt-sql-handle-from-sql-stmt-transact-sql.md)
    :::column-end:::
:::row-end:::

### Query Store Catalog Views

Catalog views present information about the Query Store.

:::row:::
    :::column:::
        [sys.database_query_store_options &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-database-query-store-options-transact-sql.md)
    :::column-end:::
    :::column:::
        [sys.query_context_settings &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-context-settings-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        [sys.query_store_plan &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-plan-transact-sql.md)
    :::column-end:::
    :::column:::
        [sys.query_store_query &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-query-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        [sys.query_store_query_text &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-query-text-transact-sql.md)
    :::column-end:::
    :::column:::
        [sys.query_store_runtime_stats &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-runtime-stats-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        [sys.query_store_wait_stats &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-wait-stats-transact-sql.md)
    :::column-end:::
    :::column:::
        [sys.query_store_runtime_stats_interval &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-query-store-runtime-stats-interval-transact-sql.md)
    :::column-end:::
:::row-end:::


### Query Store stored procedures

Stored procedures configure the Query Store.

:::row:::
    :::column:::
        [sp_query_store_flush_db &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-flush-db-transact-sql.md)
    :::column-end:::
    :::column:::
        [sp_query_store_reset_exec_stats &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-reset-exec-stats-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        [sp_query_store_force_plan &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-force-plan-transact-sql.md)
    :::column-end:::
    :::column:::
        [sp_query_store_unforce_plan &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-unforce-plan-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        [sp_query_store_remove_plan &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-remove-plan-transct-sql.md)
    :::column-end:::
    :::column:::
        [sp_query_store_remove_query &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/sp-query-store-remove-query-transact-sql.md)
    :::column-end:::
:::row-end:::
:::row:::
    :::column:::
        sp_query_store_consistency_check &#40;Transact-SQL&#41;<sup>1</sup>
    :::column-end:::
    :::column:::
    :::column-end:::
:::row-end:::

<sup>1</sup> In extreme scenarios Query Store can enter an ERROR state because of internal errors. Starting with SQL Server 2017 (14.x), if this happens, Query Store can be recovered by executing the `sp_query_store_consistency_check` stored procedure in the affected database. See [sys.database_query_store_options](../../relational-databases/system-catalog-views/sys-database-query-store-options-transact-sql.md) for more details described in the `actual_state_desc` column description.

## <a name="Scenarios"></a> Key Usage Scenarios

### <a name="OptionMgmt"></a> Option Management

This section provides some guidelines on managing Query Store feature itself.

#### Query Store state

Query Store stores its data inside the user database and that is why it has size limit (configured with `MAX_STORAGE_SIZE_MB`). If data in Query Store hits that limit Query Store will automatically change state from read-write to read-only and stop collecting new data.

Query [sys.database_query_store_options](../../relational-databases/system-catalog-views/sys-database-query-store-options-transact-sql.md) to determine if Query Store is currently active, and whether it is currently collects runtime stats or not.

```sql
SELECT actual_state, actual_state_desc, readonly_reason,
    current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;
```

Query Store status is determined by the `actual_state` column. If it's different than the desired status, the `readonly_reason` column can give you more information. When Query Store size exceeds the quota, the feature will switch to read_only mode and provide a reason. For information on reasons, see [sys.database_query_store_options &#40;Transact-SQL&#41;](../system-catalog-views/sys-database-query-store-options-transact-sql.md).

#### Get Query Store options

To find out detailed information about Query Store status, execute following in a user database.

```sql
SELECT * FROM sys.database_query_store_options;
```

#### Setting Query Store interval

You can override interval for aggregating query runtime statistics (default is 60 minutes). New value for interval is exposed through `sys.database_query_store_options` view.

```sql
ALTER DATABASE <database_name>
SET QUERY_STORE (INTERVAL_LENGTH_MINUTES = 15);
```

Arbitrary values are not allowed for `INTERVAL_LENGTH_MINUTES`. Use one of the following: 1, 5, 10, 15, 30, 60, or 1440 minutes.

> [!NOTE]
> For Azure Synapse Analytics, customizing Query Store configuration options, as demonstrated in this section, is not supported. 

#### Query Store space usage

To check current the Query Store size and limit execute the following statement in the user database.

```sql
SELECT current_storage_size_mb, max_storage_size_mb
FROM sys.database_query_store_options;
```

If the Query Store storage is full use the following statement to extend the storage.

```sql
ALTER DATABASE <database_name>
SET QUERY_STORE (MAX_STORAGE_SIZE_MB = <new_size>);
```

#### Set Query Store options

You can set multiple Query Store options at once with a single ALTER DATABASE statement.

```sql
ALTER DATABASE <database name>
SET QUERY_STORE (
    OPERATION_MODE = READ_WRITE,
    CLEANUP_POLICY = (STALE_QUERY_THRESHOLD_DAYS = 30),
    DATA_FLUSH_INTERVAL_SECONDS = 3000,
    MAX_STORAGE_SIZE_MB = 500,
    INTERVAL_LENGTH_MINUTES = 15,
    SIZE_BASED_CLEANUP_MODE = AUTO,
    QUERY_CAPTURE_MODE = AUTO,
    MAX_PLANS_PER_QUERY = 1000,
    WAIT_STATS_CAPTURE_MODE = ON
);
```

For the full list of configuration options, see [ALTER DATABASE SET Options (Transact-SQL)](../../t-sql/statements/alter-database-transact-sql-set-options.md).

#### Cleaning up the space

Query Store internal tables are created in the PRIMARY filegroup during database creation and that configuration cannot be changed later. If you are running out of space you might want to clear older Query Store data by using the following statement.

```sql
ALTER DATABASE <db_name> SET QUERY_STORE CLEAR;
```

Alternatively, you might want to clear up only ad-hoc query data, since it is less relevant for query optimizations and plan analysis but takes up just as much space. 

In Azure Synapse Analytics, clearing the query store is not available. Data is automatically retained for the past 30 days.

#### Delete ad-hoc queries

This purges adhoc and internal queries from the Query Store so that the Query Store does not run out of space and remove queries we really need to track.

```sql
SET NOCOUNT ON
-- This purges adhoc and internal queries from 
-- the Query Store in the current database 
-- so that the Query Store does not run out of space 
-- and remove queries we really need to track

DECLARE @id int;
DECLARE adhoc_queries_cursor CURSOR
FOR
    SELECT q.query_id
    FROM sys.query_store_query_text AS qt
    JOIN sys.query_store_query AS q
    ON q.query_text_id = qt.query_text_id
    JOIN sys.query_store_plan AS p
    ON p.query_id = q.query_id
    JOIN sys.query_store_runtime_stats AS rs
    ON rs.plan_id = p.plan_id
    WHERE q.is_internal_query = 1  -- is it an internal query then we dont care to keep track of it
       OR q.object_id = 0 -- if it does not have a valid object_id then it is an adhoc query and we don't care about keeping track of it
    GROUP BY q.query_id
    HAVING MAX(rs.last_execution_time) < DATEADD (minute, -5, GETUTCDATE())  -- if it has been more than 5 minutes since the adhoc query ran
    ORDER BY q.query_id;
OPEN adhoc_queries_cursor ;
FETCH NEXT FROM adhoc_queries_cursor INTO @id;
WHILE @@fetch_status = 0
BEGIN
    PRINT 'EXEC sp_query_store_remove_query ' + str(@id);
    EXEC sp_query_store_remove_query @id;
    FETCH NEXT FROM adhoc_queries_cursor INTO @id;
END
CLOSE adhoc_queries_cursor;
DEALLOCATE adhoc_queries_cursor;
```

You can define your own procedure with different logic for clearing up data you no longer want.

The example above uses the `sp_query_store_remove_query` extended stored procedure for removing unnecessary data. You can also:

- Use `sp_query_store_reset_exec_stats` to clear runtime statistics for a given plan.
- Use `sp_query_store_remove_plan` to remove a single plan.

### <a name="Performance"></a> Performance Auditing and Troubleshooting

Query Store keeps a history of compilation and runtime metrics throughout query executions, allowing you to ask questions about your workload. The following sample queries may be helpful in your performance baseline and query performance investigation:

#### Last queries executed on the database

The last *n* queries executed on the database:

```sql
SELECT TOP 10 qt.query_sql_text, q.query_id,
    qt.query_text_id, p.plan_id, rs.last_execution_time
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
ORDER BY rs.last_execution_time DESC;
```

#### Execution counts

Number of executions for each query:

```sql
SELECT q.query_id, qt.query_text_id, qt.query_sql_text,
    SUM(rs.count_executions) AS total_execution_count
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
GROUP BY q.query_id, qt.query_text_id, qt.query_sql_text
ORDER BY total_execution_count DESC;
```

#### Longest average execution time

The number of queries with the longest average execution time within last hour:

```sql
SELECT TOP 10 rs.avg_duration, qt.query_sql_text, q.query_id,
    qt.query_text_id, p.plan_id, GETUTCDATE() AS CurrentUTCTime,
    rs.last_execution_time
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
WHERE rs.last_execution_time > DATEADD(hour, -1, GETUTCDATE())
ORDER BY rs.avg_duration DESC;
```

#### Biggest average physical I/O reads

The number of queries that had the biggest average physical I/O reads in last 24 hours, with corresponding average row count and execution count:

```sql
SELECT TOP 10 rs.avg_physical_io_reads, qt.query_sql_text,
    q.query_id, qt.query_text_id, p.plan_id, rs.runtime_stats_id,
    rsi.start_time, rsi.end_time, rs.avg_rowcount, rs.count_executions
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
JOIN sys.query_store_runtime_stats AS rs
    ON p.plan_id = rs.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi
    ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(hour, -24, GETUTCDATE())
ORDER BY rs.avg_physical_io_reads DESC;
```

#### Queries with multiple plans

These queries are especially interesting because they are candidates for regressions due to plan choice change. The following query identifies these queries along with all plans:

```sql
WITH Query_MultPlans
AS
(
SELECT COUNT(*) AS cnt, q.query_id
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p
    ON p.query_id = q.query_id
GROUP BY q.query_id
HAVING COUNT(distinct plan_id) > 1
)

SELECT q.query_id, object_name(object_id) AS ContainingObject,
    query_sql_text, plan_id, p.query_plan AS plan_xml,
    p.last_compile_start_time, p.last_execution_time
FROM Query_MultPlans AS qm
JOIN sys.query_store_query AS q
    ON qm.query_id = q.query_id
JOIN sys.query_store_plan AS p
    ON q.query_id = p.query_id
JOIN sys.query_store_query_text qt
    ON qt.query_text_id = q.query_text_id
ORDER BY query_id, plan_id;
```

#### Highest wait durations 

This query will return top 10 queries with the highest wait durations:

```sql
SELECT TOP 10
    qt.query_text_id,
    q.query_id,
    p.plan_id,
    sum(total_query_wait_time_ms) AS sum_total_wait_ms
FROM sys.query_store_wait_stats ws
JOIN sys.query_store_plan p ON ws.plan_id = p.plan_id
JOIN sys.query_store_query q ON p.query_id = q.query_id
JOIN sys.query_store_query_text qt ON q.query_text_id = qt.query_text_id
GROUP BY qt.query_text_id, q.query_id, p.plan_id
ORDER BY sum_total_wait_ms DESC;
```

> [!NOTE]
> In Azure Synapse Analytics, the Query Store sample queries in this section are supported with the exception of wait stats, which are not available in the Azure Synapse Analytics Query Store DMVs.

 #### Queries that recently regressed in performance

The following query example returns all queries for which execution time doubled in last 48 hours due to a plan choice change. This query compares all runtime stat intervals side by side:

```sql
SELECT
    qt.query_sql_text,
    q.query_id,
    qt.query_text_id,
    rs1.runtime_stats_id AS runtime_stats_id_1,
    rsi1.start_time AS interval_1,
    p1.plan_id AS plan_1,
    rs1.avg_duration AS avg_duration_1,
    rs2.avg_duration AS avg_duration_2,
    p2.plan_id AS plan_2,
    rsi2.start_time AS interval_2,
    rs2.runtime_stats_id AS runtime_stats_id_2
FROM sys.query_store_query_text AS qt
JOIN sys.query_store_query AS q
    ON qt.query_text_id = q.query_text_id
JOIN sys.query_store_plan AS p1
    ON q.query_id = p1.query_id
JOIN sys.query_store_runtime_stats AS rs1
    ON p1.plan_id = rs1.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi1
    ON rsi1.runtime_stats_interval_id = rs1.runtime_stats_interval_id
JOIN sys.query_store_plan AS p2
    ON q.query_id = p2.query_id
JOIN sys.query_store_runtime_stats AS rs2
    ON p2.plan_id = rs2.plan_id
JOIN sys.query_store_runtime_stats_interval AS rsi2
    ON rsi2.runtime_stats_interval_id = rs2.runtime_stats_interval_id
WHERE rsi1.start_time > DATEADD(hour, -48, GETUTCDATE())
    AND rsi2.start_time > rsi1.start_time
    AND p1.plan_id <> p2.plan_id
    AND rs2.avg_duration > 2*rs1.avg_duration
ORDER BY q.query_id, rsi1.start_time, rsi2.start_time;
```

If you want to see performance all regressions (not only those related to plan choice change), remove condition `AND p1.plan_id <> p2.plan_id` from the previous query.

 #### Queries with historical regression in performance

Comparing recent execution to historical execution, the next query compares query execution based on period of execution. In this particular example, the query compares execution in recent period (1 hour) vs. history period (last day) and identifies those that introduced `additional_duration_workload`. This metric is calculated as a difference between recent average execution and history average execution multiplied by the number of recent executions. It actually represents how much of additional duration recent executions introduced compared to history:

```sql
--- "Recent" workload - last 1 hour
DECLARE @recent_start_time datetimeoffset;
DECLARE @recent_end_time datetimeoffset;
SET @recent_start_time = DATEADD(hour, -1, SYSUTCDATETIME());
SET @recent_end_time = SYSUTCDATETIME();

--- "History" workload
DECLARE @history_start_time datetimeoffset;
DECLARE @history_end_time datetimeoffset;
SET @history_start_time = DATEADD(hour, -24, SYSUTCDATETIME());
SET @history_end_time = SYSUTCDATETIME();

WITH
hist AS
(
    SELECT
        p.query_id query_id,
        ROUND(ROUND(CONVERT(FLOAT, SUM(rs.avg_duration * rs.count_executions)) * 0.001, 2), 2) AS total_duration,
        SUM(rs.count_executions) AS count_executions,
        COUNT(distinct p.plan_id) AS num_plans
     FROM sys.query_store_runtime_stats AS rs
        JOIN sys.query_store_plan AS p ON p.plan_id = rs.plan_id
    WHERE (rs.first_execution_time >= @history_start_time
               AND rs.last_execution_time < @history_end_time)
        OR (rs.first_execution_time <= @history_start_time
               AND rs.last_execution_time > @history_start_time)
        OR (rs.first_execution_time <= @history_end_time
               AND rs.last_execution_time > @history_end_time)
    GROUP BY p.query_id
),
recent AS
(
    SELECT
        p.query_id query_id,
        ROUND(ROUND(CONVERT(FLOAT, SUM(rs.avg_duration * rs.count_executions)) * 0.001, 2), 2) AS total_duration,
        SUM(rs.count_executions) AS count_executions,
        COUNT(distinct p.plan_id) AS num_plans
    FROM sys.query_store_runtime_stats AS rs
        JOIN sys.query_store_plan AS p ON p.plan_id = rs.plan_id
    WHERE  (rs.first_execution_time >= @recent_start_time
               AND rs.last_execution_time < @recent_end_time)
        OR (rs.first_execution_time <= @recent_start_time
               AND rs.last_execution_time > @recent_start_time)
        OR (rs.first_execution_time <= @recent_end_time
               AND rs.last_execution_time > @recent_end_time)
    GROUP BY p.query_id
)
SELECT
    results.query_id AS query_id,
    results.query_text AS query_text,
    results.additional_duration_workload AS additional_duration_workload,
    results.total_duration_recent AS total_duration_recent,
    results.total_duration_hist AS total_duration_hist,
    ISNULL(results.count_executions_recent, 0) AS count_executions_recent,
    ISNULL(results.count_executions_hist, 0) AS count_executions_hist
FROM
(
    SELECT
        hist.query_id AS query_id,
        qt.query_sql_text AS query_text,
        ROUND(CONVERT(float, recent.total_duration/
                   recent.count_executions-hist.total_duration/hist.count_executions)
               *(recent.count_executions), 2) AS additional_duration_workload,
        ROUND(recent.total_duration, 2) AS total_duration_recent,
        ROUND(hist.total_duration, 2) AS total_duration_hist,
        recent.count_executions AS count_executions_recent,
        hist.count_executions AS count_executions_hist
    FROM hist
        JOIN recent
            ON hist.query_id = recent.query_id
        JOIN sys.query_store_query AS q
            ON q.query_id = hist.query_id
        JOIN sys.query_store_query_text AS qt
            ON q.query_text_id = qt.query_text_id
) AS results
WHERE additional_duration_workload > 0
ORDER BY additional_duration_workload DESC
OPTION (MERGE JOIN);
```

### <a name="Stability"></a> Maintaining query performance stability

For queries executed multiple times you may notice that [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] uses different plans, resulting in different resource utilization and duration. With Query Store, you can detect when query performance regressed and determine the optimal plan within a period of interest. You can then force that optimal plan for future query execution.

You can also identify inconsistent query performance for a query with parameters (either auto-parameterized or manually parameterized). Among different plans, you can identify the plan that is fast and optimal enough for all or most of the parameter values and force that plan, keeping predictable performance for the wider set of user scenarios.

### Force a plan for a query (apply forcing policy)

When a plan is forced for a certain query, [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] tries to force the plan in the optimizer. If plan forcing fails, an XEvent is fired and the optimizer is instructed to optimize in the normal way.

```sql
EXEC sp_query_store_force_plan @query_id = 48, @plan_id = 49;
```

When using `sp_query_store_force_plan` you can only force plans that were recorded by Query Store as a plan for that query. In other words, the only plans available for a query are those that were already used to execute that query while Query Store was active.

> [!NOTE]
> Forcing plans in Query Store is not supported in Azure Synapse Analytics. 

#### <a name="ctp23"><a/> Plan forcing support for fast forward and static cursors

Starting with [!INCLUDE[sql-server-2019](../../includes/sssql19-md.md)] and Azure SQL Database (all deployment models), Query Store supports the ability to force query execution plans for fast forward and static [!INCLUDE[tsql](../../includes/tsql-md.md)] and API cursors. Forcing is supported via `sp_query_store_force_plan` or through [!INCLUDE[ssManStudioFull](../../includes/ssmanstudiofull-md.md)] Query Store reports.

### Remove plan forcing for a query

To rely again on the [!INCLUDE[ssNoVersion](../../includes/ssnoversion-md.md)] query optimizer to calculate the optimal query plan, use `sp_query_store_unforce_plan` to unforce the plan that was selected for the query.

```sql
EXEC sp_query_store_unforce_plan @query_id = 48, @plan_id = 49;
```

## See Also

- [Best Practice with the Query Store](../../relational-databases/performance/best-practice-with-the-query-store.md)
- [Using the Query Store with In-Memory OLTP](../../relational-databases/performance/using-the-query-store-with-in-memory-oltp.md)
- [Query Store Usage Scenarios](../../relational-databases/performance/query-store-usage-scenarios.md)
- [How Query Store Collects Data](../../relational-databases/performance/how-query-store-collects-data.md)
- [Query Store Stored Procedures &#40;Transact-SQL&#41;](../../relational-databases/system-stored-procedures/query-store-stored-procedures-transact-sql.md)
- [Query Store Catalog Views &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/query-store-catalog-views-transact-sql.md)
- [Monitor and Tune for Performance](../../relational-databases/performance/monitor-and-tune-for-performance.md)
- [Performance Monitoring and Tuning Tools](../../relational-databases/performance/performance-monitoring-and-tuning-tools.md)
- [Open Activity Monitor &#40;SQL Server Management Studio&#41;](../../relational-databases/performance-monitor/open-activity-monitor-sql-server-management-studio.md)
- [Live Query Statistics](../../relational-databases/performance/live-query-statistics.md)
- [Activity Monitor](../../relational-databases/performance-monitor/activity-monitor.md)
- [sys.database_query_store_options &#40;Transact-SQL&#41;](../../relational-databases/system-catalog-views/sys-database-query-store-options-transact-sql.md)

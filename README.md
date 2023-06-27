# Extended Event Solution

---
Author: [Alvaro Lamg](https://github.com/Haisha10)
...

## Table of contents

- [Requeriments](#requeriments)
- [Extended Event](#extended-event)
  - [Session creation](#session-creation)
- [Gathering the data](#gathering-the-data)
  - [nodes() method](#nodes-method)
  - [Cross Apply](#cross-apply)
  - [Data extraction](#data-extraction)
- [Resetting the ring_buffer](#resetting-the-ring_buffer)
  - [Method 1](#method-1)
  - [Method 2](#method-2)
- [Process automation](#process-automation)

## Requeriments

Automate data collection in order to monitor the performance of the SQL Server.
The data needed will be:

- User information
- Executed script/query
- Timestamp
- Execution time

## Extended Event

SQL Server Extended Events is a performance monitoring tool that helps to collect and monitor the database engine actions to diagnose problems in SQL Server. Microsoft has introduced the extended events with the SQL Server 2008 and then decided to retire the SQL profiler.

### Session creation

It is required to collect any script executed by the user, so the event to track must be **'sql_statement_completed'**. Saving this data into a **.xel file** is convenient, since it can be accesible any time, but it may cause some vulnerabilities at the moment of reading the file. So it is safier to gather the from memory. Thats the reason to use **'ring_buffer'**.
Query used for the session creation:

```sql
CREATE EVENT SESSION [QueryTrack] ON SERVER 
ADD EVENT sqlserver.sql_statement_completed(ACTION(sqlserver.client_app_name, sqlserver.client_connection_id, sqlserver.database_name, sqlserver.sql_text, sqlserver.username))
ADD TARGET package0.ring_buffer
WITH (MAX_MEMORY=4096 KB, EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS, MAX_DISPATCH_LATENCY=30 SECONDS, MAX_EVENT_SIZE=0 KB,MEMORY_PARTITION_MODE=NONE, TRACK_CAUSALITY=OFF, STARTUP_STATE=OFF)
GO
```

This session will track any sql statement that is completed. It will collect data like:

- Database name
- Client application name
- Client hostname
- Username
- SQL Text and statement
- Timestamp
- Duration of the query execution
- CPU time
- Number of the logical reads
- Number of the physical reads
- Number of writes

## Gathering the data

You should be able to know the data location. So first of all, you will need to the address of the session. With that you can view the target data from the session target.
You can get it from here:

```sql
SELECT * FROM sys.dm_xe_sessions AS session
SELECT * FROM sys.dm_xe_session_targets AS target_data 
```

### nodes() method

The nodes() method is useful when you want to shred an xml data type instance into relational data. It allows you to identify nodes that will be mapped into a new row.

### Cross Apply

Cross Apply is an SQL feature that was introduced in SQL Server that works in a similar way to a join. It lets you join a table to a “table-valued function”, or to join to a subquery that refers to the outer query for each row, which is not possible with joins.

### Data extraction

If your read the target_data, you will notice that the data is in a **XML format**. In order to extract the data from the XML format into relational data, the **.node()** function will be needed. And also **Cross Apply** its needed in orden to join a table to a "table-valued-function" like .node()

```sql
--If the temporal table exists, drop it
IF OBJECT_ID('tempdb..#temp_querytrack_data') IS NOT NULL DROP TABLE #temp_querytrack_data;

--Create a temporal table
SELECT database_name, client_app_name, client_hostname, username, sql_text, event_timestamp, duration, cpu_time, logical_reads, physical_reads, writes
INTO #temp_querytrack_data
FROM
(
 --Gather especific data from the XML
 SELECT
  event.value('(action[@name="database_name"]/value)[1]', 'nvarchar(max)') AS database_name,
  event.value('(action[@name="client_app_name"]/value)[1]', 'nvarchar(max)') AS client_app_name,
  event.value('(action[@name="client_hostname"]/value)[1]', 'nvarchar(max)') AS client_hostname,
  event.value('(action[@name="username"]/value)[1]', 'nvarchar(max)') AS username,
  event.value('(action[@name="sql_text"]/value)[1]', 'nvarchar(max)') AS sql_text,
  event.value('(@timestamp)[1]', 'datetime') AS event_timestamp,
  event.value('(data[@name="duration"]/value)[1]', 'bigint') AS duration,
  event.value('(data[@name="cpu_time"]/value)[1]', 'bigint') AS cpu_time,  
  event.value('(data[@name="logical_reads"]/value)[1]', 'bigint') AS logical_reads,
  event.value('(data[@name="physical_reads"]/value)[1]', 'bigint') AS physical_reads,
  event.value('(data[@name="writes"]/value)[1]', 'bigint') AS writes
 FROM 
  (
   --Gather the XML of the XE Session
   SELECT 
    CAST(target_data.target_data AS XML) AS event_data_xml
   FROM 
    sys.dm_xe_sessions AS session
   JOIN 
    sys.dm_xe_session_targets AS target_data 
   ON 
    session.address = target_data.event_session_address
   WHERE 
    session.name = 'QueryTrack'
    AND target_data.target_name = 'ring_buffer'
  ) AS x
 --Read the XML and turn it into table data
 CROSS APPLY 
  event_data_xml.nodes('/RingBufferTarget/event') AS events(event)
) AS y
```

In the previous code, the data is exracted from the ring_buffer of our session 'QuertTrack'. Then the XML format is shredded into relational data and inserted into a temporal table.
We can create a table with this:

```sql
CREATE TABLE querytrack_data (
 database_name nvarchar(max) NOT NULL,
 client_app_name nvarchar(max) NOT NULL,
 client_hostname nvarchar(max) NOT NULL,
 username nvarchar(max) NOT NULL,
 sql_text nvarchar(max) NOT NULL,
 event_timestamp datetime NOT NULL,
 duration bigint NOT NULL,
 cpu_time bigint NOT NULL,
 logical_reads bigint NOT NULL,
 physical_reads bigint NOT NULL,
 writes bigint NOT NULL
)
```

Then we are ready for saving the data from the temporal table into a persistent one.

```sql
INSERT INTO querytrack_data
SELECT database_name, client_app_name, client_hostname, username, sql_text, event_timestamp, duration, cpu_time, logical_reads, physical_reads, writes FROM #temp_querytrack_data
EXCEPT
SELECT database_name, client_app_name, client_hostname, username, sql_text, event_timestamp, duration, cpu_time, logical_reads, physical_reads, writes FROM querytrack_data
```

The previous code its inseting the #temporal_querytrack_data into the persistant table querytrack_data. And in case there is any duplicated data, which is not likely to happen since we drop the temporal table every time we need to fill it, it will ignore them.
You can efficiently filter the data from the temporal table and insert the result into the persistant table with a **WHERE statement**.

***NOTE: You can insert the data directly into a previous created table, but the temporal table is prefered is you want to filter the data before inserting it into another table. The reason of this is because accessing the ring_buffer and parsing the XML format into relational data repeatly, consumes a lot of resources and takes longer to complete.***

## Resetting the ring_buffer

You should remember that ring_buffer starage data directly on memory. So it is very inconvenient to keep the data that have been saved into the persistant table, since the time pass the ring_buffer will take over more resources. Therefor, you may want to clear the ring_buffer after saving it into the table.

### Method 1

One method is to simply turn off the event session and start it over again.

```sql
ALTER EVENT SESSION QueryTrack ON SERVER
STATE = STOP;
GO
ALTER EVENT SESSION QueryTrack ON SERVER
STATE = START;
GO
```

### Method 2

This method is more efficient than the last method, since it an SQL Server system's stored procedure that is pretty optimized and will have less impact in performance.

```sql
-- Declare a variable to store the address of the ring_buffer target
DECLARE @ring_buffer_target_address varbinary(64);

-- Get the address of the ring_buffer target for the specified XE session
SELECT @ring_buffer_target_address = target_data.event_session_address
FROM sys.dm_xe_sessions AS session
-- Join the 'sys.dm_xe_sessions and 'sys.dm_xe_session_targets' tables to get the target information
JOIN sys.dm_xe_session_targets AS target_data ON session.address = target_data.event_session_address
 WHERE session.name = 'QueryTrack'
 AND target_data.target_name = 'ring_buffer';

-- Check if the ring buffer target address is not null
IF @ring_buffer_target_address IS NOT NULL
BEGIN
 -- Declare a variable to store an XML string used to clear the ring buffer target
 DECLARE @clear_ring_buffer_xml XML = 
 N'<RingBufferTarget truncate="true" />';
 
 -- Execute the `sp_reset_session_target_ring_buffer` stored procedure to clear the ring buffer target
 EXEC sys.sp_reset_session_target_ring_buffer 
 @session_address = @ring_buffer_target_address, 
 @target_xml = @clear_ring_buffer_xml;
END
```

## Process Automation

Finally, you may want to run this query repeatly between intervals of time. You can do this with:

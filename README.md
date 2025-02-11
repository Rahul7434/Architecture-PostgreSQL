 # Architecture-PostgreSQL
### Postgres Architecture is divided into Three Parts:- 
  - Postmaster
  - Background Proces
  - Memory
-----------------------------------------------------------------------------------------------------------------------------------------------------------------
#### Postmaster:- 
```
  - Postmaster is the main or super Process in Postgresql.
  - Responsible for handling Client connections & Background Processes.
  - It listens for new connections on the Configured IP address & TCP Port (default 5432 port) to the “Listen_address & port” parameter In the Postgresql. conf file to establish communication between the client & Database.
  - When the client sends a request, the Postmaster process accepts the request and checks the “pg_hba.conf” file for Authentication (The authentication method selected depends on the configuration in pg_hba.conf.) Like method, password, IP, Port.
  - Process checks if the client IP Address, Username, Database, and Password Match will establish a connectio n otherwise it will deny the request. 
  - One Connection is established then Postmaster will Start all Background Processes for each Connection.
```
### Background Process:- [bg_writer , wal_writer, checkpointer, Archiever, stat_collector, loggin_collector,  autovacuum_launcher, logical_replication_launcher.]
- BG Writer:- 
  ```
  Responsible for the performance of i/o operations. It continuously writes Dirty Pages from shared buffer to the Disk file.
  background writer periodically scans the shared buffer based on  "bgwriter_delay" parameter value and writes dirty pages to disk.
  Also during checkpoint intervals.
  bgwriter_delay (default 200ms) :- will scan each 200ms for dirty buffer.
  bgwriter_lru_maxpages (100 default):-  maximum numbers of dirty buffers written in each background write cycle.
  bgwriter_lru_multiplier (default 2.0) :-  (recent freed buffer * bgwritter_lru_multiplier) if recent 50 pages are written then next time it will write 100 pages (50 * 2.0).
  [If bgwriter_lru_maxpage=50 even though bgwriter wants to write 100 pages, it will only write 50.]
  ```
- Wal Writer:- 
    ```
    It flushes regularly modified transaction in to the walfile.
    (flushes the log from wal buffer to disk file if the walbuffer full, Transaction commited, time interval).
     wal_writer_delay (default 200ms) :- wal wrtier process will wakes up to flush wal buffer to disk based on this value. (lower values).
     wal_writer_flush_after (defsult 1MB) :- Specific amount of WAL Data must be written before forcing a flush to disk. it helps reduce fsync and improves performance by batching writes.
    
     
    ```
- CheckPointer:- 
    ```
     Ensures all dirty buffers has stored on the disk files and records current state of the database in the wal.
		 When Checkpoint is completed, the data up to that point is considered safe. 
     Recycling Process will recycle old file once they no longer needed (because they have been safetly Archived or they are beyond the last checkpoint). 
     checkpointer force to BG writer to write remaining all dirty pages on the disk.

    Checkpointer wakes up based on "checkpint_timeout" (default 5m) and max_wal_size(default 1GB)
    --------
	2. checkpoint_timeout
    ◦ Description: This setting specifies the maximum amount of time (from 30 seconds to 1 hour) between automatic checkpoints.
    	◦ Workflow:
        ▪ If the time since the last checkpoint exceeds this value, PostgreSQL will initiate a checkpoint, ensuring data is flushed to disk. The checkpoint will occur even if the checkpoint_segments threshold has not been reached, providing a time-based control mechanism.
	3. checkpoint_completion_target
    ◦ Description: This parameter indicates the target duration (between 0.0 and 1.0) for completing a checkpoint relative to the checkpoint_timeout.
    	◦ Workflow:
        ▪ If set to 0.5, for instance, PostgreSQL will aim to complete the checkpoint in half of the checkpoint_timeout duration. This helps to spread out the I/O load over time instead of performing all writes at once, which can lead to spikes in I/O operations.
	4. checkpoint_warning
    ◦ Description: This parameter sets the threshold (in seconds) for how long a checkpoint can take before a warning is logged. The default is 30 seconds. Setting it to 0 disables the warning.
    	◦ Workflow:
        ▪ If a checkpoint exceeds the specified duration, PostgreSQL will log a warning in the logs, allowing administrators to monitor long-running checkpoints that may indicate performance issues.	
    ```
- Archiver:- 
   ```
   This Process copies completed WAL segment files from the pg_wal directory.
	 for recovery purposes or replication.
		To enable archiving we need to ON “archive_mode” parameter.
		“archive_command” parameter specifies the shell command used to copy or move the wal segment file to a location. 
		 When the wal file is completed (it is fully written and closed) the archive process will trigger the “archive_command” to copy the file to the archive location.
    archive_mode :- on / off to archive wal files from pg_wal directory to another location.
    archive_command: - 'cp %p /path/archiveidr/%f';
    archive_timeout :- (default 60s) it will create new wal file even before file is not fully written each 60s.
    wal_keep_size:- ():-
    wal_level:- () defines how much information can store in wal based on (minimal, replica, logical) values.
   	minimal :- only crash recovery. not for replication and PITR. (low wal size fast performance).
   	replica :- for standby server & PITR. ()
   	logical :- luse full for ogical replication & PITR.
   	restore_command:- 
   
   ```
- Stat_Collector:- 
  ```
            Responsible for maintaining Statistics about the database activity and performance. 
            (query execution, table & Index usage, server activity) This data used for performance tuning, Monitoring, and troubleshooting.  
            [Pg_stat_activity, pg_stat_database, pg_stat_stat_usr_tables] :- current state and performance of the DB.
		        [Pg_stat_all_tables,]:- provides information about table & index usages.

  track_activities :- on /off :- Tracks active SQL quries of all sessions.
  track_counts :- on/off :- Enables collections of tables and index usage statistics.
  track_io_timing:-default off :-
  track_functions : default none:-
  track_wal_io_timing :- default off
  ```

- Logging Collecter| writter:- 
    ```
            -Responsible for capturing and managing log messages generated by the database server.
            -including error messages, warnings, info messages, log entries, etc.
            -it logs Client connections and query execution details into the “pg_log”.
    
            -[logging_collector]:- parameter for enable and disable logging collector.
            -[log_directory]:-specifies the directory where log files are stored.(log)
            -[log_filename,:- log file name format (postgresql-%y-%m-%d_%h%m%s.log)
             log_statement :- to log only ddl, dml, all, none (mod ddl+dml).
             

    
     ```

- autovacuum launcher:- 
   ```
            -Autovacuum Launcher starts the autovacuum Workers to clean up and analyze tables that require maintenance.
            -it scans tables if they need to be vacuumed or analyzed. The decision is based on deleted rows(tuples) or Updated age of statistics.
   	autovacuum :-  Enable or disable
   	autovacuum_naptime:- sets the delay between runs.
   	autovacuum_vacuum_threshold :- (default size 50) This is the minimum number of dead tuples required before autovacuum.
    	autovacuum_analyze_threshold:- (default size 50) Similar vacuum threshold but applies to ANALYZE.
   	autovacuum_vacuum_scale_factor (default 0.2 20%):- This controls when autoivaccum is triggered based on table size.(IT DEFINES percentage of dead tuples that must exist before 			autovacuum runs).
   	autovacuum_analyze_scale_factor:-(default size 0.1 10%)This determines when the ANALYZE runs based on the number of inserted/updated/deleted rows ().
    ```
- Logical Replication Launcher:- 
  ```
           responsible for handling logical replication subscriptions and publications.
  ```
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### Memory component:-  for Optimizing performance and handling data efficiently.

- “Shared Buffer”:-
  ```
  	 it acts as a cache, Shared Buffer Stores recently accessed or modified pages. Allowing Postgresql to access data directly from Memory.
	-It helps reduce disk i/o performance.
	- To configure shared buffer use the Parameter:- “shared_buffers” located in the “postgresql. conf” file. By default size is 128MB.
	-It uses the Last Recently Used (LRU) algorithm to manage buffer. 
	-changes made to data In the shared buffer are not immediately written to disk. If the page size is full then the “bg_writer” process flushes the dirty pages to 
       the disk. (by default page size is 8kb).
  {shared buffer size need 25% of the total memory}
  ```
- “Wal Buffer”:-
  ```
    	When a transaction make changes to the database, these changes are first written to the wal buffers before they are stored on disk.
	-when the transactions is committed or the buffer is full then the “wal_writer” background process will flush all data into the wal segment file.
	-crash or system failure, the wal files are used to recover committed transactions that may not have been fully written to the main data file.
	-Default size is 16MB of “wal_buffer” We can Adjust this parameter in the “PostgreSQL.conf” to optimize performance based on workload.
   {wal buffer size need to be 3% of shared buffer}.
  ```
- “tem_buffer”:-
  ```
	-temp_buffers is used for temporary tables created by a session. 
	-It defines the amount of memory allocated for temporary table data.
	-Workflow:-
    	2. When a session creates a temporary table, PostgreSQL allocates memory from temp_buffers.
    	3. Data written to temporary tables first goes into these buffers.
    	4. If temp_buffers is exhausted, the data spills to disk (in pgsql_tmp).
	Session Start → Create Temp Table → Allocate Memory (temp_buffers) → Buffer Data → Release on Session End.

  	-----------------------------------------------------------------------------------------------------------
  	Example:-
	[SET temp_buffers = '8MB'; -- Configures temp_buffers to 8MB for the session 
	CREATE TEMP TABLE temp_data AS SELECT * FROM large_table; -- Creates a temporary table 
	INSERT INTO temp_data SELECT * FROM another_large_table; -- Inserts into the temp table, using temp_buffers]
  	
  ```
- work_mem
  ```
	Description:
    	• It defines the amount of memory allocated per operation. Multiple operations within a
	- During query execution, PostgreSQL allocates memory for each sorting or hashing operation according to the work_mem setting.
	-Each session and operation can have its own work_mem, so the total memory usage can multiply quickly.
	-We can set session level, database level size of work_mem to manage query performance.
	-By default, it's size is 4MB. If any query Consumes a high CPU because query is large we can set work_mem size.
	-----------------------------------------------------------------------------------------------------------------------------------------
	Example:
    	• Consider a query that involves sorting and joining:
	SET work_mem = '16MB';  -- Set work_mem to 16MB
	SELECT * FROM orders ORDER BY order_date;  -- Sorting operation
   	• The work_mem setting of 16MB is used for sorting orders by order_date. If the data size exceeds 16MB, the extra data will spill to the disk, impacting performance.
  	(size need total ram * 0.25 /100)
  ```
- maintenance_work_mem
  ```
	Description:
    		• maintenance_work_mem is used for maintenance tasks such as VACUUM, CREATE INDEX, REINDEX, and ALTER TABLE.
	Workflow:
    	5. When maintenance tasks like indexing or vacuuming are executed, PostgreSQL allocates memory according to maintenance_work_mem.
    	6. Efficient use of this memory helps reduce disk I/O and speeds up these operations.
    	7. Maintenance tasks running concurrently will each get their allocation from maintenance_work_mem.
  	------------------------------------------------------------------------------------------------------------
	Example:
    	• During an indexing operation:
	SET maintenance_work_mem = '64MB';  -- Set maintenance_work_mem to 64MB
	CREATE INDEX idx_order_date ON orders(order_date);  -- Index creation
  ```
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------
### Query Flow:-
```
In PostgreSQL, a query flow for a read operation follows a structured process. Here's an overview of the query flow, breaking down the steps that a typical SELECT query goes through in PostgreSQL.
1. Client Request (Query Parsing)
    • Step 1: The client sends a SQL query (e.g., SELECT * FROM users WHERE age > 30;) to the PostgreSQL server.
    • Step 2: PostgreSQL performs parsing to check for correct syntax and convert the SQL query into a parse tree using the SQL grammar rules.
2. Query Rewriting
    • The rewrite system takes the parse tree and rewrites it based on the rules defined in PostgreSQL, including any views or rules that might modify the query.
    • For example, if the query involves a view, the view is expanded at this stage.
3. Planner/Optimizer
    • Step 1: The query planner examines the rewritten query tree and determines the most efficient way to execute the query.
    • Step 2: It considers multiple execution paths (such as sequential scans, index scans, joins) and selects the cheapest execution plan based on estimated costs.
    • Costs are calculated in terms of disk I/O, CPU usage, and memory usage.
4. Executor
    • The query executor executes the selected query plan.
    • Access Methods: If needed, the executor retrieves data from the database using different access methods such as:
        ◦ Sequential Scans (scanning all rows in a table).
        ◦ Index Scans (using B-tree, Hash, or other index types).
    • Joins: If the query involves multiple tables, different join algorithms (Nested Loop, Hash Join, etc.) may be applied.
5. Buffer Manager and Cache
    • Data is fetched from the shared buffer (a cache in memory) if available. If not, the buffer manager retrieves the data from disk into memory.
    • If data is not in memory, PostgreSQL requests it from the PostgreSQL buffer manager, which manages disk I/O.
6. Disk I/O (Storage Manager)
    • If data is not found in memory, it is retrieved from disk via the storage manager. PostgreSQL interacts with the OS to fetch data blocks (or pages, typically 8 KB in size).
    • It uses read-ahead and other optimizations to reduce disk access time.
7. Execution Results
    • The executor returns the result set (e.g., rows of data) back to the client.
    • If the data is large, PostgreSQL streams it in chunks to the client.
8. The Client Receives Data
    • The client receives the data and processes it accordingly (displaying it, saving it, or using it for further analysis).
```
##### Summary of the PostgreSQL Read Query Flow
```
    8. Client Request: Query is sent to the server.
    9. Parsing: Syntax check and creation of a parse tree.
    10. Query Rewrite: Modifications are applied if views or rules are involved.
    11. Planning/Optimization: Different execution plans are evaluated and the optimal one is selected.
    12. Execution: Data retrieval through sequential scans, index scans, or joins.
    13. Buffer Manager/Cache: Data is retrieved from memory or disk.
    14. Disk I/O: Data is fetched from disk if necessary.
    15. Results: Data is sent back to the client.
```

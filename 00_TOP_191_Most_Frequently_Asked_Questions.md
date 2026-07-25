# Top 191 Most Frequently Asked PostgreSQL DBA Interview Questions

> Curated and cross-checked against the recurring themes seen across multiple real interview loops (installation & config, port/instance management, backup strategies, PITR, replication types, upgrade process, performance-tuning war stories, ticket/escalation handling, cloud migration, day-to-day ops, checkpointer/background processes, autovacuum & locking behavior, shared_buffers, hot standby, and more). These are the questions most likely to come up in a 5–8 year PostgreSQL DBA interview loop, pulled from the larger master question bank and organized to match how a real interview typically flows: background → setup → internals → operations → troubleshooting.

Use this as your primary prep list; use the 21 topic files for deeper/backup coverage on any area you feel weak in.


## Getting Started & Background

1. Tell me about yourself and your experience as a PostgreSQL DBA.
- Focus on years of experience, PostgreSQL versions managed, and scale (database sizes, TPS, number of servers).
- Call out specific disciplines: backup/recovery, replication setup, performance tuning, migrations, HA.
- Mention tools in your stack: Patroni, PgBouncer, pgBackRest, Ansible, cloud platforms.
- End with a war story or inflection point — an incident or project that shaped how you work today.

2. Describe your current PostgreSQL environment, including the number of servers, databases, PostgreSQL versions, and deployment architecture.
- State the version(s), number of clusters, and rough data volume per cluster.
- Describe the HA topology: streaming replication, Patroni, number of standbys, sync vs. async.
- Mention connection pooling layer (PgBouncer, RDS Proxy) and backup tooling (pgBackRest, pg_basebackup).
- Note any partitioning, sharding, or multi-region setups if relevant.

3. What are your day-to-day responsibilities as a PostgreSQL DBA?
- Monitoring: replication lag, connection counts, autovacuum activity, checkpoint frequency, bloat.
- Change management: reviewing DDL from dev teams, estimating lock impact, scheduling maintenance.
- Backup verification: spot-checking restore tests, monitoring archive pipeline health.
- On-call/incident response: locking issues, runaway queries, replication failures.

4. What is the largest PostgreSQL database you have managed, and what were the key challenges?
- State the size clearly (e.g., 10 TB, 500M-row tables) and the PostgreSQL version.
- Name the actual challenges: autovacuum tuning at scale, PITR window management, index bloat, pg_upgrade across large datasets.
- Explain what you changed to address each challenge and the measurable outcome.

5. Walk me through the complete process of installing PostgreSQL from scratch on a Linux server.
- Add the PGDG repo and install the package:
```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf install -y postgresql16-server
```
- Initialize the cluster: `postgresql-16-setup initdb`
- Edit `postgresql.conf` (listen_addresses, shared_buffers, wal_level) and `pg_hba.conf`.
- Enable and start: `systemctl enable --now postgresql-16`
- Verify: `psql -U postgres -c "SELECT version();"`

6. What are the hardware and operating system prerequisites for deploying PostgreSQL in a production environment?
- Storage: XFS or ext4 on dedicated spindles or NVMe; separate mount for `pg_wal` if high write volume.
- OS: set `vm.overcommit_memory=2`, `vm.swappiness=1`, disable transparent hugepages, enable Huge Pages if RAM > 32 GB.
- Filesystem: `noatime` mount option; I/O scheduler `noop`/`none` for SSDs.
- ulimits: `nofile` ≥ 65536, `nproc` unlimited for the postgres user.
- Network: jumbo frames for replication traffic on 10 GbE+; dedicated NIC if possible.

## Installation, initdb & Configuration Files

7. Explain the purpose of **initdb**. What happens internally when a new PostgreSQL cluster is initialized?
- Creates the `PGDATA` directory structure (`base/`, `global/`, `pg_wal/`, etc.).
- Writes the system catalog tables (in `template1`) and bootstrap superuser.
- Generates `postgresql.conf`, `pg_hba.conf`, `pg_ident.conf` with defaults.
- Records locale, encoding, and data checksum setting — these cannot be changed without reinitializing.
- Assigns the initial checkpoint LSN and writes `pg_control`.

8. What files and directories are created after running **initdb**, and what is the purpose of each?
- `postgresql.conf` — main server configuration.
- `pg_hba.conf` — client authentication rules.
- `pg_ident.conf` — OS-to-database username mapping.
- `base/` — per-database subdirectories holding heap and index files.
- `global/` — cluster-wide tables (pg_database, pg_authid, etc.).
- `pg_wal/` — WAL segment files.
- `pg_xact/` — transaction commit/abort status (CLOG).
- `pg_control` — cluster state, last checkpoint LSN, timeline.

9. Explain the difference between a PostgreSQL **cluster**, **instance**, **database**, and **schema**.
- **Cluster**: single `PGDATA` directory managed by one postmaster; contains all databases.
- **Instance**: the running postmaster process (and its child backends) serving a cluster.
- **Database**: isolated namespace within a cluster; you connect to a specific database.
- **Schema**: a namespace within a database; tables, indexes, functions live inside schemas.

10. Explain the purpose of **postgresql.conf**. Which configuration parameters do you modify most frequently in production?
- Controls every server-side tunable: memory, WAL, autovacuum, connections, logging.
- Most-changed in production: `shared_buffers`, `work_mem`, `wal_level`, `max_wal_senders`, `checkpoint_completion_target`, `log_min_duration_statement`, `autovacuum_*`, `effective_cache_size`.
- Changes take effect on reload (`pg_reload_conf()`) except those marked `postmaster` which need a restart.

11. Explain the purpose of **pg_hba.conf**. How does PostgreSQL use this file during client authentication?
- Defines which hosts can connect, to which databases, as which users, and by what auth method.
- PostgreSQL reads it top-down and uses the first matching rule — order matters.
- Reloaded with `SELECT pg_reload_conf();` or `systemctl reload postgresql-16` — no restart needed.
- Columns: type (local/host/hostssl/hostnossl), database, user, address/mask, auth-method, options.

12. Explain the purpose of **pg_ident.conf**. When would you use user name mapping?
- Maps OS usernames to PostgreSQL usernames when using `peer` or `ident` authentication.
- Useful when the OS user running an app doesn't match the database role name.
- Referenced from `pg_hba.conf` via the `map=<mapname>` option.

13. How do you configure PostgreSQL to accept remote client connections securely?
- Set `listen_addresses = '*'` (or specific IP) in `postgresql.conf`.
- Add a `hostssl` rule in `pg_hba.conf` with `scram-sha-256`.
- Generate or install a TLS certificate; set `ssl = on`, `ssl_cert_file`, `ssl_key_file`.
- Reload config: `SELECT pg_reload_conf();`
- Restrict at the firewall level to known application server IPs — don't rely solely on pg_hba.

14. Explain the different PostgreSQL authentication methods, such as **trust**, **peer**, **password**, **md5**, **scram-sha-256**, and **LDAP**. When would you choose each method?
- `trust` — no password, accepts anyone matching the rule; only for local socket in isolated dev environments.
- `peer` — matches OS username to PG username; good for local Unix socket connections (e.g., postgres superuser).
- `password` — plaintext password over the wire; never use without TLS.
- `md5` — hashed challenge-response; deprecated, vulnerable to offline dictionary attacks.
- `scram-sha-256` — current standard; password never sent over wire; use this for all production remote connections.
- `ldap` — delegates to an LDAP server; good for centralized enterprise identity management.

15. A server is running five PostgreSQL instances. How would you identify and stop only the instance listening on port **5434**?
```bash
# Find the PID of the postmaster on port 5434
ss -tlnp | grep 5434
# or
pg_lsclusters   # on Debian/Ubuntu

# Get PGDATA from the PID
ls -l /proc/<pid>/cwd

# Stop gracefully using that cluster's pg_ctl
pg_ctl -D /var/lib/postgresql/16/cluster2 stop -m fast
```
- Never use `kill -9` on postmaster — it leaves shared memory attached and requires crash recovery.

16. How do you verify whether a PostgreSQL instance is running, and what information do you check before stopping or restarting it?
```bash
pg_ctl -D $PGDATA status
systemctl status postgresql-16
psql -c "SELECT pg_postmaster_start_time();"
```
- Before stopping: check `pg_stat_activity` for long-running queries, check replication lag on standbys, confirm no active backups (`pg_stat_activity` for `backup` state or `pg_backup_start`).

17. Explain the PostgreSQL startup process from the moment the server starts until it begins accepting client connections.
- `postmaster` reads `postgresql.conf` and `pg_hba.conf`.
- Acquires shared memory and initializes shared buffers.
- Reads `pg_control` to determine if crash recovery is needed; if so, replays WAL from last checkpoint.
- Starts background workers: checkpointer, bgwriter, walwriter, autovacuum launcher, archiver, stats collector.
- Opens the listen socket and begins accepting connections.

18. Explain the PostgreSQL shutdown modes (**Smart**, **Fast**, and **Immediate**). When would you use each mode?
- `Smart` (`-m smart`): waits for all clients to disconnect; never use in production — can wait forever.
- `Fast` (`-m fast`): rolls back active transactions, disconnects clients, clean shutdown; standard production use.
- `Immediate` (`-m immediate`): like a hard kill, requires crash recovery on next start; use only when Fast hangs (e.g., stuck recovery or unresponsive backend).

19. What log files and system logs do you examine when a PostgreSQL instance fails to start?
```bash
journalctl -u postgresql-16 -n 100
tail -100 /var/log/postgresql/postgresql-16-main.log
cat $PGDATA/pg_log/postgresql-*.log | tail -200
```
- Look for: `pg_control` version mismatch, lock file conflicts (`postmaster.pid` from a previous crash), port already in use, data directory permission errors, WAL corruption during recovery.

## Architecture & Background Processes

20. Explain PostgreSQL architecture from the moment a client establishes a connection until the query result is returned.
- Client connects to the listen socket; postmaster forks a dedicated backend process.
- Backend parses the SQL → rewrites → plans (planner/optimizer chooses scan/join strategy) → executes.
- Executor reads pages from shared buffers (or triggers I/O from OS if not cached), acquires necessary locks.
- Results stream back to the client via the wire protocol; WAL is flushed on COMMIT.

21. Explain the role of the **Postmaster (postgres)** process in PostgreSQL.
- Parent process that owns shared memory and listen sockets.
- Forks a new backend per client connection; monitors child processes and restarts them on crash.
- Manages cluster-wide state: reads `pg_control`, coordinates shutdown.
- Does not handle queries itself — that's the forked backend's job.

22. Explain the responsibilities of the **Background Writer** process. How does it differ from the Checkpointer?
- bgwriter proactively writes dirty shared buffer pages to disk between checkpoints to keep clean buffers available for backends.
- It targets `bgwriter_lru_maxpages` pages per round, sleeping `bgwriter_delay` between rounds.
- Checkpointer is responsible for the periodic forced flush of all dirty pages at checkpoint time, writing the checkpoint WAL record, and syncing files to disk with `fsync`.
- Difference: bgwriter reduces checkpoint I/O spikes by spreading writes; checkpointer is the definitive durability boundary.

23. Explain the responsibilities of the **Checkpointer** process. How does checkpoint frequency affect database performance and crash recovery?
- Writes all dirty shared buffer pages to disk, calls `fsync`, and writes a checkpoint WAL record.
- Controlled by `checkpoint_timeout` (time-based) and `max_wal_size` (WAL-size-based).
- More frequent checkpoints → shorter crash recovery (less WAL to replay) but higher ongoing I/O.
- Less frequent checkpoints → lower steady-state I/O but longer recovery time; watch `max_wal_size`.
- `checkpoint_completion_target = 0.9` spreads writes over 90% of the interval to avoid spikes.

24. Explain the purpose of the **WAL Writer** process. Why doesn't PostgreSQL write every transaction directly to the data files?
- WAL Writer flushes the WAL buffer (`wal_buffers`) to the WAL segment files periodically and on each COMMIT.
- Writing to the sequential WAL file is far cheaper than random I/O to heap/index files.
- WAL provides durability: crash recovery replays WAL to bring data files to a consistent state.
- Actual data file updates happen asynchronously via bgwriter/checkpointer — this is the foundation of write-ahead logging.

25. Explain the purpose of the **Autovacuum Launcher** and **Autovacuum Worker** processes. How do they work together?
- Launcher wakes up every `autovacuum_naptime` (default 1 min) and evaluates which tables need vacuuming or analyzing.
- It spawns worker processes (up to `autovacuum_max_workers`, default 3) to process those tables.
- Workers reclaim dead tuple space, update visibility map, freeze old XIDs, and update table statistics.
- Workers throttle themselves via `vacuum_cost_limit`/`vacuum_cost_delay` to reduce I/O impact.

26. Explain the purpose of the **Archiver** process. Under what conditions does it become active?
- Copies completed WAL segments to the archive location using `archive_command`.
- Only active when `archive_mode = on` (or `always`) and `wal_level >= replica`.
- If archive_command fails, the segment stays in `pg_wal` and the archiver retries — unarchived segments can fill `pg_wal` if the command is broken.
- Essential for PITR: without archived WAL, you can only restore to the last base backup.

27. Explain the purpose of the **Statistics Collector** (or the statistics subsystem in modern PostgreSQL versions). How does PostgreSQL use runtime statistics for monitoring and query optimization?
- Tracks per-table/per-index access counts, heap fetches, sequential and index scans, dead tuples — exposed via `pg_stat_*` views.
- In PG 15+, statistics are maintained in shared memory (not a separate process); in older versions a stats collector process wrote to files.
- Planner uses `pg_statistic` (populated by ANALYZE) for row count estimates, not the runtime stats.
- Runtime stats (`pg_stat_user_tables`) drive autovacuum decisions and are the DBA's primary monitoring signal.

28. Explain how PostgreSQL creates a dedicated backend process for each client connection. What are the advantages and limitations of this architecture?
- Postmaster calls `fork()` for each new connection; the child inherits shared memory mappings.
- Advantages: isolation (one crashed backend doesn't kill others), simplicity, no internal scheduler needed.
- Limitations: process creation overhead at high connection rates; each backend has its own `work_mem` allocation; context switching cost at 1000+ processes; this is why PgBouncer is essential.

29. Explain the lifecycle of a SQL query from parsing to execution. Which PostgreSQL components are involved at each stage?
- **Parse**: lexer/parser converts SQL text to a parse tree; checked for syntax only.
- **Analyze/Rewrite**: semantic analysis (name resolution, type checking); rule system applies view rewrites.
- **Plan**: planner generates candidate plans, optimizer picks lowest-cost one using statistics.
- **Execute**: executor walks the plan tree, fetches tuples from storage (via buffer manager), applies quals, returns rows.

30. Explain the role of the PostgreSQL **Parser**, **Planner**, **Optimizer**, and **Executor** during query execution.
- **Parser**: validates SQL syntax, produces a raw parse tree.
- **Planner/Optimizer** (same component): uses table statistics and cost model to enumerate plan options (scan types, join orders) and pick the cheapest.
- **Executor**: implements the chosen plan node by node — scans, joins, sorts, aggregations — and streams results.

31. Explain how PostgreSQL stores data on disk. What are relations, pages, tuples, and blocks?
- A **relation** is any heap/index file — tables, indexes, sequences, TOAST tables.
- Files are divided into 8 KB **pages** (also called blocks); page number is the `blkno`.
- Each page contains a header, item pointers (line pointer array), and **tuples** (rows) growing from the bottom.
- A tuple includes a `HeapTupleHeader` (xmin, xmax, ctid, infomask) followed by the actual column data.

32. Explain the purpose of **TOAST (The Oversized-Attribute Storage Technique)**. How does PostgreSQL store very large column values?
- Any column value > ~2 KB triggers TOAST; PostgreSQL compresses and/or slices it into 2 KB chunks stored in a per-table `pg_toast_<oid>` table.
- Four TOAST strategies per column: `PLAIN` (no TOAST), `EXTENDED` (compress then out-of-line), `EXTERNAL` (out-of-line, no compress), `MAIN` (compress, prefer inline).
- The main table row stores a TOAST pointer (18 bytes); retrieval is transparent.
- Large `text`, `jsonb`, `bytea` columns are the usual candidates.

33. Explain the PostgreSQL directory structure (**base**, **global**, **pg_wal**, **pg_tblspc**, **pg_stat**, **pg_xact**, **pg_multixact**, and other important directories). What is the purpose of each?
- `base/` — one subdirectory per database (named by OID); holds heap and index files.
- `global/` — shared catalog tables (pg_database, pg_authid, pg_tablespace).
- `pg_wal/` — WAL segments (16 MB each by default).
- `pg_xact/` — CLOG: commit/abort status bits for every XID.
- `pg_multixact/` — multi-transaction state for row-level shared locks.
- `pg_tblspc/` — symlinks to tablespace directories outside PGDATA.
- `pg_stat/` — stats files (pre-PG15); in PG15+ stats live in shared memory.
- `pg_subtrans/` — subtransaction parent tracking.
- `pg_twophase/` — state files for prepared (2PC) transactions.

34. Explain the purpose of tablespaces in PostgreSQL. When would you create additional tablespaces?
- A tablespace is a directory outside PGDATA where PostgreSQL stores relation files; created with `CREATE TABLESPACE`.
- Use cases: put indexes on faster NVMe while heap lives on slower HDD; spread I/O across multiple volumes; place temp tables on a scratch volume.
```sql
CREATE TABLESPACE fast_nvme LOCATION '/mnt/nvme/pgdata';
CREATE INDEX idx_orders_date ON orders(created_at) TABLESPACE fast_nvme;
```

35. Explain the difference between a database and a tablespace in PostgreSQL.
- **Database**: logical namespace; you connect to it; contains schemas, tables, functions; isolated from other databases.
- **Tablespace**: physical storage location; a database's objects can live across multiple tablespaces; a single tablespace can hold objects from multiple databases.

## Memory Architecture

36. Explain PostgreSQL's memory architecture. What are the major shared memory and local memory components?
- **Shared memory** (all backends): shared buffers, WAL buffers, lock table, clog buffers, shared memory queues.
- **Local memory per backend**: `work_mem` (sorts, hash joins), `maintenance_work_mem` (VACUUM, CREATE INDEX), `temp_buffers` (temp tables), catalog cache, plan cache.

37. Explain the purpose of **Shared Buffers** in PostgreSQL. How does it improve query performance, and what happens when the required pages are not found in the buffer cache?
- Shared buffers is the in-memory page cache shared across all backends; avoids repeated disk reads for hot pages.
- On a cache miss, PostgreSQL requests the page from the OS (which may hit Linux page cache) then copies it into a shared buffer.
- Eviction uses a clock-sweep algorithm; frequently accessed pages (high `usage_count`) stay longer.
- Typical sizing: 25% of RAM for dedicated servers; larger for read-heavy OLTP.

38. Explain the purpose of **work_mem** in PostgreSQL. How is it used during query execution, and what are the risks of setting it too high?
- Each sort operation and hash table in a query plan can use up to `work_mem` before spilling to disk.
- A complex query with 5 sort nodes and 200 concurrent sessions could allocate `5 × 200 × work_mem` simultaneously.
- Setting too high risks OOM — the OS kills PostgreSQL processes.
- Tune per-session for heavy analytical queries: `SET work_mem = '256MB';` rather than raising the global default.

39. Explain the purpose of **maintenance_work_mem**. Which maintenance operations use this memory, and how does it affect their performance?
- Used by: `VACUUM`, `CREATE INDEX`, `REINDEX`, `CLUSTER`, `ALTER TABLE` (when rebuilding), `pg_restore`.
- Larger value lets VACUUM collect more dead tuple TIDs per heap scan (fewer passes); lets CREATE INDEX sort more data in memory (fewer merge runs).
- Safe to set high (e.g., 1–2 GB) per session for maintenance windows since only a few maintenance operations run at a time.

40. Explain **effective_cache_size** in PostgreSQL. How does it influence the query planner's execution plan?
- A hint to the planner about the total memory available for caching (shared buffers + OS page cache).
- Does not allocate memory — it only adjusts cost estimates.
- Higher value lowers the estimated cost of index scans (planner assumes more of the index will be in cache), making it prefer index scans over seq scans.
- Typical value: 50–75% of total RAM.

41. How do you determine the appropriate value for **shared_buffers** in a production environment?
- Start at 25% of RAM; for RAM > 128 GB, 25% can still be right but diminishing returns appear.
- Monitor `pg_statio_user_tables` (heap_blks_hit vs heap_blks_read) and `pg_stat_bgwriter` (buffers_clean vs buffers_backend) — high backend writes mean shared_buffers is undersized.
- Also watch Linux page cache hit rate via `vmstat`; PostgreSQL relies on the OS cache for data that overflows shared_buffers.
- Don't exceed 40% of RAM — leaves too little for `work_mem`, OS cache, and other processes.

42. How do you estimate an appropriate **work_mem** value for a busy OLTP database?
- Formula baseline: `(RAM - shared_buffers) / (max_connections × avg_sort_nodes_per_query)`.
- For a 32 GB server, 8 GB shared_buffers, 200 connections, avg 2 sorts: `(24 GB) / (200 × 2)` ≈ 60 MB — but this assumes all connections sort at once, which is rare in OLTP.
- In practice: set global `work_mem = 4MB`–`16MB` for OLTP; let heavy queries override with `SET work_mem`.
- Check `pg_stat_statements` for queries with high temp file usage — those need more work_mem.

## MVCC, Transactions & Isolation

43. Explain **Multi-Version Concurrency Control (MVCC)** in PostgreSQL. How does it enable concurrent transactions without blocking readers?
- Every row version has `xmin` (creating XID) and `xmax` (deleting XID); a transaction sees a snapshot of the database as of its start.
- Readers never block writers and writers never block readers — each sees its own consistent snapshot.
- Old versions are left in place until VACUUM reclaims them; this is the source of table bloat if VACUUM is delayed.
- Contrast with lock-based systems (MySQL with REPEATABLE READ + undo log): PostgreSQL stores old versions inline in the heap.

44. Explain how PostgreSQL stores multiple row versions under MVCC.
- An UPDATE does not modify in place — it marks the old tuple's `xmax` with the updating XID and inserts a new tuple with the new values and `xmin` = current XID.
- Both versions coexist in the heap until VACUUM removes the old one (once no snapshot can see it).
- `ctid` (page, item) of the new tuple is recorded in the old tuple's header for HOT chain traversal.

45. Explain the difference between **xmin**, **xmax**, **ctid**, and tuple visibility.
- `xmin`: XID of the transaction that inserted this tuple version; tuple is visible once xmin is committed.
- `xmax`: XID of the transaction that deleted/updated this tuple; zero means not deleted; non-zero and committed means tuple is dead to new snapshots.
- `ctid`: physical location `(page, item_offset)` of this tuple; for HOT chains, the old tuple's ctid points to the new version.
- Visibility: a backend checks its snapshot against `xmin`/`xmax` commit status in CLOG to decide if it can see the tuple.

46. What are **Transaction IDs (XIDs)** in PostgreSQL? Why is transaction ID wraparound a serious concern, and how does PostgreSQL prevent it?
- XIDs are 32-bit counters; PostgreSQL uses ~2 billion XIDs as "past" and ~2 billion as "future".
- Wraparound: after 2^31 transactions, an old XID could appear "in the future," making old tuples invisible — effectively data loss.
- Prevention: VACUUM freezes old tuples (sets `xmin` to `FrozenXID = 2`, always visible), reclaiming the XID space.
- `autovacuum_freeze_max_age` (default 200M) triggers aggressive freeze vacuum when a table's oldest XID exceeds this threshold.
- Monitor: `SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY age DESC;` — alert when age > 1.5 billion.

47. Explain transaction ID freezing in PostgreSQL. Why is it necessary?
- Freezing replaces `xmin` with `FrozenTransactionId` (2), meaning "this tuple is visible to all current and future transactions."
- Necessary because XIDs wrap around at 2^32; without freezing, old tuples would become "future" and disappear.
- Controlled by `vacuum_freeze_min_age` (how old before freezing) and `vacuum_freeze_table_age` (force whole-table freeze scan).
- Anti-wraparound VACUUM is the most critical form — PostgreSQL will force it even with autovacuum disabled at `autovacuum_freeze_max_age`.

48. Explain the purpose of the **Visibility Map (VM)** in PostgreSQL. How does it help VACUUM and Index-Only Scans?
- One bit per heap page: set when all tuples on the page are known visible to all current transactions (no dead tuples).
- VACUUM skips all-visible pages during its dead-tuple scan, dramatically reducing I/O for large tables.
- A second bit (all-frozen) marks pages where all tuples are frozen — skipped entirely during freeze scans.
- Index-Only Scans check the VM: if the page is all-visible, no heap fetch is needed to verify tuple visibility.

49. Explain the purpose of the **Free Space Map (FSM)** in PostgreSQL. How does PostgreSQL use it during INSERT and UPDATE operations?
- Tracks approximate free space available in each heap page.
- On INSERT/UPDATE, PostgreSQL consults the FSM to find a page with enough space, avoiding scanning all pages.
- VACUUM updates the FSM after reclaiming dead tuple space.
- Stored as a tree structure in `<relation>_fsm` fork; not critical for correctness but important for performance.

50. Explain **HOT (Heap-Only Tuple) Updates** in PostgreSQL. When does PostgreSQL perform a HOT update, and what are its benefits?
- HOT update occurs when: the updated columns are not part of any index AND the new tuple fits on the same heap page as the old one.
- Instead of creating a new index entry, PostgreSQL chains the new tuple from the old one via `ctid`.
- Benefits: no index bloat from the update; faster UPDATE execution; VACUUM can reclaim the old version without touching the index.
- Check HOT rate: `n_tup_hot_upd / n_tup_upd` in `pg_stat_user_tables` — low rate signals excessive indexing of volatile columns.

51. Explain PostgreSQL isolation levels. How do they differ from the ANSI SQL standard?
- PostgreSQL implements: Read Committed (default), Repeatable Read, Serializable. Read Uncommitted maps to Read Committed.
- Key difference from ANSI: PostgreSQL's Repeatable Read prevents phantom reads (unlike ANSI where phantoms are allowed at RR); it uses snapshot isolation.
- Serializable uses SSI (Serializable Snapshot Isolation) — detects and aborts serialization anomalies without traditional locking.

52. Explain the differences between **Read Committed**, **Repeatable Read**, and **Serializable** isolation levels in PostgreSQL.
- **Read Committed**: snapshot taken at each statement; sees committed data from other transactions mid-transaction; default for OLTP.
- **Repeatable Read**: snapshot taken at transaction start; never sees concurrent commits during the transaction; can get serialization errors on write conflicts.
- **Serializable**: full SSI; prevents all anomalies (write skew, serialization anomalies); aborts transactions that violate serializability; use for financial or inventory systems requiring strict correctness.

53. Explain the purpose of **SELECT \... FOR UPDATE**, **FOR SHARE**, **FOR KEY SHARE**, and **FOR NO KEY UPDATE**.
- `FOR UPDATE`: acquires RowExclusiveLock on selected rows; blocks other `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`; use when you intend to UPDATE/DELETE the rows.
- `FOR NO KEY UPDATE`: like FOR UPDATE but doesn't block FOR KEY SHARE; for updates that don't touch the primary key.
- `FOR SHARE`: blocks concurrent writes but allows other FOR SHARE readers; use for read-modify-write where you need the row stable.
- `FOR KEY SHARE`: weakest; only blocks FOR UPDATE; used internally by FK checks.

54. Explain how PostgreSQL detects and resolves deadlocks.
- Each backend waiting for a lock records its wait in `pg_locks`; the lock manager builds a wait-for graph.
- `deadlock_timeout` (default 1 s): after waiting this long, the backend runs a deadlock detection cycle.
- If a cycle is found, PostgreSQL aborts one of the transactions (usually the one that triggered detection) with `ERROR: deadlock detected`.
- Fix at the application level: always acquire locks in the same order; keep transactions short; use `SELECT ... FOR UPDATE` explicitly.

## WAL & Crash Recovery

55. Explain Write-Ahead Logging (WAL) in PostgreSQL. Why is WAL essential for crash recovery and durability?
- Every data modification is first written to the WAL buffer, then flushed to a WAL segment file before the COMMIT ack is sent to the client.
- On crash, PostgreSQL replays WAL from the last checkpoint to bring heap/index files to a consistent state.
- Without WAL, partial writes to 8 KB pages could leave the database corrupted after a crash.
- WAL also enables streaming replication and PITR — standbys and restore processes replay the same WAL stream.

56. Explain the complete lifecycle of a WAL record from the moment a transaction modifies data until the transaction is committed.
- Backend modifies the page in shared buffers, marks it dirty.
- Generates a WAL record (change description + before/after image for full_page_writes) and copies it to the WAL buffer in shared memory.
- On COMMIT: WAL Writer (or the backend itself if `synchronous_commit=on`) flushes the WAL buffer to the WAL segment file via `write()` + `fsync()`/`fdatasync()`.
- COMMIT ack returned to client only after WAL is durable on disk.
- Eventually, checkpointer flushes the dirty heap/index pages to disk.

57. Explain the purpose of **LSN (Log Sequence Number)** in PostgreSQL. How is it used during recovery and replication?
- LSN is a byte offset into the WAL stream; monotonically increasing; identifies the exact position of every WAL record.
- During recovery, PostgreSQL reads `pg_control` for the last checkpoint LSN and replays WAL forward from there.
- In streaming replication, standby reports its `received_lsn`, `flushed_lsn`, and `applied_lsn` to the primary — the difference is replication lag.
```sql
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());
```

58. Explain the purpose of a checkpoint in PostgreSQL. How does it affect recovery time and database performance?
- Checkpoint writes all dirty shared buffer pages to disk and records the checkpoint LSN in `pg_control`.
- On crash recovery, PostgreSQL only needs to replay WAL from the last checkpoint — earlier WAL is irrelevant.
- More frequent checkpoints (lower `checkpoint_timeout` or `max_wal_size`) = shorter recovery but more I/O.
- `checkpoint_completion_target` (default 0.9) spreads dirty-page writes over 90% of the interval.

59. Explain how PostgreSQL performs crash recovery after an unexpected shutdown.
- On startup, reads `pg_control`; if state is not `shut down`, enters recovery mode.
- Locates last valid checkpoint record via `pg_control` (primary) or `backup_label` (after basebackup restore).
- Replays WAL records forward from that checkpoint, redoing all committed changes and rolling back in-flight transactions.
- Rebuilds `pg_xact` entries if needed; clears `pg_twophase` state; updates `pg_control` to `shut down` on completion.

60. Explain the difference between **Crash Recovery**, **Archive Recovery**, and **Point-in-Time Recovery (PITR)**.
- **Crash Recovery**: automatic, uses only WAL in `pg_wal`; replays from last checkpoint to end of WAL; no configuration needed.
- **Archive Recovery**: triggered by presence of `recovery.signal`; reads WAL from the archive (`restore_command`) when `pg_wal` doesn't have it; restores to end of archived WAL.
- **PITR**: archive recovery with `recovery_target_time` (or XID/LSN/name) set; stops replay at the specified point.

61. Explain **full-page writes** in PostgreSQL. Why are they required?
- When `full_page_writes = on`, the first modification to a page after a checkpoint writes the entire 8 KB page image into WAL.
- Required because storage devices may write a partial 8 KB page during a crash (torn write), leaving an inconsistent page; WAL replay needs the complete before-image to restore it correctly.
- Increases WAL volume; can be disabled only with hardware guaranteeing atomic 8 KB writes (very rare).
- `wal_compression` can reduce the overhead by compressing these full-page images.

## Backup, Restore & PITR

62. Explain the different backup strategies available in PostgreSQL. When would you choose logical backups over physical backups?
- **Logical** (`pg_dump`/`pg_dumpall`): SQL or custom-format dump; cross-version restore, selective object restore, human-readable; slow for large DBs, no PITR.
- **Physical** (`pg_basebackup`, pgBackRest, Barman): file-level copy of PGDATA + WAL; fast restore, PITR capable, version-locked.
- Choose logical for: cross-version migration, single-table restore, smaller DBs, dev/test environments.
- Choose physical for: production, large DBs (TBs), PITR requirement, fastest RTO.

63. Explain the differences between **pg_dump**, **pg_dumpall**, and **pg_basebackup**. What are the advantages and limitations of each?
- `pg_dump`: single database, logical, consistent snapshot (uses MVCC), selective restore via pg_restore.
- `pg_dumpall`: entire cluster including roles and tablespaces; outputs plain SQL only; no selective restore.
- `pg_basebackup`: physical file copy, requires WAL archiving for PITR, same major version for restore, fastest for large databases.

64. Explain how **pg_dump** works internally. What types of objects does it back up?
- Opens a transaction with `REPEATABLE READ` (or `SERIALIZABLE` with `--serializable-deferrable`) for a consistent snapshot.
- Exports: tables, views, sequences, functions, triggers, indexes, constraints, permissions, extensions, schemas, types.
- Does NOT export: WAL, `pg_global` catalog objects (roles, tablespaces) unless using `pg_dumpall`.
- Uses `COPY` for table data (fast) or `INSERT` statements if `--inserts` specified.

65. Explain the different output formats supported by **pg_dump**. When would you use the **plain**, **custom**, **directory**, or **tar** format?
- `plain` (`-Fp`): SQL text; pipe directly to psql; not parallelizable; no selective restore.
- `custom` (`-Fc`): compressed binary; supports parallel restore (`-j`), selective object restore, reordering — best default for production.
- `directory` (`-Fd`): one file per table; parallel dump (`-j`) and restore; good for large databases.
- `tar` (`-Ft`): portable tar archive; no parallel; size-limited; rarely preferred over custom.

66. Explain the purpose of **pg_restore**. How does it differ from restoring a plain SQL dump?
- Reads custom, directory, or tar format dumps and restores selectively or fully.
- Supports `-j N` parallel restore (multiple workers), `-t tablename` for single-table restore, `-n schema` filtering.
- Plain SQL dumps are restored with `psql -f dump.sql` — sequential, no parallelism, no selective restore.
```bash
pg_restore -d targetdb -j 8 -Fc backup.dump
```

67. Explain how **pg_basebackup** works. What files are included in a physical backup?
- Connects to PostgreSQL as a replication user, starts a checkpoint, and streams a copy of PGDATA over the replication protocol.
- Includes all files in PGDATA: heap files, indexes, WAL segments needed for consistency, `pg_control`, config files.
- Optionally streams WAL during the backup (`-X stream`) to make it self-sufficient.
- Does not include: tablespace files in external locations unless `--tablespace-map` is used.

68. Explain the prerequisites for taking a consistent physical backup using **pg_basebackup**.
- `wal_level >= replica`; `max_wal_senders >= 1`; a replication slot or sufficient `wal_keep_size` to retain WAL during backup.
- A replication user with `REPLICATION` privilege in `pg_hba.conf`.
- `archive_mode = on` + working `archive_command` if you want PITR capability from this backup.

69. Explain what **Point-in-Time Recovery (PITR)** is. What are the prerequisites for restoring a database to a specific point in time?
- PITR restores a base backup then replays archived WAL up to a target time, XID, LSN, or named restore point.
- Prerequisites: a base backup taken before the target time, all archived WAL segments from backup time to target time, and a `restore_command` that can retrieve them.
- Create `recovery.signal` in PGDATA, set `restore_command` and `recovery_target_time` in `postgresql.conf` (or `recovery.conf` pre-PG12), then start PostgreSQL.

70. Explain how WAL archiving works in PostgreSQL. Why is it essential for PITR?
- When `archive_mode=on`, completed WAL segments are passed to `archive_command`; PostgreSQL waits for the command to succeed before recycling the segment.
- The archive is the only way to fill the gap between base backup time and the target recovery time — without it, you can only restore to the backup point.
- Verify archiving health: `pg_stat_archiver` view, check `last_failed_time` and `failed_count`.

71. Explain the purpose of **archive_command**. What happens if the archive command repeatedly fails?
- Shell command PostgreSQL executes for each completed WAL segment; must return exit code 0 on success.
- If it fails, PostgreSQL retries but the segment stays in `pg_wal` — unarchived segments accumulate, eventually filling the WAL partition.
- Common cause of `pg_wal` full: broken S3 credentials, full archive disk, wrong path in archive_command.
- Monitor: `SELECT * FROM pg_stat_archiver;` — alert on `failed_count` increasing or `last_archived_time` lagging.

72. Explain the purpose of the **recovery.signal** file in modern PostgreSQL versions.
- Introduced in PG12 replacing `recovery.conf`; creating this empty file in PGDATA signals PostgreSQL to enter archive recovery mode on next start.
- Recovery parameters (`restore_command`, `recovery_target_*`) are now in `postgresql.conf`.
- PostgreSQL removes `recovery.signal` automatically when recovery completes (unless `recovery_target_action = pause`).

73. Explain how to verify whether a PostgreSQL backup is valid and restorable before relying on it for disaster recovery.
- Restore the backup to a separate server (not the production server) and start PostgreSQL — verify it completes recovery.
- Confirm data integrity: `SELECT count(*) FROM critical_table;`; run `pg_dump` on the restored instance.
- For pgBackRest: `pgbackrest --stanza=main verify` checks WAL completeness and file checksums.
- Document RTO: time the restore from start to `database accepting connections` — this is your real RTO.

74. Explain the best practices for backup retention in a production PostgreSQL environment.
- Keep daily base backups for 7–30 days depending on RPO/RTO requirements.
- Archive WAL continuously; retain WAL for at least as long as the oldest base backup you want to restore from.
- Store backups off-site or in a different cloud region from production.
- Automate restore testing weekly or monthly; a backup never tested is not a backup.
- Use pgBackRest's built-in retention policy (`repo1-retention-full`, `repo1-retention-diff`) to avoid manual cleanup.

75. Explain how you would restore a single database from a cluster backup.
- From a `pg_dump` backup: `pg_restore -d newdb -Fc single_db.dump`
- From a physical (pg_basebackup) backup: restore the full cluster, start it, then `pg_dump` only the target database from the restored instance and restore to production.
- There is no way to extract a single database from a physical backup without starting the full cluster.

76. Explain how you would restore a single table from a logical backup without affecting the rest of the database.
```bash
pg_restore -d targetdb -t tablename -Fc backup.dump
```
- If the table exists, drop it first or use `--clean`; check FK constraints — may need to disable them temporarily.
- For a partial restore, restore to a staging database first, then `INSERT INTO prod.table SELECT * FROM staging.table`.

77. Explain how you would restore a database after accidental deletion of several tables.
- If PITR is available: restore base backup + replay WAL to just before the DROP; rename/export only the affected tables; apply to production.
- If only logical backup: restore from latest `pg_dump` to a staging DB; extract only affected tables; insert into production.
- If neither: check if autovacuum hasn't reclaimed the pages yet — forensic recovery with `pg_filedump` may recover some data; this is a last resort.

78. Explain the difference between **Recovery Point Objective (RPO)** and **Recovery Time Objective (RTO)**. How do they influence backup strategy?
- **RPO**: maximum acceptable data loss measured in time (e.g., 15 minutes of data loss is acceptable).
- **RTO**: maximum acceptable downtime to recover the system (e.g., system must be back up within 2 hours).
- Low RPO → continuous WAL archiving + streaming replication; low RTO → standby server ready for promotion rather than restore from scratch.
- Balance: streaming replication gives near-zero RPO and RTO; PITR-only gives RPO = archive interval, RTO = full restore time.

79. Explain the purpose of streaming replication in PostgreSQL. How does it maintain a standby server?
- Primary sends WAL records in real time to standby via the WAL sender/receiver process pair.
- Standby replays received WAL and stays as close to primary as network/disk allows.
- Eliminates the delay inherent in file-based WAL archiving (where you'd wait for a full 16 MB segment to complete).
- Standby can serve read queries with `hot_standby = on`.

## Streaming & Logical Replication, HA

80. Explain the complete process of configuring streaming replication from scratch between a primary server and a standby server.
- On primary: set `wal_level=replica`, `max_wal_senders≥1`, create replication user.
- Add `pg_hba.conf` entry: `host replication replicator <standby_ip>/32 scram-sha-256`.
- On standby: `pg_basebackup -h primary -U replicator -D $PGDATA -P -R` (the `-R` flag writes `standby.signal` and connection info).
- Start standby PostgreSQL; verify: `SELECT * FROM pg_stat_replication;` on primary.

81. Explain the role of **wal_level**, **max_wal_senders**, **max_replication_slots**, and **hot_standby** in a streaming replication setup.
- `wal_level=replica` (or `logical`): enables WAL content needed for standbys; `minimal` disables replication.
- `max_wal_senders`: max concurrent WAL sender processes; must be ≥ number of standbys + basebackup connections.
- `max_replication_slots`: max replication slots; must be ≥ number of slots you plan to create.
- `hot_standby=on`: allows read queries on the standby while it's in recovery mode.

82. Explain how a standby server continuously receives and replays WAL records from the primary.
- WAL receiver connects to primary, authenticates, sends its current LSN.
- WAL sender streams WAL records as they're written; WAL receiver writes them to local `pg_wal`.
- Startup process reads from `pg_wal` and applies changes to heap/index files.
- Standby reports `received_lsn`, `flushed_lsn`, `replay_lsn` back to primary for lag monitoring.

83. Explain the purpose of the **WAL Sender** and **WAL Receiver** processes.
- **WAL Sender** (on primary): one per standby/basebackup client; reads WAL and streams it; also handles replication slot advancement.
- **WAL Receiver** (on standby): maintains the TCP connection to the primary; writes received WAL to local `pg_wal`; reports progress back.
- Each standby has exactly one WAL receiver; primary has one WAL sender per connected standby.

84. Explain how PostgreSQL determines replication lag between the primary and standby servers.
```sql
SELECT client_addr,
       pg_current_wal_lsn() - sent_lsn AS send_lag,
       sent_lsn - flush_lsn           AS receive_lag,
       flush_lsn - replay_lsn         AS replay_lag,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
```
- Three components: send lag (primary hasn't sent yet), receive lag (standby received but not flushed), replay lag (received but not applied).
- `replay_lag` is what matters for read consistency on the standby.

85. Explain how synchronous replication differs from asynchronous replication. What are the advantages and trade-offs of each approach?
- **Async** (default): COMMIT returns as soon as WAL is flushed locally on primary; standby may lag; data loss possible on primary failure.
- **Sync** (`synchronous_standby_names`): COMMIT waits until at least one standby acknowledges WAL flush (or apply, depending on `synchronous_commit` level); no data loss on primary failure; adds latency equal to RTT to standby.
- Trade-off: sync replication protects RPO = 0 but degrades write throughput and latency; async gives full write performance but accepts potential data loss.

86. Explain the difference between **physical replication** and **logical replication**. When would you choose one over the other?
- **Physical**: byte-for-byte WAL stream copy; standby is identical to primary; same PG version required; used for HA standbys.
- **Logical**: row-level change stream decoded from WAL; cross-version, cross-platform, selective (per-table/per-operation); used for selective replication, zero-downtime upgrades, feeding data pipelines.
- Choose physical for HA/DR standbys; choose logical for selective replication, read scaling to different PG versions, or Kafka/analytics pipelines.

87. Explain how logical replication works in PostgreSQL. What are publications and subscriptions?
- **Publication**: defined on the publisher; specifies which tables (or `ALL TABLES`) and which operations (INSERT/UPDATE/DELETE/TRUNCATE) to replicate.
- **Subscription**: defined on the subscriber; connects to the publisher, receives the logical change stream, applies changes.
```sql
-- Publisher
CREATE PUBLICATION my_pub FOR TABLE orders, customers;
-- Subscriber
CREATE SUBSCRIPTION my_sub CONNECTION 'host=primary dbname=mydb user=rep' PUBLICATION my_pub;
```
- Initial data sync copies existing rows; ongoing changes are streamed as they happen.

88. Explain the prerequisites for configuring logical replication.
- Publisher: `wal_level=logical`; `max_replication_slots ≥ 1`; `max_wal_senders ≥ 1`.
- Tables must have a PRIMARY KEY or `REPLICA IDENTITY FULL` (for UPDATE/DELETE to work).
- Subscriber must have matching table schemas (logical replication does not replicate DDL).
- Replication user needs `REPLICATION` privilege and `SELECT` on published tables.

89. Explain how replication slots can cause uncontrolled WAL growth if not monitored properly.
- A slot tracks the oldest LSN that a subscriber/standby still needs; PostgreSQL retains all WAL after that LSN.
- If a subscriber disconnects or falls far behind, the slot's `restart_lsn` stops advancing — WAL accumulates indefinitely.
- Monitor:
```sql
SELECT slot_name, pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE active = false;
```
- Drop stale slots immediately: `SELECT pg_drop_replication_slot('slot_name');`

90. Explain the purpose of **hot_standby**. What operations are allowed on a Hot Standby server?
- `hot_standby = on` allows read-only queries on a standby while it applies WAL.
- Allowed: `SELECT`, `COPY TO`, read-only transactions, `SET` for session parameters.
- Not allowed: `INSERT`/`UPDATE`/`DELETE`, DDL, `CREATE`/`DROP`, write transactions.
- Conflict resolution: long-running queries on standby can conflict with WAL cleanup (`hot_standby_feedback = on` to prevent, at cost of bloat on primary).

91. Explain how timeline switching occurs after a failover.
- Each timeline represents a distinct WAL history; the primary always runs on the current timeline.
- After promotion, the new primary increments the timeline ID and writes a timeline history file to the archive.
- If the old primary comes back, it must not continue on the old timeline — it must follow the new timeline by rewinding (`pg_rewind`) and rejoining as a standby.
- `recovery_target_timeline = 'latest'` tells a restoring standby to follow the latest timeline in the archive.

92. Explain how PostgreSQL prevents an old primary server from rejoining the cluster with stale data after failover.
- The old primary has data that diverged after the failover point — it cannot simply be reattached as a standby.
- Solution: run `pg_rewind` on the old primary to revert its data directory to the divergence point and then let it follow the new primary.
```bash
pg_rewind --target-pgdata=$PGDATA --source-server="host=new_primary dbname=postgres"
```
- After rewind, create `standby.signal` and configure `primary_conninfo`; start it as a standby.
- Without pg_rewind, the old primary must be re-initialized with `pg_basebackup`.

93. Explain the purpose of High Availability (HA) in PostgreSQL. What are the common HA architectures used in production?
- HA ensures the database continues serving requests after a node failure, with minimal downtime and data loss.
- Common architectures: Patroni + etcd/Consul/ZooKeeper (most popular), repmgr, Pgpool-II, cloud-native (RDS Multi-AZ, Aurora).
- Minimum viable HA: 1 primary + 1 sync standby + automated failover manager (Patroni) + VIP or HAProxy for client routing.

94. Explain the difference between failover and switchover.
- **Failover**: unplanned; primary is dead; standby is promoted; old primary cannot be gracefully demoted; data loss risk with async replication.
- **Switchover**: planned; primary is healthy; controlled demotion to standby, promotion of target standby; zero data loss; used for maintenance, upgrades.
- In Patroni: `patronictl switchover --master old --candidate new` performs a graceful switchover.

95. Explain the purpose of automatic failover. What components are typically required to implement it?
- Automatic failover detects primary failure and promotes a standby without manual intervention, reducing RTO.
- Required components: a DCS (distributed consensus store: etcd, Consul, ZooKeeper) to avoid split-brain; a failover manager (Patroni) that monitors health and controls promotion; a client routing layer (HAProxy, VIP, or DNS) to redirect traffic.

96. Explain how Patroni manages PostgreSQL High Availability.
- Each Patroni node runs alongside PostgreSQL and registers in the DCS (etcd/Consul).
- The leader holds a renewable DCS lock (TTL-based); if it fails to renew, the lock expires and another node competes.
- Patroni controls PostgreSQL start/stop/promotion; exposes a REST API and HTTP health check for HAProxy.
- Configuration is centralized in DCS; all nodes receive config changes automatically.

97. Explain how Patroni elects a new primary during failover.
- When the leader lock expires, all replicas race to acquire it in the DCS using a compare-and-swap operation.
- The replica with the highest LSN (most up-to-date) is preferred — Patroni checks `pg_stat_replication` or compares `replay_lsn`.
- The winner acquires the DCS lock and promotes its PostgreSQL instance; losers continue as standbys of the new primary.
- Fencing: Patroni can call a STONITH script to ensure the old primary is truly dead before promotion.

98. Explain the process of reinitializing a failed standby server in a Patroni cluster.
```bash
patronictl reinit <cluster_name> <member_name>
```
- Patroni stops the failed standby's PostgreSQL, wipes its PGDATA, runs `pg_basebackup` from the current primary, and restarts it as a standby.
- The node re-registers in the DCS and begins streaming WAL from the new primary.
- Manual alternative: `pg_basebackup -h primary -D $PGDATA -R -P` then start PostgreSQL.

## Query Planner & Performance Tuning

99. Explain the PostgreSQL query planner. How does it determine the most efficient execution plan for a SQL statement?
- Planner enumerates candidate plans: for each relation, considers seq scan, index scans (all applicable indexes), bitmap scan.
- For joins, considers all orderings (up to `join_collapse_limit`) and join methods (nested loop, hash, merge).
- Assigns a cost to each plan using statistics (row counts, column histograms from `pg_statistic`) and cost constants (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`).
- Selects the plan with the lowest total cost and returns it to the executor.

100. Explain the difference between **EXPLAIN** and **EXPLAIN ANALYZE**. When would you use each?
- `EXPLAIN`: shows estimated plan with cost estimates only; zero query execution; safe to run on production at any time.
- `EXPLAIN ANALYZE`: actually executes the query, shows actual row counts and timing per node — reveals planner estimate errors.
- `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)`: also shows buffer hit/miss counts — essential for diagnosing I/O-heavy plans.
- Never run `EXPLAIN ANALYZE` on a DML statement in production without wrapping in a `ROLLBACK`.

101. Explain the purpose of database statistics. How does PostgreSQL collect and use statistics during query optimization?
- `ANALYZE` samples up to `default_statistics_target` × 300 rows per column and stores histograms, MCV lists, and correlation in `pg_statistic`.
- Planner uses these to estimate selectivity (how many rows match a predicate), which drives row count estimates in the plan.
- Bad statistics → bad estimates → bad plans; symptoms: planner underestimates rows → chooses nested loop where hash join would be better.
- Increase `ALTER TABLE t ALTER COLUMN c SET STATISTICS 500;` for skewed or high-cardinality columns.

102. Explain the purpose of the **ANALYZE** command. When should you run it manually?
- Updates `pg_statistic` by sampling table data; autovacuum runs it automatically when `n_mod_since_analyze` exceeds the threshold.
- Run manually after: bulk loads, large deletes, or whenever query plans suddenly degrade after data changes.
```sql
ANALYZE VERBOSE orders;
```
- Check staleness: `SELECT relname, last_analyze, last_autoanalyze FROM pg_stat_user_tables;`

103. Explain why PostgreSQL may choose a Sequential Scan instead of an Index Scan.
- Planner estimates that fetching via index will cost more random I/O than scanning the table sequentially.
- Triggers: low selectivity predicate (e.g., `status = 'active'` when 70% of rows match); small table (fits in a few pages); stale statistics underestimating table size.
- Also happens when `random_page_cost` is too high relative to `seq_page_cost` for SSD storage — set `random_page_cost = 1.1` for NVMe.
- Force check: `SET enable_seqscan = off; EXPLAIN ...` to see the index plan cost for comparison.

104. Explain the purpose of an **Index Only Scan**. Under what conditions can PostgreSQL use it?
- Returns data directly from the index without touching the heap; much faster for large tables when the index covers all needed columns.
- Conditions: all columns referenced in the query (SELECT list + WHERE + ORDER BY) must be in the index; the visibility map must show the page as all-visible (otherwise a heap fetch is needed to check tuple visibility).
- VACUUM regularly to keep visibility map current — stale VM causes fallback to heap fetches.

105. Explain how PostgreSQL decides between a Nested Loop Join, Hash Join, and Merge Join.
- **Nested Loop**: for each outer row, scans inner relation; efficient when outer is small and inner has an index; O(N×M) without index.
- **Hash Join**: builds a hash table from the smaller relation, probes with each outer row; good for large equi-joins without sort order; requires `work_mem` for the hash table.
- **Merge Join**: both inputs sorted on join key; merges in one pass; efficient when inputs are already sorted (index scan) or join is large; needs sort step if not pre-sorted.
- Planner picks based on estimated cost; disable individually with `enable_nestloop`, `enable_hashjoin`, `enable_mergejoin` for testing.

106. Explain the advantages and disadvantages of Nested Loop Joins.
- Advantages: low startup cost, works with any join condition, efficient when outer set is small and inner has an index.
- Disadvantages: O(N×M) without an index on the inner side — catastrophic for large tables without proper indexing; can be slow even with an index if outer cardinality is underestimated.
- Red flag in EXPLAIN: nested loop where actual rows >> estimated rows on the outer side.

107. Explain the purpose of **Memoize** in modern PostgreSQL versions. How can it improve query performance?
- Added in PG14; caches the result of inner-side lookups in a nested loop join when the same parameter value repeats.
- Useful for correlated subqueries or parameterized index scans with repeated values (e.g., joining to a small lookup table where the same FK repeats many times).
- Cache size bounded by `work_mem`; evicts on overflow; check `EXPLAIN ANALYZE` for `Cache Hits` vs `Cache Misses`.

## Indexing, Vacuum & Bloat

108. Explain the different index types available in PostgreSQL, including **B-tree**, **Hash**, **GIN**, **GiST**, **SP-GiST**, **BRIN**, and **Bloom** indexes. When would you choose each?
- **B-tree**: default; equality, range, ORDER BY, prefix LIKE; covers 95% of use cases.
- **Hash**: equality only; faster point lookups than B-tree on equality; no ordering, not WAL-logged pre-PG10 (now safe).
- **GIN**: inverted index for `jsonb`, `tsvector`, arrays; multiple values per row; slow to build, fast to search.
- **GiST**: generalized search tree; geometric types, range types, full-text (`tsvector`); supports nearest-neighbor.
- **SP-GiST**: space-partitioning tree; point data, inet, text prefix; efficient for non-balanced data distributions.
- **BRIN**: block range index; tiny footprint; only useful for columns highly correlated with physical storage order (e.g., auto-incrementing IDs, time-series timestamps).
- **Bloom**: probabilistic multi-column equality; very small size; false positives mean some heap fetches; niche use for multi-column equality with many columns.

109. Explain the factors that cause index bloat in PostgreSQL.
- MVCC updates: every UPDATE creates a new index entry pointing to the new tuple version; old entries remain until VACUUM removes them.
- High UPDATE/DELETE rate on indexed columns without sufficient autovacuum.
- Failed HOT updates (updated column is indexed) force new index entries per update.
- Large transaction holding an old snapshot prevents VACUUM from removing dead tuples.

110. Explain the factors that cause table bloat in PostgreSQL.
- Dead tuples accumulate when VACUUM is too infrequent or too slow.
- Long-running transactions (or idle-in-transaction sessions) hold snapshots that prevent VACUUM from removing dead tuples visible to those snapshots.
- Autovacuum cost throttling set too conservatively; autovacuum_vacuum_scale_factor too high for large tables.
- Disabled autovacuum on a table.

111. Explain the purpose of **VACUUM**. Why doesn't it reduce the physical size of a table?
- Marks dead tuples as reusable free space; updates the FSM and visibility map; advances the table's `relfrozenxid`.
- Does NOT return space to the OS — it marks pages as reusable for future INSERTs within PostgreSQL.
- To return disk space to the OS: use `VACUUM FULL` (rewrites the table, takes `AccessExclusiveLock`) or `pg_repack` (online, no long lock).

112. Explain the difference between **VACUUM**, **VACUUM FULL**, **ANALYZE**, **REINDEX**, **CLUSTER**, and **pg_repack**. When would you use each?
- `VACUUM`: reclaim dead tuple space; lightweight, concurrent; run regularly via autovacuum.
- `VACUUM FULL`: rewrite table, return space to OS; requires `AccessExclusiveLock`; use for severe bloat during maintenance.
- `ANALYZE`: update statistics; no space reclamation; run after bulk changes.
- `REINDEX`: rebuild index from scratch; use for corrupted or heavily bloated indexes; `REINDEX CONCURRENTLY` avoids locking.
- `CLUSTER`: physically reorder table by an index; `AccessExclusiveLock`; improves range-scan performance temporarily.
- `pg_repack`: online table/index rebuild without long locks; preferred alternative to VACUUM FULL in production.

113. Explain how **Autovacuum** works in PostgreSQL. Which background processes are responsible for automatic vacuuming?
- Autovacuum launcher wakes every `autovacuum_naptime` (1 min default) and checks `pg_stat_user_tables` for tables needing work.
- Criteria: `n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × n_live_tup`.
- Launcher spawns a worker process (up to `autovacuum_max_workers`) per qualifying table.
- Workers self-throttle via vacuum cost parameters to minimize I/O impact.

114. Explain the key configuration parameters that control Autovacuum behavior.
- `autovacuum_max_workers` (3): max parallel workers; increase for large databases with many tables.
- `autovacuum_naptime` (1min): sleep between launcher cycles.
- `autovacuum_vacuum_threshold` (50) + `autovacuum_vacuum_scale_factor` (0.2): trigger formula.
- `autovacuum_analyze_threshold` + `autovacuum_analyze_scale_factor`: ANALYZE trigger.
- `autovacuum_vacuum_cost_delay` (2ms), `autovacuum_vacuum_cost_limit` (200): I/O throttle.
- `autovacuum_freeze_max_age` (200M): max age before forced freeze vacuum.

115. Explain **autovacuum_vacuum_scale_factor** and **autovacuum_vacuum_threshold**. How do these parameters determine when Autovacuum runs?
- Trigger: `n_dead_tup > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor × reltuples`
- Default: 50 + 0.2 × row_count — for a 10M-row table, triggers when 2,000,050 rows are dead.
- Problem: 20% of a 100M-row table is 20M dead rows before autovacuum kicks in — excessive bloat.
- Fix for large tables: `ALTER TABLE big_table SET (autovacuum_vacuum_scale_factor = 0.01, autovacuum_vacuum_threshold = 1000);`

116. Explain **autovacuum_freeze_max_age**. Why is it one of the most important PostgreSQL maintenance parameters?
- When a table's `relfrozenxid` is older than `autovacuum_freeze_max_age` XIDs, autovacuum is forced to run a full-table freeze scan regardless of dead tuple count.
- Default 200M XIDs; at ~2B XIDs before wraparound, this leaves only 1.8B XIDs of headroom — fine for most workloads.
- If you lower `autovacuum_freeze_max_age` too aggressively, freeze vacuums run too frequently.
- If this is disabled or broken and the table's age approaches 2B, PostgreSQL will shut down and refuse to start in `--single` mode to prevent data corruption.

117. Explain **vacuum_cost_limit** and **vacuum_cost_delay**. How do they reduce I/O impact during vacuum operations?
- After each unit of work, vacuum accumulates cost points; when accumulated cost exceeds `vacuum_cost_limit`, it sleeps for `vacuum_cost_delay` ms.
- This prevents vacuum from monopolizing I/O at the expense of user queries.
- Autovacuum: `autovacuum_vacuum_cost_limit` (default 200) and `autovacuum_vacuum_cost_delay` (default 2ms).
- For maintenance windows, disable throttling: `ALTER TABLE t SET (autovacuum_vacuum_cost_delay = 0);` or `SET vacuum_cost_delay = 0;` in a manual VACUUM session.

118. Explain the locking behavior of **VACUUM** and **VACUUM FULL**. Why does VACUUM FULL require an Access Exclusive Lock?
- Regular `VACUUM`: takes `ShareUpdateExclusiveLock` — allows concurrent reads and writes; only blocks DDL.
- `VACUUM FULL`: takes `AccessExclusiveLock` — blocks all access including reads; rewrites the entire table into a new file; this is why it's dangerous in production.
- `pg_repack` achieves the same space reclamation as VACUUM FULL using only brief locks at completion (swapping the new and old table files).

119. Explain how to identify table and index bloat in a production database.
```sql
-- Dead tuple ratio (quick signal)
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric/NULLIF(n_live_tup,0)*100,2) AS dead_pct
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Actual bloat with pgstattuple extension
SELECT * FROM pgstattuple('schema.tablename');

-- Index bloat
SELECT * FROM pgstattuple('schema.indexname');
```
- Also use the community `check_postgres` bloat query for estimation without the extension.

120. Explain the purpose of **REINDEX CONCURRENTLY**. How does it differ from a regular REINDEX?
- `REINDEX CONCURRENTLY`: builds the new index alongside the old one, keeps old index live for queries; swaps atomically at the end; only takes brief locks.
- Regular `REINDEX`: takes `AccessShareLock` (for table) + `AccessExclusiveLock` on the index — blocks writes during the rebuild.
- `REINDEX CONCURRENTLY` is slower and uses more space (old + new index exist simultaneously) but is safe for production.
- Caveat: cannot be run inside a transaction block.

## Locking, Blocking & Deadlocks

121. Explain PostgreSQL locking mechanisms. What types of locks can PostgreSQL acquire automatically?
- **Table-level locks** (heavyweight): `ACCESS SHARE` (SELECT), `ROW SHARE` (SELECT FOR UPDATE), `ROW EXCLUSIVE` (INSERT/UPDATE/DELETE), `SHARE UPDATE EXCLUSIVE` (VACUUM), `SHARE` (CREATE INDEX), `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` (DROP, TRUNCATE, ALTER TABLE).
- **Row-level locks**: not stored in lock table; stored in the tuple's `infomask` bits; track via `pg_locks` join to `pg_stat_activity`.
- **Advisory locks**: application-managed; `pg_try_advisory_lock()` for cooperative locking outside transaction boundaries.

122. Explain the difference between lightweight locks, heavyweight locks, and row-level locks.
- **LWLocks** (lightweight): protect shared memory data structures (buffer pool, WAL, catalog); microsecond duration; not exposed to users; contention visible in `pg_stat_activity.wait_event_type = 'LWLock'`.
- **Heavyweight locks**: transaction-duration table/relation/page locks; recorded in `pg_locks`; MVCC-aware; can deadlock.
- **Row-level locks**: stored in tuple header (xmax + infomask); don't appear in `pg_locks` unless multi-transaction; scale without lock manager overhead.

123. Explain how PostgreSQL detects deadlocks and resolves them automatically.
- After waiting `deadlock_timeout` (1s default) for a lock, the backend runs a deadlock detection cycle.
- Builds a wait-for graph from `pg_locks`; checks for cycles.
- If a cycle is found, PostgreSQL chooses one transaction to abort (the one that triggered the check) and sends it `ERROR: deadlock detected`.
- The aborted transaction releases its locks, breaking the deadlock for the others.

124. Explain the purpose of **pg_locks**. How do you use it to troubleshoot blocking sessions?
```sql
SELECT bl.pid AS blocked_pid, a.query AS blocked_query,
       kl.pid AS blocking_pid, ka.query AS blocking_query
FROM pg_locks bl
JOIN pg_stat_activity a  ON a.pid = bl.pid
JOIN pg_locks kl         ON kl.transactionid = bl.transactionid AND kl.pid != bl.pid
JOIN pg_stat_activity ka ON ka.pid = kl.pid
WHERE NOT bl.granted;
```
- `NOT granted = true` means the row is a waiter; find the matching granted lock holder to identify the blocker.

125. Explain how to identify long-running queries in PostgreSQL.
```sql
SELECT pid, now() - query_start AS duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;
```
- Also check for `idle in transaction` state — these hold locks without executing queries and are particularly dangerous.

126. Explain how to identify blocking sessions and blocked sessions.
```sql
SELECT pid, query, state, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocked_by
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
- `pg_blocking_pids()` (PG9.6+) returns an array of PIDs blocking this session.

127. Explain the safest way to terminate a PostgreSQL backend process. What is the difference between **pg_cancel_backend()** and **pg_terminate_backend()**?
- `pg_cancel_backend(pid)`: sends `SIGINT` to the backend; cancels the current query; session remains alive; least disruptive.
- `pg_terminate_backend(pid)`: sends `SIGTERM`; disconnects the client; transaction is rolled back.
- Always try cancel first; escalate to terminate only if cancel doesn't work (e.g., backend is stuck in a non-interruptible state).
- Never use `kill -9` on a backend — it can corrupt shared memory state.

## Monitoring & PgBouncer

128. Explain the purpose of **pg_stat_activity**. Which columns do you frequently use during troubleshooting?
- Shows one row per backend process with connection and query state.
- Key columns: `pid`, `usename`, `datname`, `state` (active/idle/idle in transaction), `query`, `query_start`, `wait_event_type`, `wait_event`, `client_addr`.
- `wait_event_type = 'Lock'` + `wait_event = 'relation'` → table-level blocking.
- `state = 'idle in transaction'` + old `query_start` → session holding locks without doing work.

129. Explain the purpose of **pg_stat_statements**. How does it help identify expensive SQL statements?
- Extension that normalizes and aggregates query performance across executions.
```sql
SELECT query, calls, total_exec_time, mean_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
```
- Use `total_exec_time` for overall cost to the system; `mean_exec_time` for per-execution tuning; `calls` to identify frequently executed cheap queries that add up.
- Reset with `SELECT pg_stat_statements_reset();` after tuning to see fresh data.

130. Explain the purpose of **pg_stat_database**, **pg_stat_bgwriter**, **pg_stat_replication**, and **pg_stat_io**. What information does each provide?
- `pg_stat_database`: per-database connections, transactions, cache hit ratio, deadlocks, temp file usage.
- `pg_stat_bgwriter`: checkpoint counts and timing, buffers written by bgwriter vs backends (backend writes = pressure on shared_buffers).
- `pg_stat_replication`: per-standby WAL send/flush/replay positions, sync state, lag intervals.
- `pg_stat_io` (PG16+): granular I/O accounting by backend type, object type, context — replaces/supplements `pg_stat_bgwriter`.

131. Explain the key health metrics you continuously monitor in a production PostgreSQL environment.
- Replication lag (replay_lag from `pg_stat_replication`).
- Connection utilization (`numbackends / max_connections`).
- Cache hit ratio (`blks_hit / (blks_hit + blks_read)` from `pg_stat_database`) — should be > 99%.
- Long-running queries and idle-in-transaction sessions.
- Autovacuum: tables not vacuumed recently, XID age approaching limits.
- WAL generation rate and archiver success/failure.
- Checkpoint frequency and spread (buffers_backend in `pg_stat_bgwriter`).

132. Explain how you monitor replication lag, connection usage, WAL generation, checkpoints, and Autovacuum activity.
```sql
-- Replication lag
SELECT replay_lag FROM pg_stat_replication;
-- Connections
SELECT count(*) FROM pg_stat_activity WHERE state != 'idle';
-- WAL rate
SELECT pg_current_wal_lsn();  -- compare over time
-- Checkpoints
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint FROM pg_stat_bgwriter;
-- Autovacuum
SELECT relname, last_autovacuum, n_dead_tup FROM pg_stat_user_tables
WHERE n_dead_tup > 10000 ORDER BY n_dead_tup DESC;
```

133. Explain the purpose of **PgBouncer**. Why is connection pooling important for PostgreSQL?
- PgBouncer is a connection pooler that maintains a pool of server connections and multiplexes many client connections through them.
- PostgreSQL's process-per-connection model means each connection consumes ~5–10 MB of RAM and has fork() overhead; at 1000+ connections, performance degrades.
- PgBouncer decouples client connections from server connections; 1000 app clients can share 50 PostgreSQL backends.

134. Explain the difference between **Session Pooling**, **Transaction Pooling**, and **Statement Pooling** in PgBouncer. What are the advantages and limitations of each mode?
- **Session Pooling**: server connection assigned for the life of the client session; safest, but low multiplexing — barely better than direct connections.
- **Transaction Pooling**: server connection released after each transaction; best multiplexing; most apps work fine; incompatible with `SET` session params, prepared statements (without `server_reset_query`), advisory locks.
- **Statement Pooling**: connection released after each statement; only for autocommit apps; barely practical.
- Production default: Transaction Pooling with careful application review.

135. Explain how to size a PgBouncer deployment for a high-concurrency PostgreSQL application.
- `max_client_conn`: total clients PgBouncer accepts; set to max your app can generate.
- `default_pool_size`: server connections per database+user pair; target 2–4× CPU cores on the DB server for CPU-bound workloads.
- Rule of thumb: keep PostgreSQL `max_connections` ≤ 4× CPU cores; let PgBouncer handle the rest.
- Multiple PgBouncer instances for HA and horizontal scaling; they're stateless so any number can point to the same PostgreSQL.

## Upgrades, Linux & Automation

136. Explain the complete process of upgrading PostgreSQL from one major version to another. What planning activities should be completed before the upgrade?
- Pre-upgrade: run `pg_upgrade --check` to identify incompatibilities; test on staging; check extension compatibility; review deprecated features; notify application teams.
- Choose method: `pg_upgrade` (fastest, in-place), dump/restore (safest, slowest), logical replication (near-zero downtime).
- For `pg_upgrade`:
```bash
pg_upgrade -b /usr/lib/postgresql/14/bin -B /usr/lib/postgresql/16/bin \
           -d /var/lib/pgsql/14/data -D /var/lib/pgsql/16/data
```
- Post-upgrade: run `ANALYZE` on all databases (`vacuumdb --all --analyze-only`), recheck statistics, monitor for plan regressions.

137. Explain the difference between **pg_upgrade**, dump-and-restore, and logical replication-based upgrades. When would you choose each approach?
- `pg_upgrade`: renames/links data files; fast (minutes for TB databases); requires downtime (stop old, upgrade, start new); safest for straightforward upgrades.
- Dump/restore: `pg_dumpall` + `psql`; slowest (hours/days for large DBs); full compatibility guarantee; downtime = dump time + restore time.
- Logical replication: stream changes from old to new version cluster simultaneously; cutover is near-zero downtime; complex setup; requires schema pre-creation on target; best for large databases with strict downtime SLA.

138. Explain the pre-upgrade and post-upgrade validation checks you perform during a PostgreSQL major version upgrade.
- Pre: `pg_upgrade --check`; extension versions available in new cluster; deprecated SQL/GUC usage; application connections drainable.
- Post: row counts on key tables; `ANALYZE` cluster-wide; query plan review on critical queries; verify all extensions load; replication re-established; monitoring dashboards back to baseline.

139. Explain how PostgreSQL minor version upgrades differ from major version upgrades.
- Minor version (e.g., 16.1 → 16.3): binary-compatible; replace binaries and restart; no data directory changes; no pg_upgrade needed.
- Major version (e.g., 15 → 16): on-disk format changes; requires `pg_upgrade`, dump/restore, or logical replication; extensions may need updating; catalog schema changes.

140. Explain how you minimize application downtime during PostgreSQL upgrades and schema changes.
- For upgrades: use logical replication to keep new cluster in sync; test cutover; switch connection string; downtime = DNS/VIP failover time.
- For schema changes: use `CREATE INDEX CONCURRENTLY`; add nullable columns first (PG11+ makes NOT NULL with DEFAULT instant); use `ALTER TABLE ... ADD CONSTRAINT ... NOT VALID` + `VALIDATE CONSTRAINT` in a separate transaction.
- Always test in staging with production-sized data; measure lock acquisition time before production.

141. Explain the Linux commands you frequently use while administering PostgreSQL servers.
```bash
ps aux | grep postgres       # process list
ss -tlnp | grep 5432         # port listening check
iostat -xz 1 5               # disk I/O
free -h                      # memory
df -h                        # disk usage
du -sh $PGDATA/pg_wal/       # WAL directory size
lsof -p <pid>                # open file handles for a backend
strace -p <pid>              # syscall tracing for stuck process
```

142. Explain how **top**, **vmstat**, **iostat**, **sar**, **free**, **df**, **du**, **lsof**, and **ss** help diagnose PostgreSQL performance issues.
- `top`/`htop`: CPU per PID — identify runaway backends.
- `vmstat 1`: `si`/`so` (swap in/out) = memory pressure; `wa` = I/O wait.
- `iostat -xz 1`: `%util`, `await`, `r/s`, `w/s` per device — identify I/O bottleneck.
- `sar -u/-r/-d`: historical CPU, memory, disk — compare current vs. baseline.
- `free -h`: check available memory and swap usage.
- `df -h` / `du -sh`: disk space on `pg_wal`, `base/`, log directories.
- `lsof`: open file count per backend; check for too many open files.
- `ss -s`: connection counts; `ss -tlnp` for listening sockets.

143. Explain how PostgreSQL interacts with the Linux page cache. How does this influence memory tuning?
- PostgreSQL reads data files via `read()`; Linux caches the pages in the page cache automatically.
- Shared buffers is a second copy of some pages — double buffering can waste RAM.
- `effective_cache_size` should reflect shared_buffers + available page cache to guide the planner.
- `direct_io` is not used by PostgreSQL; it relies entirely on OS buffering outside shared_buffers.
- Use `vmstat` and `/proc/meminfo` to measure actual page cache size; tune `shared_buffers` based on workload profile (OLTP: moderate shared_buffers, rely on page cache for read-heavy; DW: larger shared_buffers).

144. Explain how Huge Pages improve PostgreSQL performance. What checks do you perform after enabling them?
- Huge Pages (2 MB vs 4 KB) reduce TLB pressure for the large shared memory segment; fewer page table entries = lower context switch overhead.
- Configure: `vm.nr_hugepages` in `/etc/sysctl.conf`; set `huge_pages = on` in `postgresql.conf`.
```bash
# Calculate required huge pages
grep -i hugepagesize /proc/meminfo   # typically 2048 kB
# Required = shared_memory_size_in_bytes / 2MB
```
- Post-enable checks: PostgreSQL starts successfully; `grep HugePages /proc/meminfo` shows pages in use; no OOM events; confirm `shared_memory_size` in `pg_file_settings`.

145. Explain your approach to capacity planning for PostgreSQL. Which metrics help you predict CPU, memory, storage, and connection growth?
- CPU: trend `pg_stat_statements.total_exec_time` growth; watch for TPS increase rate.
- Memory: `shared_buffers` utilization; `work_mem` spill-to-disk frequency; OS free memory trend.
- Storage: `pg_database_size()` growth rate; WAL generation rate; bloat accumulation.
- Connections: peak `numbackends / max_connections` ratio; PgBouncer queue depth.
- Build dashboards with 30/60/90-day trends; alert at 70% thresholds for storage and connections.

146. Explain the maintenance activities you perform daily, weekly, and monthly on production PostgreSQL servers.
- Daily: check autovacuum activity, replication lag, connection counts, disk usage, backup success, XID age, long-running queries, archiver failures.
- Weekly: verify backup restorability (restore test); review `pg_stat_statements` for new expensive queries; check bloat on top 10 tables.
- Monthly: validate PITR procedure; review capacity trends; check for PostgreSQL minor version releases; audit user permissions; review and tune autovacuum settings.

147. Explain your process for reviewing SQL queries and schema changes before they are deployed to production.
- Run `EXPLAIN (ANALYZE, BUFFERS)` on staging with production-sized data.
- For DDL: classify the lock level required (e.g., `ALTER TABLE ADD COLUMN` vs `ADD COLUMN NOT NULL`); estimate lock acquisition time.
- Check for missing indexes on new JOIN conditions or WHERE predicates.
- Use `lock_timeout` and `statement_timeout` on DDL deployments to fail fast rather than holding locks indefinitely.
- For large table changes: prefer `CREATE INDEX CONCURRENTLY`, `NOT VALID` constraints, table rewrites via `pg_repack`.

148. Explain the common causes of PostgreSQL performance degradation. How do you systematically narrow down the root cause?
- Start with: `pg_stat_activity` (blocking? long queries?), `pg_stat_statements` (new expensive queries?), `pg_stat_bgwriter` (checkpoint pressure?).
- Check OS: `vmstat` (swap?), `iostat` (I/O saturation?), `free` (OOM risk?).
- Check autovacuum: bloat? XID age? dead tuples causing seq scans?
- Check replication: if using hot standby, lag causing stale reads?
- Compare `EXPLAIN` output before/after — look for plan changes due to stale statistics.

149. Describe the most challenging PostgreSQL production issue you have handled. What was the root cause, how did you resolve it, and what preventive measures did you implement afterward?
- Structure your answer: describe the symptom (latency spike, outage, data loss risk), your diagnostic sequence (what you checked first), root cause identified, immediate fix, and post-incident changes.
- Strong answers involve: XID wraparound avoidance, replication slot WAL accumulation, autovacuum unable to keep up during bulk loads, or a schema change taking an unexpected lock.
- Show you understand the prevention: monitoring alerts, autovacuum tuning, slot monitoring, lock_timeout policies.

## Cloud & Migration

150. A migration from on-premises PostgreSQL to AWS RDS must be completed with minimal downtime. Which migration approach would you recommend and why?
- Recommend logical replication (AWS DMS or native PG logical replication) for near-zero downtime.
- Process: snapshot initial data to RDS, keep logical replication running to catch up, verify data consistency, switch application connection string during a brief cutover window.
- DMS handles type mapping and schema conversion; native logical rep requires identical schemas and PG version compatibility.
- Fallback: if logical rep is not feasible (extensions, superuser requirements), use pg_basebackup + WAL streaming to an EC2 PostgreSQL, then promote and switch.

151. Your organization plans to migrate a 10 TB on-premises PostgreSQL database to AWS RDS with less than one hour of downtime. How would you design the migration strategy?
- Phase 1: set up logical replication from on-prem to RDS target; let it catch up (days if needed) while production continues.
- Phase 2: verify consistent row counts and checksums on key tables; test the RDS instance under load.
- Phase 3: during cutover window — drain writes (maintenance mode), verify replication fully caught up (lag = 0), update application connection strings, test, go live.
- 1-hour window is achievable if replication lag is near-zero before the window opens.
- Watch for: extensions not available on RDS (`pg_repack`, `pg_audit`), superuser-required operations, replication slot limits.

152. An application running on Amazon RDS PostgreSQL experiences sudden performance degradation, but you do not have operating system access. How would your troubleshooting approach differ from self-managed PostgreSQL?
- No OS access: no `top`, `iostat`, `vmstat` — rely entirely on CloudWatch metrics (CPUUtilization, ReadIOPS, WriteIOPS, FreeStorageSpace, DatabaseConnections, ReadLatency).
- PostgreSQL level: `pg_stat_activity`, `pg_stat_statements`, `pg_stat_user_tables`, `pg_locks` — same as self-managed.
- Check RDS Performance Insights for top SQL and wait events.
- Check Enhanced Monitoring (if enabled) for per-OS-process view.
- Common RDS-specific causes: storage burst credit exhaustion (gp2 IOPS), parameter group change, Multi-AZ failover, maintenance event.

153. Management asks whether Amazon Aurora PostgreSQL or Amazon RDS PostgreSQL is the better choice for a high-volume OLTP system. What architectural differences would influence your recommendation?
- Aurora: shared distributed storage across 3 AZs (6 copies); storage auto-scales to 128 TiB; faster failover (< 30s vs RDS ~60s); Aurora Serverless option; higher cost per GB.
- RDS: standard EBS-backed storage; gp3 gives predictable IOPS; more control over instance type and storage; lower cost for steady workloads.
- For high-volume OLTP with strict HA: Aurora — faster failover, no storage management, better read replica scaling.
- For predictable workload with cost sensitivity: RDS gp3 with Multi-AZ may be sufficient.

154. Management asks whether remaining on self-managed PostgreSQL or migrating to a managed cloud database service is the better long-term strategy. How would you evaluate the operational, financial, performance, and reliability trade-offs?
- Operational: managed removes OS patching, minor upgrades, hardware failure; DBA focus shifts to query tuning, schema design, capacity planning.
- Financial: managed services cost 2–4× self-managed IaaS at scale; break-even depends on DBA headcount and time saved.
- Performance: self-managed gives full hardware control (NVMe, huge pages, kernel tuning); managed locks you into instance types.
- Reliability: RDS Multi-AZ / Aurora give 99.95%+ SLA with automated failover; self-managed with Patroni can match this but requires expertise.
- Recommendation: managed for teams without dedicated DBAs or at scale < a few TB; self-managed or hybrid for cost-sensitive large-scale deployments.

## Real-World Troubleshooting Scenarios

155. Correlation statistics are ignored for a heavily clustered table.
- Happens when `pg_stats.correlation` is high (data physically ordered like its index) but the planner still chooses a seq scan.
- Check: `SELECT attname, correlation FROM pg_stats WHERE tablename = 'mytable';`
- If correlation is high but index scan is avoided, check `random_page_cost` — may be too high for SSD; set to 1.1.
- Also check if the predicate selectivity is low enough that seq scan is genuinely cheaper.
- Run `ANALYZE` to refresh correlation statistics; confirm statistics target is adequate.

156. Random I/O increases despite unchanged indexes.
- Likely causes: table bloat has grown (more pages to read), data growth pushed working set out of shared_buffers/page cache, or a plan changed from index to seq scan.
- Check `pg_stat_user_tables` for `n_dead_tup` growth; check `pg_statio_user_tables` for `heap_blks_read` increase.
- Verify plan with `EXPLAIN (BUFFERS, ANALYZE)` — look for shared hit vs read ratio.
- Check if OS page cache is under pressure (`vmstat`, `free`).

157. Bulk INSERT operations invalidate correlation assumptions.
- After a large bulk load in random order, `CLUSTER` ordering is destroyed; correlation drops toward 0.
- Result: index scans become less efficient (random I/O); planner may correctly prefer seq scan.
- Fix: run `ANALYZE` post-load to update statistics; consider `CLUSTER ON index` to restore physical ordering (takes `AccessExclusiveLock`); or accept that post-bulk-load index efficiency is lower until rows are accessed and evicted/reinserted in order.

158. Correlation differs significantly between primary and standby.
- Statistics (`pg_stats`) on the standby are not updated by streaming replication — they reflect the last ANALYZE run on the standby.
- If the standby runs ANALYZE independently, statistics may diverge from primary.
- In older PostgreSQL: autovacuum on standby runs ANALYZE (PG10+); check `last_autoanalyze` on both sides.
- Impact: query plans on the standby may differ from primary, leading to poor read replica performance.
- Fix: ensure autovacuum/ANALYZE runs regularly on standby; or use `pg_dump`+`pg_restore --schema-only` to copy statistics.

159. A PostgreSQL production server suddenly stops accepting new client connections after a restart. Walk me through your troubleshooting approach.
- Check if PostgreSQL is listening: `ss -tlnp | grep 5432` — not listening means it failed to start.
- Check logs: `journalctl -u postgresql-16 -n 50` for startup errors.
- Check `pg_hba.conf` syntax: `pg_hba_file_rules` view or `pg_ctl reload` error output.
- Verify `listen_addresses` is set correctly (not just `localhost`).
- Check `max_connections` — if set very low, slots could be exhausted by reserved superuser connections.
- Check filesystem: `df -h` — full disk prevents startup.

160. Users report that the database server is running, but applications cannot connect remotely. How would you isolate whether the issue is related to PostgreSQL configuration, authentication, networking, or the operating system?
- Network: `telnet <db_host> 5432` or `nc -zv <db_host> 5432` from app server — if fails, it's network/firewall.
- PostgreSQL listening: `ss -tlnp | grep 5432` on DB server; check `listen_addresses` includes the right IP.
- Authentication: try connecting locally with `psql -U username -d db` — if works locally, check `pg_hba.conf` for `host` rules.
- Logs: `pg_log` will show rejected connection attempts with specific error codes.
- SSL: if app requires SSL and cert is expired or `ssl=off`, connections will fail with SSL-specific errors.

161. During a maintenance window, someone accidentally modifies the **pg_hba.conf** file and users immediately lose access. How would you restore connectivity while maintaining security?
- Connect locally as postgres OS user via Unix socket (bypasses TCP/IP pg_hba rules): `psql -U postgres`
- Restore previous pg_hba.conf from backup or version control.
- Reload: `SELECT pg_reload_conf();`
- If no backup: add a minimal safe rule (scram-sha-256 from known IPs) and reload.
- Post-incident: put pg_hba.conf under version control; use `pg_hba_file_rules` view to validate before reload.

162. Multiple PostgreSQL instances are running on the same Linux server, but one of them refuses to start because the port is already in use. How would you identify the conflicting process and resolve the issue?
```bash
ss -tlnp | grep 5433         # find what's using the port
lsof -i :5433                # alternative
cat /var/lib/pgsql/cluster2/postmaster.pid  # stale PID file?
```
- If a stale `postmaster.pid` exists from a crash, remove it and retry.
- If another PostgreSQL owns the port: check `postgresql.conf` of all instances for port conflicts; update the port of the failing instance.
- If a non-PostgreSQL process owns the port: identify it with `ss -tlnp` and either move it or change PostgreSQL's port.

163. Application response time suddenly increases after reducing **shared_buffers** during a configuration change. How would you determine whether the memory change is responsible?
```sql
SELECT blks_hit, blks_read,
       round(blks_hit::numeric/(blks_hit+blks_read)*100,2) AS cache_hit_pct
FROM pg_stat_database WHERE datname = 'mydb';
```
- If cache hit rate dropped after the change, shared_buffers reduction is the cause — more I/O from disk.
- Cross-check `pg_stat_bgwriter.buffers_backend` — increase means backends are evicting buffers directly (shared_buffers too small).
- Revert `shared_buffers` to previous value and reload/restart; confirm cache hit rate recovers.

164. The **pg_wal** directory grows continuously until the filesystem reaches 100% utilization. What are the most likely causes?
- Archive command failing: WAL segments pile up waiting to be archived; check `pg_stat_archiver.failed_count`.
- Inactive/stale replication slot holding `restart_lsn`: `SELECT slot_name, restart_lsn, active FROM pg_replication_slots;`
- `wal_keep_size` set too high; or `max_wal_size` too high combined with lots of writes.
- Long-running transaction preventing WAL recycling.
- Fix: fix archive command or drop stale slot; if disk is critically full, drop slot or temporarily set `archive_mode=off` then fix root cause.

165. A heavily updated table continues growing every day even though users delete large amounts of data. What could be causing this behavior?
- Dead tuples are accumulating because VACUUM isn't reclaiming space fast enough.
- Check: `SELECT n_dead_tup, last_autovacuum, autovacuum_count FROM pg_stat_user_tables WHERE relname = 'mytable';`
- Possible causes: autovacuum disabled on the table, long-running transactions blocking VACUUM, autovacuum too slow (cost throttle too conservative), table bloat from HOT update failures.
- Fix: tune autovacuum for the table or run `VACUUM ANALYZE mytable` manually; check for idle-in-transaction sessions.

166. Users complain that UPDATE statements have become progressively slower over several months. How would you determine whether table bloat is responsible?
```sql
-- Check dead tuple ratio
SELECT n_live_tup, n_dead_tup FROM pg_stat_user_tables WHERE relname = 'orders';
-- Check actual table size vs expected
SELECT pg_size_pretty(pg_relation_size('orders')) AS heap_size;
SELECT pg_size_pretty(pg_total_relation_size('orders')) AS total_size;
-- pgstattuple for accurate bloat
SELECT * FROM pgstattuple('orders');
```
- Compare `table_len` vs `tuple_len + free_space` — large `dead_tuple_len` confirms bloat.
- Fix: `VACUUM ANALYZE orders;` for mild bloat; `pg_repack -t orders` for severe bloat without downtime.

167. A table has reached hundreds of gigabytes, but most of its rows have already been deleted. How would you reclaim disk space with minimal disruption?
- First verify bloat: `pgstattuple` or community bloat query.
- Option 1 (online, preferred): `pg_repack --table=schema.tablename` — rebuilds the table concurrently, only brief lock at swap.
- Option 2 (offline, maintenance window): `VACUUM FULL schema.tablename` — rewrites table, returns space to OS, requires `AccessExclusiveLock`.
- Run `ANALYZE` after; update autovacuum settings to prevent recurrence.

168. Your monitoring system reports that transaction IDs are approaching wraparound limits. What immediate actions would you take?
```sql
-- Check XID age per database
SELECT datname, age(datfrozenxid), datfrozenxid FROM pg_database ORDER BY age DESC;
-- Check per-table
SELECT relname, age(relfrozenxid) FROM pg_class WHERE relkind='r' ORDER BY age DESC LIMIT 20;
```
- If age > 1.5B: immediate manual VACUUM FREEZE on the oldest tables.
- `VACUUM FREEZE ANALYZE tablename;` — forces freezing regardless of age thresholds.
- Suspend large write workloads temporarily to slow XID consumption.
- After stabilization: fix autovacuum_freeze_max_age and ensure autovacuum isn't lagging.

169. A DBA disables Autovacuum temporarily during data loading but forgets to re-enable it. Several weeks later, performance degrades significantly. What problems would you expect to find?
- Massive dead tuple accumulation → table bloat → larger table scans.
- Stale statistics → bad query plans (misestimated row counts, wrong join methods).
- XID age growing uncontrolled → risk of approaching wraparound.
- FSM not updated → INSERT performance degrades (backend must search for free space).
- Fix: re-enable autovacuum immediately; run `VACUUM ANALYZE` on affected tables manually; check XID ages.

170. A VACUUM FULL operation has been running for hours, and users are reporting application outages. Why is this happening, and what alternatives could have been considered?
- `VACUUM FULL` holds `AccessExclusiveLock` for its entire duration — blocks all reads and writes on the table.
- On a large table with lots of bloat, this can run for hours.
- To recover: you cannot cancel and have partial progress — you must let it complete or kill it (and the table goes back to its original state).
- Alternatives that should have been used: `pg_repack` (online, brief lock at swap), `CLUSTER` + reattach (shorter window), partitioning and dropping old partitions.

171. A standby server suddenly falls twenty minutes behind the primary during peak business hours. Walk me through your investigation.
```sql
-- On primary
SELECT client_addr, replay_lag, state FROM pg_stat_replication;
-- On standby
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
```
- Check standby I/O: is it CPU/disk-saturated? (`iostat`, `top`)
- Check for large transactions or DDL generating burst WAL on the primary.
- Check WAL receiver status on standby: `pg_stat_wal_receiver`.
- Check network between primary and standby: `ss`, ping latency, packet loss.
- If sync replication: primary may be waiting — consider temporarily switching to async.

172. A standby server reports that it is connected, but WAL replay has stopped. What could cause this situation?
- A recovery conflict: standby query conflicting with a WAL cleanup operation (e.g., a query reading a page about to be vacuumed away on primary). Check `pg_stat_activity` on standby for `recovery conflict` wait events.
- WAL corruption: check PostgreSQL logs for checksum or WAL record errors.
- A replication pause: `pg_wal_replay_pause()` may have been called. Check: `SELECT pg_is_wal_replay_paused();` → resume with `SELECT pg_wal_replay_resume();`
- A pending 2PC transaction on the standby blocking replay.

173. A replication slot has prevented WAL removal, causing the primary server to run out of disk space. How would you safely resolve the issue?
```sql
-- Identify the problem slot
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
FROM pg_replication_slots ORDER BY restart_lsn;
```
- If slot is inactive and subscriber is clearly gone: drop it immediately.
  ```sql
  SELECT pg_drop_replication_slot('slot_name');
  ```
- If active subscriber is far behind: pause writes if possible, or temporarily increase disk (add volume); then fix the subscriber.
- Preventive: set `max_slot_wal_keep_size` (PG13+) to cap WAL retained per slot.

174. A standby server is accidentally promoted while the original primary is still online. How would you prevent split-brain and safely recover the cluster?
- Immediate: disconnect the accidentally promoted standby from all application traffic; do not let any writes reach it.
- Stop one of them immediately — choose which has more complete data (usually the original primary or the one that received more writes).
- If using Patroni: `patronictl pause` stops any automatic actions; manually demote the accidental primary by stopping its PostgreSQL.
- Recover: run `pg_rewind` on the node being demoted to resync it with the authoritative primary; restart as standby.
- Prevent: use fencing (STONITH) in HA setup to ensure the old primary is killed before promotion completes.

175. After a failover, the old primary comes back online. What steps are required before it can safely rejoin the cluster?
- Do NOT start it as a primary — it will diverge from the new primary's timeline.
- Run `pg_rewind`: compares old primary's data with new primary, copies diverged blocks, resets to the divergence point.
```bash
pg_rewind --target-pgdata=$PGDATA --source-server="host=new_primary dbname=postgres"
```
- Create `standby.signal`, configure `primary_conninfo` pointing to new primary.
- Start PostgreSQL — it will replay WAL from new primary and catch up.

176. A query that normally completes in 200 milliseconds suddenly starts taking 30 seconds. Walk me through your troubleshooting process.
- Run `EXPLAIN (ANALYZE, BUFFERS)` on the query; compare with a cached good plan (if available).
- Check for plan change: seq scan where index scan used to be? Hash join replaced by nested loop?
- Check `pg_stat_activity` for blocking: is the query waiting for a lock?
- Check `pg_stat_statements` — is total_exec_time/calls consistent with 30s or is it new?
- Check for stale statistics: `SELECT last_analyze, last_autoanalyze FROM pg_stat_user_tables WHERE relname='table';`
- Check bloat: has the table grown significantly?
- Fix based on finding: run ANALYZE, create missing index, kill blocking session, or rewrite query.

177. A production query switches from an Index Scan to a Sequential Scan immediately after a deployment. What would you investigate?
- Did the deployment include a large data load or DELETE? → stale statistics → run ANALYZE.
- Did the deployment add or modify an index? → check `pg_indexes` for existence; check index validity.
- Did a config parameter change? → `random_page_cost`, `enable_indexscan`, `enable_seqscan`.
- Did the data distribution change dramatically? (e.g., column that was 10% true is now 80% true) → index scan genuinely less efficient; add a partial index or accept seq scan.
- Force test: `SET enable_seqscan = off; EXPLAIN ...` to see the index scan cost.

178. Users complain that every query has become slower, including simple primary key lookups. What would you isolate the root cause?
- System-wide slowdown (including PK lookups) rules out query-specific issues.
- Check `pg_stat_activity` for blocking or very high active connection count.
- Check OS: disk I/O saturation (`iostat`), memory pressure (swap usage in `vmstat`), CPU saturation.
- Check for checkpoint storm: `pg_stat_bgwriter.buffers_checkpoint` spiking.
- Check if OOM Killer ran: `journalctl -k | grep -i oom`
- Check replication: if read traffic hits a lagging standby, stale reads could appear as "slowness".
- Check for table-level lock held by a long DDL: `pg_locks` join `pg_stat_activity`.

179. A newly created index is never used by the optimizer. What possible explanations would you investigate?
- Predicate selectivity: the indexed column has few distinct values (e.g., boolean), so a seq scan is cheaper.
- `random_page_cost` too high: set `random_page_cost = 1.1` for SSD and re-check.
- Statistics stale: run `ANALYZE` and re-run `EXPLAIN`.
- Index definition mismatch: query uses `LOWER(col)` but index is on `col` (not functional index).
- Partial index condition doesn't match the query's WHERE clause.
- Small table: fewer pages than `min_parallel_table_scan_size`, planner prefers seq scan.
- Validate with: `SET enable_seqscan = off; EXPLAIN ...` to see if the index would be used and its cost.

180. An application team reports that users are randomly experiencing lock timeouts during business hours. How would you identify the blocking session and resolve the issue without affecting other users?
```sql
-- Find blockers
SELECT pid, query, state, wait_event, pg_blocking_pids(pid)
FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
- Identify the blocking PID; check what it's doing (long UPDATE? idle in transaction?).
- If idle in transaction: `SELECT pg_cancel_backend(<pid>);` — cancels current query, session stays.
- If unresponsive: `SELECT pg_terminate_backend(<pid>);`
- Long-term: set `idle_in_transaction_session_timeout` to automatically kill idle-in-transaction sessions; set `lock_timeout` on DDL deployments.

181. A deployment executes an ALTER TABLE statement, and suddenly every application request begins waiting. What happened, and how would you recover?
- `ALTER TABLE` acquires `AccessExclusiveLock`; if it's waiting behind an existing long-running query, it blocks and all subsequent queries that need even a `AccessShareLock` (SELECT) queue behind it.
- Identify the blocking chain: `pg_blocking_pids()` on the ALTER TABLE pid; find the original blocker.
- Recover: cancel or terminate the blocking query; the ALTER TABLE will then proceed and release its lock.
- Prevention: use `lock_timeout` on DDL (`SET lock_timeout = '5s'; ALTER TABLE ...`) — fails fast rather than building a queue.

182. Deadlocks begin appearing in the PostgreSQL logs after a new application release. How would you identify the conflicting transactions and work with developers to prevent future deadlocks?
- Extract deadlock details from logs: `grep "deadlock detected" postgresql.log` — the log includes the full transaction state and involved PIDs.
- Identify the lock order: Transaction A holds lock on table X, waits for Y; Transaction B holds Y, waits for X.
- Work with developers to: enforce a consistent lock acquisition order; use `SELECT ... FOR UPDATE` explicitly before updates; reduce transaction duration.
- Use `pg_locks` during the next occurrence to capture in-flight state.
- Consider `deadlock_timeout` lowering to detect deadlocks faster (default 1s is fine for most cases).

183. A backend process has been waiting on a lock for over fifteen minutes. At what point would you cancel the query, terminate the session, or allow it to continue?
- First: understand what it's waiting for (`pg_blocking_pids`, `pg_locks`) — is the blocker a legitimate long transaction or a stuck session?
- If the blocker is stuck (idle in transaction with no recent activity): cancel the blocker, not the waiter.
- If the waiter itself is a critical operation (DDL, maintenance): allow it if the business impact is acceptable; set `lock_timeout` next time.
- If the waiter is an application query that users are waiting on: cancel it (`pg_cancel_backend`), let the app retry; fix the blocking issue.
- Never blindly terminate without understanding the blast radius — a terminated backend rolls back its transaction.

184. CPU utilization suddenly spikes to 100% at 2:00 AM every night. How would you determine whether the cause is PostgreSQL, the operating system, or another scheduled job?
- Check scheduled jobs: `crontab -l`, `systemctl list-timers` — is there a backup, VACUUM, or ETL job at 2 AM?
- `top` at 2:01 AM or `sar -u` historical data — which PID is consuming CPU?
- If PostgreSQL: check `pg_stat_activity` at that time; check `pg_stat_statements` for queries with high `max_exec_time`.
- Autovacuum anti-wraparound? Check `pg_stat_user_tables.last_autovacuum` around 2 AM.
- If OS: kernel process (`kswapd`, `jbd2`)? → memory or disk pressure.

185. The Linux Out-of-Memory (OOM) Killer terminates PostgreSQL. What configuration or workload issues could lead to this situation?
- `work_mem` set too high with many concurrent connections: total RAM usage = `max_connections × work_mem × sort_nodes_per_query` in worst case.
- `shared_buffers` + OS page cache + all backend `work_mem` exceeds physical RAM.
- `vm.overcommit_memory = 1` (default on many systems) allows overcommit; set to `2` with appropriate `vm.overcommit_ratio`.
- Check: `journalctl -k | grep -i oom` for which PID was killed and why.
- Fix: reduce `work_mem`; protect postmaster from OOM: `echo -17 > /proc/<postmaster_pid>/oom_score_adj`

186. Your monitoring dashboard reports that active connections have increased from 300 to 2000 within a few minutes. How would you determine whether this is an application issue or a database issue?
- Check `pg_stat_activity` breakdown: are connections `active` (DB is processing), `idle in transaction` (app holding connections open), or `idle` (connection pool leak)?
- `idle in transaction` spike → application code not committing/rolling back; or long transactions.
- `active` spike with long durations → DB-side problem (locking, slow queries).
- Pure `idle` spike → connection pool misconfiguration or connection leak in application.
- Check if PgBouncer pool is misconfigured (`pool_size` change?) or if a new application deployment happened.

187. Application users begin receiving **"too many connections"** errors. What steps would you take before increasing max_connections?
- Immediate: `SELECT count(*), state FROM pg_stat_activity GROUP BY state;` — identify idle connections consuming slots.
- Kill idle-in-transaction sessions: `pg_terminate_backend` for sessions idle > X minutes.
- Check if PgBouncer is bypassed (app connecting directly to PostgreSQL).
- Check `reserved_connections` (PG16+) / `superuser_reserved_connections` — these reserve slots for superusers, reducing available slots.
- If PgBouncer is in use: check `pool_size` and `max_client_conn`; may need to increase pool.
- Only increase `max_connections` as a last resort — it increases memory usage and context switching.

188. The application team requests increasing max_connections from 500 to 5000. How would you evaluate whether this is the right solution?
- 5000 connections × ~5–10 MB per backend = 25–50 GB RAM just for processes — not feasible on most servers.
- First question: why are 5000 connections needed? Is PgBouncer in use? If not, deploying PgBouncer should be the answer.
- With PgBouncer in transaction mode: keep `max_connections` at 500–1000 (PostgreSQL server), allow 5000 `max_client_conn` in PgBouncer.
- If PgBouncer is already in use and pool is saturated: increase `default_pool_size` in PgBouncer first; add PgBouncer instances.
- Raising `max_connections` requires a PostgreSQL restart and increases `shared_buffers` minimum — plan for it.

189. After enabling Transaction Pooling in PgBouncer, several applications begin failing unexpectedly. What application behavior could explain this problem?
- `SET` statements: in transaction pooling, session-level `SET` commands don't persist across transactions (different backend each time) — app may rely on `SET work_mem` per session.
- Prepared statements: `PREPARE`/`EXECUTE` at SQL level fail because prepared statements are backend-local; use PgBouncer's `server_reset_query` or disable prepared statements.
- Advisory locks: `pg_advisory_lock()` is session-scoped; in transaction pooling, lock is released when the connection returns to pool.
- `LISTEN`/`NOTIFY`: doesn't work in transaction pooling — requires session pooling.
- Fix: use `server_reset_query = DISCARD ALL`; or move incompatible features to session-pooling pools.

190. A PostgreSQL major version upgrade must be completed over a weekend with less than thirty minutes of downtime. How would you plan the migration?
- Use logical replication approach: stand up new version cluster in advance; replicate all tables.
- Weekend: let replication catch up (lag < 1s); put application in maintenance mode; verify lag = 0; update connection strings; switch to new cluster; smoke test; end maintenance mode.
- 30 minutes is achievable if pre-work is done: new cluster provisioned, schemas migrated, replication caught up, cutover rehearsed.
- Rollback plan: keep old cluster running for 24 hours post-cutover; point DNS/VIP back if critical issue found.
- pg_upgrade alternative: if DB is small, pg_upgrade + `--link` (hard links, near-instant) can complete in under 10 minutes.

191. A 500-million-row table requires a new NOT NULL column. How would you implement the change while minimizing downtime?
- In PostgreSQL 11+: `ALTER TABLE t ADD COLUMN col INT NOT NULL DEFAULT 0;` is instant if the default is a literal constant — stored in catalog, no table rewrite.
- For a non-constant default (e.g., `DEFAULT now()`): PG11+ still avoids rewrite for `NOT NULL DEFAULT expr` that is stable; verify with `EXPLAIN` after — check for `AccessExclusiveLock` duration.
- Pre-PG11 approach: add column as nullable (`ALTER TABLE ADD COLUMN col INT`); backfill in batches (`UPDATE ... WHERE id BETWEEN x AND y`); add `NOT NULL` constraint with `NOT VALID`; `VALIDATE CONSTRAINT` in a separate transaction; finally `SET NOT NULL`.
- Always test on staging with production-sized data first; measure lock duration before production deployment.

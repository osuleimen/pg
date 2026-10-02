# PostgreSQL Support Runbook

Scenario-based. Each scenario: **Diagnose -> Decide -> Fix -> Verify -> Report**.
Search with Ctrl+F by symptom or keyword. Every "Verify" step is a test that CAN fail.
Every command here must be checked against the actual server (paths, version, OS).

## Contents
- 0. Operating rules
- 1. First 10 minutes (orientation)
- 2. Linux: storage, OS tuning, software, ports, firewall, disk full
- 3. PostgreSQL install and base configuration (RHEL / Ubuntu)
- 4. Configuration changes: reload vs restart
- 5. PostgreSQL does not start
- 6. Roles, ownership, privileges, pg_hba
- 7. Incidents: locks, idle in transaction, connections, long queries
- 8. Performance: top queries, EXPLAIN, indexes, vacuum, statistics
- 9. Monitoring queries (health check)
- 10. Backup and restore
- 11. Replication
- 12. SQL patterns
- 13. Customer messages (templates)
- 14. Report templates
- 15. Final checklist before sending the report
- 16. Monitoring (what, tools, Nagios-style checks)
- 17. Statistics: activity vs planner
- 18. Patroni (concepts)
- 19. PgBouncer (install and configure)

---

## 0. Operating rules

1. First 5 minutes: read ALL tasks. Copy every requirement word for word into notes.md (REQ1..N).
2. First message: ask what is allowed (docs, notes, search, AI). Send all questions in one message.
3. Treat the server as production. Before any change: backup the file (`cp -p f f.bak.$(date +%F-%H%M)`).
4. Destructive command (stop, restart, drop, kill, rm, terminate): pause, say what it affects, ask if needed.
5. Restart only with approval and the restart template (section 13).
6. After every step: copy the key output into report.txt under the right heading.
7. "Done" only when every REQ has a verification output.
8. Status to the customer every 30 minutes.
9. Report sent at the deadline, even incomplete, with Open items.
10. Never claim what you did not test.

---

## 1. First 10 minutes (orientation)

```bash
cat /etc/os-release | head -3          # OS and version
hostname -I; ip -br a                  # IP addresses
nproc; free -h                         # CPU, memory
lsblk -f; df -hT                       # disks, filesystems, free space
uptime                                 # load
systemctl list-units --type=service | grep -i -E 'postgres|pgbouncer|patroni|haproxy'
ss -ltnp                               # listening ports and processes
sudo -u postgres psql -c "select version();"
```
Ubuntu only:
```bash
pg_lsclusters                          # version, cluster, port, status, data dir, log file
```
Inside psql:
```sql
SHOW data_directory; SHOW config_file; SHOW hba_file;
SELECT pg_is_in_recovery();            -- t = standby, f = primary
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;
SELECT name, setting, source FROM pg_settings WHERE source NOT IN ('default','override') ORDER BY name;
```
Several servers? Run this on each and note: role (primary/standby), IP, version.

### Path cheat sheet

| Item | Ubuntu / Debian | RHEL / Rocky (PGDG) |
|---|---|---|
| Config | /etc/postgresql/16/main/ | $PGDATA (e.g. /var/lib/pgsql/16/data) |
| Data | /var/lib/postgresql/16/main | /var/lib/pgsql/16/data |
| Service | postgresql@16-main | postgresql-16 |
| Binaries | /usr/lib/postgresql/16/bin | /usr/pgsql-16/bin |
| Log (default) | /var/log/postgresql/postgresql-16-main.log | $PGDATA/log/ |
| Cluster tools | pg_lsclusters, pg_ctlcluster, pg_createcluster | postgresql-16-setup |
| Firewall | ufw | firewalld |
| MAC | AppArmor | SELinux |
| Package files | dpkg -L pkg | rpm -ql pkg |

---
## 2. Linux

### 2.1 New disk: XFS, permanent mount
Diagnose:
```bash
lsblk -f            # find the EMPTY disk by size and no MOUNTPOINTS. Never touch the root disk.
```
Fix (whole disk, no partition, is fine; with partition use nvme1n1p1):
```bash
sudo mkfs.xfs /dev/nvme1n1                  # missing? Ubuntu: apt install xfsprogs
sudo blkid /dev/nvme1n1                     # copy UUID
sudo mkdir -p /pgdata
sudo cp -p /etc/fstab /etc/fstab.bak.$(date +%F-%H%M)
echo 'UUID=<uuid> /pgdata xfs defaults,nofail 0 0' | sudo tee -a /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
```
Verify:
```bash
findmnt /pgdata            # SOURCE, FSTYPE xfs
df -hT /pgdata
grep pgdata /etc/fstab
```
Report: "Formatted <device> as XFS and mounted at /pgdata via /etc/fstab using the UUID,
because NVMe device names on EC2 can change after a reboot. nofail prevents a boot failure if the
volume is missing."
Note: if RHEL + SELinux and PostgreSQL data will be here, see 3.3.

### 2.2 OS tuning, persistent (sysctl)
Diagnose (show BEFORE values):
```bash
sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio vm.overcommit_memory
```
Fix:
```bash
sudo tee /etc/sysctl.d/99-postgresql.conf <<'X'
vm.swappiness = 1
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
X
sudo sysctl --system            # apply NOW
```
Verify (AFTER values, active now):
```bash
sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio
cat /proc/sys/vm/swappiness
```
Report: before -> after, why, "active now and persistent via /etc/sysctl.d/99-postgresql.conf".
Why: swappiness 1 = avoid swapping DB memory; lower dirty ratios = smaller, more frequent flushes,
fewer IO stalls on large-memory hosts. vm.overcommit_memory = 2 is often recommended for
PostgreSQL but can cause failures of other processes: recommend, do not apply without approval.

### 2.3 Transparent Huge Pages (THP), persistent
Diagnose:
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled     # [always] = on, [never] = off
cat /sys/kernel/mm/transparent_hugepage/defrag
```
Fix: systemd unit (no reboot needed):
```bash
sudo tee /etc/systemd/system/disable-thp.service <<'X'
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=basic.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled; echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
X
sudo systemctl daemon-reload
sudo systemctl enable --now disable-thp.service
```
Verify:
```bash
systemctl is-enabled disable-thp.service
cat /sys/kernel/mm/transparent_hugepage/enabled       # always madvise [never]
```
Alternative: kernel parameter transparent_hugepage=never in GRUB (needs a reboot) or a tuned
profile on RHEL. Report why you chose the unit: effective immediately, no reboot.
Why disable: THP causes latency spikes and memory bloat for database workloads.

### 2.4 Install software from a repository and start at boot (generic)
```bash
# Ubuntu                               # RHEL
sudo apt update                        sudo dnf install -y <pkg>
sudo apt install -y <pkg>
systemctl status <service>
sudo systemctl enable --now <service>
```
Verify: `systemctl is-enabled <service>; systemctl is-active <service>; ss -ltnp | grep <port>`
plus a functional test (connect, curl, login).
Report (Nagios example style): what, where (URL/port), credentials, what was also installed
(dependencies, repositories), what starts at boot, what was configured.

### 2.5 Port already in use
```bash
sudo ss -ltnp | grep ':5432'
ps -fp <pid>
systemctl status <pid>                 # which unit owns the process
```
Decide: ask the customer before stopping anything that is not yours.
Fix (after approval): `sudo systemctl disable --now <unit>`. Do not delete files.
Report: what was on the port, who approved, what was done, files kept.

### 2.6 Firewall
```bash
# Ubuntu
sudo ufw status verbose
sudo ufw allow from 172.31.0.0/16 to any port 5432 proto tcp
# RHEL
sudo firewall-cmd --list-all
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="172.31.0.0/16" port port="5432" protocol="tcp" accept'
sudo firewall-cmd --reload
```
Only change the firewall if it is active and the task requires it. Changing AWS security groups
is a change outside the host: ask and report.

### 2.7 Disk full
Diagnose:
```bash
df -hT; df -i                                   # space and inodes
sudo du -xh --max-depth=1 /pgdata | sort -h     # what is big
sudo lsof +L1 2>/dev/null | head                # deleted files still held open
```
```sql
SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir();               -- WAL size
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;                                           -- inactive slot keeps WAL
SELECT archived_count, failed_count, last_failed_wal, last_failed_time FROM pg_stat_archiver;
```
Typical causes: inactive replication slot, failing archive_command, huge logs, temp files, bloat.
NEVER delete files in pg_wal by hand. Fix the cause: drop an unused slot (after approval),
fix archiving, rotate/compress logs, move logs to another disk.
Report: cause with evidence, what was freed, how to prevent (monitoring, alert on disk %).

### 2.8 Scheduled job (cron / systemd timer)
```bash
sudo -u postgres crontab -e
# m h dom mon dow command
30 2 * * * /usr/bin/pg_dump -Fc -d shop -f /backup/shop_$(date +\%F).dump
```
Note: `%` must be escaped as `\%` in crontab. Verify: `sudo -u postgres crontab -l`, run the
command once manually, check the output file.

---
## 3. PostgreSQL install and base configuration

### 3.1 Ubuntu: install PostgreSQL 16 from PGDG
```bash
sudo apt install -y curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
. /etc/os-release
echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt ${VERSION_CODENAME}-pgdg main" | sudo tee /etc/apt/sources.list.d/pgdg.list
sudo apt update
apt-cache policy postgresql-16        # Candidate must come from apt.postgresql.org
sudo apt install -y postgresql-16 postgresql-contrib-16
```
(apt-key is deprecated; signed-by is the current method.)
Ubuntu creates cluster 16/main automatically on /var/lib/postgresql/16/main.

### 3.2 Ubuntu: data directory on /pgdata
New server, no data yet (cleanest):
```bash
sudo pg_dropcluster 16 main --stop
sudo install -d -o postgres -g postgres -m 700 /pgdata/16
sudo pg_createcluster 16 main -d /pgdata/16/main -p 5432 --start
pg_lsclusters
```
Existing cluster with data (downtime, approval):
```bash
sudo systemctl stop postgresql@16-main
sudo rsync -a /var/lib/postgresql/16/main/ /pgdata/16/main/
sudo chown -R postgres:postgres /pgdata/16 && sudo chmod 700 /pgdata/16/main
# /etc/postgresql/16/main/postgresql.conf: data_directory = '/pgdata/16/main'
sudo systemctl start postgresql@16-main
```
Verify: `pg_lsclusters` (status online, data dir), `SHOW data_directory;`.
Keep the old directory until the customer confirms, then remove (mention in report).
Tip: use a subdirectory (/pgdata/16/main), not the mount point itself: if the disk is not mounted,
PostgreSQL fails instead of silently starting on the root disk.

### 3.3 RHEL/Rocky: install PostgreSQL 16 from PGDG, data on /pgdata
```bash
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql16-server postgresql16-contrib
sudo install -d -o postgres -g postgres -m 700 /pgdata/16/data
sudo systemctl edit postgresql-16          # add the two lines below
#   [Service]
#   Environment=PGDATA=/pgdata/16/data
sudo semanage fcontext -a -t postgresql_db_t "/pgdata(/.*)?"   # dnf install policycoreutils-python-utils
sudo restorecon -Rv /pgdata
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable --now postgresql-16
```
Never edit /usr/lib/systemd/system/*.service: package upgrades overwrite it.
Verify: `systemctl cat postgresql-16` (drop-in visible), `SHOW data_directory;`, `getenforce`,
`ls -Zd /pgdata/16/data`, `sudo ausearch -m avc -ts recent` (expect no matches).

### 3.4 listen_addresses and port
```sql
SELECT name, setting, context FROM pg_settings WHERE name IN ('listen_addresses','port');
-- context = postmaster -> restart required
```
Edit postgresql.conf (Ubuntu: /etc/postgresql/16/main/postgresql.conf):
```
listen_addresses = 'localhost,172.31.70.8'     # not '*': only what is needed
port = 5432
```
```sql
SELECT pg_reload_conf();
SELECT name, setting, pending_restart FROM pg_settings WHERE pending_restart;   -- listed = waits for restart
SELECT sourcefile, sourceline, name, setting, error FROM pg_file_settings WHERE error IS NOT NULL;
```
Ask approval (template 13.3), then restart and verify:
```bash
sudo ss -ltnp | grep 5432            # 127.0.0.1:5432 and 172.31.70.8:5432
```
Non-default port on RHEL with SELinux: `sudo semanage port -a -t postgresql_port_t -p tcp 5433`.

### 3.5 Autostart after reboot
```bash
# Ubuntu: cluster start is controlled by start.conf (auto)
cat /etc/postgresql/16/main/start.conf
systemctl is-enabled postgresql postgresql@16-main
# RHEL
systemctl is-enabled postgresql-16
```

### 3.6 Memory settings (starting point, explain in report)
| Parameter | Starting value | Restart? |
|---|---|---|
| shared_buffers | 25% of RAM | yes |
| effective_cache_size | 50-75% of RAM | no (reload) |
| work_mem | 4-64MB (per sort/hash, per query node) | no |
| maintenance_work_mem | 256MB-1GB | no |
| max_connections | as needed; use a pooler instead of large values | yes |
Always check with `SELECT name, setting, unit, context FROM pg_settings WHERE name = '...';`

### 3.7 Useful logging (all reload, no restart)
```sql
ALTER SYSTEM SET log_min_duration_statement = '300ms';
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET log_temp_files = 0;
ALTER SYSTEM SET log_autovacuum_min_duration = '1s';
SELECT pg_reload_conf();
SHOW log_min_duration_statement;
```
ALTER SYSTEM writes to postgresql.auto.conf in the data directory; it overrides postgresql.conf.
Verify slow log: `SELECT pg_sleep(0.4);` then find "duration:" in the log.

---

## 4. Configuration changes: reload vs restart

Decision rule: `pg_settings.context`
| context | How to apply |
|---|---|
| postmaster | restart |
| sighup | reload: `SELECT pg_reload_conf();` or `systemctl reload ...` |
| superuser-backend / backend | new sessions after reload |
| superuser / user | reload, or SET in a session, or ALTER ROLE/DATABASE ... SET |

Workflow:
```bash
sudo cp -p <conf> <conf>.bak.$(date +%F-%H%M)
```
```sql
SELECT name, setting, context FROM pg_settings WHERE name = 'x';
-- edit file or ALTER SYSTEM SET x = ...;
SELECT pg_reload_conf();
SELECT sourcefile, sourceline, error FROM pg_file_settings WHERE error IS NOT NULL;  -- must be empty
SELECT name, setting, pending_restart FROM pg_settings WHERE name = 'x';
```
- Values for postmaster parameters: after reload `pending_restart = t` and in pg_file_settings
  `applied = f` -> expected until restart.
- pg_hba.conf changes: reload only. Check `SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;`
- Before `ALTER SYSTEM SET shared_preload_libraries`: check the current value, list must include
  all existing libraries, e.g. 'pg_stat_statements,auto_explain'.

Rollback: restore the .bak file (or `ALTER SYSTEM RESET x;`), reload/restart.

---

## 5. PostgreSQL does not start

Diagnose in this order:
```bash
systemctl status postgresql@16-main        # RHEL: postgresql-16
journalctl -u postgresql@16-main -n 50 --no-pager
sudo tail -n 50 /var/log/postgresql/postgresql-16-main.log   # RHEL: $PGDATA/log/*.log
```
| Log message | Cause | Fix |
|---|---|---|
| syntax error in file ... line N | bad postgresql.conf | fix line / restore .bak |
| could not bind ... Address already in use | port busy | ss -ltnp, see 2.5 |
| data directory ... has invalid permissions | not 0700 or wrong owner | chown postgres:postgres, chmod 700 |
| could not open file ... Permission denied (RHEL) | SELinux label | semanage fcontext + restorecon, see 3.3 |
| directory ... does not exist / No such file | disk not mounted | findmnt, mount -a |
| No space left on device | disk full | see 2.7 |
| could not map anonymous shared memory | not enough memory / huge_pages=on | lower shared_buffers or huge_pages=try |
| invalid value for parameter | typo in value/unit | fix, check units (MB, not M) |
| FATAL: lock file "postmaster.pid" already exists | another postmaster running | ps -ef ; never delete if the PID is alive |
Verify after fix: service active, `psql -c 'select 1'`, `ss -ltnp`.
Report: symptom, root cause with the log line, fix, verification, how to prevent.

---
## 6. Roles, ownership, privileges, pg_hba

Key facts:
- Everything is a role. `CREATE USER` = `CREATE ROLE ... LOGIN`. `CREATE ROLE` default is NOLOGIN.
- Superuser bypasses all checks: test privileges only as the target role.
- The owner can do everything with its objects; "read-only" means "not the owner".
- `GRANT ... ON ALL TABLES` = one-time snapshot. Future objects need `ALTER DEFAULT PRIVILEGES`.
- `ALTER DEFAULT PRIVILEGES` applies to objects created by the role in `FOR ROLE`. Always
  `FOR ROLE <owner>`. Per database: run it after `\c <db>`.
- Serial/identity columns use sequences: writers need `USAGE` on sequences.
- PostgreSQL 15+: PUBLIC has no CREATE on schema public. PUBLIC still has CONNECT and TEMPORARY
  on databases by default.

### 6.1 Owner role + database + load schema/dump with the right owner
```sql
CREATE ROLE app_owner NOLOGIN;
CREATE DATABASE shop OWNER app_owner;
```
Load an SQL file as the owner (objects get the right owner from the start):
```bash
sudo -u postgres psql -d shop -v ON_ERROR_STOP=1 -c 'SET ROLE app_owner' -f /opt/app/seed.sql
```
Without ON_ERROR_STOP psql continues after errors: read the whole output, report every ERROR.
Load a custom-format dump:
```bash
sudo -u postgres pg_restore -d shop --no-owner --role=app_owner /tmp/x.dump
pg_restore -l /tmp/x.dump | grep -E 'ACL|OWNER'    # what grants/owners the dump contains
```
Add `--no-privileges` if the dump has GRANTs to roles that do not exist here.
Fix ownership after the fact (existing objects):
```sql
ALTER TABLE public.customers OWNER TO app_owner;   -- owned sequences and indexes follow
```
Never `REASSIGN OWNED BY postgres` (system objects).
Verify:
```sql
\l shop
SELECT schemaname, tablename, tableowner FROM pg_tables WHERE schemaname = 'public';
SELECT sequencename, sequenceowner FROM pg_sequences WHERE schemaname = 'public';
```

### 6.2 Read-only role (including future tables)
```sql
CREATE ROLE report_ro LOGIN PASSWORD 'xxx';
\c shop
GRANT CONNECT ON DATABASE shop TO report_ro;
GRANT USAGE ON SCHEMA public TO report_ro;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO report_ro;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA public GRANT SELECT ON TABLES TO report_ro;
```
Optional stricter read-only: `REVOKE TEMPORARY ON DATABASE shop FROM PUBLIC;` (check that no
application role needs temp tables; grant them explicitly first).

### 6.3 Read-write application role (including future tables and sequences)
```sql
CREATE ROLE shop_app LOGIN PASSWORD 'xxx';
\c shop
GRANT CONNECT ON DATABASE shop TO shop_app;
GRANT USAGE ON SCHEMA public TO shop_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO shop_app;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO shop_app;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO shop_app;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA public
  GRANT USAGE ON SEQUENCES TO shop_app;
```
Wrong entry to remove (e.g. FOR ROLE of the wrong role):
```sql
ALTER DEFAULT PRIVILEGES FOR ROLE shop_app IN SCHEMA public
  REVOKE SELECT, INSERT, UPDATE, DELETE ON TABLES FROM shop_app;
```

### 6.4 Verification of privileges (tests that can fail)
As postgres:
```sql
\c shop
-- \ddp: the Owner column must be app_owner
\ddp
\dp public.*
SELECT has_table_privilege('shop_app','public.orders','INSERT');
SELECT has_sequence_privilege('shop_app','public.orders_id_seq','USAGE');
SET ROLE app_owner; CREATE TABLE public.future_test(id int); RESET ROLE;
SELECT has_table_privilege('shop_app','public.future_test','INSERT');   -- expect t
DROP TABLE public.future_test;
```
As the role itself, over TCP, like the application (changes rolled back):
```sql
BEGIN;
INSERT INTO orders (customer_id, status, total) VALUES (1, 'new', 1.00) RETURNING id;  -- no explicit id!
ROLLBACK;
CREATE TABLE hack(id int);        -- expect: permission denied for schema public
```
Never insert test rows with explicit ids into sequence-backed tables: it causes duplicate key
errors later. If it happened: delete the row, or `SELECT setval('orders_id_seq', (SELECT max(id) FROM orders));`

### 6.5 pg_hba: "only from subnet"
First matching line wins. Put these lines ABOVE the generic lines:
```
# TYPE  DATABASE  USER      ADDRESS          METHOD
host    shop      shop_app  172.31.0.0/16    scram-sha-256
host    all       shop_app  0.0.0.0/0        reject
host    all       shop_app  ::/0             reject
local   all       shop_app                   reject
```
```sql
SELECT line_number, error FROM pg_hba_file_rules WHERE error IS NOT NULL;   -- must be empty
SELECT pg_reload_conf();
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules ORDER BY line_number;
```
Verify all three paths:
```bash
psql -h 172.31.70.8 -U shop_app -d shop -c 'select 1'     # allowed (host inside subnet)
psql -h 127.0.0.1   -U shop_app -d shop -c 'select 1'     # rejected (TCP localhost)
sudo -u postgres psql -U shop_app -d shop -c 'select 1'   # rejected (unix socket)
```
Unix socket (no -h) and TCP 127.0.0.1 are DIFFERENT pg_hba lines.
Check `SHOW password_encryption;` = scram-sha-256. Set passwords with `\password <role>`.

### 6.6 Drop a role that owns or has privileges
```sql
\c <each database>
SELECT pg_describe_object(classid, objid, objsubid), deptype
FROM pg_shdepend WHERE refobjid = 'olduser'::regrole;
REASSIGN OWNED BY olduser TO app_owner;
DROP OWNED BY olduser;          -- after REASSIGN: removes only privileges
\c postgres
DROP ROLE olduser;
```

### 6.7 Report snippet
"shop_app can log in with a password (scram-sha-256) and has SELECT, INSERT, UPDATE, DELETE on all
tables and USAGE on all sequences in shop. Default privileges FOR ROLE app_owner grant the same on
future tables and sequences. pg_hba allows shop_app only from 172.31.0.0/16 and rejects it from any
other address, including localhost and the local socket. Verified: <outputs>. The password is: <...>."

---
## 7. Incidents

### 7.1 "Everything hangs" / blocking / idle in transaction
Diagnose (one query):
```sql
SELECT pid, usename, application_name, client_addr, state, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocked_by,
       now() - xact_start  AS xact_age,
       now() - state_change AS state_age,
       left(query, 60) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY xact_start NULLS LAST;
```
Read it: the root blocker has empty `blocked_by` and others point to it. `idle in transaction` +
old `xact_age` = forgotten transaction. A waiting ALTER TABLE blocks even SELECTs behind it.

Decide: `pg_cancel_backend(pid)` = cancel the running query, session stays.
`pg_terminate_backend(pid)` = close the session, its transaction is rolled back.
For `idle in transaction` cancel does nothing (no running query) -> terminate.
NEVER `kill -9` a backend: the postmaster drops ALL sessions and runs crash recovery.

Ask approval (template 13.2), then:
```sql
SELECT pg_terminate_backend(<pid>);
```
Verify: rerun the diagnose query; nothing blocked; waiting statements finished.

Prevent (no restart, context user):
```sql
ALTER ROLE <app_role> SET idle_in_transaction_session_timeout = '5min';   -- new sessions only
SELECT rolname, rolconfig FROM pg_roles WHERE rolname = '<app_role>';     -- verify
```
Recommend: `SET lock_timeout = '5s'` in migrations; app fix (always commit/rollback); alert on
transactions older than N minutes.

### 7.2 Lock details
```sql
SELECT l.pid, l.locktype, l.mode, l.granted, l.relation::regclass AS relation, a.state, left(a.query,50)
FROM pg_locks l JOIN pg_stat_activity a USING (pid)
WHERE l.relation IS NOT NULL AND a.datname = current_database()
ORDER BY l.granted, l.pid;
```
Log future lock waits: `ALTER SYSTEM SET log_lock_waits = on; SELECT pg_reload_conf();`

### 7.3 Too many connections
Messages: `sorry, too many clients already` (max_connections) or
`too many connections for role "x"` (role limit) or `... for database "x"` (database limit).
```sql
SHOW max_connections;
SELECT count(*) FROM pg_stat_activity;
SELECT usename, application_name, state, count(*) FROM pg_stat_activity GROUP BY 1,2,3 ORDER BY 4 DESC;
SELECT rolname, rolconnlimit FROM pg_roles WHERE rolconnlimit <> -1;
SELECT datname, datconnlimit FROM pg_database WHERE datconnlimit <> -1;
```
Fix without restart: find stuck/idle sessions (7.1), raise the role limit
`ALTER ROLE x CONNECTION LIMIT n;` (checked at connect time, immediate).
Raising max_connections needs a restart and memory: recommend a pooler (PgBouncer) instead.
Plain `idle` connections of a pool are normal; `idle in transaction` is not.

### 7.4 Long-running query
```sql
SELECT pid, usename, now() - query_start AS runtime, state, left(query, 80)
FROM pg_stat_activity
WHERE state = 'active' AND pid <> pg_backend_pid()
ORDER BY runtime DESC;
```
Cancel after approval: `SELECT pg_cancel_backend(<pid>);`
Prevent: `ALTER ROLE x SET statement_timeout = '5min';` (agree the value with the customer).

---

## 8. Performance

### 8.1 Find the queries with the most load
pg_stat_statements needs `shared_preload_libraries` (restart) + `CREATE EXTENSION` in the database.
```sql
SHOW shared_preload_libraries;                 -- check current value first
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';   -- keep existing entries!
SELECT pg_reload_conf();
SELECT name, setting, pending_restart FROM pg_settings WHERE name = 'shared_preload_libraries';
-- approval + restart, then:
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```
Top by total time:
```sql
SELECT calls,
       round(total_exec_time::numeric)    AS total_ms,
       round(mean_exec_time::numeric, 1)  AS mean_ms,
       round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 1) AS pct,
       rows,
       shared_blks_hit + shared_blks_read AS blocks,
       left(query, 80) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```
Slow log (log_min_duration_statement) shows long queries only; frequent fast queries can be the
main load. Without a restart: temporarily lower log_min_duration_statement (reload) or sample
pg_stat_activity, and say it in the report.
After a fix: `SELECT pg_stat_statements_reset();`, wait a few minutes, measure again.

### 8.2 EXPLAIN checklist
```sql
EXPLAIN (ANALYZE, BUFFERS) <query>;   -- ANALYZE executes it! For DML: BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;
```
1. Read bottom-up. Find the node with the biggest actual time (x loops).
2. Estimated rows vs actual rows: >10x off -> statistics (ANALYZE) or non-sargable condition.
3. Seq Scan + big "Rows Removed by Filter" -> missing index or function on the column.
4. "Index Cond" = filtered inside the index (good). "Filter" = checked after reading rows.
5. Buffers: shared hit (memory) / read (disk). Compare plans by Buffers, not only time.
6. Sort Method: external merge Disk -> work_mem too small for this query.
7. JIT section with large Total -> short query paying for JIT compilation.
8. Compare after warm-up (run 2-3 times).

### 8.3 Index
Choose columns: equality columns first, then range/sort column. INCLUDE for index-only scans.
```sql
SET lock_timeout = '5s';
CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders (status, created_at);
SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE indrelid = 'orders'::regclass;
```
CONCURRENTLY: does not block writes, slower, cannot run inside a transaction block.
If it fails the index stays INVALID (not used, but maintained on writes):
`DROP INDEX CONCURRENTLY idx_name;` and create again.
Plain CREATE INDEX blocks writes (not reads) for the whole build.
Verify: EXPLAIN before/after, same result before/after (count/sum side by side), index size:
`SELECT pg_size_pretty(pg_relation_size('idx_orders_status_created'));`

### 8.4 Rewrite non-sargable conditions
| Bad (no index use) | Good |
|---|---|
| `to_char(created_at,'YYYY-MM-DD') = '2026-09-01'` | `created_at >= DATE '2026-09-01' AND created_at < DATE '2026-09-02'` |
| `date(created_at) = '2026-09-01'` | same as above |
| `EXTRACT(YEAR FROM created_at) = 2026` | `created_at >= DATE '2026-01-01' AND created_at < DATE '2027-01-01'` |
| `lower(email) = 'x'` | expression index `ON t (lower(email))` |
| `created_at = DATE '2026-09-01'` | WRONG: equals midnight only |
Always prove the result did not change: run old and new query, compare count/sum.

### 8.5 Vacuum, dead tuples, autovacuum settings
```sql
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 1) AS dead_pct,
       last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

SELECT c.relname, c.reloptions, t.reloptions AS toast_reloptions
FROM pg_class c LEFT JOIN pg_class t ON t.oid = c.reltoastrelid
WHERE c.reloptions IS NOT NULL OR t.reloptions IS NOT NULL;
```
Fix (online):
```sql
ALTER TABLE t RESET (autovacuum_enabled, toast.autovacuum_enabled);
VACUUM (ANALYZE, VERBOSE) t;
```
VACUUM does not return space to the OS (except empty pages at the end). VACUUM FULL rewrites the
table with ACCESS EXCLUSIVE lock (downtime): propose pg_repack instead (online).
Never analyzed table -> wrong estimates -> bad plans: `ANALYZE t;`

### 8.6 Statistics
```sql
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables ORDER BY n_mod_since_analyze DESC LIMIT 10;
```

---

## 9. Monitoring queries (health check)

```sql
-- Database sizes
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY pg_database_size(datname) DESC;
-- Biggest tables (with indexes and TOAST)
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total,
       pg_size_pretty(pg_relation_size(relid)) AS table_only
FROM pg_statio_user_tables ORDER BY pg_total_relation_size(relid) DESC LIMIT 10;
-- Cache hit ratio per database (OLTP target > 99%)
SELECT datname, round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database WHERE datname IS NOT NULL;
-- Unused indexes (since stats reset)
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;
-- Transaction ID age (wraparound risk)
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;
-- Settings changed from default
SELECT name, setting, source FROM pg_settings WHERE source NOT IN ('default','override');
```
Deeper collection without installing anything: Percona pg_gather (SQL script run with psql).

---
## 10. Backup and restore

### 10.1 Logical: pg_dump / pg_restore
```bash
sudo -u postgres pg_dump -Fc -d shop -f /backup/shop_$(date +%F_%H%M).dump
sudo -u postgres pg_dumpall --globals-only -f /backup/globals.sql      # roles, not in pg_dump
pg_restore -l /backup/shop.dump | head                                   # verify the file is readable
```
Restore into a NEW database (never over production without approval):
```bash
sudo -u postgres createdb -O app_owner shop_restore
sudo -u postgres pg_restore -d shop_restore -j 2 --no-owner --role=app_owner /backup/shop.dump
```
Recover one table: `pg_restore -d shop_restore -t orders ...` then copy the rows back.
Verify: row counts old vs restored.

### 10.2 Physical: pg_basebackup
```bash
sudo -u postgres pg_basebackup -D /backup/base_$(date +%F_%H%M) -Ft -z -X stream -P
```
Needs a replication line in pg_hba for the user (default: `local replication all peer`).
Verify: files exist, `backup_manifest` present; `pg_verifybackup` for plain format backups.

### 10.3 WAL archiving + PITR
Enable archiving (archive_mode = restart, archive_command = reload):
```
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
```
```bash
sudo install -d -o postgres -g postgres -m 700 /archive
```
Verify: `SELECT pg_switch_wal();` then `SELECT archived_count, failed_count, last_archived_wal FROM pg_stat_archiver;`
and `ls /archive`.

Point-in-time restore, safest: into a separate directory and port, not over production.
```bash
sudo -u postgres mkdir -m 700 /pgdata/restore
sudo -u postgres tar -xzf /backup/base_X/base.tar.gz   -C /pgdata/restore
sudo -u postgres tar -xzf /backup/base_X/pg_wal.tar.gz -C /pgdata/restore/pg_wal
sudo -u postgres touch /pgdata/restore/recovery.signal
```
Recovery settings. RHEL: the backup already has postgresql.conf and pg_hba.conf, append to
postgresql.auto.conf. Ubuntu: config lives in /etc, so create minimal files in the restore dir
(NEVER copy /etc/postgresql/.../postgresql.conf: its data_directory points to production).
```bash
# Ubuntu: minimal config for the temporary instance
sudo -u postgres tee /pgdata/restore/pg_hba.conf >/dev/null <<'X'
local all postgres peer
X
sudo -u postgres tee /pgdata/restore/postgresql.conf >/dev/null <<'X'
port = 5433
listen_addresses = ''
X
# both OS: recovery settings
sudo -u postgres tee -a /pgdata/restore/postgresql.auto.conf >/dev/null <<'X'
port = 5433
listen_addresses = ''
archive_mode = off
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-09-28 10:15:00+00'
recovery_target_action = 'promote'
X
sudo -u postgres /usr/lib/postgresql/16/bin/pg_ctl -D /pgdata/restore -l /tmp/restore.log start
# RHEL: /usr/pgsql-16/bin/pg_ctl
sudo -u postgres psql -p 5433 -c 'select pg_is_in_recovery();'
```
Check /tmp/restore.log for "recovery stopping before ..." and "database system is ready".
Then pg_dump the needed table from port 5433 and load it into production after approval.
Stop the temporary instance: `pg_ctl -D /pgdata/restore stop`.
Report: target time chosen and why, what was recovered, row counts, what was left in place.

### 10.4 pgBackRest (short)
```bash
sudo apt install -y pgbackrest          # or dnf install -y pgbackrest (PGDG repo)
ls /etc/pgbackrest* 2>/dev/null         # config: /etc/pgbackrest/pgbackrest.conf or /etc/pgbackrest.conf
```
```
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
start-fast=y

[main]
pg1-path=/pgdata/16/main
```
postgresql.conf: `archive_mode = on` (restart), `archive_command = 'pgbackrest --stanza=main archive-push %p'`
```bash
sudo install -d -o postgres -g postgres -m 750 /var/lib/pgbackrest
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main check
sudo -u postgres pgbackrest --stanza=main --type=full backup
sudo -u postgres pgbackrest --stanza=main info
# PITR restore (PostgreSQL stopped):
sudo -u postgres pgbackrest --stanza=main --delta --type=time "--target=2026-09-28 10:15:00+00" --target-action=promote restore
```

### 10.5 Recover deleted rows with PITR (end to end)
Goal: get back rows deleted by mistake without losing rows written after the incident.
Idea: restore a COPY of the cluster to the moment just before the DELETE, take the rows from
the copy, insert them into production. Production is never stopped or overwritten.

Step 1. Find the exact time of the incident.
```bash
sudo grep -n -i -E 'delete|truncate|drop' /var/lib/pgsql/16/data/log/*.log | tail -20   # RHEL
sudo grep -n -i -E 'delete|truncate|drop' /var/log/postgresql/*.log | tail -20          # Ubuntu
```
Works if statements are logged (log_statement = 'mod' or 'ddl', or the statement was slow).
The line prefix has the timestamp, e.g. `2026-09-30 17:02:11.123 UTC [4242] cleanup_job@billing LOG: statement: DELETE ...`
No log line? Ask the customer / application logs. Last resort: pg_waldump on archived WAL.
Target = a moment BEFORE the DELETE started, e.g. 1 second earlier: `2026-09-30 17:02:10+00`.

Step 2. Check that backup + archive cover the target.
```bash
ls -l /var/lib/pgsql/16/backups/
sudo tar -xzOf /var/lib/pgsql/16/backups/base_*/base.tar.gz backup_label   # START TIME must be < target
ls /var/lib/pgsql/16/archive | tail -3                                      # WAL continues after the target
df -h /var/lib/pgsql                                                        # space for a full copy
```

Step 3. Restore the copy into a separate directory on port 5433 (RHEL example).
```bash
sudo -u postgres mkdir -m 700 /var/lib/pgsql/16/restore
cd /var/lib/pgsql/16/backups/base_*/
sudo -u postgres tar -xzf base.tar.gz   -C /var/lib/pgsql/16/restore
sudo -u postgres tar -xzf pg_wal.tar.gz -C /var/lib/pgsql/16/restore/pg_wal
sudo -u postgres touch /var/lib/pgsql/16/restore/recovery.signal
sudo -u postgres tee -a /var/lib/pgsql/16/restore/postgresql.auto.conf >/dev/null <<'X'
port = 5433
listen_addresses = ''
archive_mode = off
restore_command = 'cp /var/lib/pgsql/16/archive/%f %p'
recovery_target_time = '2026-09-30 17:02:10+00'
recovery_target_action = 'promote'
X
sudo -u postgres /usr/pgsql-16/bin/pg_ctl -D /var/lib/pgsql/16/restore -l /tmp/restore.log start
sudo tail -20 /tmp/restore.log                       # early startup messages
sudo tail -30 /var/lib/pgsql/16/restore/log/*.log     # RHEL: logging_collector writes the rest here
```
archive_mode = off is essential: otherwise the copy writes WAL into the production archive.
Expected in the log: "starting point-in-time recovery to ...", "recovery stopping before commit of
transaction ...", "database system is ready to accept connections".
Ubuntu: create minimal postgresql.conf and pg_hba.conf in the restore dir (see 10.3).

Step 4. Check the copy.
```bash
sudo -u postgres psql -p 5433 -d billing -c "SELECT count(*), sum(amount) FROM payments WHERE customer_id BETWEEN 1000 AND 1999;"
sudo -u postgres psql -p 5432 -d billing -c "SELECT count(*), sum(amount) FROM payments WHERE customer_id BETWEEN 1000 AND 1999;"
```
Copy: the deleted rows are there. Production: 0, or only rows created after the incident.

Step 5. Export from the copy, import into production (keep original ids, skip existing ids).
```bash
sudo -u postgres psql -p 5433 -d billing -c "\copy (SELECT * FROM payments WHERE customer_id BETWEEN 1000 AND 1999) TO '/tmp/recovered_payments.csv' CSV"
wc -l /tmp/recovered_payments.csv
```
```sql
-- production, port 5432, as postgres
\c billing
BEGIN;
CREATE TEMP TABLE recovered (LIKE payments INCLUDING DEFAULTS);
\copy recovered FROM '/tmp/recovered_payments.csv' CSV
SELECT count(*) FROM recovered;
INSERT INTO payments
SELECT r.* FROM recovered r
WHERE NOT EXISTS (SELECT 1 FROM payments p WHERE p.id = r.id);
-- check the INSERT count = recovered count minus rows that already exist
SELECT count(*), sum(amount) FROM payments WHERE customer_id BETWEEN 1000 AND 1999;
COMMIT;      -- or ROLLBACK if numbers do not match
```
Why this is safe: original ids are lower than the current sequence value, so no conflict with new
rows; NOT EXISTS skips anything already present; rows created after the incident are not touched;
all in one transaction.

Step 6. Verify and clean up.
- production count for the range = copy count + rows created after the incident in that range;
- replica (if streaming) receives the rows automatically: check on the standby;
- stop the copy: `sudo -u postgres /usr/pgsql-16/bin/pg_ctl -D /var/lib/pgsql/16/restore stop`;
- delete /tmp/recovered_payments.csv (it contains customer financial data) and, after the customer
  confirms, the restore directory.

Report: incident time and how it was found (log line), target time and why, backup used,
restored count/sum, inserted count, verification numbers, what was cleaned up, recommendation
(log_statement = 'mod' for batch jobs, test cleanup jobs, keep backups verified).

---

## 11. Replication

### 11.1 Status on the primary
```sql
SELECT client_addr, application_name, state, sync_state,
       sent_lsn, replay_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) AS replay_lag_bytes,
       write_lag, flush_lag, replay_lag
FROM pg_stat_replication;
SELECT slot_name, slot_type, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;
```
### 11.2 Status on the standby
```sql
SELECT pg_is_in_recovery();                         -- t
SELECT status, sender_host, sender_port, latest_end_lsn FROM pg_stat_wal_receiver;
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       now() - pg_last_xact_replay_timestamp() AS replay_delay;   -- grows if the primary is idle
```
No row in pg_stat_wal_receiver = not streaming: read the standby log.
"requested WAL segment ... has already been removed" -> the standby fell behind: rebuild it
(11.3) and use a replication slot or WAL archive to prevent it.

### 11.3 Build (or rebuild) a streaming standby
Primary:
```sql
CREATE ROLE replicator REPLICATION LOGIN PASSWORD 'xxx';
```
pg_hba on primary: `host replication replicator <standby_ip>/32 scram-sha-256` + reload.
Standby (PostgreSQL stopped, data directory EMPTY - move the old one aside, do not delete):
```bash
sudo systemctl stop postgresql@16-main
sudo mv /var/lib/postgresql/16/main /var/lib/postgresql/16/main.old
sudo -u postgres pg_basebackup -h <primary_ip> -U replicator -D /var/lib/postgresql/16/main \
     -X stream -R -P -C -S standby1
sudo systemctl start postgresql@16-main
```
-R writes primary_conninfo + standby.signal; -C -S creates replication slot standby1 on the primary.
Verify: 11.1 on the primary shows the standby "streaming"; 11.2 on the standby.

### 11.4 Promote a standby
```sql
SELECT pg_promote();
SELECT pg_is_in_recovery();       -- f
```
Before promoting: make sure the old primary is down or fenced (split brain risk). Unused slots on
the old primary keep WAL forever: drop them after the switch (`SELECT pg_drop_replication_slot('x');`).
Queries cancelled on the standby with "conflict with recovery": hot_standby_feedback = on or
max_standby_streaming_delay (trade-offs: bloat on primary vs replay delay).

### 11.5 Standby is not streaming (troubleshooting)
On the standby, read the log first:
```bash
sudo tail -50 /var/lib/pgsql/16/data/log/*.log        # RHEL; Ubuntu: /var/log/postgresql/
```
| Standby log | Cause | Fix |
|---|---|---|
| password authentication failed for user "replicator" | wrong password in primary_conninfo, or changed on primary | correct primary_conninfo (below) or reset the password on the primary (ask) |
| no pg_hba.conf entry for replication connection | primary pg_hba lacks `host replication ...` for the standby IP | add line on primary + reload |
| could not connect ... Connection refused / timeout | primary down, listen_addresses, firewall, security group | check each |
| requested WAL segment ... has already been removed | standby too far behind | restore_command from archive, or rebuild (11.3) |
Fix primary_conninfo on the standby (PostgreSQL 13+: reload is enough):
```sql
-- on the standby
SELECT pg_is_in_recovery();
SHOW primary_conninfo;                                 -- the password is shown: do not paste it in reports
ALTER SYSTEM SET primary_conninfo = 'host=<primary_ip> port=5432 user=replicator password=<pw> application_name=db2';
SELECT pg_reload_conf();
SELECT status, sender_host FROM pg_stat_wal_receiver;   -- streaming
```
Check the file for duplicate primary_conninfo lines: the LAST one wins.
Verify on the primary: section 11.1 (state = streaming, lag bytes near 0).


---

## 12. SQL patterns

Method: REQ comments word for word + `GRAIN: one row = ...`. Grow the query step by step:
FROM/JOIN -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT. Control query at the end.

5 questions: grain? did JOIN multiply rows? row condition WHERE / group condition HAVING? NULL?
sort exactly as required + tie-breaker?

```sql
-- Aggregation per entity with filter on the group
SELECT c.name, count(*) AS orders, sum(o.total) AS total
FROM customers c JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= DATE '2026-09-01' AND o.created_at < DATE '2026-10-01'
  AND o.status = 'shipped'
GROUP BY c.id, c.name
HAVING count(*) >= 5
ORDER BY total DESC, c.name ASC;

-- Top N per group
SELECT * FROM (
  SELECT c.country, c.name, sum(o.total) AS total,
         ROW_NUMBER() OVER (PARTITION BY c.country ORDER BY sum(o.total) DESC, c.name) AS rn
  FROM customers c JOIN orders o ON o.customer_id = c.id
  GROUP BY c.country, c.id, c.name) s
WHERE rn <= 3 ORDER BY country, rn;

-- Latest row per group
SELECT DISTINCT ON (customer_id) customer_id, id, created_at, total
FROM orders ORDER BY customer_id, created_at DESC, id DESC;

-- Has X but not Y
SELECT c.id, c.name FROM customers c
WHERE EXISTS     (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'shipped')
  AND NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'cancelled')
ORDER BY c.id;

-- Include entities with zero (LEFT JOIN, condition on the right table in ON, count the right key)
SELECT c.id, c.name, count(o.id) AS orders_sept
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
                  AND o.created_at >= DATE '2026-09-01' AND o.created_at < DATE '2026-10-01'
GROUP BY c.id, c.name;

-- Monthly series including empty months
SELECT m.month, COALESCE(x.cnt, 0) AS orders
FROM generate_series(DATE '2026-01-01', DATE '2026-12-01', INTERVAL '1 month') AS m(month)
LEFT JOIN (SELECT date_trunc('month', created_at) AS month, count(*) AS cnt
           FROM orders
           WHERE created_at >= DATE '2026-01-01' AND created_at < DATE '2027-01-01'
           GROUP BY 1) x ON x.month = m.month
ORDER BY m.month;

-- Month over month
SELECT month, revenue, revenue - lag(revenue) OVER (ORDER BY month) AS diff
FROM (SELECT date_trunc('month', created_at) AS month, sum(total) AS revenue
      FROM orders GROUP BY 1) s ORDER BY month;

-- Share of total
SELECT status, count(*), round(100.0 * count(*) / sum(count(*)) OVER (), 1) AS pct
FROM orders GROUP BY status ORDER BY 2 DESC;

-- Duplicates
SELECT name, count(*) FROM customers GROUP BY name HAVING count(*) > 1;
```
Traps: `NOT IN` with NULL returns nothing (use NOT EXISTS); `ORDER BY x DESC` puts NULLs first
(`NULLS LAST`); integer division `5/2 = 2` (use 100.0); `round(float, 2)` needs `::numeric`;
`2026-01-01` without quotes = 2024; `<= DATE '...-31'` loses the last day.
Control queries: `count(*) = count(DISTINCT key)`; min/max of the filtered value; spot-check 1 row;
old vs new result when rewriting.

---
## 13. Customer messages (templates)

### 13.1 First message
Hi, I have started. Before I begin: am I allowed to use documentation, my own notes, search
engines and AI assistants? I will read all tasks first and send my questions in one message.

### 13.2 Approval to stop a session
I found the cause: PID <pid> (<application>, user <user>) has been idle in transaction for <age>
and holds locks. It blocks <N> sessions: <pids / what>.
Proposal: pg_terminate_backend(<pid>). It closes only this session; its uncommitted changes
(<what>) are rolled back. Committed data is not affected. pg_cancel_backend would not help because
the session is not running a query.
May I proceed?

### 13.3 Restart request
To <goal> I need to change <parameter>. Its context is "postmaster", so a reload is not enough
(after reload pending_restart = t).
- Impact: PostgreSQL will be unavailable for about <N> seconds. All connections are dropped;
  running requests fail and the application reconnects.
- Preparation: config backed up (<file>.bak.<date>), configuration validated (pg_file_settings has
  no errors).
- Rollback: if it does not start, I restore the backup and start again (1-2 minutes).
- Alternative without restart: <option>, but <why worse>.
May I restart now, or do you prefer a time window?

### 13.4 Before/after a restart
Restarting PostgreSQL now.
PostgreSQL is back at <time> UTC, downtime <N> s. Verified: <service active, ss, select 1>.

### 13.5 Asking about something unexpected
I found <what> (<evidence>). It is not part of the task. Do you need it? Options: <A> / <B>.
I will not change it until you confirm.

### 13.6 Plan before applying
Findings: <top problem with numbers>.
Plan: 1) <step, online/offline, risk> 2) <step> 3) <verification>.
Expected effect: <metric before -> expected after>. OK to proceed?

### 13.7 Status (every 30 minutes)
Status T+<min>: done <1,2,3>; working on <4>; next <5,6>. Report expected at <time>.

### 13.8 I made a mistake
I need to tell you about a mistake: at <time> I ran <command>. Effect: <impact, evidence>.
Current state: <restored / verified>. The correct action would have been <x>.
Nothing else was affected: <evidence>.

### 13.9 Answer to "does it need a restart?"
No/Yes. <parameter> has context <x>: <reload is enough / restart needed because ...>.
It applies to <new sessions / immediately / after restart>.

---

## 14. Report templates

Style (from examples): letter to a technically competent person. For each item: what was
done, why this way, how to use it, how it was verified. Explain decisions and trade-offs.
Do not paste the whole log: key commands + key output. Full log in an Appendix.

### 14.1 Full report skeleton
```text
Subject: <server / task> - work summary

Hi,

Summary
<2-4 sentences: what was requested, what is done, what is open, anything that needs your attention>

1. <Requirement, word for word>
What I did: <short, key commands>
Why: <decision, alternatives, trade-off>
Verification:
<key output>

2. ...

Restarts and downtime
- <time UTC>: PostgreSQL restart to apply <parameter>, approved by you, ~<N> s. (or: none)

Changes outside the requested scope
- <e.g. stopped and disabled legacy-proxy.service (approved), files kept>
- <e.g. security group rule, test objects created and removed>

Open items / not completed
- <item, reason, what is needed from you>

Recommendations (not applied)
- <recommendation + reason>

Credentials
- <role / password, or where it is stored>


Appendix: command log (key outputs)
```

### 14.2 Item examples
Storage:
```text
1. Format the additional 8 GB disk with XFS and mount it permanently at /pgdata.
What I did: formatted /dev/nvme1n1 as XFS, added it to /etc/fstab by UUID with nofail.
Why: NVMe device names on EC2 can change after reboot, the UUID does not. nofail keeps the server
bootable if the volume is missing (PostgreSQL then fails to start instead of writing to the root disk,
because PGDATA is a subdirectory of the mount).
Verification:
$ findmnt /pgdata
/pgdata /dev/nvme1n1 xfs rw,relatime,...
```
OS tuning (example #2 style):
```text
2. vm.swappiness and THP
Before: vm.swappiness = 60, THP [always]. After: vm.swappiness = 1, THP [never].
Swappiness 1 keeps the kernel from swapping database memory. THP causes latency spikes and memory
bloat for PostgreSQL. Both are active now (no reboot needed) and persistent:
/etc/sysctl.d/99-postgresql.conf and disable-thp.service (enabled).
Verification:
$ sysctl vm.swappiness
vm.swappiness = 1
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
```
Query optimization:
```text
7. Slow query
Cause: to_char(created_at, ...) in WHERE prevents index use: every run scanned the whole table
(Seq Scan, 3212 buffers, 117 ms).
Fix: rewrote the condition as a half-open range on created_at and created index
orders_status_created_idx (status, created_at) with CREATE INDEX CONCURRENTLY (no write lock).
Result: Index Scan, <N> buffers, <X> ms. Same result before and after: count = <N>, sum = <S> (outputs below).
Note: the application must use the rewritten query; the old form still cannot use the index.
```

### 14.3 Phrases
- I verified this by ... / The output below shows ...
- I chose X over Y because ... The trade-off is ...
- This requires a restart because the parameter has context "postmaster".
- This change applies to new sessions only.
- I did not do X because ... If you want, I can do it in a maintenance window.
- I could not complete X in the available time. What remains: ...
- The seed script failed on its last line (GRANT to bi_reader: role does not exist). Everything
  else was loaded. Do you want me to create bi_reader, and with which privileges?

---

## 15. Final checklist before sending the report

- [ ] Every REQ has a section with verification output
- [ ] Every claim was tested by a test that could fail (as the target role, over TCP, etc.)
- [ ] SQL: result checked against every REQ (sort direction, tie-breaker, LIMIT only if asked,
      no extra filters); control query done
- [ ] Rewritten query returns the same result as the original (numbers side by side)
- [ ] Restarts listed with time and reason; changes outside scope listed
- [ ] Every ERROR seen during the work is mentioned and explained
- [ ] Test objects/rows removed (and said so)
- [ ] Open items and recommendations sections present
- [ ] Credentials the customer needs are provided
- [ ] No placeholders, no Cyrillic: `grep -nP '[\p{Cyrillic}\[\]]' report.txt`
- [ ] Spell check names: customer name, database, roles
- [ ] Sent before the deadline

---

## 16. Monitoring

### 16.1 What to monitor (layers)
| Layer | Metric | Where | Typical alert |
|---|---|---|---|
| Host | CPU, load, RAM, swap use | top, vmstat, free | swap in use, load >> cores |
| Host | Disk space and inodes (data, WAL, logs, archive) | df -h, df -i | > 80% warning, > 90% critical |
| Host | IO latency / utilisation | iostat -x | await high, %util near 100 |
| PostgreSQL | Up and accepting connections | pg_isready | not 0 |
| PostgreSQL | Connections vs max_connections | pg_stat_activity | > 80% |
| PostgreSQL | Long / idle-in-transaction sessions | pg_stat_activity (xact_start) | xact age > N min |
| PostgreSQL | Lock waits | pg_stat_activity wait_event_type = 'Lock' | waiting > N sec |
| PostgreSQL | Replication state and lag | pg_stat_replication (primary), pg_stat_wal_receiver (standby) | not streaming, lag > N MB / sec |
| PostgreSQL | Replication slots retaining WAL | pg_replication_slots | inactive slot, retained > N GB |
| PostgreSQL | WAL archiving | pg_stat_archiver | failed_count growing |
| PostgreSQL | Transaction ID age (wraparound) | age(datfrozenxid) | > 500-1000 million |
| PostgreSQL | Dead tuples / last autovacuum | pg_stat_user_tables | dead % high, never vacuumed |
| PostgreSQL | Cache hit ratio | pg_stat_database | < 99% (OLTP) |
| PostgreSQL | Checkpoints too frequent | log "checkpoints are occurring too frequently" | any |
| PostgreSQL | Errors in log | FATAL / PANIC / ERROR rate | any PANIC |
| Queries | Top queries by total time | pg_stat_statements / pg_stat_monitor | regressions |
| Backups | Last successful backup, last restore test | backup tool | older than policy |

### 16.2 Monitoring user (least privilege)
```sql
CREATE ROLE monitor LOGIN PASSWORD 'xxx';
GRANT pg_monitor TO monitor;      -- read all pg_stat_* views and monitoring functions
```
pg_hba: allow it only from the monitoring server.

### 16.3 Tools
- **PMM (Percona Monitoring and Management)**: PMM Server (usually Docker) + pmm-client on each DB host.
  `pmm-admin add postgresql ...` with a user that has pg_monitor; uses pg_stat_monitor or
  pg_stat_statements for query analytics.
- **Prometheus + postgres_exporter + Grafana**: exporter on the DB host, dashboards and alerts.
- **Nagios / Icinga + check_postgres**: one check per metric, e.g. actions connection, backends,
  txn_idle, txn_time, txn_wraparound, hot_standby_delay, archive_ready, locks, last_autovacuum.
- **Zabbix**: agent templates for PostgreSQL.

### 16.4 Nagios-style check = script with exit code
| Exit code | State |
|---|---|
| 0 | OK |
| 1 | WARNING |
| 2 | CRITICAL |
| 3 | UNKNOWN |
pg_isready exit codes: 0 accepting, 1 rejecting, 2 no response, 3 no attempt (bad parameters).
Example: replication lag check on the primary.
```bash
#!/bin/bash
# check_repl_lag.sh <warn_bytes> <crit_bytes>
W=${1:-16777216}; C=${2:-104857600}
LAG=$(psql -XAtq -c "SELECT COALESCE(max(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)),-1) FROM pg_stat_replication") || { echo "UNKNOWN - query failed"; exit 3; }
LAG=${LAG%.*}
[ "$LAG" -lt 0 ] && { echo "CRITICAL - no replica connected"; exit 2; }
[ "$LAG" -ge "$C" ] && { echo "CRITICAL - lag ${LAG} bytes"; exit 2; }
[ "$LAG" -ge "$W" ] && { echo "WARNING - lag ${LAG} bytes"; exit 1; }
echo "OK - lag ${LAG} bytes"; exit 0
```
Report (Nagios example style): what is monitored, thresholds and why, where to see it,
who gets alerts, what starts at boot, credentials.


##  «PostgreSQL availiable»

```bash
#!/bin/bash
# check_pg_up.sh [host] [port] - is PostgreSQL accepting connections
H=${1:-127.0.0.1}; P=${2:-5432}
PGISREADY=$(command -v pg_isready || echo /usr/pgsql-16/bin/pg_isready)
OUT=$("$PGISREADY" -h "$H" -p "$P" -t 5 2>&1); RC=$?
case $RC in
  0) echo "OK - $OUT"; exit 0 ;;
  1|2) echo "CRITICAL - $OUT"; exit 2 ;;
  *) echo "UNKNOWN - $OUT"; exit 3 ;;
esac
```

## Free space check

```bash
#!/bin/bash
# check_pg_disk.sh [path] [warn%] [crit%] - free space on the PostgreSQL data filesystem
D=${1:-/var/lib/pgsql/16/data}; W=${2:-80}; C=${3:-90}
USE=$(df -P "$D" 2>/dev/null | awk 'NR==2 {gsub("%","",$5); print $5}')
[ -z "$USE" ] && { echo "UNKNOWN - cannot read usage of $D"; exit 3; }
if   [ "$USE" -ge "$C" ]; then echo "CRITICAL - $D ${USE}% used (crit ${C}%)"; exit 2
elif [ "$USE" -ge "$W" ]; then echo "WARNING - $D ${USE}% used (warn ${W}%)"; exit 1
else echo "OK - $D ${USE}% used"; exit 0; fi
```

## failing state, production



| Проверка | OK | Failing |
|---|---|---|
| PostgreSQL up | `check_pg_up.sh 127.0.0.1 5432` | `check_pg_up.sh 127.0.0.1 5499` —  → CRITICAL |
| Диск | `check_pg_disk.sh` (80/90) | `check_pg_disk.sh /var/lib/pgsql/16/data 10 15` —  18 % → CRITICAL; `... 10 50` → WARNING |
| Lag | `check_repl_lag.sh` | `check_repl_lag.sh 0 1000000000` → WARNING, `check_repl_lag.sh 0 0` → CRITICAL |



```bash
bash check_pg_disk.sh /var/lib/pgsql/16/data 10 15; echo "exit code: $?"
```


---

## 17. Statistics: two different things

### 17.1 Activity (cumulative) statistics: what the server is doing
| View | Shows |
|---|---|
| pg_stat_activity | sessions, state, waits, current query |
| pg_stat_database | commits, rollbacks, cache hits, temp files, deadlocks per database |
| pg_stat_user_tables | seq vs index scans, live/dead tuples, last (auto)vacuum/analyze |
| pg_stat_user_indexes | index usage (idx_scan = 0 -> unused) |
| pg_statio_user_tables | blocks read from disk vs cache per table |
| pg_stat_bgwriter | checkpoints and buffer writes (PG17: checkpoints moved to pg_stat_checkpointer) |
| pg_stat_io (PG16+) | IO by backend type and context |
| pg_stat_wal, pg_stat_archiver | WAL generation, archiving |
| pg_stat_replication, pg_stat_wal_receiver, pg_replication_slots | replication |
| pg_stat_statements | per-query totals (extension) |
Counters are cumulative since `stats_reset`; compare two snapshots or reset
(`SELECT pg_stat_reset();`, `pg_stat_statements_reset()`) before measuring a change.

### 17.2 Planner statistics: what the optimizer believes about the data
Collected by ANALYZE (and autoanalyze) from a sample; stored in pg_statistic, readable in pg_stats.
```sql
SELECT attname, null_frac, n_distinct, most_common_vals, most_common_freqs, correlation
FROM pg_stats WHERE tablename = 'payments';
```
- Symptom of bad statistics: estimated rows vs actual rows in EXPLAIN ANALYZE off by 10x+.
- Fix: `ANALYZE table;` After restore / pg_upgrade: `vacuumdb --all --analyze-in-stages`.
- Skewed column: `ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000; ANALYZE t;`
  (default_statistics_target = 100).
- Correlated columns (city + zip): extended statistics
  `CREATE STATISTICS s1 (dependencies, ndistinct, mcv) ON city, zip FROM t; ANALYZE t;`
- Autoanalyze threshold: 50 + 0.1 x rows changed (autovacuum_analyze_threshold / _scale_factor).

---

## 18. Patroni (concepts for the interview)

- Components: Patroni agent on every node; distributed configuration store (DCS: etcd, Consul,
  ZooKeeper); routing layer (HAProxy and/or PgBouncer, sometimes a virtual IP).
- Leader election: the primary holds a leader key in the DCS with a TTL (default ttl 30 s,
  loop_wait 10 s). If it is not renewed, replicas race for the key; the winner is promoted.
  A primary that loses the key demotes itself -> protection against split brain.
- HAProxy routes by Patroni REST API health checks (port 8008): /primary returns 200 only on the
  leader, /replica only on replicas.
- Switchover = planned role change; failover = unplanned. `patronictl list`, `switchover`,
  `failover`, `restart`, `reload`, `reinit`, `pause`/`resume`.
- PostgreSQL settings are managed by Patroni: change them with `patronictl edit-config`, not by
  editing postgresql.conf (Patroni overwrites it).
- Old primary rejoins with pg_rewind (needs wal_log_hints or data checksums) or reinit.
- Options: maximum_lag_on_failover (do not promote a replica that is too far behind),
  synchronous_mode (no data loss, latency cost).
- Percona Distribution for PostgreSQL ships Patroni, etcd-compatible setups, HAProxy, PgBouncer.

---

## 19. PgBouncer (install and configure a connection pooler)

```bash
sudo apt install -y pgbouncer        # Ubuntu (PGDG repo)
sudo dnf install -y pgbouncer        # RHEL (PGDG repo)
```
Users and their SCRAM secrets (copy exactly, including the SCRAM-SHA-256$ prefix):
```sql
SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'billing_app';
```
/etc/pgbouncer/userlist.txt (owner postgres or pgbouncer, mode 600):
sudo cp -p /etc/pgbouncer/userlist.txt /etc/pgbouncer/userlist.txt.bak.$(date +%F-%H%M)
sudo -u postgres psql -XAtc "SELECT '\"' || rolname || '\" \"' || rolpassword || '\"' FROM pg_authid WHERE rolname = 'billing_app'" \
  | sudo tee /etc/pgbouncer/userlist.txt

```
"billing_app" "SCRAM-SHA-256$4096:...."
```
/etc/pgbouncer/pgbouncer.ini (backup the original first):
```
[databases]
billing = host=127.0.0.1 port=5432 dbname=billing

[pgbouncer]
listen_addr = 127.0.0.1,172.31.70.8
listen_port = 6432
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 500
default_pool_size = 20
admin_users = postgres
```
```bash
sudo systemctl enable --now pgbouncer
sudo systemctl status pgbouncer
ss -ltnp | grep 6432
psql -h 127.0.0.1 -p 6432 -U billing_app -d billing -c 'select 1'      # through the pooler
psql -h 127.0.0.1 -p 6432 -U postgres pgbouncer -c 'SHOW POOLS;'       # admin console
psql -h 172.31.75.233 -p 6432 -U billing_app -d billing -c 'select current_user, current_database(), inet_server_port();'
psql -h 172.31.75.233 -p 6432 -U billing_app -d billing -c 'select pg_sleep(30);'
psql -h 127.0.0.1 -p 6432 -U pgb_admin pgbouncer -c 'SHOW POOLS;'
psql -h 127.0.0.1 -p 6432 -U pgb_admin pgbouncer -c 'SHOW CONFIG;' | grep -E 'pool_mode|listen_port|default_pool_size'

```
pool_mode: session (safe, least saving), transaction (most efficient; session state such as SET,
advisory locks, temp tables and some prepared statement usage needs care), statement (rare).
PostgreSQL pg_hba must allow connections from the PgBouncer host (here 127.0.0.1).
Report: listen address/port, pool mode and why, pool sizes vs max_connections, how the
application must connect (port 6432), what starts at boot, how it was tested.

---

## 20. User administration cheat sheet

### 20.1 PostgreSQL roles: create and change
```sql
CREATE ROLE alice LOGIN PASSWORD 'xxx' VALID UNTIL '2026-12-31' CONNECTION LIMIT 5;
\password alice                          -- change password safely (not logged, not in history)
ALTER ROLE alice VALID UNTIL 'infinity'; -- remove expiry
ALTER ROLE alice NOLOGIN;                -- lock (new sessions only)
ALTER ROLE alice LOGIN;                  -- unlock
ALTER ROLE alice CONNECTION LIMIT 10;    -- -1 = unlimited
ALTER ROLE alice CREATEDB;               -- attributes: CREATEDB, CREATEROLE, REPLICATION, BYPASSRLS, SUPERUSER
ALTER ROLE alice SET statement_timeout = '5min';               -- for all databases
ALTER ROLE alice IN DATABASE billing SET work_mem = '16MB';    -- for one database
ALTER ROLE alice RESET statement_timeout;
```
`CREATE ROLE ... PASSWORD '...'` can appear in the server log (log_statement = ddl/all) and in
.psql_history: prefer `\password`.

### 20.2 Group roles and membership
```sql
CREATE ROLE readers NOLOGIN;
GRANT readers TO alice;                                   -- alice inherits readers' privileges
GRANT readers TO alice WITH INHERIT TRUE, SET FALSE;      -- PG16+: control inherit / SET ROLE separately
REVOKE readers FROM alice;
SET ROLE readers; RESET ROLE;                             -- act as the group explicitly
```
Built-in roles: pg_read_all_data, pg_write_all_data (all tables, all databases),
pg_monitor, pg_read_all_stats, pg_read_all_settings, pg_signal_backend (cancel/terminate others),
pg_checkpoint (PG15+), pg_use_reserved_connections (PG16+), pg_database_owner.

### 20.3 Privileges (layers: database -> schema -> object)
```sql
GRANT CONNECT, TEMPORARY ON DATABASE billing TO alice;
GRANT USAGE ON SCHEMA public TO alice;                    -- see objects in the schema
GRANT CREATE ON SCHEMA public TO alice;                   -- create objects (not default in PG15+)
GRANT SELECT ON public.payments TO alice;
GRANT SELECT (id, status) ON public.payments TO alice;    -- column-level
GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO alice;   -- needed for INSERT with serial/identity
GRANT EXECUTE ON FUNCTION f(int) TO alice;
REVOKE EXECUTE ON FUNCTION f(int) FROM PUBLIC;            -- functions are executable by PUBLIC by default
REVOKE ALL ON public.payments FROM alice;
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA public GRANT SELECT ON TABLES TO alice;
```

### 20.4 Inspect
```sql
\du+                         -- roles and attributes
\dp public.*                 -- table/sequence privileges
\ddp                         -- default privileges
\dn+                         -- schema privileges
\l+                          -- database privileges
\drds                        -- per-role / per-database settings
SELECT rolname FROM pg_roles WHERE rolsuper;                                -- who is superuser
SELECT rolname, rolcanlogin, rolconnlimit, rolvaliduntil FROM pg_roles ORDER BY 1;
SELECT r.rolname AS role, m.rolname AS member, am.inherit_option, am.set_option
FROM pg_auth_members am JOIN pg_roles r ON r.oid = am.roleid JOIN pg_roles m ON m.oid = am.member
ORDER BY 1, 2;                                                              -- group membership
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants WHERE grantee = 'alice';
SELECT has_table_privilege('alice','public.payments','SELECT'),
       has_schema_privilege('alice','public','USAGE'),
       has_database_privilege('alice','billing','CONNECT');
SELECT rolname FROM pg_authid WHERE rolpassword LIKE 'md5%';                -- old md5 hashes: reset passwords
SELECT usename, count(*) FROM pg_stat_activity GROUP BY 1;                  -- who is connected
```

### 20.5 Remove a user safely
```sql
SELECT pid, datname, state FROM pg_stat_activity WHERE usename = 'alice';  -- connected?
ALTER ROLE alice NOLOGIN;
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE usename = 'alice';   -- only with approval
\c <each database>
REASSIGN OWNED BY alice TO app_owner;
DROP OWNED BY alice;
\c postgres
DROP ROLE alice;
```

### 20.6 Linux users (quick)
```bash
sudo useradd -m -s /bin/bash bob && sudo passwd bob
sudo usermod -aG wheel bob          # RHEL sudo group; Ubuntu: -aG sudo
id bob; groups bob
sudo usermod -L bob                 # lock;  -U unlock
sudo chage -l bob                   # password ageing
sudo userdel -r bob                 # delete with home (careful)
sudo visudo -f /etc/sudoers.d/bob   # fine-grained sudo, validated syntax
# SSH key: ~/.ssh (700), ~/.ssh/authorized_keys (600), owned by the user
```

# BloodHound CE Frozen — Neo4j Startup Diagnosis
**Date:** 2026-03-19
**Category:** Troubleshooting
**System:** Kali Linux
**Risk Level:** Medium

## Overview
BloodHound CE was frozen waiting for Neo4j to start. This walkthrough documents the
full diagnostic findings and the exact fix required.

---

## Diagnostic Findings

### 1. No systemd unit for Neo4j
```
systemctl status neo4j
# Unit neo4j.service could not be found.
```
Neo4j on this system is NOT managed by systemd. It is managed directly by the
`/usr/bin/bloodhound` script via the `neo4j` CLI binary.

### 2. Neo4j process IS running (stale from prior session)
```
PID 366841 — /usr/lib/jvm/java-11-openjdk-amd64/bin/java ... org.neo4j.server.CommunityEntryPoint
```
The Java process is alive, but Neo4j is NOT listening on its ports.

### 3. Ports 7474 and 7687 are NOT listening
```
ss -tlnp | grep -E '7474|7687'
# No output — both ports are dead
```
Neo4j is running as a process but has shut down its network listeners.

### 4. Neo4j log reveals the root cause — stuck shutdown
From `/etc/neo4j/logs/neo4j.log` (14:48 UTC):
```
2026-03-19 14:48:33 INFO  Neo4j Server shutdown initiated by request
2026-03-19 14:48:33 INFO  Stopping...
```
From `/etc/neo4j/logs/debug.log` (14:48 UTC):
```
2026-03-19 14:48:43 INFO  [neo4j/34b0d72d] Waiting for closing transactions.
```
Neo4j received a shutdown request, began stopping, and then stalled waiting for
open transactions to close. The process is lingering in a zombie-like state —
alive but non-functional — with no network listeners active.

### 5. BloodHound startup script loop
`/usr/bin/bloodhound` contains:
```bash
if ! neo4j status; then
    neo4j start
fi
until curl "http://localhost:7474/" &>/dev/null; do printf ...; sleep .5; done
```
Because the Java process is still running, `neo4j status` returns "running" and
`neo4j start` is never called. The script then enters an infinite `until curl`
loop waiting for port 7474, which will never come up because Neo4j is stuck in
shutdown limbo. This is the freeze BloodHound exhibits.

### 6. Memory — Not the problem
```
Mem:   8.3Gi total, 2.8Gi free, 5.6Gi available
```
Sufficient memory. Not a memory pressure issue.

### 7. Config — Unconfigured heap (warnings only, not the cause)
Neo4j logs three WARN entries about unconfigured heap and pagecache sizes.
These are non-fatal but should be addressed for stability.

---

## Root Cause Summary

A previous Neo4j session initiated a graceful shutdown but got stuck waiting for
transactions to drain. The Java process remained alive, preventing `neo4j start`
from being called, while no ports were listening — causing BloodHound's curl loop
to spin forever.

---

## Fix

### Step 1 — Kill the stale Neo4j process (requires root)
```bash
# Identify the PID
pgrep -a java | grep neo4j

# Send SIGKILL to force-terminate the stuck process
kill -9 <PID>

# Confirm it is gone
pgrep -a java | grep neo4j   # should return nothing
```

### Step 2 — Remove any stale lock files (if present)
```bash
ls /etc/neo4j/data/databases/neo4j/database_lock
ls /etc/neo4j/data/databases/system/database_lock
# The database_lock files are 0 bytes and created on each start — they are normal.
# Only remove them if Neo4j refuses to start citing a lock conflict.
```

### Step 3 — Start Neo4j cleanly
```bash
neo4j start   # as root or via the bloodhound script
```

### Step 4 — Wait for port 7474 to come up, then start BloodHound
```bash
until curl http://localhost:7474/ &>/dev/null; do printf .; sleep 1; done
echo "Neo4j is up"
bloodhound
```

---

## Optional: Configure Heap to Eliminate Warnings
Add to `/etc/neo4j/neo4j.conf` (back it up first):
```bash
cp /etc/neo4j/neo4j.conf /etc/neo4j/neo4j.conf.bak.$(date +%Y%m%d_%H%M%S)
```
Then append:
```
dbms.memory.heap.initial_size=512m
dbms.memory.heap.max_size=1G
dbms.memory.pagecache.size=512m
```
Run `neo4j-admin memrec` for machine-specific recommendations.

---

## Rollback / Recovery
- Kill only the neo4j Java PID — no config changes required for the fix
- If Neo4j fails to start after the kill, restore data from backup or check
  `/etc/neo4j/logs/debug.log` for new errors
- BloodHound data is in `/etc/neo4j/data/` — do not delete this directory

---

## Notes & Gotchas
- There is no `neo4j.service` systemd unit on this system. Neo4j is invoked
  directly by the `/usr/bin/bloodhound` wrapper script.
- BloodHound CE on this system uses **both** Neo4j (graph DB) and PostgreSQL
  (relational DB for BHAPI). Both must be healthy.
- The `store_lock` entries in `debug.log` are informational — they show the lock
  was acquired at startup, not that a lock is blocking anything.
- Neo4j 4.4.26 is the installed version (Community Edition).
- The `bloodhound` script requires root (`id -u` check at line 14).

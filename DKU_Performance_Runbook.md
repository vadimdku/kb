# Dataiku DSS on AKS — Performance Troubleshooting Runbook

**Audience:** SREs managing Dataiku DSS containerized deployments on Azure Kubernetes Services  
**Scope:** Diagnosing and resolving UI slowness and performance degradation  
**DSS Version:** 14.x  
**References:** [Dataiku Operations Docs](https://doc.dataiku.com/dss/latest/operations/index.html) | [Troubleshooting](https://doc.dataiku.com/dss/latest/troubleshooting/diagnosing.html)

---

## How to Use This Runbook

When a Dataiku instance is reported as slow, start with **Step 1: First Triage** to narrow down the category, then jump to the relevant pillar section for detailed diagnosis and remediation.

---

## Step 1: First Triage — Narrow Down the Problem

Before diving into a specific pillar, run these quick checks to determine where the bottleneck is.

### 1.1 — Check the Pod Health

```bash
# Are any DSS pods in CrashLoopBackOff or restarting?
kubectl get pods -n <namespace> | grep -v "Running\|Completed"

# High restart count is a red flag even for "Running" pods
kubectl get pods -n <namespace> -o wide

# Check resource consumption
kubectl top pods -n <namespace>
kubectl top nodes
```

**What to look for:**
- Any pod in `CrashLoopBackOff` → go to **Pillar 5: Pod Instability**
- A DSS pod with CPU throttled near its limit → go to **Pillar 3: Resource Constraints**
- Memory near the container limit → go to **Pillar 2: JVM / Memory**

### 1.2 — Check DSS Backend Health (Inside the Pod)

```bash
# Exec into the DSS pod
kubectl exec -it <dss-pod-name> -n <namespace> -- bash

# Check if backend is running and how long it's been up
$DSS_HOME/bin/dss status

# Look for recent crashes, restarts, or OOM kills
grep -E "exited|SIGKILL|reaped|startup" $DSS_HOME/run/backend.log | tail -30

# Check for memory-related errors
grep -E "OutOfMemoryError|GC overhead|heap space" $DSS_HOME/run/backend.log | tail -20
```

### 1.3 — Check Disk / PVC Pressure

```bash
# Inside the DSS pod
df -h $DSS_HOME

# Find large directories
du -sh $DSS_HOME/*/ | sort -rh | head -15
```

If PVC is above 80% capacity → go to **Pillar 4: Disk / PVC Pressure**.

### 1.4 — Check Active Workload

Log in to the DSS UI as an admin and navigate to:
- **Administration → Monitoring → Background tasks** to see all running jobs, scenarios, notebooks

If you see a large number of active jobs/scenarios → go to **Pillar 1: Workload Overload**.

---

## Pillar 1: Workload Overload (Too Many Jobs, Scenarios, or Notebooks)

### Background

Every running job in DSS spawns a separate JEK (Job Execution Kernel) — a Java process. Every open Jupyter notebook keeps a kernel alive indefinitely, even when the browser tab is closed. When too many of these accumulate, they compete for CPU, memory, and I/O, causing the backend and UI to become sluggish.

### Symptoms
- UI is slow or unresponsive but no OOM errors in logs
- `kubectl top pods` shows high CPU and memory on the DSS pod
- Many running/queued jobs visible in Administration → Monitoring

### Diagnosis

**Check running jobs and scenarios from the DSS UI:**

Administration → Monitoring → Background tasks

**Check from the command line (inside pod):**

```bash
# Count running JEK processes
ps aux | grep DSSJobKernelMain | grep -v grep | wc -l

# Count running FEK processes
ps aux | grep DSSFutureKernelName | grep -v grep | wc -l

# Count open Jupyter kernels
ps aux | grep ipykernel | grep -v grep | wc -l
```

**Check JEK pre-start configuration:**

In DSS UI: Administration → Resources control → Job Execution Kernels

The "pre-started JEKs" setting keeps warm JEK processes alive even when idle. Each one consumes memory. If this value is too high relative to available memory, it will contribute to slowness.

### Remediation

**Abort stuck or runaway scenarios/jobs via Python API:**

```python
import dataiku

timeout_ms = 5 * 3600 * 1000  # 5 hours — adjust as needed

client = dataiku.api_client()
running = client.list_running_scenarios(all_users=True)
active = [s for s in running if s['alive']]

for s in active:
    if s['runningTime'] >= timeout_ms:
        p_key = s['payload']['targets'][0]["projectKey"]
        s_key = s['payload']['targets'][0]['objectId']
        project = client.get_project(p_key)
        scenario = project.get_scenario(s_key)
        scenario.abort()
        print(f"Aborted {s_key} in {p_key} (ran {s['runningTime']/3600000:.1f}h)")
```

**Kill stale Jupyter kernels:**

In DSS UI: Administration → Monitoring → Background tasks → Notebooks — manually unload idle sessions.

To automate: in an admin project, create a scheduled scenario that runs the **"Kill Jupyter Sessions"** macro on a daily basis to clean up kernels idle for more than N days.

**Reduce pre-started JEKs:**

Administration → Resources control → Job Execution Kernels → reduce "Pre-started JEKs" to a value appropriate for your memory budget.

**Tune max concurrent jobs:**

Administration → Resources control → set "Max simultaneous jobs" to a value that reflects the available CPU and memory. There is no universal recommendation — size it based on observed memory usage per job and available pod memory.

---

## Pillar 2: JVM Memory Exhaustion

### Background

DSS consists of several Java processes, each with a fixed heap configured via `install.ini`. The most important is the **backend** (handles all UI, API, and scheduling) and the **JEKs** (one per running job). If the heap is undersized, the JVM will spend excessive time in garbage collection, causing the backend to become unresponsive — this surfaces as the "Disconnected" overlay in the UI.

### Symptoms
- "Disconnected" overlay appearing for all users
- Backend restarting repeatedly (check `supervisord.log`)
- GC pause log entries like:
  ```
  [GC pause (G1 Evacuation Pause) (young) 12258M->12258M(12288M), 0.027 secs]
  ```
- `OutOfMemoryError` in `backend.log` or job logs

### Diagnosis

```bash
# Inside the DSS pod
# Check current heap settings
grep -A5 "\[javaopts\]" $DSS_HOME/install.ini

# Check config directory size (drives backend.xmx requirement)
find $DSS_HOME/config -ls | grep -v ".git" | awk '{s+=$7} END { print s/1024/1024 " MB" }'

# Confirm OOM is the crash cause
grep -E "OutOfMemoryError|GC overhead" $DSS_HOME/run/backend.log | tail -20

# Check backend restart frequency
grep -E "DSS startup|exited|SIGKILL" $DSS_HOME/run/backend.log | tail -30
```

**Sizing rules of thumb (from Dataiku docs):**

| Parameter | Default | Recommendation |
|---|---|---|
| `backend.xmx` | — | 12–20 GB for large production instances; at minimum ~2× config dir size |
| `jek.xmx` | 2g | Increase to 3–4g if jobs crash with OOM; remember: multiplied by number of JEKs |
| `fek.xmx` | 2g | Rarely needs adjustment; only change at direction of Dataiku Support |

> **Important:** Do not set `backend.xmx` between 32g and 48g — this range disables JVM compressed references and actually increases memory consumption. Go above 48g or stay below 32g.

### Remediation

```bash
# Inside the DSS pod — edit install.ini
vi $DSS_HOME/install.ini
```

Add or update the `[javaopts]` section:

```ini
[javaopts]
backend.xmx = 16g
jek.xmx = 4g
```

Apply the change:

```bash
$DSS_HOME/bin/dssadmin regenerate-config
$DSS_HOME/bin/dss restart
```

For containerized deployments, the preferred approach is to bake the correct `install.ini` into the container image or inject it via a ConfigMap, so the setting persists across pod restarts.

---

## Pillar 3: Kubernetes Resource Constraints (CPU Throttling / Memory Limits)

### Background

When a Dataiku pod is CPU-throttled or memory-limited at the Kubernetes level, the DSS process runs slower even though there are no obvious DSS-level errors. This is a common and often overlooked cause of slowness in containerized deployments.

### Symptoms
- DSS UI is slow but `backend.log` shows no errors
- `kubectl top pods` shows the pod near its CPU/memory limit
- `kubectl describe pod` shows `OOMKilled` in termination state (previous restarts)
- No evidence of problematic JVM GC in logs

### Diagnosis

```bash
# Check current resource usage vs limits
kubectl top pods -n <namespace>
kubectl describe pod <dss-pod-name> -n <namespace> | grep -A10 "Limits\|Requests\|OOM\|Last State"

# Check if pod was OOM-killed recently
kubectl describe pod <dss-pod-name> -n <namespace> | grep -A5 "Last State"

# Check node-level pressure
kubectl describe node <node-name> | grep -A10 "Conditions\|Allocatable\|Allocated resources"
```

### Remediation

**Adjust pod resource requests and limits in the Deployment/StatefulSet manifest:**

```yaml
resources:
  requests:
    memory: "20Gi"
    cpu: "4"
  limits:
    memory: "24Gi"
    cpu: "8"
```

Guidelines:
- Memory limit should be at least 2–4 GB above `backend.xmx` + sum of JEK and FEK heaps, to leave room for OS and non-heap processes.
- CPU limit should reflect actual throughput requirements. Excessive throttling causes latency even when memory is fine.
- If DSS and job execution run in the same pod, account for peak JEK memory: `(jek.xmx × max_concurrent_jobs) + backend.xmx + OS headroom`.

**Check for noisy neighbor issues:**

If other pods on the same node are consuming excessive resources, cordon the node or use node affinity/taints to isolate DSS pods:

```bash
kubectl cordon <noisy-node>
# or use node affinity in the DSS pod spec to pin to dedicated nodes
```

---

## Pillar 4: Disk / PVC Pressure

### Background

The Dataiku data directory is mounted as a PVC in Kubernetes. When the PVC fills up, DSS will start logging errors and eventually become unable to write job logs, temporary files, or runtime database updates — leading to hangs and UI unresponsiveness. High disk IOPS utilization (even without filling up) can also throttle database operations and log writes, causing slowness.

### Symptoms
- `df -h` shows PVC near 100% capacity
- DSS errors about inability to write files
- Slow job startup or H2 database errors in `backend.log`
- Azure Monitor showing disk IOPS near the limit for the disk tier

### Diagnosis

```bash
# Inside the DSS pod
df -h $DSS_HOME

# Identify largest consumers
du -sh $DSS_HOME/*/ 2>/dev/null | sort -rh | head -20

# Specifically check key directories:
du -sh $DSS_HOME/databases/    # H2 runtime databases
du -sh $DSS_HOME/jobs/         # Job logs (never auto-purged)
du -sh $DSS_HOME/scenarios/    # Scenario run logs
du -sh $DSS_HOME/config/       # Project configs (affects backend.xmx requirement)
du -sh $DSS_HOME/saved_models/ # ML model versions
du -sh $DSS_HOME/analysis-data/ # ML training data and splits
du -sh $DSS_HOME/tmp/          # Temporary files
```

### Remediation by Directory

**Job and Scenario Logs** (`jobs/`, `scenarios/`):

These are never auto-purged. Use the DSS "Clear job logs" macro in an admin project:
- DSS UI: Macros → "Clear job logs" → check "All projects"
- Automate: Create a scheduled scenario in an admin project that runs this macro periodically.

Manual cleanup (while DSS is running is safe for completed jobs):
```bash
# Remove logs older than 30 days for non-active jobs
find $DSS_HOME/jobs -maxdepth 3 -type d -mtime +30 | xargs rm -rf
```

**Temporary Files** (`tmp/`):

Only safe to clear when DSS is stopped:
```bash
$DSS_HOME/bin/dss stop
rm -rf $DSS_HOME/tmp/*
$DSS_HOME/bin/dss start
```

**Runtime Databases** (`databases/`):

If `databases/` exceeds 1–2 GB, this is a strong indicator that migration to external PostgreSQL is needed. See **Pillar 6** for details.

**Saved Models and Analysis Data**:

Old model versions can be removed from the DSS UI (Saved Models → Versions). Old ML sessions can be removed from the Visual Analysis UI.

**PVC Resize (if available):**

If the StorageClass supports volume expansion:
```bash
kubectl edit pvc <dataiku-pvc-name> -n <namespace>
# Increase the storage request
```

**Azure Disk IOPS Throttling:**

Azure Premium SSD IOPS are capped based on disk size (e.g., a 256 GB P15 disk is capped at 1100 IOPS). If DSS is generating heavy I/O (especially from H2 databases), check Azure Monitor for `Disk Read/Write Operations/Sec` and compare against the disk's IOPS cap. Remediation options:
- Upgrade the disk tier (larger disk = higher IOPS cap)
- Switch to Azure Premium SSD v2 or Ultra Disk for higher throughput
- Migrate runtime databases to external PostgreSQL (removes H2 disk I/O from the PVC entirely)

---

## Pillar 5: Pod Instability (CrashLoopBackOff / Frequent Restarts)

### Background

In AKS, a DSS pod in CrashLoopBackOff means the container keeps crashing and Kubernetes keeps restarting it. The restart loop itself causes instability for users — even if the pod briefly comes back, it crashes again before users can complete their work. Every DSS restart also aborts all running jobs and scenarios.

### Symptoms
- `kubectl get pods` shows `CrashLoopBackOff` or high `RESTARTS` count
- Users get the "Disconnected" overlay repeatedly
- Running jobs abort unexpectedly

### Diagnosis

```bash
# Check restart count and pod state
kubectl get pods -n <namespace>

# Check last crash reason (OOMKilled, Exit code, etc.)
kubectl describe pod <dss-pod-name> -n <namespace> | grep -A10 "Last State\|Reason\|Exit Code"

# View logs from the previous (crashed) container
kubectl logs <dss-pod-name> -n <namespace> --previous | tail -100

# Inside the pod — check supervisord and backend logs for crash reason
kubectl exec -it <dss-pod-name> -n <namespace> -- bash
grep -E "exited|SIGKILL|reaped|Fatal|OutOfMemory" $DSS_HOME/run/backend.log | tail -50
grep -E "exited|SIGKILL|reaped" $DSS_HOME/run/supervisord.log | tail -30
```

### Common Crash Causes and Fixes

**OOMKilled (Exit Code 137):**

Kubernetes killed the pod because it exceeded its memory limit. Fix: increase the pod memory limit and/or tune `backend.xmx` — see **Pillar 2** and **Pillar 3**.

**Accumulated temporary projects from `keepInstance` flag:**

App-as-recipe recipes with `"keepInstance": true` left enabled create a new temporary project on every run. Tens of thousands of these projects inflate the config directory, increase backend memory usage, and can cause repeated OOM crashes.

```bash
# Check how many RUN_ projects exist
ls $DSS_HOME/config/projects/ | grep "^RUN_" | wc -l

# Find all recipes with keepInstance enabled
grep -r '"keepInstance": true' $DSS_HOME/config/projects/
```

If you find thousands of `RUN_*` projects:
```bash
$DSS_HOME/bin/dss stop
cd $DSS_HOME/config/projects/
rm -rf RUN_*
$DSS_HOME/bin/dss start
```

Then disable `keepInstance` on all identified recipes (set to `false` or remove the key). This flag is intended for debugging only and should never be left on in production.

**Startup failing due to corrupted H2 database:**

If DSS cannot start because an H2 database is corrupted:
```bash
$DSS_HOME/bin/dss stop

# Identify the corrupted database from backend.log error messages, then:
# Example: flow_state.mv.db is unrecoverable
mv $DSS_HOME/databases/flow_state.mv.db $DSS_HOME/databases/flow_state.mv.db.bak

$DSS_HOME/bin/dss start  # DSS will recreate the database on startup
```
> **Note:** Deleting `flow_state.mv.db` loses partition build state. A subsequent "Rebuild what's needed" run will rebuild all partitions from scratch.

---

## Pillar 6: Runtime Database Overload (H2 → PostgreSQL Migration)

### Background

By default, DSS uses an embedded H2 database for runtime data (job history, flow state, partition state, metrics, timelines, discussions). H2 is suitable for small to medium instances but does not scale well. When databases grow beyond 1–2 GB — which happens quickly with high partition counts or high job volume — H2 becomes a bottleneck, causing backend thread contention, slow UI, and eventually crashes.

This is especially important for workloads with large numbers of unique partitions (tens of thousands or more).

### Symptoms
- Slowness even when few or no jobs are running
- H2 thread contention in JVM stack traces:
  ```
  "MVStore background writer nio:/...flow_state.mv.db" TIMED_WAITING
  "H2 TCP Server thread" BLOCKED on org.h2.engine.Engine
  ```
- `databases/` directory exceeds 1–2 GB:
  ```bash
  du -sh $DSS_HOME/databases/
  ```

### Diagnosis

```bash
# Check database directory size
du -sh $DSS_HOME/databases/*

# Get JVM thread dump to confirm H2 contention (use backend PID from dss status)
kill -3 $(cat $DSS_HOME/run/backend.pid)
# Thread dump will appear in backend.log — look for H2/MVStore threads BLOCKED or TIMED_WAITING
```

### Remediation

**Immediate relief (emergency only):** If a specific database file has grown to an unmanageable size and DSS cannot recover, delete and let DSS recreate it:

```bash
$DSS_HOME/bin/dss stop
mv $DSS_HOME/databases/flow_state.mv.db $DSS_HOME/databases/flow_state.mv.db.bak
$DSS_HOME/bin/dss start
```

**Long-term fix: Migrate to external PostgreSQL**

This is the correct fix for any production instance with heavy use. Follow the official guide: [Runtime Databases — External PostgreSQL](https://doc.dataiku.com/dss/latest/operations/runtime-databases.html)

Key requirements:
- PostgreSQL ≥ 9.5
- `max_connections` set to at least 500
- Prefer hosting PostgreSQL on the same network as the DSS pod (or same AKS cluster); avoid cloud-managed databases that may have intermittent connectivity (e.g., Azure Database for PostgreSQL Flexible Server requires careful configuration)
- Avoid PGBouncer between DSS and PostgreSQL

Migration steps (in brief):
1. Stop DSS
2. Edit `$DSS_HOME/config/general-settings.json` — populate the `internalDatabase` block with PostgreSQL connection details
3. Run: `$DSS_HOME/bin/dssadmin copy-databases-to-external`
4. Start DSS

For high-partition workloads (millions of unique partitions), plan for several hundred GB of PostgreSQL storage.

---

## Pillar 7: Websocket / Network Issues Causing UI Slowness

### Background

DSS relies on WebSockets for real-time UI updates (job progress, flow updates, notifications). If WebSockets cannot be established through the AKS ingress, the UI will fall back to polling, which is much slower and creates the perception of a "slow" UI even if the backend is healthy.

### Symptoms
- "Could not establish WebSocket connection" message in browser console
- UI updates are delayed or require manual refresh
- No obvious issues in DSS backend logs

### Diagnosis

Check browser developer tools (F12 → Network → filter by "WS") for WebSocket connection failures.

Confirm the ingress supports WebSocket upgrades:
```bash
kubectl describe ingress <dataiku-ingress-name> -n <namespace>
```

### Remediation

Ensure your ingress (typically nginx-ingress) has WebSocket support enabled:

```yaml
# In the Ingress annotations:
nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
nginx.ingress.kubernetes.io/configuration-snippet: |
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
```

Refer to: [Dataiku WebSocket Troubleshooting](https://doc.dataiku.com/dss/latest/troubleshooting/problems/websockets.html)

---

## Quick Reference: First 10 Commands to Run

When an instance is reported slow, run these in order:

```bash
# 1. Pod health
kubectl get pods -n <namespace>

# 2. Resource usage
kubectl top pods -n <namespace>

# 3. Check for OOM kills
kubectl describe pod <pod> -n <namespace> | grep -A5 "Last State"

# 4. PVC fill level (inside pod)
df -h $DSS_HOME

# 5. Largest directories
du -sh $DSS_HOME/*/ | sort -rh | head -10

# 6. Backend crash history
grep -E "exited|SIGKILL|startup" $DSS_HOME/run/backend.log | tail -20

# 7. JVM heap settings
grep -A5 "\[javaopts\]" $DSS_HOME/install.ini

# 8. Database directory size
du -sh $DSS_HOME/databases/

# 9. Temp project accumulation
ls $DSS_HOME/config/projects/ | grep "^RUN_" | wc -l

# 10. keepInstance flag check
grep -r '"keepInstance": true' $DSS_HOME/config/projects/ | wc -l
```

---

## Decision Tree Summary

```
UI is slow
│
├─ Pod in CrashLoopBackOff?
│   └─ YES → Pillar 5 (Pod Instability)
│
├─ kubectl top shows pod near memory limit?
│   └─ YES → Pillar 3 (K8s Resource Constraints) + Pillar 2 (JVM Memory)
│
├─ backend.log shows OutOfMemoryError or GC pauses?
│   └─ YES → Pillar 2 (JVM Memory)
│
├─ PVC > 80% full?
│   └─ YES → Pillar 4 (Disk / PVC Pressure)
│
├─ databases/ > 2 GB?
│   └─ YES → Pillar 6 (H2 Database Overload → PostgreSQL)
│
├─ Many running jobs / stale notebooks?
│   └─ YES → Pillar 1 (Workload Overload)
│
└─ WebSocket errors in browser console?
    └─ YES → Pillar 7 (Network / Ingress)
```

---

## Escalation to Dataiku Support

Collect the following before opening a support case:

1. **DSS instance diagnosis** — Administration → Maintenance → Diagnostic tool → Run and download
2. **JVM thread dump** — `kill -3 <backend_pid>` (output in `backend.log`)
3. **Pod describe output** — `kubectl describe pod <pod> -n <namespace>`
4. **kubectl top output** — pods and nodes at time of slowness
5. **Output of Quick Reference commands** above
6. **Timeline** — when slowness started, what changed (deployments, load patterns, data volume)

---

## Preventive Monitoring Recommendations

To catch these issues before users report them:

- Set up alerts on PVC usage > 70%
- Alert on pod restarts > 2 in 1 hour
- Alert on `kubectl top` pod memory > 85% of limit
- Run a scheduled "canary" scenario (a small, fast job) every 15–30 minutes and alert if it fails or takes longer than expected ([Dataiku monitoring docs](https://doc.dataiku.com/dss/latest/operations/monitoring.html))
- Schedule a weekly DSS macro to purge job logs and scenario logs older than 30 days
- Schedule a daily "Kill Jupyter Sessions" macro to unload kernels idle for more than 3 days
- Monitor `databases/` directory growth — alert if it exceeds 1 GB (plan PostgreSQL migration)
- Periodically audit for `keepInstance: true` in recipes:
  ```bash
  grep -r '"keepInstance": true' $DSS_HOME/config/projects/
  ```

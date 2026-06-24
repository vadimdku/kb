# Dataiku DSS Slowness Triage Runbook — Containerized on Azure Kubernetes Service (AKS)

**Audience:** Senior Dataiku technical staff / platform administrators  **Orientation:** Symptom-driven incident response
**Applies to:** DSS 14 (design / automation nodes) on AKS  **Version:** 1.0  **Updated:** June 2026

---

> **How to use this runbook.** Work top-down: **(1) Scope** the incident → **(2) Run first-response checks** → **(3) Jump to the matching Symptom Card** → apply the fix → **(4) Escalate** with a clean diagnosis if unresolved. Most incidents resolve to one of three layers — the **DSS application/JVM** (backend, JEK/FEK, runtime DBs), the **pod & node** (CPU/memory/IO limits, OOMKills), or **AKS storage/network** (PVC IOPS, disk pressure). Identify the layer first.

**Conventions.** `<DATA_DIR>` = the DSS data directory (often the mounted PVC, e.g. `/opt/dataiku/data`). Commands prefixed for the DSS container assume `kubectl exec -it <dss-pod> -n <ns> -- bash`; cluster commands run from a workstation with `kubectl` / `az` context on the cluster. Replace `<ns>`, `<dss-pod>`, `<node>`, `<pod>`, `<pvc>` as appropriate.

---

## 1. Step 0 — Scope the incident (30 seconds)

Before touching anything, answer four questions. The answers point you at the right layer and Symptom Card.

| Question | Why it matters / where it points |
|---|---|
| **What is slow?** | UI load across all projects vs. one job vs. one scenario vs. errors vs. crashes. UI-wide slowness ⇒ backend/JVM or node; a single job ⇒ compute or remote DB. |
| **Who is affected?** | All users ⇒ shared resource (backend, node, PVC). One user/project ⇒ a specific session, notebook, browser, or project artifact. |
| **When does it happen?** | Constant vs. intermittent vs. tied to a specific action vs. a time-of-day pattern (a recurring scenario/trigger storm). |
| **Where does execution run?** | Local DSS pod vs. remote AKS build pods vs. SQL pushdown on an external DB vs. Spark. Determines whether to look inside DSS or downstream. |

---

## 2. Step 1 — First-response triage (first 2 minutes)

Run these read-only checks immediately to localize the bottleneck. Capture output for the diagnosis bundle (Section 4).

### 2.1 Inside the DSS container

```bash
ps auxf                       # process tree: count JEK / FEK / python / jupyter
top    (or: top -H)           # CPU & memory hotspots; -H shows busiest threads
df -h                         # PVC / data-dir fullness  (look for ~100% mounts)
tail -f <DATA_DIR>/run/backend.log          # live backend activity during slowness

# Garbage-collection check (backend heap pressure):
grep -v JEK <DATA_DIR>/run/backend.log | grep -v FEK | grep 'Full GC'

du -sh <DATA_DIR>/databases   # internal H2 runtime DB size (>1-2 GB = red flag)
iostat -x 3 3                 # disk: w/s, wkB/s, %iowait, aqu-sz (if sysstat present)
```

If `iostat` is unavailable or does not expose useful PVC-backed disk data from inside the container, check Azure Monitor for the backing disk and node VM: read/write IOPS, throughput, latency, throttling, and burst-credit/on-demand bursting state.

**Read the GC line like this:**
`36431.480: [Full GC (Allocation Failure) 12268M->12266M(12288M), 39.1 secs]` — heap is ~maxed (12268→12266 of 12288 MB) and each collection costs tens of seconds. Multi-second Full GCs that recover almost nothing ⇒ backend memory pressure (**Card C**).

### 2.2 On the AKS cluster

```bash
kubectl get pods -n <ns> -o wide              # Running vs CrashLoopBackOff / OOMKilled / Pending
kubectl top nodes ; kubectl top pods -n <ns>  # live CPU/mem (needs metrics-server)
kubectl describe node <node> | grep -A5 Conditions   # MemoryPressure / DiskPressure
kubectl get pvc -n <ns> ; kubectl describe pvc <pvc>  # bound, capacity, events
kubectl get events -n <ns> --sort-by=.lastTimestamp | tail -30   # evictions, OOM, FailedScheduling
```

### 2.3 Capture a diagnosis (for escalation)

In the UI: **Administration ▸ Maintenance ▸ Diagnostic tool ▸ Run diagnostic tool.** Download the ZIP. If it exceeds the 15 MB support limit, trim it first (Section 4) and share via a file-transfer service.

---

## 3. Step 2 — Symptom triage cards

Find the card that matches the dominant signal from Sections 1–2 and work the cause→action table. Cards run from the most common UI-wide symptoms to specific job and network issues.

### Card A — UI slow or "Disconnected" for ALL users

**Signature.** UI takes 30–60 s+ across every project; periodic "Disconnected" overlay then auto-reconnect; a DSS restart helps for a while, then degrades again.

**Confirm.**
```bash
grep -v JEK <DATA_DIR>/run/backend.log | grep -v FEK | grep 'Full GC'   # multi-sec GCs?
grep -n 'OutOfMemoryError\|DSS startup: backend version' <DATA_DIR>/run/backend.log | tail
top    # backend (java) RSS near the pod memory limit?
```

| Likely cause | Action / fix |
|---|---|
| **Backend heap too small ⇒ GC thrash** | Raise `backend.xmx` in `install.ini [javaopts]`, then `./bin/dssadmin regenerate-config` and restart. Large production = 12–20 GB. DSS derives a recommended minimum from the `config/` directory size (a ~7 GB config implies ≥14 GB heap). |
| **Backend OOM crash loop** | Backend log shows `OutOfMemoryError: Java heap space` / `GC Overhead limit exceeded` just before a "DSS startup" line. Increase `backend.xmx`; ensure the pod memory limit exceeds xmx + ~1–2 GB JVM overhead. |
| **A user loaded a massive sample** | e.g. raising the Explore sample to a full multi-GB dataset. Identify in backend/FEK logs, reset the sample, reinforce the 10,000-row default with users. |
| **Config bloat inflating backend footprint** | Tens of thousands of projects/recipes enlarge heap and slow scheduling. Audit `config/projects` count; remove accumulated junk (see Cards C and H). |

### Card B — High CPU, low disk I/O, stable memory

**Signature.** Load average / uptime high; %iowait low; backend RSS steady; UI sluggish and jobs slow to start.

**Confirm.**
```bash
top -H                 # which threads burn CPU (backend vs python vs jupyter)
ps auxf | grep -c JEK  # number of running/pre-started job kernels
iostat -x 3            # confirm %iowait is LOW (rules out disk)
```

| Likely cause | Action / fix |
|---|---|
| **Too many concurrent scenarios / jobs / activities** | Lower the global concurrency limits in **Administration ▸ Settings** (max running jobs, max running activities, max activities per job) to match node capacity; stagger scenario triggers so they don't all fire at once. |
| **Custom-trigger storm** | Thousands of custom triggers, each firing often and spawning several threads, saturate CPU. Audit scenario triggers; widen intervals; consolidate or disable unused ones. |
| **Too many pre-started JEKs** | Each pre-started Job Execution Kernel consumes resources even when idle. Reduce the pre-start count in **Administration ▸ Resources control ▸ Job Execution Kernels**. |
| **One local subprocess pinning cores** | A single Python/notebook/webapp can consume all CPU when DSS-managed cgroups are off or unavailable. Identify the PID; if the AKS container is configured with delegated cgroup access, enable DSS cgroups for local subprocesses. Otherwise isolate heavy work with Kubernetes requests/limits and containerized execution. |

### Card C — High memory, long Full GC, or backend OOM

**Signature.** Pod RSS climbs toward the limit; Full GC pauses of seconds–tens of seconds; eventual OOMKilled (exit code 137) or repeated "Disconnected".

**Confirm.**
```bash
kubectl describe pod <dss-pod> -n <ns> | grep -A3 'Last State'   # OOMKilled? exit 137?
grep 'Full GC' <DATA_DIR>/run/backend.log | tail                 # multi-second collections?
```

| Likely cause | Action / fix |
|---|---|
| **Heap genuinely too small for workload** | Raise `backend.xmx`; size the pod memory *request/limit* above xmx plus JVM overhead so the kubelet doesn't OOMKill a healthy JVM. |
| **Accumulated state bloating heap** | Tens of thousands of temporary / application-as-recipe projects, or an oversized config. Stop DSS and remove the junk (e.g. instantiated `RUN_*` projects under `config/projects`); disable debug "keep instance" flags in app-as-recipe steps. |
| **Jupyter kernels never released** | Kernels stay alive for days/weeks after users navigate away. Abort via **Administration ▸ Monitoring ▸ Background tasks**; schedule the **"Kill Jupyter Sessions"** macro (e.g. idle > 15 days). |
| **Pod limit below JVM need** | If the container memory limit is under the JVM's real footprint, AKS kills it (OOMKilled) regardless of JVM health. Align container limits with xmx + overhead. |

### Card D — High disk IOPS / high %iowait

**Signature.** %iowait high in top/iostat; w/s or r/s saturated; disk queue depth (aqu-sz) high; both UI and jobs drag.

**Confirm.**
```bash
iostat -x 3            # watch w/s, wkB/s, %iowait, aqu-sz
iotop                  # which process is doing the heavy I/O (run as root)
du -sh <DATA_DIR>/*    # find the directories generating I/O / growth
```

On AKS, also check Azure Monitor for the backing disk/PVC and the node VM; `iostat` inside the DSS container may not show the full Azure Disk bottleneck.

| Likely cause | Action / fix |
|---|---|
| **Millions of small files / permission churn** | Custom triggers on UIF instances can leave stray files that DSS re-walks (re-setting permissions) on every run, generating huge write I/O. With DSS down, delete stale `custom-trigger-*` folders under `<DATA_DIR>/scenarios/…` (run as root; can be tens of millions of files — parallelize the delete). |
| **Oversized H2 runtime DBs** | Heavy MVStore read/write I/O from bloated internal databases — see **Card G**. |
| **Azure disk hitting its IOPS/throughput cap** | The effective cap is whichever limit is hit first: disk SKU/capacity, VM cached/uncached disk limit, cache path, or burst policy. Some Premium/Standard SSD sizes use credit-based bursting; larger Premium SSDs may use on-demand bursting if enabled. Move `<DATA_DIR>`/DBs to a higher tier (Premium SSD v2 / Ultra) or a larger disk for more baseline IOPS; confirm the VM size isn't the ceiling. |
| **Log / temp / cache growth thrashing disk** | Run the cleanup macros; clear `tmp/` and `caches/` while DSS is stopped (Card E). |

### Card E — PVC filling up / "No space left on device"

**Signature.** df shows the data-dir mount near 100%; writes fail; jobs and UI error; backend may crash. Disk growth on a PVC does not shrink itself.

**Confirm.**
```bash
df -h ; du -sh <DATA_DIR>/* | sort -h | tail -20   # biggest consumers
kubectl get pvc -n <ns>                            # capacity & expansion support
```

| Likely cause | Action / fix |
|---|---|
| **`jobs/` and `scenarios/` logs** | Not auto-garbage-collected. Run the **"Clear job logs"** macro (check "All projects"); it is safe to `rm` folders of jobs/scenario-runs that are no longer active. |
| **`saved_models/`, `analysis-data/`** | Old model versions and ML splits/sessions are kept forever. Delete obsolete versions/sessions from the UI (don't delete the *active* version's folder). |
| **`tmp/`, `caches/`** | Clear while DSS is stopped: `./bin/dss stop && rm -rf tmp/* caches/* && ./bin/dss start`. Never alter `tmp/` while DSS runs. |
| **`exports/`, pre-migration backups** | Remove stale export subfolders and old `_pre_migration_backup_YYYYMMDD` files at the data-dir root once migrations are done. |
| **`databases/` is huge** | Externalize runtime DBs to PostgreSQL — see **Card G**. |
| **No headroom left** | Expand the PVC (if the StorageClass allows volume expansion) for breathing room, then institute scheduled cleanup (Section 5). |

### Card F — Pods in CrashLoopBackOff / OOMKilled / Pending

**Signature.** `kubectl get pods` shows CrashLoopBackOff (restart backoff grows 10 s → 20 s → … → 5 min), OOMKilled, or Pending. Build/job pods or the DSS pod itself fail; cluster-wide compute is affected.

**Confirm.**
```bash
kubectl describe pod <pod> -n <ns>                     # Events, Last State, Reason
kubectl logs <pod> -n <ns> --previous                  # logs from the crashed container
kubectl logs <pod> -c <container> -n <ns> --previous   # specific container
kubectl top pod <pod> -n <ns>
```

| Likely cause | Action / fix |
|---|---|
| **OOMKilled (exit 137)** | Container memory limit too low. Raise memory requests/limits; for the DSS pod, align the limit with backend.xmx + overhead; for build pods, raise the containerized-execution memory request. |
| **Node MemoryPressure / DiskPressure** | Pods are evicted when a node is exhausted. `kubectl describe node <node>`; add nodes / scale the node pool / use a larger SKU; clear node disk; set proper requests so the scheduler spreads load. |
| **Bad image / missing config / failing liveness probe** | Read `--previous` logs and Events; fix the image tag, mounted secret/config, or an over-aggressive liveness/readiness probe. |
| **Pending = unschedulable** | Insufficient CPU/mem, an unbound PVC, or taints/affinity. Check describe-pod Events; scale nodes; verify the PVC binds in the right zone. |
| **Many DSS build pods crashlooping** | Check the containerized-execution base image, code-env build, and namespace resource quotas; see Dataiku "Elastic AI computation ▸ Troubleshooting". |

### Card G — Internal (H2) runtime database bloat or contention

**Signature.** Stacks show H2 "MVStore background writer …" and "H2 TCP Server … BLOCKED"; `databases/` is large; metrics/flow/timeline operations and the whole instance drag.

**Confirm.**
```bash
du -sh <DATA_DIR>/databases ; ls -lhS <DATA_DIR>/databases/*.mv.db | head
# Look for an oversized flow_state.mv.db / metrics / timelines DB
# In the diagnosis 'stacks' file, search for: H2 TCP Server ... BLOCKED
```

| Likely cause | Action / fix |
|---|---|
| **H2 past its scale (>1–2 GB; field cases 10–50×)** | Migrate runtime DBs to external PostgreSQL: edit `config/general-settings.json → internalDatabase`, run `./bin/dssadmin copy-databases-to-external`, restart. Set PostgreSQL `max_connections ≥ 500`. For AKS, choose the PG topology with Dataiku Support/Field Engineering: low latency, coordinated backups, and a clear operating rule that DSS is restarted whenever PG restarts are the important requirements. |
| **One DB unmanageably huge (e.g. flow_state)** | Emergency only, with Dataiku Support guidance: stop DSS, take a backup/snapshot, delete the specific unmanageable H2 file, and let DSS recreate it on start. Deleting `flow_state.mv.db` loses the record of which partitions were built, so the next "build what's needed" rebuilds everything. |
| **Atypical partitioning (10s of millions of partitions)** | Plan a large PostgreSQL (hundreds of GB) and revisit the partitioning design; this is the usual root of an enormous flow_state. |
| **Do NOT use the "Clean Internal Databases" macro** | That macro must not be run against internally-hosted H2 databases. |

### Card H — One slow job/scenario (instance otherwise healthy)

**Signature.** A single scenario/job runs far longer than usual (e.g. 8 h vs 17 min; or hangs ~23 h) and may lock DB tables and block other scenarios, while the rest of the instance is fine.

**Confirm.**
```text
# Open the run ▸ Workload diagnostics; compare a fast run vs the slow run
# In scenario-run-details.json compare per-step start/end; isolate the slow step
# Compare 'triggered' timestamp vs first 'State: RUNNING' (queue wait vs exec time)
```

| Likely cause | Action / fix |
|---|---|
| **Remote DB contention / lock / missing index** | For SQL pushdown (e.g. Oracle), a step like index creation can jump from minutes to hours under DB-side contention. Extract the SQL, run `EXPLAIN`, add indexes, and use bulk/fast-write instead of row-by-row INSERTs; coordinate with the DBA on locks around the slow window. |
| **Resource starvation (long queue before RUNNING)** | A wide gap between trigger time and `State: RUNNING` is a capacity problem, not a recipe problem. Raise concurrency limits or add compute. |
| **Unoptimized custom code pinning one core** | Enable resource profiling; if a Python/R process sits at 100% of a single core, refactor (vectorize, chunked streaming, multiprocessing) or push the recipe to AKS. |
| **Safety net: auto-abort overruns** | A Python scenario can list running scenarios and `scenario.abort()` those over a max runtime — but treat this as a guardrail, not a fix; address the DB-level root cause. |

### Card I — Network / browser latency (backend is fast)

**Signature.** UI feels slow but server CPU/memory/IO are fine; the backend answers in milliseconds while the browser waits seconds.

**Confirm.**
```text
# In frontend.log, API lines show both timings:
grep ' bkd=' <DATA_DIR>/run/frontend.log* | tail -50
GET /dip/api/get-configuration (1533ms) (bkd=26)
#   first number = browser-observed latency ; bkd= = backend time (ms)
```

| Likely cause | Action / fix |
|---|---|
| **Large gap between browser time and `bkd=`** | A big difference (e.g. 2209 ms seen vs bkd=1) means the time is spent on the network/client path, not in DSS. Investigate the ingress/load balancer, VPN/proxy, TLS, and WebSocket handling between client and cluster; engage the network team. |
| **Proxy / WebSocket misconfiguration** | Ensure any HTTP proxy in front of DSS is configured for WebSocket and isn't buffering; confirm the AKS ingress passes the long-lived connections used by Code Studios / webapps. |

---

## 4. Step 3 — Escalate with a clean diagnosis

If the cause isn't resolved, collect a complete-but-lean evidence bundle before opening/upgrading a support case.

- **Instance diagnosis:** Administration ▸ Maintenance ▸ Diagnostic tool ▸ Run diagnostic tool. It includes machine info, DSS logs, config, a data-dir file listing, and current activity — but no dataset/managed-folder data.
- **Shrink a large diag:** disable "datadir listing" and "include config dir" when the config holds millions of files. Support accepts ≤ 15 MB; use a file-transfer service for larger files.
- **Also attach:** `run/backend.log` (plus rotated `backend.log.X`), `iostat -x` output, and `kubectl describe` / `logs --previous` for failing pods.
- **Thread dump when threads are the question:** `kill -3 <backend_pid>` writes a full Java thread dump into `backend.log` (non-destructive). Useful for H2 / lock contention.
- **AKS evidence:** `kubectl get pod <dss-pod> -n <ns> -o yaml`, `kubectl describe pod <dss-pod> -n <ns>`, `kubectl logs <dss-pod> -n <ns> --previous`, `kubectl describe node <node>`, `kubectl get events -n <ns> --sort-by=.lastTimestamp`, and `kubectl get pvc,pv -n <ns> -o wide`. Attach Azure Monitor disk/node metrics for the incident window.

---

## 5. Preventive hardening checklist

Standing controls that prevent most of the incidents above. Treat as a configuration baseline for any DSS-on-AKS instance.

| Control | Why | Where / how |
|---|---|---|
| **Enable DSS cgroups where supported** | Stops one local notebook/recipe/webapp/custom scenario step from starving other local subprocesses. | **Administration ▸ Settings ▸ Resource control.** Requires delegated cgroup access from the AKS container; cgroup v2 requires UIF. Does not limit the DSS backend or containerized-execution pods. Memory limit by VM RAM: 32 GB→10g, 64 GB→38g, 96 GB→60g, 128 GB+→70% of RAM (256 GB→180g). |
| **Externalize runtime DBs to PostgreSQL** | H2 doesn't scale; PG is more resilient and backs up live. | Switch before `databases/` exceeds 1–2 GB. On AKS, validate topology with Dataiku; ensure low latency, coordinated backups, and a rule to restart DSS after any PG restart. Use `copy-databases-to-external`. |
| **Schedule "Kill Jupyter Sessions" macro** | Idle kernels accumulate memory for weeks. | Weekly scenario with a Run-macro step; target idle > N days. |
| **Scheduled cleanup macros** | Job/scenario logs and temp data are never auto-purged. | An admin-only project with scheduled "Clear job logs", scenario-log, and temp cleanup macros. |
| **Right-size `backend.xmx` + pod limits** | Prevents GC thrash and OOMKills. | `backend.xmx` ≥ config-dir size (12–20 GB large prod); pod memory limit > xmx + ~1–2 GB. |
| **Offload heavy work to AKS build pods** | Moves CPU/RAM-intensive recipes/ML off the design node. | Containerized execution with explicit memory/CPU requests and limits per workload. |
| **Pod requests/limits + node autoscaling** | Lets the scheduler spread load and avoids node pressure. | Set requests/limits on all workloads; watch node `MemoryPressure/DiskPressure`; enable the cluster autoscaler. |
| **Fast, separate storage for data dir / DBs** | Avoids IOPS/throughput throttling. | Premium SSD v2 / Ultra when appropriate; size disks for baseline IOPS/throughput and verify the node VM is not the lower cap. |
| **Cap global concurrency** | Prevents scenario/job storms from overloading the node. | **Administration ▸ Settings:** max running jobs / activities / activities-per-job tuned to node size; stagger triggers. |
| **Disable debug flags & repo hygiene in prod** | Stops silent config/disk bloat. | Turn off app-as-recipe "keep instance"; avoid committing large binaries or stray `.venv` into project libraries. |
| **Proactive monitoring & alerts** | Catch pressure before users feel it. | **Administration ▸ Monitoring** (background tasks, internal metrics); historize metrics and raise alerts; wire Azure Monitor + metrics-server. |

---

## 6. Quick reference

### 6.1 DSS processes

| Process | Role | Identify / logs |
|---|---|---|
| **backend** | Main process: config, users, UI, API, scenario scheduling. Crash ⇒ "Disconnected" for all. | `./bin/dss status`; `run/backend.log` |
| **JEK** | Job Execution Kernel — one per running job; pre-started for speed; consumes resources even idle. | cmdline `DSSJobKernelMain`; `jobs/PROJECT/JOB_ID/output.log` |
| **FEK** | Future Execution Kernel — backend delegates memory-heavy work; killed on overrun without taking down backend. | cmdline `DSSFutureKernelName`; in backend log |
| **jupyter server / kernels** | One server (supervisord) + one kernel per open notebook; kernels persist after navigating away. | `run/ipython.log` |
| **supervisord** | Starts/restarts/monitors nginx, backend, jupyter. | — |

### 6.2 Key paths, parameters & thresholds

| Item | Value / location |
|---|---|
| **Logs** | `<DATA_DIR>/run/backend.log`, `frontend.log`, `ipython.log`, `nginx.log`; jobs in `jobs/PROJECT/JOB_ID/output.log` |
| **Heavy data dirs** | `databases/` (H2), `config/`, `jobs/`, `scenarios/`, `saved_models/`, `analysis-data/`, `tmp/`, `caches/` |
| **JVM params** | `install.ini [javaopts]`: `backend.xmx`, `jek.xmx` (default 2g), `fek.xmx` (default 2g) → `./bin/dssadmin regenerate-config` → restart |
| **Externalize DBs** | when `databases/` > 1–2 GB → PostgreSQL, `max_connections ≥ 500`, `dssadmin copy-databases-to-external` |
| **GC red flag** | Full GC pauses of several seconds that reclaim almost nothing ⇒ raise `backend.xmx` |
| **DSS cgroup memory by VM RAM** | 32 GB→10g · 64 GB→38g · 96 GB→60g · 128 GB+→70% of RAM; applies only where DSS cgroups are supported/enabled |
| **Diagnosis** | Administration ▸ Maintenance ▸ Diagnostic tool; ≤ 15 MB for support (disable datadir listing / config dir to shrink) |
| **OOMKilled** | container exit code 137 → raise pod memory limit / `backend.xmx` |

---

## Appendix A — Field incident patterns

*Real, anonymized patterns seen in support cases, included as corroboration. They map common symptoms to the root cause ultimately found and the fix applied.*

| Observed symptom | Root cause found | Fix applied |
|---|---|---|
| UI slow despite ample CPU/RAM | Network/client latency — backend `bkd≈1 ms` but browser saw seconds | Investigated network/ingress path; not a DSS config issue (Card I) |
| High CPU + high write I/O after upgrade; stable memory | Millions of stray files from custom triggers (UIF) re-walked each run; plus oversized config | Deleted stale custom-trigger folders while down; capped concurrency; trimmed config (Cards B/D) |
| Instance-wide slowness; H2 stacks blocked | Runtime H2 DBs 10–50× over threshold; enormous `flow_state` | Migrated runtime DBs to PostgreSQL; deleted unmanageable flow_state to recreate (Card G) |
| High memory, long G1 pauses, backend SIGKILL with no jobs running | ~51,000 temporary app-as-recipe projects from a debug "keep instance" flag | Stopped DSS, `rm -rf RUN_*`; disabled keep-instance; raised xmx to ≥ config size (Card C) |
| Scenario ran ~8.5 h vs normal ~17 min | Oracle-side resource contention during a SQL index-creation step | Compared fast vs slow workload diagnostics; engaged DBA on DB-side contention (Card H) |
| Scenario hung ~23 h and locked tables, blocking others | DB/storage bottleneck + queue wait; row-by-row writes | EXPLAIN'd the SQL, added indexes, switched to bulk write; added a Python auto-abort guardrail (Card H) |

---

## Sources

Primary references used to build this runbook (Dataiku DSS 14 documentation, Dataiku Knowledge Base, and Microsoft Azure documentation).

- Operating DSS (index) — https://doc.dataiku.com/dss/latest/operations/index.html
- Diagnosing and debugging issues — https://doc.dataiku.com/dss/latest/troubleshooting/diagnosing.html
- Understanding and tracking DSS processes — https://doc.dataiku.com/dss/latest/operations/processes.html
- Tuning and controlling memory usage — https://doc.dataiku.com/dss/latest/operations/memory.html
- Using cgroups for resource control — https://doc.dataiku.com/dss/latest/operations/cgroups.html
- The runtime databases — https://doc.dataiku.com/dss/latest/operations/runtime-databases.html
- Managing DSS disk usage — https://doc.dataiku.com/dss/latest/operations/disk-usage.html
- Monitoring DSS — https://doc.dataiku.com/dss/latest/operations/monitoring.html
- Elastic AI computation (Kubernetes / AKS) — https://doc.dataiku.com/dss/latest/containers/index.html
- KB — Scoping performance issues — https://knowledge.dataiku.com/latest/admin-operating/performance-issues/tip-scoping-performance-issues.html
- KB — DSS UI slow for all users — https://knowledge.dataiku.com/latest/admin-operating/performance-issues/troubleshoot-dss-ui-slow-for-all.html
- KB — Diagnosing instance-wide performance — https://knowledge.dataiku.com/latest/admin-operating/performance-issues/troubleshoot-instance-wide-performance.html
- Azure — Troubleshoot OOMKilled in AKS — https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/availability-performance/troubleshoot-oomkilled-aks-clusters
- Azure — Pod stuck in CrashLoopBackOff — https://learn.microsoft.com/en-us/troubleshoot/azure/azure-kubernetes/create-upgrade-delete/pod-stuck-crashloopbackoff-mode
- Azure — VM & disk performance (IOPS/throughput) — https://learn.microsoft.com/en-us/azure/virtual-machines/disks-performance
- Azure — Storage concepts in AKS — https://learn.microsoft.com/en-us/azure/aks/concepts-storage

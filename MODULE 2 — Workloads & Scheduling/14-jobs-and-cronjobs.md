# 📌 Topic 2.6 — Jobs & CronJobs

> **Module 2 — Workloads & Scheduling**
> **File:** `14-jobs-and-cronjobs.md`
> **CKA Weight:** ~15% (Workloads) — Jobs appear in CKA batch processing scenarios

---

## 🎯 What You'll Learn
- What Jobs are and how they differ from Deployments
- Job completion modes — single, parallel, indexed
- Job failure handling, backoff limits, and retries
- CronJob — scheduled Jobs with cron expressions
- CronJob policies — concurrency, history, deadlines
- Real-world use cases for Jobs and CronJobs
- Full YAML anatomy for both objects
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Task Worker Analogy 📋
> Think of Kubernetes workload types as different types of workers:
>
> - **Deployment** = A **permanent employee** who shows up every day, works 9-5, never stops. If they quit, HR immediately hires a replacement. (long-running apps)
>
> - **Job** = A **contractor** hired for ONE specific task. Once the task is done, they leave. If they fail, you can hire them again (retry). No need to keep them around after completion. (batch tasks)
>
> - **CronJob** = A **recurring contractor** on a schedule. "Come in every Monday at 9am, do the task, leave." Managed by a schedule, automatically creates new contracts (Jobs) at the right time.

---

## 🔬 What is a Job?

A Job creates one or more pods and ensures that a **specified number of them successfully terminate** (exit code 0). Unlike Deployments that keep pods running forever, Jobs are designed for **tasks that have a finite end**.

```
DEPLOYMENT behavior:
  Pod exits → controller restarts it forever ♻️

JOB behavior:
  Pod exits with 0  → Job marks pod as completed ✅
                    → Job is Done (doesn't restart)
  Pod exits non-0   → Job retries (up to backoffLimit)
                    → If retries exhausted → Job Failed ❌

Key difference:
  Deployment → "keep running forever"
  Job        → "run to completion ONCE"
```

---

## 📄 Job YAML — Full Anatomy

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: default
  labels:
    app: db-migration

spec:
  # ── HOW MANY COMPLETIONS NEEDED ─────────────────────────
  completions: 1               # how many pods must succeed (default: 1)
                                # for parallel batch: set to total work items

  # ── HOW MANY RUN IN PARALLEL ────────────────────────────
  parallelism: 1               # how many pods run simultaneously (default: 1)
                                # for parallel: set > 1

  # ── RETRY LIMIT ─────────────────────────────────────────
  backoffLimit: 4              # how many times to retry on failure (default: 6)
                                # exponential backoff: 10s, 20s, 40s, 80s...

  # ── TIMEOUT ─────────────────────────────────────────────
  activeDeadlineSeconds: 300   # kill job if not done in 5 minutes

  # ── CLEANUP ─────────────────────────────────────────────
  ttlSecondsAfterFinished: 100 # delete job + pods 100s after completion
                                # without this, completed jobs stay forever

  # ── COMPLETION MODE ─────────────────────────────────────
  completionMode: NonIndexed   # NonIndexed | Indexed

  # ── POD TEMPLATE ────────────────────────────────────────
  template:
    metadata:
      labels:
        job-name: db-migration
    spec:
      # CRITICAL: restartPolicy for Jobs
      restartPolicy: OnFailure  # Never | OnFailure (NOT Always!)
      # OnFailure → restart container in same pod on failure
      # Never     → create NEW pod on failure (cleaner logs)

      containers:
        - name: migration
          image: my-app:v2.0
          command: ["python", "manage.py", "migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: "200m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

---

## 🔄 Job Completion Modes

### Mode 1: Single Job (Default)

```yaml
spec:
  completions: 1    # 1 pod must succeed
  parallelism: 1    # 1 pod at a time

# Flow:
# Create pod → runs → exits 0 → Job DONE ✅
# Create pod → runs → exits 1 → retry → exits 0 → Job DONE ✅
# Create pod → fails 4 times (backoffLimit=4) → Job FAILED ❌
```

### Mode 2: Fixed Completion Count (Batch Work)

```yaml
spec:
  completions: 5    # need 5 successful pods
  parallelism: 2    # run 2 at a time

# Flow:
# Round 1: Run pod-1 and pod-2 simultaneously
#          pod-1 succeeds (count: 1/5), pod-2 succeeds (count: 2/5)
# Round 2: Run pod-3 and pod-4 simultaneously
#          pod-3 succeeds (count: 3/5), pod-4 fails → retry
#          pod-4-retry succeeds (count: 4/5)
# Round 3: Run pod-5
#          pod-5 succeeds (count: 5/5)
# Job DONE ✅

# Use case: Process 5 chunks of data, 2 at a time
```

### Mode 3: Indexed Jobs (K8s 1.21+)

```yaml
spec:
  completions: 3
  parallelism: 3
  completionMode: Indexed   # each pod gets a unique index (0, 1, 2)

# Each pod gets env var: JOB_COMPLETION_INDEX = 0, 1, or 2
# Use this to process different data partitions:
# pod-0 processes items 0-99
# pod-1 processes items 100-199
# pod-2 processes items 200-299

# Pod can read its index:
containers:
  - name: worker
    command:
      - sh
      - -c
      - |
        echo "Processing partition $JOB_COMPLETION_INDEX"
        python process.py --partition=$JOB_COMPLETION_INDEX
```

---

## 🔁 Job Failure Handling

```
FAILURE FLOW:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

backoffLimit: 4  (default: 6)
restartPolicy: Never

  Attempt 1: Pod fails (exit code 1) → wait 10s
  Attempt 2: New pod fails           → wait 20s
  Attempt 3: New pod fails           → wait 40s
  Attempt 4: New pod fails           → wait 80s
  Attempt 5: New pod fails
  → backoffLimit exceeded → Job marked FAILED ❌

With restartPolicy: OnFailure (vs Never):
  OnFailure → same pod restarts (container restarts in place)
              cheaper, but logs from previous attempt lost
  Never     → new pod created each attempt
              cleaner logs, easier debugging, slightly more overhead

activeDeadlineSeconds: 300
  → Even if backoffLimit not hit,
    kill the job if it hasn't completed in 5 minutes
  → Takes priority over backoffLimit

kubectl describe job db-migration
# Look for:
# Conditions:
#   Complete  False
#   Failed    True  Reason: BackoffLimitExceeded
```

---

## 🕐 CronJob — Scheduled Jobs

A CronJob creates Jobs on a repeating schedule, like a Linux cron task.

```
CronJob creates Jobs
Jobs create Pods

CronJob ──► Job-1 ──► Pod-1 (at scheduled time 1)
       ──► Job-2 ──► Pod-2 (at scheduled time 2)
       ──► Job-3 ──► Pod-3 (at scheduled time 3)
```

---

## 📄 CronJob YAML — Full Anatomy

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: default

spec:
  # ── CRON SCHEDULE ───────────────────────────────────────
  schedule: "0 2 * * *"          # every day at 2:00 AM UTC
  # Format: minute hour day-of-month month day-of-week
  # ┌──── minute (0-59)
  # │ ┌── hour (0-23)
  # │ │ ┌─ day of month (1-31)
  # │ │ │ ┌ month (1-12)
  # │ │ │ │ ┌ day of week (0-7, 0=Sun, 7=Sun)
  # │ │ │ │ │
  # * * * * *

  # ── TIMEZONE ────────────────────────────────────────────
  timeZone: "Asia/Kolkata"       # K8s 1.27+ feature
  # Without this, schedule uses UTC

  # ── CONCURRENCY POLICY ──────────────────────────────────
  concurrencyPolicy: Forbid      # Allow | Forbid | Replace
  # Allow   → run new job even if previous still running
  # Forbid  → skip new run if previous still running
  # Replace → cancel previous, start new one

  # ── START DEADLINE ──────────────────────────────────────
  startingDeadlineSeconds: 200   # if job misses schedule by 200s, skip it
                                  # nil = no deadline (run even if very late)

  # ── HISTORY LIMITS ──────────────────────────────────────
  successfulJobsHistoryLimit: 3  # keep last 3 successful jobs (default: 3)
  failedJobsHistoryLimit: 1      # keep last 1 failed job (default: 1)
  # Set to 0 to auto-delete all completed jobs

  # ── SUSPEND ─────────────────────────────────────────────
  suspend: false                  # true = pause all future runs

  # ── JOB TEMPLATE ────────────────────────────────────────
  jobTemplate:
    spec:
      # activeDeadlineSeconds, backoffLimit apply to each created Job
      activeDeadlineSeconds: 3600    # 1 hour max per run
      backoffLimit: 2

      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: my-backup-tool:v1
              command:
                - /bin/sh
                - -c
                - |
                  echo "Starting backup at $(date)"
                  pg_dump $DATABASE_URL > /backup/db-$(date +%Y%m%d).sql
                  aws s3 cp /backup/ s3://my-backups/ --recursive
                  echo "Backup complete!"
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: url
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              emptyDir: {}
```

---

## 🕑 Cron Schedule Expressions

```
CRON FORMAT: minute hour day-of-month month day-of-week

COMMON EXAMPLES:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

"* * * * *"           → every minute
"*/5 * * * *"         → every 5 minutes
"0 * * * *"           → every hour (at :00)
"0 */6 * * *"         → every 6 hours
"0 0 * * *"           → every day at midnight
"0 2 * * *"           → every day at 2 AM
"0 9 * * 1"           → every Monday at 9 AM
"0 9 * * 1-5"         → every weekday at 9 AM
"0 9,17 * * *"        → every day at 9 AM and 5 PM
"0 0 1 * *"           → first day of every month
"0 0 1 1 *"           → every January 1st (yearly)
"30 23 * * 5"         → every Friday at 11:30 PM
"@hourly"             → same as 0 * * * *
"@daily"              → same as 0 0 * * *
"@weekly"             → same as 0 0 * * 0
"@monthly"            → same as 0 0 1 * *
"@yearly"             → same as 0 0 1 1 *

SPECIAL CHARACTERS:
  *    → every value
  */n  → every n units (*/5 = every 5)
  n-m  → range (1-5 = 1,2,3,4,5)
  n,m  → list (1,3,5 = 1 and 3 and 5)
```

---

## ⚙️ CronJob Policies — Deep Dive

### Concurrency Policy

```
Scenario: CronJob runs every minute, but the job takes 3 minutes

Allow (default):
  T=0:  Job-1 starts
  T=1:  Job-2 starts (Job-1 still running)
  T=2:  Job-3 starts (Job-1 and Job-2 still running)
  T=3:  Job-4 starts, Job-1 finishes
  → Multiple concurrent runs ✅ or ❌ depending on app

Forbid:
  T=0:  Job-1 starts
  T=1:  SKIP (Job-1 still running)
  T=2:  SKIP (Job-1 still running)
  T=3:  Job-1 finishes
  T=4:  Job-2 starts (next available slot)
  → Guarantees only one instance runs ✅ (good for DB backups)

Replace:
  T=0:  Job-1 starts
  T=1:  Job-1 KILLED, Job-2 starts (fresh start)
  T=2:  Job-2 KILLED, Job-3 starts
  → Always runs the latest version ✅ (good for data syncs)
```

### startingDeadlineSeconds

```
CronJob scheduled for 10:00 AM
startingDeadlineSeconds: 300 (5 minutes)

Scenario: Control plane was down, missed 10:00 AM

  10:05 AM → Control plane recovers
  → K8s checks: "Was 10:00 AM within the last 300 seconds?"
  → 10:05 - 10:00 = 5 minutes = 300 seconds ← EXACT boundary
  → If within deadline: Job runs (late start)
  → If past deadline: Job skipped for this cycle

  Without startingDeadlineSeconds:
  → K8s checks all missed runs in last 100 windows
  → If > 100 missed: CronJob stops scheduling entirely!
  → Setting a reasonable deadline prevents this issue
```

---

## 💻 Job & CronJob Commands

```bash
# ── JOB OPERATIONS ────────────────────────────────────────
# Create job from YAML
kubectl apply -f job.yaml

# Create job imperatively
kubectl create job db-migrate --image=my-app:v2 \
  -- python manage.py migrate

# Create job from CronJob template (manual trigger)
kubectl create job --from=cronjob/db-backup manual-backup-$(date +%Y%m%d)

# List jobs
kubectl get jobs
kubectl get jobs -w      # watch completions

# Output:
# NAME          COMPLETIONS   DURATION   AGE
# db-migration  1/1           45s        5m    ← done!
# data-process  2/5           2m         1m    ← in progress

# Describe (see failures, events)
kubectl describe job db-migration

# See pods created by job
kubectl get pods --selector=job-name=db-migration
kubectl get pods -l job-name=db-migration

# View logs of job pod
kubectl logs -l job-name=db-migration
kubectl logs job/db-migration     # shortcut

# Delete job (also deletes its pods)
kubectl delete job db-migration

# ── CRONJOB OPERATIONS ────────────────────────────────────
# Create CronJob
kubectl apply -f cronjob.yaml

# Create imperatively
kubectl create cronjob hourly-cleanup \
  --image=busybox \
  --schedule="0 * * * *" \
  -- /bin/sh -c "find /tmp -mtime +1 -delete"

# List CronJobs
kubectl get cronjobs
kubectl get cj           # shorthand

# Output:
# NAME        SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# db-backup   0 2 * * *   False     0        8h              5d

# Describe CronJob
kubectl describe cronjob db-backup

# See jobs created by CronJob
kubectl get jobs --selector=\
  'batch.kubernetes.io/cronjob-name=db-backup'

# Suspend CronJob (pause all future runs)
kubectl patch cronjob db-backup -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob db-backup -p '{"spec":{"suspend":false}}'

# Manually trigger a CronJob NOW
kubectl create job --from=cronjob/db-backup manual-$(date +%s)

# Delete CronJob (also deletes active jobs)
kubectl delete cronjob db-backup
```

---

## 🌍 Real-World Use Cases

```
JOBS:
  ✅ Database migrations (run once before deployment)
  ✅ Data processing (process a batch of files/records)
  ✅ Machine learning training (train model on dataset)
  ✅ Report generation (generate and email a report)
  ✅ One-time data transformation (ETL pipeline)
  ✅ Test runners (run test suite in CI/CD)
  ✅ Cache warming (pre-populate cache after deployment)

CRONJOBS:
  ✅ Database backups (every night at 2 AM)
  ✅ Log rotation / cleanup (delete old logs daily)
  ✅ Certificate renewal check (weekly)
  ✅ Scheduled reports (weekly business reports)
  ✅ Cache invalidation (hourly cache refresh)
  ✅ Data synchronization (sync from external API hourly)
  ✅ Health check aggregation (every 5 minutes)
  ✅ Billing/invoice generation (first of every month)
  ✅ Metrics aggregation (hourly rollup of raw data)
  ✅ Email digests (daily digest to users)
```

---

## 🔍 Job Status & Debugging

```bash
# Check job status
kubectl get job db-migration -o yaml | grep -A10 status

# status output:
# status:
#   completionTime: "2026-01-15T02:00:45Z"
#   conditions:
#   - lastTransitionTime: "2026-01-15T02:00:45Z"
#     status: "True"
#     type: Complete            ← SUCCESS
#   startTime: "2026-01-15T02:00:00Z"
#   succeeded: 1

# Failed job:
# status:
#   conditions:
#   - type: Failed
#     reason: BackoffLimitExceeded
#   failed: 4                  ← number of failed attempts

# Debug a failed job
kubectl describe job db-migration
# Check: Events section for detailed errors

kubectl get pods -l job-name=db-migration
# Pods may show: Error or OOMKilled

kubectl logs <failed-pod-name>
kubectl logs <failed-pod-name> --previous

# Common failure reasons:
# BackoffLimitExceeded  → reached max retries
# DeadlineExceeded      → activeDeadlineSeconds exceeded
# ImagePullBackOff      → wrong image in job spec
# OOMKilled             → job needs more memory
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| `restartPolicy: Always` in Job | Jobs CANNOT use `Always` — only `Never` or `OnFailure` |
| Not setting `ttlSecondsAfterFinished` | Completed jobs stay forever, polluting your cluster |
| ConcurrencyPolicy: Allow for DB backup | If previous backup still runs, 2nd starts too — can corrupt backup |
| Not setting `activeDeadlineSeconds` | A hung job runs forever, wasting resources |
| Forgetting `backoffLimit: 0` for strict once-only | Default is 6 retries — set to 0 if one attempt only |
| CronJob `schedule` uses UTC by default | Use `timeZone` field (K8s 1.27+) for local timezone |
| `startingDeadlineSeconds` too small | Job misses too many windows → CronJob auto-suspends! |

---

## 🎓 CKA Exam Tips

```
✅ Job restartPolicy CANNOT be Always:
   Use OnFailure or Never
   (Very common exam trap!)

✅ Quick Job creation:
   kubectl create job my-job --image=busybox \
     -- /bin/sh -c "echo hello"

✅ Quick CronJob creation:
   kubectl create cronjob my-cron \
     --image=busybox \
     --schedule="*/5 * * * *" \
     -- /bin/sh -c "echo hello"

✅ Manually trigger a CronJob:
   kubectl create job manual-run --from=cronjob/<cron-name>

✅ Watch job completion:
   kubectl get jobs -w

✅ Get logs from job:
   kubectl logs job/<job-name>
   kubectl logs -l job-name=<job-name>

✅ Check job pods:
   kubectl get pods -l job-name=<job-name>

✅ apiVersion for both: batch/v1
   (NOT apps/v1 — this is a common mistake!)

✅ CronJob shorthand: cj
   kubectl get cj
```

---

## ❓ Interview & Scenario Questions

### Q1: What is the difference between a Job and a Deployment?
**Answer:**
A Deployment manages long-running pods that should never stop — if a pod exits, the controller restarts it. A Job manages pods that run to completion — once a pod exits with code 0, it's counted as successful and not restarted. A Job tracks how many successful completions occurred and marks itself as complete when the target count is reached. Jobs are for finite tasks (database migrations, data processing, report generation), while Deployments are for always-on services (web servers, APIs).

---

### Q2: What `restartPolicy` values are valid for a Job?
**Answer:**
Jobs only support `restartPolicy: OnFailure` or `restartPolicy: Never`. They cannot use `Always` (which is the default for Deployments). With `OnFailure`, the same pod restarts its container in place when it fails. With `Never`, a new pod is created on each failure attempt. `Never` is generally preferred for Jobs because it gives cleaner log separation per attempt and avoids pod restart count inflation. The Job's `backoffLimit` controls total retry attempts regardless of which restartPolicy is used.

---

### Q3: Scenario — Your CronJob is supposed to run every hour but you notice multiple job instances running simultaneously. How do you fix it?
**Answer:**
Set `concurrencyPolicy: Forbid` in the CronJob spec:
```yaml
spec:
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid    # skip if previous run still active
```
This prevents a new Job from starting if the previous one hasn't finished yet. If the job regularly takes longer than 1 hour, consider also setting `activeDeadlineSeconds` to kill hung jobs and `startingDeadlineSeconds` to handle missed runs. After patching: `kubectl patch cronjob my-cron -p '{"spec":{"concurrencyPolicy":"Forbid"}}'`

---

### Q4: How do you run a batch job that processes 100 items using 10 workers in parallel?
**Answer:**
Use `completions` and `parallelism` with an Indexed Job:
```yaml
spec:
  completions: 10         # 10 successful pods needed
  parallelism: 5          # 5 run simultaneously
  completionMode: Indexed # each pod gets JOB_COMPLETION_INDEX 0-9
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: my-processor
          command:
            - python
            - process.py
            - --partition=$(JOB_COMPLETION_INDEX)
            - --items-per-partition=10
```
Each worker processes its own slice (0-9, 10-19, etc.) using `JOB_COMPLETION_INDEX`.

---

### Q5: What happens to Jobs and Pods when a CronJob is deleted?
**Answer:**
When a CronJob is deleted, active (currently running) Jobs are deleted along with their pods, causing the running jobs to be terminated. However, completed Jobs (both successful and failed) are handled based on `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` — by default 3 successful and 1 failed job are kept for history. These history jobs and their pods are retained even after CronJob deletion (to preserve logs/history). To clean up everything, delete the individual Jobs manually or set history limits to 0 before deleting the CronJob.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│             TOPIC 2.6 — JOBS & CRONJOBS                      │
├──────────────────────────────────────────────────────────────┤
│  JOB:                                                        │
│  apiVersion: batch/v1   kind: Job                            │
│  Purpose: Run pods to completion (finite tasks)              │
│  restartPolicy: OnFailure | Never  (NOT Always!)             │
│  backoffLimit: retries on failure (default: 6)               │
│  activeDeadlineSeconds: kill if takes too long               │
│  ttlSecondsAfterFinished: auto-cleanup after done            │
├──────────────────────────────────────────────────────────────┤
│  JOB MODES:                                                  │
│  completions:1, parallelism:1  → single run (default)        │
│  completions:N, parallelism:P  → batch (N total, P at once)  │
│  completionMode: Indexed       → each pod gets unique index  │
├──────────────────────────────────────────────────────────────┤
│  CRONJOB:                                                    │
│  apiVersion: batch/v1   kind: CronJob  (shorthand: cj)       │
│  schedule: "min hr dom month dow"  (UTC by default)          │
│  concurrencyPolicy: Allow | Forbid | Replace                 │
│  startingDeadlineSeconds: skip if missed by N seconds        │
│  suspend: true/false  → pause/resume scheduling              │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl create job <n> --image=<i> -- <cmd>                 │
│  kubectl create cronjob <n> --image=<i> \                    │
│    --schedule="* * * * *" -- <cmd>                           │
│  kubectl create job manual --from=cronjob/<cj-name>          │
│  kubectl get jobs / kubectl get cj                           │
│  kubectl logs job/<job-name>                                 │
├──────────────────────────────────────────────────────────────┤
│  CRON QUICK REFERENCE:                                       │
│  "*/5 * * * *"  → every 5 min                               │
│  "0 * * * *"    → every hour                                 │
│  "0 2 * * *"    → daily at 2AM                               │
│  "0 0 * * 1"    → every Monday midnight                      │
│  "0 0 1 * *"    → first of every month                       │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `15-resource-requests-limits.md`
> *CPU/Memory requests vs limits, QoS classes, OOMKilled — deep dive*

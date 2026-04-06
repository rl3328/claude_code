---
name: slurm-monitor
description: Generate a SLURM job monitor script that detects OOM failures and auto-resubmits with more memory (single retry or iterative increment). Supports both individual batch scripts and SLURM array jobs.
user-invocable: true
argument-hint: "[batch_dir_or_script] [job_prefix] [initial_mem] [mem_increment] [max_mem] [check_interval] [output_path]"
---

# SLURM Job Monitor Generator

Generate a `monitor.sh` script that monitors SLURM jobs and auto-resubmits OOM-killed jobs with more memory, then writes a final summary.

## Two job modes (auto-detected)

- **Batch mode**: `$0` is a directory containing `batch_*.sh` scripts → monitor each batch script individually
- **Array mode**: `$0` is a single `.sh` file with `#SBATCH --array=...` → monitor each array task individually

Detection: if `$0` ends in `.sh` and is a file → array mode. If `$0` is a directory → batch mode.

## Two memory retry strategies

1. **Iterative increment mode** (default): On OOM, increase memory by `mem_increment` each retry, up to `max_mem`. E.g. 20G → 30G → 40G → ... → max_mem.
2. **Single retry mode**: If `mem_increment` is set to `0` or not provided, fall back to a single retry jumping from `initial_mem` to `max_mem`.

## Arguments

Parse from `$ARGUMENTS` (space-separated positional args):
- `$0` = batch_dir OR submit_script: directory containing `batch_*.sh` scripts, or a single array job `.sh` file (REQUIRED)
- `$1` = job_prefix: the SBATCH `--job-name` prefix used in scripts (REQUIRED)
- `$2` = initial_mem (default: 50G) — memory in the original scripts
- `$3` = mem_increment (default: 50G) — how much to add each retry. Set to `0` for single-retry mode.
- `$4` = max_mem (default: 200G) — stop retrying beyond this
- `$5` = check_interval in seconds (default: 1800)
- `$6` = output_path for monitor.sh (default: parent of batch_dir + /logs/monitor.sh, or same dir as submit_script + /logs/monitor.sh)

If any required args are missing (batch_dir/submit_script, job_prefix), ask the user.

## Generated script structure

```bash
#!/bin/bash
#SBATCH --job-name=${job_prefix}_monitor
#SBATCH --output=${monitor_log_dir}/monitor.out
#SBATCH --error=${monitor_log_dir}/monitor.err
#SBATCH --mem=2G
#SBATCH --partition=cpu
#SBATCH --time=2-00:00:00
#SBATCH --cpus-per-task=1

MODE="<batch|array>"             # auto-detected
BATCH_DIR="<batch_dir>"          # batch mode only
SUBMIT_SCRIPT="<submit_script>"  # array mode only
JOB_PREFIX="<job_prefix>"
INITIAL_MEM="<initial_mem>"       # e.g. 20G
MEM_INCREMENT=<mem_increment_num> # e.g. 10 (just the number, in GB)
MAX_MEM=<max_mem_num>             # e.g. 60 (just the number, in GB)
CHECK_INTERVAL=<check_interval>

MONITOR_LOG="<log_dir>/monitor.log"
SUMMARY_FILE="<log_dir>/monitor_summary.txt"
```

---

## Batch mode — core loop logic

```
# Auto-detect LOG_DIR from first batch script's #SBATCH --output line
LOG_DIR=$(grep '#SBATCH --output=' "${BATCH_DIR}"/batch_1.sh | head -1 | sed 's/.*--output=//' | xargs dirname)

while true:
    1. Count running/pending jobs matching JOB_PREFIX via squeue

    2. For each batch_N.sh (excluding _retryXG.sh files):
       a. Extract batch number N from filename
       b. Find the LATEST attempt for this batch:
          - Check batch_N_retry{MEM}G.sh files in descending mem order
          - Or the original batch_N.sh if no retries exist
       c. Determine current_mem for that batch (from the latest script's --mem line)

       # Skip conditions (in order):
       - Latest attempt still in queue (squeue) -> skip
       - sacct State = COMPLETED -> count as success, skip
       - (Fallback) Latest attempt .out has "Batch N completed at" -> count as success, skip

       # Detect OOM on latest attempt:
       - Primary: Check sacct for OUT_OF_ME% state or FAILED with OOM reason
       - Fallback: Check stderr for: oom_kill, exitcode=-9, out of memory, MEMLIMIT

       # Actions on OOM:
       - Calculate next_mem = current_mem + MEM_INCREMENT
       - If next_mem > MAX_MEM -> log "MAX MEM REACHED", count as max_mem_fail
       - Else -> generate batch_N_retry{next_mem}G.sh (sed to set --mem=${next_mem}G
                 and jobname to ${JOB_PREFIX}_N_retry${next_mem}G), sbatch it

       # Non-OOM failure:
       - Has .out but no completion and not OOM -> count as other_fail

    3. Count completed vs total batches
    4. resolved = completed + max_mem_fail + other_fail
    5. All resolved (resolved >= total_batches) AND 0 running/pending -> write summary, break
    6. Nothing running + nothing resubmitted + resolved < total_batches -> warn (but still exit after 3 consecutive stuck cycles to avoid infinite loop)
    7. Sleep CHECK_INTERVAL
```

### Batch mode retry naming convention

- Original: `batch_N.sh`, jobname `${JOB_PREFIX}_N`
- 1st retry: `batch_N_retry100G.sh`, jobname `${JOB_PREFIX}_N_retry100G`
- 2nd retry: `batch_N_retry150G.sh`, jobname `${JOB_PREFIX}_N_retry150G`

---

## Array mode — core loop logic

### Setup

```bash
# Parse all expected task IDs from the --array line in SUBMIT_SCRIPT
# e.g. --array=1-55,57-169 → expand to full list: 1 2 3 ... 55 57 58 ... 169
ARRAY_SPEC=$(grep '#SBATCH --array=' "$SUBMIT_SCRIPT" | head -1 | sed 's/.*--array=//' | sed 's/%[0-9]*//')
# expand_array_spec() function: parse ranges like "1-55,57-169" into individual IDs
ALL_TASK_IDS=( $(expand_array_spec "$ARRAY_SPEC") )
TOTAL_TASKS=${#ALL_TASK_IDS[@]}

# Auto-detect LOG_DIR from SUBMIT_SCRIPT's #SBATCH --output line
# e.g. logs/mfsusie_%A_%a.out → LOG_DIR=logs
LOG_DIR=$(grep '#SBATCH --output=' "$SUBMIT_SCRIPT" | head -1 | sed 's/.*--output=//' | xargs dirname)

# Directory for retry scripts (same dir as SUBMIT_SCRIPT)
RETRY_DIR="$(dirname "$SUBMIT_SCRIPT")/retries"
mkdir -p "$RETRY_DIR"

# Track: which task IDs are being retried at which mem level
# Use a state file to persist across monitor restarts
STATE_FILE="<log_dir>/monitor_state.txt"
# Format: TASK_ID CURRENT_MEM JOB_ID STATUS
```

### expand_array_spec function

```bash
expand_array_spec() {
    local spec="$1"
    local result=()
    IFS=',' read -ra parts <<< "$spec"
    for part in "${parts[@]}"; do
        if [[ "$part" =~ ^([0-9]+)-([0-9]+)$ ]]; then
            for ((i=${BASH_REMATCH[1]}; i<=${BASH_REMATCH[2]}; i++)); do
                result+=($i)
            done
        elif [[ "$part" =~ ^([0-9]+)-([0-9]+):([0-9]+)$ ]]; then
            for ((i=${BASH_REMATCH[1]}; i<=${BASH_REMATCH[2]}; i+=${BASH_REMATCH[3]})); do
                result+=($i)
            done
        else
            result+=($part)
        fi
    done
    echo "${result[@]}"
}
```

### Main loop

```
while true:
    1. Count running/pending jobs matching JOB_PREFIX via squeue (exclude _monitor)

    2. For each TASK_ID in ALL_TASK_IDS:
       a. Determine latest attempt for this task:
          - Check state file for retry info, or
          - Check for retry scripts: ${RETRY_DIR}/${JOB_PREFIX}_task${TASK_ID}_retry*G.sh
          - Find the one with highest mem, get its SLURM job ID
          - Fall back to original array job task
       b. Get current_mem and job_id for latest attempt

       # Determine status of latest attempt:
       - Use sacct: sacct -j ${JOB_ID}_${TASK_ID} (for original array) or sacct -j ${RETRY_JOB_ID} (for retry)
         Parse State field: COMPLETED, FAILED, OUT_OF_MEMORY, RUNNING, PENDING, CANCELLED, TIMEOUT

       # Skip conditions:
       - RUNNING or PENDING -> skip
       - COMPLETED -> count as success, skip

       # Detect OOM:
       - sacct State = OUT_OF_ME% or FAILED with reason OOM
       - Also check .err file for: oom_kill, exitcode=-9, out of memory, MEMLIMIT
       - Also check .out file for OOM indicators

       # Actions on OOM:
       - Calculate next_mem = current_mem + MEM_INCREMENT
       - If next_mem > MAX_MEM -> log "MAX MEM REACHED for task TASK_ID", count as max_mem_fail
       - Else -> create retry script (see below), sbatch it, record in state file

       # Non-OOM failure (FAILED, TIMEOUT, CANCELLED without OOM):
       - count as other_fail

    3. resolved = completed + max_mem_fail + other_fail
    4. All resolved (resolved >= TOTAL_TASKS) AND 0 running/pending -> write summary, break
    5. Stuck detection: 3 consecutive cycles with nothing running, nothing resubmitted -> exit
    6. Sleep CHECK_INTERVAL
```

### Array mode retry script generation

For each OOM'd task, generate a **single-task** retry script:

```bash
# Create: ${RETRY_DIR}/${JOB_PREFIX}_task${TASK_ID}_retry${NEXT_MEM}G.sh
# Based on SUBMIT_SCRIPT but with:
#   --array=${TASK_ID}           (just this one task)
#   --mem=${NEXT_MEM}G           (increased memory)
#   --job-name=${JOB_PREFIX}_t${TASK_ID}_r${NEXT_MEM}G  (unique name for tracking)
#   --output and --error updated to include retry mem in filename

sed -e "s/#SBATCH --array=.*/#SBATCH --array=${TASK_ID}/" \
    -e "s/#SBATCH --mem=.*/#SBATCH --mem=${NEXT_MEM}G/" \
    -e "s/#SBATCH --job-name=.*/#SBATCH --job-name=${JOB_PREFIX}_t${TASK_ID}_r${NEXT_MEM}G/" \
    -e "s|#SBATCH --output=.*|#SBATCH --output=${LOG_DIR}/${JOB_PREFIX}_retry${NEXT_MEM}G_%A_${TASK_ID}.out|" \
    -e "s|#SBATCH --error=.*|#SBATCH --error=${LOG_DIR}/${JOB_PREFIX}_retry${NEXT_MEM}G_%A_${TASK_ID}.err|" \
    "$SUBMIT_SCRIPT" > "${RETRY_SCRIPT}"
```

**Note on batching**: Do NOT batch multiple OOM'd tasks into one `--array=ID1,ID2,...` retry script. Each retry must be a single-task submission so that the state file can unambiguously map `TASK_ID → JOB_ID → STATUS`. Batched retries create a one-to-many mapping that breaks per-task status tracking and can cause duplicate retries or incorrect status attribution.

### Array mode — finding log files for a task

```bash
# For original array job (JOB_ID known):
#   stdout: ${LOG_DIR}/${JOB_PREFIX}_${JOB_ID}_${TASK_ID}.out
#   stderr: ${LOG_DIR}/${JOB_PREFIX}_${JOB_ID}_${TASK_ID}.err
#
# For retry jobs:
#   stdout: ${LOG_DIR}/${JOB_PREFIX}_retry${MEM}G_${RETRY_JOB_ID}_${TASK_ID}.out
#
# The monitor should resolve %A and %a from the SBATCH template using actual job IDs.
# Alternatively, use sacct to find log paths.
```

### Array mode — detecting original array job ID

The monitor must track the exact array job ID to avoid binding to an unrelated historical job that shares the same prefix.

```bash
# At startup, determine the array job ID (strict matching):
# 1. REQUIRED: User provides it via ARRAY_JOB_ID env var, OR
# 2. Check state file for a previously persisted ARRAY_JOB_ID, OR
# 3. Auto-detect: find the MOST RECENT array job matching JOB_PREFIX that is
#    still RUNNING/PENDING, or was submitted within the last 24 hours.
#    If multiple candidates exist, prompt user or exit with error.

if [ -n "$ARRAY_JOB_ID" ]; then
    echo "Using user-provided ARRAY_JOB_ID=$ARRAY_JOB_ID"
elif [ -f "$STATE_FILE" ] && grep -q "^ARRAY_JOB_ID=" "$STATE_FILE"; then
    ARRAY_JOB_ID=$(grep "^ARRAY_JOB_ID=" "$STATE_FILE" | cut -d= -f2)
    echo "Restored ARRAY_JOB_ID=$ARRAY_JOB_ID from state file"
else
    # Auto-detect: prefer running/pending jobs, then most recent completed
    CANDIDATES=$(sacct --name="${JOB_PREFIX}" -X --format=JobID,State --noheader \
        --starttime=now-1day | grep -E 'RUNNING|PENDING' | awk '{print $1}' | head -1 | tr -d ' ')
    if [ -z "$CANDIDATES" ]; then
        CANDIDATES=$(sacct --name="${JOB_PREFIX}" -X --format=JobID --noheader \
            --starttime=now-1day | head -1 | tr -d ' ')
    fi
    if [ -z "$CANDIDATES" ]; then
        echo "ERROR: Cannot auto-detect ARRAY_JOB_ID. Set ARRAY_JOB_ID env var and retry."
        exit 1
    fi
    ARRAY_JOB_ID="$CANDIDATES"
    echo "Auto-detected ARRAY_JOB_ID=$ARRAY_JOB_ID"
fi

# Persist to state file for safe restarts
echo "ARRAY_JOB_ID=$ARRAY_JOB_ID" >> "$STATE_FILE"
```

---

## Summary file (CRITICAL)

When monitor exits (all done or giving up), write `${SUMMARY_FILE}` with:

```
========== MONITOR SUMMARY ==========
Date: $(date)
Job prefix: ${JOB_PREFIX}
Mode: ${MODE}
Memory strategy: ${INITIAL_MEM} + ${MEM_INCREMENT}G increments, max ${MAX_MEM}G

Total tasks: N
First-run success (${INITIAL_MEM}): N
Retried tasks: N
  Succeeded after retry: N (breakdown by mem level)
  Hit max mem (${MAX_MEM}G) still OOM: N
Other failures (non-OOM): N
  [list task IDs]
Incomplete (unknown): N
  [list task IDs]

Overall: N/N completed
======================================
```

Also append this summary to MONITOR_LOG.

## Key design principles
- **Auto-detect mode**: file → array mode, directory → batch mode
- **Prevent duplicate monitors**: At startup, check if another monitor with the same JOB_PREFIX is already running via `squeue`. If found, log a warning and exit immediately.
- **Exit condition**: Exit when ALL tasks are resolved (completed + max_mem_fail + other_fail >= total) AND no jobs running/pending. Also exit after 3 consecutive "stuck" cycles (nothing running, nothing resubmitted, not all resolved) to prevent infinite loops.
- **Auto-detect log directory** from script SBATCH headers, don't hardcode
- **Batch retry naming**: `batch_N_retry{MEM}G.sh` with unique jobnames per mem level
- **Array retry isolation**: each OOM'd task gets its own single-task retry script to preserve unambiguous state tracking
- **State persistence**: array mode uses a state file so monitor can be restarted safely
- **Memory parsing**: extract numeric value from `--mem=XXG` lines for arithmetic
- **Exclude monitor from running count**: When counting running/pending jobs via squeue, exclude the monitor job itself (grep -v "_monitor")
- **Completion detection**: both modes use `sacct` State=COMPLETED as primary detection; batch mode falls back to grepping "Batch N completed at" in .out files; array mode falls back to .out file check
- **Idempotent**: re-running monitor is safe, checks state before acting
- **Summary**: always write summary file on exit with per-mem-level breakdown, list failed task IDs

## Output

Write the complete monitor.sh script to `output_path`. Tell the user:
```
sbatch <output_path>
```

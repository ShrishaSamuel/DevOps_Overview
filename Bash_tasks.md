# Bash & Shell Scripting Interview Preparation — Hands-On Tasks

> A curated set of real-world Shell Scripting challenges organized by difficulty.  
> Each task mirrors scenarios you will encounter in DevOps interviews and on the job.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Automation & Design Tasks](#automation--design-tasks)

---

## Beginner Tasks

---

### Task B1 — Write a Safe, Logged Deployment Script

**Problem Statement**  
A team manually runs 6 commands to deploy an application. Write a Bash script that automates those steps: stop the service, back up the current binary, copy the new binary, set permissions, start the service, and verify it is running. The script must log every step with a timestamp, exit immediately on any failure, and send a Slack webhook notification on success or failure.

**Expected Outcome**
- Script starts with `#!/usr/bin/env bash` and `set -euo pipefail`.
- Every action is logged to `/var/log/deploy.log` with format `[2026-06-29 14:30:01] INFO: Stopping service...`.
- If any step fails, the script exits non-zero and logs `ERROR:` with the failed command.
- Slack notification is sent with the deployment result and the hostname.
- Running the script twice in a row produces two log entries — logs are appended, not overwritten.

**Hints**
- `set -e` exits on error; `set -u` treats unset variables as errors; `set -o pipefail` catches failures in pipes.
- Log function: `log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/deploy.log; }`.
- Use `trap 'log "ERROR: Script failed at line $LINENO"' ERR` to catch any failure.
- Slack webhook: `curl -s -X POST -H 'Content-type: application/json' --data '{"text":"'"$MSG"'"}' "$WEBHOOK_URL"`.
- Store `WEBHOOK_URL` in an environment variable, never hardcoded in the script.

**Skills Tested**
- `set -euo pipefail` and why it matters
- Timestamped logging with `tee`
- `trap ERR` for error handling
- Environment variables for configuration
- `$LINENO` and `$BASH_SOURCE` for debugging context

---

### Task B2 — Parse and Summarize a Log File with grep, awk, and sed

**Problem Statement**  
An nginx access log (`/var/log/nginx/access.log`) contains millions of lines. Write a script that produces a daily summary report: top 10 most requested URLs, top 10 IPs by request count, count of each HTTP status code, and total bytes transferred. The report must be readable and written to a dated file like `report-2026-06-29.txt`.

**Expected Outcome**
- Report file created at `./reports/report-$(date +%F).txt`.
- Top 10 URLs sorted by count descending with counts displayed.
- Top 10 IPs sorted by request count descending.
- Status code summary: `200: 45321`, `404: 892`, `500: 13`, etc.
- Total bytes transferred in human-readable format (MB or GB).
- Script runs in under 30 seconds on a 500MB log file.

**Hints**
- nginx combined log format: `IP - - [timestamp] "METHOD URL PROTOCOL" STATUS BYTES "referer" "user-agent"`.
- URL field (column 7): `awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -10`.
- Status codes (column 9): `awk '{print $9}' access.log | sort | uniq -c | sort -rn`.
- Total bytes (column 10): `awk '{sum += $10} END {print sum}' access.log` — format with `awk 'BEGIN{printf "%.2f GB\n", sum/1073741824}'`.
- Use `mkdir -p reports` before writing the report file.

**Skills Tested**
- `awk` for columnar log parsing
- `sort | uniq -c | sort -rn` pipeline for frequency counting
- `grep` filtering and pattern matching
- Output formatting and report generation
- Pipeline performance for large files

---

### Task B3 — Write a Backup Script with Retention Policy

**Problem Statement**  
Write a Bash script that backs up a directory (`/opt/app/data`) as a compressed tarball to `/backups/`, names the archive with the current timestamp, keeps only the last 7 daily backups (deletes older ones), logs all actions, and sends an alert if disk usage in `/backups/` exceeds 80%.

**Expected Outcome**
- Archive created: `/backups/data-backup-2026-06-29T14-30-00.tar.gz`.
- After 8+ runs (simulated with date offsets), only 7 most recent archives remain.
- Log file at `/var/log/backup.log` shows each backup's size and duration.
- If disk usage > 80%, an alert message is printed (and optionally emailed).
- Script is idempotent — running it multiple times per day creates multiple timestamped backups safely.

**Hints**
- Create tarball: `tar -czf "/backups/data-backup-$(date +%Y-%m-%dT%H-%M-%S).tar.gz" /opt/app/data`.
- List old backups sorted by date: `ls -1t /backups/data-backup-*.tar.gz | tail -n +8`.
- Delete old backups: `ls -1t /backups/data-backup-*.tar.gz | tail -n +8 | xargs rm -f`.
- Disk usage check: `df /backups | awk 'NR==2 {print $5}' | tr -d '%'` — compare with `if [[ $usage -gt 80 ]]`.
- Measure duration: `START=$(date +%s)` ... `DURATION=$(( $(date +%s) - START ))`.

**Skills Tested**
- `tar` for compressed archiving
- File retention with `ls`, `tail`, `xargs rm`
- `df` and disk usage monitoring
- Date arithmetic and timestamp formatting
- Idempotent script design

---

### Task B4 — Monitor System Resources and Alert on Thresholds

**Problem Statement**  
Write a monitoring script that checks CPU usage, memory usage, disk usage for all mounted filesystems, and the number of running processes every 60 seconds. If any metric exceeds its threshold (CPU > 85%, memory > 90%, disk > 80%), write an alert to a log file and output it to stderr. The script must run continuously until killed.

**Expected Outcome**
- Script runs in an infinite loop with a 60-second sleep between checks.
- Each check cycle logs a timestamped status line even if no thresholds are exceeded.
- Alert lines are written to both the log and stderr (so they're visible in terminal and capturable by monitoring).
- CPU usage is averaged over 3 samples (not a single instantaneous reading).
- `kill <pid>` cleanly stops the script (graceful shutdown with `trap SIGTERM`).

**Hints**
- CPU usage: `top -bn3 | grep 'Cpu(s)' | tail -1 | awk '{print $2}' | cut -d. -f1` (average of 3 batches).
- Or use: `awk '{u=$2+$4; t=$2+$4+$5; if (NR==1){u1=u; t1=t;} else print (u-u1)/(t-t1)*100}' <(grep 'cpu ' /proc/stat) <(sleep 1; grep 'cpu ' /proc/stat)`.
- Memory: `free | awk '/^Mem:/{printf "%.0f", $3/$2*100}'`.
- Disk: `df -h | awk 'NR>1 && /^\// {gsub(/%/,"",$5); if($5+0 > threshold) print $0}'`.
- `trap 'echo "Shutting down..."; exit 0' SIGTERM SIGINT` for clean exit.

**Skills Tested**
- `/proc/stat` and `/proc/meminfo` for system metrics
- `free`, `df`, `top` parsing
- Infinite loop with `sleep` and clean exit via `trap`
- Threshold alerting logic
- stderr vs stdout output routing

---

## Intermediate Tasks

---

### Task I1 — Build a Multi-Function Script with Argument Parsing

**Problem Statement**  
Write a Swiss-army-knife operations script (`ops.sh`) with subcommands: `ops.sh deploy <env>`, `ops.sh rollback <env>`, `ops.sh status <env>`, and `ops.sh logs <env> [lines]`. Each subcommand validates its arguments, shows a `--help` message if called incorrectly, and all share a common logging and configuration-loading function. Unknown subcommands print usage and exit with code 2.

**Expected Outcome**
- `./ops.sh deploy production` executes the deploy logic for production.
- `./ops.sh logs staging 100` shows last 100 lines of the staging app log.
- `./ops.sh` (no args) prints full usage with all subcommands listed.
- `./ops.sh unknown-cmd` exits with code 2 and a clear error message.
- Valid environments are `dev`, `staging`, `production` — invalid env exits with code 1 and a list of valid options.
- A config file (`./ops.conf`) is loaded at startup and variables are available in all subcommands.

**Hints**
- Use a `case "$1" in` switch for subcommand dispatch.
- Argument validation: `[[ -z "$2" ]] && usage && exit 1`.
- Load config: `[[ -f ./ops.conf ]] && source ./ops.conf || { echo "Config not found"; exit 1; }`.
- Valid env check: `valid_envs=("dev" "staging" "production"); [[ ! " ${valid_envs[*]} " =~ " $env " ]] && echo "Invalid env" && exit 1`.
- Use a `usage()` function and call it from `--help` and the default case.
- `"${@:2}"` captures all arguments from the second onwards — useful for passing remaining args to subcommands.

**Skills Tested**
- `case` statement for subcommand dispatch
- Argument validation and usage functions
- `source` for config file loading
- Array-based valid value checking
- Exit code conventions (0 = success, 1 = error, 2 = misuse)

---

### Task I2 — Process Files in Parallel with Job Control

**Problem Statement**  
You have 10,000 CSV files in `/data/raw/` that each need to be processed (validated, transformed, and moved to `/data/processed/`). A serial script takes 6 hours. Rewrite it to process files in parallel, limiting concurrency to `N` workers (where N is the number of CPU cores). Failed files must be logged separately and not block the rest.

**Expected Outcome**
- Processing uses exactly `$(nproc)` parallel workers at any given time.
- Completed files are moved to `/data/processed/`.
- Failed files are logged to `/data/failed.log` with the filename and error reason.
- A progress indicator shows `[1234/10000] files processed` updating in place.
- Total runtime is approximately `serial_time / N`.
- Script handles the case where `/data/raw/` has 0 files gracefully.

**Hints**
- Semaphore pattern: `while [[ $(jobs -r | wc -l) -ge $MAX_JOBS ]]; do sleep 0.1; done; process_file "$f" &`.
- Better: use GNU `parallel`: `find /data/raw -name '*.csv' | parallel -j "$(nproc)" process_file {}`.
- Or `xargs -P "$(nproc)" -I{} bash -c 'process_file "$@"' _ {}`.
- In-place progress: `printf "\r[%d/%d] files processed" "$done" "$total"` (no newline).
- Capture failures: `process_file "$f" || echo "$f: failed" >> /data/failed.log`.
- `wait` at the end of the script waits for all background jobs before exiting.

**Skills Tested**
- Bash job control (`&`, `wait`, `jobs -r`)
- Parallel execution with concurrency limiting
- GNU `parallel` and `xargs -P`
- In-place terminal progress updates with `\r`
- Error isolation in parallel workloads

---

### Task I3 — Write a Log Rotation and Archiving Script

**Problem Statement**  
An application writes to `/var/log/app/app.log` and the file has grown to 8 GB, filling the disk. Write a log rotation script that: renames the current log file with a timestamp, sends `SIGHUP` to the application to reopen its log file, compresses the rotated log, removes logs older than 30 days, and runs automatically every night at midnight via cron.

**Expected Outcome**
- `app.log` is renamed to `app.log.2026-06-29` and the application immediately starts writing to a new `app.log`.
- Rotated log is compressed to `app.log.2026-06-29.gz` within 5 minutes.
- Files older than 30 days in `/var/log/app/` are deleted automatically.
- `crontab -l` shows the job: `0 0 * * * /opt/scripts/rotate_logs.sh`.
- Script is safe to run manually mid-day without disrupting the application.

**Hints**
- Rename: `mv /var/log/app/app.log "/var/log/app/app.log.$(date +%F)"`.
- Signal application: `pkill -HUP -f "app_process_name"` or `kill -HUP $(cat /var/run/app.pid)`.
- Compress in background (non-blocking): `gzip "/var/log/app/app.log.$(date +%F)" &`.
- Remove old logs: `find /var/log/app/ -name 'app.log.*.gz' -mtime +30 -delete`.
- Add cron: `(crontab -l 2>/dev/null; echo "0 0 * * * /opt/scripts/rotate_logs.sh >> /var/log/rotate.log 2>&1") | crontab -`.

**Skills Tested**
- Log rotation without losing log lines (SIGHUP pattern)
- `kill` and signal handling from scripts
- `find -mtime` for age-based file deletion
- Cron job installation from a script
- Background compression with `&`

---

### Task I4 — Extract and Transform Data with awk and sed

**Problem Statement**  
A legacy system exports a pipe-delimited file (`data.psv`) with inconsistent formatting: mixed-case values, leading/trailing whitespace, some fields quoted, and dates in `MM/DD/YYYY` format. Transform it to a clean CSV with: lowercase values, trimmed whitespace, unquoted fields, and dates converted to `YYYY-MM-DD`. Write a single pipeline (no temp files) that handles 1 million rows in under 60 seconds.

**Sample Input**
```
" John Doe " | " ADMIN " | " 06/29/2026 "
" jane smith " | "user " | " 01/15/2025 "
```

**Expected Outcome**
```
john doe,admin,2026-06-29
jane smith,user,2025-01-15
```

**Hints**
- Remove leading/trailing spaces and quotes: `sed 's/^ *"*//;s/"* *$//'` per field, or use `awk` with `gsub`.
- In `awk`, split on `|` with `FS="|"` and process each field: `gsub(/^[ \t"]+|[ \t"]+$/, "", $i)`.
- Convert to lowercase: `echo "$var" | tr '[:upper:]' '[:lower:]'` or `awk '{print tolower($0)}'`.
- Date conversion: `sed 's|\([0-9]\{2\}\)/\([0-9]\{2\}\)/\([0-9]\{4\}\)|\3-\1-\2|g'`.
- Chain as one pipeline: `cat data.psv | awk ... | sed ... | tr ...` — avoid writing intermediate files.

**Skills Tested**
- `awk` with custom field separator (`FS`)
- `sed` for regex substitution and date reformatting
- `tr` for character-level transformation
- Pipeline chaining for large file processing
- Regex for whitespace trimming and quote removal

---

### Task I5 — Write a Health Check and Auto-Recovery Script

**Problem Statement**  
A critical service (`myapp`) occasionally crashes and doesn't auto-restart (it has no systemd unit file). Write a watchdog script that checks if the process is running every 30 seconds, restarts it if it's down, limits restart attempts to 3 within a 5-minute window (to avoid a crash-loop), logs all events with timestamps, and notifies a webhook after 3 failed restart attempts.

**Expected Outcome**
- If `myapp` is running: log `HEALTH_CHECK: OK` and sleep 30s.
- If `myapp` is down: attempt restart, log the attempt number.
- After 3 failed restart attempts within 5 minutes: send webhook alert, log `CRITICAL: giving up`, and stop restarting (but keep health checking).
- Restart counter resets after 5 minutes of successful operation.
- A second instance of the watchdog script detects the first is running and exits gracefully (prevent duplicate watchdogs).

**Hints**
- Check if process is running: `pgrep -f "myapp" > /dev/null 2>&1`.
- Prevent duplicates with a lockfile: `LOCKFILE=/tmp/watchdog.lock; exec 9>"$LOCKFILE"; flock -n 9 || { echo "Already running"; exit 1; }`.
- Track restart count with a timestamp array or a counter file: `echo "$(date +%s)" >> /tmp/restart_times`.
- Count recent restarts: `awk -v cutoff="$(( $(date +%s) - 300 ))" '$1 > cutoff' /tmp/restart_times | wc -l`.
- Reset counter when app runs healthy for a full 5-minute window.

**Skills Tested**
- `pgrep`/`pkill` for process management
- `flock` for exclusive lockfiles (prevent duplicate daemons)
- Sliding window rate limiting logic
- Epoch time arithmetic with `date +%s`
- Process restart and crash-loop detection

---

## Advanced Tasks

---

### Task A1 — Build a Zero-Downtime Deployment Script

**Problem Statement**  
Write a Bash deployment script that performs a zero-downtime rolling update across 5 application servers. For each server: SSH in, stop the old version, deploy the new binary, start it, wait for the health check (`/health` endpoint) to return `200` within 60 seconds, then move to the next server. If any server fails the health check, immediately roll back ALL already-updated servers to the old version and exit non-zero.

**Expected Outcome**
- Servers are updated one at a time (not all at once).
- Between each server update, a 5-second wait ensures the load balancer routes away from it.
- Health check polls every 5 seconds for up to 60 seconds before declaring failure.
- On any failure, all previously updated servers are rolled back.
- Script produces a deployment report showing time-per-server and overall status.

**Hints**
- SSH without interactive prompts: `ssh -o StrictHostKeyChecking=no -o BatchMode=yes user@host "command"`.
- Health check loop: `for i in $(seq 1 12); do curl -sf http://$host/health && break; sleep 5; done || { rollback; exit 1; }`.
- Track updated servers in an array: `updated_servers+=("$server")` — use this for rollback.
- Rollback function iterates `"${updated_servers[@]}"` in reverse order.
- Use `set +e` before the health check loop and `set -e` after — you need to handle the exit code manually.

**Skills Tested**
- SSH automation from Bash scripts
- Rolling update loop with state tracking
- Health check polling with timeout
- Array-based rollback state management
- `set +e` / `set -e` toggling for controlled error handling

---

### Task A2 — Write a Secret-Safe Configuration Management Script

**Problem Statement**  
Your team stores application config in a mix of environment variables, config files, and AWS SSM Parameter Store. Write a script that: loads a config template (`app.conf.tmpl`) with placeholders like `${DB_HOST}`, resolves each placeholder from SSM first (falling back to environment variables, then defaults), writes the rendered config to `/etc/app/app.conf`, validates the rendered config has no unreplaced placeholders, and never logs or echoes any secret values.

**Expected Outcome**
- `app.conf.tmpl` with `${DB_HOST}`, `${DB_PASSWORD}`, `${APP_PORT}` is rendered correctly.
- `DB_PASSWORD` is fetched from SSM (`/myapp/prod/db_password`) and never appears in logs.
- `APP_PORT` falls back to environment variable `APP_PORT` or default `8080` if not in SSM.
- Script fails with a clear error if any placeholder remains unresolved in the output.
- Output file permissions are `640` (owner read/write, group read, no others).

**Hints**
- Fetch from SSM: `aws ssm get-parameter --name "/myapp/prod/db_password" --with-decryption --query 'Parameter.Value' --output text 2>/dev/null`.
- `envsubst` renders templates: `envsubst < app.conf.tmpl > /etc/app/app.conf` — but export variables first.
- Detect unreplaced placeholders: `grep -E '\$\{[A-Z_]+\}' /etc/app/app.conf && echo "ERROR: unreplaced placeholders" && exit 1`.
- Never log secrets: use `log "Fetched DB_PASSWORD from SSM"` — not `log "DB_PASSWORD=$DB_PASSWORD"`.
- Set file permissions: `install -m 640 -o root -g appgroup /dev/stdin /etc/app/app.conf <<< "$rendered_config"`.

**Skills Tested**
- `envsubst` for template rendering
- AWS SSM Parameter Store integration from Bash
- Secret-safe logging discipline
- Unreplaced placeholder detection with `grep`
- `install` for permission-controlled file creation

---

### Task A3 — Parse and Correlate Distributed System Logs

**Problem Statement**  
A microservices system writes logs to `/var/log/services/*.log`. Each log line contains a `trace_id`. A customer complaint references `trace_id: abc-123`. Write a script that: searches all service log files for that trace ID, collects all matching lines, sorts them by timestamp (the format may differ between services), outputs a unified timeline, and highlights `ERROR` and `WARN` lines in color if the terminal supports it.

**Expected Outcome**
- All lines matching `trace_id: abc-123` across all log files are collected.
- Lines are sorted chronologically regardless of source file or timestamp format.
- Output shows `[service-name] timestamp: log message` in a unified view.
- `ERROR` lines are printed in red, `WARN` in yellow, `INFO` in green (using ANSI codes, only if `tty`).
- Script handles timestamp formats: ISO 8601, Unix epoch, and `Mon DD HH:MM:SS`.
- Runs in under 5 seconds even on 50 log files totaling 2 GB.

**Hints**
- `grep -r "trace_id: abc-123" /var/log/services/ --include="*.log" -h` collects matching lines.
- Normalize timestamps to epoch for sorting: `date -d "$timestamp" +%s` — handle different formats with `case`.
- Add service name from filename: `grep -H "trace_id" *.log | awk -F: '{print $1, $0}'`.
- ANSI color only when stdout is a tty: `[[ -t 1 ]] && RED='\033[0;31m' || RED=''`.
- `sort -k1,1n` after prepending epoch timestamp — strip the epoch prefix from output.

**Skills Tested**
- Multi-file log correlation with `grep -r`
- Timestamp normalization and sorting across formats
- ANSI color codes and `tty` detection
- `awk` for log line augmentation
- Performance with `grep` on large files (`-l`, `--include`)

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Debug a Script That Works Locally but Fails in CI

**Problem Statement**  
A deployment script passes on every developer's machine but fails in CI with cryptic errors like `[: ==: unary operator expected`, `command not found`, and `bad substitution`. The script runs in CI as `sh deploy.sh` not `bash deploy.sh`. Find all the issues and fix them to work in both environments.

**Expected Outcome**
- All bash-specific syntax identified and either fixed or the shebang changed to `#!/usr/bin/env bash`.
- `[[ ]]` replaced with `[ ]` or proper quoting added if POSIX `sh` is required.
- Variables that might be empty are always quoted: `"$var"` not `$var`.
- `==` inside `[ ]` replaced with `=` (POSIX sh uses `=`, not `==`).
- Script passes `shellcheck deploy.sh` with zero warnings.

**Hints**
- `#!/bin/sh` invokes `dash` on Ubuntu (not bash) — `[[ ]]`, `local`, arrays, and `source` are bash-only.
- Unquoted variables: `[ $VAR == "foo" ]` fails if `$VAR` is empty → becomes `[ == "foo" ]` (unary error).
- Use `shellcheck` (free, installable with `apt install shellcheck`) — it catches 90% of these issues.
- `bash -n deploy.sh` checks syntax without executing.
- `bash -x deploy.sh` traces execution line-by-line — the actual command after variable expansion is shown.

**Skills Tested**
- POSIX `sh` vs Bash compatibility
- Variable quoting rules (`"$var"` prevents word splitting and globbing)
- `shellcheck` static analysis
- `bash -x` execution tracing
- `bash -n` syntax checking

---

### Task T2 — Find and Kill a Port-Hogging Process

**Problem Statement**  
An application fails to start because port `8080` is already in use. Write a script that: identifies which process is using the port, displays the process name, PID, and the user running it, asks for confirmation before killing it, optionally accepts `--force` flag to skip confirmation, and verifies the port is free after killing.

**Expected Outcome**
- `./free_port.sh 8080` shows: `Port 8080 is used by: nginx (PID 1234, user www-data)`.
- Prompts: `Kill process? [y/N]:` — 'y' kills it, anything else aborts.
- `./free_port.sh 8080 --force` kills without prompting.
- After killing, confirms `Port 8080 is now free`.
- If no process is using the port: `Port 8080 is not in use`.

**Hints**
- Find process by port: `ss -tlnp sport = :8080` or `lsof -ti tcp:8080` or `fuser 8080/tcp`.
- Get PID: `ss -tlnp | grep ':8080' | awk '{print $6}' | grep -oP 'pid=\K[0-9]+'`.
- Process name from PID: `ps -p $PID -o comm=`.
- User from PID: `ps -p $PID -o user=`.
- Argument parsing: `while [[ "$#" -gt 0 ]]; do case "$1" in --force) FORCE=true;; *) PORT="$1";; esac; shift; done`.

**Skills Tested**
- `ss`, `lsof`, `fuser` for port investigation
- `ps` for process inspection
- Argument parsing with `while`/`case`
- User-confirmation prompt pattern
- Port availability verification

---

### Task T3 — Diagnose High Disk I/O with a Script

**Problem Statement**  
A server is experiencing severe I/O slowdown. Write a diagnostic script that: identifies the top 5 processes by disk read/write using `/proc`, shows which files are currently open by those processes, finds the largest files modified in the last 24 hours on all mounted filesystems, checks for inode exhaustion (separate from disk space), and writes a diagnostic report with all findings.

**Expected Outcome**
- Top 5 I/O processes shown with PID, process name, read bytes, and write bytes.
- Open files for each top process listed from `/proc/<pid>/fd/`.
- Top 10 largest recently modified files shown with size and path.
- Filesystems with inode usage > 80% flagged as warnings.
- Report written to `/tmp/io_diagnosis_$(date +%F_%H%M%S).txt`.

**Hints**
- Read/write bytes per process: `cat /proc/$PID/io | grep -E '^read_bytes|^write_bytes'`.
- Top I/O processes: iterate over `/proc/[0-9]*/io` and sum `read_bytes + write_bytes`, sort descending.
- Open files: `ls -la /proc/$PID/fd/ | grep -v "^total"` — symlinks point to actual file paths.
- Large recent files: `find / -xdev -type f -newer /tmp/24h_ago -printf '%s %p\n' 2>/dev/null | sort -rn | head -10`.
- Inode usage: `df -i | awk 'NR>1 {gsub(/%/,"",$5); if($5+0 > 80) print "WARNING: "$0}'`.

**Skills Tested**
- `/proc` filesystem navigation for system diagnostics
- Inode vs block storage concepts
- `find -newer` for recently modified files
- `df -i` for inode monitoring
- Diagnostic report generation

---

### Task T4 — Recover from a Broken PATH and Environment

**Problem Statement**  
A junior engineer ran `export PATH=/broken/path` in a shell session, breaking all commands. You're SSH'd into the server and your session is affected — `ls`, `cat`, `vim`, all fail with `command not found`. Fix the session, diagnose how it happened, and write a script that validates and repairs a broken PATH automatically.

**Expected Outcome**
- Session is recovered without logging out (using full paths or shell builtins).
- `PATH` is restored to a sane default including `/usr/bin:/usr/sbin:/bin:/sbin`.
- A `repair_env.sh` script checks for common PATH issues and auto-corrects them.
- Script detects: empty PATH, missing `/usr/bin` in PATH, duplicate entries, and non-existent directories.
- `source repair_env.sh` restores the environment without requiring a new login.

**Hints**
- In a broken PATH session, use full paths: `/bin/ls`, `/usr/bin/cat`, `/usr/bin/vim`.
- Or use builtins: `echo`, `cd`, `read` work without PATH.
- `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` restores a safe default.
- Detect missing paths: `IFS=: read -ra DIRS <<< "$PATH"; for d in "${DIRS[@]}"; do [[ -d "$d" ]] || echo "Missing: $d"; done`.
- Deduplicate PATH: `PATH=$(echo "$PATH" | tr ':' '\n' | awk '!seen[$0]++' | tr '\n' ':' | sed 's/:$//')`.

**Skills Tested**
- Bash builtins vs external commands
- Full path command execution without PATH
- PATH validation and deduplication
- `IFS` and array splitting
- `source` vs executing a script (subshell vs current shell)

---

## Automation & Design Tasks

---

### Task D1 — Build a Full Infrastructure Provisioning Script

**Problem Statement**  
Write a script that provisions a complete application environment on a fresh Ubuntu 22.04 server: installs Docker, Docker Compose, and required system packages, creates a dedicated `appuser`, sets up the directory structure, copies config files from a template directory, generates a self-signed TLS certificate, starts the application with Docker Compose, runs a smoke test, and is fully idempotent (safe to run multiple times).

**Expected Outcome**
- Running the script on a fresh server produces a fully configured, running environment.
- Running it again on an already-configured server makes no changes and exits with code 0.
- Each provisioning step checks if already done before doing it (idempotent).
- Self-signed cert is only regenerated if it's missing or expiring within 30 days.
- Smoke test: `curl -sk https://localhost/health | jq '.status == "ok"'` returns `true`.

**Hints**
- Idempotent package install: `dpkg -l docker-ce &>/dev/null || apt-get install -y docker-ce`.
- Idempotent user creation: `id appuser &>/dev/null || useradd -m -s /bin/bash appuser`.
- Idempotent directory: `mkdir -p /opt/app/{config,data,logs}` — `mkdir -p` is naturally idempotent.
- Check cert expiry: `openssl x509 -checkend $((30*86400)) -noout -in /etc/app/cert.pem || generate_cert`.
- Use a `STEP_DONE_DIR=/var/lib/provision/done/` directory and `touch` files to track completed steps.

**Skills Tested**
- Idempotent script design
- Package and user management from scripts
- OpenSSL certificate generation and expiry checking
- Docker Compose startup from scripts
- Smoke test automation

---

### Task D2 — Write a Cron-Based Database Backup and Restore System

**Problem Statement**  
Build a complete backup system for a PostgreSQL database: a backup script that dumps all databases, compresses and encrypts the dump (using GPG), uploads to an S3 bucket, cleans up local temp files, and records backup metadata to a local SQLite registry. A separate restore script lists available backups, downloads the selected one, decrypts and restores it. A test script verifies the backup is restorable without touching the production database.

**Expected Outcome**
- `backup.sh` produces an encrypted file `backup-<db>-<timestamp>.sql.gz.gpg` in S3.
- `restore.sh --list` shows available backups from the SQLite registry with size and timestamp.
- `restore.sh --backup-id 42 --target-db testdb` restores without touching production.
- `verify.sh` restores the latest backup to a temp database, runs a row count check, and drops the temp database.
- Cron entry runs backup nightly and verify weekly.

**Hints**
- Dump all databases: `pg_dumpall -U postgres | gzip | gpg --encrypt --recipient backup@company.com`.
- Or dump individually: `for db in $(psql -tAc "SELECT datname FROM pg_database WHERE datistemplate=false"); do pg_dump $db | ...; done`.
- Upload to S3: `aws s3 cp backup.sql.gz.gpg s3://my-backups/postgres/`.
- SQLite metadata: `sqlite3 /var/lib/backup/registry.db "INSERT INTO backups (filename, size, timestamp) VALUES ('$file', $size, datetime('now'))"`.
- Decrypt for restore: `gpg --decrypt backup.sql.gz.gpg | gunzip | psql -U postgres -d $TARGET_DB`.

**Skills Tested**
- PostgreSQL `pg_dump`/`pg_dumpall`/`pg_restore`
- GPG encryption for backup security
- AWS S3 CLI from shell scripts
- SQLite as a lightweight metadata store
- Backup verification and restore testing methodology

---

## Quick Reference — Essential Commands

| Tool | Key Usage |
|---|---|
| `awk '{print $2}' file` | Print second field (space-delimited) |
| `awk -F: '{print $1}' /etc/passwd` | Custom field separator |
| `awk 'NR>1 && $3>100'` | Skip header, filter by numeric field |
| `sed 's/foo/bar/g'` | Global string replacement |
| `sed -n '10,20p'` | Print lines 10–20 |
| `sed '/^#/d'` | Delete comment lines |
| `grep -E 'pattern'` | Extended regex |
| `grep -v 'exclude'` | Invert match |
| `grep -oP '(?<=key=)\w+'` | Extract value after `key=` (PCRE) |
| `sort -k2,2n -k1,1` | Sort by field 2 numerically, then field 1 |
| `sort \| uniq -c \| sort -rn` | Frequency count, most common first |
| `find / -name '*.log' -mtime +7 -size +100M` | Old large logs |
| `xargs -I{} -P4 cmd {}` | Parallel execution |
| `tee -a file` | Write to stdout AND append to file |
| `exec > >(tee -a "$LOGFILE") 2>&1` | Redirect ALL script output to log |
| `date +%s` | Unix epoch timestamp |
| `date -d '7 days ago' +%F` | Date arithmetic |
| `openssl rand -hex 16` | Generate random token |
| `pgrep -f "process name"` | Find PID by command string |
| `ss -tlnp sport = :8080` | Find process using a port |
| `lsof -p $PID` | All files opened by a process |
| `strace -p $PID` | System calls made by a running process |
| `bash -x script.sh` | Trace execution |
| `shellcheck script.sh` | Static analysis |

---

## Interview Cheat Sheet

- **"What does `set -euo pipefail` do?"**  
  `-e`: exit immediately if any command exits non-zero. `-u`: treat unset variables as errors. `-o pipefail`: the pipeline's exit status is the last non-zero exit code (without this, `false | true` exits 0). Every production script should start with this.

- **"What is the difference between `[ ]` and `[[ ]]`?"**  
  `[ ]` is POSIX `test`, works in any shell, has quoting quirks. `[[ ]]` is a Bash built-in — safer with unquoted variables, supports `==` pattern matching, `=~` regex, and `&&`/`||` without quoting issues. Use `[[ ]]` in Bash scripts.

- **"When do you use `$()` vs backticks?"**  
  Both run command substitution. `$()` is nestable (`$(cmd1 $(cmd2))`), more readable, and the modern standard. Backticks are legacy — avoid them.

- **"What is the difference between `source script.sh` and `./script.sh`?"**  
  `./script.sh` runs in a subshell — variable changes, `cd`, and `export` don't affect the parent shell. `source script.sh` (or `. script.sh`) runs in the current shell — all changes persist. Use `source` for setting environment variables.

- **"How do you handle errors in a pipe?"**  
  `set -o pipefail` makes the pipe fail if any command in it fails. Without it, `failing_cmd | grep something` succeeds if `grep` succeeds. Also use `PIPESTATUS` array: `${PIPESTATUS[0]}` is the exit code of the first command.

- **"What is word splitting and when does it cause bugs?"**  
  Unquoted variables undergo word splitting — `$var` with value `hello world` becomes two arguments. Always quote: `"$var"`. Also affects `for f in $files` — use `for f in "${files[@]}"` for arrays.

- **"How do you make a script idempotent?"**  
  Check before acting: `[[ -f /etc/done ]] || install_thing`. Use tools that are idempotent by nature (`mkdir -p`, `apt-get install` with already-installed packages). Store state in marker files or check current system state before each action.

- **"What is a here-document and when do you use it?"**  
  `<<EOF` embeds multi-line strings without creating temp files. `<<'EOF'` (quoted) disables variable expansion — useful for passing scripts to `ssh` or `sudo bash`. `<<-EOF` strips leading tabs for indented heredocs.

---

## Study Path

```
Week 1  → B1, B2, B3, B4         (Script structure, log parsing, backup, monitoring)
Week 2  → I1, I2, I3             (Argument parsing, parallel processing, log rotation)
Week 3  → I4, I5, T1, T2         (Text transformation, watchdog, sh vs bash, port debugging)
Week 4  → T3, T4, A1             (I/O diagnosis, broken PATH, rolling deployment)
Week 5  → A2, A3, D1             (Secret management, log correlation, provisioning)
Week 6  → D2 + shellcheck all    (Backup system, static analysis pass on all scripts)
```

---

*Write every script with `shellcheck` open in a split terminal. The discipline of zero `shellcheck` warnings produces production-quality scripts that won't surprise you at 3am.*

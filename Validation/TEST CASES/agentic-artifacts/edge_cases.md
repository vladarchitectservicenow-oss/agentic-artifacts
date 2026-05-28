Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# agentic-artifacts – Edge Case Tests

## Overview
Edge case tests probe boundary conditions, rare inputs, and environmental stress scenarios that fall outside normal operating parameters but must not cause data loss, corruption, or undefined behavior.

---

### EC-01: Empty ServiceNow Instance – Zero Tables
**Condition**: The target ServiceNow instance has no custom tables — only OOB (`sys_*`, `task`, etc.).  
**Execution**: Run `agentic_artifacts discover --scope custom`.  
**Expected Behavior**: The tool prints `"No custom tables found in scope 'custom'."` to stderr and exits with code 0. No exception traceback. The output directory is created but contains only a `.gitkeep` file.  
**Priority**: P1 — Rare but must be handled gracefully in demo/pilot environments.

---

### EC-02: 50k+ Record Table Scan
**Condition**: A ServiceNow table (e.g., `sys_audit`) contains ≥50,000 rows.  
**Execution**: Run `agentic_artifacts generate --type business_rule --table sys_audit --row-limit 0` (0 = no limit).  
**Expected Behavior**: The tool paginates using `sysparm_offset` and `sysparm_limit` (500 per page). Memory consumption stays below 512 MB. Progress is logged every 5,000 records. If the scan exceeds 15 minutes, a warning is emitted but the job is not aborted.  
**Priority**: P1 — Large instances are common in enterprise deployments.

---

### EC-03: Null / Missing Config Properties
**Condition**: The `config.yaml` file has keys with explicit `null` values or entirely missing required sections.  
**Execution**: Populate `config.yaml` as follows and run `agentic_artifacts health-check`:

```yaml
instance_url: null
auth:
  username: admin
  # password key intentionally absent
templates_dir: /nonexistent/path
```

**Expected Behavior**: The tool reports three distinct errors: (1) `instance_url` is null, (2) `auth.password` is missing, (3) `templates_dir` does not exist on disk. The exit code is non-zero. No partial config is applied.  
**Priority**: P0 — Null config must never silently default to production.

---

### EC-04: Unicode / Emoji in Field Names
**Condition**: A ServiceNow table contains a column whose label is `"🔥 Urgent Escalation 🚨 – Résumé vérifiée"`.  
**Execution**: Generate an artifact that references this field in its `condition` and `script` blocks.  
**Expected Behavior**: The Jinja2 template engine encodes all strings as UTF-8. The output file contains the exact emoji and accented characters. Opening the file in VS Code or `cat` displays them correctly. `file -i output.js` reports `charset=utf-8`.  
**Priority**: P2 — Cosmetic but important for internationalized deployments.

---

### EC-05: Network Timeout Mid-Scan
**Condition**: A long-running table scan loses network connectivity for 45 seconds after processing 3,200 of 10,000 pages.  
**Execution**: Use `tc qdisc` to introduce a 45 s network outage on the SN instance route, then restore it.  
**Expected Behavior**: The tool detects `requests.exceptions.ReadTimeout` / `ConnectionError`, retries with exponential backoff (capped at 5 retries), and resumes scanning from the last successful page. No records are skipped or double-counted.  
**Priority**: P1 — Network blips are inevitable in cloud-to-cloud integrations.

---

### EC-06: Corrupted Jinja2 Template File
**Condition**: A `.j2` template file contains a syntax error — an unclosed `{% if %}` block without a matching `{% endif %}`.  
**Execution**: Run `agentic_artifacts generate --template corrupted_template.j2`.  
**Expected Behavior**: The tool catches `jinja2.exceptions.TemplateSyntaxError`, logs the exact file path and line number, and skips that artifact. The batch continues processing remaining templates. Exit code reflects partial failure (e.g., 2).  
**Priority**: P1 — A single malformed template must not block an entire batch.

---

### EC-07: Concurrent Artifact Generation – Race Condition
**Condition**: Two `agentic_artifacts generate` processes write to the same output directory simultaneously.  
**Execution**: Launch process A and process B (both with `--output-dir /tmp/shared_out`) within 0.5 s of each other.  
**Expected Behavior**: The tool uses advisory file locking (`fcntl.flock` on Linux). One process acquires the lock; the other fails immediately with `"Output directory /tmp/shared_out is locked by PID #####. Use --force to override."`. No interleaved file writes occur.  
**Priority**: P0 — Data corruption from concurrent writes is unacceptable.

---

### EC-08: GitHub Repo with No README
**Condition**: A template source repository has zero files named `README.md` or `README.rst`.  
**Execution**: Point the tool at such a repo: `agentic_artifacts generate --repo https://github.com/example/no-readme-repo`.  
**Expected Behavior**: The tool logs `"WARNING: No README found in repo no-readme-repo; README generation skipped."` and proceeds to generate other artifacts from the same repo (e.g., `.py` scripts). Exit code is 0 but the log contains the warning.  
**Priority**: P2 — Non-blocking but must not crash.

---

### EC-09: Python 3.12 vs 3.10 Behavioral Differences
**Condition**: The tool is running under Python 3.12, which has stricter `dict` ordering guarantees and deprecated `distutils`.  
**Execution**: Run the full test suite under both Python 3.10 (minimum supported) and Python 3.12.  
**Expected Behavior**: All tests pass on both versions. Explicit `sys.version_info` checks are in place for any version-sensitive code paths. `distutils` imports are gated with `try/except ImportError`.  
**Priority**: P1 — LTS Linux distributions ship different Python versions.

---

### EC-10: Disk Full During Artifact Write
**Condition**: The output filesystem has less than 1 KB of free space when the tool attempts to write a 15 KB artifact file.  
**Execution**: Create a tiny filesystem: `dd if=/dev/zero of=/tmp/smallfs.img bs=1K count=10; mkfs.ext4 /tmp/smallfs.img; mount -o loop /tmp/smallfs.img /mnt/small`. Run the tool with `--output-dir /mnt/small`.  
**Expected Behavior**: The tool receives `OSError: [Errno 28] No space left on device`. It logs the error, cleans up the partially-written file (unlink), and exits with code 28. No zero-byte or truncated files remain.  
**Priority**: P1 — Must fail safely without corrupting the output directory.

---

### EC-11: Invalid SSL Certificate on SN Instance
**Condition**: The ServiceNow instance presents a self-signed or expired TLS certificate.  
**Execution**: Point `SERVICENOW_INSTANCE` at a mock server with an invalid cert. Run any command that calls the REST API.  
**Expected Behavior**: By default, the tool refuses the connection with `requests.exceptions.SSLError` and prints `"SSL verification failed for <url>. Set AGENTIC_INSECURE_SSL=1 to bypass (not recommended)."`. With `AGENTIC_INSECURE_SSL=1`, the connection proceeds but a prominent warning is logged.  
**Priority**: P0 — Security-sensitive default behavior.

---

### EC-12: Token Expiry During Batch Run
**Condition**: An OAuth access token expires 30 minutes into a 2-hour batch run.  
**Execution**: Configure OAuth with a token lifetime of 1,800 s. Run a batch job that takes ≥3,600 s.  
**Expected Behavior**: The tool's HTTP session adapter detects a 401 on the next request, uses the refresh token to obtain a new access token transparently, and retries the failed request. The batch log shows exactly one `[AUTH] Token refreshed` entry. No artifacts are lost.  
**Priority**: P0 — Long-running batch jobs are a primary use case.

---

## Execution

```bash
python3 -m pytest tests/edge/ -v \
    --timeout=900 \
    --run-slow \
    --junitxml=reports/edge_results.xml
```

## Pass Criteria
- **12 / 12** edge cases must pass.
- Slow tests (EC-02, EC-05, EC-12) may be skipped in pre-commit hooks but are mandatory in the nightly pipeline.

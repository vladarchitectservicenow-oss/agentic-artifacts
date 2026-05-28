Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# Risk Register — agentic-artifacts

## 1. Risk Matrix Overview

| Severity | Definition | Response SLA |
|----------|-----------|-------------|
| **P0** | Critical — causes data breach, system compromise, or complete service outage | Immediate (within 1 hour) |
| **P1** | High — causes significant degradation, blocks core functionality | Within 24 hours |
| **P2** | Medium — causes partial degradation, affects non-critical features | Within 1 sprint (2 weeks) |
| **P3** | Low — minor inconvenience, cosmetic issues, edge cases | Backlog prioritization |

---

## 2. Risk Catalog

### R01 — GitHub Token Exposure in Logs
- **Severity**: P0 (Critical)
- **Likelihood**: Medium
- **Impact**: Critical — full repository access compromise
- **Description**: PAT or installation token accidentally written to log output via unredacted exception tracebacks or debug-level logging.
- **Mitigation**:
  1. Centralized log redaction middleware strips `Authorization` and `X-GitHub-Token` headers before write.
  2. `SecretStr` type from pydantic ensures tokens never appear in `repr()` or `str()` output.
  3. Pre-commit hook scans for token patterns (`ghp_`, `github_pat_`).
  4. Log level enforced to `INFO` minimum in production; `DEBUG` requires explicit opt-in with redaction warning.
  5. Token rotation policy: PATs expire after 90 days maximum.

### R02 — ServiceNow Instance Credential Leak
- **Severity**: P0 (Critical)
- **Likelihood**: Low
- **Impact**: Critical — unauthorized ServiceNow instance access with read/write capability
- **Description**: Basic auth credentials or OAuth client secret leaked via `.env` file misconfiguration, backup inclusion, or terminal history.
- **Mitigation**:
  1. `.env` file permissions enforced to `0600`; startup validates and refuses to run otherwise.
  2. `.env` added to `.gitignore` and `.dockerignore` by default.
  3. `python-dotenv` loads credentials only into process memory, never serialized.
  4. Integration user has minimum required ACLs (read-only on config tables, write-only on log tables).
  5. Credential rotation enforced every 180 days with audit trail.
  6. OAuth 2.0 preferred over Basic Auth; client secret stored in OS keyring where available.

### R03 — GitHub API Rate Limit Exhaustion (5,000/hr)
- **Severity**: P1 (High)
- **Likelihood**: High
- **Impact**: High — batch jobs fail partway through, leaving partial results
- **Description**: Batch processing of many repositories exhausts the authenticated rate limit, causing `403 Rate Limit Exceeded` errors mid-batch.
- **Mitigation**:
  1. Conditional requests using `ETag` and `If-None-Match` headers; unchanged resources return `304 Not Modified` without consuming rate quota.
  2. Rate limit remaining tracked via `X-RateLimit-Remaining` response header; proactive throttling when below 10%.
  3. Exponential backoff with jitter via `tenacity` library (1s → 2s → 4s → 8s → max 60s).
  4. Batch processing with adaptive concurrency: reduce parallelism when approaching limits.
  5. GitHub App installation tokens provide higher limits (15,000/hr) for organization deployments.
  6. Local cache of repository metadata (TTL: 1 hour) avoids redundant API calls within a batch window.

### R04 — ServiceNow API Rate Throttling
- **Severity**: P1 (High)
- **Likelihood**: Medium
- **Impact**: High — generation pipeline stalls; sequential gate dependencies amplify delays
- **Description**: ServiceNow instances enforce concurrent request limits and per-user rate throttling; exceeding these causes `429 Too Many Requests` responses.
- **Mitigation**:
  1. Semaphore-based concurrency limiter (default: 5 concurrent requests to ServiceNow).
  2. `Retry-After` header parsing with automatic backoff.
  3. Batch API aggregation: combine multiple table queries into a single `/api/now/batch` request.
  4. Local SQLite cache of table records with configurable TTL (default: 5 minutes).
  5. Graceful degradation: continue generating other artifacts while waiting for throttled endpoints.

### R05 — Jinja2 Template Injection
- **Severity**: P1 (High)
- **Likelihood**: Low
- **Impact**: Critical — remote code execution on artifact generation host
- **Description**: Malicious template content stored in `x_agentic_artifacts_template.content` field could execute arbitrary Python code if template source is compromised.
- **Mitigation**:
  1. `jinja2.SandboxedEnvironment` is always used; no builtins, no imports, no attribute traversal on arbitrary objects.
  2. Template content validated via SHA-256 checksum in `x_agentic_artifacts_template.checksum`; mismatch aborts generation.
  3. Templates are stored in ServiceNow DB, not filesystem — no filesystem write path for template injection.
  4. Template upload requires `admin` role on ServiceNow; audit log captures all template modifications.
  5. Context variables passed to templates are limited to known-safe types: `str`, `int`, `bool`, `list`, `dict` (no callables, no class instances).
  6. Template syntax validation on load: any Jinja2 compile error rejects the template before use.

### R06 — Partial Artifact Generation on Timeout
- **Severity**: P2 (Medium)
- **Likelihood**: Medium
- **Impact**: Medium — artifacts appear valid but are missing sections; downstream consumers rely on bad data
- **Description**: Long-running API calls time out mid-generation, producing truncated artifact files that pass some gates but are structurally incomplete.
- **Mitigation**:
  1. Atomic file writes: artifact is written to a `.tmp` file first; only renamed to final name on successful completion of all gates.
  2. Per-phase timeouts with circuit breaker: 30s for SN fetch, 30s for GitHub fetch, 10s for template render.
  3. Incomplete artifacts flagged with `PARTIAL` status in log; `DONE.marker` is NOT created for partial runs.
  4. G6 (Data Integrity) gate validates that all expected sections contain data, not just headings.
  5. Resume capability: on re-run, engine detects existing `.tmp` files and starts fresh rather than appending.

### R07 — Git Merge Conflicts on Concurrent Runs
- **Severity**: P2 (Medium)
- **Likelihood**: Low
- **Impact**: Medium — one run's artifacts overwritten or conflict markers introduced into committed files
- **Description**: Two instances of agentic-artifacts running against the same output repository simultaneously could produce merge conflicts in git.
- **Mitigation**:
  1. Filesystem-level locking via `fcntl.flock` on a `.agentic-artifacts.lock` file in the output directory.
  2. Lock acquisition timeout: 5 seconds; if lock is held, the second process exits with a clear error message.
  3. Per-run output subdirectories (`/memory/checkpoints/{run_id}/`) prevent filename collisions.
  4. Git operations use `--ff-only` merge strategy to fail fast on divergence.
  5. CI/CD integration: single worker ensures serial execution.
  6. Run ID embedded in commit message for traceability.

### R08 — Large Repository Timeouts
- **Severity**: P2 (Medium)
- **Likelihood**: Medium
- **Impact**: Medium — dependency report missing files, architecture summary incomplete
- **Description**: Repositories with very large file trees (>10,000 files) or deep directory hierarchies cause GitHub API tree queries to time out.
- **Mitigation**:
  1. GitHub `git/trees` API with `recursive=1` limited to repositories under 100,000 entries.
  2. Progressive loading: fetch top-level tree first, then expand subdirectories on demand.
  3. File glob filtering: only fetch files matching relevant patterns (`*.py`, `*.js`, `*.ts`, `package.json`, etc.).
  4. Timeout budget: 25 seconds for tree fetch; if exceeded, generate artifact with `PARTIAL` status.

### R09 — Unicode Corruption in Field Names
- **Severity**: P3 (Low)
- **Likelihood**: Low
- **Impact**: Low — artifact readability degraded; gate G7 (Formatting Consistency) may fail
- **Description**: ServiceNow field names or values containing non-ASCII characters (Cyrillic, CJK, emoji) could be corrupted during transport or Markdown rendering.
- **Mitigation**:
  1. All HTTP requests set `Accept-Charset: utf-8` and `Content-Type: application/json; charset=utf-8`.
  2. Response encoding verified via `response.apparent_encoding` fallback.
  3. Jinja2 templates render with `ensure_ascii=False`.
  4. Output files written with `encoding='utf-8'`, BOM omitted.
  5. G7 gate includes Unicode normalization check (NFC form).
  6. Test suite includes Unicode stress tests: 中文, Русский, émojis 🚀.

### R10 — Python Version Incompatibility
- **Severity**: P3 (Low)
- **Likelihood**: Low
- **Impact**: Medium — tool fails to start, blocking all artifact generation
- **Description**: Dependency or syntax features used that are unavailable in the minimum supported Python version (3.10), causing `SyntaxError` or `ImportError`.
- **Mitigation**:
  1. `python_requires='>=3.10'` in `pyproject.toml`.
  2. CI matrix tests across Python 3.10, 3.11, 3.12 on every push.
  3. Pre-commit hook: `vermin` checks minimum Python version compatibility.
  4. `__init__.py` version guard: `if sys.version_info < (3, 10): raise SystemExit(...)`.
  5. No use of 3.11+ features without fallback imports.
  6. `pip install` validates Python version before proceeding.

### R11 — GitHub Webhook Race Condition
- **Severity**: P3 (Low)
- **Likelihood**: Low
- **Impact**: Low — generation uses stale data (previous commit)
- **Description**: If artifact generation is triggered by a GitHub webhook (push event), a race condition could occur where the webhook fires before the pushed commit is fully replicated.
- **Mitigation**:
  1. GitHub webhook processing includes a 5-second delay before API queries.
  2. Explicit commit SHA passed to the tool rather than relying on `HEAD` / default branch.
  3. API response includes `X-GitHub-Request-Id` for traceability.
  4. Retry with exponential backoff if commit SHA is not found on first attempt.

### R12 — Disk Space Exhaustion
- **Severity**: P3 (Low)
- **Likelihood**: Low
- **Impact**: Medium — batch job fails mid-execution; cleanup required
- **Description**: Batch processing 100+ large repositories could fill the output filesystem, causing write failures and partial artifact sets.
- **Mitigation**:
  1. Pre-flight disk space check: abort if available space < 500MB.
  2. Artifact size estimation: typical artifact is 10–20KB; 100 repos × 5 artifacts = ~10MB.
  3. Automatic cleanup of `.tmp` files from failed runs on startup.
  4. Configurable output rotation: keep last N runs, archive older ones.
  5. Alert threshold: warn at 80% disk usage on the output volume.

---

## 3. Risk Scoring Summary

| Count | Severity |
|-------|----------|
| 2 | P0 — Critical |
| 3 | P1 — High |
| 3 | P2 — Medium |
| 4 | P3 — Low |
| **12** | **Total Risks Identified** |

---

## 4. Residual Risk Statement

After applying all mitigation strategies, the residual risk profile is:

**P0 Residual**: Reduced from Medium→Low for token exposure (R01) and credential leak (R02) through layered controls including central redaction, `SecretStr` types, and minimum-privilege accounts. No single point of failure remains; compromise requires simultaneous failure of multiple controls.

**P1 Residual**: Rate limits (R03, R04) mitigated to acceptable levels with conditional requests and caching, but remain P1 for extreme batch sizes (>500 repos/hour). Template injection (R05) reduced to Low via `SandboxedEnvironment` and checksum verification — attacker would need both ServiceNow admin access and the ability to forge SHA-256 checksums.

**P2 Residual**: Timeout/partial artifacts (R06) and concurrency conflicts (R07) are mitigated but require operational discipline (CI/CD single-worker enforcement). Large repo handling (R08) is bounded by GitHub API limits; repos exceeding GitHub's tree size limits will always produce PARTIAL results.

**P3 Residual**: Unicode (R09) and Python compatibility (R10) are well-covered by CI testing; only novel edge cases pose residual risk. Webhook races (R11) and disk space (R12) are operationally manageable with standard monitoring.

**Overall Assessment**: The tool is suitable for production use with the mitigations in place. Continuous monitoring of API rate limits, credential hygiene, and disk space is recommended. Quarterly risk review cadence advised.

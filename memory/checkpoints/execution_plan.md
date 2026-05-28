Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# Execution Plan — agentic-artifacts

## Overview

This execution plan defines the end-to-end process for deploying, configuring, and running agentic-artifacts against a ServiceNow instance with GitHub integration. Each phase includes preconditions, execution steps, validation criteria, and rollback procedures. Successful completion of all eight phases results in a `DONE.marker` file in the output directory.

**Target Environment**
- OS: Linux (Ubuntu 22.04 LTS recommended) or macOS 14+
- Python: 3.10–3.12
- ServiceNow: Utah Patch 3+ (Washington DC recommended)
- GitHub: Organization or personal account with PAT

---

## Phase 1: Environment Setup

### Objective
Ensure the runtime environment has the correct Python version, all dependencies installed, and configuration files in place.

### Preconditions
- Linux or macOS host with shell access
- Python 3.10+ available on PATH
- pip 23.0+ available
- Git 2.39+ installed

### Execution Steps
```bash
# 1.1 Verify Python version
python3 --version  # Must be >= 3.10

# 1.2 Create virtual environment
python3 -m venv .venv
source .venv/bin/activate

# 1.3 Install dependencies
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt

# 1.4 Create .env configuration
cat > .env << 'ENVEOF'
SNOW_INSTANCE_URL=https://your-instance.service-now.com
SNOW_USERNAME=svc_agentic_artifacts
SNOW_PASSWORD=***REDACTED***
SNOW_OAUTH_CLIENT_ID=
SNOW_OAUTH_CLIENT_SECRET=
GITHUB_TOKEN=ghp_***
GITHUB_ORG=your-org
OUTPUT_DIR=./memory/checkpoints
MAX_WORKERS=10
LOG_LEVEL=INFO
ENVEOF

# 1.5 Set restrictive file permissions
chmod 600 .env
```

### Validation
- [ ] `python3 --version` returns 3.10 or higher
- [ ] `pip list | grep -E "requests|jinja2|pyyaml|pydantic|rich"` shows all core packages
- [ ] `.env` file exists with 0600 permissions
- [ ] Virtual environment is activated

### Rollback
```bash
rm -rf .venv .env
# Re-clone repository if needed
```

---

## Phase 2: Instance Connectivity

### Objective
Verify that the tool can authenticate against both ServiceNow and GitHub, and that required scoped tables exist.

### Preconditions
- Phase 1 completed successfully
- `.env` populated with valid credentials
- Network access to `*.service-now.com` (TCP 443) and `api.github.com` (TCP 443)

### Execution Steps
```bash
# 2.1 Test ServiceNow connectivity
python -m agentic_artifacts test-connection snow

# Expected output:
# ✅ ServiceNow: Connected to {instance_name} ({release})
# ✅ Tables found: x_agentic_artifacts_config, x_agentic_artifacts_log,
#    x_agentic_artifacts_template, x_agentic_artifacts_quality_result

# 2.2 Test GitHub connectivity
python -m agentic_artifacts test-connection github

# Expected output:
# ✅ GitHub: Authenticated as {username}
# ✅ Rate limit: {remaining}/5000 remaining, resets at {time}
# ✅ Organization access: {org_name}

# 2.3 Validate scoped tables exist
python -m agentic_artifacts validate-tables
```

### Validation
- [ ] ServiceNow connection succeeds with 200 OK
- [ ] All four `x_agentic_artifacts_*` tables are accessible
- [ ] GitHub authentication succeeds
- [ ] Rate limit remaining > 1000
- [ ] Organization name matches `.env` configuration

### Rollback
```bash
# Clear any cached credentials
rm -rf ~/.agentic_artifacts/cache/
# Verify .env credentials are correct
python -m agentic_artifacts test-connection snow --verbose
```

---

## Phase 3: Template Rendering

### Objective
Validate that Jinja2 templates compile successfully and produce valid Markdown output for all five artifact types.

### Preconditions
- Phase 2 completed
- Template records exist in `x_agentic_artifacts_template` (or default bundled templates used)

### Execution Steps
```bash
# 3.1 List available templates
python -m agentic_artifacts templates list

# 3.2 Validate each template compiles
python -m agentic_artifacts templates validate --all

# Expected output:
# ✅ architecture_summary   v1.0.0  Compiled OK  Checksum: a1b2c3...
# ✅ dependency_report       v1.0.0  Compiled OK  Checksum: d4e5f6...
# ✅ risk_report             v1.0.0  Compiled OK  Checksum: g7h8i9...
# ✅ execution_plan          v1.0.0  Compiled OK  Checksum: j0k1l2...
# ✅ test_suite              v1.0.0  Compiled OK  Checksum: m3n4o5...

# 3.3 Dry-run rendering with sample context
python -m agentic_artifacts templates render --type architecture_summary --sample-data --dry-run
```

### Validation
- [ ] All five templates compile without Jinja2 errors
- [ ] SHA-256 checksums match expected values
- [ ] Dry-run output is valid Markdown (parsable by `markdown` library)
- [ ] No undefined variable errors in template rendering

### Rollback
```bash
# Revert to default bundled templates if custom templates fail
python -m agentic_artifacts templates reset --all
```

---

## Phase 4: Artifact Generation (Single Repository)

### Objective
Generate all five artifact types for a single test repository and verify basic file output.

### Preconditions
- Phase 3 completed
- At least one GitHub repository accessible with the configured token
- Repository has a `README.md` in the default branch

### Execution Steps
```bash
# 4.1 Generate artifacts for a single repository
python -m agentic_artifacts generate \
  --repo your-org/test-repo \
  --artifacts all \
  --output ./memory/checkpoints/

# Expected output:
# 🚀 Generating artifacts for your-org/test-repo
#   [1/5] Architecture summary... ✅ (1.2s)
#   [2/5] Dependency report...   ✅ (0.8s)
#   [3/5] Risk register...       ✅ (1.5s)
#   [4/5] Execution plan...      ✅ (0.9s)
#   [5/5] Test suite...          ✅ (1.1s)
# 🎉 Generation complete: 5/5 artifacts created

# 4.2 Verify output files exist
ls -la memory/checkpoints/
# Expected: architecture_summary.md, dependency_report.md,
#           risk_report.md, execution_plan.md, test_suite.md

# 4.3 Check file sizes (non-empty)
find memory/checkpoints/ -name "*.md" -size +0 | wc -l
# Expected: 5
```

### Validation
- [ ] All 5 artifacts generated without errors
- [ ] Each file is non-empty (> 0 bytes)
- [ ] Files contain the copyright header
- [ ] Console output shows total generation time < 30 seconds

### Rollback
```bash
rm -f memory/checkpoints/*.md memory/checkpoints/DONE.marker
# Fix configuration and re-run
```

---

## Phase 5: Quality Gate Validation

### Objective
Execute all eight quality gates (G0–G8) against the generated artifacts and verify they pass.

### Preconditions
- Phase 4 completed with artifacts on disk
- Quality gate thresholds configured (default: all 8 gates)

### Execution Steps
```bash
# 5.1 Run quality gates on all generated artifacts
python -m agentic_artifacts quality-check \
  --directory ./memory/checkpoints/ \
  --artifacts all

# Expected output:
# 🔍 Running quality gates on 5 artifacts
#
# architecture_summary.md:
#   ✅ G0: File exists
#   ✅ G1: Copyright header
#   ✅ G2: Valid Markdown
#   ✅ G3: Min line count (152 lines ≥ 150)
#   ✅ G4: All sections present
#   ✅ G5: Schema conformant
#   ✅ G6: Data integrity verified
#   ✅ G7: Formatting consistent
#   ✅ G8: Git-ready
#   Score: 8/8 PASSED
#
# [... repeated for each artifact ...]
#
# 📊 Summary: 40/40 gates passed (100%)

# 5.2 Inspect specific gate details
python -m agentic_artifacts quality-check \
  --artifact memory/checkpoints/architecture_summary.md \
  --verbose
```

### Validation
- [ ] All 5 artifacts pass all 8 gates (40/40 total)
- [ ] G6 validates no broken cross-references
- [ ] G8 confirms no trailing whitespace or merge conflict markers
- [ ] `x_agentic_artifacts_quality_result` table populated with results

### Rollback
```bash
# If gates fail, review remediation hints and re-generate
python -m agentic_artifacts quality-check --artifact {file} --verbose
# Apply suggested fixes to templates or source data, then re-run Phase 4
```

---

## Phase 6: Batch Processing (50-Repository Stress Test)

### Objective
Validate that the tool scales to process 50 repositories in parallel within the 10-minute performance target.

### Preconditions
- Phase 5 completed (gates pass for single repo)
- List of 50 accessible GitHub repositories available
- Sufficient disk space (> 500MB free)
- GitHub rate limit > 1000 remaining

### Execution Steps
```bash
# 6.1 Create repository list file
cat > repos.txt << 'EOF'
your-org/repo-01
your-org/repo-02
...
your-org/repo-50
EOF

# 6.2 Run batch processing
time python -m agentic_artifacts batch \
  --repos repos.txt \
  --artifacts all \
  --max-workers 10 \
  --output ./memory/checkpoints/

# Expected output:
# 🔄 Batch processing 50 repositories with 10 workers
# Progress: [████████████████████] 50/50 | ✅ 48  ❌ 2
# ⏱️  Elapsed: 7m 42s
# 📊 Success rate: 96.0%

# 6.3 Verify batch results
python -m agentic_artifacts batch-report --run-id {run_id}

# 6.4 Check per-repo output directories
ls memory/checkpoints/ | wc -l
# Expected: 50 run directories + aggregate report
```

### Validation
- [ ] Total elapsed time < 10 minutes
- [ ] Success rate ≥ 90% (45/50 repos minimum)
- [ ] Failed repos have clear error messages in logs
- [ ] All successful repos pass quality gates G0–G8
- [ ] No rate limit exhaustion errors
- [ ] Memory usage < 500MB peak (verified via `htop` or `/usr/bin/time -v`)

### Rollback
```bash
# Remove failed run artifacts
rm -rf memory/checkpoints/{failed_run_ids}/
# Adjust max_workers and re-run with smaller batches
python -m agentic_artifacts batch --repos repos.txt --max-workers 5
```

---

## Phase 7: Git Commit + Push

### Objective
Commit generated artifacts to the output repository and push to the remote origin.

### Preconditions
- Phase 6 completed (or Phase 4 for single-repo workflows)
- Git repository initialized in the output directory
- Remote origin configured
- SSH deploy key or PAT with `repo` scope for push access

### Execution Steps
```bash
# 7.1 Navigate to output directory
cd memory/checkpoints/

# 7.2 Verify git status
git status

# 7.3 Stage generated artifacts
git add *.md DONE.marker */ 2>/dev/null || true

# 7.4 Commit with auto-generated message
RUN_ID=$(python -c "import uuid; print(uuid.uuid4())")
git commit -m "agentic-artifacts: run ${RUN_ID} — batch generation for 50 repos

- Architecture summaries: 50
- Dependency reports: 50
- Risk registers: 50
- Execution plans: 50
- Test suites: 50
- Quality gates: 2500/2500 passed (50 × 5 × 10 gates per run)
- Generated by: agentic-artifacts v1.0.0"

# 7.5 Push to remote
git push origin main

# 7.6 Verify push via GitHub API
python -m agentic_artifacts verify-push --commit $(git rev-parse HEAD)
# Expected: ✅ Commit {sha} confirmed on origin/main
```

### Validation
- [ ] `git push` succeeds with exit code 0
- [ ] Commit appears in GitHub web UI
- [ ] All artifact files are present on the remote
- [ ] Commit message includes run ID for traceability
- [ ] No merge conflicts or rejected pushes

### Rollback
```bash
# If push fails, reset to remote state
git fetch origin
git reset --hard origin/main
# Re-generate and try again
cd -
python -m agentic_artifacts batch --repos repos.txt --resume
```

---

## Phase 8: Completion Marker

### Objective
Create the `DONE.marker` file to signal successful completion and enable downstream automation triggers.

### Preconditions
- Phases 1–7 all completed successfully
- Quality gates pass for all artifacts
- Git push confirmed

### Execution Steps
```bash
# 8.1 Create DONE.marker with metadata
cat > memory/checkpoints/DONE.marker << 'MARKEREOF'
# agentic-artifacts Run Complete
timestamp: $(date -u +"%Y-%m-%dT%H:%M:%SZ")
run_id: ${RUN_ID}
status: SUCCESS
artifacts_generated: 250
gates_passed: 2500
repos_processed: 50
elapsed_seconds: 462
version: 1.0.0
MARKEREOF

# 8.2 Atomic write with fsync
python -c "
import os
path = 'memory/checkpoints/DONE.marker'
fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
os.write(fd, open(path).read().encode())
os.fsync(fd)
os.close(fd)
"

# 8.3 Commit and push the marker
cd memory/checkpoints/
git add DONE.marker
git commit -m "agentic-artifacts: completion marker for run ${RUN_ID}"
git push origin main
```

### Validation
- [ ] `DONE.marker` file exists and is non-empty
- [ ] File contains all required metadata fields
- [ ] Marker is committed and pushed to remote
- [ ] Timestamp is within expected window
- [ ] `x_agentic_artifacts_log` shows `SUCCESS` status for the run

### Rollback
```bash
# If DONE.marker creation fails, check disk space and permissions
df -h memory/checkpoints/
ls -la memory/checkpoints/
# Fix issues and re-run Phase 8
```

---

## End-to-End Execution Summary

```
Phase 1: Environment Setup  ──▶  ✅  (2 min)
Phase 2: Connectivity Test  ──▶  ✅  (1 min)
Phase 3: Template Compile    ──▶  ✅  (30 sec)
Phase 4: Single Repo Gen     ──▶  ✅  (30 sec)
Phase 5: Quality Gates       ──▶  ✅  (10 sec)
Phase 6: Batch 50 Repos      ──▶  ✅  (8 min target)
Phase 7: Git Commit + Push   ──▶  ✅  (30 sec)
Phase 8: DONE.marker         ──▶  ✅  (5 sec)
─────────────────────────────────────────
Total Estimated Time:            ~12 minutes
```

---

## Troubleshooting Quick Reference

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| `ConnectionError` to ServiceNow | VPN down, IP not allowlisted | Check VPN, verify instance URL |
| `401 Unauthorized` | Expired/incorrect credentials | Rotate credentials, update `.env` |
| `403 Rate Limit Exceeded` (GitHub) | Batch too large | Reduce `max_workers`, wait for reset |
| `404 Table Not Found` | Scoped app not installed on instance | Install `x_agentic_artifacts` scoped app |
| `Jinja2 TemplateSyntaxError` | Malformed template in DB | Validate and repair templates (Phase 3) |
| `G3: Min line count FAILED` | Insufficient source data | Check SN records, re-fetch data |
| `git push rejected` | Remote has newer commits | `git pull --rebase` then push |
| `MemoryError` | Batch too large for available RAM | Reduce `max_workers` or increase swap |

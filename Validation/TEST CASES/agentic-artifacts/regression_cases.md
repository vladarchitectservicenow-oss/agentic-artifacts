Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# agentic-artifacts – Regression Test Cases

## Overview
Regression tests ensure that previously working functionality remains intact across code changes. These cases are executed on every pull request targeting the `main` branch and as part of the nightly CI pipeline.

---

### RC-01: Idempotent Execution – Same Input, Same Output
**Objective**: Running the artifact generator twice with identical inputs must produce bit-for-bit identical output files.

**Steps**:
1. Seed a known config file and template directory.
2. Run `agentic_artifacts generate --all --output-dir /tmp/run1`.
3. Run the same command again with `--output-dir /tmp/run2`.
4. Compute SHA-256 checksums of every file under both directories.

**Expectation**: `diff -r /tmp/run1 /tmp/run2` returns exit code 0. Timestamps embedded in generated headers (if any) must use the source template modification time, not wall-clock time, to guarantee determinism.

---

### RC-02: Format Consistency Across Runs
**Objective**: The structural format of generated artifacts (indentation, line endings, trailing whitespace) must not drift between runs.

**Steps**:
1. Generate a full artifact suite on commit `abc123`.
2. Checkout commit `def456` (a refactor that claims no behavioral changes) and generate again.
3. Run `diff -w -B` between the two output directories.

**Expectation**: Only whitespace-insignificant diffs (or none). If `pre-commit` hooks reformat templates, the reformatting must produce identical output to the human-authored form.

---

### RC-03: Config Persistence After Restart
**Objective**: Configuration values written during one session must survive a process restart and be loaded correctly.

**Steps**:
1. Run `agentic_artifacts config set output_format json --persist`.
2. Kill the process (`SIGTERM`).
3. Restart and run `agentic_artifacts config get output_format`.

**Expectation**: The persisted value `json` is returned. No fallback to the default (`yaml`).

---

### RC-04: Template Versioning Compatibility
**Objective**: Artifacts generated from a template tagged `v1.2.0` must remain parseable by the tool's `v1.3.0` parser (forward compatibility), and templates stamped `v2.0.0` must be rejected with a clear "unsupported template version" error when the tool is at `v1.x`.

**Steps**:
1. Checkout template repo at tag `v1.2.0`.
2. Generate artifacts with tool version `v1.3.0`.
3. Run `agentic_artifacts validate` on the output — all gates pass.
4. Bump a single template's `version` field to `2.0.0`.
5. Retry generation.

**Expectation**: Step 3 passes; step 5 exits non-zero with stderr containing `"Unsupported template version: 2.0.0"`.

---

### RC-05: Git State Preservation – Clean Working Tree
**Objective**: After a full artifact generation run, the local Git repository must have zero modified or untracked files (output directory is `.gitignore`d).

**Steps**:
1. Run `agentic_artifacts generate --all`.
2. Execute `git status --porcelain`.

**Expectation**: Output is empty. Even `.pyc` files and `__pycache__/` directories must be covered by `.gitignore`.

---

### RC-06: Rate Limit Recovery Behavior
**Objective**: When the ServiceNow REST API rate-limits the tool (HTTP 429), the exponential-backoff retry loop must eventually succeed without human intervention.

**Steps**:
1. Configure a mock SN instance that returns 429 for the first 4 requests, then 200.
2. Run `agentic_artifacts generate --type client_script --name "RateLimitTest"`.

**Expectation**: Log output shows 4 retry attempts with increasing delays (1 s, 2 s, 4 s, 8 s), followed by a successful artifact write. Total wall-clock time is ≤20 s.

---

### RC-07: Partial Failure Recovery – Resume from Checkpoint
**Objective**: If a batch run processing 100 templates crashes after completing 47, a subsequent run must skip those 47 and resume at template 48.

**Steps**:
1. Initiate `agentic_artifacts generate --batch templates_100.txt --checkpoint`.
2. Artificially kill the process after it logs `[PROGRESS] 47/100`.
3. Inspect the checkpoint file (`~/.agentic_artifacts/checkpoint.json`).
4. Re-run the same command.

**Expectation**: The log shows `[RESUME] Skipping 47 completed templates`. No duplicate artifacts. The checkpoint file is deleted on successful completion.

---

### RC-08: Multi-Instance Isolation – Separate Runs Don't Collide
**Objective**: Two independent invocations of the tool, pointed at different output directories, must not interfere with each other's temporary files, lock files, or cached data.

**Steps**:
1. Open two terminals. In terminal A, run against `--output-dir /tmp/outA`.
2. Simultaneously (within 2 s), in terminal B, run against `--output-dir /tmp/outB`.
3. Wait for both to finish.

**Expectation**: Both exit 0. No file-not-found errors attributable to a temp file being deleted by the other process. The combined set of output files equals the union of what each run would produce in isolation.

---

### RC-09: License Header Integrity After Regeneration
**Objective**: Running artifact generation on an already-populated output directory must not strip, corrupt, or duplicate the AGPL-3.0 license header.

**Steps**:
1. Generate output once.
2. Verify every `.py` and `.js` file starts with the copyright block exactly once.
3. Run generation again into the same output directory (`--force`).
4. Re-verify.

**Expectation**: Step 2 and step 4 produce identical results. No double headers, no truncation.

---

### RC-10: README Section Deduplication
**Objective**: The generated `README.md` must never contain duplicated sections (e.g., two "Installation" headings) even when multiple input templates contribute overlapping content.

**Steps**:
1. Prepare three templates, each containing a `## Installation` section with slightly different text.
2. Generate a single README from the merged template set.

**Expectation**: Exactly one `## Installation` heading appears in the output. The tool must either select the most specific section or concatenate unique content under a single heading — never duplicate the heading itself.

---

## Execution

```bash
python3 -m pytest tests/regression/ -v \
    --timeout=600 \
    --junitxml=reports/regression_results.xml
```

## Pass Criteria
- **10 / 10** regression cases must pass before merging to `main`.
- Any regression failure triggers an automatic Slack notification to `#platform-tools-alerts`.

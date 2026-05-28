Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# agentic-artifacts – Test Suite SOP (Standard Operating Procedure)

## 1. Purpose
This SOP defines the system-level validation test suite for the **agentic-artifacts** Python CLI tool, which auto-generates ServiceNow platform artifacts (Business Rules, Client Scripts, UI Policies, etc.) from structured input templates. The suite ensures functional correctness, resilience under failure, and compliance with quality gates before every release.

## 2. Test Environment Setup

```bash
# 1. Create and activate a Python virtual environment
python3 -m venv /tmp/agentic_artifacts_venv
source /tmp/agentic_artifacts_venv/bin/activate

# 2. Install runtime dependencies
pip install --upgrade pip setuptools wheel
pip install requests jinja2 pyyaml python-dotenv pytest pytest-cov pytest-xdist pytest-timeout

# 3. Install the CLI tool in development mode
cd /tmp/mass_val/agentic-artifacts
pip install -e .

# 4. Configure environment variables
export SERVICENOW_INSTANCE="https://dev123456.service-now.com"
export SERVICENOW_USERNAME="test_admin"
export SERVICENOW_PASSWORD="${SN_PASSWORD}"         # from secrets manager
export GITHUB_TOKEN="${GH_PAT}"                     # read-only PAT for template repos
export AGENTIC_CONFIG_DIR="/tmp/agentic_test_config"

# 5. Seed test configuration
mkdir -p "${AGENTIC_CONFIG_DIR}"
cp config/test_config.yaml "${AGENTIC_CONFIG_DIR}/config.yaml"

# 6. Validate connectivity
python3 -m agentic_artifacts health-check
```

## 3. Scenario Table

| ID      | Scenario                        | Priority | Description                                                                 | Expected Result                                                       | Execution Steps                                                                 |
|---------|---------------------------------|----------|-----------------------------------------------------------------------------|-----------------------------------------------------------------------|---------------------------------------------------------------------------------|
| TC-001  | plugin_active                   | P0       | Verify the CLI plugin registers correctly under the ServiceNow application scope | `agentic_artifacts --version` returns version ≥2.0.0; no import errors | Run `agentic_artifacts --version`; check exit code 0 and stdout matches semver  |
| TC-002  | config_load                     | P0       | Confirm that configuration YAML loads without schema violations             | All required keys present; defaults applied to optional fields          | Parse config; assert `instance_url`, `auth`, `templates_dir` are non-null       |
| TC-003  | rest_api_connect                | P0       | Validate that a REST API call to the ServiceNow Table API succeeds          | HTTP 200 on `GET /api/now/table/sys_user?sysparm_limit=1`               | Issue authenticated GET; assert status 200 and response contains `result` array |
| TC-004  | artifact_generate               | P0       | Generate a single Business Rule artifact from a valid Jinja2 template       | Output file written with correct sys_class_name and script content      | Run `agentic_artifacts generate --type business_rule --name "TestBR"`; verify    |
| TC-005  | batch_50repos                   | P1       | Process 50 GitHub template repositories in a single batch run               | All 50 repos processed; ≤2 transient retries; zero fatal errors         | Invoke batch mode with `--repos-list tests/fixtures/50_repos.txt`; monitor logs  |
| TC-006  | quality_gate_validation         | P0       | Assert that generated artifacts pass all G0–G8 quality gates                | 100 % gate pass rate; no gate-violation warnings                       | Run `agentic_artifacts validate --gates all` on output directory; parse report   |
| TC-007  | empty_instance                  | P1       | Run against a blank ServiceNow developer instance (zero custom tables)      | Tool exits gracefully with warning; no traceback                         | Point `SERVICENOW_INSTANCE` to a fresh dev instance; run full generate pipeline  |
| TC-008  | auth_failure                    | P0       | Simulate invalid credentials and confirm the tool surfaces a clear error    | Non-zero exit code; stderr contains "Authentication failed"              | Set invalid password; run any API-dependent command; assert exit ≠ 0            |
| TC-009  | rate_limit_handling             | P1       | Exceed the ServiceNow REST API rate limit and verify exponential backoff    | 429 responses retried up to 5 times; eventual success or clean failure   | Flood API with concurrent requests; observe log for `[RETRY]` and `backoff`      |
| TC-010  | timeout_recovery                | P1       | Induce a socket timeout mid-request and confirm retry logic                 | Request retried after timeout; no orphaned connections                   | Use `iptables` to drop traffic for 5 s; run artifact generate; restore; verify   |
| TC-011  | unicode_fields                  | P2       | Generate artifacts with Unicode/emoji in field labels and descriptions      | UTF-8 output preserved; no `UnicodeEncodeError`                          | Pass template containing "🔥 Urgent Escalation 🚨"; inspect output file          |
| TC-012  | concurrent_execution            | P1       | Run two `agentic_artifacts generate` processes simultaneously               | Both complete without file-lock deadlocks; outputs do not interleave     | Launch two processes with distinct output dirs; wait; compare checksums          |
| TC-013  | license_header_validation       | P0       | Every generated `.py` / `.js` file must contain the AGPL-3.0 header         | `grep -rL "Copyright (C) 2026 Vladimir Kapustin" output/` returns empty  | Recursive grep on output directory; assert zero misses                          |
| TC-014  | readme_word_count_check         | P2       | Generated README.md files must exceed 200 words                             | `wc -w < readme_file` ≥ 200                                              | Generate README; pipe through `wc -w`; assert threshold met                      |

## 4. Execution Command

```bash
# Run the full validation suite with verbose output and JUnit XML report
python3 -m pytest tests/ -v \
    --tb=short \
    --timeout=300 \
    --junitxml=reports/test_results.xml \
    --cov=agentic_artifacts \
    --cov-report=term-missing
```

## 5. Pass Criteria

- **Minimum pass rate**: 14 out of 14 scenarios must return `PASS`.
- Any `P0` failure blocks the release pipeline.
- `P1` failures require a written justification and approval from the QA lead.
- `P2` failures are documented in the release notes but do not block the release.

## 6. Failure Escalation

1. Capture full console output and the JUnit XML report.
2. Attach the relevant log file (default: `logs/agentic_artifacts.log`).
3. Open a defect in the issue tracker with label `phase2-validation`.
4. Notify the `#platform-tools` Slack channel with the defect ID.

## Revision History

| Date       | Version | Author           | Change Description                       |
|------------|---------|------------------|------------------------------------------|
| 2026-05-27 | 2.0     | Vladimir Kapustin | Phase 2 rebuild – 14 scenarios, 50+ loc |

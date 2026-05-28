Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# agentic-artifacts – Validation Checklist

## Instructions
This checklist must be completed and signed off before every release candidate is promoted to production. Check each box after the corresponding verification is performed.

---

## 1. Pre-Flight Checks

- [ ] **Python version**: `python3 --version` reports 3.10 or higher (tested matrix: 3.10, 3.11, 3.12).
- [ ] **pip packages**: `pip check` returns no broken dependencies. All pinned versions in `requirements.txt` are installable.
- [ ] **.env file**: All required environment variables are set and non-empty (`SERVICENOW_INSTANCE`, `SERVICENOW_USERNAME`, `SERVICENOW_PASSWORD`, `GITHUB_TOKEN`).
- [ ] **git config**: `git config user.name` and `git config user.email` are set. `git config commit.gpgSign` is `true` for signed releases.
- [ ] **Virtual environment**: Active venv is isolated (no system-site-packages leakage). `pip list --local` shows only project dependencies.
- [ ] **Disk space**: Output filesystem has ≥1 GB of free space. Check with `df -h <output_dir>`.

---

## 2. Connectivity Checks

- [ ] **ServiceNow REST API**: `curl -s -o /dev/null -w "%{http_code}" -u "${SERVICENOW_USERNAME}:${SERVICENOW_PASSWORD}" "${SERVICENOW_INSTANCE}/api/now/table/sys_user?sysparm_limit=1"` returns 200.
- [ ] **ServiceNow OAuth endpoint** (if applicable): Token endpoint reachable; refresh token grants a new access token.
- [ ] **GitHub API**: `curl -s -H "Authorization: token ${GITHUB_TOKEN}" https://api.github.com/rate_limit` returns 200 with `rate.remaining > 100`.
- [ ] **File system R/W**: `touch /tmp/agentic_write_test && rm /tmp/agentic_write_test` succeeds. Output directory is writable.
- [ ] **DNS resolution**: `nslookup` (or `dig`) resolves the SN instance hostname and `api.github.com`.

---

## 3. Quality Gate Checks (G0–G8)

- [ ] **G0 – Syntax Validity**: All generated `.py` files pass `python3 -m py_compile`. All `.js` files pass `node --check`.
- [ ] **G1 – License Header**: Every source file contains the AGPL-3.0 copyright header block. Verified via `grep -rL "Copyright (C) 2026 Vladimir Kapustin" output/`.
- [ ] **G2 – No Hardcoded Secrets**: Zero occurrences of passwords, tokens, or API keys in generated output. Scanned with `detect-secrets` or `gitleaks`.
- [ ] **G3 – Template Completeness**: Every required field in the template schema has a non-null, non-empty value in the generated artifact.
- [ ] **G4 – Cross-Reference Integrity**: Internal references (e.g., `sys_id` lookups, script include names) resolve correctly against the ServiceNow instance.
- [ ] **G5 – Naming Convention**: Artifact file names match the pattern `<type>_<name>.<ext>` (e.g., `business_rule_AutoCloseIncident.py`). No spaces or special characters.
- [ ] **G6 – README Quality**: Generated README.md has ≥200 words, a valid markdown structure, and no duplicate headings.
- [ ] **G7 – Changelog Accuracy**: `CHANGELOG.md` entries match the semantic version bump and contain links to closed issues/PRs.
- [ ] **G8 – Performance Budget**: Full artifact generation for the reference dataset (50 repos) completes in ≤10 minutes on CI hardware (4 vCPU, 16 GB RAM).

---

## 4. Output Checks

- [ ] **File existence**: `find output/ -type f | wc -l` matches the expected artifact count (≥1 per template input).
- [ ] **File sizes > threshold**: Zero files are 0 bytes. Minimum expected size is 256 bytes (license header + minimal content). Check with `find output/ -type f -size -256c`.
- [ ] **No corruption**: `file output/*` reports correct MIME types (`text/x-python`, `text/javascript`, `text/plain`). No binary garbage.
- [ ] **Encoding**: All files are UTF-8 encoded. `file -i output/* | grep -v 'charset=utf-8'` returns nothing.
- [ ] **Line endings**: All text files use LF (`\n`), not CRLF. `file output/* | grep 'CRLF'` returns nothing.

---

## 5. Git Checks

- [ ] **Clean staging area**: `git status --porcelain` returns empty (ignoring `.gitignore`d paths).
- [ ] **Conventional commit message**: The release commit message follows the Conventional Commits spec (e.g., `chore(release): v2.1.0`).
- [ ] **Signed commit**: `git log -1 --show-signature` shows a valid GPG signature from an authorized key.
- [ ] **Push verified**: `git push --dry-run` succeeds. Remote `origin` URL matches the canonical repository.
- [ ] **Tag exists**: `git tag -l "v*" --sort=-v:refname | head -1` shows the newly created version tag. Tag is GPG-signed.
- [ ] **No large files**: `git ls-files | xargs -I{} git cat-file -s {} | awk '$1 > 1048576'` returns nothing (no files > 1 MB tracked in git).

---

## 6. Final Sign-Off

- [ ] **Test suite green**: All 14 SOP scenarios, 10 regression cases, and 12 edge cases pass. JUnit XML reports are archived.
- [ ] **Security scan**: SAST tool (e.g., Bandit, Semgrep) reports zero high/critical findings.
- [ ] **Documentation updated**: `README.md`, `CONTRIBUTING.md`, and `CHANGELOG.md` reflect the current release.
- [ ] **Peer review**: At least one other engineer has reviewed and approved the release PR.
- [ ] **Rollback plan**: A documented rollback procedure exists and has been tested in the staging environment.

---

| Role            | Name | Signature | Date       |
|-----------------|------|-----------|------------|
| QA Engineer     |      |           |            |
| Release Manager |      |           |            |
| Security Lead   |      |           |            |

---

*This checklist is version-controlled alongside the codebase. The latest version is always at `Validation/TEST CASES/agentic-artifacts/validation_checklist.md`.*

Copyright (C) 2026 Vladimir Kapustin. Licensed under AGPL-3.0.

# Dependency Report — agentic-artifacts

## 1. ServiceNow Platform Dependencies

### Required Plugins
| Plugin ID | Name | Minimum Version | Purpose |
|-----------|------|----------------|---------|
| `com.glideapp.servicecatalog` | Service Catalog | v3.0+ | Template storage via catalog items |
| `com.snc.cmdb` | Configuration Management Database (CMDB) | v5.0+ | Application relationship data for dependency reports |
| `com.snc.change_management` | Change Management | v7.0+ | Change record extraction for risk registers |
| `com.snc.task_management` | Task Management | v4.0+ | Task and project metadata for execution plans |
| `com.glide.rest.api` | REST API Enabler | v2.0+ | Table REST API access (default on all instances) |
| `com.glide.oauth` | OAuth 2.0 Support | v1.5+ | OAuth authentication when Basic Auth is disabled |

### REST API Versions
| Endpoint | Version | Authentication | Pagination |
|----------|---------|---------------|------------|
| `/api/now/table/` | v2 (default) | Basic + OAuth | `sysparm_limit` + `sysparm_offset` |
| `/api/now/attachment/` | v2 | Basic + OAuth | Standard |
| `/api/now/batch` | v2 | Basic + OAuth | N/A (batch endpoint) |
| `/oauth_token.do` | v1 | Client Credentials | N/A |

### ServiceNow Release Compatibility
| Release | Version | Status | Notes |
|---------|---------|--------|-------|
| Utah | Utah Patch 3+ | ✅ Supported | Minimum required release |
| Vancouver | All patches | ✅ Supported | Recommended for OAuth 2.0 enhancements |
| Washington DC | All patches | ✅ Supported | Best performance (API throughput improvements) |
| Xanadu | TBD | 🔄 Planned | Pending validation |
| Australia | TBD | 🔄 Planned | Pending validation |

---

## 2. Python Runtime Dependencies

### Core Dependencies (requirements.txt)
| Package | Version | Purpose | License |
|---------|---------|---------|---------|
| `requests` | ≥2.31.0, <3.0 | HTTP client for ServiceNow REST API and GitHub API | Apache 2.0 |
| `jinja2` | ≥3.1.2, <4.0 | Template rendering engine for artifact generation | BSD-3-Clause |
| `pyyaml` | ≥6.0, <7.0 | YAML configuration parsing for `.env.yaml` and pipeline config | MIT |
| `pydantic` | ≥2.4.0, <3.0 | Data validation and settings management via model classes | MIT |
| `rich` | ≥13.5.0, <14.0 | Terminal UI: progress bars, tables, syntax highlighting | MIT |
| `tenacity` | ≥8.2.0, <9.0 | Retry logic with exponential backoff for API calls | Apache 2.0 |
| `gitpython` | ≥3.1.30, <4.0 | Git repository operations (commit, push, diff-check) | BSD-3-Clause |
| `python-dotenv` | ≥1.0.0, <2.0 | `.env` file loading for credential management | BSD-3-Clause |
| `markdown` | ≥3.4.0, <4.0 | Markdown parsing and validation for G2 gate | BSD-3-Clause |
| `jsonschema` | ≥4.19.0, <5.0 | JSON Schema validation for G5 gate | MIT |
| `click` | ≥8.1.0, <9.0 | CLI framework (alternative to argparse, optional) | BSD-3-Clause |

### Dependency Tree (Simplified)
```
agentic-artifacts
├── requests ─────── urllib3, certifi, charset-normalizer, idna
├── jinja2 ───────── MarkupSafe
├── pyyaml ───────── (no sub-deps)
├── pydantic ─────── pydantic-core, typing-extensions, annotated-types
├── rich ─────────── markdown-it-py, pygments, mdurl
├── tenacity ─────── (no sub-deps)
├── gitpython ────── gitdb, smmap
├── python-dotenv ── (no sub-deps)
├── markdown ─────── (no sub-deps)
├── jsonschema ───── attrs, rpds-py, referencing, pyrsistent
└── click ────────── (no sub-deps)
```

### Python Version Compatibility
| Python Version | Status | Notes |
|---------------|--------|-------|
| 3.9 | ⚠️ Deprecated | Security-only, end-of-life in 2025 |
| 3.10 | ✅ Minimum | Required for `match`/`case` and type union syntax |
| 3.11 | ✅ Recommended | Significant performance improvements |
| 3.12 | ✅ Supported | Verified with all dependencies |
| 3.13 | 🔄 Testing | Awaiting dependency wheel availability |

---

## 3. GitHub API Dependencies

### API Endpoints Used
| Endpoint | Method | Purpose | Rate Limit Cost |
|----------|--------|---------|----------------|
| `/repos/{owner}/{repo}` | GET | Repository metadata (language, default branch, topics) | 1 |
| `/repos/{owner}/{repo}/contents/{path}` | GET | File/directory listing and content retrieval | 1 |
| `/repos/{owner}/{repo}/commits` | GET | Commit history (last N commits) | 1 |
| `/repos/{owner}/{repo}/git/trees/{sha}?recursive=1` | GET | Full file tree for dependency scanning | 1 |
| `/search/repositories` | GET | Repository discovery | 10 (search endpoint) |
| `/repos/{owner}/{repo}/languages` | GET | Language breakdown for dependency reports | 1 |

### Rate Limits
| Tier | Limit | Reset Period | Strategy |
|------|-------|-------------|----------|
| Authenticated (PAT) | 5,000 requests/hour | 60 minutes | Conditional requests (ETag) reduce consumed quota |
| Authenticated (GitHub App) | 5,000–15,000 requests/hour | 60 minutes | Installation token with higher limits |
| Unauthenticated | 60 requests/hour | 60 minutes | Never used; authentication is mandatory |
| Search endpoint | 30 requests/minute | 1 minute | Rate-limited separately; batch search queries |

### Authentication Methods
- **Personal Access Token (PAT)**: Classic or fine-grained with `repo` and `read:org` scopes
- **GitHub App Installation Token**: Recommended for organization-wide deployments
- **Environment Variable**: `GITHUB_TOKEN` loaded via `python-dotenv`

---

## 4. Test Dependencies

### Test Framework Packages
| Package | Version | Purpose |
|---------|---------|---------|
| `pytest` | ≥7.4.0, <9.0 | Primary test framework |
| `pytest-cov` | ≥4.1.0, <6.0 | Code coverage reporting (target: ≥85%) |
| `pytest-xdist` | ≥3.3.0, <4.0 | Parallel test execution |
| `pytest-mock` | ≥3.12.0, <4.0 | Mocking utilities for unit tests |
| `responses` | ≥0.23.0, <1.0 | HTTP request mocking for ServiceNow and GitHub API tests |
| `freezegun` | ≥1.2.0, <2.0 | Time freezing for deterministic date-sensitive tests |
| `faker` | ≥19.0.0, <21.0 | Synthetic test data generation |
| `coverage` | ≥7.3.0, <8.0 | Code coverage measurement engine |

---

## 5. External System Integration Points

### Integration Diagram
```
┌─────────────────────────────────────────────────────────┐
│                   agentic-artifacts                     │
│                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────────────┐    │
│  │ REST     │   │ GitHub   │   │ Git (local)      │    │
│  │ Client   │   │ Client   │   │                  │    │
│  └────┬─────┘   └────┬─────┘   └───────┬──────────┘    │
│       │              │                  │               │
└───────┼──────────────┼──────────────────┼───────────────┘
        │              │                  │
        ▼              ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  ServiceNow  │ │  GitHub.com  │ │  Local Git   │
│  Instance    │ │  API         │ │  Repository  │
│  ─────────── │ │  ─────────── │ │  ─────────── │
│  • Table API │ │  • REST v3   │ │  • clone     │
│  • OAuth     │ │  • GraphQL   │ │  • commit    │
│  • Batch API │ │  • Webhooks  │ │  • push      │
└──────────────┘ └──────────────┘ └──────────────┘
        │              │                  │
        ▼              ▼                  ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  .env File   │ │  GitHub      │ │  Remote      │
│  Credentials │ │  App / PAT   │ │  Origin      │
└──────────────┘ └──────────────┘ └──────────────┘
```

### Integration Dependencies
| System | Protocol | Port | TLS | Timeout (s) |
|--------|----------|------|-----|-------------|
| ServiceNow REST API | HTTPS | 443 | TLS 1.2+ | Connect: 30, Read: 60 |
| GitHub REST API | HTTPS | 443 | TLS 1.2+ | Connect: 15, Read: 30 |
| GitHub GraphQL API | HTTPS | 443 | TLS 1.2+ | Connect: 15, Read: 30 |
| Local Git (filesystem) | N/A | N/A | N/A | N/A |
| Remote Git (SSH) | SSH | 22 | SSH | 30 |
| Remote Git (HTTPS) | HTTPS | 443 | TLS 1.2+ | 30 |

---

## 6. Version Compatibility Matrix

| Component | Minimum | Recommended | Maximum Tested |
|-----------|---------|-------------|----------------|
| ServiceNow | Utah Patch 3 | Washington DC | Washington DC Patch 6 |
| Python | 3.10.0 | 3.11.7 | 3.12.3 |
| requests | 2.31.0 | 2.32.3 | 2.32.3 |
| Jinja2 | 3.1.2 | 3.1.4 | 3.1.4 |
| PyYAML | 6.0 | 6.0.1 | 6.0.1 |
| pydantic | 2.4.0 | 2.7.1 | 2.7.1 |
| Git | 2.39.0 | 2.44.0 | 2.44.0 |
| pytest | 7.4.0 | 8.2.0 | 8.2.0 |
| GitHub PAT scope | `repo` | `repo`, `read:org` | Full `repo` |
| OS (Linux) | Ubuntu 20.04 | Ubuntu 22.04 | Ubuntu 24.04 |
| OS (macOS) | 12 Monterey | 14 Sonoma | 14 Sonoma |
| OS (Windows) | Windows 10 | Windows 11 | Windows 11 |

---

## 7. Installation Command

```bash
# Production install (pinned with hashes)
pip install --require-hashes -r requirements.txt

# Development install (editable with test deps)
pip install -e ".[dev,test]"

# Verify installation
python -c "import agentic_artifacts; print(agentic_artifacts.__version__)"
```

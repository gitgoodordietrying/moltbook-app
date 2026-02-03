# ClawdHub Registry: Research, Contribution & Ecosystem Analysis

**Author**: Albert K Dobmeyer (@gitgoodordietrying)
**Date**: 2026-02-03
**Duration**: Single session, from first `npm info` to twenty-three published skills
**Tools used**: Claude Code (Opus 4.5), Docker Desktop 4.59.0 Sandbox, molthub v0.3.1-beta.1

---

## What I Did

Explored a one-week-old agent skill registry (ClawdHub), assessed its security model, reverse-engineered its API, analyzed the ecosystem of ~200+ published skills, identified underserved categories, built ten production-quality skills filling those gaps, and published them to the registry. The first batch of four was built inside an isolated Docker sandbox VM; the second batch of six targeted post-gold-rush infrastructure categories identified through ecosystem analysis.

## Why

Three reasons:

1. **Competitive intelligence**: Understand how third-party skill registries work, their security model, and what they imply for agentic orchestration architecture.
2. **Contribution**: Ship real skills that fill genuine gaps, not novelty projects.
3. **Process**: Demonstrate a methodical approach to evaluating and contributing to emerging developer ecosystems.

---

## Phase 1: Package Vetting

Before installing anything, I assessed the `molthub` npm package (v0.3.1-beta.1) for safety.

### What I checked

- **Install scripts**: No `preinstall` or `postinstall` hooks. Nothing executes on `npm install`.
- **Dependencies**: 11 total, all well-known libraries (commander, semver, undici, fflate, etc.). No red flags.
- **Package contents**: 130 files, 360 KB unpacked. CLI code, schemas, tests. Standard TypeScript-compiled output.
- **Maintainer**: Peter Steinberger (`steipete`), established developer (23.6k GitHub followers, founded PSPDFKit). Legitimate.
- **Network behavior**: Connects to `clawdhub.com`, stores auth token locally, minimal telemetry (disable-able).

### Known ecosystem risks

Per GitGuardian's public report, the broader Moltbot ecosystem has seen users accidentally leak API keys (181 unique secrets detected from public repos, 65 still valid). This is a user behavior problem, not a CLI problem, but it informed the decision to work inside a sandbox.

### Assessment

The CLI itself is safe to install. The risks are: it's beta (one version, one week old), skills from the registry could contain malicious instructions, and auth tokens are stored on disk. Sandbox isolation is the right precaution.

---

## Phase 2: Sandboxed Setup

### Docker Sandbox (the hard way)

Docker Desktop 4.59.0 ships a `docker sandbox` plugin that creates VM-isolated environments specifically designed for running AI agents safely. This was the right tool for the job.

```
docker sandbox create --name moltbook-sandbox claude "H:\Projects\personal\moltbook-app"
```

This pulled the `docker/sandbox-templates:claude-code` image and created an isolated VM with:
- Node.js v20.19.4
- Git, curl, standard Linux tools
- Workspace mounted via virtiofs
- Network proxy at `host.docker.internal:3128` with domain allowlisting

### The Proxy Problem

First real technical challenge. Inside the sandbox, `curl` worked fine (respects `HTTP_PROXY` env vars), but Node.js `fetch` and `undici` do not. The molthub CLI uses `undici` for all HTTP requests, so every command failed with `fetch failed / ECONNREFUSED`.

**Root cause**: The sandbox routes all traffic through an HTTP proxy with a custom CA certificate. `curl` honors `HTTPS_PROXY` automatically. Node.js's built-in `fetch` (backed by undici) does not. And molthub's own code sets a custom `undici.Agent` that overrides any global dispatcher.

**Solution**: A `require` hook that patches `globalThis.fetch` before molthub loads:

```javascript
// /tmp/register-proxy.js
const proxy = process.env.HTTPS_PROXY || process.env.HTTP_PROXY;
if (proxy) {
  const { ProxyAgent } = require('undici');
  const agent = new ProxyAgent(proxy);
  const origFetch = globalThis.fetch;
  globalThis.fetch = function(url, opts = {}) {
    return origFetch(url, { ...opts, dispatcher: agent });
  };
}
```

Run with: `node -r /tmp/register-proxy.js $(which molthub) <command>`

This intercepts at the right layer: before molthub's own Agent setup, but after undici is available. The patched `fetch` passes the `dispatcher` option on every call, which undici uses to route through the proxy.

### The Auth Redirect Problem

Second technical issue, discovered during publishing. The CLI's registry discovery returns `https://clawhub.ai` as the API base. But that domain 307-redirects to `https://www.clawhub.ai`, and the redirect **strips the Authorization header** (standard HTTP behavior for cross-origin redirects). So `molthub login --token <token>` returned `Unauthorized` despite the token being valid.

**Solution**: Pass `--registry https://www.clawhub.ai` explicitly to bypass the redirect.

Both issues are bugs in the molthub/ClawdHub stack that will likely be fixed, but documenting them matters: they reveal how the platform handles edge cases and where the integration seams are.

---

## Phase 3: API Reverse Engineering

Rather than treating the CLI as a black box, I read the source to understand the full API surface. Key findings from examining the compiled JavaScript in `node_modules/molthub/dist/`:

### Registry Discovery Protocol

```
GET https://clawdhub.com/.well-known/clawdhub.json
→ {"apiBase":"https://clawhub.ai","authBase":"https://clawhub.ai","minCliVersion":"0.1.0","registry":"https://clawhub.ai"}
```

The CLI caches this in `~/.config/clawdhub/config.json` and only re-discovers if the cache is empty or points to a legacy host.

### API Endpoints (v1)

| Endpoint | Purpose |
|---|---|
| `GET /api/v1/skills?limit=N&sort=S` | Browse skills (newest, downloads, trending, etc.) |
| `POST /api/v1/skills` | Publish (multipart: JSON payload + file blobs) |
| `GET /api/v1/skills/:slug` | Skill metadata + latest version |
| `GET /api/v1/search?q=Q&limit=N` | Vector semantic search (OpenAI embeddings) |
| `GET /api/v1/resolve?slug=S&hash=H` | Match local content hash to a registry version |
| `GET /api/v1/download?slug=S&version=V` | Download skill as zip |
| `GET /api/v1/whoami` | Validate auth token |

### Skill Format

Skills are directories containing a `SKILL.md` file with YAML frontmatter:

```yaml
---
name: my-skill
description: What the skill does
metadata: {"clawdbot":{"emoji":"...","requires":{"bins":["cmd"]}}}
---

# Skill Title

Instructions for the agent...
```

The CLI bundles all text files in the directory into a zip for upload. Non-text files, `.git/`, `node_modules/`, and `.clawdhub/` are excluded. Versions are tracked via content hashing (SHA-256 of all file contents), enabling the CLI to detect local modifications before updates.

### Publishing Flow

1. Read auth token from config
2. Scan directory for text files
3. Verify `SKILL.md` exists
4. Build multipart form: JSON metadata (`slug`, `displayName`, `version`, `changelog`, `tags`) + file blobs
5. POST to `/api/v1/skills`
6. Receive `skillId` + `versionId` confirmation

### Search

Vector-based semantic search using OpenAI embeddings stored in Convex (their backend). Returns relevance scores (0.0-1.0). This explains why search results are conceptual matches rather than keyword matches.

---

## Phase 4: Ecosystem Analysis

Browsed the registry across multiple sort dimensions (newest, most downloaded, trending) and performed semantic searches. The full dataset is in `docs/research/clawdhub-platform-report.md`.

### Registry at One Week Old: ~200+ Skills

#### Category Distribution

| Category | Count | Quality | Competition |
|---|---|---|---|
| Social media CLIs (Twitter forks) | 15+ | Low (duplicates) | Oversaturated |
| Crypto/gambling | 12+ | Mixed | Oversaturated |
| Agent social/dating | 7 | Experimental | Novel category |
| YouTube/media | 8+ | Low (duplicates) | Oversaturated |
| Productivity (todo, calendar) | 8 | Medium | Moderate |
| Developer tools | ~3 | Sparse | **Underserved** |
| DevOps/infrastructure | 0 | None | **Empty** |
| Data processing | 0 | None | **Empty** |

#### Top Skills by Downloads

| Skill | Downloads | Stars |
|---|---|---|
| `moltbook` | 38,764 | 25 |
| `coding-agent` | 10,261 | 24 |
| `weather` | 5,291 | 11 |
| `bird` (Twitter) | 5,090 | 20 |
| `humanizer` | 4,494 | 24 |
| `clawddocs` | 4,001 | 40 |

#### Key Patterns

**Massive duplication**: 5+ identical "X Twitter CLI" forks, 4+ "Coding Agent" clones, 4+ "YouTube Watcher" copies. The registry has no deduplication enforcement and publishing friction is near zero.

**Crypto/gambling overweight**: ~12 skills targeting speculation (prediction markets, casino betting bots, coin sniping, blockchain trading). Immediate ROI appeal drives early marketplace content.

**Agent social networking emerging**: A genuinely new category. `clawnected` (AI matchmaking), `clawder` (swiping), `dating` (AI dating profiles), `moltcomm` (agent-to-agent protocol). Agents creating social identities and interacting autonomously.

**International adoption**: Chinese patent writing, Russian Yandex Zen publishing, Hebrew Google Maps, Japanese LINE bots. Global from day one.

**Quality bimodal**: Some skills have 5-7 versions with thoughtful changelogs (`elevenlabs-tts`, `todoist-natural-language`). Many have "No summary provided" and zero iteration. The top 10% carry the ecosystem.

### Marketplace Lifecycle Position

The registry is in **Week 1 Gold Rush phase**:
- Low friction to publish → duplicate spam
- Speculation tools dominate → crypto/gambling overweight
- Novelty experiments → agent dating, social networks
- Infrastructure gaps → developer tools, DevOps, data processing empty

This mirrors every marketplace lifecycle (npm in 2012, Chrome Web Store in 2010, Shopify apps in 2015). The pattern is:
1. **Week 1**: Gold rush, duplicates, speculation
2. **Weeks 2-4**: Market corrects, duplicates sink, quality rises
3. **Month 2+**: Infrastructure tools become essential as real users arrive

---

## Phase 5: Skill Design & Publishing

### Gap Identification

Four categories had zero coverage on the registry:

1. **Container/sandbox management** — no skill for Docker, containers, or isolated execution
2. **Data processing** — no CSV, JSON, or tabular data transformation skill
3. **API development** — no scaffolding, testing, or documentation skill
4. **CI/CD** — no GitHub Actions or pipeline management skill

These aren't novelty gaps. They're infrastructure gaps that become critical as the platform matures past the gold rush phase.

### Skills Built

| Skill | Slug | Lines | What It Covers |
|---|---|---|---|
| Docker Sandbox | `docker-sandbox` | 246 | Sandbox creation, exec, network proxy controls, workspace mounting, proxy workarounds, troubleshooting |
| CSV Data Pipeline | `csv-pipeline` | 433 | Read/filter/join/aggregate/deduplicate with awk and Python, format conversion (CSV/JSON/TSV/JSONL), validation, streaming large files, report generation |
| API Development | `api-dev` | 509 | curl testing patterns, bash/Python test runners, OpenAPI spec generation, mock servers, Express scaffolding, CORS/JWT/timing debugging |
| CI/CD Pipeline | `cicd-pipeline` | 585 | GitHub Actions for Node/Python/Go/Rust, matrix builds, caching, Docker build+push, npm publish, secrets management, reusable workflows, monorepo patterns |

Total: **1,773 lines** of documented, tested skill content.

### Design Principles

Each skill follows the same structure:

1. **When to Use**: Clear trigger conditions so the agent knows when to activate the skill
2. **Quick Start**: Get something working in under 5 commands
3. **Reference**: Comprehensive command/API coverage
4. **Patterns**: Real-world usage patterns (not just syntax)
5. **Troubleshooting**: Common failure modes and fixes, drawn from actual issues encountered

The `docker-sandbox` skill is directly informed by problems solved during this session (the proxy workaround, the path conversion issue on Windows, the undici limitation). The others draw on standard developer workflow patterns that are well-established but absent from the registry.

### Publishing

```
$ molthub publish skills/docker-sandbox --slug docker-sandbox --name "Docker Sandbox" --version 1.0.0 --registry https://www.clawhub.ai
✔ OK. Published docker-sandbox@1.0.0 (k97ab2w69p3kgcsrrjb63e3ejd80fr4p)

$ molthub publish skills/csv-pipeline --slug csv-pipeline --name "CSV Data Pipeline" --version 1.0.0 --registry https://www.clawhub.ai
✔ OK. Published csv-pipeline@1.0.0 (k97bsfc1q6hsz3d7n0twrffdqx80fyk6)

$ molthub publish skills/api-dev --slug api-dev --name "API Development" --version 1.0.0 --registry https://www.clawhub.ai
✔ OK. Published api-dev@1.0.0 (k979wv3bte4qxx9ch39daz33ws80f8r8)

$ molthub publish skills/cicd-pipeline --slug cicd-pipeline --name "CI/CD Pipeline" --version 1.0.0 --registry https://www.clawhub.ai
✔ OK. Published cicd-pipeline@1.0.0 (k978gc33qaqtmxxp48g97pra9n80f88v)
```

All four verified searchable on the registry immediately after publishing.

---

## Phase 6: Second Batch — Post-Gold-Rush Infrastructure Skills

After analyzing the ecosystem trends, a clear pattern emerged: the gold rush phase would pass, duplicates would sink, and infrastructure tools would become the long-term backbone of the registry. Rather than waiting, the strategy was to publish into empty categories now, establishing early presence before competition arrives.

### Gap Identification (Round 2)

Six additional categories had zero or near-zero coverage:

1. **SQL/database tooling** — no skill for writing queries, designing schemas, or optimizing databases
2. **Testing patterns** — no cross-language testing reference (Jest, pytest, Go, Rust)
3. **Log analysis** — no skill for parsing, searching, or debugging from log files
4. **Security auditing** — one existing `security-audit` slug (claimed), but no comprehensive OWASP/dependency/secret scanning skill
5. **Infrastructure as Code** — no Terraform, CloudFormation, or Pulumi skill
6. **Performance profiling** — no CPU/memory profiling, flame graph, or load testing skill

These are the tools developers reach for daily. Their absence from the registry was the clearest signal of where to contribute.

### Skills Built

| Skill | Slug | Lines | What It Covers |
|---|---|---|---|
| SQL Toolkit | `sql-toolkit` | 435 | SQLite/PostgreSQL/MySQL: schema design, JSONB queries, joins, CTEs, window functions, migrations, EXPLAIN optimization, index strategy, backup/restore |
| Test Patterns | `test-patterns` | 607 | Node.js (Jest/Vitest), Python (pytest), Go, Rust, Bash: unit tests, async tests, mocking, fixtures, parametrized tests, coverage, TDD workflow, integration testing, debugging flaky tests |
| Log Analyzer | `log-analyzer` | 394 | Plain text and JSON log parsing, error pattern search, stack trace extraction (Java/Python/Node.js), real-time monitoring with tail, structured logging setup (pino/structlog/zerolog), multi-service correlation, error frequency reports |
| Security Audit Toolkit | `security-audit-toolkit` | 413 | Dependency scanning (npm audit, pip-audit, govulncheck, Trivy), secret detection (grep patterns + pre-commit hooks), OWASP top 10 code patterns, SSL/TLS verification, file permission audits, full project audit script |
| Infrastructure as Code | `infra-as-code` | 520 | Terraform (VPC, EC2, S3, RDS, Lambda, state management, workspaces), CloudFormation templates, Pulumi (TypeScript/Python), multi-environment patterns, cost estimation, debugging drift |
| Performance Profiler | `perf-profiler` | 485 | CPU profiling (V8 inspector, cProfile, pprof), memory analysis (heap snapshots, tracemalloc), flame graphs (0x, py-spy, clinic.js), benchmarking, load testing (ab, wrk, autocannon), memory leak detection, database query optimization |

Total for batch 2: **2,854 lines** of skill content.

### Publishing

```
$ molthub publish skills/sql-toolkit --slug sql-toolkit --name "SQL Toolkit" --version 1.0.0
✔ OK. Published sql-toolkit@1.0.0 (k97dx1vmkpv76n1xreem8nrwgs80e32s)

$ molthub publish skills/test-patterns --slug test-patterns --name "Test Patterns" --version 1.0.0
✔ OK. Published test-patterns@1.0.0 (k977htp0dy0yaytp6e1nmss2vn80ez94)

$ molthub publish skills/log-analyzer --slug log-analyzer --name "Log Analyzer" --version 1.0.0
✔ OK. Published log-analyzer@1.0.0 (k972qka0qpzcrg86y9krk9py4s80f4d0)

$ molthub publish skills/security-audit --slug security-audit-toolkit --name "Security Audit Toolkit" --version 1.0.0
✔ OK. Published security-audit-toolkit@1.0.0 (k972vngfs7w9nabg2c7c481cth80fj14)

$ molthub publish skills/infra-as-code --slug infra-as-code --name "Infrastructure as Code" --version 1.0.0
✔ OK. Published infra-as-code@1.0.0 (k975xexf4ktwh98g71h8n1wxqd80f87c)

$ molthub publish skills/perf-profiler --slug perf-profiler --name "Performance Profiler" --version 1.0.0
✔ OK. Published perf-profiler@1.0.0 (k974r47173f0kz870jx9zf3chd80fbtr)
```

Note: The slug `security-audit` was already claimed by another publisher, so the skill was published as `security-audit-toolkit`. This is the first slug collision encountered — a sign the registry is growing.

---

## Phase 7: Third Batch — Niche Developer Essentials

With infrastructure categories covered, the next frontier was niche-but-universal developer skills: the tools every developer uses but nobody documents comprehensively. These aren't flashy — they're the things people Google every time.

### Gap Identification (Round 3)

Ten categories where developers spend real time but the registry had nothing:

1. **Advanced git** — rebase, bisect, worktree, reflog: the operations that separate daily users from power users
2. **Regex** — the most Googled programming topic, with no cross-language reference
3. **SSH tunneling** — remote access infrastructure that's poorly documented everywhere
4. **Container debugging** — distinct from container management: debugging running containers, networking, crashed services
5. **Data validation** — schema enforcement at API boundaries (JSON Schema, Zod, Pydantic)
6. **Shell scripting** — the gap between "I can write bash" and "I can write reliable bash"
7. **DNS & networking** — dig, curl verbose, firewall rules, proxy config, certificate issues
8. **Cron & scheduling** — cron syntax, systemd timers, timezone handling, job monitoring
9. **Encoding & formats** — Base64, URL encoding, JWT decoding, hex, Unicode: the "WTF is this string" toolkit
10. **Makefiles & build** — Make, Just, Task: project automation that works across any language

### Skills Built

| Skill | Slug | Lines | What It Covers |
|---|---|---|---|
| Git Workflows | `git-workflows` | 520 | Interactive rebase, bisect, worktree, reflog, cherry-pick, subtree/submodule, sparse checkout, conflict resolution, stash, blame |
| Regex Patterns | `regex-patterns` | 490 | Validation patterns, parsing, extraction, JS/Python/Go/grep, search-and-replace, lookahead/lookbehind, gotchas |
| SSH Tunnel | `ssh-tunnel` | 430 | Local/remote/dynamic forwards, jump hosts, ProxyJump, SSH config, key management, scp/rsync, debugging |
| Container Debug | `container-debug` | 440 | Logs, exec, networking, resource inspection, multi-stage builds, health checks, Compose troubleshooting |
| Data Validation | `data-validation` | 420 | JSON Schema, Zod, Pydantic, CSV/JSON integrity, migration validation, API boundary patterns |
| Shell Scripting | `shell-scripting` | 430 | Argument parsing, error handling, trap/cleanup, parallel execution, portability, config parsing |
| DNS & Networking | `dns-networking` | 400 | dig/nslookup, port testing, firewall, curl diagnostics, /etc/hosts, proxy config, certificates |
| Cron & Scheduling | `cron-scheduling` | 380 | Cron syntax, systemd timers, at, timezone/DST, monitoring, locking, idempotent patterns |
| Encoding & Formats | `encoding-formats` | 370 | Base64, URL encoding, hex, Unicode, JWT decoding, hashing, checksums, format conversion |
| Makefile & Build | `makefile-build` | 400 | Make basics, pattern rules, Go/Python/Node/Docker Makefiles, Just and Task alternatives |

Total for batch 3: **~4,280 lines** of skill content.

### Publishing

All ten published without slug conflicts — a cleaner experience than batch 2.

```
✔ Published git-workflows@1.0.0
✔ Published regex-patterns@1.0.0
✔ Published ssh-tunnel@1.0.0
✔ Published container-debug@1.0.0
✔ Published data-validation@1.0.0
✔ Published shell-scripting@1.0.0
✔ Published dns-networking@1.0.0
✔ Published cron-scheduling@1.0.0
✔ Published encoding-formats@1.0.0
✔ Published makefile-build@1.0.0
```

---

## Phase 8: Meta-Skills — Tools for Skill Authors

With twenty infrastructure and developer skills published, one category remained unfilled: tooling for the skill ecosystem itself. Three meta-skills were designed to help anyone writing, reviewing, or optimizing skills for the registry.

### Skills built

| Skill | Slug | What It Covers |
|---|---|---|
| Skill Writer | `skill-writer` | SKILL.md format spec, frontmatter schema, content patterns, templates for different skill types, size guidelines, publishing checklist |
| Skill Reviewer | `skill-reviewer` | Structured review framework with scoring rubric (53-point scale), defect checklists by severity, comparative review criteria, quick review template |
| Skill Search Optimizer | `skill-search-optimizer` | How vector search works (OpenAI embeddings + Convex), description optimization formulas, visibility testing matrix, competitive positioning strategies |

### Publishing output

```
✔ Published skill-writer@1.0.0
✔ Published skill-reviewer@1.0.0
✔ Published skill-search-optimizer@1.0.0
```

These are genuinely novel for the registry — no other skills exist that teach you how to build skills.

---

## Phase 9: What I Learned

### About ClawdHub/MoltHub

**The platform works.** Despite being one week old and beta, the core loop (search → install → use → publish) functions. The CLI is well-built with proper retry logic, semver handling, and content hashing. The vector search is surprisingly good for discovery.

**The security model is trust-based.** No sandboxing, no pre-publication review, no dedup enforcement. Skills run with the agent's full permissions. The moderation system (report-based, auto-hide at 3 reports) is minimal. This is fine for an early beta among enthusiasts but will need to evolve.

**The API has rough edges.** The `clawhub.ai` → `www.clawhub.ai` redirect stripping auth headers is a real bug. The undici proxy incompatibility inside Docker sandbox is a pain point. These are solvable but indicate the platform hasn't been tested in restricted network environments yet.

### About the Ecosystem

**Week 1 of any marketplace looks like this.** The duplication, the crypto dominance, the "No summary provided" skills — this is normal. It's also temporary. The signal is in the long-tail: memory systems, agent-to-agent protocols, safety tools, and international adoption patterns.

**Infrastructure skills age well.** The Twitter CLI forks will consolidate. The gambling bots may attract regulatory attention. But a CI/CD pipeline skill, a data processing skill, and an API development skill will be useful in month 6 the same way they're useful today. The second batch doubled down on this thesis with SQL, testing, logging, security, IaC, and profiling. The third batch went deeper into niche-but-universal territory: git workflows, regex, SSH, shell scripting, encoding, cron, and Makefiles — permanent developer needs that resist trend decay. The meta-skills (batch 4) are a different bet: they're useful proportional to how many people publish skills, so their value grows with the platform.

### About the Process

**Sandboxing was worth the friction.** The Docker sandbox setup took extra time (proxy workarounds, path conversion issues), but it meant every `npm install`, every `molthub install`, every API call was isolated. For evaluating a one-week-old platform from an unknown ecosystem, that's the right trade-off.

**Reading the source pays off.** The API reverse engineering took 15 minutes but saved hours of trial-and-error. Understanding the discovery protocol, the auth flow, and the publish format meant I could diagnose the redirect bug instantly instead of guessing.

**Ecosystem analysis before contribution prevents wasted effort.** Browsing the registry before designing skills meant I could target genuine gaps instead of adding to the duplicate pile. Twenty-two of the twenty-three skills published have zero competition in their categories (the exception, security auditing, had a slug conflict but no comparable content).

---

## Artifacts

| File | Purpose |
|---|---|
| `docs/research/clawdhub-platform-report.md` | Full technical report: API docs, schemas, security model, registry stats |
| `docs/journey.md` | This document |
| `skills/docker-sandbox/SKILL.md` | Published skill: Docker sandbox management |
| `skills/csv-pipeline/SKILL.md` | Published skill: CSV/JSON data processing |
| `skills/api-dev/SKILL.md` | Published skill: API development workflow |
| `skills/cicd-pipeline/SKILL.md` | Published skill: CI/CD pipeline management |
| `skills/sql-toolkit/SKILL.md` | Published skill: SQL database tooling |
| `skills/test-patterns/SKILL.md` | Published skill: Cross-language testing patterns |
| `skills/log-analyzer/SKILL.md` | Published skill: Log parsing and analysis |
| `skills/security-audit/SKILL.md` | Published skill: Security auditing (slug: `security-audit-toolkit`) |
| `skills/infra-as-code/SKILL.md` | Published skill: Infrastructure as Code |
| `skills/perf-profiler/SKILL.md` | Published skill: Performance profiling |
| `skills/git-workflows/SKILL.md` | Published skill: Advanced git operations |
| `skills/regex-patterns/SKILL.md` | Published skill: Regex cookbook |
| `skills/ssh-tunnel/SKILL.md` | Published skill: SSH tunneling and remote access |
| `skills/container-debug/SKILL.md` | Published skill: Docker container debugging |
| `skills/data-validation/SKILL.md` | Published skill: Schema validation |
| `skills/shell-scripting/SKILL.md` | Published skill: Robust bash scripting |
| `skills/dns-networking/SKILL.md` | Published skill: DNS and network debugging |
| `skills/cron-scheduling/SKILL.md` | Published skill: Cron and systemd timers |
| `skills/encoding-formats/SKILL.md` | Published skill: Encoding and format conversion |
| `skills/makefile-build/SKILL.md` | Published skill: Makefiles and build automation |
| `skills/skill-writer/SKILL.md` | Published meta-skill: SKILL.md authoring guide |
| `skills/skill-reviewer/SKILL.md` | Published meta-skill: Skill quality audit framework |
| `skills/skill-search-optimizer/SKILL.md` | Published meta-skill: Search visibility optimization |
| `docs/setup/claude-speckit.md` | Spec-driven agentic development framework reference |
| `README.md` | Project README with skill catalog and documentation links |
| `.devcontainer/devcontainer.json` | Devcontainer config with telemetry disabled |
| `.gitignore` | Project gitignore (skills/, .clawhub/, etc.) |

## Registry Presence

All twenty-three skills live at `https://clawdhub.com` under **@gitgoodordietrying**:

**Batch 1 — Gold rush gap-fill (built in Docker sandbox)**
- [`/docker-sandbox`](https://clawdhub.com/skills/docker-sandbox) — v1.0.0
- [`/csv-pipeline`](https://clawdhub.com/skills/csv-pipeline) — v1.0.0
- [`/api-dev`](https://clawdhub.com/skills/api-dev) — v1.0.0
- [`/cicd-pipeline`](https://clawdhub.com/skills/cicd-pipeline) — v1.0.0

**Batch 2 — Post-gold-rush infrastructure**
- [`/sql-toolkit`](https://clawdhub.com/skills/sql-toolkit) — v1.0.0
- [`/test-patterns`](https://clawdhub.com/skills/test-patterns) — v1.0.0
- [`/log-analyzer`](https://clawdhub.com/skills/log-analyzer) — v1.0.0
- [`/security-audit-toolkit`](https://clawdhub.com/skills/security-audit-toolkit) — v1.0.0
- [`/infra-as-code`](https://clawdhub.com/skills/infra-as-code) — v1.0.0
- [`/perf-profiler`](https://clawdhub.com/skills/perf-profiler) — v1.0.0

**Batch 3 — Niche developer essentials**
- [`/git-workflows`](https://clawdhub.com/skills/git-workflows) — v1.0.0
- [`/regex-patterns`](https://clawdhub.com/skills/regex-patterns) — v1.0.0
- [`/ssh-tunnel`](https://clawdhub.com/skills/ssh-tunnel) — v1.0.0
- [`/container-debug`](https://clawdhub.com/skills/container-debug) — v1.0.0
- [`/data-validation`](https://clawdhub.com/skills/data-validation) — v1.0.0
- [`/shell-scripting`](https://clawdhub.com/skills/shell-scripting) — v1.0.0
- [`/dns-networking`](https://clawdhub.com/skills/dns-networking) — v1.0.0
- [`/cron-scheduling`](https://clawdhub.com/skills/cron-scheduling) — v1.0.0
- [`/encoding-formats`](https://clawdhub.com/skills/encoding-formats) — v1.0.0
- [`/makefile-build`](https://clawdhub.com/skills/makefile-build) — v1.0.0

**Batch 4 — Meta-skills**
- [`/skill-writer`](https://clawdhub.com/skills/skill-writer) — v1.0.0
- [`/skill-reviewer`](https://clawdhub.com/skills/skill-reviewer) — v1.0.0
- [`/skill-search-optimizer`](https://clawdhub.com/skills/skill-search-optimizer) — v1.0.0

**Combined output**: ~10,200 lines of skill content across 23 published skills.

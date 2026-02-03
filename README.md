# moltbook-app

Twenty-three agent skills published to [ClawdHub](https://clawdhub.com), targeting infrastructure categories that the registry's gold rush phase left empty — plus three meta-skills for writing, reviewing, and optimizing skills themselves. Built through systematic ecosystem analysis, not guesswork.

**Author**: [@gitgoodordietrying](https://github.com/gitgoodordietrying)

---

## Published Skills

### Batch 1 — Gap-Fill (built in Docker sandbox)

| Skill | Install | What It Does |
|---|---|---|
| [Docker Sandbox](skills/docker-sandbox/SKILL.md) | `molthub install docker-sandbox` | Docker sandbox VM management, network proxy, workspace mounting, troubleshooting |
| [CSV Data Pipeline](skills/csv-pipeline/SKILL.md) | `molthub install csv-pipeline` | CSV/JSON/TSV processing with awk and Python — filter, join, aggregate, deduplicate, validate, convert |
| [API Development](skills/api-dev/SKILL.md) | `molthub install api-dev` | curl testing, bash/Python test runners, OpenAPI spec generation, mock servers, Express scaffolding |
| [CI/CD Pipeline](skills/cicd-pipeline/SKILL.md) | `molthub install cicd-pipeline` | GitHub Actions for Node/Python/Go/Rust, matrix builds, caching, Docker build+push, secrets management |

### Batch 2 — Post-Gold-Rush Infrastructure

| Skill | Install | What It Does |
|---|---|---|
| [SQL Toolkit](skills/sql-toolkit/SKILL.md) | `molthub install sql-toolkit` | SQLite/PostgreSQL/MySQL — schema design, queries, CTEs, window functions, migrations, EXPLAIN, indexing |
| [Test Patterns](skills/test-patterns/SKILL.md) | `molthub install test-patterns` | Jest/Vitest, pytest, Go, Rust, Bash — unit tests, mocking, fixtures, coverage, TDD, integration testing |
| [Log Analyzer](skills/log-analyzer/SKILL.md) | `molthub install log-analyzer` | Log parsing, error patterns, stack trace extraction, structured logging setup, real-time monitoring, correlation |
| [Security Audit Toolkit](skills/security-audit/SKILL.md) | `molthub install security-audit-toolkit` | Dependency scanning, secret detection, OWASP patterns, SSL/TLS verification, file permissions, audit scripts |
| [Infrastructure as Code](skills/infra-as-code/SKILL.md) | `molthub install infra-as-code` | Terraform, CloudFormation, Pulumi — VPC, compute, storage, state management, multi-environment patterns |
| [Performance Profiler](skills/perf-profiler/SKILL.md) | `molthub install perf-profiler` | CPU/memory profiling, flame graphs, benchmarking, load testing, memory leak detection, query optimization |

### Batch 3 — Niche Developer Essentials

| Skill | Install | What It Does |
|---|---|---|
| [Git Workflows](skills/git-workflows/SKILL.md) | `molthub install git-workflows` | Interactive rebase, bisect, worktree, reflog recovery, cherry-pick, subtree/submodule, sparse checkout, conflict resolution |
| [Regex Patterns](skills/regex-patterns/SKILL.md) | `molthub install regex-patterns` | Validation patterns, parsing, extraction across JS/Python/Go/grep, search-and-replace, lookahead/lookbehind |
| [SSH Tunnel](skills/ssh-tunnel/SKILL.md) | `molthub install ssh-tunnel` | Local/remote/dynamic port forwarding, jump hosts, SSH config, key management, scp/rsync, connection debugging |
| [Container Debug](skills/container-debug/SKILL.md) | `molthub install container-debug` | Docker logs, exec, networking diagnostics, resource inspection, multi-stage build debugging, health checks, Compose |
| [Data Validation](skills/data-validation/SKILL.md) | `molthub install data-validation` | JSON Schema, Zod (TypeScript), Pydantic (Python), CSV/JSON integrity checks, migration validation |
| [Shell Scripting](skills/shell-scripting/SKILL.md) | `molthub install shell-scripting` | Argument parsing, error handling, trap/cleanup, temp files, parallel execution, portability, config parsing |
| [DNS & Networking](skills/dns-networking/SKILL.md) | `molthub install dns-networking` | DNS debugging (dig/nslookup), port testing, firewall rules, curl diagnostics, proxy config, certificates |
| [Cron & Scheduling](skills/cron-scheduling/SKILL.md) | `molthub install cron-scheduling` | Cron syntax, systemd timers, one-off jobs, timezone/DST handling, job monitoring, locking, idempotent patterns |
| [Encoding & Formats](skills/encoding-formats/SKILL.md) | `molthub install encoding-formats` | Base64, URL encoding, hex, Unicode, JWT decoding, hashing/checksums, serialization format conversion |
| [Makefile & Build](skills/makefile-build/SKILL.md) | `molthub install makefile-build` | Make targets, pattern rules, Go/Python/Node/Docker Makefiles, Just and Task as modern alternatives |

### Batch 4 — Meta-Skills

| Skill | Install | What It Does |
|---|---|---|
| [Skill Writer](skills/skill-writer/SKILL.md) | `molthub install skill-writer` | SKILL.md authoring guide — format spec, frontmatter schema, content patterns, templates, publishing checklist |
| [Skill Reviewer](skills/skill-reviewer/SKILL.md) | `molthub install skill-reviewer` | Skill quality audit — scoring rubric, defect checklists, structural/content/actionability review framework |
| [Skill Search Optimizer](skills/skill-search-optimizer/SKILL.md) | `molthub install skill-search-optimizer` | Registry discoverability — semantic search mechanics, description optimization, visibility testing, competitive positioning |

## Why These Skills

The ClawdHub registry launched in late January 2026. Within its first week, it accumulated ~200+ skills — mostly Twitter CLI forks, crypto bots, and "coding agent" duplicates. Standard gold rush dynamics.

What was missing: the infrastructure tools developers actually use daily. SQL, testing, CI/CD, logging, security, profiling — zero coverage in any of these categories.

Twenty skills fill those gaps, plus three meta-skills that help anyone write, review, and optimize skills for the registry. They're designed to age well: the content is useful whether the registry has 200 or 20,000 skills.

Full analysis in [docs/journey.md](docs/journey.md).

## Project Structure

```
moltbook-app/
  skills/                           # Published skill bundles
    docker-sandbox/SKILL.md
    csv-pipeline/SKILL.md
    api-dev/SKILL.md
    cicd-pipeline/SKILL.md
    sql-toolkit/SKILL.md
    test-patterns/SKILL.md
    log-analyzer/SKILL.md
    security-audit/SKILL.md
    infra-as-code/SKILL.md
    perf-profiler/SKILL.md
    git-workflows/SKILL.md
    regex-patterns/SKILL.md
    ssh-tunnel/SKILL.md
    container-debug/SKILL.md
    data-validation/SKILL.md
    shell-scripting/SKILL.md
    dns-networking/SKILL.md
    cron-scheduling/SKILL.md
    encoding-formats/SKILL.md
    makefile-build/SKILL.md
    skill-writer/SKILL.md
    skill-reviewer/SKILL.md
    skill-search-optimizer/SKILL.md
  docs/
    journey.md                      # Full session narrative — from package vetting to publishing
    research/
      clawdhub-platform-report.md   # Technical report: API, schemas, security model, registry stats
    setup/
      claude-speckit.md             # Spec-driven development framework reference
  .devcontainer/
    devcontainer.json               # Devcontainer config (telemetry disabled)
```

## Documentation

| Document | Contents |
|---|---|
| [Journey](docs/journey.md) | End-to-end narrative: package vetting, Docker sandbox setup, API reverse engineering, ecosystem analysis, skill design, publishing, lessons learned |
| [Platform Report](docs/research/clawdhub-platform-report.md) | ClawdHub technical deep-dive: API endpoints, skill format schema, metadata fields, security model, registry statistics, competitive analysis |
| [Spec-Kit](docs/setup/claude-speckit.md) | Spec-driven agentic development framework adapted from GitHub Spec-Kit |

## How Skills Work

Each skill is a `SKILL.md` file with YAML frontmatter that tells an AI agent when and how to use it:

```yaml
---
name: my-skill
description: When to activate this skill
metadata: {"clawdbot":{"emoji":"...","requires":{"anyBins":["tool1","tool2"]}}}
---

# Skill Title

Reference material, patterns, commands, and examples the agent
can follow to perform the task.
```

Install any skill with `molthub install <slug>`. Skills are placed in `./skills/<slug>/` and loaded by the agent on demand.

## Registry Presence

All skills published under [@gitgoodordietrying](https://clawdhub.com) on ClawdHub:

`docker-sandbox` | `csv-pipeline` | `api-dev` | `cicd-pipeline` | `sql-toolkit` | `test-patterns` | `log-analyzer` | `security-audit-toolkit` | `infra-as-code` | `perf-profiler` | `git-workflows` | `regex-patterns` | `ssh-tunnel` | `container-debug` | `data-validation` | `shell-scripting` | `dns-networking` | `cron-scheduling` | `encoding-formats` | `makefile-build` | `skill-writer` | `skill-reviewer` | `skill-search-optimizer`

## License

Skills are published to ClawdHub under its registry terms. Source files in this repo are available for reference.

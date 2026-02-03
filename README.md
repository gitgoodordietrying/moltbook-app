# moltbook-app

Ten agent skills published to [ClawdHub](https://clawdhub.com), targeting infrastructure categories that the registry's gold rush phase left empty. Built through systematic ecosystem analysis, not guesswork.

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

## Why These Skills

The ClawdHub registry launched in late January 2026. Within its first week, it accumulated ~200+ skills — mostly Twitter CLI forks, crypto bots, and "coding agent" duplicates. Standard gold rush dynamics.

What was missing: the infrastructure tools developers actually use daily. SQL, testing, CI/CD, logging, security, profiling — zero coverage in any of these categories.

These ten skills fill those gaps. They're designed to age well: the content is useful whether the registry has 200 or 20,000 skills.

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

`docker-sandbox` | `csv-pipeline` | `api-dev` | `cicd-pipeline` | `sql-toolkit` | `test-patterns` | `log-analyzer` | `security-audit-toolkit` | `infra-as-code` | `perf-profiler`

## License

Skills are published to ClawdHub under its registry terms. Source files in this repo are available for reference.

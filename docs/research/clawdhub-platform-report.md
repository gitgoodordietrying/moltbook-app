# ClawdHub Platform - Comprehensive Technical Report

**Date**: 2026-02-03
**CLI Version**: molthub v0.3.1-beta.1
**Registry**: https://clawhub.ai (discovered via https://clawdhub.com/.well-known/clawdhub.json)

---

## 1. Architecture Overview

### Registry Discovery
- CLI defaults to `https://clawdhub.com`
- On first request, fetches `/.well-known/clawdhub.json` which returns:
  ```json
  {"apiBase":"https://clawhub.ai","authBase":"https://clawhub.ai","minCliVersion":"0.1.0","registry":"https://clawhub.ai"}
  ```
- Discovered registry is cached in `~/.config/clawdhub/config.json` (or OS equivalent)
- Legacy registry hosts (`auth.clawdhub.com`) are auto-migrated

### API Endpoints (v1)
| Route | Method | Purpose |
|---|---|---|
| `/api/v1/skills` | GET | List/explore skills (supports `limit`, `sort` params) |
| `/api/v1/skills` | POST | Publish skill (multipart form: payload JSON + file blobs) |
| `/api/v1/skills/:slug` | GET | Get skill metadata + latest version |
| `/api/v1/search` | GET | Vector search (`q`, `limit` params) |
| `/api/v1/resolve` | GET | Resolve version by content hash (`slug`, `hash` params) |
| `/api/v1/download` | GET | Download skill zip (`slug`, `version` params) |
| `/api/v1/stars` | POST | Star a skill |
| `/api/v1/stars` | DELETE | Unstar a skill |
| `/api/v1/whoami` | GET | Validate auth token |
| `/api/v1/souls` | - | Soul/profile management (undocumented) |

### Legacy API (still exists)
| Route | Purpose |
|---|---|
| `/api/search` | Old search endpoint |
| `/api/skill` | Old skill metadata |
| `/api/skill/resolve` | Old version resolution |
| `/api/download` | Old download |
| `/api/cli/whoami` | Old auth check |
| `/api/cli/upload-url` | Old upload (pre-signed URL flow) |
| `/api/cli/publish` | Old publish |
| `/api/cli/skill/delete` | Old delete |
| `/api/cli/telemetry/sync` | Telemetry reporting |

### HTTP Layer
- Uses `undici` for HTTP/2 support with 15s timeout
- Retry logic: 2 retries with exponential backoff for 429/5xx errors
- Bun runtime fallback: uses `curl` subprocess instead of `fetch`
- Auth: Bearer token in `Authorization` header
- **Known issue**: Does not respect `HTTP_PROXY`/`HTTPS_PROXY` env vars (undici limitation)

---

## 2. CLI Commands Reference

### Authentication
```
molthub login [--token <t>]    # Browser OAuth (GitHub) or direct token
molthub logout                 # Remove stored token
molthub whoami                 # Validate current token
molthub auth                   # Auth management subcommands
```
- Auth flow: Opens browser to `{site}/cli/auth` for GitHub OAuth
- Token stored in: `~/.config/clawdhub/config.json`

### Discovery
```
molthub search <query> [--limit <n>]     # Vector (semantic) search
molthub explore [--limit <n>] [--sort <order>] [--json]
```
Sort options: `newest` (default), `downloads`, `rating`/`stars`, `installs`, `installsAllTime`, `trending`

### Install/Update
```
molthub install <slug> [--version <v>] [--force]   # Install to ./skills/<slug>
molthub update [slug] [--all] [--version <v>] [--force]
molthub list                                        # List from lockfile
```
- Installs to `{workdir}/skills/{slug}/`
- Creates `.clawdhub/origin.json` per skill with registry, version, timestamp
- Lockfile at `{workdir}/.clawdhub/lock.json`
- Update uses content hashing to detect local modifications

### Publishing
```
molthub publish <path> --slug <s> --name <n> --version <v> [--changelog <text>] [--tags <csv>] [--fork-of <slug[@version]>]
molthub sync                    # Scan local skills, publish new/updated
molthub delete <slug>           # Soft-delete (owner/admin only)
molthub undelete <slug>         # Restore soft-deleted
```
- Requires SKILL.md in the folder
- Uploads all text files as multipart form
- Version must be valid semver
- Tags default to "latest"

### Social
```
molthub star <slug>
molthub unstar <slug>
```

### Global Options
```
--workdir <dir>     # Working directory (default: cwd)
--dir <dir>         # Skills directory relative to workdir (default: "skills")
--site <url>        # Site base URL
--registry <url>    # Registry API base URL
--no-input          # Disable interactive prompts
-V                  # Show CLI version
```

### Environment Variables
```
CLAWDHUB_SITE       # Override site URL
CLAWDHUB_REGISTRY   # Override registry URL
CLAWDHUB_WORKDIR    # Override working directory
CLAWHUB_DISABLE_TELEMETRY=1  # Disable telemetry
```

---

## 3. Skill Format

### Directory Structure
```
my-skill/
  SKILL.md              # Required - main skill file
  scripts/              # Optional - supporting scripts
    helper.sh
  .clawdhub/
    origin.json         # Auto-generated on install (not published)
```

### SKILL.md Frontmatter
```yaml
---
name: my-skill-slug
description: Short summary for search/display
metadata: {"clawdbot":{"emoji":"...","requires":{"bins":["cmd1"],"anyBins":["cmd1","cmd2"],"env":["API_KEY"],"config":["setting"]},"always":true,"skillKey":"custom-key","primaryEnv":"API_KEY","homepage":"https://...","os":["linux","darwin"],"install":[{"kind":"brew","formula":"pkg"},{"kind":"node","package":"pkg"},{"kind":"uv","module":"pkg"}]}}
---
```

### Metadata Schema
| Field | Type | Purpose |
|---|---|---|
| `always` | boolean | Always-active skill (loaded without explicit request) |
| `skillKey` | string | Custom activation key |
| `primaryEnv` | string | Primary env var needed |
| `emoji` | string | Display emoji |
| `homepage` | string | Project homepage URL |
| `os` | string[] | Supported OS list |
| `requires.bins` | string[] | Required binaries (all must exist) |
| `requires.anyBins` | string[] | Required binaries (at least one) |
| `requires.env` | string[] | Required environment variables |
| `requires.config` | string[] | Required config settings |
| `install` | array | Install specs: brew, node, go, uv |

### Text File Detection
- Only text files are included in published bundles
- Detection via file extension (known text types) and binary content check
- Excludes: `.clawdhub/`, `node_modules/`, `.git/`, etc. (via .gitignore patterns)

### Content Hashing
- Skills are fingerprinted by hashing all text file contents
- Used for update detection: `resolve` API matches hash to find which version matches local state
- Allows detecting local modifications before updates

---

## 4. Registry Statistics (as of 2026-02-03)

### Top Skills by Downloads
| Skill | Downloads | Stars | Description |
|---|---|---|---|
| `moltbook` | 38,764 | 25 | Social network for AI agents |
| `coding-agent` | 10,261 | 24 | Multi-agent coding (Codex, Claude, etc.) |
| `clawddocs` | 4,001 | 40 | Clawdbot documentation expert |
| `weather` | 5,291 | 11 | Weather forecasts (no API key) |
| `bird` | 5,090 | 20 | X/Twitter CLI |
| `humanizer` | 4,494 | 24 | Remove AI writing patterns |
| `proactive-agent` | 4,328 | 31 | Self-improving agent patterns |
| `remind-me` | 4,247 | 14 | Natural language reminders |
| `homeassistant` | 4,074 | 23 | Smart home control |
| `duckduckgo-search` | 78 | 0 | DDG web search (new, rising fast) |

### Categories Observed
1. **Productivity/Tools**: todo-management, obsidian-tasks, caldav-calendar, email, remind-me
2. **Search/Research**: brave-search, tavily-search, duckduckgo-search, deep-research, firecrawl
3. **Social/Communication**: bird (Twitter), slack, wacli (WhatsApp), moltbook-interact
4. **Media/Content**: nano-banana-pro (image gen), elevenlabs-tts, youtube-watcher, nano-pdf
5. **Developer Tools**: coding-agent, mcporter (MCP), clawdhub CLI, session-logs, file-search
6. **Smart Home/IoT**: homeassistant, sensibo, sonoscli
7. **AI Agent Meta**: proactive-agent, cognitive-memory, evolver, research-engine
8. **Finance**: yahoo-finance, simmer (prediction markets)
9. **Security**: secops-by-joes, wed (supply chain demo)
10. **Browser**: agent-browser, verify-on-browser, browser-use

### Gaps Identified
- **No CSV/data processing skill** - data ETL, transformation, analysis
- **No Docker/container management skill**
- **No database query skill** (except Snowflake MCP)
- **No CI/CD skill**
- **Limited testing/QA skills**
- **No API development/documentation skill**

---

## 5. Security Model

### Authentication
- GitHub OAuth via browser redirect
- Token stored in plaintext at OS-specific config path
- Token transmitted as Bearer header over HTTPS

### Skill Execution
- **No sandboxing**: Skills run with full agent permissions
- Skills are text instruction files - they tell the agent what to do
- The agent then executes commands with its own permissions
- This means a malicious skill can instruct the agent to:
  - Read/write any file the agent can access
  - Execute arbitrary commands
  - Exfiltrate data via network calls

### Moderation
- Report-based system
- Auto-hide at 3 reports
- Soft-delete available for owners/admins
- No pre-publication review

### Telemetry
- Syncs installed skill list (`/api/cli/telemetry/sync`)
- Sends: root workspace ID, label, installed skill slugs+versions
- Disable: `CLAWHUB_DISABLE_TELEMETRY=1`

---

## 6. Docker Sandbox Integration Notes

### Setup
- Docker Desktop 4.59.0 includes `docker sandbox` plugin v0.10.1
- `docker sandbox create claude <workspace-path>` creates an isolated VM
- Workspace mounted via virtiofs at the original host path (e.g., `/h/Projects/...`)
- Includes proxy at `host.docker.internal:3128` with custom CA cert

### Proxy Workaround
Node.js `undici`/`fetch` does NOT respect `HTTP_PROXY` env vars. Workaround:
```javascript
// /tmp/register-proxy.js - require hook
const proxy = process.env.HTTPS_PROXY || process.env.HTTP_PROXY;
if (proxy) {
  const { ProxyAgent } = require('<path-to-undici>');
  const agent = new ProxyAgent(proxy);
  const origFetch = globalThis.fetch;
  globalThis.fetch = function(url, opts = {}) {
    return origFetch(url, { ...opts, dispatcher: agent });
  };
}
```
Run with: `node -r /tmp/register-proxy.js $(which molthub) <command>`

---

## 7. Competitive Analysis Notes

### Strengths
1. **Simple skill format**: Just a SKILL.md + optional files. Low barrier to entry.
2. **Semantic search**: Vector-based search using embeddings provides good discoverability.
3. **Version management**: Proper semver, content hashing for change detection.
4. **Growing ecosystem**: ~200+ skills across many categories in first week.
5. **Well-known established maintainer** (steipete).

### Weaknesses
1. **No sandboxing**: Skills execute with full agent permissions.
2. **No pre-publication review**: Anyone can publish anything.
3. **Beta maturity**: Single version (0.3.1-beta.1), new ecosystem.
4. **Proxy/network issues**: undici doesn't respect HTTP_PROXY.
5. **Duplicate skills**: Multiple "YouTube Watcher", "coding-agent" variants with no dedup.
6. **No dependency management**: Skills can't declare dependencies on other skills.
7. **No testing framework**: No way to validate skills work before publishing.

### Comparison with Custom Agentic Orchestrator
| Aspect | ClawdHub | Custom Orchestrator |
|---|---|---|
| Skill format | Markdown + YAML frontmatter | Custom JSON/config |
| Distribution | Central registry | Self-hosted |
| Search | Vector (OpenAI embeddings) | - |
| Sandboxing | None | Configurable |
| Dependencies | None | DAG-based |
| Testing | None | Built-in validation |
| Auth | GitHub OAuth | Flexible |

# SecureGuard AI

> AI-powered security agent that detects vulnerabilities and delivers context-aware remediations directly inside the developer workflow — VS Code, Git, GitHub Copilot Chat, and Pull Requests.

---

## Why SecureGuard

Anthropic's Mythos model proved AI can find vulnerabilities faster than any human team. GitHub Copilot writes code faster than ever. But neither tool remediates security issues inside the developer workflow with company-policy grounding and a hard enforcement gate.

SecureGuard closes that gap. It pairs every finding with an AI-generated explanation, a working code patch, and a unit test — streamed directly into Copilot Chat or posted as a one-click PR suggestion. A policy gate blocks merges until issues are resolved or explicitly approved.

**Result:** 60–80% reduction in time from vulnerability detection to remediation implementation.

---

## What it covers

| # | Vulnerability | CWE | OWASP | Languages |
|---|---|---|---|---|
| 1 | SQL Injection | CWE-89 | A03:2021 | Python, TypeScript, JavaScript |
| 2 | Hardcoded secrets | CWE-798 | A02:2021 | Python, TypeScript, JavaScript |
| 3 | Broken authentication (weak JWT) | CWE-287 | A07:2021 | Python, TypeScript, JavaScript |
| 4 | Vulnerable dependencies | CWE-937 | A06:2021 | Python, TypeScript, JavaScript |
| 5 | Cross-site scripting (XSS) | CWE-79 | A03:2021 | Python, TypeScript, JavaScript |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Developer surfaces                       │
│                                                             │
│  VS Code + Copilot   Pre-commit hook   GitHub Actions PR    │
│  @secureguard chat   git commit block  policy gate          │
└──────────────┬──────────────┬──────────────┬───────────────┘
               │              │              │
               ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│              SecureGuard Core  (single Express service)      │
│                                                             │
│  Request router                                             │
│  ├── /scan    → Scanner only  (no LLM — fast, cheap)        │
│  ├── /copilot → Scanner + Claude API  (streamed)            │
│  └── /pr      → Scanner + Claude API  (batch, GitHub API)   │
│                                                             │
│  Scanner         Normalizer        Remediator               │
│  Semgrep         RawFinding[]   →  Claude Sonnet 4.6        │
│  Gitleaks        Finding[]         Prompt + parse           │
│  npm/pip audit                     Explanation + patch      │
│                                                             │
│  Registry             Policy engine      Audit log          │
│  OWASP + CWE map      BLOCK/PASS/OVERRIDE  SQLite           │
└─────────────────────────────────────────────────────────────┘
               │              │              │
               ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│                    External dependencies                     │
│                                                             │
│  Semgrep    Gitleaks    npm/pip audit    Claude API          │
│  (SAST)     (secrets)   (SCA / CVE)     (Anthropic)         │
└─────────────────────────────────────────────────────────────┘
```

### The LLM routing decision

The single most important architectural choice: Claude is only called on `/copilot` and `/pr` routes. The IDE background daemon and pre-commit hook use Semgrep and Gitleaks directly — no API call, no latency, no cost. This keeps file-save scanning under 2 seconds and commit hooks under 8 seconds.

### The three enforcement layers

| Layer | Trigger | Blocks on | Can bypass? | Purpose |
|---|---|---|---|---|
| VS Code daemon | File save | Nothing — advisory | N/A | Catch issues while coding |
| Pre-commit hook | `git commit` | CRITICAL only | `--no-verify` (logged) | Stop worst issues at source |
| PR gate | PR open / push | HIGH + CRITICAL | `security-override` label | Hard enforcement, audit trail |

---

## Prerequisites

```bash
# Node.js 20+
node --version   # v20.x.x

# Python 3.9+
python3 --version   # 3.9.x or higher

# Semgrep
pip install semgrep
semgrep --version

# Gitleaks
brew install gitleaks         # macOS
# or: https://github.com/gitleaks/gitleaks/releases

# pip-audit (Python dependency scanning)
pip install pip-audit

# Bandit (Python SAST)
pip install bandit
```

---

## Quick start

```bash
# 1. Clone and install
git clone https://github.com/your-org/secureguard.git
cd secureguard
npm install

# 2. Configure environment
cp packages/core/.env.example packages/core/.env
# Edit .env — add your ANTHROPIC_API_KEY and SECUREGUARD_API_KEY

# 3. Build
npm run build

# 4. Start the server
cd packages/core
npm run dev
# → SecureGuard running on http://localhost:3000

# 5. Verify
curl http://localhost:3000/health
# → { "status": "ok", "version": "1.0.0", "uptime": 3.2 }
```

Or with Docker:

```bash
cp packages/core/.env.example packages/core/.env
# Edit .env with your keys

docker-compose up
# → SecureGuard running on http://localhost:3000
```

---

## Configuration

All configuration via environment variables. Copy `.env.example` to `.env` and fill in values.

### Required

| Variable | Description |
|---|---|
| `ANTHROPIC_API_KEY` | Your Anthropic API key — get one at console.anthropic.com |
| `SECUREGUARD_API_KEY` | Shared secret for authenticating requests to the server |

### Optional (shown with defaults)

| Variable | Default | Description |
|---|---|---|
| `NODE_ENV` | `development` | Set to `production` for JSON logs and no stack traces in errors |
| `PORT` | `3000` | Port the Express server listens on |
| `LOG_LEVEL` | `info` | Pino log level: `trace`, `debug`, `info`, `warn`, `error` |
| `CLAUDE_MODEL` | `claude-sonnet-4-6` | Claude model for remediations |
| `DB_PATH` | `./secureguard.db` | SQLite audit database path |
| `SEMGREP_TIMEOUT_MS` | `30000` | Semgrep subprocess timeout |
| `GITLEAKS_TIMEOUT_MS` | `15000` | Gitleaks subprocess timeout |
| `AUDIT_TIMEOUT_MS` | `20000` | npm/pip audit timeout |
| `POLICY_BLOCK_ON` | `CRITICAL,HIGH` | Comma-separated severities that block PR merge |
| `POLICY_MAX_MEDIUM` | `10` | Block if MEDIUM finding count exceeds this |
| `POLICY_OVERRIDE_LABEL` | `security-override` | GitHub label name that allows override |
| `MAX_CONCURRENT_LLM_CALLS` | `5` | Max parallel Claude API calls per PR scan |

---

## Installing the VS Code extension

```bash
# Build the extension
cd packages/vscode-extension
npm install
npm run build

# Install locally
code --install-extension secureguard-1.0.0.vsix
```

Once installed:

- SecureGuard scans automatically on every file save (Python and TypeScript/JavaScript)
- Vulnerabilities appear as red squiggles with severity and CWE in the Problems panel
- Click a squiggle → Quick Fix → **View AI Fix** to open Copilot Chat with a pre-filled message
- Use `Ctrl+Shift+P` → **SecureGuard: Scan Repository** for a full repo scan

### Copilot Chat commands

With the extension installed, type `@secureguard` in GitHub Copilot Chat:

```
@secureguard scan this file
@secureguard scan repo
@secureguard /explain <findingId>
@secureguard what's vulnerable in this function?
@secureguard fix this
```

---

## Installing the pre-commit hook

```bash
# Install via CLI (recommended)
npx secureguard install-hook

# This writes .git/hooks/pre-commit:
# secureguard scan --staged --fail-on CRITICAL

# Or install manually
echo '#!/bin/sh
npx secureguard scan --staged --fail-on CRITICAL' > .git/hooks/pre-commit
chmod +x .git/hooks/pre-commit
```

The hook scans only staged files (fast, 3–8 seconds). It blocks commits if a CRITICAL severity finding is detected and prints the CWE, file, and line number with a one-line fix hint. Developers can run `@secureguard explain <findingId>` in Copilot Chat for the full AI remediation.

To bypass in an emergency: `git commit --no-verify` (this is logged in the audit trail).

---

## Adding the GitHub Actions workflow

Copy the workflow file to your repository:

```bash
cp .github/workflows/secureguard.yml your-repo/.github/workflows/secureguard.yml
```

Add secrets to your repository (Settings → Secrets and variables → Actions):

```
ANTHROPIC_API_KEY   your-anthropic-key
SECUREGUARD_API_KEY your-shared-secret
```

Configure branch protection (Settings → Branches → Add rule):
- Require status checks to pass before merging
- Add `secureguard/policy-gate` as a required status check

The workflow triggers on every pull request. It:

1. Installs Semgrep, Gitleaks, and pip-audit on the runner
2. Starts the SecureGuard server
3. Scans all changed files with the full Semgrep ruleset
4. Calls Claude to generate explanations and patches for HIGH/CRITICAL findings
5. Posts inline PR review comments with one-click fix suggestions
6. Sets a commit status (`secureguard/policy-gate`) to pass or fail
7. Fails the Actions check (exit code 1) if any HIGH or CRITICAL findings remain unresolved

To unblock a PR when a finding cannot be fixed immediately, a security team member adds the `security-override` label. This changes the gate to OVERRIDE (passing) and records the approver in the audit log.

---

## CLI reference

```bash
# Scan a file or directory
secureguard scan [path]
secureguard scan src/routes/users.py
secureguard scan src/                      # all Python + TypeScript files recursively

# Options
  --staged                                 # scan only git staged files (for pre-commit)
  --fail-on <severity>                     # exit 1 if this severity found (default: CRITICAL)
  --output <format>                        # text (default) or json
  --mode <mode>                            # fast (default) or full

# Get AI explanation for a specific finding
secureguard explain <findingId>

# Install pre-commit hook
secureguard install-hook

# Query audit log
secureguard audit
secureguard audit --since 2024-01-01
secureguard audit --repo /path/to/repo
```

---

## API reference

All endpoints require `X-SecureGuard-Token: <SECUREGUARD_API_KEY>` header except `/health`.

### `GET /health`

Returns server status. No authentication required.

```json
{
  "status": "ok",
  "version": "1.0.0",
  "uptime": 142.3,
  "timestamp": "2026-05-20T10:30:00.000Z"
}
```

### `GET /metrics`

Returns audit summary statistics. Used by the demo dashboard.

```json
{
  "totalScans": 847,
  "totalFindings": 1203,
  "findingsByCategory": {
    "CWE-89": 312,
    "CWE-798": 198,
    "CWE-287": 145,
    "CWE-937": 401,
    "CWE-79": 147
  },
  "findingsBySeverity": {
    "CRITICAL": 87,
    "HIGH": 334,
    "MEDIUM": 612,
    "LOW": 170
  },
  "blockRate": 0.23,
  "overrideRate": 0.04
}
```

### `POST /scan`

IDE background daemon, pre-commit hook, and CLI path. **No LLM called.**

Request:
```json
{
  "requestId": "uuid-v4",
  "source": "ide",
  "filePaths": ["src/routes/users.py"],
  "repoRoot": "/workspace/my-app",
  "language": "python",
  "scanMode": "fast"
}
```

Response:
```json
{
  "requestId": "uuid-v4",
  "findings": [
    {
      "findingId": "uuid-v4",
      "cweId": "CWE-89",
      "cweTitle": "SQL Injection",
      "severity": "CRITICAL",
      "filePath": "src/routes/users.py",
      "lineStart": 23,
      "lineEnd": 23,
      "snippet": "query = f\"SELECT * FROM users WHERE id = {user_id}\"",
      "scanner": "semgrep",
      "ruleId": "python.django.security.injection.sql.sql-injection"
    }
  ],
  "policyResult": {
    "decision": "PASS",
    "blockedFindings": [],
    "advisoryFindings": [...]
  },
  "scanDurationMs": 1847
}
```

### `POST /copilot`

GitHub Copilot Chat path. Calls Claude API. Returns SSE stream.

Request: Copilot Extension skill payload (forwarded automatically by GitHub Copilot).

Response: `text/event-stream` — tokens streamed as `data: {token}\n\n`. Includes finding details, AI explanation, fixed code, and unit test in GitHub Flavored Markdown.

### `POST /pr`

GitHub Actions PR bot path. Full deep scan + AI remediations + GitHub API calls.

Request:
```json
{
  "requestId": "uuid-v4",
  "source": "pr",
  "prNumber": 42,
  "githubRepo": "your-org/your-repo",
  "githubToken": "ghp_...",
  "changedFiles": ["src/routes/users.py", "src/routes/auth.ts"],
  "repoRoot": "/workspace",
  "scanMode": "full"
}
```

Response:
```json
{
  "requestId": "uuid-v4",
  "decision": "BLOCK",
  "findingsCount": 3,
  "blockedFindingsCount": 2,
  "remediationsPosted": 2,
  "auditId": "uuid-v4"
}
```

---

## Extending SecureGuard — adding a new vulnerability category

1. Add the CWE entry to `packages/core/remediator-registry.json`:

```json
{
  "cweId": "CWE-611",
  "name": "XML External Entity (XXE)",
  "owaspRef": "A05:2021",
  "owaspUrl": "https://owasp.org/Top10/A05_2021-Security_Misconfiguration/",
  "severity": "HIGH",
  "guidance": "Disable external entity processing in your XML parser. In Python: use defusedxml instead of stdlib xml. In Node.js: set resolveEntities: false.",
  "fixTemplate": "Replace xml.etree.ElementTree with defusedxml.ElementTree in Python. Set { resolveEntities: false } in Node.js XML parsers.",
  "references": [
    "https://owasp.org/Top10/A05_2021-Security_Misconfiguration/",
    "https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html",
    "https://cwe.mitre.org/data/definitions/611.html"
  ],
  "languages": ["python", "typescript", "javascript"]
}
```

2. Add a Semgrep rule to `packages/core/semgrep-rules/`:

```yaml
# semgrep-rules/xxe.yaml
rules:
  - id: secureguard-python-xxe
    languages: [python]
    severity: ERROR
    message: "XXE (CWE-611): stdlib xml.etree.ElementTree is vulnerable to XXE. Use defusedxml."
    pattern: |
      import xml.etree.ElementTree
    metadata:
      cwe: CWE-611
      owasp: A05:2021
```

3. Add the rule-to-CWE mapping in `normalizer.ts`:

```typescript
'python.lang.security.audit.xml-stdlib': {
  cweId: 'CWE-611',
  cweTitle: 'XML External Entity',
  severity: Severity.HIGH
}
```

That's it. No code changes anywhere else — the registry, normalizer, and remediator pick it up automatically.

---

## Audit log

The audit log records every scan — what was found, what decision was made, whether Claude was called, and who triggered it.

```bash
# Query via CLI
secureguard audit --since 2026-01-01

# Query directly (SQLite)
sqlite3 secureguard.db "
  SELECT
    source,
    developer,
    findings_count,
    policy_decision,
    datetime(created_at) as scanned_at
  FROM audit_records
  ORDER BY created_at DESC
  LIMIT 20;
"
```

**Schema tables:**

- `audit_records` — one row per scan: source, developer, repo, finding counts by severity, policy decision, LLM usage
- `findings` — one row per finding: CWE, severity, file, line, scanner
- `remediations` — one row per AI remediation: explanation, patch, test stub, token counts, latency

**Key metrics you can derive:**

```sql
-- Bypass rate (how often devs use --no-verify)
SELECT
  COUNT(*) FILTER (WHERE source = 'commit' AND policy_decision = 'BLOCK') as bypassed
FROM audit_records;

-- Average time from CRITICAL finding to PASS (proxy for remediation time)
-- Join consecutive audit records for the same repo and CWE

-- Most common vulnerabilities
SELECT cwe_id, COUNT(*) as count
FROM findings
GROUP BY cwe_id
ORDER BY count DESC;

-- LLM usage and cost tracking
SELECT
  DATE(created_at) as day,
  SUM(remediations_generated) as remediations,
  SUM(llm_called) as llm_calls
FROM audit_records
GROUP BY day
ORDER BY day DESC;
```

---

## Production deployment

### Prerequisites for production

- Node.js 20+ on your server or container
- Postgres (replace SQLite — schema is compatible, change `better-sqlite3` to `pg`)
- A secrets manager for `ANTHROPIC_API_KEY` (AWS Secrets Manager, Azure Key Vault, etc.)
- A reverse proxy (nginx or a load balancer) in front of the Express server
- Set `NODE_ENV=production` — this enables JSON logging and removes stack traces from error responses

### Environment-specific configuration

```bash
# Production .env (via secrets manager, not a file)
NODE_ENV=production
PORT=3000
LOG_LEVEL=info
ANTHROPIC_API_KEY=<from-secrets-manager>
SECUREGUARD_API_KEY=<from-secrets-manager>
DB_PATH=/data/secureguard.db         # or postgres connection string
POLICY_BLOCK_ON=CRITICAL,HIGH
MAX_CONCURRENT_LLM_CALLS=5
```

### Docker

```bash
docker build -t secureguard-core ./packages/core
docker run \
  -p 3000:3000 \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  -e SECUREGUARD_API_KEY=$SECUREGUARD_API_KEY \
  -v secureguard-data:/data \
  secureguard-core
```

### Kubernetes (production pattern)

```yaml
# Deploy as a Deployment with 2+ replicas behind a ClusterIP Service
# Mount ANTHROPIC_API_KEY from a Kubernetes Secret
# Use a PersistentVolumeClaim for the SQLite data directory
# Or replace SQLite with a managed Postgres instance and remove the PVC
# Add a HorizontalPodAutoscaler targeting CPU utilization at 60%
# Configure liveness probe: GET /health
# Configure readiness probe: GET /health with initialDelaySeconds: 5
```

### LLM gateway (enterprise pattern)

In production, route all Claude API calls through a centralized LLM gateway (Azure API Management, AWS API Gateway, or Tyk) for:

- **Rate limiting** — enforce per-team or per-developer Claude API budgets
- **Audit logging** — record every prompt and response for compliance
- **Model switching** — change from `claude-sonnet-4-6` to a different model without code changes
- **Cost attribution** — charge Claude API usage back to the team or project

Set `CLAUDE_API_BASE_URL` to your gateway URL. The `remediator.ts` module reads this env var when constructing the Anthropic client.

---

## Security considerations

**API keys:** `ANTHROPIC_API_KEY` and `SECUREGUARD_API_KEY` must never be committed to source control. Use environment variables or a secrets manager. The logger redacts these keys from all log output even if accidentally referenced.

**Code snippets:** Vulnerable code snippets sent to the Claude API are truncated to 500 characters. Do not send entire files. Ensure your Anthropic API usage is covered by your organization's data processing agreement.

**Scanner subprocesses:** All scanner commands use `spawn` with argument arrays, never shell string interpolation. File paths from user input are never concatenated into command strings.

**Branch protection:** The PR policy gate only works if branch protection is enabled in GitHub. Without the required status check, developers can merge without the gate passing.

**Override audit trail:** Every use of the `security-override` label is recorded in the audit log with the approver's GitHub username and timestamp. Security teams should review override usage weekly.

---

## Troubleshooting

**Semgrep not found:**
```bash
pip install semgrep
# verify: semgrep --version
```

**Gitleaks not found:**
```bash
brew install gitleaks   # macOS
# Linux: download from https://github.com/gitleaks/gitleaks/releases
# verify: gitleaks version
```

**Claude API returning 401:**
Check `ANTHROPIC_API_KEY` in your `.env` file. Verify the key is active at console.anthropic.com.

**Scanner timeout in CI:**
Increase `SEMGREP_TIMEOUT_MS` in the Actions environment. Default is 30 seconds which may be too short for large repos. Set to `60000` for repos over 100k lines.

**Pre-commit hook not running:**
```bash
ls -la .git/hooks/pre-commit   # check file exists
chmod +x .git/hooks/pre-commit  # check executable
```

**VS Code extension not showing squiggles:**
Check the SecureGuard server is running on port 3000. Open the Output panel → SecureGuard to see connection status and errors.

**PR gate not appearing in required status checks:**
The `secureguard/policy-gate` status check only appears after the Actions workflow has run at least once on a PR. Open a test PR, let it run, then add it to branch protection rules.

---

## Project structure

```
secureguard/
├── packages/
│   ├── core/                    Main Express service
│   │   ├── src/
│   │   │   ├── server.ts        Express app, middleware, graceful shutdown
│   │   │   ├── router.ts        Route handlers for /scan /copilot /pr
│   │   │   ├── scanner.ts       Semgrep, Gitleaks, npm/pip audit runners
│   │   │   ├── normalizer.ts    RawFinding → Finding, CWE mapping, dedup
│   │   │   ├── remediator.ts    Claude API, prompt builder, response parser
│   │   │   ├── registry.ts      OWASP/CWE registry loader and lookup
│   │   │   ├── policy.ts        Severity rules, GitHub status API
│   │   │   ├── audit.ts         SQLite write/read/query
│   │   │   ├── logger.ts        Pino structured logger with redaction
│   │   │   ├── errors.ts        Custom error classes with codes
│   │   │   ├── types.ts         All shared TypeScript interfaces
│   │   │   ├── middleware/      auth, errorHandler, requestLogger, rateLimiter
│   │   │   └── formatters/      lsp, cli, copilot, pr output formatters
│   │   ├── remediator-registry.json    5 CWE entries with OWASP guidance
│   │   └── semgrep-rules/       Custom org-specific Semgrep YAML rules
│   ├── vscode-extension/        VS Code LSP client + Copilot Chat handler
│   ├── cli/                     secureguard CLI + pre-commit hook installer
│   └── github-action/           GitHub Actions PR bot
├── .github/workflows/
│   └── secureguard.yml          PR trigger workflow
├── demo-app/                    Intentionally vulnerable app for demo
├── docker-compose.yml
└── README.md
```

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/add-xxe-detection`
3. Add your CWE entry to `remediator-registry.json`
4. Add the Semgrep rule to `semgrep-rules/`
5. Add the rule mapping to `normalizer.ts`
6. Add tests covering the new vulnerability detection
7. Run `npm test` — all tests must pass
8. Open a PR — SecureGuard will scan it automatically

---

## License

MIT — see LICENSE file.

---

## Acknowledgements

Built on the shoulders of:

- [Semgrep](https://semgrep.dev) — open-source SAST engine with 3,000+ security rules
- [Gitleaks](https://github.com/gitleaks/gitleaks) — secrets detection in git history and staged files
- [OWASP Top 10](https://owasp.org/Top10/) — the industry standard for application security risks
- [Anthropic Claude](https://anthropic.com) — the AI that turns a finding into a fix

---

*SecureGuard AI — because finding a vulnerability is only half the job.*

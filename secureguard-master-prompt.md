# SecureGuard AI — Master Build Prompt
## For Claude Opus 4.6 Agent in GitHub Copilot (GHCP)

---

> **HOW TO USE THIS PROMPT**
> Open GitHub Copilot Chat in VS Code. Switch model to Claude Opus 4.6.
> Paste this entire document as your first message.
> The agent will scaffold, build, and wire the complete application autonomously.
> When it asks a clarifying question, answer it and it will continue.

---

## AGENT INSTRUCTIONS — READ FIRST

You are Claude Opus 4.6, an autonomous software engineering agent operating inside GitHub Copilot in VS Code. Your task is to build **SecureGuard AI** — a production-grade, enterprise-ready AI security agent — from scratch in this workspace.

You have full access to:
- The file system (read, write, create, delete)
- The terminal (run commands, install packages, execute scripts)
- The editor (create and modify any file)

**Work autonomously. Do not ask for permission before each step. Make sensible decisions, document them inline, and keep building. Only pause if you hit a genuine blocker that requires a secret, credential, or architectural decision that changes the entire design.**

Build the complete application in this order. Do not skip steps. Do not stub functions — implement them fully.

---

## SECTION 1 — PROJECT OVERVIEW AND GOALS

### What you are building
SecureGuard AI is a three-surface AI security agent that detects vulnerabilities and returns AI-generated, context-aware remediations grounded in OWASP and company policy. It runs:
- **Surface 1**: VS Code background daemon (Language Server Protocol) — scans on file save, shows inline diagnostics, no LLM
- **Surface 2**: Git pre-commit hook — scans staged files, blocks commit on CRITICAL severity, no LLM
- **Surface 3**: GitHub Copilot Chat (@secureguard) — on-demand scan + AI remediation via Claude API, streamed
- **Surface 4**: GitHub Actions PR bot — deep scan on PR open, inline fix suggestions, policy gate that blocks merge on HIGH/CRITICAL

### Architecture decision — single service
All four surfaces call one Express server (packages/core). The router decides whether to invoke Claude (LLM path: Copilot Chat + PR) or return scanner results directly (no-LLM path: LSP + CLI + pre-commit). This keeps IDE scanning fast (<2s) and cheap.

### Languages supported
TypeScript/JavaScript and Python (both required by hackathon spec).

### Five vulnerability categories (required deliverables)
1. SQL Injection (CWE-89) — OWASP A03:2021
2. Hardcoded secrets/credentials (CWE-798) — OWASP A02:2021
3. Broken authentication — weak JWT, missing expiry (CWE-287) — OWASP A07:2021
4. Vulnerable dependencies — known CVEs in npm/pip packages (CWE-937) — OWASP A06:2021
5. Cross-site scripting — unescaped output (CWE-79) — OWASP A03:2021

---

## SECTION 2 — REPOSITORY STRUCTURE

Create this exact directory structure. Every file listed must be created and fully implemented:

```
secureguard/
├── packages/
│   ├── core/
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── router.ts
│   │   │   ├── scanner.ts
│   │   │   ├── normalizer.ts
│   │   │   ├── remediator.ts
│   │   │   ├── registry.ts
│   │   │   ├── policy.ts
│   │   │   ├── audit.ts
│   │   │   ├── logger.ts
│   │   │   ├── errors.ts
│   │   │   ├── types.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── errorHandler.ts
│   │   │   │   ├── requestLogger.ts
│   │   │   │   └── rateLimiter.ts
│   │   │   └── formatters/
│   │   │       ├── lsp.ts
│   │   │       ├── cli.ts
│   │   │       ├── copilot.ts
│   │   │       └── pr.ts
│   │   ├── remediator-registry.json
│   │   ├── semgrep-rules/
│   │   │   ├── sql-injection.yaml
│   │   │   ├── hardcoded-secrets.yaml
│   │   │   ├── weak-auth.yaml
│   │   │   └── xss.yaml
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── .env.example
│   │
│   ├── vscode-extension/
│   │   ├── src/
│   │   │   ├── extension.ts
│   │   │   ├── lspClient.ts
│   │   │   └── copilotHandler.ts
│   │   └── package.json
│   │
│   ├── cli/
│   │   ├── src/
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   └── github-action/
│       ├── src/
│       │   └── action.ts
│       ├── action.yml
│       └── Dockerfile
│
├── .github/
│   └── workflows/
│       └── secureguard.yml
│
├── demo-app/
│   ├── python/
│   │   ├── routes/
│   │   │   ├── users.py          ← SQL injection demo
│   │   │   └── auth.py           ← weak auth demo
│   │   └── requirements.txt      ← vulnerable deps demo
│   └── typescript/
│       ├── routes/
│       │   ├── users.ts          ← SQL injection demo
│       │   └── render.ts         ← XSS demo
│       └── package.json          ← vulnerable deps demo
│
├── docker-compose.yml
├── package.json                  ← workspace root
└── README.md
```

---

## SECTION 3 — TYPES (implement first, everything depends on this)

### File: packages/core/src/types.ts

Implement all of these interfaces and enums with full JSDoc comments:

```typescript
// Severity levels — used for routing, blocking, and display
export enum Severity {
  CRITICAL = 'CRITICAL',
  HIGH = 'HIGH',
  MEDIUM = 'MEDIUM',
  LOW = 'LOW',
  INFO = 'INFO'
}

// Source surface that triggered the scan
export enum ScanSource {
  IDE = 'ide',
  COMMIT = 'commit',
  PR = 'pr',
  CLI = 'cli',
  COPILOT = 'copilot'
}

// Scanner that produced the raw finding
export enum ScannerTool {
  SEMGREP = 'semgrep',
  GITLEAKS = 'gitleaks',
  NPM_AUDIT = 'npm_audit',
  PIP_AUDIT = 'pip_audit',
  BANDIT = 'bandit'
}

// Scan mode — fast for IDE/commit, full for PR/copilot
export enum ScanMode {
  FAST = 'fast',
  FULL = 'full'
}

export interface ScanRequest {
  requestId: string          // uuid v4
  source: ScanSource
  filePaths: string[]
  repoRoot: string
  language?: 'python' | 'typescript' | 'javascript' | 'auto'
  scanMode: ScanMode
  prNumber?: number
  githubRepo?: string        // owner/repo format
  githubToken?: string
  requestedAt: Date
}

export interface RawFinding {
  ruleId: string
  filePath: string
  lineStart: number
  lineEnd: number
  columnStart: number
  columnEnd: number
  message: string
  snippet: string
  scanner: ScannerTool
  extra?: Record<string, unknown>
}

export interface Finding {
  findingId: string          // uuid v4
  requestId: string          // FK to ScanRequest
  cweId: string              // e.g. 'CWE-89'
  cweTitle: string           // e.g. 'SQL Injection'
  severity: Severity
  filePath: string
  lineStart: number
  lineEnd: number
  snippet: string            // vulnerable code excerpt
  scanner: ScannerTool
  ruleId: string
  detectedAt: Date
}

export interface RegistryEntry {
  cweId: string
  name: string
  owaspRef: string           // e.g. 'A03:2021'
  owaspUrl: string
  severity: Severity
  guidance: string           // company-specific secure dev guidance
  fixTemplate: string        // language-agnostic fix description
  references: string[]
  languages: string[]        // which languages this applies to
}

export interface Remediation {
  remediationId: string      // uuid v4
  findingId: string          // FK to Finding
  explanation: string        // plain English for the developer
  patch: string              // unified diff or fixed code block
  testStub: string           // unit test that would catch this
  model: string              // claude-sonnet-4-6
  promptTokens: number
  completionTokens: number
  generatedAt: Date
  latencyMs: number
}

export enum PolicyDecision {
  PASS = 'PASS',
  BLOCK = 'BLOCK',
  OVERRIDE = 'OVERRIDE'
}

export interface PolicyResult {
  decision: PolicyDecision
  blockedFindings: Finding[]
  advisoryFindings: Finding[]
  overrideLabel?: string
  overrideApprover?: string
  evaluatedAt: Date
}

export interface AuditRecord {
  auditId: string            // uuid v4
  requestId: string
  source: ScanSource
  developer?: string         // git config user.email if available
  repoPath: string
  scanDurationMs: number
  findingsCount: number
  findingsBySeverity: Record<Severity, number>
  policyResult: PolicyResult
  remediationsGenerated: number
  llmCalled: boolean
  createdAt: Date
}

export interface SecureGuardError {
  code: string               // e.g. 'SCANNER_TIMEOUT', 'LLM_UNAVAILABLE'
  message: string
  requestId?: string
  retryable: boolean
  context?: Record<string, unknown>
}
```

---

## SECTION 4 — LOGGER (implement second, everything else uses it)

### File: packages/core/src/logger.ts

Use the `pino` library. Implement structured JSON logging with:
- Log levels: trace, debug, info, warn, error, fatal
- Always include: timestamp (ISO 8601), level, requestId (if available), source, message
- Never log: ANTHROPIC_API_KEY, GITHUB_TOKEN, file snippets containing secrets, PII
- Production mode: JSON output. Development mode: pino-pretty formatted output
- Export a child logger factory: `createRequestLogger(requestId: string)`

```typescript
import pino from 'pino'

const isDev = process.env.NODE_ENV !== 'production'

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  transport: isDev ? { target: 'pino-pretty', options: { colorize: true } } : undefined,
  base: { service: 'secureguard-core', version: process.env.npm_package_version },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ['*.apiKey', '*.token', '*.secret', '*.password', 'req.headers.authorization'],
    censor: '[REDACTED]'
  }
})

export function createRequestLogger(requestId: string) {
  return logger.child({ requestId })
}
```

---

## SECTION 5 — ERROR HANDLING

### File: packages/core/src/errors.ts

Define all custom error classes. Every error must have a `code`, `message`, `retryable` flag, and optional `context`. These are the error codes you must implement:

```typescript
export class SecureGuardBaseError extends Error {
  code: string
  retryable: boolean
  context?: Record<string, unknown>
  requestId?: string

  constructor(code: string, message: string, retryable: boolean, context?: Record<string, unknown>) {
    super(message)
    this.name = 'SecureGuardBaseError'
    this.code = code
    this.retryable = retryable
    this.context = context
  }
}

// Implement these specific error classes extending SecureGuardBaseError:
export class ScannerTimeoutError         // code: SCANNER_TIMEOUT, retryable: true
export class ScannerNotFoundError        // code: SCANNER_NOT_FOUND, retryable: false
export class ScannerParseError           // code: SCANNER_PARSE_ERROR, retryable: false
export class RegistryNotFoundError       // code: REGISTRY_ENTRY_NOT_FOUND, retryable: false
export class LLMUnavailableError         // code: LLM_UNAVAILABLE, retryable: true
export class LLMRateLimitError           // code: LLM_RATE_LIMIT, retryable: true — include retryAfterMs
export class LLMContextLengthError       // code: LLM_CONTEXT_TOO_LONG, retryable: false
export class PolicyViolationError        // code: POLICY_VIOLATION, retryable: false
export class GitHubAPIError              // code: GITHUB_API_ERROR, retryable: true
export class InvalidRequestError         // code: INVALID_REQUEST, retryable: false
export class AuditWriteError             // code: AUDIT_WRITE_ERROR, retryable: true
```

### File: packages/core/src/middleware/errorHandler.ts

Express error handling middleware that:
- Catches all thrown errors (SecureGuardBaseError and unknown errors)
- Logs the error with full context using the request logger
- Returns structured JSON error responses — NEVER expose stack traces in production
- Maps error codes to HTTP status codes:
  - INVALID_REQUEST → 400
  - LLM_RATE_LIMIT → 429 with Retry-After header
  - POLICY_VIOLATION → 422
  - SCANNER_TIMEOUT, LLM_UNAVAILABLE → 503 with Retry-After: 30
  - Unknown errors → 500
- In development mode: include stack trace in response for debugging

---

## SECTION 6 — SCANNER MODULE

### File: packages/core/src/scanner.ts

Implement a `ScannerService` class with these methods. Every method must have full error handling, timeout enforcement, and structured logging.

```typescript
export class ScannerService {

  // Run Semgrep with appropriate config based on scanMode
  // fast mode: --config p/python --config p/javascript --config ./semgrep-rules/
  // full mode: additionally --config p/owasp-top-ten --config p/ci
  // Always: --json --timeout 30 --max-memory 512
  // Throws ScannerTimeoutError if process exceeds timeoutMs
  // Throws ScannerParseError if JSON output is malformed
  async runSemgrep(filePaths: string[], scanMode: ScanMode, timeoutMs: number): Promise<RawFinding[]>

  // Run Gitleaks for secrets detection
  // Command: gitleaks detect --source . --report-format json --report-path /tmp/gitleaks-{requestId}.json
  // Parse report file, clean up temp file in finally block
  // Map gitleaks output fields to RawFinding schema
  async runGitleaks(repoRoot: string, filePaths: string[], timeoutMs: number): Promise<RawFinding[]>

  // Run npm audit for JavaScript/TypeScript projects
  // Command: npm audit --json
  // Parse advisories, map CVE → CWE-937
  // Only run if package-lock.json exists in repoRoot
  async runNpmAudit(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // Run pip-audit for Python projects
  // Command: pip-audit --format json
  // Only run if requirements.txt or Pipfile.lock exists
  async runPipAudit(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // Orchestrator: runs appropriate scanners in parallel based on source + scanMode
  // IDE/fast: semgrep fast + gitleaks only
  // commit/fast: semgrep fast + gitleaks + audit
  // PR/full: all scanners concurrently, longer timeouts
  // Returns merged, deduplicated RawFinding[]
  // Never throws — catches scanner errors individually, logs them, continues with remaining results
  async scan(request: ScanRequest, log: Logger): Promise<RawFinding[]>
}
```

**Important implementation details:**
- Use Node.js `child_process.spawn` (not exec) to avoid shell injection
- Always pass file paths as array arguments, never string interpolation
- Set `maxBuffer: 10 * 1024 * 1024` (10MB) to handle large scan outputs
- Use `Promise.race` against a timeout promise for each scanner
- Log scanner start, completion, and finding count at INFO level
- Log scanner errors at WARN level (not ERROR — partial results are acceptable)
- Clean up any temp files in `finally` blocks

---

## SECTION 7 — NORMALIZER MODULE

### File: packages/core/src/normalizer.ts

```typescript
export class NormalizerService {

  // Convert RawFinding[] → Finding[]
  // Rules:
  // 1. Map Semgrep rule IDs to CWE IDs using RULE_TO_CWE_MAP
  // 2. Map Gitleaks rule IDs to CWE-798
  // 3. Map npm/pip audit entries to CWE-937
  // 4. Deduplicate: same (filePath + lineStart + cweId) = one finding, keep highest severity
  // 5. Generate findingId (uuid v4) for each unique finding
  // 6. Truncate snippet to 500 chars max
  // 7. Normalize filePath to be relative to repoRoot
  normalize(raw: RawFinding[], requestId: string, repoRoot: string): Finding[]
}

// Define this mapping — must cover all Semgrep rules used:
const RULE_TO_CWE_MAP: Record<string, { cweId: string; cweTitle: string; severity: Severity }> = {
  'python.lang.security.audit.dangerous-sys-exec.dangerous-sys-exec': { cweId: 'CWE-78',  cweTitle: 'OS Command Injection',    severity: Severity.CRITICAL },
  'python.sqlalchemy.security.sqlalchemy-execute-raw-query':           { cweId: 'CWE-89',  cweTitle: 'SQL Injection',           severity: Severity.CRITICAL },
  'python.django.security.injection.sql.sql-injection':                { cweId: 'CWE-89',  cweTitle: 'SQL Injection',           severity: Severity.CRITICAL },
  'javascript.sequelize.security.audit.sequelize-injection':           { cweId: 'CWE-89',  cweTitle: 'SQL Injection',           severity: Severity.CRITICAL },
  'python.jwt.security.jwt-none-alg':                                  { cweId: 'CWE-287', cweTitle: 'Broken Authentication',   severity: Severity.HIGH },
  'javascript.express.security.audit.xss.pug-var-in-code-block':       { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',   severity: Severity.HIGH },
  'typescript.react.security.audit.react-dangerouslysetinnerhtml':     { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',   severity: Severity.HIGH },
  // add more as needed — agent should expand this map based on Semgrep docs
}
```

---

## SECTION 8 — REGISTRY MODULE

### File: packages/core/src/registry.ts

```typescript
export class RegistryService {
  private entries: Map<string, RegistryEntry>

  // Load remediator-registry.json at startup
  // Validate all required fields exist for each entry
  // Throw RegistryNotFoundError with clear message if file missing or malformed
  // Log entry count at INFO on successful load
  async load(registryPath: string): Promise<void>

  // Look up by CWE ID — throws RegistryNotFoundError if not found
  lookup(cweId: string): RegistryEntry

  // Return all entries
  all(): RegistryEntry[]
}
```

### File: packages/core/remediator-registry.json

Create the full registry with all 5 required vulnerability categories. Each entry must have every field populated — no empty strings, no TODO placeholders:

```json
{
  "entries": [
    {
      "cweId": "CWE-89",
      "name": "SQL Injection",
      "owaspRef": "A03:2021",
      "owaspUrl": "https://owasp.org/Top10/A03_2021-Injection/",
      "severity": "CRITICAL",
      "guidance": "Never concatenate user input into SQL strings. Use parameterized queries or an ORM. If you must use raw SQL, use the database driver's parameter binding (e.g. cursor.execute(sql, params) in Python, db.query(sql, [params]) in Node.js). Validate and sanitize all inputs even when using parameterized queries. Apply principle of least privilege to database accounts — the app user should not have DROP or ALTER permissions.",
      "fixTemplate": "Replace string concatenation with parameterized query. Pass user input as separate parameter array. Example: cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,)) in Python. db.query('SELECT * FROM users WHERE id = $1', [userId]) in Node.js.",
      "references": [
        "https://owasp.org/Top10/A03_2021-Injection/",
        "https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/89.html"
      ],
      "languages": ["python", "typescript", "javascript"]
    },
    {
      "cweId": "CWE-798",
      "name": "Hardcoded credentials",
      "owaspRef": "A02:2021",
      "owaspUrl": "https://owasp.org/Top10/A02_2021-Cryptographic_Failures/",
      "severity": "CRITICAL",
      "guidance": "Never hardcode secrets, API keys, passwords, tokens, or connection strings in source code. Use environment variables accessed via process.env or os.environ. For production, use a secrets manager (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault). Rotate any secret that has been committed immediately — treat it as compromised even if the commit was private. Add secret scanning to pre-commit hooks and CI to prevent recurrence.",
      "fixTemplate": "Move the hardcoded value to an environment variable. Load with process.env.SECRET_NAME (Node.js) or os.environ.get('SECRET_NAME') (Python). Add the variable name to .env.example with a placeholder value. Add the actual value to .env (gitignored) and your secrets manager.",
      "references": [
        "https://owasp.org/Top10/A02_2021-Cryptographic_Failures/",
        "https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/798.html"
      ],
      "languages": ["python", "typescript", "javascript"]
    },
    {
      "cweId": "CWE-287",
      "name": "Broken authentication",
      "owaspRef": "A07:2021",
      "owaspUrl": "https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/",
      "severity": "HIGH",
      "guidance": "JWT tokens must always specify an expiry (exp claim). Never use the 'none' algorithm. Always verify the signature with a strong secret (HS256 minimum, RS256 preferred). Validate issuer (iss) and audience (aud) claims. Implement token refresh patterns — short-lived access tokens (15 min) with longer-lived refresh tokens. Never store JWTs in localStorage — use httpOnly cookies.",
      "fixTemplate": "Add expiresIn option to jwt.sign(). Validate exp in jwt.verify(). Reject tokens with alg: none. Use strong secret from environment variable with minimum 32 bytes entropy. Example: jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '15m', algorithm: 'HS256' })",
      "references": [
        "https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/",
        "https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/287.html"
      ],
      "languages": ["python", "typescript", "javascript"]
    },
    {
      "cweId": "CWE-937",
      "name": "Vulnerable dependency",
      "owaspRef": "A06:2021",
      "owaspUrl": "https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/",
      "severity": "HIGH",
      "guidance": "Keep all dependencies updated. Run npm audit or pip-audit in CI on every PR. Pin dependency versions in lock files and commit them. Subscribe to security advisories for critical dependencies. For high-severity CVEs, update or replace the dependency immediately — do not defer. Consider using Dependabot or Renovate for automated dependency updates.",
      "fixTemplate": "Update the vulnerable package to the patched version specified in the advisory. Run: npm update package-name@safe-version (Node.js) or pip install package-name==safe-version (Python). Verify the update resolves the advisory: npm audit or pip-audit. Update lock file and commit.",
      "references": [
        "https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/",
        "https://cheatsheetseries.owasp.org/cheatsheets/Vulnerable_Dependency_Management_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/937.html"
      ],
      "languages": ["python", "typescript", "javascript"]
    },
    {
      "cweId": "CWE-79",
      "name": "Cross-site scripting (XSS)",
      "owaspRef": "A03:2021",
      "owaspUrl": "https://owasp.org/Top10/A03_2021-Injection/",
      "severity": "HIGH",
      "guidance": "Never insert unescaped user input into HTML. Use your framework's built-in escaping (React JSX auto-escapes, Jinja2 auto-escapes). Avoid innerHTML, dangerouslySetInnerHTML, document.write. Implement Content-Security-Policy headers. Sanitize rich text input with a trusted library (DOMPurify). Set httpOnly and SameSite on cookies to limit XSS impact.",
      "fixTemplate": "Use framework-safe rendering instead of raw HTML injection. In React: use JSX ({variable}) not dangerouslySetInnerHTML. In vanilla JS: use textContent not innerHTML. In Python/Jinja2: use {{ variable }} not {{ variable | safe }}. If rich HTML is required, sanitize with DOMPurify.sanitize(input) before rendering.",
      "references": [
        "https://owasp.org/Top10/A03_2021-Injection/",
        "https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/79.html"
      ],
      "languages": ["python", "typescript", "javascript"]
    }
  ]
}
```

---

## SECTION 9 — AI REMEDIATOR MODULE

### File: packages/core/src/remediator.ts

This is the most critical module — where Claude generates the fix. Implement the full `RemediatorService` class:

```typescript
export class RemediatorService {

  // Build the Claude prompt from finding + registry entry + surrounding code context
  // The prompt must:
  // 1. Identify the specific vulnerability in the ACTUAL code snippet (not generic)
  // 2. Include the registry guidance as grounding
  // 3. Request three specific outputs: explanation, patch, test stub
  // 4. Specify output format as structured XML tags for reliable parsing
  // 5. Keep total prompt under 4000 tokens to leave room for response
  buildPrompt(finding: Finding, entry: RegistryEntry, surroundingCode?: string): string

  // Call Claude API with retry logic
  // Model: claude-sonnet-4-6 (use process.env.CLAUDE_MODEL with this as default)
  // max_tokens: 1500
  // For streaming (Copilot Chat): use stream: true, return AsyncIterable<string>
  // For batch (PR): use stream: false, return string
  // Retry on LLMRateLimitError: wait retryAfterMs then retry up to 3 times
  // Throw LLMUnavailableError after 3 failed retries
  // Log: prompt token count, completion token count, latency at INFO level
  async callClaude(prompt: string, stream: boolean, log: Logger): Promise<string | AsyncIterable<string>>

  // Parse the Claude response into structured Remediation object
  // Parse <explanation>, <patch>, <test> XML tags from response
  // If parsing fails: log WARN, return raw response as explanation, empty patch and test
  parseResponse(raw: string, finding: Finding): Remediation

  // Orchestrator: remediate a single finding
  // 1. Look up registry entry
  // 2. Build prompt
  // 3. Call Claude
  // 4. Parse response
  // 5. Return Remediation
  async remediate(finding: Finding, stream: boolean, log: Logger): Promise<Remediation>

  // Remediate multiple findings in parallel with concurrency limit
  // Max 5 concurrent Claude API calls
  // Return partial results if some fail — log failures at WARN, continue
  async remediateAll(findings: Finding[], log: Logger): Promise<Remediation[]>
}
```

**Claude prompt template — implement this exactly:**

```
You are SecureGuard AI, an expert application security engineer. A security scanner has detected a vulnerability in the developer's code. Your job is to explain it clearly and provide a working fix.

VULNERABILITY DETECTED:
- Type: {cweTitle} ({cweId})
- OWASP Reference: {owaspRef}
- Severity: {severity}
- File: {filePath}, Line {lineStart}

VULNERABLE CODE:
```{language}
{snippet}
```

SURROUNDING CONTEXT (if available):
```{language}
{surroundingCode}
```

COMPANY SECURITY GUIDANCE:
{guidance}

STANDARD FIX APPROACH:
{fixTemplate}

YOUR TASK:
Respond with exactly three sections using these XML tags. Do not include any text outside the tags.

<explanation>
Write 2-4 sentences in plain English that a developer will understand immediately.
Explain: (1) what the vulnerability is, (2) how an attacker could exploit it in THIS specific code, (3) what the impact would be.
Do not use jargon. Write as if explaining to a smart developer who is not a security expert.
</explanation>

<patch>
Provide the corrected version of the vulnerable code only.
Show the complete fixed function or block — not just the changed line.
Include a brief inline comment explaining what changed and why.
Use the same language as the vulnerable code.
</patch>

<test>
Provide a single unit test that would catch this vulnerability.
The test must actually test the security property, not just call the function.
Use the testing framework most appropriate for the language (pytest for Python, jest for TypeScript).
</test>
```

---

## SECTION 10 — POLICY ENGINE

### File: packages/core/src/policy.ts

```typescript
export interface PolicyRules {
  blockOnSeverities: Severity[]   // default: [CRITICAL, HIGH]
  allowOverrideLabel: string       // default: 'security-override'
  maxAllowedMedium: number         // default: 10 — block if more than this many MEDIUM findings
}

export class PolicyEngine {

  // Evaluate findings against policy rules
  // Returns PolicyResult with decision, blockedFindings, advisoryFindings
  // BLOCK if: any finding matches blockOnSeverities
  // BLOCK if: MEDIUM count exceeds maxAllowedMedium
  // OVERRIDE if: overrideLabel is present (passed in as parameter)
  // PASS otherwise
  evaluate(findings: Finding[], rules: PolicyRules, overrideLabel?: string): PolicyResult

  // Check GitHub PR labels for override label
  // Call GitHub API: GET /repos/{owner}/{repo}/issues/{prNumber}/labels
  // Return the label name if found, undefined if not
  async checkOverrideLabel(prNumber: number, repo: string, token: string): Promise<string | undefined>

  // Set GitHub commit status
  // POST /repos/{owner}/{repo}/statuses/{sha}
  // state: 'success' | 'failure' | 'pending'
  // context: 'secureguard/policy-gate'
  // description: short summary of result
  // target_url: link to audit log entry
  async setCommitStatus(result: PolicyResult, sha: string, repo: string, token: string, log: Logger): Promise<void>
}

// Default policy rules — load from env vars with these as fallback
export const DEFAULT_POLICY_RULES: PolicyRules = {
  blockOnSeverities: [Severity.CRITICAL, Severity.HIGH],
  allowOverrideLabel: process.env.POLICY_OVERRIDE_LABEL || 'security-override',
  maxAllowedMedium: parseInt(process.env.POLICY_MAX_MEDIUM || '10')
}
```

---

## SECTION 11 — AUDIT MODULE

### File: packages/core/src/audit.ts

Use `better-sqlite3` for hackathon (zero setup, file-based). Design schema to be Postgres-compatible for production migration.

```typescript
export class AuditService {

  // Initialize SQLite database and run migrations
  // Create tables if not exist — idempotent
  async initialize(dbPath: string): Promise<void>

  // Write audit record — never throw, log errors at ERROR level
  // If DB write fails, log and continue — audit must not break the main flow
  async write(record: AuditRecord): Promise<void>

  // Query audit records for a developer
  async findByDeveloper(developer: string, limit: number): Promise<AuditRecord[]>

  // Query audit records for a repo
  async findByRepo(repoPath: string, limit: number): Promise<AuditRecord[]>

  // Get summary statistics — used by dashboard/reporting
  async getSummary(since: Date): Promise<{
    totalScans: number
    totalFindings: number
    findingsByCategory: Record<string, number>
    findingsBySeverity: Record<Severity, number>
    averageRemediationTimeMs: number
    blockRate: number
    overrideRate: number
  }>
}
```

**SQL schema — implement this:**

```sql
CREATE TABLE IF NOT EXISTS audit_records (
  audit_id TEXT PRIMARY KEY,
  request_id TEXT NOT NULL,
  source TEXT NOT NULL,
  developer TEXT,
  repo_path TEXT NOT NULL,
  scan_duration_ms INTEGER NOT NULL,
  findings_count INTEGER NOT NULL,
  findings_critical INTEGER NOT NULL DEFAULT 0,
  findings_high INTEGER NOT NULL DEFAULT 0,
  findings_medium INTEGER NOT NULL DEFAULT 0,
  findings_low INTEGER NOT NULL DEFAULT 0,
  policy_decision TEXT NOT NULL,
  blocked_findings_count INTEGER NOT NULL DEFAULT 0,
  override_label TEXT,
  override_approver TEXT,
  remediations_generated INTEGER NOT NULL DEFAULT 0,
  llm_called INTEGER NOT NULL DEFAULT 0,
  created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS findings (
  finding_id TEXT PRIMARY KEY,
  audit_id TEXT NOT NULL REFERENCES audit_records(audit_id),
  cwe_id TEXT NOT NULL,
  cwe_title TEXT NOT NULL,
  severity TEXT NOT NULL,
  file_path TEXT NOT NULL,
  line_start INTEGER NOT NULL,
  scanner TEXT NOT NULL,
  rule_id TEXT NOT NULL,
  detected_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS remediations (
  remediation_id TEXT PRIMARY KEY,
  finding_id TEXT NOT NULL REFERENCES findings(finding_id),
  explanation TEXT NOT NULL,
  patch TEXT NOT NULL,
  test_stub TEXT NOT NULL,
  model TEXT NOT NULL,
  prompt_tokens INTEGER NOT NULL,
  completion_tokens INTEGER NOT NULL,
  latency_ms INTEGER NOT NULL,
  generated_at TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_audit_developer ON audit_records(developer);
CREATE INDEX IF NOT EXISTS idx_audit_repo ON audit_records(repo_path);
CREATE INDEX IF NOT EXISTS idx_audit_created ON audit_records(created_at);
CREATE INDEX IF NOT EXISTS idx_findings_audit ON findings(audit_id);
CREATE INDEX IF NOT EXISTS idx_findings_cwe ON findings(cwe_id);
```

---

## SECTION 12 — MIDDLEWARE

### File: packages/core/src/middleware/auth.ts
Validate `X-SecureGuard-Token` header against `SECUREGUARD_API_KEY` env var.
For Copilot Extension requests: validate the Copilot skill token using GitHub's public key.
Skip auth for `/health` and `/metrics` endpoints.
Return 401 with structured error if auth fails.

### File: packages/core/src/middleware/requestLogger.ts
Log every request: method, path, requestId (generate uuid if not in header), source IP, user-agent.
Log every response: status code, response time in ms, requestId.
Attach requestId to `res.locals` for use in route handlers.
Never log request body (may contain code snippets with secrets).

### File: packages/core/src/middleware/rateLimiter.ts
Use `express-rate-limit`.
- `/scan` endpoint: 60 requests per minute per IP (IDE saves frequently)
- `/copilot` endpoint: 20 requests per minute per user
- `/pr` endpoint: 10 requests per minute per repo
- Return 429 with `Retry-After` header and structured error body.

---

## SECTION 13 — SERVER AND ROUTER

### File: packages/core/src/server.ts

Full Express server setup:

```typescript
// Required middleware (in order):
// 1. helmet() — security headers
// 2. compression() — gzip responses
// 3. express.json({ limit: '5mb' }) — parse request body
// 4. requestLogger middleware
// 5. rateLimiter middleware
// 6. auth middleware (skip health + metrics)

// Routes:
// GET  /health       → { status: 'ok', version, uptime, timestamp }
// GET  /metrics      → audit summary stats (for demo dashboard)
// POST /scan         → IDE background + pre-commit + CLI path (no LLM)
// POST /copilot      → Copilot Chat path (LLM, streaming)
// POST /pr           → PR bot path (LLM, batch, GitHub API)

// Error handling middleware (last):
// errorHandler middleware

// Graceful shutdown:
// Listen for SIGTERM and SIGINT
// Stop accepting new connections
// Wait for in-flight requests to complete (max 10s)
// Close DB connection
// Log shutdown at INFO
```

### File: packages/core/src/router.ts

```typescript
// POST /scan handler
// 1. Parse and validate ScanRequest from body — throw InvalidRequestError if missing required fields
// 2. Create request logger with requestId
// 3. Start audit timer
// 4. Call scanner.scan()
// 5. Call normalizer.normalize()
// 6. Apply policy.evaluate() — but for IDE/commit, advisory only (never block at this route)
// 7. Format with lsp.ts or cli.ts based on source
// 8. Write audit record
// 9. Return formatted findings

// POST /copilot handler
// Copilot Extension SSE protocol:
// 1. Set headers: Content-Type: text/event-stream, Cache-Control: no-cache, X-Accel-Buffering: no
// 2. Parse intent from last message (scan/fix/explain/repo)
// 3. Run scanner on current file or repo
// 4. For each HIGH/CRITICAL finding: call remediator.remediate(stream: true)
// 5. Stream each token back as SSE: data: {token}\n\n
// 6. Stream formatted finding + remediation as markdown
// 7. End stream with data: [DONE]\n\n
// 8. Write audit record in background (don't await — don't block stream)

// POST /pr handler
// 1. Validate: prNumber, repo, githubToken, changedFiles required
// 2. Run full deep scan on all changedFiles
// 3. Check override label via policy.checkOverrideLabel()
// 4. Evaluate policy
// 5. For all HIGH/CRITICAL: remediateAll() in parallel
// 6. Post PR review comments via GitHub API with inline suggestions
// 7. Set commit status via policy.setCommitStatus()
// 8. Write audit record
// 9. Return { decision, findingsCount, remediationsPosted }
```

---

## SECTION 14 — FORMATTERS

### File: packages/core/src/formatters/lsp.ts
Convert `Finding[]` to LSP `Diagnostic[]` format.
Severity mapping: CRITICAL/HIGH → DiagnosticSeverity.Error (1), MEDIUM → Warning (2), LOW/INFO → Information (3).
Message format: `[{severity}] {cweTitle} — {one-line-fix-hint-from-registry}`
Include `code: cweId` and `source: 'secureguard'` on every diagnostic.

### File: packages/core/src/formatters/cli.ts
Use `chalk` for colored terminal output.
Format: colored severity badge, finding title, file:line, one-line guidance.
CRITICAL: red background. HIGH: red text. MEDIUM: yellow. LOW: gray.
Summary line at end: `SecureGuard found N findings (X critical, Y high, Z medium)`.
Exit code guidance: print `Run 'secureguard explain <findingId>' for AI-powered fix` for each blocking finding.

### File: packages/core/src/formatters/copilot.ts
Format AI remediation response as GitHub Flavored Markdown for Copilot Chat.
Structure:
```
## SecureGuard found {N} issue(s)

### 🔴 {severity}: {cweTitle} — {filePath}:{lineStart}

{explanation}

**Vulnerable code:**
```{lang}
{snippet}
```

**Fixed code:**
```{lang}
{patch}
```

**Unit test:**
```{lang}
{testStub}
```

> OWASP {owaspRef} · [Learn more]({owaspUrl})
```

### File: packages/core/src/formatters/pr.ts
Format for GitHub PR review comment using the Suggestions API.
The suggestion block must be a valid GitHub suggestion diff — single-line or multi-line.
Include OWASP reference and link in comment body.
If patch is multi-file or complex, post as comment without suggestion block.

---

## SECTION 15 — VS CODE EXTENSION

### File: packages/vscode-extension/src/extension.ts

```typescript
// activate(context):
// 1. Check if core server is running (GET /health) — start it if not (spawn child process)
// 2. Register LSP client (see lspClient.ts)
// 3. Register @secureguard Copilot Chat participant (see copilotHandler.ts)
// 4. Register command: secureguard.scanFile
// 5. Register command: secureguard.scanRepo
// 6. Register command: secureguard.showAuditLog
// 7. Add status bar item showing "SecureGuard: Active" with finding count

// deactivate():
// Stop LSP client
// Dispose all disposables
```

### File: packages/vscode-extension/src/lspClient.ts

```typescript
// Create LanguageClient connected to core server /scan endpoint
// ServerOptions: call POST /scan via HTTP (not stdio)
// DocumentFilter: { scheme: 'file', language: 'python' } and { scheme: 'file', language: 'typescript' }
// Trigger scan onDidSave with 500ms debounce
// Display diagnostics using VS Code DiagnosticsCollection
// On diagnostic click: show quick fix code action with "View AI Fix" option
// "View AI Fix" opens Copilot Chat with pre-filled @secureguard fix this message
```

### File: packages/vscode-extension/src/copilotHandler.ts

```typescript
// Register vscode.chat.createChatParticipant('secureguard', handler)
// Handle these slash commands:
//   /scan — scan current file
//   /scanrepo — scan entire workspace
//   /explain — explain a finding by ID
//   /fix — apply a patch to the current file
// Parse user message to determine intent when no slash command given
// Stream response using stream.markdown() for each token
// Use vscode.workspace.applyEdit() to apply patches from AI response
```

---

## SECTION 16 — CLI

### File: packages/cli/src/index.ts

Use `commander` for CLI parsing. Implement these commands:

```bash
secureguard scan [path]          # scan path (default: current dir)
  --staged                       # only scan git staged files
  --fail-on <severity>           # exit 1 if severity found (default: CRITICAL)
  --output <format>              # json | text (default: text)
  --mode <mode>                  # fast | full (default: fast)

secureguard explain <findingId>  # get AI explanation for a finding ID (calls /copilot)

secureguard install-hook         # install git pre-commit hook in current repo
  # writes .git/hooks/pre-commit with: secureguard scan --staged --fail-on CRITICAL

secureguard audit                # show audit log summary
  --since <date>                 # filter by date
  --repo <path>                  # filter by repo
```

---

## SECTION 17 — GITHUB ACTION

### File: packages/github-action/src/action.ts

```typescript
// Read inputs from action.yml: anthropic-api-key, github-token, fail-on, server-url
// Get PR context from GitHub Actions environment variables
// Get list of changed files: git diff --name-only origin/{base}...HEAD
// Call SecureGuard server POST /pr
// Set action outputs: findings-count, blocked, decision
// Exit process with code 1 if decision is BLOCK (fails the GitHub Actions check)
```

### File: packages/github-action/action.yml

```yaml
name: 'SecureGuard AI Security Check'
description: 'AI-powered vulnerability detection and remediation in Pull Requests'
inputs:
  anthropic-api-key:
    description: 'Anthropic API key for AI remediation'
    required: true
  github-token:
    description: 'GitHub token for posting PR comments'
    required: true
  fail-on:
    description: 'Minimum severity to fail the check (CRITICAL, HIGH, MEDIUM)'
    default: 'HIGH'
  server-url:
    description: 'SecureGuard server URL'
    default: 'http://localhost:3000'
outputs:
  findings-count:
    description: 'Total number of findings'
  decision:
    description: 'Policy decision: PASS, BLOCK, or OVERRIDE'
runs:
  using: 'node20'
  main: 'dist/action.js'
```

### File: .github/workflows/secureguard.yml

```yaml
name: SecureGuard Security Check
on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write
  statuses: write

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # needed for changed file detection

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Semgrep
        run: pip install semgrep

      - name: Install Gitleaks
        run: |
          wget -q https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz
          tar -xzf gitleaks_linux_x64.tar.gz
          sudo mv gitleaks /usr/local/bin/

      - name: Install pip-audit
        run: pip install pip-audit

      - name: Start SecureGuard server
        run: |
          cd packages/core
          npm ci
          npm run build
          node dist/server.js &
          sleep 3
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          NODE_ENV: production

      - name: Run SecureGuard
        uses: ./packages/github-action
        with:
          anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
          github-token: ${{ secrets.GITHUB_TOKEN }}
          fail-on: 'HIGH'
```

---

## SECTION 18 — DEMO APPLICATION

### Intentionally vulnerable files for the demo — implement all of these:

**demo-app/python/routes/users.py** — SQL injection:
```python
# VULNERABLE: SQL injection on line 23
import sqlite3

def get_user(user_id):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    # BUG: Direct string interpolation — attacker can inject arbitrary SQL
    query = f"SELECT * FROM users WHERE id = {user_id}"
    cursor.execute(query)
    return cursor.fetchone()
```

**demo-app/python/routes/auth.py** — hardcoded secret + weak JWT:
```python
# VULNERABLE: hardcoded secret, no JWT expiry
import jwt

SECRET_KEY = "super_secret_key_123"  # BUG: hardcoded, weak

def create_token(user_id):
    # BUG: no expiresIn, uses default algorithm which can be 'none'
    token = jwt.encode({"user_id": user_id}, SECRET_KEY)
    return token
```

**demo-app/typescript/routes/render.ts** — XSS:
```typescript
// VULNERABLE: XSS via innerHTML
export function renderUserProfile(req: Request, res: Response) {
  const username = req.query.name as string;
  // BUG: unescaped user input injected into DOM
  res.send(`<div id="profile"><h1>Hello ${username}</h1></div>`);
}
```

**demo-app/python/requirements.txt** — vulnerable dep:
```
# This version has a known CVE
django==3.2.0
requests==2.25.0
```

---

## SECTION 19 — CONFIGURATION AND ENVIRONMENT

### File: packages/core/.env.example

```bash
# Required
ANTHROPIC_API_KEY=your-key-here
SECUREGUARD_API_KEY=your-shared-secret-here

# Optional — shown with defaults
NODE_ENV=development
PORT=3000
LOG_LEVEL=info
CLAUDE_MODEL=claude-sonnet-4-6
DB_PATH=./secureguard.db
SEMGREP_TIMEOUT_MS=30000
GITLEAKS_TIMEOUT_MS=15000
AUDIT_TIMEOUT_MS=20000
POLICY_BLOCK_ON=CRITICAL,HIGH
POLICY_MAX_MEDIUM=10
POLICY_OVERRIDE_LABEL=security-override
MAX_CONCURRENT_LLM_CALLS=5
```

### File: docker-compose.yml

```yaml
version: '3.8'
services:
  secureguard:
    build:
      context: ./packages/core
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
    env_file:
      - ./packages/core/.env
    volumes:
      - secureguard-db:/app/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  secureguard-db:
```

---

## SECTION 20 — PACKAGE.JSON FILES

### Root package.json (workspace):
```json
{
  "name": "secureguard",
  "private": true,
  "workspaces": ["packages/*"],
  "scripts": {
    "build": "npm run build --workspaces",
    "dev": "cd packages/core && npm run dev",
    "test": "npm run test --workspaces",
    "install:tools": "pip install semgrep pip-audit bandit && brew install gitleaks"
  }
}
```

### packages/core/package.json — all dependencies:
```json
{
  "name": "@secureguard/core",
  "version": "1.0.0",
  "main": "dist/server.js",
  "scripts": {
    "build": "tsc",
    "dev": "ts-node-dev --respawn src/server.ts",
    "test": "jest",
    "lint": "eslint src/"
  },
  "dependencies": {
    "@anthropic-ai/sdk": "^0.24.0",
    "better-sqlite3": "^9.4.3",
    "chalk": "^5.3.0",
    "commander": "^12.0.0",
    "compression": "^1.7.4",
    "express": "^4.18.3",
    "express-rate-limit": "^7.2.0",
    "helmet": "^7.1.0",
    "pino": "^8.19.0",
    "pino-pretty": "^11.0.0",
    "uuid": "^9.0.1"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.8",
    "@types/compression": "^1.7.5",
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.0",
    "@types/uuid": "^9.0.7",
    "jest": "^29.7.0",
    "ts-jest": "^29.1.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.3.3"
  }
}
```

---

## SECTION 21 — SEMGREP CUSTOM RULES

### File: packages/core/semgrep-rules/sql-injection.yaml

```yaml
rules:
  - id: secureguard-python-sql-injection-fstring
    languages: [python]
    severity: ERROR
    message: "SQL Injection (CWE-89): f-string used to build SQL query with user input. Use parameterized queries."
    patterns:
      - pattern: |
          $CURSOR.execute(f"...{$VAR}...")
      - pattern: |
          $CURSOR.execute("..." + $VAR + "...")
    metadata:
      cwe: CWE-89
      owasp: A03:2021

  - id: secureguard-js-sql-injection-template
    languages: [javascript, typescript]
    severity: ERROR
    message: "SQL Injection (CWE-89): Template literal used in SQL query. Use parameterized queries."
    patterns:
      - pattern: |
          $DB.query(`...${$VAR}...`)
      - pattern: |
          $DB.execute(`...${$VAR}...`)
    metadata:
      cwe: CWE-89
      owasp: A03:2021
```

Create similar YAML files for hardcoded-secrets.yaml, weak-auth.yaml, and xss.yaml covering the remaining CWEs.

---

## SECTION 22 — TESTS

Create tests for every module. Use Jest + ts-jest. Minimum coverage requirements:

- `normalizer.test.ts` — test CWE mapping, deduplication, severity assignment, snippet truncation
- `registry.test.ts` — test load, lookup, missing entry error
- `policy.test.ts` — test PASS/BLOCK/OVERRIDE decisions for all severity combinations
- `remediator.test.ts` — test prompt building, response parsing, XML tag extraction, partial failure handling
- `audit.test.ts` — test write, read, summary stats with in-memory SQLite
- `scanner.test.ts` — test timeout handling, parse error handling, empty results
- `router.test.ts` — integration tests for /scan, /copilot, /pr endpoints using supertest

Mock the Claude API in tests using `jest.mock('@anthropic-ai/sdk')`.
Mock subprocess execution in scanner tests using `jest.mock('child_process')`.

---

## SECTION 23 — README AND DOCUMENTATION

### File: README.md

Write a complete README covering:
1. Architecture overview (reference the HLD — three surfaces, single service, LLM routing decision)
2. Prerequisites (Node.js 20+, Python 3.9+, Semgrep, Gitleaks)
3. Quick start — get running in 5 commands
4. Configuration — all env vars with descriptions
5. The five vulnerability categories and how each is detected + remediated
6. How to install the VS Code extension
7. How to install the pre-commit hook
8. How to add the GitHub Actions workflow
9. How to extend — add a new CWE to the registry
10. API reference — /health, /scan, /copilot, /pr endpoints with request/response examples
11. Audit log — how to query it, what it stores
12. Production deployment guide

---

## SECTION 24 — BUILD ORDER FOR THE AGENT

Execute these steps in order. Complete each step fully before moving to the next:

1. `npm init` workspace root, create package.json
2. Create all directory structure
3. Implement `types.ts` — all interfaces and enums
4. Implement `logger.ts` — pino setup
5. Implement `errors.ts` — all error classes
6. Implement `registry.ts` + create `remediator-registry.json` with all 5 entries
7. Implement `normalizer.ts` + `RULE_TO_CWE_MAP`
8. Implement `scanner.ts` — all four scanners
9. Implement `remediator.ts` — Claude API integration with full prompt
10. Implement `policy.ts` — evaluation + GitHub API calls
11. Implement `audit.ts` — SQLite setup + all queries
12. Implement all middleware files
13. Implement all formatter files
14. Implement `router.ts` — all three route handlers
15. Implement `server.ts` — full Express app with graceful shutdown
16. Implement VS Code extension files
17. Implement CLI
18. Implement GitHub Action
19. Create demo-app vulnerable files
20. Create semgrep custom YAML rules
21. Create docker-compose.yml + .env.example
22. Write all tests
23. Run `npm run build` — fix all TypeScript errors
24. Run `npm test` — fix all failing tests
25. Run `docker-compose up` — verify /health returns 200
26. Write README.md
27. Final check: run `secureguard scan demo-app/` — verify findings are detected

---

## SECTION 25 — QUALITY GATES

Before declaring the build complete, verify every item on this checklist:

### Functionality
- [ ] VS Code squiggle appears within 2 seconds of saving a vulnerable file
- [ ] `git commit` on demo-app/python/routes/users.py is blocked with CRITICAL message
- [ ] `@secureguard scan this file` in Copilot Chat returns streamed AI fix within 8 seconds
- [ ] GitHub Actions workflow runs on PR and posts inline review comment with suggestion
- [ ] PR status check shows 'failure' state when HIGH findings exist
- [ ] PR status check shows 'success' after applying all fixes
- [ ] Audit log records every scan with correct findings count and policy decision

### Error handling
- [ ] Server returns structured JSON error (not HTML stack trace) when Claude API is down
- [ ] Scanner timeout returns partial results + logged warning (does not crash)
- [ ] Missing registry entry returns clear error message with the CWE ID
- [ ] Rate limit returns 429 with Retry-After header

### Logging
- [ ] Every request has a requestId in all log lines
- [ ] No secrets appear in any log line
- [ ] Scanner subprocess start/end/result logged at INFO
- [ ] Claude API call latency and token count logged at INFO
- [ ] All errors logged with full context at ERROR level

### Security
- [ ] All endpoints require auth header (except /health)
- [ ] No user input ever interpolated into subprocess commands
- [ ] Snippets truncated to 500 chars before being sent to Claude API
- [ ] .env file gitignored

---

## SECTION 26 — HACKATHON DEMO SCRIPT

Implement this exact scenario end-to-end so judges can see the complete flow in under 3 minutes:

**Setup:** Terminal 1: `docker-compose up`. Terminal 2: VS Code with demo-app open.

**Scene 1 — IDE (30 seconds):**
Open `demo-app/python/routes/users.py`. Save the file. Show red squiggle appearing on line 23 with "CRITICAL: SQL Injection — use parameterized queries" in the Problems panel.

**Scene 2 — Copilot Chat (60 seconds):**
In Copilot Chat, type: `@secureguard what's vulnerable in this file?`
Show streaming response: severity badge → plain English explanation → fixed code block → unit test → OWASP reference.

**Scene 3 — Pre-commit block (30 seconds):**
In terminal: `cd demo-app && git add python/routes/users.py && git commit -m "add user route"`
Show: commit blocked with colored CRITICAL message and `Run secureguard explain <id> for AI fix`.

**Scene 4 — PR gate (60 seconds):**
Open a pre-prepared PR that includes the vulnerable file.
Show: GitHub Actions running → PR review comment with inline fix suggestion → red X status check.
Apply the suggestion → push → Actions re-runs → green check → merge button enabled.

---

## FINAL AGENT INSTRUCTION

You now have everything you need. Begin building immediately. Start with step 1 of Section 24 — create the workspace package.json. Work through all 27 steps in order. Do not skip any step. Do not stub any function. When you complete the build, run the quality gate checklist from Section 25 and fix everything that fails.

This is a hackathon submission. The code must actually work, not just compile. Every function must be fully implemented. The demo scenario in Section 26 must run successfully from start to finish.

Build it.

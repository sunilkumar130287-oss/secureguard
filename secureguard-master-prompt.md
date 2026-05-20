# SecureGuard AI — Master Build Prompt v2
## For Claude Opus 4.6 Agent in GitHub Copilot (GHCP)
## Languages: Python · TypeScript · JavaScript · Angular · Java

---

> **HOW TO USE THIS PROMPT**
> Open GitHub Copilot Chat in VS Code. Switch model to Claude Opus 4.6.
> Paste this entire document as your first message.
> The agent will scaffold, build, and wire the complete application autonomously.
> Only pause when asked for a secret or credential — have ANTHROPIC_API_KEY and GITHUB_TOKEN ready.

---

## AGENT INSTRUCTIONS — READ FIRST

You are Claude Opus 4.6, an autonomous software engineering agent operating inside GitHub Copilot in VS Code. Your task is to build **SecureGuard AI** — a production-grade, enterprise-ready AI security agent that supports Python, TypeScript, JavaScript, Angular, and Java — from scratch in this workspace.

You have full access to:
- The file system (read, write, create, delete)
- The terminal (run commands, install packages, execute scripts)
- The editor (create and modify any file)

**Work autonomously. Do not ask for permission before each step. Make sensible decisions, document them inline, and keep building. Only pause if you hit a genuine blocker that requires a secret, credential, or an architectural decision that changes the entire design.**

Build the complete application in the order defined in Section 24. Do not skip steps. Do not stub functions — implement them fully.

---

## SECTION 1 — PROJECT OVERVIEW AND GOALS

### What you are building
SecureGuard AI is a four-surface AI security agent that detects vulnerabilities and returns AI-generated, context-aware remediations grounded in OWASP and company policy. It runs:

- **Surface 1**: VS Code background daemon (Language Server Protocol) — scans on file save, inline diagnostics, no LLM
- **Surface 2**: Git pre-commit hook — scans staged files, blocks commit on CRITICAL, no LLM
- **Surface 3**: GitHub Copilot Chat (`@secureguard`) — on-demand scan + AI remediation via Claude API, streamed
- **Surface 4**: GitHub Actions PR bot — deep scan on PR open, inline fix suggestions, policy gate blocking merge on HIGH/CRITICAL

### Architecture decision — single service
All four surfaces call one Express server (`packages/core`). The router decides whether to invoke Claude (LLM path: Copilot Chat + PR) or return scanner results directly (no-LLM path: LSP + CLI + pre-commit). IDE scanning stays under 2 seconds. Java's build step (Maven compile) runs only on the PR/full path — never on IDE save.

### Languages fully supported
- **Python** — Semgrep + Bandit + pip-audit
- **TypeScript / JavaScript** — Semgrep + npm audit
- **Angular** (TypeScript) — Semgrep with Angular-specific rules + npm audit + CSRF detection
- **Java** — Semgrep + SpotBugs + OWASP Dependency-Check

### Language detection — automatic
The scanner auto-detects language from file extensions and build files:
- `.py` → Python scanners
- `.ts`, `.tsx`, `.js`, `.jsx` → TypeScript/JavaScript scanners
- `.component.ts`, `.service.ts`, `.module.ts`, `angular.json` present → also run Angular-specific rules
- `.java`, `pom.xml`, `build.gradle` present → Java scanners

### Nine vulnerability categories (five required + four Java/Angular specific)

**Core five (all languages):**
1. SQL Injection (CWE-89) — OWASP A03:2021
2. Hardcoded secrets/credentials (CWE-798) — OWASP A02:2021
3. Broken authentication — weak JWT, missing expiry (CWE-287) — OWASP A07:2021
4. Vulnerable dependencies — known CVEs (CWE-937) — OWASP A06:2021
5. Cross-site scripting — unescaped output (CWE-79) — OWASP A03:2021

**Angular specific:**
6. Angular template injection via bypassSecurityTrustHtml (CWE-79) — OWASP A03:2021
7. Cross-Site Request Forgery — missing CSRF token (CWE-352) — OWASP A01:2021

**Java specific:**
8. XML External Entity injection — XXE (CWE-611) — OWASP A05:2021
9. Insecure deserialization (CWE-502) — OWASP A08:2021

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
│   │   │   ├── languageDetector.ts       ← NEW: auto-detects language from file paths
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
│   │   ├── remediator-registry.json      ← 9 CWE entries (5 core + 2 Angular + 2 Java)
│   │   ├── semgrep-rules/
│   │   │   ├── sql-injection.yaml        ← Python + JS/TS + Java
│   │   │   ├── hardcoded-secrets.yaml    ← All languages
│   │   │   ├── weak-auth.yaml            ← Python + JS/TS + Java JWT
│   │   │   ├── xss.yaml                  ← JS/TS + React + Angular
│   │   │   ├── angular-security.yaml     ← NEW: Angular-specific rules
│   │   │   ├── java-security.yaml        ← NEW: Java-specific rules
│   │   │   └── java-xxe.yaml             ← NEW: Java XXE rules
│   │   ├── spotbugs/
│   │   │   └── secureguard-exclude.xml   ← NEW: SpotBugs filter config
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
│   │   ├── routes/users.py               ← SQL injection
│   │   ├── routes/auth.py                ← hardcoded secret + weak JWT
│   │   └── requirements.txt              ← vulnerable deps
│   ├── typescript/
│   │   ├── routes/users.ts               ← SQL injection
│   │   ├── routes/render.ts              ← XSS
│   │   └── package.json                  ← vulnerable deps
│   ├── angular/                          ← NEW
│   │   ├── src/app/
│   │   │   ├── user-profile/
│   │   │   │   ├── user-profile.component.ts   ← innerHTML XSS
│   │   │   │   └── user-profile.component.html ← template injection
│   │   │   ├── auth/
│   │   │   │   └── auth.service.ts              ← bypassSecurityTrustHtml
│   │   │   └── api/
│   │   │       └── data.service.ts              ← missing CSRF header
│   │   ├── angular.json
│   │   └── package.json                  ← vulnerable deps
│   └── java/                             ← NEW
│       ├── src/main/java/com/demo/
│       │   ├── UserController.java        ← SQL injection
│       │   ├── XmlParser.java             ← XXE
│       │   ├── AuthService.java           ← hardcoded credentials
│       │   └── DataProcessor.java         ← insecure deserialization
│       └── pom.xml                        ← vulnerable deps (Log4Shell era)
│
├── docker-compose.yml
├── package.json                           ← workspace root
└── README.md
```

---

## SECTION 3 — TYPES

### File: packages/core/src/types.ts

Implement all interfaces and enums with full JSDoc comments:

```typescript
export enum Severity {
  CRITICAL = 'CRITICAL',
  HIGH     = 'HIGH',
  MEDIUM   = 'MEDIUM',
  LOW      = 'LOW',
  INFO     = 'INFO'
}

export enum ScanSource {
  IDE     = 'ide',
  COMMIT  = 'commit',
  PR      = 'pr',
  CLI     = 'cli',
  COPILOT = 'copilot'
}

export enum ScannerTool {
  SEMGREP       = 'semgrep',
  GITLEAKS      = 'gitleaks',
  NPM_AUDIT     = 'npm_audit',
  PIP_AUDIT     = 'pip_audit',
  BANDIT        = 'bandit',
  SPOTBUGS      = 'spotbugs',        // NEW: Java bytecode analysis
  OWASP_DC      = 'owasp_dep_check'  // NEW: Java dependency CVE scanner
}

export enum ScanMode {
  FAST = 'fast',
  FULL = 'full'
}

// NEW: supported language identifiers
export type Language =
  | 'python'
  | 'typescript'
  | 'javascript'
  | 'angular'      // TypeScript + Angular-specific rules
  | 'java'
  | 'auto'         // auto-detect from file extensions + build files

export interface ScanRequest {
  requestId:    string        // uuid v4
  source:       ScanSource
  filePaths:    string[]
  repoRoot:     string
  language?:    Language      // if omitted, languageDetector.ts resolves it
  scanMode:     ScanMode
  prNumber?:    number
  githubRepo?:  string        // owner/repo format
  githubToken?: string
  requestedAt:  Date
}

export interface RawFinding {
  ruleId:       string
  filePath:     string
  lineStart:    number
  lineEnd:      number
  columnStart:  number
  columnEnd:    number
  message:      string
  snippet:      string
  scanner:      ScannerTool
  extra?:       Record<string, unknown>
}

export interface Finding {
  findingId:   string         // uuid v4
  requestId:   string         // FK to ScanRequest
  cweId:       string         // e.g. 'CWE-89'
  cweTitle:    string         // e.g. 'SQL Injection'
  severity:    Severity
  filePath:    string
  lineStart:   number
  lineEnd:     number
  snippet:     string         // vulnerable code, max 500 chars
  scanner:     ScannerTool
  ruleId:      string
  language:    Language       // which language this finding is for
  detectedAt:  Date
}

export interface RegistryEntry {
  cweId:        string
  name:         string
  owaspRef:     string        // e.g. 'A03:2021'
  owaspUrl:     string
  severity:     Severity
  guidance:     string        // company-specific secure-dev guidance
  fixTemplate:  string        // language-agnostic fix description
  references:   string[]
  languages:    Language[]    // which languages this entry applies to
}

export interface Remediation {
  remediationId:    string    // uuid v4
  findingId:        string    // FK to Finding
  explanation:      string    // plain English for the developer
  patch:            string    // fixed code block
  testStub:         string    // unit test that would catch this
  model:            string    // claude-sonnet-4-6
  promptTokens:     number
  completionTokens: number
  generatedAt:      Date
  latencyMs:        number
}

export enum PolicyDecision {
  PASS     = 'PASS',
  BLOCK    = 'BLOCK',
  OVERRIDE = 'OVERRIDE'
}

export interface PolicyResult {
  decision:          PolicyDecision
  blockedFindings:   Finding[]
  advisoryFindings:  Finding[]
  overrideLabel?:    string
  overrideApprover?: string
  evaluatedAt:       Date
}

export interface AuditRecord {
  auditId:               string    // uuid v4
  requestId:             string
  source:                ScanSource
  language:              Language
  developer?:            string    // git config user.email if available
  repoPath:              string
  scanDurationMs:        number
  findingsCount:         number
  findingsBySeverity:    Record<Severity, number>
  policyResult:          PolicyResult
  remediationsGenerated: number
  llmCalled:             boolean
  createdAt:             Date
}
```

---

## SECTION 4 — LANGUAGE DETECTOR (NEW MODULE)

### File: packages/core/src/languageDetector.ts

This module inspects file paths and the repo root to determine which language pipeline to activate. It is called by the router before dispatching to the scanner.

```typescript
export class LanguageDetector {

  // Inspect filePaths and repoRoot to determine Language
  // Rules (evaluated in order — first match wins):
  // 1. If any filePath ends with .java → 'java'
  //    OR if pom.xml or build.gradle exists in repoRoot → 'java'
  // 2. If angular.json exists in repoRoot → 'angular'
  //    OR if any filePath matches *.component.ts, *.service.ts, *.module.ts → 'angular'
  // 3. If any filePath ends with .py → 'python'
  // 4. If any filePath ends with .ts or .tsx or .jsx → 'typescript'
  // 5. If any filePath ends with .js → 'javascript'
  // 6. Default → 'typescript' (safest fallback for unknown)
  // Mixed repos (e.g. Java backend + Angular frontend) → return ALL detected languages
  // as an array; the scanner runs the appropriate tools for each
  detect(filePaths: string[], repoRoot: string): Language[]

  // Convenience: returns true if any filePath looks like an Angular file
  isAngular(filePaths: string[], repoRoot: string): boolean

  // Returns true if pom.xml or build.gradle exists
  isJava(repoRoot: string): boolean

  // Returns the file extension → language map used internally
  getExtensionMap(): Record<string, Language>
}
```

---

## SECTION 5 — LOGGER

### File: packages/core/src/logger.ts

Use `pino`. Structured JSON logging with:
- Always include: ISO 8601 timestamp, level, requestId (if available), service name
- Never log: ANTHROPIC_API_KEY, GITHUB_TOKEN, file snippets containing secrets, PII
- Production mode: JSON output. Development mode: pino-pretty colored output
- Export child logger factory: `createRequestLogger(requestId: string)`

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

## SECTION 6 — ERROR HANDLING

### File: packages/core/src/errors.ts

```typescript
export class SecureGuardBaseError extends Error {
  code:       string
  retryable:  boolean
  context?:   Record<string, unknown>
  requestId?: string

  constructor(code: string, message: string, retryable: boolean, context?: Record<string, unknown>) {
    super(message)
    this.name = 'SecureGuardBaseError'
    this.code = code
    this.retryable = retryable
    this.context = context
  }
}

// Implement every class below extending SecureGuardBaseError:
export class ScannerTimeoutError        // SCANNER_TIMEOUT, retryable: true
export class ScannerNotFoundError       // SCANNER_NOT_FOUND, retryable: false
export class ScannerParseError          // SCANNER_PARSE_ERROR, retryable: false
export class JavaBuildError             // JAVA_BUILD_ERROR, retryable: false — NEW
export class RegistryNotFoundError      // REGISTRY_ENTRY_NOT_FOUND, retryable: false
export class LLMUnavailableError        // LLM_UNAVAILABLE, retryable: true
export class LLMRateLimitError          // LLM_RATE_LIMIT, retryable: true — include retryAfterMs
export class LLMContextLengthError      // LLM_CONTEXT_TOO_LONG, retryable: false
export class PolicyViolationError       // POLICY_VIOLATION, retryable: false
export class GitHubAPIError             // GITHUB_API_ERROR, retryable: true
export class InvalidRequestError        // INVALID_REQUEST, retryable: false
export class AuditWriteError            // AUDIT_WRITE_ERROR, retryable: true
export class UnsupportedLanguageError   // UNSUPPORTED_LANGUAGE, retryable: false — NEW
```

### File: packages/core/src/middleware/errorHandler.ts

Express error handling middleware:
- Catches all SecureGuardBaseError and unknown errors
- Logs with full context — NEVER expose stack traces in production
- HTTP status mappings:
  - INVALID_REQUEST, UNSUPPORTED_LANGUAGE → 400
  - LLM_RATE_LIMIT → 429 with Retry-After header
  - POLICY_VIOLATION → 422
  - JAVA_BUILD_ERROR → 422 with message "Java project must be compiled before scanning"
  - SCANNER_TIMEOUT, LLM_UNAVAILABLE → 503 with Retry-After: 30
  - Unknown errors → 500
- Development mode: include stack trace in response body

---

## SECTION 7 — SCANNER MODULE

### File: packages/core/src/scanner.ts

Implement `ScannerService` with all methods below. Every method: full error handling, timeout enforcement, structured logging.

```typescript
export class ScannerService {

  // ── UNIVERSAL SCANNERS ──────────────────────────────────────────────────

  // Semgrep: language-aware rules selected by scanMode + detected language
  // fast mode configs: p/python, p/javascript, p/java, ./semgrep-rules/
  // full mode adds:    p/owasp-top-ten, p/ci, p/java-security, p/angular
  // Always: --json --timeout 30 --max-memory 512 --no-git-ignore
  async runSemgrep(filePaths: string[], languages: Language[], scanMode: ScanMode, timeoutMs: number): Promise<RawFinding[]>

  // Gitleaks: secrets detection across all languages
  async runGitleaks(repoRoot: string, filePaths: string[], timeoutMs: number): Promise<RawFinding[]>

  // ── JAVASCRIPT / TYPESCRIPT / ANGULAR ───────────────────────────────────

  // npm audit — only if package-lock.json exists in repoRoot
  async runNpmAudit(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // ── PYTHON ──────────────────────────────────────────────────────────────

  // pip-audit — only if requirements.txt or Pipfile.lock exists
  async runPipAudit(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // Bandit — Python SAST, supplement to Semgrep for Python
  // Command: bandit -r <filePaths> -f json
  async runBandit(filePaths: string[], timeoutMs: number): Promise<RawFinding[]>

  // ── JAVA ─────────────────────────────────────────────────────────────────

  // SpotBugs — Java bytecode SAST. REQUIRES compiled classes.
  // Step 1: detect build system (Maven or Gradle)
  // Step 2: run compile: mvn compile -q OR gradle compileJava -q
  //         If compile fails → throw JavaBuildError with message
  //         "SpotBugs requires compiled Java classes. Run: mvn compile"
  // Step 3: run SpotBugs:
  //   spotbugs -textui -xml:withMessages -effort:max
  //            -exclude spotbugs/secureguard-exclude.xml
  //            -output /tmp/spotbugs-{requestId}.xml
  //            target/classes/ (Maven) or build/classes/ (Gradle)
  // Step 4: parse XML → RawFinding[]
  // Step 5: clean up temp file in finally block
  // Only run in FULL scan mode (not fast — compile too slow for IDE)
  async runSpotBugs(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // OWASP Dependency-Check — Java CVE scanner for Maven/Gradle deps
  // Command: dependency-check --project secureguard --scan pom.xml
  //          --format JSON --out /tmp/dc-{requestId}/ --noupdate
  // Parse /tmp/dc-{requestId}/dependency-check-report.json
  // Map CVE entries → CWE-937 findings
  // Only run if pom.xml or build.gradle exists
  // Only run in FULL scan mode
  async runOwaspDependencyCheck(repoRoot: string, timeoutMs: number): Promise<RawFinding[]>

  // ── ORCHESTRATOR ─────────────────────────────────────────────────────────

  // Run appropriate scanners for detected languages, in parallel
  //
  // FAST MODE (IDE/commit):
  //   All languages: Semgrep fast rules + Gitleaks
  //   JS/TS/Angular:  + npm audit
  //   Python:         + pip-audit (quick) + Bandit
  //   Java:           Semgrep ONLY (no SpotBugs — requires compile, too slow)
  //
  // FULL MODE (PR/Copilot):
  //   All languages: Semgrep full rules + Gitleaks
  //   JS/TS/Angular:  + npm audit
  //   Python:         + pip-audit + Bandit
  //   Java:           + SpotBugs + OWASP Dependency-Check
  //
  // Never throws — catches each scanner error individually, logs WARN, continues
  // Returns merged RawFinding[] from all scanners that succeeded
  async scan(request: ScanRequest, languages: Language[], log: Logger): Promise<RawFinding[]>
}
```

**Important implementation details:**
- Use `child_process.spawn` (not exec) — never interpolate user paths into shell strings
- Pass file paths as array arguments to spawn, never concatenated strings
- Set `maxBuffer: 10 * 1024 * 1024` (10MB)
- Use `Promise.race` against a timeout promise for each scanner
- Log scanner start/completion/finding-count at INFO
- Log scanner errors at WARN (partial results are acceptable — don't fail the whole scan)
- Always clean up temp files in `finally` blocks
- SpotBugs compile step: log "Compiling Java project for SpotBugs analysis" at INFO before mvn/gradle runs

---

## SECTION 8 — NORMALIZER MODULE

### File: packages/core/src/normalizer.ts

```typescript
export class NormalizerService {

  // Convert RawFinding[] → Finding[]
  // 1. Map scanner rule IDs → CWE using RULE_TO_CWE_MAP
  // 2. Map SpotBugs bug patterns → CWE using SPOTBUGS_TO_CWE_MAP
  // 3. Map Gitleaks IDs → CWE-798
  // 4. Map npm/pip/owasp-dc audit entries → CWE-937
  // 5. Deduplicate: same (filePath + lineStart + cweId) = one finding, keep highest severity
  // 6. Generate findingId (uuid v4)
  // 7. Truncate snippet to 500 chars max
  // 8. Normalize filePath to be relative to repoRoot
  // 9. Set language field from file extension
  normalize(raw: RawFinding[], requestId: string, repoRoot: string): Finding[]
}

// ── SEMGREP RULE → CWE MAP ─────────────────────────────────────────────────
// Must cover every rule ID that Semgrep reports for all five languages:
const RULE_TO_CWE_MAP: Record<string, { cweId: string; cweTitle: string; severity: Severity }> = {

  // Python — SQL Injection
  'python.lang.security.audit.dangerous-sys-exec.dangerous-sys-exec':  { cweId: 'CWE-78',  cweTitle: 'OS Command Injection',        severity: Severity.CRITICAL },
  'python.sqlalchemy.security.sqlalchemy-execute-raw-query':            { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },
  'python.django.security.injection.sql.sql-injection':                 { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },

  // Python — Auth
  'python.jwt.security.jwt-none-alg':                                   { cweId: 'CWE-287', cweTitle: 'Broken Authentication',       severity: Severity.CRITICAL },
  'python.jwt.security.jwt-hardcoded-secret':                           { cweId: 'CWE-798', cweTitle: 'Hardcoded Credentials',       severity: Severity.CRITICAL },

  // JavaScript / TypeScript — SQL Injection
  'javascript.sequelize.security.audit.sequelize-injection':            { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },
  'javascript.typeorm.security.audit.sql-injection':                    { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },
  'typescript.knex.security.audit.knex-sql-injection':                  { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },

  // JavaScript / TypeScript — XSS
  'javascript.express.security.audit.xss.pug-var-in-code-block':        { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',        severity: Severity.HIGH },
  'javascript.browser.security.innerHTML-injection.innerHTML-injection': { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',        severity: Severity.HIGH },

  // React / TypeScript — XSS
  'typescript.react.security.audit.react-dangerouslysetinnerhtml':      { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',        severity: Severity.HIGH },
  'typescript.react.security.react-href-user-input':                    { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',        severity: Severity.HIGH },

  // Angular — XSS / Template Injection (custom rules)
  'secureguard-angular-bypass-sanitizer':                               { cweId: 'CWE-79',  cweTitle: 'Angular XSS Bypass',          severity: Severity.CRITICAL },
  'secureguard-angular-innerhtml-binding':                              { cweId: 'CWE-79',  cweTitle: 'Angular innerHTML Binding',    severity: Severity.HIGH },
  'secureguard-angular-no-csrf-header':                                 { cweId: 'CWE-352', cweTitle: 'Cross-Site Request Forgery',  severity: Severity.HIGH },
  'secureguard-angular-href-injection':                                 { cweId: 'CWE-79',  cweTitle: 'Angular href Injection',      severity: Severity.HIGH },

  // Java — SQL Injection (Semgrep)
  'java.lang.security.audit.sqli.jdbc-sqli':                            { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },
  'java.lang.security.audit.sqli.spring-sqli':                          { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },
  'java.hibernate.security.audit.hibernate-sqli':                       { cweId: 'CWE-89',  cweTitle: 'SQL Injection',               severity: Severity.CRITICAL },

  // Java — XXE (Semgrep)
  'java.lang.security.audit.xxe.sax-xxe':                              { cweId: 'CWE-611', cweTitle: 'XML External Entity',         severity: Severity.HIGH },
  'java.lang.security.audit.xxe.documentbuilder-xxe':                  { cweId: 'CWE-611', cweTitle: 'XML External Entity',         severity: Severity.HIGH },
  'java.lang.security.audit.xxe.xmlreader-xxe':                        { cweId: 'CWE-611', cweTitle: 'XML External Entity',         severity: Severity.HIGH },
  'secureguard-java-xxe-saxparser':                                     { cweId: 'CWE-611', cweTitle: 'XML External Entity',         severity: Severity.HIGH },

  // Java — Auth / Secrets (Semgrep)
  'java.lang.security.audit.hardcoded-credentials.hardcoded-credentials': { cweId: 'CWE-798', cweTitle: 'Hardcoded Credentials',    severity: Severity.CRITICAL },
  'java.spring.security.audit.spring-csrf-disabled':                    { cweId: 'CWE-352', cweTitle: 'CSRF Protection Disabled',   severity: Severity.HIGH },

  // Java — Deserialization (Semgrep)
  'java.lang.security.audit.unsafe-deserialization':                    { cweId: 'CWE-502', cweTitle: 'Insecure Deserialization',    severity: Severity.CRITICAL },
  'secureguard-java-objectinputstream':                                 { cweId: 'CWE-502', cweTitle: 'Insecure Deserialization',    severity: Severity.CRITICAL },
}

// ── SPOTBUGS BUG PATTERN → CWE MAP ────────────────────────────────────────
const SPOTBUGS_TO_CWE_MAP: Record<string, { cweId: string; cweTitle: string; severity: Severity }> = {
  'SQL_INJECTION':                           { cweId: 'CWE-89',  cweTitle: 'SQL Injection',             severity: Severity.CRITICAL },
  'SQL_INJECTION_HIBERNATE':                 { cweId: 'CWE-89',  cweTitle: 'SQL Injection',             severity: Severity.CRITICAL },
  'SQL_INJECTION_SPRING_JDBC':               { cweId: 'CWE-89',  cweTitle: 'SQL Injection',             severity: Severity.CRITICAL },
  'SQL_NONCONSTANT_STRING_PASSED_TO_EXECUTE':{ cweId: 'CWE-89',  cweTitle: 'SQL Injection',             severity: Severity.CRITICAL },
  'HARD_CODE_PASSWORD':                      { cweId: 'CWE-798', cweTitle: 'Hardcoded Credentials',     severity: Severity.CRITICAL },
  'HARD_CODE_KEY':                           { cweId: 'CWE-798', cweTitle: 'Hardcoded Credentials',     severity: Severity.CRITICAL },
  'WEAK_MESSAGE_DIGEST_MD5':                 { cweId: 'CWE-327', cweTitle: 'Weak Cryptographic Algorithm', severity: Severity.HIGH },
  'WEAK_MESSAGE_DIGEST_SHA1':                { cweId: 'CWE-327', cweTitle: 'Weak Cryptographic Algorithm', severity: Severity.HIGH },
  'COOKIE_NO_HTTPONLY':                      { cweId: 'CWE-1004',cweTitle: 'Missing HttpOnly Cookie',   severity: Severity.MEDIUM },
  'INSECURE_COOKIE':                         { cweId: 'CWE-614', cweTitle: 'Insecure Cookie',           severity: Severity.MEDIUM },
  'XSS_REQUEST_PARAMETER_TO_SEND_ERROR':     { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',      severity: Severity.HIGH },
  'XSS_REQUEST_PARAMETER_TO_JSP_WRITER':     { cweId: 'CWE-79',  cweTitle: 'Cross-Site Scripting',      severity: Severity.HIGH },
  'PATH_TRAVERSAL_IN':                       { cweId: 'CWE-22',  cweTitle: 'Path Traversal',            severity: Severity.HIGH },
  'PATH_TRAVERSAL_OUT':                      { cweId: 'CWE-22',  cweTitle: 'Path Traversal',            severity: Severity.HIGH },
  'PREDICTABLE_RANDOM':                      { cweId: 'CWE-330', cweTitle: 'Weak Randomness',           severity: Severity.MEDIUM },
  'XXE_SAXPARSER':                           { cweId: 'CWE-611', cweTitle: 'XML External Entity',       severity: Severity.HIGH },
  'XXE_XMLREADER':                           { cweId: 'CWE-611', cweTitle: 'XML External Entity',       severity: Severity.HIGH },
  'XXE_DOCUMENT':                            { cweId: 'CWE-611', cweTitle: 'XML External Entity',       severity: Severity.HIGH },
  'OBJECT_DESERIALIZATION':                  { cweId: 'CWE-502', cweTitle: 'Insecure Deserialization',   severity: Severity.CRITICAL },
  'UNSAFE_DESERIALIZATION':                  { cweId: 'CWE-502', cweTitle: 'Insecure Deserialization',   severity: Severity.CRITICAL },
}
```

---

## SECTION 9 — REGISTRY MODULE

### File: packages/core/src/registry.ts

```typescript
export class RegistryService {
  private entries: Map<string, RegistryEntry>

  // Load remediator-registry.json at startup
  // Validate all required fields, throw RegistryNotFoundError if missing or malformed
  // Log entry count per language at INFO on successful load
  async load(registryPath: string): Promise<void>

  // Lookup by CWE ID — throws RegistryNotFoundError if not found
  lookup(cweId: string): RegistryEntry

  // Filter entries by language
  byLanguage(lang: Language): RegistryEntry[]

  all(): RegistryEntry[]
}
```

### File: packages/core/remediator-registry.json

Create the full registry with all nine vulnerability categories. Every field must be populated — no empty strings, no TODO placeholders. Include explicit Java and Angular guidance:

```json
{
  "entries": [
    {
      "cweId": "CWE-89",
      "name": "SQL Injection",
      "owaspRef": "A03:2021",
      "owaspUrl": "https://owasp.org/Top10/A03_2021-Injection/",
      "severity": "CRITICAL",
      "guidance": "Never concatenate user input into SQL strings. Use parameterized queries or an ORM. In Java use PreparedStatement or JPA named parameters (@Query with :param). In Python use cursor.execute(sql, params). In Node.js use db.query(sql, [params]). Apply least-privilege to DB accounts — the app user must not have DROP or ALTER. Enable query logging in development to catch regressions early.",
      "fixTemplate": "Python: cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,)). Node.js: db.query('SELECT * FROM users WHERE id = $1', [userId]). Java PreparedStatement: pstmt = conn.prepareStatement('SELECT * FROM users WHERE id = ?'); pstmt.setInt(1, userId). Java JPA: @Query('SELECT u FROM User u WHERE u.id = :id') User findById(@Param('id') Long id);",
      "references": [
        "https://owasp.org/Top10/A03_2021-Injection/",
        "https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/89.html"
      ],
      "languages": ["python", "typescript", "javascript", "angular", "java"]
    },
    {
      "cweId": "CWE-798",
      "name": "Hardcoded credentials",
      "owaspRef": "A02:2021",
      "owaspUrl": "https://owasp.org/Top10/A02_2021-Cryptographic_Failures/",
      "severity": "CRITICAL",
      "guidance": "Never hardcode secrets, API keys, passwords, tokens, or connection strings. In Java use System.getenv() or Spring @Value('${secret.key}'). In Python use os.environ.get(). In Node.js use process.env. For production use a secrets manager (AWS Secrets Manager, Azure Key Vault, HashiCorp Vault). Rotate any secret that has been committed immediately — treat it as compromised even if the commit was private.",
      "fixTemplate": "Java: String secret = System.getenv('SECRET_KEY'); or @Value('${app.secret}') private String secret; Python: secret = os.environ.get('SECRET_KEY') Node.js: const secret = process.env.SECRET_KEY. Add to .env.example with placeholder. Add actual value to secrets manager.",
      "references": [
        "https://owasp.org/Top10/A02_2021-Cryptographic_Failures/",
        "https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/798.html"
      ],
      "languages": ["python", "typescript", "javascript", "angular", "java"]
    },
    {
      "cweId": "CWE-287",
      "name": "Broken authentication",
      "owaspRef": "A07:2021",
      "owaspUrl": "https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/",
      "severity": "HIGH",
      "guidance": "JWT tokens must always specify an expiry. Never use the none algorithm. Verify the signature with a strong secret (HS256 minimum, RS256 preferred). In Java use Spring Security with JwtDecoder and set setJwtValidators including NimbusJwtDecoder. Validate iss and aud claims. Short-lived access tokens (15 min) with refresh token rotation.",
      "fixTemplate": "Node.js: jwt.sign(payload, process.env.JWT_SECRET, { expiresIn: '15m', algorithm: 'HS256' }). Python: jwt.encode(payload, os.environ['JWT_SECRET'], algorithm='HS256', headers={'exp': ...}). Java Spring Security: NimbusJwtDecoder.withSecretKey(key).build() plus JwtTimestampValidator(Duration.ofMinutes(15)).",
      "references": [
        "https://owasp.org/Top10/A07_2021-Identification_and_Authentication_Failures/",
        "https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/287.html"
      ],
      "languages": ["python", "typescript", "javascript", "angular", "java"]
    },
    {
      "cweId": "CWE-937",
      "name": "Vulnerable dependency",
      "owaspRef": "A06:2021",
      "owaspUrl": "https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/",
      "severity": "HIGH",
      "guidance": "Keep all dependencies updated. Run npm audit (Node.js), pip-audit (Python), or OWASP Dependency-Check (Java/Maven/Gradle) in CI on every PR. Pin versions in lock files. For Java use mvn versions:use-latest-releases or the OWASP dependency-check-maven plugin. For critical CVEs update immediately. Use Dependabot or Renovate for automated update PRs.",
      "fixTemplate": "Node.js: npm update package-name@safe-version then verify with npm audit. Python: pip install package-name==safe-version then verify with pip-audit. Java/Maven: update <version> in pom.xml to the patched version specified in the CVE advisory, then run mvn dependency:tree to check for transitive issues.",
      "references": [
        "https://owasp.org/Top10/A06_2021-Vulnerable_and_Outdated_Components/",
        "https://owasp.org/www-project-dependency-check/",
        "https://cwe.mitre.org/data/definitions/937.html"
      ],
      "languages": ["python", "typescript", "javascript", "angular", "java"]
    },
    {
      "cweId": "CWE-79",
      "name": "Cross-site scripting (XSS)",
      "owaspRef": "A03:2021",
      "owaspUrl": "https://owasp.org/Top10/A03_2021-Injection/",
      "severity": "HIGH",
      "guidance": "Never insert unescaped user input into HTML. In Angular use text binding {{variable}} not [innerHTML]. Never call bypassSecurityTrustHtml unless the input has been sanitized by DOMPurify. In React use JSX {variable} not dangerouslySetInnerHTML. In Java/JSP use JSTL <c:out> or fn:escapeXml() and Thymeleaf th:text not th:utext. Implement Content-Security-Policy headers. Set httpOnly and SameSite on cookies.",
      "fixTemplate": "Angular: replace [innerHTML]='userInput' with {{userInput}} in the template. Remove DomSanitizer.bypassSecurityTrustHtml calls — let Angular sanitize automatically. React: use {variable} not dangerouslySetInnerHTML. Java JSP: use <c:out value='${userInput}'/> not <%= userInput %>. Thymeleaf: th:text='${variable}' not th:utext.",
      "references": [
        "https://owasp.org/Top10/A03_2021-Injection/",
        "https://angular.dev/best-practices/security",
        "https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/79.html"
      ],
      "languages": ["python", "typescript", "javascript", "angular", "java"]
    },
    {
      "cweId": "CWE-352",
      "name": "Cross-Site Request Forgery (CSRF)",
      "owaspRef": "A01:2021",
      "owaspUrl": "https://owasp.org/Top10/A01_2021-Broken_Access_Control/",
      "severity": "HIGH",
      "guidance": "Angular's HttpClient does NOT add CSRF tokens automatically — you must configure HttpClientXsrfModule. Set withCredentials: true on state-changing requests. For Java Spring Security, do not call csrf().disable() — use CookieCsrfTokenRepository.withHttpOnlyFalse() so Angular can read the XSRF-TOKEN cookie. SameSite=Strict on session cookies provides defence-in-depth.",
      "fixTemplate": "Angular app.module.ts: import HttpClientXsrfModule and add HttpClientXsrfModule.withOptions({ cookieName: 'XSRF-TOKEN', headerName: 'X-XSRF-TOKEN' }) to imports[]. Java Spring Security: .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())). Remove any .csrf(AbstractHttpConfigurer::disable) calls.",
      "references": [
        "https://owasp.org/Top10/A01_2021-Broken_Access_Control/",
        "https://angular.dev/best-practices/security#cross-site-request-forgery",
        "https://docs.spring.io/spring-security/reference/features/exploits/csrf.html",
        "https://cwe.mitre.org/data/definitions/352.html"
      ],
      "languages": ["angular", "java"]
    },
    {
      "cweId": "CWE-611",
      "name": "XML External Entity (XXE)",
      "owaspRef": "A05:2021",
      "owaspUrl": "https://owasp.org/Top10/A05_2021-Security_Misconfiguration/",
      "severity": "HIGH",
      "guidance": "Java XML parsers enable external entity resolution by default — this must be disabled explicitly. Set FEATURE_SECURE_PROCESSING and XMLConstants.FEATURE_SECURE_PROCESSING on every SAXParserFactory, DocumentBuilderFactory, XMLInputFactory, and SAXReader. Never use xml.etree.ElementTree in Python for untrusted XML — use defusedxml instead.",
      "fixTemplate": "Java SAXParserFactory: factory.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true); factory.setFeature('http://apache.org/xml/features/disallow-doctype-decl', true); Java DocumentBuilderFactory: dbf.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true); dbf.setExpandEntityReferences(false); Java XMLInputFactory: xif.setProperty(XMLInputFactory.SUPPORT_DTD, false); Python: import defusedxml.ElementTree as ET instead of xml.etree.ElementTree.",
      "references": [
        "https://owasp.org/Top10/A05_2021-Security_Misconfiguration/",
        "https://cheatsheetseries.owasp.org/cheatsheets/XML_External_Entity_Prevention_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/611.html"
      ],
      "languages": ["java", "python"]
    },
    {
      "cweId": "CWE-502",
      "name": "Insecure deserialization",
      "owaspRef": "A08:2021",
      "owaspUrl": "https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/",
      "severity": "CRITICAL",
      "guidance": "Never deserialize data from untrusted sources using Java's ObjectInputStream. Use JSON (Jackson, Gson) or Protocol Buffers instead. If ObjectInputStream is unavoidable, implement a ValidatingObjectInputStream that only allows a specific allowlist of classes. Add the SerialKiller or NotSoSerial agent in production. Never deserialize data unless you control the source completely.",
      "fixTemplate": "Replace ObjectInputStream deserialization with Jackson: ObjectMapper mapper = new ObjectMapper(); MyClass obj = mapper.readValue(json, MyClass.class); If ObjectInputStream is unavoidable, use Apache Commons IO ValidatingObjectInputStream: ValidatingObjectInputStream vois = new ValidatingObjectInputStream(is); vois.accept(MyAllowedClass.class); MyAllowedClass obj = (MyAllowedClass) vois.readObject();",
      "references": [
        "https://owasp.org/Top10/A08_2021-Software_and_Data_Integrity_Failures/",
        "https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html",
        "https://cwe.mitre.org/data/definitions/502.html"
      ],
      "languages": ["java"]
    }
  ]
}
```

---

## SECTION 10 — AI REMEDIATOR MODULE

### File: packages/core/src/remediator.ts

```typescript
export class RemediatorService {

  // Build Claude prompt — language-aware
  // Includes: finding details, vulnerable snippet, surrounding context,
  //           registry guidance, fix template, language-specific examples
  // For Java: include import statements needed for the fix
  // For Angular: include whether the fix is in .ts or .html file
  // Keep total under 4000 tokens
  buildPrompt(finding: Finding, entry: RegistryEntry, surroundingCode?: string): string

  // Call Claude API with retry
  // Model: process.env.CLAUDE_MODEL || 'claude-sonnet-4-6'
  // max_tokens: 1500
  // stream: true for Copilot Chat, false for PR batch
  // Retry on LLMRateLimitError up to 3 times
  async callClaude(prompt: string, stream: boolean, log: Logger): Promise<string | AsyncIterable<string>>

  // Parse <explanation> <patch> <test> XML tags from response
  // If parsing fails: log WARN, return raw as explanation, empty patch + test
  parseResponse(raw: string, finding: Finding): Remediation

  async remediate(finding: Finding, stream: boolean, log: Logger): Promise<Remediation>

  // Max 5 concurrent Claude calls; partial failures logged + skipped
  async remediateAll(findings: Finding[], log: Logger): Promise<Remediation[]>
}
```

**Claude prompt template — implement this exactly, with language-specific sections:**

```
You are SecureGuard AI, an expert application security engineer specialising in {language} applications. A security scanner has detected a vulnerability. Your job is to explain it clearly and provide a working fix in {language}.

VULNERABILITY DETECTED:
- Type: {cweTitle} ({cweId})
- OWASP Reference: {owaspRef}
- Severity: {severity}
- File: {filePath}, Line {lineStart}
- Language: {language}

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

STANDARD FIX APPROACH FOR {language}:
{fixTemplate}

{languageSpecificNote}

YOUR TASK:
Respond with exactly three sections using these XML tags. No text outside the tags.

<explanation>
2-4 sentences in plain English a developer understands immediately.
Explain: (1) what the vulnerability is, (2) how an attacker exploits it in THIS specific code, (3) what the impact would be.
No jargon. Write as if explaining to a smart developer who is not a security expert.
</explanation>

<patch>
The corrected version of the vulnerable code in {language}.
Show the complete fixed function or block — not just the changed line.
For Java: include necessary import statements at the top.
For Angular: if the fix spans .ts and .html, label each section with // --- TypeScript --- and // --- Template ---.
Include a brief inline comment explaining what changed and why.
</patch>

<test>
A single unit test that catches this vulnerability.
Must actually test the security property, not just call the function.
For Java: use JUnit 5 + Mockito. For Angular: use Jasmine + TestBed. For Python: use pytest. For Node.js: use Jest.
</test>
```

**Language-specific notes to inject into `{languageSpecificNote}`:**

- Java: "NOTE: For Java fixes, include the full import statement for any new class used (e.g. import java.sql.PreparedStatement). Show the complete method, not just the changed line. If the fix requires a Spring Security configuration change, show the SecurityConfig class snippet."
- Angular: "NOTE: For Angular fixes, distinguish between TypeScript component changes and HTML template changes. If the fix requires an AppModule change (e.g. HttpClientXsrfModule), show the module import snippet too."
- Python: "NOTE: For Python fixes using defusedxml, include the pip install command as a comment."
- TypeScript/JavaScript: "NOTE: For Node.js fixes, show the complete function with the parameterized query pattern."

---

## SECTION 11 — POLICY ENGINE

### File: packages/core/src/policy.ts

```typescript
export interface PolicyRules {
  blockOnSeverities:  Severity[]   // default: [CRITICAL, HIGH]
  allowOverrideLabel: string        // default: 'security-override'
  maxAllowedMedium:   number        // default: 10
}

export class PolicyEngine {
  evaluate(findings: Finding[], rules: PolicyRules, overrideLabel?: string): PolicyResult
  async checkOverrideLabel(prNumber: number, repo: string, token: string): Promise<string | undefined>
  async setCommitStatus(result: PolicyResult, sha: string, repo: string, token: string, log: Logger): Promise<void>
}

export const DEFAULT_POLICY_RULES: PolicyRules = {
  blockOnSeverities:  [Severity.CRITICAL, Severity.HIGH],
  allowOverrideLabel: process.env.POLICY_OVERRIDE_LABEL || 'security-override',
  maxAllowedMedium:   parseInt(process.env.POLICY_MAX_MEDIUM || '10')
}
```

---

## SECTION 12 — AUDIT MODULE

### File: packages/core/src/audit.ts

Use `better-sqlite3`. Schema is Postgres-compatible for production migration. Add `language` column to `audit_records`. The `findings` table includes a `language` column.

```sql
CREATE TABLE IF NOT EXISTS audit_records (
  audit_id                TEXT PRIMARY KEY,
  request_id              TEXT NOT NULL,
  source                  TEXT NOT NULL,
  language                TEXT NOT NULL,
  developer               TEXT,
  repo_path               TEXT NOT NULL,
  scan_duration_ms        INTEGER NOT NULL,
  findings_count          INTEGER NOT NULL,
  findings_critical       INTEGER NOT NULL DEFAULT 0,
  findings_high           INTEGER NOT NULL DEFAULT 0,
  findings_medium         INTEGER NOT NULL DEFAULT 0,
  findings_low            INTEGER NOT NULL DEFAULT 0,
  policy_decision         TEXT NOT NULL,
  blocked_findings_count  INTEGER NOT NULL DEFAULT 0,
  override_label          TEXT,
  override_approver       TEXT,
  remediations_generated  INTEGER NOT NULL DEFAULT 0,
  llm_called              INTEGER NOT NULL DEFAULT 0,
  created_at              TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS findings (
  finding_id   TEXT PRIMARY KEY,
  audit_id     TEXT NOT NULL REFERENCES audit_records(audit_id),
  cwe_id       TEXT NOT NULL,
  cwe_title    TEXT NOT NULL,
  severity     TEXT NOT NULL,
  file_path    TEXT NOT NULL,
  line_start   INTEGER NOT NULL,
  scanner      TEXT NOT NULL,
  rule_id      TEXT NOT NULL,
  language     TEXT NOT NULL,
  detected_at  TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS remediations (
  remediation_id   TEXT PRIMARY KEY,
  finding_id       TEXT NOT NULL REFERENCES findings(finding_id),
  explanation      TEXT NOT NULL,
  patch            TEXT NOT NULL,
  test_stub        TEXT NOT NULL,
  model            TEXT NOT NULL,
  prompt_tokens    INTEGER NOT NULL,
  completion_tokens INTEGER NOT NULL,
  latency_ms       INTEGER NOT NULL,
  generated_at     TEXT NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_audit_developer ON audit_records(developer);
CREATE INDEX IF NOT EXISTS idx_audit_repo      ON audit_records(repo_path);
CREATE INDEX IF NOT EXISTS idx_audit_language  ON audit_records(language);
CREATE INDEX IF NOT EXISTS idx_audit_created   ON audit_records(created_at);
CREATE INDEX IF NOT EXISTS idx_findings_audit  ON findings(audit_id);
CREATE INDEX IF NOT EXISTS idx_findings_cwe    ON findings(cwe_id);
CREATE INDEX IF NOT EXISTS idx_findings_lang   ON findings(language);
```

Implement `AuditService` with `initialize`, `write`, `findByDeveloper`, `findByRepo`, `getSummary`. The `getSummary` method must break down `findingsByLanguage` in addition to `findingsBySeverity`.

---

## SECTION 13 — MIDDLEWARE

### File: packages/core/src/middleware/auth.ts
Validate `X-SecureGuard-Token` against `SECUREGUARD_API_KEY` env var. For Copilot Extension requests validate the Copilot skill token. Skip auth for `/health` and `/metrics`.

### File: packages/core/src/middleware/requestLogger.ts
Log every request: method, path, requestId, IP, user-agent, detected language (if present). Log every response: status, duration ms, requestId. Attach requestId to `res.locals`.

### File: packages/core/src/middleware/rateLimiter.ts
Use `express-rate-limit`:
- `/scan`: 60 req/min per IP
- `/copilot`: 20 req/min per user
- `/pr`: 10 req/min per repo
Return 429 with `Retry-After` header and structured error body.

---

## SECTION 14 — SERVER AND ROUTER

### File: packages/core/src/server.ts

```typescript
// Middleware order:
// 1. helmet()
// 2. compression()
// 3. express.json({ limit: '5mb' })
// 4. requestLogger
// 5. rateLimiter
// 6. auth (skip /health, /metrics)

// Routes:
// GET  /health   → { status, version, uptime, timestamp, supportedLanguages }
// GET  /metrics  → audit summary + breakdown by language
// POST /scan     → no-LLM path (IDE + commit + CLI)
// POST /copilot  → LLM streaming path
// POST /pr       → LLM batch path + GitHub API

// Graceful shutdown: SIGTERM + SIGINT, drain in-flight, close DB
```

### File: packages/core/src/router.ts

```typescript
// POST /scan handler:
// 1. Validate ScanRequest — throw InvalidRequestError if required fields missing
// 2. Detect languages via languageDetector.detect()
// 3. Create request logger with requestId
// 4. Run scanner.scan() with detected languages
// 5. Run normalizer.normalize()
// 6. Run policy.evaluate() — advisory only at this route (never block)
// 7. Format with lsp.ts or cli.ts based on source
// 8. Write audit record (fire-and-forget, do not await)
// 9. Return formatted findings

// POST /copilot handler:
// 1. Set SSE headers
// 2. Detect language from context.currentFile
// 3. Parse intent (scan / fix / explain / repo)
// 4. Run scanner with detected language
// 5. For HIGH/CRITICAL: remediator.remediate(stream: true)
// 6. Stream tokens as: data: {token}\n\n
// 7. Format as GFM markdown with language-specific code fences
// 8. End with: data: [DONE]\n\n
// 9. Write audit record in background

// POST /pr handler:
// 1. Validate: prNumber, repo, githubToken, changedFiles required
// 2. Detect languages from changedFiles extensions + repoRoot
// 3. Full deep scan for all detected languages
// 4. Check override label
// 5. Evaluate policy
// 6. remediateAll() for HIGH/CRITICAL
// 7. Post PR review comments with inline suggestions
// 8. Set commit status
// 9. Write audit record
// 10. Return { decision, findingsCount, remediationsPosted, languagesScanned }
```

---

## SECTION 15 — FORMATTERS

### File: packages/core/src/formatters/lsp.ts
Convert `Finding[]` to LSP `Diagnostic[]`. Severity map: CRITICAL/HIGH → Error (1), MEDIUM → Warning (2), LOW/INFO → Information (3). Message: `[{severity}] {cweTitle} ({language}) — {one-line hint}`. Include `code: cweId`, `source: 'secureguard'`.

### File: packages/core/src/formatters/cli.ts
Use `chalk`. CRITICAL: red background. HIGH: red text. MEDIUM: yellow. LOW: gray. Include detected language in the summary line: `SecureGuard found N findings in {language} files (X critical, Y high)`.

### File: packages/core/src/formatters/copilot.ts
GFM markdown. Use language-correct code fences (`java`, `typescript`, `python`). For Angular findings that span TS and HTML, use two separate code blocks labelled accordingly. Include OWASP reference URL.

### File: packages/core/src/formatters/pr.ts
GitHub Suggestions API format. For Java: suggest the PreparedStatement replacement in the diff. For Angular: if fix spans both .ts and .html, post two separate review comments — one per file. If patch is too complex for a suggestion block, post as a regular comment.

---

## SECTION 16 — SEMGREP CUSTOM RULES

### File: packages/core/semgrep-rules/sql-injection.yaml

```yaml
rules:
  - id: secureguard-python-sql-injection-fstring
    languages: [python]
    severity: ERROR
    message: "SQL Injection (CWE-89): f-string builds SQL query. Use parameterized queries."
    patterns:
      - pattern: $CURSOR.execute(f"...{$VAR}...")
      - pattern: $CURSOR.execute("..." + $VAR + "...")
    metadata:
      cwe: CWE-89
      owasp: A03:2021

  - id: secureguard-js-sql-injection-template
    languages: [javascript, typescript]
    severity: ERROR
    message: "SQL Injection (CWE-89): Template literal in SQL query. Use parameterized queries."
    patterns:
      - pattern: $DB.query(`...${$VAR}...`)
      - pattern: $DB.execute(`...${$VAR}...`)
    metadata:
      cwe: CWE-89
      owasp: A03:2021

  - id: secureguard-java-jdbc-sqli
    languages: [java]
    severity: ERROR
    message: "SQL Injection (CWE-89): String concatenation in JDBC query. Use PreparedStatement."
    patterns:
      - pattern: $STMT.execute("..." + $VAR + "...")
      - pattern: $STMT.executeQuery("..." + $VAR + "...")
      - pattern: $STMT.executeUpdate("..." + $VAR + "...")
    metadata:
      cwe: CWE-89
      owasp: A03:2021
```

### File: packages/core/semgrep-rules/angular-security.yaml

```yaml
rules:
  - id: secureguard-angular-bypass-sanitizer
    languages: [typescript]
    severity: ERROR
    message: "XSS (CWE-79): bypassSecurityTrustHtml disables Angular's built-in XSS sanitization. Remove this call and let Angular sanitize automatically."
    patterns:
      - pattern: this.$SAN.bypassSecurityTrustHtml($INPUT)
      - pattern: $SAN.bypassSecurityTrustHtml($INPUT)
    metadata:
      cwe: CWE-79
      owasp: A03:2021

  - id: secureguard-angular-innerhtml-binding
    languages: [typescript]
    severity: WARNING
    message: "XSS (CWE-79): [innerHTML] binding with user data bypasses Angular sanitization. Use text binding {{variable}} instead."
    pattern: |
      @Component({...})
      class $CLASS {
        ...
        $FIELD = $SANITIZER.bypassSecurityTrustHtml(...)
        ...
      }
    metadata:
      cwe: CWE-79
      owasp: A03:2021

  - id: secureguard-angular-no-csrf-header
    languages: [typescript]
    severity: WARNING
    message: "CSRF (CWE-352): HttpClient POST/PUT/DELETE without XSRF token. Configure HttpClientXsrfModule in AppModule."
    patterns:
      - pattern: this.http.post($URL, $BODY)
      - pattern: this.http.put($URL, $BODY)
      - pattern: this.http.delete($URL)
    pattern-not:
      - pattern: this.http.post($URL, $BODY, {headers: $H, ...})
    metadata:
      cwe: CWE-352
      owasp: A01:2021

  - id: secureguard-angular-href-injection
    languages: [typescript]
    severity: WARNING
    message: "XSS (CWE-79): Dynamic href from user input can execute javascript: URLs. Validate or use Angular Router."
    pattern: |
      $EL.href = $USER_VAR
    metadata:
      cwe: CWE-79
      owasp: A03:2021
```

### File: packages/core/semgrep-rules/java-security.yaml

```yaml
rules:
  - id: secureguard-java-hardcoded-password
    languages: [java]
    severity: ERROR
    message: "Hardcoded credentials (CWE-798): String literal used as password. Use System.getenv() or Spring @Value."
    patterns:
      - pattern: String $PASSWORD = "..."
      - pattern-regex: '(?i)(password|passwd|secret|key|token)\s*=\s*"[^"]+"'
    metadata:
      cwe: CWE-798
      owasp: A02:2021

  - id: secureguard-java-objectinputstream
    languages: [java]
    severity: ERROR
    message: "Insecure Deserialization (CWE-502): ObjectInputStream.readObject() on untrusted data enables remote code execution."
    patterns:
      - pattern: |
          ObjectInputStream $OIS = new ObjectInputStream($INPUT);
          $OIS.readObject()
      - pattern: (new ObjectInputStream($INPUT)).readObject()
    metadata:
      cwe: CWE-502
      owasp: A08:2021

  - id: secureguard-java-spring-csrf-disabled
    languages: [java]
    severity: ERROR
    message: "CSRF disabled (CWE-352): csrf().disable() removes Spring Security CSRF protection. Use CookieCsrfTokenRepository instead."
    pattern: .csrf(AbstractHttpConfigurer::disable)
    metadata:
      cwe: CWE-352
      owasp: A01:2021
```

### File: packages/core/semgrep-rules/java-xxe.yaml

```yaml
rules:
  - id: secureguard-java-xxe-saxparser
    languages: [java]
    severity: ERROR
    message: "XXE (CWE-611): SAXParserFactory without FEATURE_SECURE_PROCESSING. Attackers can read server files."
    patterns:
      - pattern: |
          SAXParserFactory $FACTORY = SAXParserFactory.newInstance();
          ...
          $FACTORY.newSAXParser()
      - pattern-not: $FACTORY.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true)
    metadata:
      cwe: CWE-611
      owasp: A05:2021

  - id: secureguard-java-xxe-documentbuilder
    languages: [java]
    severity: ERROR
    message: "XXE (CWE-611): DocumentBuilderFactory without FEATURE_SECURE_PROCESSING is vulnerable to XXE attacks."
    patterns:
      - pattern: |
          DocumentBuilderFactory $DBF = DocumentBuilderFactory.newInstance();
          ...
          $DBF.newDocumentBuilder()
      - pattern-not: $DBF.setFeature(XMLConstants.FEATURE_SECURE_PROCESSING, true)
    metadata:
      cwe: CWE-611
      owasp: A05:2021
```

### File: packages/core/semgrep-rules/hardcoded-secrets.yaml
Implement rules for all five languages detecting hardcoded passwords, API keys, tokens, and connection strings. Use pattern-regex for common secret patterns.

### File: packages/core/semgrep-rules/weak-auth.yaml
Python JWT none-alg, no expiry. Node.js JWT missing expiresIn. Java Spring Security JWT without expiry validation.

### File: packages/core/semgrep-rules/xss.yaml
Python Jinja2 | safe filter, JavaScript innerHTML, React dangerouslySetInnerHTML, Angular innerHTML (covered also in angular-security.yaml — include both for completeness).

### File: packages/core/spotbugs/secureguard-exclude.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<FindBugsFilter>
  <!-- Exclude test classes from SpotBugs analysis -->
  <Match>
    <Class name="~.*Test$"/>
  </Match>
  <Match>
    <Class name="~.*Tests$"/>
  </Match>
  <!-- Exclude generated code -->
  <Match>
    <Class name="~.*Generated.*"/>
  </Match>
</FindBugsFilter>
```

---

## SECTION 17 — VS CODE EXTENSION

### File: packages/vscode-extension/src/extension.ts

```typescript
// activate(context):
// 1. Check /health — start server if not running
// 2. Register LSP client (lspClient.ts) — watch .py, .ts, .tsx, .js, .java, .component.ts files
// 3. Register @secureguard Copilot Chat participant
// 4. Register commands: secureguard.scanFile, secureguard.scanRepo, secureguard.showAuditLog
// 5. Status bar: "SecureGuard: Active | {N} findings | {languages}"
```

### File: packages/vscode-extension/src/lspClient.ts

```typescript
// DocumentFilter covers all supported languages:
// { scheme: 'file', language: 'python' }
// { scheme: 'file', language: 'typescript' }
// { scheme: 'file', language: 'javascript' }
// { scheme: 'file', language: 'java' }
// Trigger scan onDidSave with 500ms debounce
// For Java: skip scan if no compiled classes exist (too slow to compile on save)
//           show hint: "Run 'mvn compile' then save again for Java analysis"
```

---

## SECTION 18 — CLI

### File: packages/cli/src/index.ts

```bash
secureguard scan [path]          # auto-detect language, scan path
  --staged                       # git staged files only
  --language <lang>              # override: python|typescript|javascript|angular|java
  --fail-on <severity>           # exit 1 if severity found (default: CRITICAL)
  --output <format>              # json | text (default: text)
  --mode <mode>                  # fast | full (default: fast)
  # Java note: full mode triggers mvn compile before SpotBugs

secureguard explain <findingId>  # AI explanation for finding ID

secureguard install-hook         # write .git/hooks/pre-commit
  # Java note: pre-commit hook uses fast mode (Semgrep only) — no compile step

secureguard audit [--since <date>] [--language <lang>] [--repo <path>]
```

---

## SECTION 19 — GITHUB ACTIONS

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
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup Java (for Java repos)
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Install Semgrep
        run: pip install semgrep

      - name: Install Gitleaks
        run: |
          wget -q https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz
          tar -xzf gitleaks_linux_x64.tar.gz && sudo mv gitleaks /usr/local/bin/

      - name: Install pip-audit and Bandit
        run: pip install pip-audit bandit

      - name: Install SpotBugs (Java)
        run: |
          wget -q https://github.com/spotbugs/spotbugs/releases/download/4.8.3/spotbugs-4.8.3.tgz
          tar -xzf spotbugs-4.8.3.tgz
          sudo ln -s $(pwd)/spotbugs-4.8.3/bin/spotbugs /usr/local/bin/spotbugs

      - name: Install OWASP Dependency-Check (Java)
        run: |
          wget -q https://github.com/jeremylong/DependencyCheck/releases/download/v9.0.9/dependency-check-9.0.9-release.zip
          unzip -q dependency-check-9.0.9-release.zip
          sudo ln -s $(pwd)/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check

      - name: Detect languages in PR
        id: detect
        run: |
          FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          HAS_JAVA=$(echo "$FILES" | grep -c '\.java$' || true)
          HAS_ANGULAR=$(echo "$FILES" | grep -c '\.component\.ts$\|angular\.json' || true)
          HAS_PY=$(echo "$FILES" | grep -c '\.py$' || true)
          echo "has_java=$HAS_JAVA" >> $GITHUB_OUTPUT
          echo "has_angular=$HAS_ANGULAR" >> $GITHUB_OUTPUT
          echo "has_python=$HAS_PY" >> $GITHUB_OUTPUT

      - name: Compile Java (if Java files changed)
        if: steps.detect.outputs.has_java != '0'
        run: mvn compile -q
        continue-on-error: true

      - name: Start SecureGuard server
        run: |
          cd packages/core && npm ci && npm run build
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

## SECTION 20 — DEMO APPLICATION

Create intentionally vulnerable files for every supported language. Every file must contain real, detectable vulnerabilities that SecureGuard will find and fix in the demo.

### Python demo files (unchanged from original spec)

**demo-app/python/routes/users.py** — SQL injection line 23:
```python
import sqlite3

def get_user(user_id):
    conn = sqlite3.connect('users.db')
    cursor = conn.cursor()
    query = f"SELECT * FROM users WHERE id = {user_id}"  # BUG: CWE-89
    cursor.execute(query)
    return cursor.fetchone()
```

**demo-app/python/routes/auth.py** — hardcoded secret + weak JWT:
```python
import jwt
SECRET_KEY = "super_secret_key_123"  # BUG: CWE-798

def create_token(user_id):
    token = jwt.encode({"user_id": user_id}, SECRET_KEY)  # BUG: no expiry CWE-287
    return token
```

**demo-app/python/requirements.txt** — vulnerable deps:
```
django==3.2.0
requests==2.25.0
```

### TypeScript demo files

**demo-app/typescript/routes/render.ts** — XSS:
```typescript
export function renderUserProfile(req: Request, res: Response) {
  const username = req.query.name as string;
  res.send(`<div><h1>Hello ${username}</h1></div>`);  // BUG: CWE-79
}
```

**demo-app/typescript/routes/users.ts** — SQL injection:
```typescript
import { pool } from '../db';
export async function getUser(userId: string) {
  const result = await pool.query(`SELECT * FROM users WHERE id = ${userId}`);  // BUG: CWE-89
  return result.rows[0];
}
```

### Angular demo files (NEW)

**demo-app/angular/src/app/user-profile/user-profile.component.ts**:
```typescript
import { Component, OnInit } from '@angular/core';
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';
import { ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-user-profile',
  templateUrl: './user-profile.component.html'
})
export class UserProfileComponent implements OnInit {
  userBio: SafeHtml = '';

  constructor(
    private sanitizer: DomSanitizer,
    private route: ActivatedRoute
  ) {}

  ngOnInit() {
    const bio = this.route.snapshot.queryParamMap.get('bio') || '';
    // BUG: CWE-79 — bypasses Angular's built-in XSS protection
    this.userBio = this.sanitizer.bypassSecurityTrustHtml(bio);
  }
}
```

**demo-app/angular/src/app/user-profile/user-profile.component.html**:
```html
<!-- BUG: CWE-79 — [innerHTML] with bypassed sanitization -->
<div class="profile">
  <div [innerHTML]="userBio"></div>
</div>
```

**demo-app/angular/src/app/api/data.service.ts**:
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';

@Injectable({ providedIn: 'root' })
export class DataService {
  constructor(private http: HttpClient) {}

  // BUG: CWE-352 — POST without CSRF token (HttpClientXsrfModule not configured)
  updateProfile(data: any) {
    return this.http.post('/api/profile', data);
  }
}
```

**demo-app/angular/src/app/app.module.ts** — missing HttpClientXsrfModule:
```typescript
import { NgModule } from '@angular/core';
import { HttpClientModule } from '@angular/common/http';
// BUG: HttpClientXsrfModule not imported — CSRF protection absent

@NgModule({
  imports: [
    HttpClientModule
    // Missing: HttpClientXsrfModule.withOptions({ cookieName: 'XSRF-TOKEN', headerName: 'X-XSRF-TOKEN' })
  ]
})
export class AppModule {}
```

**demo-app/angular/package.json** — vulnerable deps:
```json
{
  "dependencies": {
    "@angular/core": "14.0.0",
    "rxjs": "6.6.0"
  }
}
```

### Java demo files (NEW)

**demo-app/java/src/main/java/com/demo/UserController.java** — SQL injection:
```java
package com.demo;
import java.sql.*;

public class UserController {
    private Connection conn;

    public User getUser(String userId) throws SQLException {
        Statement stmt = conn.createStatement();
        // BUG: CWE-89 — string concatenation in SQL query
        ResultSet rs = stmt.executeQuery(
            "SELECT * FROM users WHERE id = " + userId
        );
        return mapResultSet(rs);
    }
}
```

**demo-app/java/src/main/java/com/demo/XmlParser.java** — XXE:
```java
package com.demo;
import javax.xml.parsers.*;
import org.w3c.dom.Document;
import java.io.InputStream;

public class XmlParser {
    public Document parse(InputStream input) throws Exception {
        // BUG: CWE-611 — DocumentBuilderFactory without FEATURE_SECURE_PROCESSING
        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        DocumentBuilder db = dbf.newDocumentBuilder();
        return db.parse(input);
    }
}
```

**demo-app/java/src/main/java/com/demo/AuthService.java** — hardcoded credentials:
```java
package com.demo;

public class AuthService {
    // BUG: CWE-798 — hardcoded database password
    private static final String DB_PASSWORD = "admin123";
    private static final String JWT_SECRET  = "my-secret-key";

    public Connection getConnection() throws Exception {
        return DriverManager.getConnection(
            "jdbc:mysql://localhost/mydb", "admin", DB_PASSWORD
        );
    }
}
```

**demo-app/java/src/main/java/com/demo/DataProcessor.java** — insecure deserialization:
```java
package com.demo;
import java.io.*;

public class DataProcessor {
    public Object processData(InputStream input) throws Exception {
        // BUG: CWE-502 — ObjectInputStream on untrusted data enables RCE
        ObjectInputStream ois = new ObjectInputStream(input);
        return ois.readObject();
    }
}
```

**demo-app/java/pom.xml** — vulnerable dependencies:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.demo</groupId>
  <artifactId>secureguard-demo</artifactId>
  <version>1.0.0</version>
  <dependencies>
    <!-- BUG: CWE-937 — Log4j 2.14.1 has CVE-2021-44228 (Log4Shell) -->
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-core</artifactId>
      <version>2.14.1</version>
    </dependency>
    <!-- BUG: Spring Framework 5.3.0 has known CVEs -->
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-webmvc</artifactId>
      <version>5.3.0</version>
    </dependency>
  </dependencies>
</project>
```

---

## SECTION 21 — CONFIGURATION AND ENVIRONMENT

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

# Scanner timeouts
SEMGREP_TIMEOUT_MS=30000
GITLEAKS_TIMEOUT_MS=15000
BANDIT_TIMEOUT_MS=20000
PIP_AUDIT_TIMEOUT_MS=20000
SPOTBUGS_COMPILE_TIMEOUT_MS=120000   # Java compile can be slow
SPOTBUGS_SCAN_TIMEOUT_MS=60000
OWASP_DC_TIMEOUT_MS=180000           # Dep-Check is slow — 3 min default

# Policy
POLICY_BLOCK_ON=CRITICAL,HIGH
POLICY_MAX_MEDIUM=10
POLICY_OVERRIDE_LABEL=security-override

# LLM
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

## SECTION 22 — PACKAGE.JSON FILES

### Root package.json:
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

### packages/core/package.json:
```json
{
  "name": "@secureguard/core",
  "version": "2.0.0",
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
    "uuid": "^9.0.1",
    "xml2js": "^0.6.2"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.8",
    "@types/compression": "^1.7.5",
    "@types/express": "^4.17.21",
    "@types/node": "^20.11.0",
    "@types/uuid": "^9.0.7",
    "@types/xml2js": "^0.4.14",
    "jest": "^29.7.0",
    "ts-jest": "^29.1.2",
    "ts-node-dev": "^2.0.0",
    "typescript": "^5.3.3"
  }
}
```

Note: `xml2js` is added to parse SpotBugs XML output.

---

## SECTION 23 — TESTS

Create tests for every module using Jest + ts-jest. Mock Claude API with `jest.mock('@anthropic-ai/sdk')` and subprocess execution with `jest.mock('child_process')`.

Required test files:
- `languageDetector.test.ts` — test detection for .java, .component.ts, angular.json, .py, .ts, mixed repos
- `normalizer.test.ts` — test CWE mapping for all languages including SpotBugs patterns, dedup, truncation
- `registry.test.ts` — test load, lookup, byLanguage filter, missing entry error
- `policy.test.ts` — test PASS/BLOCK/OVERRIDE for all severity combinations
- `remediator.test.ts` — test language-aware prompt building (Java imports, Angular TS+HTML labels), response parsing
- `audit.test.ts` — test write, read, getSummary with language breakdown, in-memory SQLite
- `scanner.test.ts` — test: SpotBugs skipped in fast mode, SpotBugs JavaBuildError on compile failure, Angular scanner uses angular-specific Semgrep config, timeout handling
- `router.test.ts` — integration tests for /scan /copilot /pr with supertest, covering Java and Angular requests

---

## SECTION 24 — BUILD ORDER FOR THE AGENT

Execute these steps in order. Complete each step fully before moving to the next:

1. `npm init` workspace root, create package.json
2. Create full directory structure including angular/ and java/ demo folders and spotbugs/ config
3. Implement `types.ts` — all interfaces including Language type with 'angular' and 'java'
4. Implement `logger.ts` — pino with redaction
5. Implement `errors.ts` — all error classes including JavaBuildError and UnsupportedLanguageError
6. Implement `languageDetector.ts` — auto-detect from file paths and build files
7. Implement `registry.ts` + create `remediator-registry.json` with all 9 entries
8. Implement `normalizer.ts` with RULE_TO_CWE_MAP + SPOTBUGS_TO_CWE_MAP
9. Implement `scanner.ts` — all scanners: Semgrep (multi-lang), Gitleaks, Bandit, pip-audit, npm audit, SpotBugs, OWASP Dependency-Check
10. Implement `remediator.ts` — language-aware prompt with Java import hints and Angular TS+HTML split
11. Implement `policy.ts` — evaluation + GitHub API
12. Implement `audit.ts` — SQLite with language column
13. Implement all middleware files
14. Implement all formatter files — Java uses java code fences, Angular splits TS+HTML
15. Implement `router.ts` — language detection integrated into all three route handlers
16. Implement `server.ts` — full Express app, /health shows supportedLanguages
17. Implement VS Code extension — DocumentFilter covers .java files
18. Implement CLI — --language flag + Java compile note in help text
19. Implement GitHub Action + workflow YAML with Java compile step and SpotBugs/OWASP install
20. Create all demo-app vulnerable files: Python, TypeScript, Angular, Java
21. Create all Semgrep YAML rules: sql-injection, hardcoded-secrets, weak-auth, xss, angular-security, java-security, java-xxe
22. Create spotbugs/secureguard-exclude.xml
23. Create docker-compose.yml + .env.example
24. Write all tests
25. Run `npm run build` — fix all TypeScript errors
26. Run `npm test` — fix all failing tests
27. Run `docker-compose up` — verify /health returns 200 with supportedLanguages array
28. Write README.md
29. Final check: run `secureguard scan demo-app/` across all four demo sub-folders and verify findings are detected for every language

---

## SECTION 25 — QUALITY GATES

Before declaring the build complete, verify every item:

### Functionality — Python and TypeScript (original)
- [ ] VS Code squiggle on save for `demo-app/python/routes/users.py` (SQL injection line 23)
- [ ] `git commit` on that file is blocked with CRITICAL message
- [ ] `@secureguard scan this file` returns streamed AI fix within 8 seconds
- [ ] PR gate posts inline suggestion and blocks merge

### Functionality — Angular (new)
- [ ] VS Code squiggle on save for `demo-app/angular/src/app/user-profile/user-profile.component.ts`
- [ ] Finding shows CWE-79 Angular XSS Bypass with CRITICAL severity
- [ ] Copilot Chat response includes TWO code blocks: TypeScript fix AND HTML template fix
- [ ] PR comment correctly labels Angular-specific fix with component and module changes
- [ ] `demo-app/angular/src/app/api/data.service.ts` triggers CWE-352 CSRF finding
- [ ] Registry lookup for CWE-352 returns Angular-specific HttpClientXsrfModule guidance

### Functionality — Java (new)
- [ ] `secureguard scan demo-app/java/ --mode fast` finds SQL injection via Semgrep (no compile needed)
- [ ] `secureguard scan demo-app/java/ --mode full` triggers Maven compile → SpotBugs → OWASP Dep-Check
- [ ] SpotBugs finds OBJECT_DESERIALIZATION in DataProcessor.java
- [ ] OWASP Dependency-Check finds Log4Shell CVE-2021-44228 in pom.xml
- [ ] Copilot Chat response for Java SQL injection includes PreparedStatement fix with import statement
- [ ] Copilot Chat response for XXE includes all three parser fix examples (SAX, DocumentBuilder, XMLInputFactory)
- [ ] JavaBuildError is returned clearly if mvn compile fails (not a generic 500)
- [ ] Java scan is skipped in IDE fast mode with hint "Run mvn compile for Java analysis"

### Language detection
- [ ] `languageDetector.detect(['UserController.java'], repoRoot)` returns `['java']`
- [ ] `languageDetector.detect(['user.component.ts'], repoRoot)` returns `['angular']`
- [ ] Mixed repo with both .java and .component.ts returns `['java', 'angular']`
- [ ] `.py` files return `['python']`

### Error handling
- [ ] JavaBuildError (mvn compile fails) returns HTTP 422 with clear message
- [ ] SpotBugs timeout returns partial results (Semgrep findings still returned)
- [ ] OWASP Dep-Check timeout returns partial results
- [ ] Claude API down returns structured JSON 503 error
- [ ] Rate limit returns 429 with Retry-After header

### Logging
- [ ] Every request has requestId in all log lines
- [ ] Language detected is logged at INFO on every request
- [ ] SpotBugs compile step logged as "Compiling Java project for SpotBugs analysis"
- [ ] No secrets in any log line
- [ ] Claude API latency and token count logged at INFO

### Security
- [ ] All endpoints require auth (except /health)
- [ ] No user file paths interpolated into subprocess commands (spawn with args array)
- [ ] Snippets truncated to 500 chars before sending to Claude
- [ ] .env gitignored

---

## SECTION 26 — HACKATHON DEMO SCRIPT

Four scenes, under 4 minutes total. Prep: `docker-compose up` in Terminal 1. VS Code open in Terminal 2.

**Scene 1 — IDE, Python (30 seconds):**
Open `demo-app/python/routes/users.py`. Save. Show CRITICAL red squiggle on line 23 (SQL Injection).

**Scene 2 — Copilot Chat, Angular (75 seconds):**
Open `demo-app/angular/src/app/user-profile/user-profile.component.ts`. Type: `@secureguard what's vulnerable here?`
Show streamed response: CRITICAL XSS badge → explanation of bypassSecurityTrustHtml risk → TypeScript fix → HTML template fix (two separate code blocks) → OWASP A03:2021 reference.

**Scene 3 — Java full scan (60 seconds):**
In terminal: `secureguard scan demo-app/java/ --mode full`
Show: "Compiling Java project..." → SpotBugs running → 4 findings: SQL Injection, XXE, Hardcoded Credentials, Insecure Deserialization → OWASP Dep-Check: Log4Shell CVE-2021-44228 CRITICAL.
Then: `@secureguard fix the XXE in XmlParser.java` → show streamed Java fix with all three parser patterns.

**Scene 4 — PR gate (60 seconds):**
Open pre-prepared PR containing `demo-app/java/src/main/java/com/demo/UserController.java`.
Show: GitHub Actions running → PR review comment with PreparedStatement suggestion (one-click apply) → red X status check blocking merge → apply suggestion → push → green check → merge enabled.

**Quantify:** Show stopwatch. Manual: Google CVE-2021-44228 + read OWASP XXE cheat sheet + write fix + write test = 25+ minutes. SecureGuard: 45 seconds.

---

## FINAL AGENT INSTRUCTION

You now have everything needed to build SecureGuard AI with full support for Python, TypeScript, JavaScript, Angular, and Java.

Begin immediately with Step 1 of Section 24. Work through all 29 steps in order. Do not skip any step. Do not stub any function.

Key things to get right for Java:
- SpotBugs requires compiled bytecode — implement the compile step clearly and fail-fast with JavaBuildError if Maven/Gradle is not available
- OWASP Dependency-Check is slow — set a 3-minute timeout and always return partial results if it times out
- SpotBugs output is XML — use xml2js to parse it, do not try to regex-parse XML
- Java is only fully scanned in FULL mode — fast mode uses Semgrep only

Key things to get right for Angular:
- Angular is TypeScript — use the same Semgrep TypeScript scanner with angular-specific rule configs added
- CSRF detection requires the angular-security.yaml custom rules — do not skip this file
- When generating AI fixes for Angular, ALWAYS check if the fix involves both a .ts component file AND the .html template — if so, output both sections labelled clearly
- The CWE-352 registry entry contains Angular-specific HttpClientXsrfModule guidance — use it

When you complete the build, run the full quality gate checklist in Section 25. Fix everything that fails. The demo in Section 26 must run successfully from start to finish.

Build it.

---
name: full-review
description: >
  Comprehensive code review combining security audit, code quality, performance,
  architecture, and bug detection. Use when asked to "review code", "audit",
  "find bugs", "security review", "code review", "check for issues", or any
  review/audit request. Supports quick triage, standard review, deep audit,
  and differential (PR/commit) review modes. Generates actionable reports.
allowed-tools:
  - Read
  - Write
  - Grep
  - Glob
  - Bash
  - Agent
---

# Full Code Review

Comprehensive review combining security, code quality, performance, architecture, and bug detection.

## Review Modes

| Mode | When to Use | Depth |
|------|------------|-------|
| **Quick Triage** | Small change, low risk, time-constrained | Checklists + surface scan |
| **Standard Review** | Normal PR or feature review | Full walkthrough + targeted deep dives |
| **Deep Audit** | Security-sensitive, critical system, pre-launch | Every file, adversarial analysis, blast radius |
| **Differential** | PR/commit review with git history context | Git blame, regression detection, blast radius |

**Auto-detect mode** based on scope:
- <100 lines changed → Quick Triage
- 100-500 lines → Standard Review
- 500+ lines or security-critical → Deep Audit
- User says "PR", "diff", "commit" → Differential

---

## Workflow

```
1. Scope & Context → 2. Attack Surface Map → 3. Review Passes → 4. Verify & Report
```

### Step 1: Scope & Context

Determine what to review and gather context:

```bash
# For branch/PR changes
git diff $(git merge-base HEAD main)..HEAD --stat
git diff $(git merge-base HEAD main)..HEAD --name-only

# For full codebase
find . -name "*.py" -o -name "*.ts" -o -name "*.tsx" -o -name "*.js" | head -50
```

**Classify each file by risk:**
- **HIGH**: Auth, crypto, payments, external calls, validation, data access, secrets
- **MEDIUM**: Business logic, state changes, new APIs, configuration
- **LOW**: Tests, docs, UI styling, logging, comments

### Step 2: Attack Surface Map

For each changed/reviewed file, identify:
- All user inputs (request params, headers, body, URL, file uploads)
- All database queries (raw SQL, ORM, query builders)
- All auth/authz checks (middleware, decorators, guards)
- All external calls (HTTP, gRPC, shell commands, file I/O)
- All cryptographic operations (hashing, signing, random generation)
- All state mutations (DB writes, cache updates, file writes)

### Step 3: Review Passes

Run these passes in order. For Quick Triage, do passes 1-3. For Standard, do 1-5. For Deep Audit, do all.

**Pass 1: Security** (see [security-patterns.md](references/security-patterns.md))
**Pass 2: Correctness & Bugs** (see [common-bugs.md](references/common-bugs.md))
**Pass 3: Code Quality**
**Pass 4: Performance** (see [performance-guide.md](references/performance-guide.md))
**Pass 5: Architecture** (see [architecture-guide.md](references/architecture-guide.md))
**Pass 6: Adversarial Analysis** (see [adversarial-analysis.md](references/adversarial-analysis.md))

### Step 4: Verify & Report

For each finding, verify it's real:
- Is there validation/sanitization elsewhere?
- Is there a test covering this scenario?
- Does the framework mitigate this?
- Is the input actually attacker-controlled?

Generate report (see [reporting.md](references/reporting.md)).

---

## Security Review

### Confidence Levels

| Level | Criteria | Action |
|-------|----------|--------|
| **HIGH** | Vulnerable pattern + attacker-controlled input confirmed | **Report** with severity |
| **MEDIUM** | Vulnerable pattern, input source unclear | **Note** as "Needs verification" |
| **LOW** | Theoretical, best practice, defense-in-depth | **Do not report** |

### Do Not Flag

- Test files (unless reviewing test security)
- Server-controlled values: `settings.*`, `os.environ`, config files, constants
- Framework-mitigated patterns (Django/React auto-escaping, ORM parameterization)
- Dead code, commented code, documentation strings

**Only flag framework-mitigated patterns when explicitly bypassed:**
- Django: `{{ var|safe }}`, `{% autoescape off %}`, `mark_safe(user_input)`
- React: `dangerouslySetInnerHTML={{__html: userInput}}`
- Vue: `v-html="userInput"`
- ORM: `.raw()`, `.extra()`, `RawSQL()` with string interpolation

### Always Flag (Critical)

```
eval(user_input)             exec(user_input)
pickle.loads(user_data)      yaml.load(user_data)       # not safe_load
unserialize($user_data)      shell=True + user_input
child_process.exec(user)     deserialize(user_data)     # Java ObjectInputStream
```

### Always Flag (High)

```
innerHTML = userInput                 dangerouslySetInnerHTML={user}
v-html="userInput"                    f"SELECT * FROM x WHERE {user}"
os.system(f"cmd {user_input}")        password = "hardcoded"
api_key = "sk-..."                    private_key = "-----BEGIN"
```

### Check Context First

```python
# SSRF — ONLY flag if URL from user input
requests.get(request.GET['url'])        # FLAG
requests.get(settings.API_URL)          # SAFE

# Path traversal — ONLY flag if path from user input
open(request.GET['file'])               # FLAG
open(settings.LOG_PATH)                 # SAFE

# Weak crypto — ONLY flag if used for security
hashlib.md5(file_content)               # SAFE (checksum)
hashlib.md5(password)                   # FLAG
random.random()                         # SAFE (non-security)
secrets.token_hex(16)                   # SAFE (security token)
```

### OWASP Top 10:2025 Quick Reference

| # | Vulnerability | Key Prevention |
|---|---------------|----------------|
| A01 | Broken Access Control | Deny by default, server-side enforcement, verify ownership |
| A02 | Security Misconfiguration | Harden configs, disable defaults, minimize features |
| A03 | Supply Chain Failures | Lock versions, verify integrity, audit deps |
| A04 | Cryptographic Failures | TLS 1.2+, AES-256-GCM, Argon2/bcrypt for passwords |
| A05 | Injection | Parameterized queries, input validation, safe APIs |
| A06 | Insecure Design | Threat model, rate limit, design security controls |
| A07 | Auth Failures | MFA, check breached passwords, secure sessions |
| A08 | Integrity Failures | Sign packages, SRI for CDN, safe serialization |
| A09 | Logging Failures | Log security events, structured format, alerting |
| A10 | Exception Handling | Fail-closed, hide internals, log with context |

---

## Correctness & Bug Detection

### Universal Checklist

- [ ] Off-by-one errors in loops and array access
- [ ] Incorrect boolean logic (De Morgan's law violations)
- [ ] Missing null/undefined/None checks
- [ ] Race conditions in concurrent code
- [ ] Integer overflow/underflow
- [ ] Floating point comparison issues
- [ ] Resource leaks (connections, file handles, listeners, timers)
- [ ] Swallowed exceptions (empty catch blocks)
- [ ] Missing error propagation
- [ ] Incorrect comparison operators (`==` vs `===`, `=` vs `==`)

### Security Checklist (Every File)

- [ ] **Injection**: SQL, command, template, header injection
- [ ] **XSS**: All outputs in templates properly escaped?
- [ ] **Authentication**: Auth checks on all protected operations?
- [ ] **Authorization/IDOR**: Access control verified, not just auth?
- [ ] **CSRF**: State-changing operations protected?
- [ ] **Race conditions**: TOCTOU in any read-then-write patterns?
- [ ] **Cryptography**: Secure random, proper algorithms, no secrets in logs?
- [ ] **Information disclosure**: Error messages, logs, timing attacks?
- [ ] **DoS**: Unbounded operations, missing rate limits, resource exhaustion?
- [ ] **Business logic**: Edge cases, state machine violations, numeric overflow?

---

## Code Quality Checklist

- [ ] Clear, descriptive variable/function names
- [ ] No code duplication (>10 lines repeated)
- [ ] Functions do one thing (SRP)
- [ ] No magic numbers/strings
- [ ] Complex logic has comments explaining *why*
- [ ] Public APIs documented
- [ ] Breaking changes noted
- [ ] No `TODO`/`FIXME` in production code without tracking issue
- [ ] No `console.log`/`print` debugging left in
- [ ] No commented-out code blocks

---

## Performance Quick Check

- [ ] No N+1 queries (use select_related/prefetch_related/eager loading)
- [ ] No `SELECT *` on large tables
- [ ] List endpoints have pagination with max limit
- [ ] No O(n^2) or worse nested loops on unbounded data
- [ ] Large lists use virtual scrolling (>100 items)
- [ ] Event listeners/timers cleaned up on component unmount
- [ ] No memory leaks (closures holding large objects, unclosed connections)
- [ ] Hot paths have caching where appropriate
- [ ] No blocking I/O on async event loops

---

## Differential Review (Git-Based)

When reviewing PRs, commits, or diffs:

### 1. Git History Analysis

```bash
# Find removed security checks
git diff <range> | grep "^-" | grep -E "require|assert|raise|throw|deny|reject"

# Find new external calls
git diff <range> | grep "^+" | grep -E "fetch|request|http|exec|system|spawn"

# Find changed access modifiers
git diff <range> | grep -E "public|private|protected|internal|external"

# Check if removed code was from security commits
git log -S "removed_pattern" --all --grep="security\|fix\|CVE\|vuln"
```

### 2. Red Flags (Stop and Investigate)

- Removed code from "security", "CVE", or "fix" commits
- Access control modifiers removed or relaxed
- Validation removed without replacement
- External calls added without input checks
- High blast radius (50+ callers) + HIGH risk change

### 3. Blast Radius

```bash
# Count callers for each modified function
grep -r "functionName(" --include="*.py" --include="*.ts" . | wc -l
```

| Change Risk | Blast Radius | Priority |
|-------------|--------------|----------|
| HIGH | 50+ callers | P0 — Deep + all deps |
| HIGH | 6-50 callers | P1 — Deep analysis |
| HIGH | 1-5 callers | P2 — Standard |
| MEDIUM | 20+ callers | P1 — Standard + callers |

### 4. Test Coverage Gaps

- NEW function + NO tests → Elevate risk
- MODIFIED validation + UNCHANGED tests → HIGH RISK
- Complex logic (>20 lines) + NO tests → Flag

---

## Severity Classification

| Severity | Impact | Examples |
|----------|--------|----------|
| **Critical** | Direct exploit, severe impact, no auth required | RCE, SQL injection to data, auth bypass, hardcoded secrets |
| **High** | Exploitable with conditions, significant impact | Stored XSS, SSRF to metadata, IDOR to sensitive data |
| **Medium** | Specific conditions required, moderate impact | Reflected XSS, CSRF on state-changing actions, path traversal |
| **Low** | Defense-in-depth, minimal direct impact | Missing headers, verbose errors, weak algorithms in non-critical context |

---

## Output Format

```markdown
## Code Review: [File/Component/PR Name]

### Summary
- **Scope**: X files reviewed, Y lines changed
- **Findings**: X (Y Critical, Z High, ...)
- **Risk Level**: Critical/High/Medium/Low
- **Recommendation**: APPROVE / REQUEST CHANGES / REJECT

### Findings

#### [SEVERITY] Title
- **Location**: `file.py:123`
- **Confidence**: High/Medium
- **Issue**: [What's wrong]
- **Impact**: [What could happen]
- **Evidence**:
  ```code
  [Vulnerable/problematic code]
  ```
- **Fix**:
  ```code
  [Corrected code]
  ```

### Needs Verification
- [Items requiring manual testing or context]

### Strengths
- [What was done well]

### Test Coverage
- [Gaps identified]
```

---

## Rationalizations (Do Not Skip)

| Rationalization | Why It's Wrong | Required Action |
|-----------------|----------------|-----------------|
| "Small change, quick review" | Heartbleed was 2 lines | Classify by RISK, not size |
| "I know this codebase" | Familiarity breeds blind spots | Build explicit context first |
| "Just a refactor" | Refactors break invariants | Analyze as HIGH until proven LOW |
| "No tests = not my problem" | Missing tests = elevated risk | Flag in report, elevate severity |
| "Framework handles it" | Verify the framework IS handling it | Check for bypasses |

---

## Quality Checklist (Before Delivering)

- [ ] All changed/reviewed files analyzed
- [ ] Git blame checked on removed security code (differential mode)
- [ ] Blast radius calculated for HIGH risk changes
- [ ] Attack scenarios are concrete, not generic
- [ ] Findings reference specific file:line locations
- [ ] Each finding has a clear fix recommendation
- [ ] Confidence level stated for each finding
- [ ] Report generated with all sections

---

## Supporting References

| Reference | Covers |
|-----------|--------|
| [security-patterns.md](references/security-patterns.md) | OWASP patterns, language-specific security, 20 languages |
| [common-bugs.md](references/common-bugs.md) | React, TypeScript, Python, SQL bugs and anti-patterns |
| [performance-guide.md](references/performance-guide.md) | Frontend (Core Web Vitals), backend, DB, algorithms |
| [architecture-guide.md](references/architecture-guide.md) | SOLID, anti-patterns, coupling, layered architecture |
| [adversarial-analysis.md](references/adversarial-analysis.md) | Attacker modeling, exploit scenarios, exploitability rating |
| [reporting.md](references/reporting.md) | Report templates, formatting, file naming |

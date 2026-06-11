# Security Review — LawIntake

**Date:** 2026-06-11
**Reviewer:** CEO Agent (Paperclip)
**Version:** 1.0 — Pre-build prospective review
**Scope:** Initial security audit of codebase, configuration, dependencies, and planned architecture

---

## Executive Summary

LawIntake is at project inception. The repository contains only a README; no application code, dependencies, or configuration exist yet. This review therefore serves as a **prospective security assessment** — establishing the security baseline, identifying domain-specific requirements, and flagging OWASP Top 10 concerns that must be addressed as the product is built.

**Key finding:** No active vulnerabilities exist in the current codebase (nothing to exploit). The critical window is *now* — security architecture decisions made in the first sprint are the hardest to undo later. Law firm data (client PII, privileged communications, case documents) demands defense-in-depth from day one.

### Severity Summary

| Severity | Count | Status |
|----------|-------|--------|
| CRITICAL | 0 | No code to exploit |
| HIGH | 5 | Prospective — must address before first user data |
| MEDIUM | 6 | Address before beta |
| LOW | 4 | Address before GA |
| INFO | 3 | Track ongoing |

---

## Scope and Methodology

### What Was Reviewed

| Artifact | Status | Notes |
|----------|--------|-------|
| Application code | None | Repository is empty beyond README |
| Dependencies | None | No package.json, requirements.txt, or equivalent |
| Configuration files | None | No .env, Dockerfile, CI config, or infra code |
| Secrets / credentials | None found | Scanned all commits — README only |
| GitHub repository settings | Reviewed | Private repo, no branch protection |
| Planned product architecture | Conceptual | Inferred from idea brief and domain |

### Methodology

- OWASP Top 10 (2021) framework
- STRIDE threat methodology (feeds into LAW-4 threat model)
- Manual inspection of all repository content
- Domain analysis for legal SaaS compliance requirements

---

## Findings

### Secrets & Credential Exposure

**Severity: INFO**

No secrets, API keys, tokens, or credentials found in any commit. The single commit (`de063ad`) contains only a README file.

**Recommendation:** Before writing any code, establish:
- `.gitignore` with entries for `.env`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `secrets/`
- A pre-commit hook (e.g., `detect-secrets` or `truffleHog`) to block accidental credential commits
- Environment variable management via a secrets manager (AWS Secrets Manager, HashiCorp Vault, or Doppler) rather than `.env` files in repos

---

### Repository Configuration

**Severity: LOW**

Branch protection is not enabled on `main`. As noted in LAW-3, this requires GitHub Pro for private repos. Without branch protection, anyone with write access can force-push or merge without review.

**Recommendation:**
- Upgrade to GitHub Pro or move to a plan that supports branch protection on private repos before any code is merged
- Require PR reviews (minimum 1 reviewer) and status checks before merge
- Disable direct push to `main`

---

## OWASP Top 10 Analysis (Prospective)

### A01:2021 — Broken Access Control

**Severity: HIGH**

**Why critical for LawIntake:** This is a multi-tenant SaaS. Each law firm must be completely isolated from other firms' data. A broken access control bug could expose one firm's privileged client communications to another firm — a catastrophic breach of attorney-client privilege with potential bar license consequences for the firm.

**Required controls:**
- Row-level security (RLS) on the database — every query must be scoped to `firm_id`
- Tenant isolation middleware that enforces firm context on every request
- Separate S3 buckets or prefixes per firm for document storage
- No "admin bypass" patterns in business logic code
- Automated tests that verify cross-tenant data cannot be accessed
- Role-based access: Firm Admin → Attorney → Paralegal → Client (read-only intake forms)

**Test requirement:** Before any deployment, include integration tests that log in as Firm A, attempt to access Firm B's resources, and assert `403 Forbidden`.

---

### A02:2021 — Cryptographic Failures

**Severity: HIGH**

**Why critical for LawIntake:** Client intake forms collect names, contact details, case descriptions, and potentially Social Security numbers, financial information, and medical records. Documents collected may include tax returns, medical reports, or identity documents. All of this data is protected by:
- Attorney-client privilege
- State data protection laws
- Potentially HIPAA (if personal injury or medical malpractice)
- CCPA / GDPR depending on client location

**Required controls:**
- TLS 1.2+ enforced everywhere; TLS 1.0/1.1 disabled
- Database encryption at rest (AES-256)
- Document storage encryption at rest (S3 SSE-KMS or equivalent)
- Per-firm KMS keys so key compromise is scoped to one firm
- Passwords hashed with bcrypt (cost ≥ 12) or Argon2id
- No MD5 or SHA-1 for security-sensitive operations
- Signed, time-limited URLs for document access (never expose raw S3 URLs)
- PII fields (SSN, DOB, financial data) field-level encrypted before database storage

---

### A03:2021 — Injection

**Severity: HIGH**

**Why critical for LawIntake:** Intake forms accept free-text input from clients (names, case descriptions, notes). Document upload filenames are user-controlled. These are the primary injection vectors.

**Required controls:**
- ORM with parameterized queries — no raw SQL string concatenation
- Input sanitization on all form fields before storage and display
- Filename sanitization on document uploads — strip path traversal characters (`../`, `..\\`), null bytes, shell metacharacters
- Content-type validation on uploads — don't trust the `Content-Type` header; verify file magic bytes
- Virus/malware scanning on uploaded documents before storage and before any processing
- Template injection prevention if using server-side templates
- Command injection prevention — never pass user input to shell commands

---

### A04:2021 — Insecure Design

**Severity: HIGH**

**Why critical for LawIntake:** Architectural security decisions (trust boundaries, authentication model, data flow) are being made now. The parallel [LAW-4](/LAW/issues/LAW-4) threat model addresses this systematically.

**Immediate design requirements:**
- Define trust boundaries before writing any API code: client (untrusted) → API gateway (validates auth) → services (trust internal) → database (trust internal only)
- Adopt a Secure SDLC: threat model before each feature, security review before each merge
- Design for least privilege: client portal users can only read their own case; attorneys can only read their firm's cases
- Plan for data minimization: collect only what is legally necessary for intake
- Design audit log as a first-class feature (who accessed what, when) — required for legal defensibility

---

### A05:2021 — Security Misconfiguration

**Severity: MEDIUM**

**Common failure modes for SaaS products:**
- Cloud storage buckets left public (S3, GCS)
- Debug mode enabled in production
- Default credentials on databases or admin panels
- Unnecessary services/ports open
- Error messages exposing stack traces to users

**Required controls:**
- Infrastructure-as-code (Terraform, Pulumi, CDK) from day one — no manual console configuration
- S3 buckets must have public access blocked at the account level
- Environment-specific configs: dev/staging/prod must be separate with least-privilege IAM roles
- Disable debug/verbose logging in production builds
- HTTP security headers: `Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`, `Content-Security-Policy`, `Referrer-Policy`
- Regular automated scanning of infra config (AWS Config rules, Prowler, or equivalent)

---

### A06:2021 — Vulnerable and Outdated Components

**Severity: MEDIUM**

**Current state:** No dependencies exist. This is the ideal moment to establish dependency hygiene.

**Required controls:**
- Pin dependency versions in package-lock.json / poetry.lock / go.sum
- Enable Dependabot or Renovate for automated dependency update PRs
- Enable GitHub Dependabot security alerts on the repository
- Run `npm audit` / `pip-audit` / `govulncheck` in CI on every PR
- Establish a policy: critical CVEs must be patched within 48 hours, high within 1 week
- Vet dependencies before adding: prefer well-maintained packages with active security disclosure programs

---

### A07:2021 — Identification and Authentication Failures

**Severity: HIGH**

**Why critical for LawIntake:** The product has two distinct user populations with different trust levels:
1. **Law firm staff** (attorneys, paralegals, admin) — privileged, need MFA
2. **Clients** — lower privilege, accessing only their own intake/case data, often not technically sophisticated

**Required controls:**
- MFA mandatory for all firm staff accounts (TOTP minimum; passkeys preferred)
- MFA optional but encouraged for clients
- Session tokens: short-lived JWTs (15 min) + refresh token rotation
- Refresh token rotation with detection of reuse (invalidate all sessions on reuse)
- Account lockout after N failed attempts (with CAPTCHA before lockout, not after)
- Password policy: minimum 12 characters, check against HaveIBeenPwned breach database
- Magic link / OAuth for client-facing portal (reduce password friction for one-time users)
- Secure password reset: time-limited tokens (15 min), single-use, delivered via email
- Logout must invalidate server-side session/token

---

### A08:2021 — Software and Data Integrity Failures

**Severity: MEDIUM**

**Concerns:**
- Document integrity: if a client uploads a signed contract, the uploaded file must be stored immutably and its integrity verifiable
- Supply chain: malicious packages in the build pipeline could compromise intake forms

**Required controls:**
- Pin CI/CD action versions to commit SHAs (not tags) to prevent tag-mutable supply chain attacks
- Content-addressable storage for documents (SHA-256 hash stored alongside document)
- Immutable document storage: once a document is submitted in an intake, it must not be modifiable
- Subresource Integrity (SRI) for any third-party CDN scripts
- Code signing for releases if distributing any desktop components

---

### A09:2021 — Security Logging and Monitoring Failures

**Severity: MEDIUM**

**Why critical for LawIntake:** Law firms may be required by state bar rules to maintain audit trails of who accessed client files. Audit logs are also essential for incident response and breach notification.

**Required controls:**
- Comprehensive audit log: every data access, modification, or deletion logged with `(user_id, firm_id, resource_type, resource_id, action, ip_address, timestamp, user_agent)`
- Audit logs must be append-only — no delete or update operations permitted
- Separate audit log storage from application database (different credentials)
- Log retention: minimum 7 years (common legal requirement)
- Real-time alerting on: multiple failed logins, impossible travel logins, bulk data exports, admin actions
- Structured logging (JSON) from day one — easier to query and alert on
- Centralized log aggregation (CloudWatch Logs, Datadog, etc.)

---

### A10:2021 — Server-Side Request Forgery (SSRF)

**Severity: MEDIUM**

**Relevance:** If LawIntake fetches documents from external URLs, processes webhooks from third-party integrations (DocuSign, payment processors), or allows import from external storage, SSRF is a real risk.

**Required controls:**
- Block requests to RFC 1918 addresses (10.x.x.x, 172.16.x.x, 192.168.x.x), localhost, and cloud metadata endpoints (169.254.169.254)
- Allowlist external domains for any URL-fetch features
- Use a dedicated egress proxy with strict allowlisting for external requests

---

## Domain-Specific Security Requirements

### Legal Data Protection

**Severity: HIGH**

Law firm client data carries special obligations beyond standard PII:

| Requirement | Obligation |
|-------------|------------|
| Attorney-client privilege | Data must not be disclosed to opposing parties, courts (without subpoena), or other firms |
| State bar rules | Many state bars have technology competence rules requiring appropriate security measures |
| Data residency | Some firms may require data stored only in the US |
| Retention & deletion | Defined retention periods; right to deletion on case close |
| Breach notification | State laws require notification within 30-72 hours of discovery |

**Recommendations:**
- Include a Data Processing Agreement (DPA) in every law firm contract
- Offer data residency selection (US regions only initially)
- Implement firm-level data retention policies with automated purge
- Document the incident response plan before first user

### HIPAA Considerations

**Severity: MEDIUM**

If law firms use LawIntake for personal injury, medical malpractice, or workers' compensation cases, uploaded documents may include Protected Health Information (PHI). If PHI passes through LawIntake's systems:
- A Business Associate Agreement (BAA) must be signed with each law firm customer
- HIPAA Security Rule technical safeguards apply
- Audit controls, access controls, transmission security, and integrity controls become mandatory

**Recommendation:** Before any personal injury or medical malpractice firm is onboarded, conduct a HIPAA gap analysis and sign BAAs.

### CCPA / GDPR

**Severity: LOW**

If any clients are California residents or EU/UK residents:
- CCPA: Right to know, delete, and opt-out applies to consumer data
- GDPR: Right to erasure, data portability, lawful basis for processing required

**Recommendation:** Build data export and deletion workflows into the product from the start — retrofitting is costly.

---

## Dependency CVE Scan

**No dependencies to scan.** Repository contains only a README.

Once the tech stack is selected, establish:
- Automated CVE scanning in CI (`npm audit --audit-level=high` or language equivalent)
- Block merges when critical CVEs are detected in direct dependencies
- Monthly review of transitive dependency vulnerabilities

---

## Configuration Issues

| Issue | Severity | Status |
|-------|----------|--------|
| No branch protection on `main` | LOW | Requires GitHub Pro — tracked in [LAW-3](/LAW/issues/LAW-3) |
| No `.gitignore` | LOW | Must be added before first code commit |
| No secret scanning | LOW | Enable GitHub secret scanning (free for private repos) |
| No Dependabot | INFO | Enable once first `package.json` / `requirements.txt` is committed |

**Immediate action — enable GitHub secret scanning:**
```
Repository Settings → Security → Secret scanning → Enable
```
This is free for private repos and will block accidental credential pushes.

---

## Security Design Principles (Before First Commit)

These decisions are cheapest to make now and most expensive to retrofit:

1. **Assume breach** — design so that a compromised API server cannot read another firm's database rows
2. **Encrypt everything** — field-level encryption for PII, envelope encryption for documents, TLS everywhere
3. **Least privilege** — IAM roles, database users, and API tokens with minimum necessary permissions
4. **Audit by default** — every data access is logged; logs cannot be deleted
5. **Defense in depth** — no single control failure should expose customer data
6. **Immutable infrastructure** — servers are replaced, not patched in place
7. **Zero trust networking** — internal services authenticate to each other, not just the perimeter
8. **Privacy by design** — collect only what is necessary; design deletion before collecting data

---

## Recommended Security Toolchain

| Layer | Tool | Priority |
|-------|------|----------|
| Secret detection | `detect-secrets` pre-commit hook | Before first commit |
| Dependency scanning | Dependabot + `npm audit` in CI | When first deps added |
| SAST | CodeQL (free for GitHub repos) | When first code added |
| Container scanning | Trivy or Snyk | When first Dockerfile added |
| Infrastructure | Prowler or Checkov | When first Terraform added |
| DAST | OWASP ZAP or Burp Suite | When first endpoint deployed |
| Penetration testing | External firm | Before launch |

---

## Action Items

### Before Writing Any Code (Priority 1)

- [ ] Add `.gitignore` with secrets exclusions
- [ ] Enable GitHub secret scanning on the repo
- [ ] Enable GitHub Dependabot alerts
- [ ] Document the intended tech stack so security toolchain can be finalized
- [ ] Add `detect-secrets` pre-commit hook to the repo

### Before First User Data (Priority 2 — HIGH severity items)

- [ ] Implement multi-tenant row-level security with firm isolation tests
- [ ] Configure TLS 1.2+ enforced, with HSTS headers
- [ ] Enable database and document storage encryption at rest
- [ ] Implement MFA for law firm staff accounts
- [ ] Establish audit log infrastructure (append-only, long retention)
- [ ] Conduct threat model (in progress: [LAW-4](/LAW/issues/LAW-4)) and address HIGH/CRITICAL items
- [ ] Enable branch protection on `main`

### Before Beta (Priority 3 — MEDIUM severity items)

- [ ] Full OWASP Top 10 penetration test against staging environment
- [ ] Implement document upload virus scanning
- [ ] HTTP security headers audit and remediation
- [ ] SSRF protection for any URL-fetch features
- [ ] Incident response plan documented and tested

### Before General Availability (Priority 4)

- [ ] External penetration test by qualified third party
- [ ] SOC 2 Type I preparation (if targeting enterprise firms)
- [ ] HIPAA gap analysis and BAA template ready
- [ ] Privacy policy and DPA legally reviewed
- [ ] Breach notification runbook tested

---

## Related Issues

- [LAW-4](/LAW/issues/LAW-4) — Conduct initial threat model (STRIDE methodology, in progress)
- [LAW-3](/LAW/issues/LAW-3) — Initialize GitHub repository (branch protection note)

---

*This document should be updated as the codebase grows. Re-run a full security audit before each major release milestone.*

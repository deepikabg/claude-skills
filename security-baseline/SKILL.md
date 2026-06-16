---
name: security-baseline
description: "Ensure minimum security practices before shipping any code. Use for any new service, API, or authentication flow. Checks: encryption, secrets management, PII handling, auth/authz, compliance scope, and dependency vulnerabilities."
---

# Security Baseline

The **design-time** security gate: it forces the security decisions that are nearly free to make on paper and expensive to retrofit — what's PII, how secrets are sourced, where the auth boundary sits, what compliance scope applies. It validates the *plan*, not the shipped code (that's `deploy-gate`'s job). The stance: security is a design input, not a pre-launch scramble. If a dimension can't be answered, the design isn't done.

## When to Trigger
Before implementation of:
- Authentication / authorization systems
- Any code handling PII
- APIs with external consumers
- Payment / financial flows
- Public-facing services

## When NOT to Trigger
- A local-only script with no PII, no network surface, no secrets
- A throwaway prototype on synthetic data (note that the security gate is deferred until it touches real data/users)
- A change wholly inside an already-baselined boundary that adds no new data, endpoint, or dependency

## Scale to the build
- **Auth / payments / PII / public services** → all five dimensions, plus Security/Legal review where required.
- **Internal tool, no sensitive data** → secrets management + dependency check; document why encryption/compliance are out of scope.
- **Prototype on synthetic data** → confirm no real secrets/PII present; defer the rest with a written reason.

## The Five Dimensions

### 1. Secrets Management
- ✓ No secrets in code
- ✓ Secrets sourced from [env vars / vault / managed service]
- ✓ Rotation strategy: [every N days]
- ✓ Audit logging: [who accessed this secret?]

### 2. Encryption
PII fields:
- [ ] Identified (what is PII in this system?)
- [ ] At rest: [AES-256 via X / managed service]
- [ ] In transit: [TLS + cert pinning where needed]
- [ ] Key management: [where are keys stored? rotation?]

### 3. Authentication / Authorization
- [ ] Auth model defined: [OAuth2 / API key / JWT / session]
- [ ] Scope boundaries: [what endpoints need auth?]
- [ ] Error handling: [don't leak whether a user exists]
- [ ] Rate limiting: [protect against brute force]

### 4. Compliance Scope
If applicable:
- [ ] GDPR: [PII handling, retention, deletion]
- [ ] SOX: [audit trails, access controls]
- [ ] HIPAA: [data classification, encryption]
- [ ] PCI-DSS: [card data handling]

### 5. Dependency Vulnerabilities
- [ ] All dependencies checked against the CVE database
- [ ] No critical CVEs
- [ ] Transitive dependencies audited
- [ ] Update strategy: [how often?]

## Guardrail Evals (Auto-Block)
Flag and require PM + Security review if:
- ✗ PII detected without an encryption spec
- ✗ Secrets in code/config
- ✗ Compliance scope stated but not addressed
- ✗ Critical CVE in dependencies
- ✗ Auth boundary undefined
- ✗ Error messages leak sensitive info

## Approval Workflow
Claude surfaces the proposal → PM (plus Security/Legal where required) reviews → approve / reject / revise.

## Handoff (End of the Design Chain → Build)
The security baseline is the final design gate. Once the PM (plus Security/Legal where required) approves:

**All specs are locked: MVP scope (brainstorm) → architecture (architecture-checkpoint) → [eval spec] → [API contract] → security baseline.**

Proceed to implementation. Tell the user the design chain is complete and what they're building against, e.g.:
> "Security baseline approved. The full design chain is locked — MVP scope, architecture, eval loops, API contract, and security are all signed off. Ready to build against these specs. Want me to start implementation?"

During the build, the remaining quality gates (test strategy, self code review) run as human-on-the-loop checks — Claude proposes, you spot-check — not as blocking design gates.

**Important — this is a DESIGN-time gate. It validated the plan, not the shipped code.** Before anything is deployed, migrated, or made public, the **deploy-gate** skill must run: it re-scans the actual built artifact for leaked secrets/PII/exposed endpoints, audits the installed dependency tree, and hard-stops for explicit human approval on irreversible actions (production deploy, schema migration) — even in bypass-permissions mode. Tell the user:
> "Design-time security is approved and we're clear to build. When the build is ready to ship, the Deploy Gate runs before any push to production — staging proceeds autonomously, but production deploy and DB migration will stop for your explicit approval."

## Anti-Patterns
- ❌ Security as a pre-launch checklist → ✅ security decided at design time, as an input
- ❌ "We'll figure out PII later" → ✅ PII fields identified and encryption specified now
- ❌ Compliance scope named but not addressed → ✅ each named regime has concrete handling
- ❌ Checking only direct dependencies → ✅ transitive tree audited too
- ❌ Auth bolted onto specific endpoints ad hoc → ✅ one defined auth model + explicit scope boundary
- ❌ Treating this as the last security step → ✅ deploy-gate re-checks the built artifact before it ships

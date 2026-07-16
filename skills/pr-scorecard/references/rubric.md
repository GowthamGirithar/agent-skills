# PR Scorecard Rubric

Calibration guide for the four axes. Match the PR's signals against these tables to pick **Low / Medium / High**. These are guides for judgment, not a checklist to tally — weigh the strongest signals, and when a PR straddles two levels, pick the higher one and name the tension in your rationale.

Polarity differs by axis — keep it straight when scoring:
- **Correctness** and **Alignment**: High = good (less scrutiny needed).
- **Impact** and **Security**: High = more scrutiny needed (that's where danger lives).

## Contents
- [Correctness](#correctness)
- [Impact](#impact)
- [Alignment](#alignment)
- [Security](#security)
- [Critical path reference](#critical-path-reference)

---

## Correctness

*How well-supported is the claim that this works as merged? High correctness means a reviewer can trust it with less scrutiny. It's a likelihood read from evidence (tests, CI, size), not a certification that the code is bug-free.*

| Level | Signals |
|-------|---------|
| **High** | Small, focused diff. Tests added/updated that exercise the change. Mechanical or generated changes (renames, formatting, dependency bumps with lockfile). CI green. Description explains what and why. Touches code with existing good coverage. |
| **Medium** | Moderate size or mixed changes. Some tests but gaps around the new behavior. Logic changes in reasonably understood code. CI passing but thin. Description present but partial. |
| **Low** | Large or sprawling diff. New logic with no tests, or tests that don't touch the changed behavior. CI red/absent. Copy-pasted blocks, commented-out code, `TODO`/`FIXME` left in. Clever/subtle code with no explanation. Manual-only verification of something testable. |

Correctness is about *how likely it works*, independent of how bad a bug would be — that's Impact.

**Always name test status in the rationale** — which tests were added/updated, which existing tests cover the change, or that none were found. Testing evidence is the strongest single signal here, and a triage tool that stays quiet about missing tests defeats one of its main purposes.

## Impact

*What is the blast radius if this change is wrong? Independent of how likely it is to be wrong. High = look harder.*

| Level | Signals |
|-------|---------|
| **High** | Touches a critical path (see below). DB schema migration, especially destructive/irreversible. Public API / shared contract / SDK change. Auth, authz, session, or payment logic. Concurrency, locking, transactions. Infra/config/IaC/pipeline changes. Feature-flag default flips. Wide dependent surface (many callers of a changed symbol). No clear rollback. |
| **Medium** | Business logic in a non-critical service. Changes to a shared internal module with a moderate number of callers. Backwards-compatible API additions. Config change that's easily reverted. |
| **Low** | Localized change with few dependents. Tests, docs, comments. Isolated UI copy/styling. Additive code behind an off-by-default flag. Dev-only tooling. |

To gauge dependent surface, search the codebase for callers of a changed public symbol — the count is your evidence. **Reversibility** is a first-class Impact signal: a destructive migration or a released public API scores high because it can't be cheaply undone, regardless of how many callers it has — call this out explicitly in the rationale.

## Alignment

*Does the PR do what it claims, and fit the codebase? High = well-aligned (good). Score the worst of the three sub-types — one bad sub-type drags alignment down.*

| Level | Signals |
|-------|---------|
| **High** | Diff matches the description and ticket. Follows the patterns of the code around it. Branch current or nearly so. |
| **Medium** | Minor undocumented extras beyond the ticket. Some deviation from surrounding conventions (naming, error handling, layering). Moderately behind base. |
| **Low** | Diff touches files/behavior with no mention in description or ticket (scope creep). Bundles unrelated changes ("while I was here…"). Directly contradicts the ticket's stated approach. Ignores an established pattern the rest of the module follows. Branch badly stale — likely conflicts or built on outdated assumptions. |

Three sub-types to check:
- **Scope alignment** — diff vs. PR description / linked Jira ticket. The most important for review triage.
- **Convention alignment** — new code vs. the project's stated conventions (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`, linter/formatter/style configs) first, then the unwritten style/patterns of surrounding and sibling files. A documented-rule violation is stronger evidence than a stylistic hunch.
- **Base alignment** — how far behind the base branch; conflict/rebase and stale-assumption risk.

## Security

*Does the change introduce a vulnerability or mishandle sensitive data?* Any Medium or higher must be flagged as requiring a dedicated security review before merge — recommend it to the reviewer; the scorecard is not a substitute for it.

| Level | Signals |
|-------|---------|
| **High** | Hardcoded secret/key/token/credential. Injection surface (SQL/command/template built from unsanitized input). Authn/authz check removed, weakened, or bypassable. Unsafe deserialization, SSRF, path traversal, unsafe eval. **PII/PCI written to logs, responses, or non-compliant storage** (customer, rider, partner, payment data). Crypto weakened or rolled by hand. |
| **Medium** | A **new** third-party dependency, or a **major/minor** version bump, or a bump to a dependency with a known relevant CVE — these actually expand or change the supply-chain surface. New external network call or endpoint. Input validation changes. Broadened permissions/scopes. Touches an auth-adjacent path without clearly changing the check. |
| **Low** | No security-relevant surface. Internal-only logic with no untrusted input, no secrets, no sensitive data. **Routine patch/point-release dependency bumps** (e.g. `v1.2.7 → v1.2.8`) with no known CVE — this is normal maintenance, not a security signal, and should not by itself push the score to Medium. |

Don't let dependency-bump noise inflate this axis: since Security ≥ Medium triggers a mandatory dedicated review, treating every `go.mod`/`package.json` patch bump as Medium would flag nearly every PR and defeat the purpose of triage. Reserve Medium for bumps that meaningfully change what code is trusted (new dependency, major/minor jump, or a version tied to a real advisory) — look up whether the new version addresses or introduces a CVE if that's not obvious from the bump alone.

**PII/PCI handling** — flag by `file:line`, never reproduce the sensitive values. Watch for: emails, phone numbers, addresses, names tied to identifiers, geolocation/rider tracks, card numbers, PANs, tokens. Logging or returning these is a High signal under GDPR/PCI and similar data-protection expectations.

## Critical path reference

Treat changes to these areas as elevated **Impact** (and often Security), even when the diff is small. Adapt to the actual repo — use these as prompts, and confirm against real paths seen in the diff:

- **Auth & identity** — login, session, token issuance/validation, permission checks.
- **Payments & checkout** — order totals, pricing, payment provider integration, refunds, vouchers.
- **Data migrations** — schema changes, backfills, destructive DDL.
- **Money & accounting** — payouts, fees, commission, invoicing.
- **Infra & deploy** — IaC, Kubernetes manifests, pipeline/CI config, secrets management.
- **Feature flags** — default-value flips and targeting changes that alter production behavior on merge.
- **Shared contracts** — public APIs, event schemas, SDKs, shared libraries with many consumers.

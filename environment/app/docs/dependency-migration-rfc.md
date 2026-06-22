# RFC-2024-047: MonoStack Dependency Security Migration

**Status:** FINAL — Implementation Required  
**Authors:** Platform Team (J. Hartwell, S. Okonkwo, T. Bergström)  
**Created:** 2024-01-08  
**Last Updated:** 2024-03-22  
**Reviewers:** Security Guild, Frontend Guild, DevOps  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Background and Motivation](#2-background-and-motivation)
3. [Scope](#3-scope)
4. [Initial Proposals (January 2024)](#4-initial-proposals-january-2024)
5. [Security Audit Findings](#5-security-audit-findings)
6. [Revised Proposals — Round 2 (February 2024)](#6-revised-proposals--round-2-february-2024)
7. [Peer Compatibility Analysis](#7-peer-compatibility-analysis)
8. [Rejected Proposals and Rationale](#8-rejected-proposals-and-rationale)
9. [Frontend Guild Review](#9-frontend-guild-review)
10. [DevOps Review](#10-devops-review)
11. [Round 3 Revisions (March 2024)](#11-round-3-revisions-march-2024)
12. [Final Decisions](#12-final-decisions)
13. [Implementation Notes](#13-implementation-notes)
14. [Appendix A: CVE Reference Table](#14-appendix-a-cve-reference-table)
15. [Appendix B: Superseded Proposals Log](#15-appendix-b-superseded-proposals-log)
16. [Appendix C: Peer Dependency Conflict Matrix](#16-appendix-c-peer-dependency-conflict-matrix)

---

## 1. Executive Summary

MonoStack has accumulated a set of transitive dependency vulnerabilities identified during our Q1 2024 security audit. This RFC documents the proposed resolution strategy, the deliberation process across three rounds of review, and the final authoritative override decisions to be applied to the workspace root `package.json`.

The overrides mechanism in npm workspaces allows us to enforce specific versions of transitive dependencies across all packages without requiring each package to independently pin them. This is the recommended approach for monorepos managing shared transitive exposure.

**This RFC went through three rounds of revision.** Earlier sections contain proposals that were subsequently superseded or rejected. The canonical final decisions are recorded in [Section 12](#12-final-decisions). Implementors should read Section 12 as the authoritative source and treat earlier version proposals as historical context only.

---

## 2. Background and Motivation

### 2.1 The Problem with Transitive Dependencies

MonoStack's four packages — `@monostack/core`, `@monostack/api`, `@monostack/ui`, and `@monostack/cli` — each declare their own direct dependencies. However, those direct dependencies pull in transitive dependencies at versions that have known CVEs or peer compatibility issues.

The standard approach of asking each package owner to upgrade their own deps has proven slow and inconsistent. The Platform Team received security advisory notices for five separate transitive packages between November 2023 and January 2024. Three attempts to coordinate cross-package upgrades during that period failed to reach full rollout before the next advisory arrived.

### 2.2 Why npm Overrides

npm workspaces introduced the `overrides` field in the root `package.json` to solve exactly this class of problem. When an override is specified, npm enforces that version across the entire dependency tree, regardless of what individual packages or their transitive dependencies request.

```json
{
  "overrides": {
    "some-package": "1.2.3"
  }
}
```

This approach:
- Requires only a single change in the root `package.json`
- Takes effect for all packages in the workspace on the next `npm install`
- Is explicit and reviewable in version control
- Does not require upstream packages to release new versions

### 2.3 Scope of This RFC

This RFC specifically covers the `overrides` block in `/app/package.json`. It does not cover:
- Direct dependency versions in individual package `package.json` files
- Dev dependency upgrades
- Node.js runtime version changes
- Any changes to CI configuration

### 2.4 Prior Art

In Q3 2022 we addressed a similar issue with `minimist` and `node-fetch` using manual `npm dedupe` passes. That approach worked for two packages but scaled poorly. This RFC establishes `overrides` as the standard going forward.

---

## 3. Scope

### 3.1 Packages in Scope

All four MonoStack workspace packages are in scope:

| Package | Path | Primary Language |
|---|---|---|
| `@monostack/core` | `packages/core` | JavaScript |
| `@monostack/api` | `packages/api` | JavaScript |
| `@monostack/ui` | `packages/ui` | JavaScript |
| `@monostack/cli` | `packages/cli` | JavaScript |

### 3.2 Dependencies Under Review

The following transitive dependencies were flagged for override consideration:

1. **lodash** — prototype pollution vulnerability
2. **semver** — regular expression denial of service (ReDoS)
3. **axios** — server-side request forgery (SSRF)
4. **express** — path traversal vulnerability
5. **react** — version fragmentation across packages causing double-render issues

### 3.3 Out of Scope

The following were considered but explicitly excluded:

- `webpack` — no CVE, upgrade would be a breaking change requiring extensive testing
- `babel` — frontend team requested deferral to Q3 2024
- `eslint` — dev dependency only, no production risk

---

## 4. Initial Proposals (January 2024)

> **NOTE:** This section contains the original Round 1 proposals from January 2024.
> Several of these were revised in subsequent rounds. Do not use these version numbers
> as the final implementation target. See Section 12 for final decisions.

### 4.1 lodash

**Background:** CVE-2021-23337 affects lodash versions prior to 4.17.21. The vulnerability allows prototype pollution via the `zipObjectDeep` function. Our `@monostack/core` package depends on `lodash@4.17.20`, which is one patch version behind the fix.

**Initial Proposal (Round 1):**
```
lodash: "4.17.21"
```

Rationale: The fix is a single patch version bump. No breaking changes. The vulnerability has a CVSS score of 7.2 (High). Immediate patching is recommended.

**Round 1 Status:** Accepted as proposed. No objections raised during initial review.

### 4.2 semver

**Background:** CVE-2022-25883 affects semver versions prior to 7.5.2. The vulnerability is a ReDoS (Regular Expression Denial of Service) in the `semver` package's version parsing logic. An attacker providing a crafted version string could cause unbounded CPU usage.

**Initial Proposal (Round 1):**
```
semver: "7.5.4"
```

Rationale: `7.5.4` was the latest stable release at the time of the initial proposal and includes the ReDoS fix. Semver follows strict semver itself so a minor version bump within the 7.x range is expected to be backward compatible.

**Round 1 Status:** Provisionally accepted, pending security team review of whether 7.5.4 is sufficient or if a higher minor version is required.

### 4.3 axios

**Background:** CVE-2023-45857 affects axios versions prior to 1.6.0. The vulnerability is a server-side request forgery (SSRF) issue related to improper handling of `XSRF-TOKEN` cookies that could expose credentials to unintended third-party origins.

**Initial Proposal (Round 1):**
```
axios: "1.4.0"
```

Rationale: The Platform Team initially proposed `1.4.0` as a conservative intermediate upgrade that would address most known issues while limiting the blast radius of the upgrade.

**Round 1 Status:** DISPUTED. Security team noted that `1.4.0` does NOT fix CVE-2023-45857. The fix was introduced in `1.6.0`. The Round 1 proposal was flagged as insufficient.

### 4.4 express

**Background:** CVE-2024-29041 affects express versions prior to 4.19.2. The vulnerability involves improper input sanitization in URL parsing, which can lead to path traversal under specific proxy configurations.

**Initial Proposal (Round 1):**
```
express: "4.18.3"
```

Rationale: `4.18.3` was an intermediate patch release the team was already planning to adopt.

**Round 1 Status:** REJECTED by Security team. CVE-2024-29041 is not fixed until `4.19.2`. See Section 8.1 for full rejection rationale.

### 4.5 react

**Background:** No CVE. The issue is version fragmentation: `@monostack/ui` uses `react@18.2.0` while newer packages added after the initial setup have been picking up `react@18.0.0` and `react@18.1.0` from their own transitive chains. This causes the "multiple React instances" problem where hooks throw invariant violations at runtime.

**Initial Proposal (Round 1):**
```
react: "18.2.0"
```

Rationale: Pin everything to the version already in `@monostack/ui`.

**Round 1 Status:** Accepted in principle. Frontend Guild requested review of whether a newer minor version should be targeted instead.

---

## 5. Security Audit Findings

The Security Guild conducted a full audit of the MonoStack dependency tree on 2024-01-29. The following findings were incorporated into the Round 2 revision process.

### 5.1 lodash — Confirmed

The audit confirmed the lodash finding from Section 4.1. `4.17.21` fully resolves CVE-2021-23337.

> *"The fix is minimal and well-understood. No compatibility risk. We recommend immediate adoption."*
> — Security Guild Audit Report, p. 4

### 5.2 semver — Escalated

The audit found that `7.5.4`, the Round 1 proposal, did not represent the latest stable release by January 2024. The security team noted:

> *"While 7.5.4 addresses the original ReDoS CVE, subsequent releases through 7.6.x have included additional hardening of the regex parser and improved handling of edge-case version strings. We recommend targeting the latest stable 7.6.x release rather than stopping at 7.5.4."*
> — Security Guild Audit Report, p. 7

The audit did not specify an exact 7.6.x version, leaving the selection to the Platform Team.

### 5.3 axios — Confirmed Insufficient

The audit confirmed the Security team's objection to the Round 1 proposal:

> *"CVE-2023-45857 is categorized CVSS 6.5 (Medium). The vulnerability was introduced in axios 0.21.1 and fully remediated in 1.6.0. Any override below 1.6.0 will leave MonoStack exposed. The `1.4.0` proposal in the current RFC draft does not meet our patching SLA for CVSS ≥ 5.0 vulnerabilities."*
> — Security Guild Audit Report, p. 9

### 5.4 express — Confirmed Insufficient

The audit confirmed the rejection of `4.18.3`:

> *"CVE-2024-29041 is a path traversal vulnerability introduced by improper handling of the Host header. It was fixed in 4.19.2. Intermediate versions 4.18.x and 4.19.0 / 4.19.1 are all vulnerable. The only acceptable target is 4.19.2 or higher."*
> — Security Guild Audit Report, p. 11

### 5.5 react — No CVE, Recommendation Revised

The audit did not flag a CVE for react. However, the Frontend Guild raised an additional concern during the audit review period:

> *"React 18.3.0 introduced a deprecation warning for legacy rendering patterns that several components in `@monostack/ui` use. We are already tracking the migration to the new API. Given that we need to override react anyway, we recommend targeting 18.3.1 (the first stable release after 18.3.0 that resolved a regression in the new warnings) to get ahead of the deprecation timeline."*
> — Frontend Guild comment on audit draft, 2024-02-05

---

## 6. Revised Proposals — Round 2 (February 2024)

> **NOTE:** This section documents Round 2 proposals from February 2024.
> Some of these were further revised in Round 3. See Section 12 for final decisions.

Based on the Security Audit findings, the Platform Team revised all five proposals:

### 6.1 lodash — Round 2

No change from Round 1. `4.17.21` confirmed as the target.

```
lodash: "4.17.21"
```

**Round 2 Status:** Confirmed.

### 6.2 semver — Round 2

The Security Guild's recommendation was to target the latest 7.6.x. At the time of Round 2 (February 2024), `7.6.0` was the latest release in the 7.6.x series.

```
semver: "7.6.0"
```

**Round 2 Status:** Provisionally accepted. The Platform Team noted this should be re-evaluated before final implementation in case a newer patch had been released.

### 6.3 axios — Round 2

Revised upward from the rejected `1.4.0` to address CVE-2023-45857:

```
axios: "1.6.0"
```

**Round 2 Status:** Accepted by Security team as meeting the minimum bar. DevOps requested a check for breaking changes between `1.3.4` (the current version in `@monostack/api`) and `1.6.0`.

### 6.4 express — Round 2

Revised from rejected `4.18.3`:

```
express: "4.19.2"
```

**Round 2 Status:** Accepted. This is the minimum version that resolves CVE-2024-29041. No further objections.

### 6.5 react — Round 2

Revised per Frontend Guild recommendation to include deprecation migration:

```
react: "18.3.1"
```

**Round 2 Status:** Accepted by Frontend Guild. DevOps noted this requires verifying that `react-dom` is also updated to match, but since we are using `overrides` the `react-dom` package will be handled separately only if fragmentation is detected there too.

---

## 7. Peer Compatibility Analysis

This section records the peer dependency compatibility checks performed before finalizing the Round 2 proposals.

### 7.1 lodash 4.17.21

Lodash does not declare peer dependencies. Compatible with all existing packages. No issues found.

### 7.2 semver 7.6.x

`semver` is widely used as an internal utility. The 7.x API is stable and backward compatible with 7.0.x. The change from `7.5.3` (current in `@monostack/core`) to `7.6.x` is non-breaking.

One transitive consumer — `@npmcli/package-json` — was found to peer-require `semver@^7.0.0`, which is satisfied by any 7.x version.

### 7.3 axios 1.6.x

The DevOps team's breaking change analysis found:

- **axios interceptors:** No breaking changes in the interceptor API between 1.3.x and 1.6.x.
- **CancelToken:** The `CancelToken` API was deprecated in 1.5.0 but not removed. Still functional in 1.6.x.
- **TypeScript types:** Improved in 1.6.x. No regressions.
- **`withCredentials` default:** No change.
- **SSRF patch:** The patch changes how `XSRF-TOKEN` is sent but only in cross-origin scenarios. MonoStack's API package uses axios for server-to-server calls only and does not use XSRF tokens.

**Conclusion:** The upgrade from `1.3.4` to `1.6.0` has no breaking impact on MonoStack's usage patterns.

### 7.4 express 4.19.2

Express follows semver. The `4.18.x` to `4.19.x` bump is a minor version increment indicating new features and potential deprecated API warnings, but no breaking changes.

The path traversal fix in `4.19.2` involves changes to the `res.redirect()` and router URL normalization logic. No MonoStack code paths use the affected patterns.

### 7.5 react 18.3.1

`react-dom` must match the `react` version. Both are included in the `@monostack/ui` dependencies at `18.2.0`. After applying the override:

- `react` will be forced to `18.3.1` workspace-wide
- `react-dom@18.2.0` in `@monostack/ui` will remain at `18.2.0` unless separately overridden

The Frontend Guild confirmed that `react@18.3.1` is compatible with `react-dom@18.2.0` for their current usage patterns, but noted this is a temporary state and a follow-up ticket has been filed to update `react-dom` in `@monostack/ui/package.json` directly.

---

## 8. Rejected Proposals and Rationale

### 8.1 express 4.18.3 — REJECTED

**Proposal:** `express: "4.18.3"`  
**Proposed by:** Platform Team (Round 1)  
**Rejected by:** Security Guild  
**Reason:** CVE-2024-29041 was introduced in express versions prior to `4.19.2`. Versions `4.18.3`, `4.19.0`, and `4.19.1` are all affected. The patch was first included in `4.19.2`. Any override below `4.19.2` leaves the vulnerability unaddressed.

### 8.2 axios 1.4.0 — REJECTED

**Proposal:** `axios: "1.4.0"`  
**Proposed by:** Platform Team (Round 1, conservative approach)  
**Rejected by:** Security Guild  
**Reason:** CVE-2023-45857 is not present in versions prior to `0.21.1` and was fixed in `1.6.0`. Version `1.4.0` is within the vulnerable range. The proposal was described in the audit as "insufficient to address the stated security concern."

### 8.3 semver 7.5.4 — SUPERSEDED (not rejected)

**Proposal:** `semver: "7.5.4"`  
**Proposed by:** Platform Team (Round 1)  
**Status:** Superseded by Round 2 proposal  
**Reason:** Not rejected for correctness — `7.5.4` does fix the original ReDoS CVE. However, the Security Guild recommended targeting the latest 7.6.x for additional hardening. The Round 2 proposal of `7.6.0` superseded this.

### 8.4 webpack and babel — OUT OF SCOPE

Both packages were raised in early discussions but explicitly removed from scope. See Section 3.3.

### 8.5 react 18.2.0 — SUPERSEDED (not rejected)

**Proposal:** `react: "18.2.0"`  
**Proposed by:** Platform Team (Round 1)  
**Status:** Superseded by Round 2 proposal  
**Reason:** Frontend Guild recommended `18.3.1` to align with deprecation migration timeline. The `18.2.0` proposal was not incorrect but was superseded by the revised recommendation.

---

## 9. Frontend Guild Review

The Frontend Guild reviewed the react-related decisions on 2024-02-12. Key discussion points:

### 9.1 react 18.3.x Deprecation Warnings

React 18.3.0 adds `console.error` deprecation warnings for:
- `ReactDOM.render()` (deprecated since React 18.0 but warning-free until 18.3)
- `ReactDOM.hydrate()` (same)
- `React.createFactory()` (removed in 19.0)
- String refs

`@monostack/ui` uses `ReactDOM.render()` in its test harness. The Frontend Guild's recommendation to adopt `18.3.1` was partly motivated by wanting visibility into these deprecation warnings before React 19 makes them breaking.

### 9.2 Why 18.3.1 Not 18.3.0

React 18.3.0 was released on 2024-04-22 and `18.3.1` was released on 2024-05-01 to fix a regression in the new deprecation warning output that caused crashes in certain SSR environments. MonoStack does not use SSR currently, but the guild recommended `18.3.1` as the safe choice.

### 9.3 Guild Decision

> *"The Frontend Guild approves the override of react to 18.3.1. We do not recommend 18.3.0 due to the regression. We do not recommend anything below 18.3.0 as it would undermine the deprecation visibility goal."*
> — Frontend Guild Review Notes, 2024-02-12

---

## 10. DevOps Review

The DevOps team reviewed the full proposal set on 2024-02-19. Their review focused on build pipeline impact and npm install reproducibility.

### 10.1 package-lock.json Considerations

The DevOps team noted that adding `overrides` will cause npm to restructure parts of `node_modules` on the next `npm install`. This will produce a diff in `package-lock.json`. The team flagged this as expected and not a concern, but asked that the PR that implements the overrides include a full regeneration of `package-lock.json` rather than an incremental update.

### 10.2 CI Build Time

The full `npm install` with overrides is expected to add approximately 15-30 seconds to CI build times due to the deduplication pass. This is acceptable.

### 10.3 Docker Layer Cache Invalidation

Any change to `package.json` will invalidate the Docker layer cache for the `npm install` step. The DevOps team requested the implementing engineer note this in the PR description so that the first pipeline run after the change is not mistaken for a build failure.

### 10.4 DevOps Approval

> *"DevOps approves the proposal with the notes above. No objections to the specific version selections."*
> — DevOps Review Sign-off, 2024-02-19

---

## 11. Round 3 Revisions (March 2024)

> **NOTE:** This section documents the final round of revisions from March 2024.
> The semver version was updated based on a newer release becoming available.
> All other packages were confirmed as-is from Round 2.

Between the Round 2 proposal (February) and the implementation deadline (March 22), the Security Guild requested a final check on semver to confirm whether a newer patch had been released.

### 11.1 semver — Final Check

The Platform Team checked the semver release history on 2024-03-15:

| Version | Release Date | Notable Changes |
|---|---|---|
| 7.6.0 | 2024-01-12 | Additional ReDoS hardening |
| 7.6.1 | 2024-02-08 | Fix regression in range comparator |
| 7.6.2 | 2024-03-01 | Fix edge case in `satisfies()` with pre-release tags |
| 7.6.3 | 2024-03-14 | Security: additional input length limit enforcement |

**Round 3 Decision:** Update semver target from `7.6.0` (Round 2) to `7.6.3` (latest at time of implementation). The Security Guild specifically endorsed `7.6.3` for the input length enforcement change.

> *"7.6.3 adds explicit input length limits that further reduce ReDoS attack surface. We recommend this over 7.6.0."*
> — Security Guild, Round 3 sign-off, 2024-03-15

```
semver: "7.6.3"
```

### 11.2 axios — Final Check

The Platform Team checked whether a newer patch was available. At the time of the Round 3 review, `axios` releases in the 1.6.x series included:

| Version | Release Date | Notable Changes |
|---|---|---|
| 1.6.0 | 2023-10-26 | CVE-2023-45857 fix |
| 1.6.1 | 2023-11-08 | Fix regression in response interceptors |
| 1.6.2 | 2023-11-21 | Fix FormData handling edge case |
| 1.6.3 | 2023-12-26 | Fix null-byte injection in URL params |
| 1.6.4 | 2024-01-25 | Dependency updates |
| 1.6.5 | 2024-02-20 | Fix for chunked transfer encoding |
| 1.6.6 | 2024-02-26 | Security: Patch for prototype pollution in mergeDeep |
| 1.6.7 | 2024-03-11 | Fix regression introduced in 1.6.6 |
| 1.6.8 | 2024-03-15 | Patch for SSRF in redirect handling |

The Security Guild reviewed the 1.6.x changelog and noted:

> *"1.6.6 addressed a prototype pollution issue and 1.6.8 patched an SSRF regression in redirect handling. We recommend targeting 1.6.8 as the final anchor for this RFC cycle. The Round 2 proposal of 1.6.0 remains functionally correct for the original CVE-2023-45857, but 1.6.8 is a strictly better target given subsequent patches."*
> — Security Guild, Round 3 sign-off, 2024-03-15

**Round 3 Decision:** Update axios target from `1.6.0` (Round 2) to `1.6.8`.

```
axios: "1.6.8"
```

### 11.3 All Other Packages — Confirmed

lodash, express, and react were confirmed as unchanged from their Round 2 versions:

- `lodash: "4.17.21"` — no change
- `express: "4.19.2"` — no change
- `react: "18.3.1"` — no change

---

## 12. Final Decisions

> **This section is the single authoritative source for implementation.**
> All earlier sections are historical record. Only the versions listed here
> should be applied to the workspace root `package.json` overrides block.

After three rounds of review by the Platform Team, Security Guild, Frontend Guild, and DevOps, the following overrides are approved for implementation:

### 12.1 Approved Overrides

```json
{
  "overrides": {
    "lodash": "4.17.21",
    "semver": "7.6.3",
    "axios": "1.6.8",
    "express": "4.19.2",
    "react": "18.3.1"
  }
}
```

### 12.2 Summary of Final Version Selections

| Package | Current Version | Override Target | Primary Reason | CVE / Issue |
|---|---|---|---|---|
| lodash | 4.17.20 | **4.17.21** | Prototype pollution fix | CVE-2021-23337 |
| semver | 7.5.3 | **7.6.3** | ReDoS hardening + input limits | CVE-2022-25883 |
| axios | 1.3.4 | **1.6.8** | SSRF fix + subsequent patches | CVE-2023-45857 |
| express | 4.18.2 | **4.19.2** | Path traversal fix | CVE-2024-29041 |
| react | 18.2.0 | **18.3.1** | Version standardization + deprecation visibility | N/A |

### 12.3 What Was NOT Included

The following were explicitly excluded from the overrides block:

- `react-dom` — managed directly in `@monostack/ui/package.json`; separate follow-up ticket filed
- `webpack` — deferred to Q3 2024
- `babel` — deferred to Q3 2024

### 12.4 Implementation Steps

1. Update `/app/package.json` to add the `overrides` block as specified in 12.1
2. Run `npm install` from `/app` to apply the overrides and regenerate `package-lock.json`
3. Verify clean install (no peer dependency warnings related to the overridden packages)
4. Submit PR with the updated `package.json` and regenerated `package-lock.json`

---

## 13. Implementation Notes

### 13.1 Order of Operations

The overrides block should be added to the root `package.json` **before** running `npm install`. Adding it after install will not take effect until install is re-run.

### 13.2 Expected Warnings

After applying the overrides, `npm install` may emit warnings along the lines of:

```
npm warn overriding lodash@4.17.20 with lodash@4.17.21
```

These warnings are expected and confirm the override mechanism is working. They are not errors.

### 13.3 CI Implications

The first CI run after this change will trigger a full Docker rebuild due to layer cache invalidation (see Section 10.3). This is expected. The build time will be longer than normal for that one run.

### 13.4 Rollback

If an issue is discovered after applying the overrides, the rollback procedure is:
1. Remove the `overrides` block from `package.json`
2. Run `npm install`
3. Revert the `package-lock.json` to the pre-override version

---

## 14. Appendix A: CVE Reference Table

| CVE | Package | CVSS Score | Affected Versions | Fixed In |
|---|---|---|---|---|
| CVE-2021-23337 | lodash | 7.2 (High) | < 4.17.21 | 4.17.21 |
| CVE-2022-25883 | semver | 5.3 (Medium) | 7.x < 7.5.2 | 7.5.2+ |
| CVE-2023-45857 | axios | 6.5 (Medium) | 0.21.1 – 1.5.x | 1.6.0+ |
| CVE-2024-29041 | express | 6.1 (Medium) | < 4.19.2 | 4.19.2 |

---

## 15. Appendix B: Superseded Proposals Log

| Section | Package | Superseded Version | Replaced By | Round |
|---|---|---|---|---|
| 4.3 | axios | 1.4.0 | 1.6.0 (Round 2) → 1.6.8 (Round 3) | 2, 3 |
| 4.4 | express | 4.18.3 | 4.19.2 | 2 |
| 4.2 | semver | 7.5.4 | 7.6.0 (Round 2) → 7.6.3 (Round 3) | 2, 3 |
| 4.5 | react | 18.2.0 | 18.3.1 | 2 |
| 6.2 | semver | 7.6.0 | 7.6.3 | 3 |
| 6.3 | axios | 1.6.0 | 1.6.8 | 3 |

---

## 16. Appendix C: Peer Dependency Conflict Matrix

This matrix documents peer dependency relationships between the overridden packages and the rest of the MonoStack dependency tree.

| Override | Peer Consumers in Tree | Peer Requirement | Satisfied by Override |
|---|---|---|---|
| lodash@4.17.21 | 14 packages | `^4.0.0` or `^4.17.0` | Yes |
| semver@7.6.3 | 8 packages | `^7.0.0` | Yes |
| axios@1.6.8 | 1 package (api) | `^1.0.0` | Yes |
| express@4.19.2 | 1 package (api) | `^4.0.0` | Yes |
| react@18.3.1 | 3 packages (ui, react-dom, testing-library) | `^18.0.0` | Yes |

All overrides satisfy the peer requirements of their consumers. No peer conflicts are introduced by these overrides.

---

---

## 17. Appendix D: Full CVE Technical Analysis

This appendix provides deep technical breakdowns of each CVE addressed by this RFC, including exploit mechanics, affected code paths, and remediation verification methodology.

### D.1 CVE-2021-23337 — lodash Prototype Pollution via `zipObjectDeep`

**Severity:** High (CVSS 7.2)
**CWE:** CWE-1321 (Improperly Controlled Modification of Object Prototype Attributes)
**Affected Versions:** < 4.17.21
**Fixed In:** 4.17.21

#### D.1.1 Exploit Mechanics

The vulnerability exists in lodash's `zipObjectDeep` function. This function is designed to create an object from arrays of paths and values, supporting nested path notation. The flaw occurs when a crafted path string containing `__proto__` or `constructor.prototype` is passed as a key.

Example exploit payload:
```javascript
const _ = require('lodash');

// Exploiting zipObjectDeep
_.zipObjectDeep(['__proto__.polluted'], [true]);
console.log({}.polluted); // true — global prototype has been mutated
```

The same vulnerability class also affects related functions:
- `_.set()` with `__proto__` path
- `_.setWith()` with `__proto__` path
- `_.merge()` with `__proto__` key in source object

However, CVE-2021-23337 specifically calls out `zipObjectDeep` as the primary attack vector.

#### D.1.2 Impact in MonoStack Context

`@monostack/core` uses lodash `4.17.20` for several utility functions:
- `_.groupBy()` — used in report aggregation (not vulnerable)
- `_.merge()` — used in config merging (vulnerable call path exists)
- `_.get()` and `_.set()` — used extensively for nested config access

The `_.merge()` usage in config merging is the relevant risk surface. If any configuration data originates from user-controlled input (e.g., API request bodies, CLI flags), a crafted payload could pollute `Object.prototype`, causing unexpected behavior across the entire Node.js process.

In `@monostack/api`, the Express route handlers accept JSON bodies that are passed to `_.merge()` in the middleware stack. This creates a viable attack chain:
```
POST /api/config
Content-Type: application/json
{"__proto__": {"isAdmin": true}}
```
If this body is passed through `_.merge()` without sanitization, every subsequently created object would have `isAdmin: true`.

#### D.1.3 Remediation Verification

The fix in `4.17.21` adds path segment validation in the `baseSet` internal function. Specifically, it rejects any path segment that equals `__proto__`, `prototype`, or `constructor` when the parent is a plain object or function.

To verify the fix:
```javascript
const _ = require('lodash@4.17.21');
_.zipObjectDeep(['__proto__.polluted'], [true]);
console.log({}.polluted); // undefined — prototype was not mutated
```

Security team verification script (run as part of the audit):
```bash
node -e "
const _ = require('lodash');
const before = Object.keys(Object.prototype).length;
_.zipObjectDeep(['__proto__.test'], [1]);
const after = Object.keys(Object.prototype).length;
process.exit(after > before ? 1 : 0);
"
```
Exit code 0 confirms the fix is active. Verified against `4.17.21` on 2024-01-29.

#### D.1.4 Why `4.17.21` and Not a Higher Version

As of January 2024, `4.17.21` is the latest release of lodash `4.x`. There is no `4.17.22` or later. The lodash maintainers have not published a new release since December 2021, so `4.17.21` is definitively the latest and the fix target.

---

### D.2 CVE-2022-25883 — semver ReDoS in Version Parsing

**Severity:** Medium (CVSS 5.3)
**CWE:** CWE-1333 (Inefficient Regular Expression Complexity)
**Affected Versions:** 7.x < 7.5.2 (also affects older major versions with different fix points)
**Fixed In:** 7.5.2 (initial), further hardened in 7.6.x

#### D.2.1 Exploit Mechanics

The `semver` package's version parsing relies on regular expressions. Prior to `7.5.2`, the regex used for parsing version strings did not limit input length, allowing a crafted input of repeating characters to cause catastrophic backtracking.

Vulnerable regex pattern (simplified):
```
/^(\d+)\.(\d+)\.(\d+)(?:-([0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*))?(?:\+([0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*))?$/
```

For very long prerelease identifiers like `1.2.3-` followed by 50,000 characters, the regex engine backtracks exponentially, causing the parsing to hang for seconds or minutes.

Proof of concept:
```javascript
const semver = require('semver');
const malicious = '1.2.3-' + 'a'.repeat(50000) + '!';
console.time('parse');
semver.valid(malicious); // hangs for many seconds in affected versions
console.timeEnd('parse');
```

On a modern CPU, this input can cause `semver.valid()` to take over 30 seconds, effectively blocking the Node.js event loop for that duration.

#### D.2.2 Impact in MonoStack Context

`@monostack/core` uses `semver@7.5.3` internally for version comparison in its package manifest processing logic. The relevant code path:
```javascript
// packages/core/src/manifest.js
const semver = require('semver');
function isCompatible(version, range) {
  return semver.satisfies(version, range);
}
```

If `version` or `range` is ever derived from external input (e.g., a plugin manifest uploaded by an end user), a malicious input could block the event loop. The Security Guild classified this as Medium severity because:
1. The input must be of significant length (tens of thousands of characters) to trigger catastrophic backtracking
2. The attack surface requires the caller to pass unvalidated external strings to `semver`
3. There is no evidence of direct external input flowing to `semver` in current code

However, as a widely depended-upon utility, the risk of this being exploited through a transitive dependency's usage is non-trivial.

#### D.2.3 Fix Evolution: 7.5.2 → 7.5.4 → 7.6.0 → 7.6.3

The fix for CVE-2022-25883 was introduced in `7.5.2` with input length limits on version strings (max 256 characters). However, subsequent releases continued hardening:

| Version | Fix Applied |
|---|---|
| 7.5.2 | Initial fix: input length limit of 256 characters on version strings |
| 7.5.3 | Additional length checks on range strings and prerelease identifiers |
| 7.5.4 | Regex engine hints to prevent backtracking on valid but complex inputs |
| 7.6.0 | Parser rewritten to use non-backtracking patterns for core version components |
| 7.6.1 | Fix regression in range comparator introduced in 7.6.0 |
| 7.6.2 | Edge case fix in `satisfies()` with pre-release version comparisons |
| 7.6.3 | Explicit input length enforcement at function entry points (belt-and-suspenders) |

The Security Guild's recommendation to target `7.6.3` reflects the cumulative hardening. Even though `7.5.2` technically "fixes" the original CVE, each subsequent release adds defense-in-depth.

#### D.2.4 Backward Compatibility Assessment

All changes from `7.5.3` to `7.6.3` are backward compatible within the semver 7.x API. The only breaking behavior change is that inputs exceeding the length limits now return `null` (for `semver.valid()`) or throw a `TypeError` (for `semver.coerce()`) rather than hanging. This is the desired behavior for any code path that previously would have hung indefinitely.

MonoStack's usage of `semver.satisfies()` and `semver.valid()` is all within version strings of normal length (< 50 characters). No behavioral changes are expected.

---

### D.3 CVE-2023-45857 — axios SSRF via XSRF-TOKEN Exposure

**Severity:** Medium (CVSS 6.5)
**CWE:** CWE-918 (Server-Side Request Forgery)
**Affected Versions:** 0.21.1 – 1.5.x
**Fixed In:** 1.6.0

#### D.3.1 Exploit Mechanics

The vulnerability is in how axios handles the `XSRF-TOKEN` cookie in cross-origin requests. Axios was designed to automatically include the value of the `XSRF-TOKEN` cookie in the `X-XSRF-TOKEN` request header for CSRF protection. However, a bug introduced in `0.21.1` caused this header to be sent on cross-origin requests as well as same-origin requests.

When a browser-based application using axios makes a cross-origin request, the `X-XSRF-TOKEN` header (containing the user's CSRF token) is sent to the third-party origin. If the application is configured to use cookies for XSRF protection (common in frameworks like Laravel, Angular, Django REST), this effectively leaks the user's CSRF token to untrusted origins.

Attack scenario:
1. A vulnerable application at `app.example.com` uses axios with XSRF cookie protection
2. User visits `app.example.com` which sets `XSRF-TOKEN` cookie
3. Attacker tricks user into visiting `evil.com` which includes a page that loads an image from `api.example.com`
4. axios, running in the `evil.com` context, sends `X-XSRF-TOKEN` with the token value to `api.example.com`
5. If the API accepts this token from cross-origin, the attacker can forge authenticated requests

Note: This is primarily a client-side (browser) vulnerability. Server-side uses of axios are not directly affected.

#### D.3.2 Impact in MonoStack Context

`@monostack/api` uses axios for server-to-server HTTP calls only. The package runs in a Node.js server environment, not in a browser. The `XSRF-TOKEN` vulnerability is strictly a browser-environment concern because:
1. Node.js does not have the concept of same-origin policy
2. No browser cookie jar is involved in server-to-server calls
3. The `X-XSRF-TOKEN` header is never set in the server-side axios usage

**Why override anyway?** Two reasons:
1. The 1.6.x series includes additional security patches unrelated to CVE-2023-45857 (prototype pollution fix in 1.6.6, SSRF in redirect handling in 1.6.8)
2. Standardizing on a modern version reduces audit surface and ensures future browser-facing usage of axios (if added) starts from a secure baseline

#### D.3.3 The Round 1 Failure: Why `1.4.0` Was Wrong

The Platform Team's initial proposal of `1.4.0` was based on a misreading of the CVE description. The team believed `1.4.0` was released after the CVE was reported and therefore included the fix. This was incorrect.

The timeline:
- CVE-2023-45857 was publicly reported in November 2023
- axios `1.6.0` (which fixes it) was released in October 2023 — before the CVE was publicly disclosed
- The fix was already in `1.6.0` for a different, related security reason
- `1.4.0` was released in April 2023, before the fix was developed

The Security Guild's audit clarified that the CVE number assignment date (November 2023) does not correspond to the fix date. The fix was already present in `1.6.0` released in October 2023.

#### D.3.4 Version Selection: From 1.6.0 to 1.6.8

| Version | Security Relevance |
|---|---|
| 1.6.0 | CVE-2023-45857 fix (XSRF-TOKEN cross-origin exposure) |
| 1.6.6 | Prototype pollution in `mergeDeep` utility function |
| 1.6.8 | SSRF vulnerability in redirect handling (new CVE) |

The Security Guild recommended `1.6.8` in Round 3 because:
- `1.6.6` fixed a prototype pollution issue that could be exploited if axios options are derived from untrusted input
- `1.6.8` fixed a new SSRF issue in redirect handling where a crafted 3xx response could redirect to an internal network address
- Both are relevant to MonoStack's server-side usage of axios

---

### D.4 CVE-2024-29041 — express Path Traversal via Host Header

**Severity:** Medium (CVSS 6.1)
**CWE:** CWE-22 (Improper Limitation of a Pathname to a Restricted Directory)
**Affected Versions:** < 4.19.2
**Fixed In:** 4.19.2

#### D.4.1 Exploit Mechanics

The vulnerability is in Express's URL parsing and redirect handling when the application is deployed behind a proxy. Specifically, `res.redirect()` does not adequately sanitize the redirect target when it is derived from user-controlled input (including the `Host` header) under certain proxy configurations.

When Express is configured with `trust proxy` and the proxy forwards a `Host` header, a crafted `Host` header value can cause `res.redirect()` to generate a redirect to an unintended URL, including paths that traverse directory structures or protocol-relative URLs that redirect to attacker-controlled domains.

Simplified example:
```javascript
// Vulnerable Express route
app.set('trust proxy', true);
app.get('/login', (req, res) => {
  const returnTo = req.query.return || '/dashboard';
  res.redirect(returnTo); // safe
});

// The vulnerability is in Express's internal handling of Host headers
// in redirect URL generation, not directly in application code
```

The full exploit requires a specific proxy configuration and the application to use `res.redirect()` with dynamic paths. The details of the exact payload are not included in this RFC to avoid providing attack instructions.

#### D.4.2 Impact in MonoStack Context

`@monostack/api` uses Express `4.18.2` and is deployed behind an AWS Application Load Balancer (ALB) with `trust proxy` enabled. The API includes redirect endpoints for authentication flows (OAuth callback redirects).

The Security Guild assessed the risk as Medium because:
1. The `trust proxy` setting is enabled (necessary condition for the exploit)
2. Redirect endpoints exist in the authentication flow
3. The `Host` header from the ALB is forwarded to Express

The fix in `4.19.2` adds stricter URL normalization in the router and in `res.redirect()` that prevents the Host header from being used in ways that could cause unintended redirects.

#### D.4.3 Why the Intermediate Versions Don't Fix It

`4.18.3`, `4.19.0`, and `4.19.1` were all considered as potential targets:

- `4.18.3` — Does not include the CVE-2024-29041 fix. Released before the fix was developed.
- `4.19.0` — Released 2024-03-25. Includes the initial attempt at a fix, but the fix was incomplete. Express maintainers subsequently discovered a bypass.
- `4.19.1` — Released 2024-03-26. Attempted to close the bypass from `4.19.0`, but was still found to be incomplete.
- `4.19.2` — Released 2024-03-25 (yes, same day as `4.19.0`, different commit). This is the version that the Express security team has confirmed fully resolves the issue.

The confusing release timeline (4.19.0 and 4.19.2 on the same day) is documented in the Express security advisory. The team effectively published `4.19.0`, immediately found a bypass, and published `4.19.2` before any users could widely upgrade. `4.19.1` was an intermediate attempt that is also incomplete.

The Security Guild's guidance is explicit: only `4.19.2` or later is acceptable.

---

### D.5 react Version Fragmentation — Not a CVE

**Issue Type:** Runtime stability / functional correctness
**Affected Versions:** Multiple React 18.x versions coexisting in the same npm workspace
**Fixed By:** Overriding to a single version via npm overrides

#### D.5.1 The Multiple Instances Problem

React relies on module-level singletons for its hook state management. When two different packages in a monorepo depend on different versions of React, npm may install both versions (e.g., `react@18.0.0` in `node_modules` and `react@18.2.0` in `node_modules/.package-a/node_modules`). When both instances are loaded by the same JavaScript process, React's internal invariant checks trigger an error:

```
Error: Invalid hook call. Hooks can only be called inside the function body of a function component.
```

This error occurs even when hooks are called correctly — the issue is that the hook is registered in one React instance but the component is rendered by another.

#### D.5.2 Version Inventory in MonoStack

Before the override, the MonoStack workspace had the following React version situation:

| Package | Direct React Dep | Resolved Version |
|---|---|---|
| `@monostack/ui` | `react@^18.2.0` | 18.2.0 |
| `@monostack/ui` (transitive via `react-beautiful-dnd`) | `react@^16.8.3 \|\| ^17 \|\| ^18` | 18.0.0 |
| `@monostack/cli` | none | — |
| `@monostack/api` | none | — |
| `@monostack/core` (transitive via `react-query@3.x`) | `react@^16.8.0 \|\| ^17.0.0 \|\| ^18.0.0` | 18.1.0 |

The transitive React pulls created three different installed versions:
- `node_modules/react` → `18.2.0`
- `node_modules/react-beautiful-dnd/node_modules/react` → `18.0.0`
- `node_modules/react-query/node_modules/react` → `18.1.0`

This created the double-render and hook invariant violation symptoms reported in the Q4 2023 bug tracker (ticket MONO-8841: "Drag-and-drop causes React hook invariant violation in production").

#### D.5.3 The 18.3.1 Choice

The Frontend Guild initially wanted to just standardize on `18.2.0`, the version already in use by the primary `@monostack/ui` package. However, `18.3.1` was chosen for two additional reasons:
1. It introduces deprecation warnings for APIs that React 19 will remove, giving the team visibility into required migration work
2. `18.3.0` had an SSR regression that was fixed in `18.3.1`

The override forces all three previously conflicting instances to resolve to `18.3.1`.

---

## 18. Appendix E: Stakeholder Communication Log

This appendix excerpts key communications from the RFC review process. Full threads are archived in Confluence under `Platform Engineering > RFC Archive > RFC-2024-047`.

### E.1 Initial Slack Thread — #platform-eng (2024-01-08)

**J. Hartwell** [9:02 AM]: Hey team, we have a security advisory situation to work through. The Q1 audit flagged five transitive deps with CVEs or stability issues. I'm going to write up an RFC this week. Pinging @S. Okonkwo and @T. Bergström to co-author.

**S. Okonkwo** [9:15 AM]: On it. I'll take the semver and lodash sections since I've been tracking those CVEs since Q4. @T. Bergström can you own the express and axios analysis?

**T. Bergström** [9:22 AM]: Sure. I looked at the axios one last week — heads up, the initial thought of 1.4.0 is probably wrong, CVE-2023-45857 wasn't fixed until 1.6.0. I'll put the full analysis in the RFC.

**J. Hartwell** [9:31 AM]: Good catch on axios. Let's make sure we're citing the actual CVE fix versions, not just "latest at the time we noticed." Security Guild will check.

**Security Guild Rep (A. Patel)** [10:47 AM]: Just saw this thread. We want to be in the review loop. Can you add a formal review gate before finalizing? We need to verify each proposed version actually contains the fix, not just a version that post-dates the CVE report.

**J. Hartwell** [10:52 AM]: Absolutely. Added as a formal review step. You'll get the draft RFC for review before we post Round 2 proposals.

---

### E.2 Email Thread: Security Guild Formal Review Request (2024-01-24)

**From:** J. Hartwell  
**To:** security-guild@monostack.internal  
**Subject:** RFC-2024-047 — Security Review Request  
**Date:** 2024-01-24

> Hi Security Guild,
>
> Attached is the current draft of RFC-2024-047 covering five transitive dependency overrides for the MonoStack workspace. We have Round 1 proposals for all five packages. Per our discussion in #platform-eng, we'd like your team to:
>
> 1. Confirm each proposed version actually contains the CVE fix
> 2. Recommend if a higher version should be targeted
> 3. Flag any packages we may have missed
>
> We're on a timeline to implement by end of March 2024. Please return your feedback by February 2.
>
> Packages under review:
> - lodash (CVE-2021-23337): proposing 4.17.21
> - semver (CVE-2022-25883): proposing 7.5.4
> - axios (CVE-2023-45857): proposing 1.4.0
> - express (CVE-2024-29041): proposing 4.18.3
> - react (version fragmentation, no CVE): proposing 18.2.0
>
> Thanks,  
> J. Hartwell, Platform Team

---

**From:** A. Patel (Security Guild Lead)  
**To:** J. Hartwell; security-guild@monostack.internal  
**Subject:** RE: RFC-2024-047 — Security Review Request  
**Date:** 2024-01-29

> J.,
>
> We've completed our review. Full report attached (see also Appendix D of this RFC). Summary of our findings:
>
> **lodash 4.17.21 — CONFIRMED CORRECT.** This is the exact fix version. No issues.
>
> **semver 7.5.4 — CONDITIONALLY APPROVED.** Fixes the original CVE. However, the 7.6.x series has additional hardening we recommend. We'll note this for re-evaluation before final implementation.
>
> **axios 1.4.0 — REJECTED.** This does not fix CVE-2023-45857. The fix was introduced in 1.6.0. Please revise to at minimum 1.6.0. We recommend checking the 1.6.x changelog for any subsequent security patches.
>
> **express 4.18.3 — REJECTED.** This does not fix CVE-2024-29041. Only 4.19.2 contains the verified fix. Versions 4.19.0 and 4.19.1 have an incomplete fix. Use only 4.19.2.
>
> **react 18.2.0 — NO SECURITY OBJECTION.** This is a stability/compatibility concern, not a CVE. We defer to Frontend Guild on version selection. We note that 18.3.x introduces deprecation warnings that may be relevant.
>
> Full audit report attached. Please revise your Round 2 proposals based on our findings.
>
> Regards,  
> A. Patel, Security Guild Lead

---

### E.3 Slack Thread — #frontend-guild (2024-02-05)

**M. Chen (Frontend Guild Lead)** [2:15 PM]: @team — Platform is asking us to review the react override version for RFC-2024-047. They're considering 18.2.0. What's our take?

**D. Kowalczyk** [2:28 PM]: 18.2.0 is fine from a compatibility standpoint but why stop there? We're eventually going to need to deal with the 18.3.x deprecation warnings before React 19 drops. If we're touching this now, we should target 18.3.1.

**R. Nakamura** [2:35 PM]: Agreed on 18.3.1. One note: 18.3.0 has a regression in SSR scenarios. Not relevant to us right now (we don't do SSR) but 18.3.1 is cleaner.

**M. Chen** [2:41 PM]: OK team consensus: recommend 18.3.1 to Platform. We also want to confirm that react-dom should NOT be in the overrides — we'll handle the react-dom version directly in @monostack/ui since it's a direct dep there.

**J. Hartwell (Platform)** [3:02 PM]: Got it. 18.3.1 for react, react-dom stays out of overrides, @monostack/ui team handles react-dom upgrade directly. Will update the RFC.

---

### E.4 Email Thread: DevOps Review (2024-02-16)

**From:** T. Bergström  
**To:** devops@monostack.internal  
**Subject:** RFC-2024-047 — DevOps Review Request  
**Date:** 2024-02-16

> Hi DevOps,
>
> We're finalizing Round 2 proposals for our transitive dependency override RFC. Before we proceed to implementation, we need your team's sign-off on:
>
> 1. Build pipeline impact (especially Docker layer cache invalidation)
> 2. npm install reproducibility with the overrides block
> 3. Any CI concerns
>
> The full RFC is linked in Confluence. The proposed overrides are:
> - lodash: 4.17.21
> - semver: 7.6.0 (updated per security recommendation)
> - axios: 1.6.0 (corrected from rejected 1.4.0)
> - express: 4.19.2 (corrected from rejected 4.18.3)
> - react: 18.3.1 (updated per frontend guild recommendation)
>
> Please confirm no objections and flag any concerns by February 19.
>
> Thanks,  
> T. Bergström

---

**From:** F. O'Sullivan (DevOps Lead)  
**To:** T. Bergström; devops@monostack.internal  
**Subject:** RE: RFC-2024-047 — DevOps Review Request  
**Date:** 2024-02-19

> T.,
>
> DevOps review complete. Approving with the following notes:
>
> **Cache invalidation:** Any change to package.json (including adding an `overrides` block) will bust the Docker layer cache at the npm install step. The first pipeline run after this change will be slow. Please note this in your PR description so the ops team doesn't mistake it for a build failure.
>
> **Lock file:** The PR must include a regenerated package-lock.json. Partial regeneration is not acceptable — run a clean `npm install` (or `npm ci` followed by `npm install` to regenerate) and commit the full updated lockfile.
>
> **Build time:** We estimate 15-30 additional seconds per pipeline run due to the deduplication pass npm performs when overrides are active. This is within acceptable bounds.
>
> **Rollback:** Document the rollback procedure. If we need to emergency-revert, the procedure should be in the PR description.
>
> No other concerns. Approved.
>
> F. O'Sullivan, DevOps Lead

---

### E.5 Slack Thread — Round 3 Security Check (2024-03-15)

**J. Hartwell** [10:05 AM]: We're at the implementation deadline for RFC-2024-047. Before we go, Security Guild asked for a final check on semver and axios to confirm we have the latest in the 7.6.x and 1.6.x series. @A. Patel confirming versions?

**A. Patel** [10:23 AM]: Checked release history. For semver: latest in 7.6.x as of today is 7.6.3 (released 2024-03-14). Has additional input length enforcement. Recommend using 7.6.3 over our Round 2 proposal of 7.6.0.

**A. Patel** [10:24 AM]: For axios: latest in 1.6.x is 1.6.8 (released 2024-03-15, literally yesterday). It patches an SSRF in redirect handling. We recommend targeting 1.6.8 over our Round 2 proposal of 1.6.0.

**J. Hartwell** [10:31 AM]: Updating RFC now. So final overrides: lodash 4.17.21 (unchanged), semver 7.6.3 (bumped from 7.6.0), axios 1.6.8 (bumped from 1.6.0), express 4.19.2 (unchanged), react 18.3.1 (unchanged).

**T. Bergström** [10:35 AM]: Confirmed, updating implementation ticket.

**S. Okonkwo** [10:38 AM]: Adding the Round 3 changes to Appendix B superseded proposals log. RFC will be final as of today.

---

## 19. Appendix F: Package Version Release Timelines

This appendix provides complete release timelines for each overridden package, covering the period relevant to this RFC's deliberation (Q4 2023 through Q1 2024).

### F.1 lodash Release History (relevant period)

| Version | Release Date | Notable Changes | Security Relevant |
|---|---|---|---|
| 4.17.19 | 2020-07-10 | Remove `array.sort` usage | No |
| 4.17.20 | 2020-08-14 | Fix `_.toNumber` edge case | No |
| 4.17.21 | 2021-02-20 | Fix prototype pollution in `zipObjectDeep` (CVE-2021-23337) | **YES** |

**Note:** No lodash 4.x releases have occurred since February 2021. The `4.17.21` release remains the latest and final 4.x release. There is ongoing community discussion about lodash 5.x (ESM-native rewrite) but no stable release as of March 2024.

**Historical context:** The lodash team has announced that `4.x` is in maintenance mode. Bug fixes and security patches will continue to be applied, but no new features. The team recommends migrating to native ES features (`.flatMap()`, `Object.fromEntries()`, etc.) where lodash is used for simple utilities.

For MonoStack, this means:
- `4.17.21` is the correct target and will remain so for the foreseeable future
- No future lodash `4.x` version will supersede `4.17.21` unless a new security issue is discovered
- The team should plan a migration away from lodash for new code, but existing usage on `4.17.21` is acceptable

---

### F.2 semver Release History (relevant period)

| Version | Release Date | Notable Changes | Security Relevant |
|---|---|---|---|
| 7.5.1 | 2023-04-03 | Performance improvements | No |
| 7.5.2 | 2023-06-13 | Initial ReDoS fix (CVE-2022-25883) | **YES** |
| 7.5.3 | 2023-06-21 | Additional length checks on range strings | **YES** |
| 7.5.4 | 2023-07-17 | Regex engine hints for backtracking prevention | **YES** |
| 7.6.0 | 2024-01-12 | Non-backtracking regex patterns for core version components | **YES** |
| 7.6.1 | 2024-02-08 | Fix regression in range comparator from 7.6.0 | Yes (regression fix) |
| 7.6.2 | 2024-03-01 | Fix edge case in `satisfies()` with pre-release tags | No |
| 7.6.3 | 2024-03-14 | Input length enforcement at function entry points | **YES** |

**Important distinction:** `7.5.3` was the version installed in `@monostack/core` at the time of the RFC (not `7.5.2`). The difference matters for CVE assessment: `7.5.3` does fix CVE-2022-25883, but the 7.6.x series provides stronger protection.

**Round 1 proposal of `7.5.4`** would have been an improvement but not optimal.  
**Round 2 proposal of `7.6.0`** addressed the Security Guild's recommendation to use 7.6.x hardening.  
**Round 3 decision of `7.6.3`** captures all subsequent security-relevant fixes in the 7.6.x series.

---

### F.3 axios Release History (relevant period)

| Version | Release Date | Notable Changes | Security Relevant |
|---|---|---|---|
| 1.3.4 | 2023-02-13 | Fix FormData circular reference | No |
| 1.4.0 | 2023-04-27 | Node.js stream improvements | No |
| 1.5.0 | 2023-08-25 | Deprecate CancelToken API | No |
| 1.5.1 | 2023-09-12 | Fix regression in response transformation | No |
| 1.6.0 | 2023-10-26 | **Fix CVE-2023-45857 (XSRF-TOKEN cross-origin exposure)** | **YES** |
| 1.6.1 | 2023-11-08 | Fix regression in response interceptors from 1.6.0 | Yes (regression fix) |
| 1.6.2 | 2023-11-21 | Fix FormData handling edge case | No |
| 1.6.3 | 2023-12-26 | Fix null-byte injection in URL params | **YES** |
| 1.6.4 | 2024-01-25 | Dependency updates (follow-me-ip removed) | No |
| 1.6.5 | 2024-02-20 | Fix for chunked transfer encoding on slow connections | No |
| 1.6.6 | 2024-02-26 | **Fix prototype pollution in `mergeDeep`** | **YES** |
| 1.6.7 | 2024-03-11 | Fix regression introduced in 1.6.6 (mergeDeep) | Yes (regression fix) |
| 1.6.8 | 2024-03-15 | **Fix SSRF vulnerability in redirect handling** | **YES** |

**Key observation:** Between Round 2 (proposing `1.6.0`) and Round 3 (finalizing `1.6.8`), three additional security-relevant fixes were released: `1.6.3` (null-byte injection), `1.6.6` (prototype pollution), and `1.6.8` (SSRF in redirects). This retroactively validates the Security Guild's process of doing a final check before implementation.

**Why `1.4.0` was the wrong answer (reprise):** `1.4.0` was released in April 2023. CVE-2023-45857 was fixed in `1.6.0` released in October 2023. The six-month gap means `1.4.0` was developed and released before the fix even existed. The Platform Team's initial assumption that `1.4.0` was "recent enough" was incorrect.

---

### F.4 express Release History (relevant period)

| Version | Release Date | Notable Changes | Security Relevant |
|---|---|---|---|
| 4.18.2 | 2022-10-08 | Fix `Content-Type` header edge case | No |
| 4.18.3 | 2024-03-05 | Various bug fixes | No |
| 4.19.0 | 2024-03-25 | Initial attempt to fix CVE-2024-29041 | **Incomplete fix** |
| 4.19.1 | 2024-03-26 | Attempt to close bypass of 4.19.0 fix | **Incomplete fix** |
| 4.19.2 | 2024-03-25 | **Full fix for CVE-2024-29041** | **YES — use this** |

**Note on `4.18.3`:** This release was published between the time the RFC Round 2 was written and the Round 3 implementation. When the Platform Team originally proposed `4.18.3` in Round 1, this version did not yet exist. `4.18.3` was published on 2024-03-05, after the Security Guild had already rejected the general `4.18.x` range as not containing the CVE fix. The Security Guild's guidance holds regardless: `4.18.3` does not fix CVE-2024-29041.

**The confusing 4.19.x timeline:** The Express maintainers published `4.19.0` and `4.19.2` on the same calendar day (2024-03-25), with `4.19.1` the following day. This rapid iteration reflects the maintainers discovering an incomplete fix immediately after publishing `4.19.0`. Do not use `4.19.0` or `4.19.1`.

---

### F.5 react Release History (relevant period)

| Version | Release Date | Notable Changes | Relevant to MonoStack |
|---|---|---|---|
| 18.0.0 | 2022-03-29 | React 18 stable release; concurrent features | Present as transitive dep |
| 18.1.0 | 2022-04-26 | Various bug fixes | Present as transitive dep |
| 18.2.0 | 2022-06-14 | Stable concurrent features; `useDeferredValue` improvements | **Direct dep in @monostack/ui** |
| 18.3.0 | 2024-04-22 | Deprecation warnings for React 19 migration | Recommended by Frontend Guild |
| 18.3.1 | 2024-05-01 | Fix SSR regression from 18.3.0 | **Final override target** |

**Note on release dates:** `18.3.0` and `18.3.1` were released after the Round 2 proposals but before the final implementation deadline. The Frontend Guild's recommendation of `18.3.1` was made in February 2024 based on anticipated release dates communicated by the React team on their blog. The actual releases aligned with this timeline.

**Why `18.2.0` was not used as the final target:** The Frontend Guild's recommendation to use `18.3.1` was motivated by the desire to surface React 19 deprecation warnings proactively. The team did not want to re-visit this override again in 6 months when React 19 is released. By targeting `18.3.1`, they accept some additional deprecation warnings now in exchange for not needing to file another RFC for this change.

---

## 20. Appendix G: Risk Assessment Matrix

This appendix provides a structured risk assessment for each override decision, covering the risk of action (applying the override) and the risk of inaction (keeping the current version).

### G.1 Risk Assessment Methodology

Each risk is scored on two dimensions:
- **Likelihood (L):** 1 (Very Low) to 5 (Very High)
- **Impact (I):** 1 (Minimal) to 5 (Critical)
- **Risk Score:** L × I (1–25)

| Score Range | Risk Level |
|---|---|
| 1–5 | Low |
| 6–10 | Medium |
| 11–15 | High |
| 16–25 | Critical |

---

### G.2 Risk of Inaction

| Package | Vulnerability | L | I | Score | Level |
|---|---|---|---|---|---|
| lodash | Prototype pollution via `zipObjectDeep` / `_.merge()` | 3 | 4 | **12** | High |
| semver | ReDoS via crafted version string | 2 | 3 | **6** | Medium |
| axios | SSRF/XSS via XSRF-TOKEN exposure | 2 | 3 | **6** | Medium |
| axios (1.6.6) | Prototype pollution in `mergeDeep` | 3 | 4 | **12** | High |
| axios (1.6.8) | SSRF in redirect handling | 3 | 4 | **12** | High |
| express | Path traversal via Host header | 3 | 4 | **12** | High |
| react | Hook invariant violations / runtime crashes | 4 | 3 | **12** | High |

**Total inaction risk exposure:** 4 High-rated risks, 2 Medium risks. Patching SLA requires remediating High-rated CVEs within 30 days.

---

### G.3 Risk of Action (Applying Overrides)

| Override | Failure Mode | L | I | Score | Level | Mitigation |
|---|---|---|---|---|---|---|
| lodash 4.17.21 | Incompatibility with existing usage | 1 | 2 | **2** | Low | Same major.minor.patch-1 → zero breaking changes |
| semver 7.6.3 | Minor behavior change in edge cases | 2 | 1 | **2** | Low | API fully backward compatible |
| axios 1.6.8 | Breaking change from 1.3.4 to 1.6.8 | 2 | 3 | **6** | Medium | DevOps compatibility analysis confirmed no breaking changes for MonoStack usage |
| express 4.19.2 | Breaking change from 4.18.2 | 1 | 2 | **2** | Low | Minor version bump; tested in staging |
| react 18.3.1 | Deprecation warnings surfaced | 3 | 1 | **3** | Low | Warnings are not errors; MONO-9102 filed to track migration |
| react 18.3.1 | Double-render during transition | 1 | 3 | **3** | Low | Override eliminates version fragmentation; reduces not increases risk |

**Total action risk exposure:** 1 Medium, 5 Low. All mitigations are in place or filed.

**Conclusion:** The risk of inaction (4 High risks) substantially exceeds the risk of action (1 Medium risk). The Platform Team's recommendation to proceed is risk-justified.

---

### G.4 Rollback Risk Assessment

If an override is found to cause a regression in production, the rollback procedure is:
1. Remove or revert the `overrides` block in `package.json`
2. Run `npm install` to regenerate `package-lock.json`
3. Deploy the reverted `package.json` and `package-lock.json`

Rollback time estimate: 15–30 minutes (deploy pipeline + npm install time).

**Rollback risk by package:**
- **lodash** — Very low. A rollback to `4.17.20` is a single patch version backward. No functionality change.
- **semver** — Very low. A rollback from `7.6.3` to `7.5.3` restores the previous behavior exactly.
- **axios** — Low. Rollback from `1.6.8` to `1.3.4` removes the CVE fix but restores the previously tested behavior. Accept only as emergency measure.
- **express** — Low. Rollback from `4.19.2` to `4.18.2` removes the CVE fix. Accept only as emergency measure.
- **react** — Medium. Rollback from `18.3.1` to fragmented versions restores the hook invariant violation symptoms. Not recommended.

---

## 21. Appendix H: Implementation Verification Checklist

Use this checklist to verify that the implementation of this RFC was completed correctly.

### H.1 Pre-Implementation Checks

- [ ] RFC-2024-047 has been read in full and Section 12 identified as the authoritative source
- [ ] Implementation will target exactly the five packages in Section 12.1
- [ ] The engineer has confirmed they will NOT use any versions from Sections 4, 6, or elsewhere in the RFC
- [ ] The engineer understands that `react-dom` is explicitly excluded from the overrides block (Section 12.3)
- [ ] The engineer has write access to the root `/app/package.json`

### H.2 Implementation Steps

- [ ] Open `/app/package.json`
- [ ] Verify no existing `overrides` field (if one exists, it must be replaced entirely)
- [ ] Add the `overrides` block as specified in Section 12.1:
  ```json
  "overrides": {
    "lodash": "4.17.21",
    "semver": "7.6.3",
    "axios": "1.6.8",
    "express": "4.19.2",
    "react": "18.3.1"
  }
  ```
- [ ] Verify no additional packages were added to `overrides` beyond the five approved packages
- [ ] Verify `react-dom` was NOT added to `overrides`
- [ ] Verify `webpack`, `babel`, and `eslint` were NOT added to `overrides`
- [ ] Run `npm install` from the `/app` directory
- [ ] Confirm no fatal errors during `npm install`
- [ ] Accept any `npm warn overriding X@Y with X@Z` messages as expected

### H.3 Post-Implementation Verification

- [ ] `package.json` contains the `overrides` block with exactly five entries
- [ ] `package-lock.json` has been regenerated (file mtime is newer than before `npm install`)
- [ ] `node_modules/lodash/package.json` shows version `4.17.21`
- [ ] `node_modules/semver/package.json` shows version `7.6.3`
- [ ] `node_modules/axios/package.json` shows version `1.6.8`
- [ ] `node_modules/express/package.json` shows version `4.19.2`
- [ ] `node_modules/react/package.json` shows version `18.3.1`
- [ ] `node_modules/react-dom/package.json` does NOT show `18.3.1` (it should remain at `18.2.0` from the direct dep in `@monostack/ui`)
- [ ] No nested `node_modules/X/node_modules/lodash` directory exists (override is effective)

### H.4 Common Mistakes

**Mistake 1: Using versions from Section 4 (Round 1)**
The Round 1 proposals are historical context. They are NOT the implementation targets. Specifically:
- `axios: "1.4.0"` ← WRONG (was explicitly rejected)
- `express: "4.18.3"` ← WRONG (was explicitly rejected)
- `semver: "7.5.4"` ← WRONG (was superseded by Round 3)
- `react: "18.2.0"` ← WRONG (was superseded by Round 3)

**Mistake 2: Using versions from Section 6 (Round 2)**
Round 2 proposals are also not the final answer. Specifically:
- `semver: "7.6.0"` ← WRONG (superseded by Round 3 decision of 7.6.3)
- `axios: "1.6.0"` ← WRONG (superseded by Round 3 decision of 1.6.8)

**Mistake 3: Including react-dom in overrides**
Section 12.3 explicitly states that `react-dom` must NOT be in the overrides block. The Frontend Guild is handling `react-dom` separately via a direct dependency update in `@monostack/ui`.

**Mistake 4: Adding extra packages**
Only the five packages approved in Section 12.1 should be in the overrides block. Do not add `webpack`, `babel`, `eslint`, or any other package mentioned elsewhere in the RFC.

**Mistake 5: Not running `npm install` after updating `package.json`**
The overrides only take effect after `npm install` is run. A `package.json` with an `overrides` block but without a corresponding `npm install` is incomplete.

---

## 22. Appendix I: Frequently Asked Questions

### Q: Why not just upgrade the direct dependencies instead of using overrides?

**A:** Overrides are appropriate when the dependency declaring the vulnerable version is a transitive dependency (i.e., you do not directly depend on it). For example, `@monostack/api` does not directly list `lodash` as a dependency — it arrives as a transitive dependency through other packages. To upgrade `lodash` by upgrading the direct dependency would require identifying and updating every package that transitively pulls in `lodash`, which may not be possible if those packages haven't released a version that uses `lodash@4.17.21`.

The `overrides` mechanism in npm workspaces is specifically designed for this scenario: forcing a workspace-wide version of a transitive dependency without requiring each consuming package to independently update.

### Q: Will overrides break semantic versioning guarantees?

**A:** Potentially, yes. If a package declares a peer dependency on `lodash@^4.17.0` and we override to `4.17.21`, the override is within the declared range, so no semantic versioning guarantees are violated. However, if we overrode to `lodash@5.0.0` when the peer requirement is `^4.17.0`, we would be violating the declared semver contract.

All overrides in this RFC are within the declared peer requirement ranges (see Appendix C for the peer compatibility matrix). No semantic versioning violations are introduced.

### Q: What happens to `react-dom` if we override `react` to `18.3.1`?

**A:** `react-dom` is a separate package. Overriding `react` does not automatically override `react-dom`. In MonoStack, `@monostack/ui` declares `react-dom@^18.2.0` as a direct dependency. After applying the `react` override, `node_modules/react` will be `18.3.1` but `node_modules/react-dom` will remain at `18.2.0`.

The Frontend Guild has confirmed that `react@18.3.1` is compatible with `react-dom@18.2.0` for the usage patterns in `@monostack/ui`. A follow-up ticket (MONO-9047) has been filed to upgrade `react-dom` directly in `@monostack/ui/package.json`.

### Q: Do overrides apply to devDependencies?

**A:** Yes. npm overrides apply to all dependency types — dependencies, devDependencies, peerDependencies, and optionalDependencies alike. If a dev tool transitively uses `lodash`, it will also receive the overridden version.

### Q: What if a future `npm update` or `npm audit fix` changes the overrides?

**A:** `npm update` does not modify the `overrides` block — overrides are explicit configuration, not automatically managed. `npm audit fix` also does not modify overrides; it upgrades direct dependencies. The `overrides` block is persistent until manually changed.

### Q: Should we add overrides for packages we haven't identified as vulnerable yet?

**A:** No. Overrides should only be added for packages with a specific, documented reason (CVE, stability issue, or version fragmentation). Pre-emptively overriding packages that aren't flagged creates unnecessary maintenance overhead and potential compatibility risks with no corresponding security benefit.

### Q: How do we know when to update or remove an override?

**A:** An override should be removed when either:
1. The consuming direct dependencies have been updated to declare the secure version themselves (making the override redundant), or
2. The overridden package is no longer used by any package in the workspace

An override should be updated when a new security advisory affects the current override target version.

The Platform Team should review all overrides quarterly as part of the dependency audit process.

---

*End of RFC-2024-047 main body. Appendices E through I follow.*

*For questions, contact the Platform Team in `#platform-eng` or open a ticket in the MonoStack project.*

---

## 18. Appendix E: Security Guild Slack Thread Archive

This appendix preserves the full Slack discussion history from the `#security-guild` channel relevant to this RFC. Thread excerpts are presented in chronological order. Usernames have been anonymized to first-name-only format per the team's archival policy.

---

### E.1 Thread: "lodash CVE triage — urgent" (2024-01-08)

**Marcus** [09:02]: Morning all. Dependabot opened 14 PRs overnight on the monostack repo flagging lodash < 4.17.21. CVE-2021-23337. I know we've been aware of this one for a while but legal is now pushing for remediation SLAs. Tagging @Priya and @Reuben since you both have lodash consumers in your packages.

**Priya** [09:14]: Yeah, core uses lodash heavily. The `_.merge()` calls in config loading are the main risk surface I'm worried about. We don't directly accept user input into those merge paths today but that could change. I'd vote for just pinning to 4.17.21 across the board via overrides.

**Reuben** [09:21]: api also uses it for some response shaping utilities. Nothing that touches user data directly but I agree, let's just pin it. Is there a 5.x we should be migrating to instead?

**Marcus** [09:28]: lodash 5 is still in alpha and has been for two years. 4.17.21 is the latest stable and the maintainers have said 4.x is in maintenance-only mode. No new features, only critical security fixes. So 4.17.21 is the right target — there's nothing newer in the 4.x line.

**Priya** [09:35]: Confirmed. I checked npm today. Latest is 4.17.21, published December 2021. No subsequent releases. So there's no "upgrade" question here — 4.17.21 is both the fix version and the latest version simultaneously.

**Jordan** [09:41]: Can someone summarize the actual exploit for the CVE? I want to make sure I understand what we're protecting against before I sign off on the remediation priority.

**Marcus** [09:55]: Sure. The vulnerability is in `_.zipObjectDeep`. When you pass a path string like `__proto__.isAdmin` as a key, lodash will walk the prototype chain and mutate `Object.prototype` directly. This means every plain object in the process subsequently has `isAdmin: true` as a property. The fix in 4.17.21 validates path segments against a denylist (`__proto__`, `prototype`, `constructor`) before traversal.

**Jordan** [10:03]: Got it. So the direct attack vector requires an attacker to control the keys passed to zipObjectDeep or related functions. In our current codebase, is that actually reachable from user input?

**Priya** [10:11]: In `@monostack/api`, there's a middleware that calls `_.merge()` on the incoming request body for some config update endpoints. Merge has the same vulnerability class. Technically a crafted request body with `{"__proto__": {"isAdmin": true}}` could trigger it. Our input validation middleware might catch it but I wouldn't bet on it.

**Reuben** [10:18]: I just checked — yes, our JSON body parser runs before our input validation middleware. So there is a window. The lodash 4.17.21 fix would close that window at the library level regardless of our middleware ordering.

**Jordan** [10:24]: That's enough for me. P1. Let's pin it. Marcus, can you track this through the RFC?

**Marcus** [10:26]: On it. I'll open the RFC draft today.

---

### E.2 Thread: "semver ReDoS — how severe is this actually?" (2024-01-10)

**Keiko** [14:15]: Picking up the semver CVE-2022-25883 discussion from standup. I looked at the actual regex and I want to share my findings before we finalize the RFC version target.

**Marcus** [14:22]: Please do. There's been some disagreement in the thread about which fix version to use.

**Keiko** [14:31]: So the vulnerability is in the version parsing regex. The original regex for parsing pre-release identifiers didn't have a length check, which means a string like `1.2.3-` followed by 50,000 `a` characters causes catastrophic backtracking. The initial fix in 7.5.2 added a length guard. However, the guard in 7.5.2 was set to 256 characters. Security researchers subsequently showed that payloads up to 256 characters could still cause measurable delays on slower hardware. 7.6.x tightened this further.

**Tobias** [14:38]: Wait, so 7.5.2 doesn't fully fix it?

**Keiko** [14:45]: It fixes the severe case. A 50k-character payload goes from blocking for 30+ seconds to completing instantly. But a carefully crafted ~200-character payload can still cause ~100ms delays. In a high-throughput API that calls semver in a hot path, that's potentially exploitable for DoS. 7.6.x reduces the limit further and also changes the regex engine approach to avoid backtracking entirely for the pre-release segment.

**Tobias** [14:52]: Which version of 7.6 should we target? I see 7.6.0, 7.6.1, 7.6.2, 7.6.3 on npm.

**Keiko** [15:01]: 7.6.3 is the latest. 7.6.0 introduced the regex rewrite but had a regression where valid pre-release versions with numeric-only identifiers were incorrectly rejected. 7.6.1 fixed that. 7.6.2 was a documentation update. 7.6.3 is a minor correctness fix for edge cases in the comparator logic. Bottom line: 7.6.3 is the version you want.

**Marcus** [15:08]: So to be clear — 7.5.4 is NOT sufficient?

**Keiko** [15:14]: 7.5.4 is better than 7.5.2 (it tightened the length check) but still uses the backtracking-prone regex approach. 7.6.x replaces the regex entirely. For a security override, we should go straight to 7.6.3.

**Tobias** [15:20]: I'll admit I was the one pushing for 7.5.4 in the early rounds of the RFC because I was worried about breaking changes in 7.6.x. But based on what Keiko just shared, I'm changing my vote to 7.6.3. The remaining exposure in 7.5.4 is real.

**Keiko** [15:26]: There are no breaking changes in semver's public API between 7.5.x and 7.6.x. I checked the changelog. The only thing that changed is the internal regex and the handling of edge cases in pre-release version parsing. Standard usages like `semver.satisfies()`, `semver.gt()`, etc. are all unaffected.

**Marcus** [15:31]: Consensus is 7.6.3. I'll update the RFC. Closing this sub-thread.

---

### E.3 Thread: "axios version — 1.4.0 vs 1.6.8 debate" (2024-01-11)

**Danielle** [11:05]: I want to reopen the axios version discussion. The RFC currently targets 1.4.0 and I don't think that's right.

**Marcus** [11:12]: 1.4.0 was proposed because it was the minimum version that fixed the SSRF vulnerability we were tracking. What's your concern?

**Danielle** [11:19]: The CVE in question is CVE-2023-45857. Let me be specific about the timeline. The vulnerability was introduced in axios 1.0.0 when XSRF token handling was changed. The token was being sent to third-party origins by default. 1.4.0 does NOT fix this. I checked the changelog and the GitHub issues. The actual fix was backported to 0.x and introduced properly in 1.6.0.

**Marcus** [11:27]: Wait, seriously? I thought 1.4.0 was the fix version.

**Danielle** [11:34]: Common misconception. NVD initially listed the affected range as < 1.4.0 because that's when some other hardening was done, but the specific XSRF issue persisted into 1.5.x. The fix PR that specifically addresses CVE-2023-45857 was merged into the main branch and released as part of 1.6.0. I can link the PR if you want.

**Marcus** [11:38]: Please link it.

**Danielle** [11:39]: https://github.com/axios/axios/pull/6028 — "fix: prevent XSRF token from being sent to cross-origin requests". Merged 2023-10-04. Released in 1.6.0.

**Reuben** [11:45]: I can confirm this. I tested it. With axios 1.4.0, if you configure a custom `xsrfHeaderName` and make a cross-origin request, the XSRF token IS still sent. With 1.6.0+, it's only sent to same-origin requests unless `withXSRFToken: true` is explicitly set.

**Marcus** [11:52]: This changes things. So our Round 1 proposal of 1.4.0 is incorrect — it doesn't actually fix the CVE we're targeting. What should the override be?

**Danielle** [11:58]: 1.6.8. That's the latest 1.6.x release and includes the XSRF fix plus three subsequent patch fixes for related issues. There's no reason to stop at 1.6.0 when 1.6.8 is available and stable.

**Tobias** [12:04]: What about axios 1.7.x? Shouldn't we go to the latest major?

**Danielle** [12:11]: 1.7.x introduced some changes to the request interceptor ordering that caused regressions in several community packages. The axios team recommends 1.6.x for production use until 1.7.x stabilizes. I'd stick with 1.6.8.

**Marcus** [12:16]: Agreed. I'm updating the RFC. Round 1 target of 1.4.0 is formally rejected. Round 3 target is 1.6.8. This is a significant change — 1.4.0 doesn't fix the CVE, and anyone who implemented the Round 1 guidance would still be vulnerable.

**Danielle** [12:21]: Correct. This is exactly why we do multiple rounds of review. Please make sure the RFC clearly marks 1.4.0 as rejected and explains WHY — I don't want anyone reading the RFC in the future and thinking 1.4.0 was a reasonable choice that was simply superseded.

**Marcus** [12:24]: Noted. The rejection reason will be: "Does not fix CVE-2023-45857. The actual fix landed in 1.6.0. 1.4.0 was based on a misreading of the NVD affected range."

---

### E.4 Thread: "express 4.18.3 — why was this proposed?" (2024-01-12)

**Priya** [16:02]: Quick question for whoever proposed express 4.18.3 in Round 1 — can you explain the reasoning? I'm not finding a clear CVE that maps to that version.

**Jordan** [16:09]: That was me. I based it on an internal security scanner report that flagged express < 4.18.3 as vulnerable. I should have cross-referenced with NVD before including it in the RFC. My mistake.

**Priya** [16:15]: What does the scanner say the vulnerability is?

**Jordan** [16:21]: It says "open redirect vulnerability" but doesn't give a CVE number. I looked it up and I think the scanner is conflating two separate issues. There's an open redirect issue in express that was partially addressed across several 4.x releases, and the actual meaningful fix for path traversal issues landed in 4.19.2.

**Priya** [16:28]: CVE-2024-29041. Yes. 4.19.2 is the fix. The vulnerability is that `res.redirect()` in earlier versions would accept arbitrary redirect targets without sufficient validation, enabling open redirect attacks. The fix in 4.19.2 tightens the URL validation.

**Jordan** [16:33]: Right. So 4.18.3 was a guess based on a scanner warning without a CVE. 4.19.2 is the actual fix version and it's been publicly documented.

**Marcus** [16:38]: So Round 1's 4.18.3 is rejected and Round 3 should use 4.19.2. Jordan, can you write up a brief explanation for the RFC so reviewers understand why 4.18.3 was wrong?

**Jordan** [16:42]: Will do. Summary: 4.18.3 was proposed based on an automated scanner report without CVE verification. The scanner was flagging the version range incorrectly. CVE-2024-29041 documents the actual open redirect vulnerability and identifies 4.19.2 as the minimum fixed version.

---

### E.5 Thread: "react — do we override react-dom too?" (2024-01-14)

**Keiko** [10:05]: The RFC currently proposes `react@18.3.1`. Should we also include `react-dom@18.3.1`? They're typically versioned together.

**Priya** [10:12]: Good question. In the MonoStack codebase, `react-dom` is a direct dependency of `@monostack/ui`. It's in the `dependencies` field of ui/package.json at version 18.2.0. Since it's a direct dep (not transitive), an override would be redundant — npm 11 workspace overrides apply to transitive dependencies, but for direct dependencies in workspace packages, the workspace package's own package.json takes precedence.

**Keiko** [10:19]: Wait, so adding `react-dom` to the root overrides wouldn't actually change anything for the ui workspace package?

**Priya** [10:25]: Correct. Overrides in npm 11 don't force workspace package direct dependencies. The ui package directly declares `react-dom@18.2.0`, so that version will be installed regardless of the root override. To change react-dom for the ui package, you'd need to update ui/package.json directly.

**Tobias** [10:31]: So we should update ui/package.json to bump react-dom to 18.3.1 instead of adding a root override?

**Priya** [10:37]: That's a separate PR. The RFC is specifically about root overrides for transitive dependencies. react-dom is a direct dep of ui, so it's out of scope for this RFC. We should track it in a separate ticket.

**Marcus** [10:43]: Makes sense. RFC §12.3 will explicitly state: "react-dom is excluded from this override set because it is a direct dependency of @monostack/ui. Updating react-dom requires a direct package.json change in packages/ui, which is tracked separately."

**Keiko** [10:47]: Perfect. So the root overrides block should have `react` but NOT `react-dom`. If an implementer adds `react-dom` to the overrides, it would be technically harmless but incorrect per the RFC's stated scope.

**Marcus** [10:51]: Correct. And our tests should verify that `react-dom` is NOT in the overrides block, to catch implementations that blindly included it.

---

### E.6 Thread: "Final decisions confirmation" (2024-01-16)

**Marcus** [09:00]: Posting the final decisions for sign-off. Please respond with your name and "approved" or raise any final concerns before EOD.

Final overrides for RFC-2024-047:
- `lodash`: `4.17.21` (fixes CVE-2021-23337; latest 4.x release)
- `semver`: `7.6.3` (fixes CVE-2022-25883 fully; 7.5.x versions remain partially vulnerable)
- `axios`: `1.6.8` (fixes CVE-2023-45857; 1.4.0 rejected — does not fix the CVE)
- `express`: `4.19.2` (fixes CVE-2024-29041; 4.18.3 rejected — wrong version)
- `react`: `18.3.1` (security hardening + latest stable; react-dom excluded — direct dep of ui)

**Priya** [09:14]: Priya — approved.

**Reuben** [09:22]: Reuben — approved.

**Danielle** [09:31]: Danielle — approved. Happy this went through proper review. The 1.4.0 → 1.6.8 correction alone was worth the extra rounds.

**Tobias** [09:38]: Tobias — approved. Glad we sorted the semver version too.

**Keiko** [09:45]: Keiko — approved.

**Jordan** [10:02]: Jordan — approved. Thanks for catching the express version issue.

**Marcus** [10:05]: All six approvals received. RFC-2024-047 is finalized. I'll update the document and send it to the Architecture Review Board for final ratification.

---

## 19. Appendix F: Architecture Review Board Session Notes

This appendix contains the notes from the Architecture Review Board (ARB) sessions that reviewed and ratified RFC-2024-047. The ARB meets bi-weekly and has authority over all dependency management decisions that affect more than one team.

---

### F.1 ARB Session 2024-01-18 — Initial RFC Presentation

**Attendees:** Marcus (presenter), Priya, Reuben, Keiko, Tobias, Danielle, Jordan, Valentina (ARB chair), Desmond (ARB security lead), Fatima (ARB platform lead)

**Session objective:** Present RFC-2024-047 for ARB review and identify any concerns before the final ratification vote.

#### F.1.1 Opening Remarks (Valentina)

Valentina opened the session by noting that this was the first RFC to go through the new multi-round review process. She emphasized that the ARB's role was not to re-litigate decisions already made by the Security Guild, but to verify that the process was followed correctly and that the final decisions were consistent with the platform's security posture.

"We've seen too many CVE remediations that were done quickly and incorrectly," Valentina said. "The whole point of the new process is to make sure we're fixing the right thing to the right version, and not just satisfying a scanner alert."

#### F.1.2 Marcus's Presentation

Marcus walked through the RFC's history, starting with the Round 1 proposals and explaining why three of the five initial proposals were revised in subsequent rounds.

Key points from the presentation:

**On lodash:** Marcus explained that the lodash situation was straightforward — CVE-2021-23337, fix in 4.17.21, and that version happens to be the latest release. No debate needed.

**On semver:** Marcus described the two-phase fix story: initial patch in 7.5.2 addressed the severe case, but the underlying regex vulnerability persisted in the 7.5.x line. The 7.6.x rewrite resolved it completely. He showed timing data from Keiko's analysis demonstrating that 7.5.4 was still measurably vulnerable to crafted inputs.

**On axios:** This was the most significant correction. Marcus explained the original 1.4.0 proposal was based on a misread NVD entry. He walked through Danielle's analysis showing that the actual XSRF fix didn't land until 1.6.0, and that 1.6.8 was the appropriate target.

**On express:** Marcus described Jordan's original proposal and its basis in an internal scanner report without CVE backing. He explained CVE-2024-29041 and why 4.19.2 is the correct target.

**On react:** Marcus noted this was relatively uncomplicated — 18.3.1 is the latest stable, it includes the security hardening changes, and the decision to exclude react-dom was made deliberately because it's a direct workspace dependency.

#### F.1.3 Desmond's Security Review

Desmond, the ARB security lead, had independently reviewed the CVE analyses before the session. His comments:

"I want to validate three specific claims from the RFC before we ratify.

"First, on axios 1.4.0: I confirmed independently that CVE-2023-45857 is not fixed in 1.4.0. The NVD entry has been updated since the original filing and the affected versions range is listed as 'all versions >= 1.0.0 prior to the patch released in 1.6.0'. This aligns with the RFC's conclusion that 1.4.0 was wrong.

"Second, on semver 7.5.4: I ran the ReDoS test payload against 7.5.4 in our test environment. Input of 200 characters designed to trigger backtracking caused a 78ms delay per invocation. In our current API rate of 5000 semver validations per minute at peak, this would allow a denial of service to be achieved with approximately 40 concurrent requests. 7.6.3 shows 0ms for the same input. The upgrade to 7.6.3 is clearly necessary.

"Third, on express 4.18.3: I reproduced the CVE-2024-29041 vulnerability on express 4.18.3. A crafted redirect URL of the form `///attacker.com/path` bypasses the origin check and results in an open redirect. 4.19.2 correctly rejects this input. The RFC's rejection of 4.18.3 is correct."

#### F.1.4 Fatima's Platform Review

Fatima reviewed the RFC from a platform compatibility perspective:

"I want to make sure none of these overrides break the build in CI or cause issues for teams building on top of MonoStack.

"lodash 4.17.21 — fully compatible with our CI pipeline. All 47 unit tests pass.

"semver 7.6.3 — I ran the full test suite for packages that import semver. The only change I observed was that one test in @monostack/cli was previously testing against a specific pre-release edge case that was corrected in 7.6.1. The test itself was wrong (it was asserting incorrect behavior), so the failure actually revealed a latent bug in the test suite. That test has been fixed separately.

"axios 1.6.8 — the XSRF behavior change requires attention. Any client code that was relying on axios to automatically send XSRF tokens to cross-origin requests will need to be updated to use `withXSRFToken: true`. I audited the MonoStack codebase and found zero instances of cross-origin requests that use XSRF tokens. So the breaking behavior change has no impact on us, but this should be documented for downstream consumers of `@monostack/api`.

"express 4.19.2 — the redirect URL validation change is backward-compatible for all valid redirect patterns. Any code that was previously relying on invalid redirect behavior (which would be a bug) might break. No such code exists in MonoStack.

"react 18.3.1 — the ui package currently pins to 18.2.0 in its direct dependencies. The root override only affects transitive consumers. However, react 18.3.1 introduced deprecation warnings for legacy lifecycle methods that are used in three of our third-party UI dependencies. This is a warning only, not a breakage, but teams should be aware."

#### F.1.5 ARB Discussion

Several questions arose during the discussion:

**Question (Desmond):** Should we include `react-dom` in the override given that react and react-dom are always expected to be in sync?

**Response (Priya):** As discussed in the Security Guild, react-dom is a direct dependency of @monostack/ui, not a transitive dependency. npm 11's override behavior doesn't force direct workspace dependencies. Adding it to the root override block would create a false sense of security — the override would be silently ignored for the workspace package that actually depends on it. The correct fix is to update ui/package.json, which is being tracked as a separate ticket.

**Question (Fatima):** Is there a lockfile strategy for ensuring these overrides persist across npm install runs?

**Response (Marcus):** Yes. The implementation plan calls for deleting the stale package-lock.json and running `npm install` fresh after the overrides are applied. The new lockfile will reflect the override versions. Going forward, `npm ci` should be used in CI to prevent the lockfile from drifting.

**Question (Valentina):** What happens if a workspace package updates its dependencies and introduces a version that conflicts with an override?

**Response (Priya):** npm will use the override version for transitive dependencies. If a workspace package has a direct dependency that conflicts with an override, the direct dependency wins for that workspace package's own node_modules resolution, but transitive consumers of the overridden package will still get the pinned version. This is a known limitation of npm 11 workspace overrides and is documented in the RFC's §11 section on known limitations.

#### F.1.6 ARB Vote

Motion: "Ratify RFC-2024-047 with the final override set as presented."

Vote:
- Valentina: Approve
- Desmond: Approve
- Fatima: Approve (with note: downstream consumers of @monostack/api should be informed of the axios XSRF behavior change)

Motion carried unanimously. RFC-2024-047 is ratified.

---

### F.2 ARB Session 2024-01-25 — Implementation Verification

**Attendees:** Marcus, Priya, Reuben, Valentina (chair), Desmond, Fatima

This follow-up session was held after the initial implementation of the overrides to verify that the implementation was correct.

#### F.2.1 Implementation Report (Marcus)

Marcus reported that the overrides had been applied to the root package.json as specified in §12. The package-lock.json had been regenerated. Key findings from the verification:

"I ran `npm ls lodash` after applying the overrides and regenerating the lockfile. All 14 transitive consumers now resolve to 4.17.21. Previously, we had a mix of 4.17.19 and 4.17.20 depending on which consumer's lockfile section was consulted. This is now consistent.

"For semver, `npm ls semver` shows 8 consumers all at 7.6.3. Previously, these ranged from 7.3.8 to 7.5.3.

"For axios, the single transitive consumer (a dependency of the metrics library) was at 1.3.4. It's now at 1.6.8.

"For express, we had two transitive consumers. Both are now at 4.19.2.

"For react, the three transitive consumers that pulled in React (testing-library packages) were at 18.2.0. They're now at 18.3.1."

#### F.2.2 Desmond's Post-Implementation Security Verification

"I re-ran the vulnerability scans after the override implementation. All five CVEs are now marked as remediated in our internal scanner. The scanner's CVE database was updated two weeks ago to correctly reflect that CVE-2023-45857 is fixed in 1.6.0+, not 1.4.0+, so this is consistent with our RFC conclusion.

"I also verified the semver fix specifically because it was the one I was most concerned about. The 200-character ReDoS test payload now returns in 0ms across 1000 iterations. The fix is confirmed effective."

#### F.2.3 Fatima's CI Verification

"Full CI pipeline run completed successfully with the new lockfile. 412 tests pass, 0 failures. The one test in @monostack/cli that was previously masking incorrect semver behavior has been fixed. All workspace packages build cleanly.

"I also ran the build on three different developer machines (two Linux, one macOS) to verify that `npm ci` produces a consistent result. All three machines produced identical node_modules trees."

#### F.2.4 ARB Closure

Valentina formally closed the RFC:

"RFC-2024-047 is implemented and verified. The implementation is consistent with the ratified decisions. No further action is required unless a new CVE surfaces for one of these packages.

"I want to note for the record that this RFC took three rounds of review and caught two significant errors from the initial proposals — axios 1.4.0 not actually fixing the CVE, and express 4.18.3 being based on an unverified scanner warning. The multi-round process worked exactly as intended. Please use this RFC as a template for future dependency management decisions."

---

## 20. Appendix G: Dependency Version Timeline and npm Registry Audit

This appendix provides a complete version history for each overridden package as it existed at the time of RFC finalization, along with notes on why intermediate versions were not chosen.

---

### G.1 lodash Complete Version History (4.x series relevant to this RFC)

lodash 4.x has been in maintenance mode since January 2022. The following table documents all 4.x releases from 4.17.15 onward, which is the range relevant to CVE-2021-23337.

**4.17.15** — Released 2019-03-28. This is the baseline version that many npm projects resolved to for years. Contains the prototype pollution vulnerability.

**4.17.16** — Released 2019-07-19. Minor bug fixes in array utilities. Vulnerability present.

**4.17.17** — Released 2019-12-18. Performance improvements to `_.cloneDeep`. Vulnerability present.

**4.17.18** — Released 2019-12-19. Hotfix for a regression in 4.17.17 affecting `_.flatten`. Vulnerability present.

**4.17.19** — Released 2020-07-01. Removed an outdated test dependency. No functional changes. Vulnerability present.

**4.17.20** — Released 2020-09-25. Security fix for CVE-2020-8203 (prototype pollution via `_.zipObjectDeep` in a specific code path). Note: this is a different CVE from CVE-2021-23337. The 4.17.20 fix addressed one pollution vector but left others open. This is the version installed by @monostack/core before this RFC.

**4.17.21** — Released 2021-12-29. Comprehensive fix for prototype pollution across `zipObjectDeep`, `set`, `setWith`, and the `merge` deep clone path. Specifically addresses CVE-2021-23337. This is the final release in the 4.x series. No subsequent 4.x releases exist.

**Why not 5.x?** lodash 5 has been in development since 2019 and has never been released as stable. The alpha versions have a significantly different API surface and would require code changes throughout the MonoStack codebase. Out of scope for this RFC, which targets minimal-change security fixes.

**Summary:** 4.17.21 is simultaneously the CVE fix and the latest available version. There is no alternative to it.

---

### G.2 semver Version History (7.x series relevant to this RFC)

**7.3.8** — Oldest version currently appearing in MonoStack transitive deps. Contains CVE-2022-25883.

**7.5.0** — Released 2023-05-12. Introduced `parse()` performance improvements. Vulnerability present.

**7.5.1** — Released 2023-05-15. Bug fix for coerce() with tag information. Vulnerability present.

**7.5.2** — Released 2023-07-10. First security response to CVE-2022-25883. Added length check of 256 characters on version strings. Reduces severity from "catastrophic" to "moderate." Inputs over 256 characters are rejected fast; inputs under 256 characters still subject to backtracking-prone regex.

**7.5.3** — Released 2023-07-13. Minor fix for multidigit comparators in pre-release identifiers. Vulnerability partially present (same as 7.5.2).

**7.5.4** — Released 2023-07-25. Tightened the length check from 256 to 16 characters. This significantly reduces the practical window for ReDoS but does not eliminate the underlying regex vulnerability. A crafted 14-character payload can still produce measurable latency. This was the Round 1 and Round 2 target in the RFC before Keiko's analysis demonstrated the residual risk.

**7.6.0** — Released 2024-01-05. Major internal change: replaced the backtracking-prone regex for pre-release version parsing with a character-class-based parser that cannot backtrack. Eliminates the ReDoS vector entirely. However, introduced a regression: numeric-only pre-release identifiers (e.g., `1.2.3-0`) were incorrectly treated as invalid by the new parser. This regression made 7.6.0 unsuitable as an override target.

**7.6.1** — Released 2024-01-09. Fixed the numeric pre-release identifier regression from 7.6.0. The core ReDoS fix is intact.

**7.6.2** — Released 2024-01-15. Documentation and README updates only. No functional changes.

**7.6.3** — Released 2024-01-22. Correctness fix for edge case in `compareIdentifiers` when comparing pre-release segments of different types (numeric vs. alphanumeric). Does not affect common usage. This is the final target specified by the RFC.

**Why 7.6.3 and not 7.6.1 or 7.6.2?** The RFC was finalized after 7.6.3 was released, and the policy is to target the latest available fix version to incorporate all subsequent corrections. 7.6.3 is the latest and includes the 7.6.1 regression fix.

---

### G.3 axios Version History (1.x series relevant to this RFC)

**1.0.0** — Released 2022-10-04. Initial 1.x release. Introduced a change to XSRF token handling that inadvertently sent XSRF tokens to cross-origin requests. This is the root cause of CVE-2023-45857.

**1.1.x through 1.3.x** — Minor releases with feature additions and bug fixes. XSRF vulnerability present throughout. @monostack/api's transitive dep chain resolved to 1.3.4 from this range.

**1.4.0** — Released 2023-04-28. Added AbortController support and several interceptor improvements. The NVD entry for CVE-2023-45857 initially listed this as the minimum fixed version, but this was incorrect. The XSRF issue is still present in 1.4.0. The RFC explicitly rejects this version as a fix target.

**1.5.x** — Minor releases. XSRF vulnerability present.

**1.6.0** — Released 2023-10-26. Contains the actual fix for CVE-2023-45857: XSRF tokens are now only sent to same-origin requests by default. Cross-origin requests require explicit opt-in via `withXSRFToken: true`. This is the minimum version that actually fixes the CVE.

**1.6.1** — Released 2023-11-01. Fixed a regression in 1.6.0 where the XSRF fix was too aggressive and blocked some legitimate same-origin requests in certain proxy configurations.

**1.6.2** — Released 2023-11-06. Additional fix for a race condition in the request queue that was surfaced by the 1.6.0 changes.

**1.6.3 through 1.6.7** — Progressive fixes for interceptor behavior, TypeScript types, and FormData handling.

**1.6.8** — Released 2024-01-05. Latest release in the 1.6.x line. Includes all prior fixes and a correction for an edge case where `baseURL` containing query parameters was not handled correctly. This is the RFC's final target.

---

### G.4 express Version History (4.x series relevant to this RFC)

**4.17.x through 4.18.1** — The versions most commonly installed before this RFC. No known critical vulnerabilities in the version range used by MonoStack transitive deps.

**4.18.2** — Released 2022-10-08. Bug fixes for router and middleware. No security relevance.

**4.18.3** — Released 2024-01-09. Minor dependency updates. The internal security scanner incorrectly flagged this as a security fix version. In reality, 4.18.3 does not address CVE-2024-29041. This version was proposed in RFC Round 1 and explicitly rejected in the Security Guild discussion.

**4.19.0** — Released 2024-02-09. First release to include the fix for CVE-2024-29041 (open redirect via `res.redirect()`). The fix validates the redirect target URL more strictly.

**4.19.1** — Released 2024-02-14. Corrected an edge case in the redirect URL validation where relative URLs containing protocol-relative paths were incorrectly rejected.

**4.19.2** — Released 2024-02-28. Additional hardening of the URL validation plus fixes for two separate issues in the query string parser that were unrelated to the CVE but were discovered during the security review. This is the RFC's final target.

**Why not express 5.x?** Express 5.0 was released as stable in 2024-09 (after this RFC was written). At the time of RFC finalization, Express 5 was in release candidate status and not recommended for production use. Additionally, Express 5 has breaking changes that would require updates to route handler signatures throughout @monostack/api.

---

### G.5 react Version History (18.x series relevant to this RFC)

**18.0.0** — Released 2022-03-29. Initial 18.x release with concurrent rendering. The version in widespread use before this RFC.

**18.1.0** — Released 2022-04-26. Bug fixes for Suspense and hydration.

**18.2.0** — Released 2022-06-14. Performance improvements and bug fixes. This is the version directly declared in @monostack/ui and the version in the transitive dependency tree of testing libraries used by the other workspace packages.

**18.3.0** — Released 2024-04-25. Security hardening release. Key changes: deprecated `ReactDOM.render` (legacy API with known memory leak patterns), added warnings for legacy lifecycle methods (`componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`), improved prop-types validation to prevent type confusion attacks in edge cases.

**18.3.1** — Released 2024-05-01. Corrected an issue in 18.3.0 where the deprecation warnings for legacy lifecycle methods were triggering in test environments even when the methods were only used in third-party libraries (not project code). Changed to only warn when the project code directly uses these methods. This is the RFC's final target.

**Why not react 19.x?** React 19.0 was released in 2024-12 (after this RFC was written). At the time of RFC finalization, React 19 was in RC status and had breaking changes for several patterns used in @monostack/ui's component tree.

---

*End of Appendix G.*

---

## 21. Appendix H: Implementation Runbook

This appendix provides the step-by-step implementation runbook used by the platform team to apply the overrides from RFC-2024-047. It is preserved here for audit purposes and to guide future implementers making similar changes.

---

### H.1 Pre-Implementation Checklist

Before making any changes to package.json, complete all items in this checklist:

**H.1.1 Confirm Current State**

Run the following commands to capture a baseline snapshot of the dependency tree:

```bash
cd /app
npm ls lodash 2>/dev/null | grep lodash | sort | uniq > /tmp/baseline_lodash.txt
npm ls semver 2>/dev/null | grep semver | sort | uniq > /tmp/baseline_semver.txt
npm ls axios 2>/dev/null | grep axios | sort | uniq > /tmp/baseline_axios.txt
npm ls express 2>/dev/null | grep express | sort | uniq > /tmp/baseline_express.txt
npm ls react 2>/dev/null | grep ' react@' | sort | uniq > /tmp/baseline_react.txt
```

These baseline files will be used in post-implementation verification to confirm the overrides took effect.

**H.1.2 Verify npm Version**

The overrides field requires npm 8.3.0 or later. npm 11.x is the version used during RFC development and testing.

```bash
npm --version
```

Expected: 11.x.x or higher. If npm is older than 8.3.0, update before proceeding.

**H.1.3 Verify Node.js Version**

```bash
node --version
```

Expected: 22.x or higher. The MonoStack repository is configured for Node 22.

**H.1.4 Confirm Clean Working Tree**

```bash
git status
```

Expected: clean working tree with no uncommitted changes. If changes exist, stash or commit them before proceeding. Applying the RFC to a dirty tree makes it harder to isolate the impact of the changes.

**H.1.5 Confirm Repository Root**

```bash
pwd
```

Expected: `/app` (or equivalent root of the MonoStack monorepo). Confirm the presence of the root package.json with the `workspaces` field:

```bash
node -e "const p = require('./package.json'); console.log('workspaces:', JSON.stringify(p.workspaces))"
```

Expected output: `workspaces: ["packages/*"]`

---

### H.2 Step 1 — Read and Verify the Current package.json

Before adding the overrides field, confirm the current state of package.json:

```bash
node -e "
const fs = require('fs');
const pkg = JSON.parse(fs.readFileSync('/app/package.json', 'utf8'));
console.log('Current overrides field:', JSON.stringify(pkg.overrides, null, 2));
console.log('Workspaces:', JSON.stringify(pkg.workspaces));
console.log('Name:', pkg.name);
console.log('Version:', pkg.version);
"
```

If the overrides field is already present, make note of its current contents. If it is null or undefined, that is the expected starting state for a fresh RFC implementation.

---

### H.3 Step 2 — Apply the Overrides

The overrides field must be added to the root package.json. The final approved overrides from RFC-2024-047 §12 are:

```json
{
  "overrides": {
    "lodash": "4.17.21",
    "semver": "7.6.3",
    "axios": "1.6.8",
    "express": "4.19.2",
    "react": "18.3.1"
  }
}
```

These can be applied programmatically using Node.js to avoid any risk of JSON formatting errors:

```javascript
const fs = require('fs');
const path = require('path');

const pkgPath = path.join('/app', 'package.json');
const pkg = JSON.parse(fs.readFileSync(pkgPath, 'utf8'));

// Apply only the approved overrides. Do not add react-dom.
pkg.overrides = {
  lodash: '4.17.21',
  semver: '7.6.3',
  axios: '1.6.8',
  express: '4.19.2',
  react: '18.3.1',
};

fs.writeFileSync(pkgPath, JSON.stringify(pkg, null, 2) + '\n');
console.log('Overrides applied successfully.');
console.log('New overrides:', JSON.stringify(pkg.overrides, null, 2));
```

Save this script as `/tmp/apply-overrides.js` and run it with:

```bash
node /tmp/apply-overrides.js
```

**Important:** Do not add `react-dom` to the overrides block. See §12.3 and Appendix E.5 for the explanation. Do not add any packages not listed above.

---

### H.4 Step 3 — Regenerate the Lockfile

After applying the overrides, the existing package-lock.json must be deleted and regenerated. The existing lockfile contains pinned resolution paths that will conflict with the new overrides.

```bash
cd /app
rm -f package-lock.json
npm install --prefer-offline
```

The `--prefer-offline` flag is used to prevent npm from attempting network requests for packages that are already in the local cache. In environments without internet access, this prevents unnecessary failures.

If the install fails with an error about missing packages in the cache, run without `--prefer-offline`:

```bash
npm install
```

Expected output: npm should install all packages successfully and produce a new `package-lock.json`. The output should not contain any `npm warn` messages about missing peer dependencies.

---

### H.5 Step 4 — Verify the Override Versions

After the install completes, verify that the overrides took effect using `npm ls`:

```bash
cd /app
echo "=== lodash ===" && npm ls lodash 2>/dev/null | grep 'lodash@'
echo "=== semver ===" && npm ls semver 2>/dev/null | grep 'semver@'
echo "=== axios ===" && npm ls axios 2>/dev/null | grep 'axios@'
echo "=== express ===" && npm ls express 2>/dev/null | grep 'express@'
echo "=== react ===" && npm ls react 2>/dev/null | grep ' react@'
```

Expected output for each package:

- All lodash entries should show `lodash@4.17.21`
- All semver entries should show `semver@7.6.3`
- All axios entries should show `axios@1.6.8`
- All express entries should show `express@4.19.2`
- All react entries (excluding react-dom and react-related packages) should show `react@18.3.1`

If any entry shows a version different from the expected value, the override did not take effect for that package. Common causes:

1. **Direct dependency in a workspace package**: If a workspace package lists the package as a direct dependency (in `dependencies`, not `devDependencies`), npm 11 workspace overrides will not force the override version for that workspace package's own resolution. Only update the root `overrides` for transitive dependencies.

2. **Stale lockfile**: If you did not delete `package-lock.json` before running `npm install`, the old lockfile may have constrained resolution. Delete the lockfile and retry.

3. **Incorrect override syntax**: Verify the overrides field in package.json uses the exact format shown in H.3. String values (not objects) are required for simple version pinning.

---

### H.6 Step 5 — Verify the Lockfile Contents

The new `package-lock.json` should reflect the override versions. Spot-check using:

```bash
node -e "
const fs = require('fs');
const lock = JSON.parse(fs.readFileSync('/app/package-lock.json', 'utf8'));
const check = ['lodash', 'semver', 'axios', 'express', 'react'];
const expected = {lodash:'4.17.21', semver:'7.6.3', axios:'1.6.8', express:'4.19.2', react:'18.3.1'};
for (const pkg of check) {
  const entry = lock.packages['node_modules/' + pkg];
  const ver = entry ? entry.version : 'NOT FOUND';
  const status = ver === expected[pkg] ? 'OK' : 'MISMATCH';
  console.log(status + ' ' + pkg + ': ' + ver + ' (expected ' + expected[pkg] + ')');
}
"
```

All entries should show `OK`. If any show `MISMATCH`, the lockfile was not correctly regenerated.

Additionally, verify the lockfile format version:

```bash
node -e "
const lock = require('/app/package-lock.json');
console.log('lockfileVersion:', lock.lockfileVersion);
"
```

Expected: `3` (npm 7+ format). If the lockfile version is 1 or 2, the lockfile was generated by an older npm version. Update npm and regenerate.

---

### H.7 Step 6 — Run the Test Suite

After verifying the dependency tree, run the full test suite to confirm no regressions were introduced:

```bash
cd /app
npm test
```

Expected: all tests pass. The test suite includes both unit tests for individual packages and integration tests that exercise cross-package imports.

If tests fail after applying the overrides:

1. Check whether the failing test relied on behavior that changed in the override version. Refer to the version history in Appendix G for known behavior changes.

2. In particular, note the axios XSRF change (described in §F.2.3): any test that was asserting that XSRF tokens ARE sent to cross-origin requests will now fail. This represents a test asserting incorrect (vulnerable) behavior, not a regression. Update the test.

3. The semver 7.6.x change may cause failures in tests that were asserting incorrect behavior for numeric-only pre-release identifiers. Same resolution: update the test.

---

### H.8 Step 7 — Post-Implementation Comparison

Compare the post-implementation dependency tree to the baseline captured in H.1.1:

```bash
npm ls lodash 2>/dev/null | grep lodash | sort | uniq > /tmp/post_lodash.txt
diff /tmp/baseline_lodash.txt /tmp/post_lodash.txt
```

Repeat for each package. The diff should show that all entries moved to the expected override version. Entries at the old version should appear as deletions and entries at the new version as additions.

---

### H.9 Known Limitations

**H.9.1 Direct Workspace Dependencies**

As noted throughout this RFC, npm 11 workspace overrides do not force override versions for direct dependencies declared in workspace packages. The following direct dependencies are NOT affected by this RFC's overrides:

- `react-dom@18.2.0` in `packages/ui/package.json` (direct dep) — must be updated separately
- Any other direct dependencies in workspace packages that happen to share a name with an overridden package

**H.9.2 Post-Override npm install Behavior**

After the overrides are applied, running `npm install` (not `npm ci`) in the future may re-resolve the lockfile if constraints change. Always prefer `npm ci` in CI environments to prevent lockfile drift.

**H.9.3 Override Propagation Timing**

If a developer runs `npm install` in an individual workspace package directory (e.g., `cd packages/api && npm install`), the root-level overrides from the monorepo root package.json are still applied because npm resolves the workspace root. However, if a workspace package is installed in isolation outside the monorepo context, the root overrides will not be in scope.

**H.9.4 Nested Package Manager Conflicts**

If any workspace package uses yarn or pnpm locally (which should not be the case in MonoStack but is worth noting), those package managers will not respect npm's `overrides` field. Yarn uses `resolutions` and pnpm uses `overrides` in a different format. MonoStack uses npm throughout, so this is not a current concern.

---

### H.10 Rollback Procedure

If the overrides need to be reverted (e.g., due to an unexpected regression that cannot be resolved quickly), follow this procedure:

```bash
cd /app

# 1. Remove the overrides field
node -e "
const fs = require('fs');
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf8'));
delete pkg.overrides;
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2) + '\n');
console.log('Overrides removed.');
"

# 2. Restore the baseline lockfile from git
git checkout package-lock.json

# 3. Reinstall using the restored lockfile
npm ci
```

Note: rolling back the overrides means the CVEs described in this RFC are no longer mitigated. A rollback should be accompanied by a security incident report and a plan to remediate the underlying issue that caused the regression.

---

## 22. Appendix I: Cross-Package Impact Analysis

This appendix documents the full impact analysis of each override across all four workspace packages (`@monostack/core`, `@monostack/api`, `@monostack/ui`, `@monostack/cli`) and their known transitive consumers.

---

### I.1 lodash@4.17.21 — Full Impact Assessment

**Summary:** 14 packages in the dependency tree depend on lodash. All 14 will receive 4.17.21 after the override is applied.

**Direct workspace consumers:**

`@monostack/core` declares `lodash@^4.17.0` in its direct dependencies. Because this is a direct workspace package dependency, the override does not force the version for this specific package. However, since ^4.17.0 is compatible with 4.17.21 and 4.17.21 is the latest available, npm will still resolve to 4.17.21 naturally. In practice, the override and the direct dep constraint agree on the same version.

`@monostack/api` does not directly depend on lodash but uses it transitively through three dependencies:
- `request-validator@2.1.0` depends on `lodash@^4.0.0`
- `schema-utils@4.2.0` depends on `lodash@^4.17.14`
- `config-loader@1.3.2` depends on `lodash@^4.0.0`

All three transitive consumers are affected by the override.

`@monostack/ui` uses lodash transitively through:
- `@testing-library/react@14.0.0` depends on `lodash@^4.0.0`
- `@testing-library/jest-dom@6.1.0` depends on `lodash@^4.0.0`

`@monostack/cli` uses lodash transitively through:
- `commander@11.0.0` has no lodash dependency
- `chalk@4.1.2` has no lodash dependency
- `inquirer@9.0.0` depends on `lodash@^4.0.0`
- `ora@7.0.1` has no lodash dependency
- `execa@8.0.1` has no lodash dependency

**Compatibility assessment:** All 14 consumers use a semver range that includes 4.17.21. No compatibility issues expected.

**Behavioral changes from 4.17.20 to 4.17.21:** Only the prototype pollution fix for `zipObjectDeep`, `set`, `setWith`, and the `merge` deep clone path. No changes to function signatures, return value shapes, or observable behavior for non-malicious inputs. All MonoStack usage of lodash passes non-malicious inputs. No test changes expected.

---

### I.2 semver@7.6.3 — Full Impact Assessment

**Summary:** 8 packages in the dependency tree depend on semver.

**@monostack/core** — no direct semver dependency. Transitive consumers:
- `node-fetch@3.3.2` depends on semver for version comparison in its peer check logic (devDependency context). This usage is not part of the runtime bundle.

**@monostack/api** — no direct semver dependency. Transitive consumers:
- `express@4.19.2` has an internal dependency on semver for version negotiation with middleware
- `body-parser@1.20.2` uses semver for version compatibility checks

**@monostack/ui** — uses semver transitively through build tooling:
- `webpack@5.88.0` depends on `semver@^7.0.0`
- `babel-loader@9.1.3` depends on `semver@^6.0.0 || ^7.0.0`
- `postcss-loader@7.3.0` depends on `semver@^7.0.0`
- `css-loader@6.8.0` depends on `semver@^7.0.0`

**@monostack/cli** — uses semver transitively through:
- `update-notifier@7.0.0` depends on `semver@^7.5.3` to compare installed vs. latest versions
- `npm-check-updates@16.14.5` depends on `semver@^7.0.0`

**Compatibility assessment:** All 8 consumers use ranges compatible with 7.6.3. The behavior change in 7.6.x (regex rewrite) is not observable through the public API for valid semver strings. The only observable change is that maliciously crafted strings no longer cause ReDoS.

**Special case — update-notifier:** update-notifier declares `semver@^7.5.3`, which technically allows 7.5.x or higher. The override to 7.6.3 satisfies this range. No issues.

**Known edge case (documented in F.1.4):** One test in @monostack/cli was asserting incorrect behavior for numeric-only pre-release identifiers. Specifically, the test was checking that `semver.parse('1.0.0-0')` returned null (indicating an invalid version). In semver 7.6.x, `1.0.0-0` is correctly recognized as valid (a pre-release version with numeric identifier `0`). The test was wrong; it was testing a bug that existed in 7.5.x. This test was fixed as a separate commit before the lockfile regeneration.

---

### I.3 axios@1.6.8 — Full Impact Assessment

**Summary:** 1 transitive consumer in the dependency tree depends on axios directly. The workspace packages do not directly list axios as a dependency in this version of the RFC, but third-party transitive deps pull it in.

**@monostack/core** — no axios dependency.

**@monostack/api** — transitive consumer:
- `metrics-client@2.3.0` depends on `axios@^1.0.0` for reporting metric events to the monitoring endpoint

**@monostack/ui** — transitive consumer:
- `@testing-library/user-event@14.5.0` uses axios for its internal API call simulation helpers

**@monostack/cli** — transitive consumer:
- `update-notifier@7.0.0` uses axios for registry API calls

**XSRF Behavior Change (Critical):**

The most significant behavioral change between pre-1.6.0 and 1.6.8 is the XSRF token handling. As described in the Security Guild discussion (Appendix E.3) and the ARB session (Appendix F.1.4 and F.2.3), axios 1.6.0 changed the default behavior so that XSRF tokens are no longer sent to cross-origin requests.

For each identified consumer:

- `metrics-client@2.3.0`: Makes requests to a monitoring service. The monitoring service is an internal endpoint on the same domain. XSRF tokens were never involved in these requests. No impact.

- `@testing-library/user-event@14.5.0`: The axios usage in this library is for simulating API calls in tests, not for making real HTTP requests. The XSRF change has no impact on test execution.

- `update-notifier@7.0.0`: Makes requests to the npm registry (external). The npm registry does not use XSRF tokens. No impact.

**Compatibility assessment:** Zero impact on MonoStack's actual behavior. The vulnerability is mitigated without any observable functional change.

---

### I.4 express@4.19.2 — Full Impact Assessment

**Summary:** 2 transitive consumers in the dependency tree use express.

**@monostack/api** directly depends on express as a devDependency (for testing the API server middleware). The root override does not affect direct workspace package dependencies, but since @monostack/api declares `express@^4.0.0`, which is satisfied by 4.19.2, npm will resolve to 4.19.2 naturally.

Transitive consumers:
- `serve-static@1.15.0` has an internal peer dependency on express (used in tests)
- `connect@3.7.0` uses express types for middleware compatibility

**Redirect URL Validation Change:**

express 4.19.2 tightens URL validation in `res.redirect()`. The following input patterns are now rejected:

1. Protocol-relative URLs in redirect targets (e.g., `//evil.com/path`) — these are now rejected unless the URL is within the allowed host list
2. URLs with backslash path separators that could be interpreted differently by different user agents

MonoStack audit results:
- All uses of `res.redirect()` in @monostack/api use either relative paths (e.g., `/login`) or absolute URLs constructed from a trusted template string. No dynamic redirect targets that could be attacker-controlled.
- Three integration tests that were calling `res.redirect('//external-service.example.com/...')` were found. These were pre-existing tests that would have been bypassing the new validation. The tests were updated to use absolute URLs with explicit scheme. This is considered a correctness improvement, not a regression.

**Compatibility assessment:** No functional regressions. Three test updates required for redirect URL format (improvements, not regressions).

---

### I.5 react@18.3.1 — Full Impact Assessment

**Summary:** 3 transitive consumers depend on react. The @monostack/ui package also has a direct dependency on react-dom@18.2.0, which is NOT covered by this RFC's override (see §12.3).

**@monostack/ui** — transitive consumers:
- `@testing-library/react@14.0.0` requires `react@^18.0.0`
- `react-beautiful-dnd@13.1.1` requires `react@^16.8.5 || ^17 || ^18`
- `react-hot-toast@2.4.1` requires `react@>=16`

All three ranges are satisfied by 18.3.1.

**Deprecation Warnings:**

react 18.3.1 introduces deprecation warnings for legacy lifecycle methods. The @monostack/ui codebase itself does not use any deprecated lifecycle methods. However, three third-party packages used in the UI layer do:

- `react-beautiful-dnd@13.1.1` uses `componentWillReceiveProps` internally. This generates deprecation warnings in development mode but does not affect production builds. The react-beautiful-dnd maintainers have a PR open to address this.

- `react-hot-toast@2.4.1` uses `componentDidMount` and `componentWillUnmount`, which are NOT deprecated. This package generates no warnings.

- `@testing-library/react@14.0.0` uses `ReactDOM.render` in some of its internal utilities for testing legacy component patterns. This generates deprecation warnings when running tests. No impact on the application itself.

**Suppressing False-Positive Test Warnings:**

To suppress the ReactDOM.render deprecation warnings that now appear in the test output (and would otherwise alarm developers running the test suite), the following configuration was added to the jest setup file in @monostack/ui:

```javascript
// jest.setup.js — suppress known react 18.3 deprecation warnings from third-party libs
const originalError = console.error;
console.error = (...args) => {
  if (args[0]?.includes('ReactDOM.render is no longer supported') && 
      args[0]?.includes('testing-library')) {
    return;
  }
  originalError(...args);
};
```

This suppression is scoped to the specific warning from testing-library and does not mask genuine errors.

**react-dom compatibility note:**

Although @monostack/ui has `react-dom@18.2.0` as a direct dependency and this RFC does not override it, react 18.3.1 and react-dom 18.2.0 are compatible. React follows the convention that react and react-dom must be the same minor version but may differ in patch version. 18.3.1 (react) and 18.2.0 (react-dom) are cross-compatible because both are in the 18.x line. The 18.3.x react-dom update is tracked as a separate ticket.

---

### I.6 Summary Impact Table

| Package | Override Version | Consumers Affected | Behavior Changes | Test Updates Needed |
|---|---|---|---|---|
| lodash | 4.17.21 | 14 | Security fix only (prototype pollution path) | None |
| semver | 7.6.3 | 8 | ReDoS fix; numeric pre-release edge case corrected | 1 (pre-release test bug fixed) |
| axios | 1.6.8 | 3 | XSRF token no longer sent to cross-origin by default | None (no cross-origin XSRF usage found) |
| express | 4.19.2 | 2 | Redirect URL validation tightened | 3 (redirect URL format corrections) |
| react | 18.3.1 | 3 | Deprecation warnings for legacy APIs | 1 (jest setup for warning suppression) |

Total test changes required across all packages: 5 (4 corrections to tests asserting incorrect behavior, 1 jest configuration addition). None of these represent functional regressions in production behavior.

---

## 23. Appendix J: Post-Implementation Monitoring and Long-Term Maintenance

This appendix documents the monitoring strategy and long-term maintenance plan for the overrides introduced by RFC-2024-047.

---

### J.1 Post-Implementation Monitoring

Following the deployment of RFC-2024-047's overrides to all environments, the platform team established the following monitoring checkpoints:

**J.1.1 Automated CVE Scanning**

Dependabot is configured to scan the MonoStack repository weekly. After the overrides are applied, the following CVE alerts should clear:
- CVE-2021-23337 (lodash) — clears when all lodash in tree >= 4.17.21
- CVE-2022-25883 (semver) — clears when all semver in tree >= 7.6.3
- CVE-2023-45857 (axios) — clears when all axios in tree >= 1.6.0
- CVE-2024-29041 (express) — clears when all express in tree >= 4.19.0
- Various react security advisories — clears when react in tree >= 18.3.1

If any of these alerts persist after the override implementation, it indicates either:
1. The override was not correctly applied to the relevant transitive consumer, or
2. The direct dependency in a workspace package is still at the older version (for direct deps, overrides don't apply — direct package.json updates are required)

**J.1.2 npm Audit**

Run weekly as part of CI:

```bash
npm audit --audit-level=high
```

Expected output after RFC implementation: zero high-severity vulnerabilities related to the five packages covered by this RFC. If `npm audit` still flags these packages after the override, it may be using the NVD's corrected data (note: npm audit for CVE-2023-45857 may still flag axios versions below 1.6.0 in some workspace packages if they are direct dependencies not covered by the override).

**J.1.3 Lockfile Drift Detection**

The CI pipeline is configured to fail if `package-lock.json` is out of sync with `package.json`. This uses `npm ci` in CI which exits with a non-zero code if the lockfile doesn't match:

```bash
npm ci --ignore-scripts
```

Any developer who adds new dependencies without regenerating the lockfile will see a CI failure. This prevents the overrides from being silently bypassed by a stale lockfile.

---

### J.2 Maintenance Plan

**J.2.1 Override Lifecycle**

Overrides in `package.json` are intended as temporary fixes, not permanent solutions. The long-term goal is for each override to become unnecessary either because:

1. The transitive consumer updates its direct dependency to a version >= the override version (at which point the override becomes redundant but harmless), or
2. A major version migration removes the transitive dependency entirely

The platform team will review the override set quarterly and remove any override that is no longer needed.

**J.2.2 Quarterly Review Procedure**

On the first Monday of each quarter:

1. Run `npm ls [package]` for each overridden package to check whether any consumers still resolve below the override version when the override is temporarily removed.

2. Temporarily remove the override for each package and run `npm install`:
   ```bash
   node -e "const f='package.json', p=JSON.parse(require('fs').readFileSync(f,'utf8')); delete p.overrides.lodash; require('fs').writeFileSync(f, JSON.stringify(p,null,2)+'\n');"
   npm install
   npm ls lodash | grep lodash
   ```
   If all consumers already resolve to >= 4.17.21 without the override, the override can be permanently removed.

3. Restore the override if any consumer still resolves below the pinned version:
   ```bash
   git checkout package.json
   ```

**J.2.3 New CVE Response**

If a new CVE is discovered for one of the packages currently covered by RFC-2024-047, open a new RFC using this document as a template. Do not modify this RFC — it represents the finalized state of the dependency migration as of January 2024.

**J.2.4 Package Manager Updates**

If the monorepo is updated to use npm 12.x or later, verify that the overrides syntax is still compatible. npm has historically maintained backward compatibility for the overrides field, but major version updates should be tested in a branch before being applied to main.

---

### J.3 Lessons Learned

The RFC-2024-047 process produced several lessons that informed the platform team's approach to future dependency management:

**Lesson 1: Always verify CVE fix versions independently.**

The initial proposal of axios 1.4.0 was based on an NVD entry that was subsequently corrected. The NVD is authoritative but not always up-to-date at the time of initial publication. Always cross-reference with the package changelog, the relevant GitHub issues, and the fix PR before specifying a minimum fixed version.

**Lesson 2: Scanner results are a starting point, not a conclusion.**

The initial proposal of express 4.18.3 was based on an internal scanner warning without a CVE identifier. This produced an incorrect target version. Scanners are useful for surfacing issues, but the remediation version must be derived from CVE documentation and the package changelog.

**Lesson 3: npm overrides have workspace limitations.**

The react-dom situation (Appendix E.5) highlighted that npm 11 workspace overrides don't apply to direct dependencies within workspace packages. This is well-documented in npm's documentation but easy to overlook. Always verify whether a target package is a direct dependency or transitive dependency of each workspace package before specifying it as an override.

**Lesson 4: "Latest minor in the fix series" is safer than "minimum fixed version."**

For semver, we could have stopped at 7.6.1 (the minimum that fully fixes CVE-2022-25883 without the 7.6.0 regression). Instead, we used 7.6.3. For axios, we could have used 1.6.0. Instead, we used 1.6.8. The policy of targeting the latest release in the fix series, rather than the minimum, ensures that subsequent bug fixes and correctness improvements are also captured.

**Lesson 5: Document rejections as thoroughly as acceptances.**

Future developers (or AI agents) reading this RFC must understand not just what the final decisions are, but why specific earlier proposals were rejected. The explicit rejection of axios 1.4.0 and express 4.18.3 — with clear reasoning — prevents those incorrect versions from being re-proposed in the future.

---

---

## 24. Appendix K: Developer FAQ and Common Misconceptions

This appendix addresses frequently asked questions received from developers after the RFC was published. It is provided to prevent common mistakes when implementing or extending the RFC's decisions.

---

### K.1 General Questions

**Q: Can I just run `npm audit fix` instead of manually applying these overrides?**

A: No. `npm audit fix` attempts to upgrade packages to minimum fixed versions as determined by npm's audit API. It has several problems that make it unsuitable for this RFC's purposes:

1. It uses npm's interpretation of the affected range for each CVE, which may be based on outdated NVD data (as we saw with axios — npm audit may still consider 1.4.0 a fix for CVE-2023-45857).
2. It modifies direct dependencies by default, not just transitive ones.
3. It does not apply root-level overrides; it modifies the workspace packages' own package.json files.
4. It may upgrade packages beyond the versions we want, potentially introducing other breaking changes.

The manual override approach described in this RFC is the correct method for workspace-level dependency pinning.

---

**Q: Why do we use `overrides` in the root package.json instead of updating each workspace package's package.json?**

A: The overridden packages are transitive dependencies — they are not listed in any workspace package's own `dependencies` field (with the exception of lodash in @monostack/core, but that's a direct dependency whose version range already satisfies 4.17.21). Updating a workspace package's package.json would require either:

1. Adding the transitive package as a direct dependency at the workspace level (incorrect — this conflates the dependency graph), or
2. Modifying the `dependencies` of the third-party package that pulls in the vulnerable version (impossible — we don't control those packages)

Root-level `overrides` is specifically designed for this use case: forcing a specific version for a transitive dependency across the entire workspace tree.

---

**Q: We use yarn in some older repos. Does this RFC apply there?**

A: No. This RFC is specific to the MonoStack repository, which uses npm. Yarn uses a `resolutions` field in the root package.json for similar functionality. The syntax is slightly different, and the behavior differs for workspace packages. If a yarn-based repo needs similar CVE remediations, a separate RFC should be written for that context.

---

**Q: Why doesn't the override for react also cover react-dom?**

A: See §12.3 and Appendix E.5 for the full explanation. Short answer: react-dom is a direct dependency of @monostack/ui, listed in packages/ui/package.json. npm 11's override system does not override direct dependencies of workspace packages. The override would be silently ignored for react-dom's main consumer. The correct approach is to update packages/ui/package.json directly, which is tracked as a separate ticket outside this RFC's scope.

---

**Q: If I add more overrides to the block (e.g., for a new CVE), do I need a new RFC?**

A: Yes. Any additions to the root overrides block should go through the same multi-round review process as this RFC. The purpose of the process is to ensure that (a) the CVE is correctly identified, (b) the minimum fixed version is verified, and (c) the fix version is the latest available in the patched series. Ad-hoc additions to the overrides block without review create the same risks that this RFC's process was designed to prevent.

---

**Q: The tests say "only 5 approved packages in overrides." What if I legitimately need to add a sixth?**

A: The tests enforce the specific decisions from this RFC. If a new package needs to be overridden, the tests should be updated alongside a new RFC that documents the new override. The test is deliberately strict to catch implementers who blindly add packages beyond the RFC's scope. The test should NOT be removed or weakened — it is a feature, not a limitation.

---

### K.2 Questions About Specific Versions

**Q: Is lodash 4.17.20 safe to use if we're not using `zipObjectDeep`?**

A: The answer depends on your usage. CVE-2021-23337 specifically targets `zipObjectDeep`, but the vulnerability class (prototype pollution via crafted path strings) affects `_.set`, `_.setWith`, and the `merge` deep clone path as well. If your codebase uses any of these functions with data that could include user-controlled keys (even indirectly), you should use 4.17.21. The performance and API surface of 4.17.21 is identical to 4.17.20 except for the security fix. There is no reason to remain on 4.17.20.

---

**Q: Is semver 7.5.4 really still vulnerable? Our scanner says it's fine.**

A: Yes. Semver 7.5.4 tightened the input length check from 256 to 16 characters, reducing the practical impact of CVE-2022-25883, but it does NOT eliminate the backtracking-prone regex. A crafted input of 14 characters that triggers the maximum backtracking case can still cause measurable delays. For low-throughput applications, this may be acceptable. For high-throughput APIs (like MonoStack's API service), the delay is exploitable for denial-of-service. The security team's recommendation is 7.6.3, and that is the decision recorded in this RFC. If your scanner says 7.5.4 is fine, your scanner is using the NVD's initial affected range, which was based on the initial (insufficient) fix.

---

**Q: We're already on axios 1.5.x. Do we need to upgrade to 1.6.8?**

A: Yes. CVE-2023-45857 is not fixed in any 1.5.x release. The fix was introduced in 1.6.0. This is one of the key corrections made in Round 2 of this RFC — the initial proposal of 1.4.0 was wrong, and any version below 1.6.0 is vulnerable.

---

**Q: express 4.18.3 was in our codebase for months with no issues. Why is it a problem now?**

A: The express vulnerability (CVE-2024-29041) affects the `res.redirect()` function specifically. If your application uses `res.redirect()` with any destination URL derived from user input, the vulnerability allows an attacker to craft a redirect target that appears to be a relative path but is actually a redirect to an arbitrary external site (open redirect). Applications that never use `res.redirect()` with user-controlled input are not at direct risk, but security posture requires we patch regardless, because:

1. Future developers may add redirect functionality that uses the vulnerable pattern without knowing the version is vulnerable.
2. Security audits and compliance checks flag the vulnerable version regardless of current usage.
3. 4.19.2 has no breaking changes relative to 4.18.3 for correct code.

---

**Q: react 18.2.0 is the same as 18.3.1 for our purposes. Can we stay on 18.2.0?**

A: The root override pins react to 18.3.1 for TRANSITIVE consumers. @monostack/ui's direct react-dom dependency is separately tracked. The react 18.3.1 changes are primarily:
- Deprecation warnings for legacy lifecycle methods (warnings only, not errors)
- Minor security hardening in prop-types handling

There is no breaking change between 18.2.0 and 18.3.1 for code that uses modern React patterns. The upgrade is low-risk and the security hardening justifies it.

---

### K.3 Questions About the Override Mechanism

**Q: What exactly is the npm `overrides` field and how does it work?**

A: The `overrides` field was introduced in npm 8.3.0. When present in the root package.json of a project, it instructs npm to use the specified version for matching packages anywhere in the dependency tree, regardless of what version is requested by the consumer package.

For example, if package A depends on `lodash@^4.17.0` and the override specifies `"lodash": "4.17.21"`, npm will install 4.17.21 for A's lodash dependency (which satisfies `^4.17.0`) rather than resolving to whatever the latest matching version is.

The override is applied to the resolved version, not to the range. If a package declares `"lodash": "^3.0.0"` (incompatible with 4.17.21), npm will still apply the override but will generate a warning about peer dependency incompatibility. This situation does not occur in MonoStack — all consumers declare ranges that include the override versions.

**Q: What's the difference between `overrides` and `resolutions` (yarn)?**

A: They serve the same purpose but have different behaviors:

- npm `overrides`: Only applies to the dependency tree of the package where it is declared. Does not apply to direct dependencies of workspace packages (the limitation described throughout this RFC).

- yarn `resolutions`: More aggressive — applies even to direct dependencies. In yarn, `"react": "18.3.1"` in resolutions would force ALL react usage to 18.3.1, including direct dependencies.

npm's more conservative behavior is why react-dom had to be handled separately in this RFC — if we had used yarn's resolutions, adding `"react-dom": "18.3.1"` would have worked. With npm overrides, it doesn't affect direct workspace package dependencies.

**Q: Does the override apply to `devDependencies`?**

A: Yes. npm overrides apply to both `dependencies` and `devDependencies` of non-workspace packages. For workspace packages in the monorepo, the same limitation applies: direct dependencies (both deps and devDeps) in workspace package.json files are not overridden.

**Q: Can I specify an override with a range (e.g., `"lodash": "^4.17.21"`) instead of an exact version?**

A: Yes, ranges are valid in the overrides field. However, ranges allow npm to resolve to a higher version at install time, which means the lockfile may drift as new versions are published. For security overrides where we have a specific target version, exact version pinning (`"lodash": "4.17.21"`) is preferred. Ranges are more appropriate for forward-compatibility overrides.

---

### K.4 Implementation Verification Q&A

**Q: How do I confirm the overrides are actually in effect after running npm install?**

A: Use `npm ls [package]` to inspect the resolution tree:

```bash
npm ls lodash
```

The output should show all consumers of lodash resolved to 4.17.21. If any consumer shows a different version, the override did not take effect for that consumer (likely because it's a direct dependency of a workspace package).

You can also inspect the lockfile:

```bash
node -e "
const lock = require('./package-lock.json');
const pkg = 'lodash';
const entry = lock.packages['node_modules/' + pkg];
console.log(pkg + ':', entry ? entry.version : 'not found at root');
"
```

This shows the version installed at the root `node_modules/` level. Override-applied packages are deduplicated to the override version at the root level.

**Q: After applying overrides, `npm ls` shows warnings about "invalid" dependencies. Should I be concerned?**

A: This typically occurs when the override version is outside the range declared by a consumer. For example, if a package declares `"semver": "7.3.x"` and the override pins to 7.6.3, npm will install 7.6.3 but note that 7.6.3 does not satisfy `7.3.x`. The warning is informational and indicates that the consumer's declared range is stale, but since semver follows semantic versioning, 7.6.3 should be backward-compatible with code that works against 7.3.x.

If you see an excessively large number of warnings, review each one individually. Warnings about packages NOT covered by this RFC should be investigated separately — this RFC does not intend to introduce new compatibility issues.

**Q: The `npm install --prefer-offline` step in H.4 fails with "Cannot find module." What should I do?**

A: The `--prefer-offline` flag requires that the packages are already in the npm cache. If the packages are not cached, npm cannot find them offline. Solutions:

1. Remove the `--prefer-offline` flag and run `npm install` with network access.
2. Pre-populate the cache by running `npm install --dry-run` while connected to the network, then retry with `--prefer-offline`.
3. Verify that the target package versions (4.17.21, 7.6.3, 1.6.8, 4.19.2, 18.3.1) are available in the registry you are using.

The `--prefer-offline` flag was included in the implementation runbook for environments where internet access is restricted. It is not required if you have normal internet access.

---

## 25. Appendix L: Comparison with Alternative Remediation Approaches

During the RFC review process, several alternative approaches to the dependency vulnerabilities were proposed and evaluated. This appendix documents each alternative and the rationale for rejecting it in favor of the root overrides approach.

---

### L.1 Alternative: Direct Dependency Upgrades in Each Workspace Package

**Proposal:** Instead of adding root-level overrides, update the `dependencies` field in each workspace package to use the vulnerable packages as direct dependencies at the fixed version.

**Example:** Add `"lodash": "4.17.21"` to `packages/core/package.json`, `packages/api/package.json`, etc., to force each workspace package to use the patched version.

**Why rejected:**

1. The vulnerable packages are transitive dependencies, not direct dependencies (with the exception of lodash in core). Adding them as direct dependencies in workspace packages conflates the dependency graph and creates "phantom dependencies" — direct dependencies that the workspace package's code doesn't actually import.

2. This approach requires modifying every workspace package, even those that don't directly consume the vulnerable package. It creates maintenance overhead: if a new workspace package is added, the direct dependency pins must be manually added there too.

3. The root overrides approach is more maintainable and semantically correct. It expresses the intention ("force these specific transitive versions") in the right place (the workspace root) rather than scattered across workspace packages.

4. Direct dependency additions can mask real dependency management issues. If a workspace package upgrades its declared dependency on one of these packages to a new range in the future, the phantom direct dependency pin would create a confusing conflict.

---

### L.2 Alternative: Fork and Patch Vulnerable Packages

**Proposal:** Fork the vulnerable packages internally, apply the security patches manually, and publish them to the internal npm registry. Use the internal forks in place of the public versions.

**Why rejected:**

1. All five vulnerable packages have official releases that include the security patches. There is no need to maintain internal forks when official fixed versions are available.

2. Internal forks create a long-term maintenance burden: the fork must be kept synchronized with the upstream as new security issues are discovered and patched.

3. Using internal forks prevents automatic CVE scanning from working correctly — the scanner would see our internal fork version and might not be able to map it back to the public CVE database.

4. This approach would have been appropriate only if the upstream maintainers were unresponsive or had declined to fix the vulnerabilities. That is not the case for any of the five packages in this RFC.

---

### L.3 Alternative: `npm audit fix --force`

**Proposal:** Run `npm audit fix --force` to automatically upgrade all vulnerable packages to their latest versions, rather than specifying exact target versions.

**Why rejected:**

1. `npm audit fix --force` performs major version upgrades when necessary, which can introduce breaking changes. For example, it might upgrade express to 5.x, which has breaking API changes.

2. The `--force` flag bypasses semver compatibility checks. This is a blunt instrument that can break the application in unpredictable ways.

3. As discussed in K.1, npm's CVE database may have incorrect fix version information (as was the case with axios). Running automated fixes based on this data can install a version that doesn't actually fix the CVE.

4. The manual review process that produced this RFC identified two significant errors in the initial automated proposals (axios 1.4.0, express 4.18.3). Automated fixes would have made the same errors.

---

### L.4 Alternative: Upgrade All Dependencies to Latest

**Proposal:** Rather than targeting specific CVE fixes, upgrade all dependencies across all workspace packages to their latest versions. This would implicitly fix the vulnerabilities and also capture all other improvements.

**Why rejected:**

1. A broad "upgrade everything" approach has unbounded blast radius. Any one of dozens of dependencies could have a breaking change that requires code modifications.

2. This RFC's scope is security remediation, not a general dependency upgrade. Mixing security fixes with feature upgrades makes it harder to audit which change fixed which issue.

3. The Platform Team does not have the capacity to validate all dependencies simultaneously. A targeted approach (five packages, well-understood impact) is tractable. A broad upgrade is not.

4. Some dependencies may be intentionally pinned to specific versions for compatibility reasons that are not documented in the package.json. A broad upgrade could override these intentional pins.

---

### L.5 Alternative: Containerization and Isolation

**Proposal:** Deploy each workspace service in its own container and limit network access between services, reducing the exploitability of the vulnerabilities even if the vulnerable packages remain.

**Why rejected:**

1. The vulnerabilities addressed by this RFC include prototype pollution (lodash) and SSRF vectors (axios). Containerization does not mitigate prototype pollution — it's a memory-level attack that affects the Node.js process regardless of container boundaries.

2. Network isolation can reduce the impact of SSRF vulnerabilities, but it does not eliminate them. An attacker who achieves SSRF can still make requests to internal services within the container network.

3. The containerization proposal is a defense-in-depth measure that is already planned as part of the platform's security roadmap. It complements but does not replace the code-level fix.

4. Container security is outside the scope of this RFC. Mixing dependency management decisions with infrastructure decisions would make the RFC's scope unmanageable.

---

### L.6 Summary: Why Root Overrides

The root `overrides` approach was chosen because it:

1. Is the semantically correct npm mechanism for forcing transitive dependency versions
2. Requires changes in exactly one file (root package.json) rather than being distributed across workspace packages
3. Has predictable, well-documented behavior as of npm 8.3.0
4. Does not require internal forks, infrastructure changes, or broad upgrades
5. Can be precisely targeted to the five CVEs identified in this RFC
6. Is reversible with a single file edit and lockfile regeneration

---

---

## 26. Appendix M: Package Usage Audit — Code Reference Survey

This appendix documents all identified call sites for the five overridden packages within the MonoStack codebase as of January 2024. The survey was conducted by the Platform Team using static analysis and manual code review. Its purpose is to confirm that the behavioral changes introduced by the override target versions do not impact any existing functionality.

---

### M.1 lodash Usage Survey

A grep of the repository for `require('lodash')` and `import _ from 'lodash'` and `import { ... } from 'lodash'` identified the following call sites.

**packages/core/src/config/merger.js** — 12 lodash call sites

```
_.merge() — 3 instances (config object merging, lines 47, 89, 134)
_.cloneDeep() — 2 instances (snapshot creation, lines 61, 142)
_.get() — 4 instances (safe nested property access, lines 73, 81, 95, 156)
_.set() — 2 instances (nested property assignment, lines 99, 163)
_.omit() — 1 instance (remove sensitive fields from logs, line 178)
```

**Risk assessment for 4.17.21:** The `_.merge()` calls on lines 47, 89, and 134 accept objects from the configuration loading pipeline. In the current implementation, config is loaded from environment variables and YAML files — not from HTTP request bodies. The prototype pollution attack vector requires user-controlled keys, which are not present in the config loading path. However, the fix in 4.17.21 protects against future code paths that might add user-controlled config sources. The `_.set()` calls similarly accept config-derived values, not user input.

**Conclusion:** All 12 call sites are safe under 4.17.21. The behavioral change (prototype pollution prevention) is strictly additive security without observable functional impact.

---

**packages/core/src/utils/transform.js** — 7 lodash call sites

```
_.groupBy() — 2 instances (report aggregation, lines 23, 67)
_.sortBy() — 1 instance (result ordering, line 44)
_.uniqBy() — 2 instances (deduplication, lines 56, 89)
_.flatMap() — 1 instance (list flattening, line 78)
_.pick() — 1 instance (field selection, line 102)
```

**Risk assessment for 4.17.21:** None of these functions are affected by the prototype pollution fix. `groupBy`, `sortBy`, `uniqBy`, `flatMap`, and `pick` operate on arrays and do not traverse `__proto__` paths. The upgrade from 4.17.20 to 4.17.21 has no effect on these call sites.

---

**packages/core/src/reporting/aggregator.js** — 5 lodash call sites

```
_.merge() — 1 instance (merging partial report segments, line 34)
_.isEqual() — 2 instances (change detection, lines 52, 88)
_.difference() — 1 instance (computing added/removed items, line 67)
_.intersection() — 1 instance (computing common items, line 75)
```

**Risk assessment for 4.17.21:** The `_.merge()` on line 34 merges report segment objects that are constructed internally from database query results. No user-controlled keys are involved. The other functions are analytical utilities with no security relevance.

---

**packages/api/src/middleware/body-transformer.js** — 3 lodash call sites

```
_.merge() — 2 instances (merging request defaults with body, lines 28, 45)
_.pick() — 1 instance (filtering allowed fields, line 62)
```

**Risk assessment for 4.17.21 (ELEVATED):** These are the highest-risk call sites in the codebase. The `_.merge()` on line 28 merges `req.body` (user-controlled) into a defaults object. This is the attack vector described in §D.1.2 — a crafted request body of `{"__proto__": {"isAdmin": true}}` would exploit this call site in lodash < 4.17.21.

The fix in 4.17.21 prevents the `__proto__` path from being traversed during the merge. After the upgrade, the crafted request body would be silently ignored rather than polluting the prototype.

**This call site is the primary remediation target for CVE-2021-23337 in the MonoStack codebase.**

---

### M.2 semver Usage Survey

A grep for `require('semver')` and `import semver from 'semver'` identified the following call sites.

**packages/cli/src/commands/update.js** — 4 semver call sites

```
semver.satisfies() — 2 instances (checking if installed version satisfies requirement, lines 34, 78)
semver.gt() — 1 instance (comparing versions to determine upgrade direction, line 56)
semver.parse() — 1 instance (parsing version string from registry response, line 89)
```

**Risk assessment for 7.6.3:** The `semver.parse()` on line 89 parses a version string returned from the npm registry API. In theory, a malicious npm registry could return a crafted version string to trigger the ReDoS vulnerability. However:

1. The MonoStack CLI connects to the official npm registry, which does not return malicious version strings.
2. Even if a crafted string were returned, the ReDoS vulnerability in semver < 7.6.3 would only cause a delay in the CLI, not a security breach.

Nevertheless, the upgrade to 7.6.3 eliminates this theoretical exposure.

The `semver.satisfies()` calls accept version strings from internal configuration, not user input. No risk.

---

**packages/api/src/routes/compatibility.js** — 2 semver call sites

```
semver.satisfies() — 2 instances (API version compatibility checking, lines 12, 34)
```

**Risk assessment for 7.6.3:** The version strings passed to `semver.satisfies()` are derived from the `X-API-Version` HTTP header, which is user-controlled. This is a potential ReDoS attack vector: an attacker could send a crafted `X-API-Version` header with a long malicious string to trigger backtracking in semver < 7.6.3.

At the measured throughput of the MonoStack API service (5000 semver validations per minute at peak), the DoS impact for a coordinated attack using 7.5.x would be significant. The upgrade to 7.6.3 is strongly justified by this call site.

**This call site is the primary remediation target for CVE-2022-25883 in the MonoStack codebase.**

---

### M.3 axios Usage Survey

axios is used as a transitive dependency through `metrics-client@2.3.0`. It is not directly imported in any MonoStack source file as of the audit date.

**node_modules/metrics-client/src/reporter.js** (third-party, not audited for call sites)

The metrics-client library uses axios for HTTP POST requests to the monitoring endpoint at `https://metrics.internal.monostack.io/events`. This is an internal endpoint, same-origin relative to the services. The XSRF vulnerability (CVE-2023-45857) applies to cross-origin requests; requests to same-origin internal endpoints are not affected.

**Conclusion:** The axios upgrade from 1.3.4 to 1.6.8 has no observable functional impact on MonoStack's behavior. The XSRF behavior change affects cross-origin requests, and the only axios usage is for same-origin internal API calls.

---

### M.4 express Usage Survey

express is used as a direct dependency in `packages/api`, but the vulnerability is in the redirect URL handling which is also exercised by transitive consumers.

**packages/api/src/app.js** — express application setup

```
app.use(express.json()) — line 12
app.use(express.urlencoded()) — line 13
app.use(express.static()) — line 18
```

None of these use `res.redirect()`. No vulnerability exposure from app.js.

**packages/api/src/routes/auth.js** — 2 res.redirect() call sites

```
res.redirect('/login') — line 45 (static relative path, not user-controlled)
res.redirect(req.query.next || '/dashboard') — line 78 (POTENTIALLY VULNERABLE)
```

**Risk assessment for 4.19.2 (ELEVATED):** Line 78 uses `req.query.next` as the redirect destination if it is present, falling back to `/dashboard`. An attacker can craft a request with `?next=//attacker.com/phishing` to cause an open redirect to `//attacker.com/phishing` in express < 4.19.0.

The fix in 4.19.2 validates the redirect target URL and rejects protocol-relative URLs (`//attacker.com/...`). After the upgrade, `req.query.next` values that are not relative paths or absolute URLs with the expected host are rejected.

**This call site is the primary remediation target for CVE-2024-29041 in the MonoStack codebase.**

Additionally, a whitelist of allowed next-redirect destinations was added to auth.js as a defense-in-depth measure alongside the express upgrade:

```javascript
const ALLOWED_NEXT_PATHS = /^\/[a-zA-Z0-9\-_/]*$/;
const nextPath = req.query.next;
if (nextPath && ALLOWED_NEXT_PATHS.test(nextPath)) {
  res.redirect(nextPath);
} else {
  res.redirect('/dashboard');
}
```

---

### M.5 react Usage Survey

react is used directly in `packages/ui` and pulled in transitively by testing libraries in `packages/core` and `packages/api`.

**packages/ui/src/components/** — extensive direct react usage

The UI package contains 47 React components. All use functional components with hooks. No class components. No deprecated lifecycle methods (`componentWillMount`, `componentWillReceiveProps`, `componentWillUpdate`) are used in any MonoStack-authored component.

React 18.3.1's deprecation warnings will not fire for any MonoStack UI code.

**node_modules/@testing-library/react** — indirect react usage

The testing library's internals use `ReactDOM.render` for compatibility with legacy test patterns. This generates deprecation warnings when running the test suite but does not affect the production application. Mitigated with the jest setup configuration described in §I.5.

**Conclusion:** React upgrade from 18.2.0 to 18.3.1 has no functional impact on MonoStack UI code. Deprecation warnings are suppressed for known third-party library warnings.

---

### M.6 Audit Summary

| Package | Highest-Risk Call Site | Impact Without Upgrade | Impact After Upgrade |
|---|---|---|---|
| lodash | packages/api/src/middleware/body-transformer.js line 28 | Prototype pollution via req.body | Crafted __proto__ keys silently ignored |
| semver | packages/api/src/routes/compatibility.js line 12 | ReDoS via X-API-Version header | ReDoS vector eliminated (no backtracking) |
| axios | metrics-client/src/reporter.js (transitive) | None identified (internal requests only) | No change to observable behavior |
| express | packages/api/src/routes/auth.js line 78 | Open redirect via ?next= query param | Protocol-relative redirects rejected |
| react | None (no deprecated APIs used) | N/A | Deprecation warnings from testing-library suppressed |

---

## 27. Appendix N: RFC Process Retrospective

This appendix documents the retrospective discussion held by the Security Guild after RFC-2024-047 was finalized. It captures what worked well, what didn't, and what changes were made to the RFC process for future iterations.

---

### N.1 What Worked Well

**Multi-round review caught two significant errors.**

The initial proposals for axios (1.4.0) and express (4.18.3) were wrong. Both were caught during the Security Guild's Round 2 discussion before the RFC was ratified. In a single-round review process, both incorrect versions would have been implemented and the CVEs would have remained unmitigated.

Marcus: "The axis 1.4.0 mistake is the one that bothers me most in retrospect. We spent a week implementing the Round 1 proposal in a test branch before Danielle flagged the NVD error. If we had merged that to main, we would have had a false sense of security — the scanner would have shown 'no vulnerabilities' but the actual XSRF exploit would still have worked."

**Keiko's deep-dive on semver was essential.**

The distinction between semver 7.5.4 and 7.6.3 would not have been obvious without Keiko's analysis of the regex internals. The initial consensus was that 7.5.4 was sufficient. The additional research changed that conclusion.

Keiko: "I was initially planning to just accept 7.5.4 because the CVE says 'fixed in 7.5.2' and 7.5.4 is later. But I got curious about why there was a 7.6.x line at all and discovered the regex rewrite. That kind of follow-up research should be standard practice."

**The ARB's independent security review added credibility.**

Desmond's independent reproduction of each vulnerability confirmed that the RFC's version choices were correct before ratification. This also caught a discrepancy in the NVD's affected range for CVE-2023-45857.

---

### N.2 What Didn't Work Well

**The initial proposals were made without CVE verification.**

Both the axios and express Round 1 proposals were based on incomplete information. The RFC process should require CVE verification before any version is proposed, not just before ratification.

**Action taken:** The RFC template was updated to require the following fields for each proposed version:
- CVE identifier
- NVD URL
- Package changelog URL pointing to the fix commit
- Confirmation that the proposer tested the fix against the exploit

**The process took too long.**

From initial CVE triage to ARB ratification was 18 days (January 8 to January 26). For a high-severity CVE, this is too slow. The multi-round review is valuable, but the turnaround time between rounds needs to be accelerated.

**Action taken:** Introduced 48-hour deadlines for each round of review. Extended discussion should move to synchronous channels (Slack calls or in-person meetings) rather than async threads.

**Documentation was incomplete in early rounds.**

The Round 1 RFC document had insufficient context for why each version was chosen. Reviewers had to ask clarifying questions that should have been answered in the document itself.

**Action taken:** The RFC template now requires a "Rationale" section for each proposed version explaining: (a) the CVE it fixes, (b) why this version and not an earlier one, (c) why this version and not a later one.

---

### N.3 Process Changes for Future RFCs

Based on the retrospective, the following changes were made to the dependency security RFC process:

**N.3.1 CVE Verification Gate**

Before a version can be included in an RFC draft, the proposer must:

1. Locate the CVE entry on NVD (nvd.nist.gov) and confirm the affected and fixed version ranges.
2. Locate the fix commit in the package's GitHub repository.
3. Confirm that the proposed version is at or after the fix commit.
4. Confirm that the proposed version is the latest in the patched minor series (not just the minimum fixed version).

If the NVD entry is ambiguous or appears incorrect (as was the case with axios), the proposer must reference the package's GitHub security advisories and the fix PR as the authoritative source.

**N.3.2 Mandatory Testing**

The Security Guild's security lead (currently Desmond) must independently reproduce each vulnerability against the proposed fix version before the RFC proceeds to the ARB.

This means:
- Writing a proof-of-concept exploit for each CVE
- Confirming the exploit works against the vulnerable version
- Confirming the exploit does not work against the proposed fix version
- Documenting the test in the RFC (examples in §D.1.3, §D.2.x, etc.)

**N.3.3 Code Audit Requirement**

The RFC must include a code audit identifying the highest-risk call sites for each vulnerability (see Appendix M for the format). This ensures that the team understands the real-world impact and confirms that the fix version's behavioral changes are compatible with existing usage.

**N.3.4 Rollback Plan**

Every RFC must include a rollback procedure (see §H.10 for the format). The rollback plan must be tested in a staging environment before the RFC is implemented in production.

---

### N.4 Metrics

The following metrics were collected to evaluate the RFC-2024-047 process:

| Metric | Value |
|---|---|
| Duration (initial CVE triage to ARB ratification) | 18 days |
| Number of review rounds | 3 |
| Number of version corrections from Round 1 to final | 3 (axios, express, semver) |
| Number of CVEs remediated | 5 |
| Number of workspace packages affected | 4 (all) |
| Number of transitive dependency consumers updated | 28 (across all 5 packages) |
| Number of test changes required | 5 |
| Number of code changes required beyond package.json | 2 (auth.js redirect whitelist, jest setup suppression) |
| Post-implementation incidents related to upgrade | 0 |
| Time from implementation to CVE scanner clearance | 3 days (scanner update lag) |

---

### N.5 Final Reflections

Marcus closed the retrospective with the following notes:

"The most important takeaway is that the multi-round process, while slower than a single-round review, caught mistakes that would have resulted in a false sense of security. The axios situation in particular is a cautionary tale: an automated tool gave us wrong information, we almost accepted it, and only a careful secondary review caught it.

"The second takeaway is that version selection is not trivial. 'Minimum fixed version' is not the same as 'recommended version.' The semver case showed that even after a CVE is 'fixed,' subsequent releases in the same fix series can provide better protection. Our policy of targeting the latest in the patched series, rather than the minimum fix, is the right policy.

"The third takeaway is that npm's override mechanism has real limitations that every developer needs to understand. The react-dom situation caught several people by surprise. We should add a section on npm override limitations to the developer onboarding documentation.

"Finally: document rejections. The most valuable part of this RFC for future readers is not the final decisions table in §12 — it's the detailed explanations of why axios 1.4.0 and express 4.18.3 were rejected. Future developers (and automated tooling that reads codebases) need to understand not just what the correct answer is, but why the wrong answers are wrong."

---

---

## 28. Appendix O: Compliance Documentation and Audit Trail

This appendix provides the formal compliance documentation required by the organization's security policy for CVE remediation efforts. It documents the timeline, responsible parties, and evidence of remediation for each CVE addressed by RFC-2024-047.

---

### O.1 CVE Remediation Timeline

The following table documents the complete timeline from discovery to remediation confirmation for each CVE addressed by this RFC.

**CVE-2021-23337 (lodash prototype pollution)**

| Date | Event | Responsible Party |
|---|---|---|
| 2021-12-29 | CVE published; lodash 4.17.21 released as fix | lodash maintainers |
| 2022-01-15 | Internal scanner first flags lodash < 4.17.21 | Automated (Dependabot) |
| 2022-01-20 | Initial Dependabot PRs opened on MonoStack; deferred due to workload | Engineering Team |
| 2023-06-01 | CVE added to security backlog for Q3 remediation | Security Team |
| 2023-10-01 | Remediation reprioritized to Q1 2024 due to competing priorities | Marcus, VP Engineering |
| 2024-01-08 | Security Guild initiates RFC-2024-047 | Marcus |
| 2024-01-26 | ARB ratifies RFC | ARB (Valentina, Desmond, Fatima) |
| 2024-02-05 | Override applied to production monorepo main branch | Marcus, Priya |
| 2024-02-08 | CVE scanner confirms remediation | Automated (Dependabot) |

**CVE-2022-25883 (semver ReDoS)**

| Date | Event | Responsible Party |
|---|---|---|
| 2022-07-01 | CVE published | semver maintainers |
| 2022-07-15 | Internal scanner flags semver < 7.5.2 | Automated (Dependabot) |
| 2022-08-01 | Dependabot PR opened; merged (upgraded to 7.5.2 for direct deps, but transitive deps not fixed) | Engineering Team |
| 2023-08-01 | Security audit reveals transitive semver instances still at 7.3.x | Desmond |
| 2024-01-08 | Added to RFC-2024-047 scope | Marcus |
| 2024-01-10 | Keiko's analysis establishes 7.6.3 as target (7.5.x insufficient) | Keiko |
| 2024-01-26 | ARB ratifies RFC with 7.6.3 target | ARB |
| 2024-02-05 | Override applied | Marcus, Priya |
| 2024-02-08 | CVE scanner confirms remediation | Automated (Dependabot) |

**CVE-2023-45857 (axios XSRF)**

| Date | Event | Responsible Party |
|---|---|---|
| 2023-10-26 | axios 1.6.0 released with fix | axios maintainers |
| 2023-11-01 | CVE published; NVD initially lists fix as 1.4.0 (incorrect) | NVD |
| 2023-11-15 | Internal scanner flags axios < 1.4.0 (incorrect threshold from NVD) | Automated (Dependabot) |
| 2023-12-01 | Dependabot PR merges axios to 1.4.0 for direct usages — does NOT fix CVE | Engineering Team |
| 2023-12-20 | Transitive axios instances remain at 1.3.4; not covered by PR | — |
| 2024-01-08 | Added to RFC-2024-047 scope with initial target of 1.4.0 | Marcus |
| 2024-01-11 | Danielle identifies that 1.4.0 does not fix CVE; target revised to 1.6.8 | Danielle |
| 2024-01-26 | ARB ratifies RFC with 1.6.8 target; Desmond confirms NVD entry was incorrect | ARB |
| 2024-02-05 | Override applied | Marcus, Priya |
| 2024-02-08 | CVE scanner confirms remediation (scanner updated NVD data) | Automated |

**CVE-2024-29041 (express open redirect)**

| Date | Event | Responsible Party |
|---|---|---|
| 2024-02-09 | express 4.19.0 released with fix | express maintainers |
| 2024-02-14 | CVE published | express maintainers / NVD |
| 2024-02-20 | Internal scanner flags express < 4.19.0 | Automated (Dependabot) |
| — | Note: RFC-2024-047 was in late stages when this CVE published; added retroactively to scope | Marcus |
| 2024-01-12 | Security Guild discusses express 4.18.3 (incorrect Round 1 target) before CVE official publication | — |
| 2024-01-26 | ARB ratifies RFC with 4.19.2 target | ARB |
| 2024-02-05 | Override applied | Marcus, Priya |
| 2024-02-10 | CVE scanner confirms remediation | Automated |

**React security hardening (no CVE assigned)**

| Date | Event | Responsible Party |
|---|---|---|
| 2024-04-25 | react 18.3.1 released with security hardening | React team |
| 2024-05-01 | Platform team reviews release notes | Fatima |
| 2024-05-03 | Added to RFC-2024-047 as non-CVE security improvement | Marcus |
| 2024-05-10 | Override applied to monorepo | Marcus |
| 2024-05-10 | No scanner entry (no assigned CVE) | — |

---

### O.2 Risk Acceptance Documentation

During the remediation process, two items were formally accepted as residual risk. These are documented here for audit purposes.

**O.2.1 react-dom Direct Dependency (Accepted Risk)**

The react-dom package in @monostack/ui is a direct dependency at version 18.2.0. It could not be updated via the root overrides mechanism (see §12.3). The update requires a direct package.json change in packages/ui, which will be tracked separately.

**Risk assessment:** react-dom 18.2.0 has no known CVEs as of the time of writing. The 18.3.1 update is a security hardening measure, not a CVE fix. The risk of remaining on 18.2.0 is low and accepted pending the separate ticket resolution.

**Accepted by:** Valentina (ARB Chair), 2024-01-26
**Review date:** 2024-07-01 (quarterly review)

**O.2.2 Historical Incorrect Dependabot Merge (Accepted Risk)**

In December 2023, a Dependabot PR merged axios to 1.4.0 for direct usages based on the NVD's incorrect affected range. This PR was merged to main before the error was discovered. The incorrect 1.4.0 versions in direct dependencies were superseded by the 1.6.8 override.

**Risk assessment:** The window during which the repo had axios 1.4.0 as a direct dependency (December 2023 through February 2024) represents a period where CVE-2023-45857 was not mitigated, despite the team believing it was. No incidents were reported during this window. The override to 1.6.8 fully mitigates the CVE.

**Accepted by:** Desmond (ARB Security Lead), 2024-01-26
**Lessons learned:** Documented in §N.3.1 (CVE verification gate requirement)

---

### O.3 Evidence of Remediation

The following evidence is available in the repository and CI system to confirm that RFC-2024-047's remediations are in effect.

**package.json overrides field:**

```json
{
  "overrides": {
    "lodash": "4.17.21",
    "semver": "7.6.3",
    "axios": "1.6.8",
    "express": "4.19.2",
    "react": "18.3.1"
  }
}
```

Verified present in: `/app/package.json`

**package-lock.json verification:**

The lockfile's `packages` object should contain no entries for the overridden packages at versions below the override targets. This is verified by the automated test suite (see `tests/test_outputs.py`).

**Dependabot alert status:**

All five CVE alerts should show as "Dismissed — Fixed" or "Closed" in the repository's security alerts dashboard.

**CI pipeline:**

The CI pipeline runs `npm audit --audit-level=high` as a required check. This check fails if any high-severity CVE is detected. The pipeline has been green since February 8, 2024.

---

### O.4 Regulatory Compliance Notes

Several regulatory frameworks and security standards require documented CVE remediation processes. This section maps RFC-2024-047 to the relevant requirements.

**SOC 2 Type II — CC7.1 (Change Management)**

RFC-2024-047 satisfies the change management requirement by providing:
- Documentation of the change and its rationale
- Multi-party review (Security Guild + ARB)
- Testing evidence (CI pipeline results)
- Rollback procedure (§H.10)

**SOC 2 Type II — CC9.1 (Risk Assessment)**

The code audit (Appendix M) and the risk assessments for each override target satisfy the requirement for documented risk assessment of production changes.

**NIST SP 800-40 (Guide to Enterprise Patch Management)**

RFC-2024-047 aligns with NIST's guidance on vulnerability remediation:
- Patch criticality assessment: performed for each CVE (see §D)
- Patch testing: performed in staging environment before production (§F.2, Fatima's CI verification)
- Deployment timeline: documented in §O.1
- Rollback capability: documented in §H.10

**PCI DSS 6.3.3 (All system components are protected from known vulnerabilities)**

The five CVEs addressed by this RFC represent known vulnerabilities in system components. RFC-2024-047 documents the remediation of all five, satisfying the PCI DSS requirement for patching known vulnerabilities.

---

### O.5 Artifact Registry

The following artifacts are associated with RFC-2024-047 and must be retained per the organization's document retention policy (7 years):

| Artifact | Location | Retention Period |
|---|---|---|
| This RFC document | `/app/docs/dependency-migration-rfc.md` | 7 years |
| ARB meeting notes | Confluence: Platform Engineering > ARB Sessions > 2024-01-18 | 7 years |
| ARB ratification vote record | Confluence: Platform Engineering > ARB Sessions > 2024-01-18 | 7 years |
| Security Guild Slack thread archive | This RFC, Appendix E | 7 years |
| Pre-implementation dependency tree baseline | `/tmp/baseline_*.txt` (snapshot in Confluence) | 7 years |
| Post-implementation CI pipeline run | CI System: build #4821 (2024-02-05) | 7 years |
| CVE scanner clearance screenshots | Confluence: Platform Engineering > CVE Remediation > RFC-2024-047 | 7 years |

---

---

## 29. Appendix P: Peer Review Comments and Responses

This appendix preserves the formal written review comments received during the RFC review rounds, along with the RFC authors' responses. Written comments were submitted through the internal RFC review system in addition to the Slack discussions archived in Appendix E.

---

### P.1 Round 1 Written Review Comments

**Reviewer: Tobias Reinholt (Senior Engineer, @monostack/ui)**
**Submitted: 2024-01-09**

*Comment 1:*
> Section 4.3 proposes axios 1.4.0. I reviewed the NVD entry and see that it lists "< 1.4.0" as the affected range, implying 1.4.0 is the fix. However, I also checked the axios changelog at that version and do not see any reference to XSRF token handling. Before we finalize this version, can someone confirm there is a specific commit in the axios repository that addresses the XSRF behavior in 1.4.0?

*Author response (Marcus, 2024-01-10):*
> Tobias, thank you for flagging this. I checked the 1.4.0 release notes and you are correct — there is no mention of XSRF in that release. I am escalating to Danielle who has been tracking this CVE more closely. This may result in a revision to the axios target version.

*Disposition:* This comment directly led to the Round 2 correction. The axios target was changed to 1.6.8 after Danielle's analysis confirmed that 1.4.0 does not fix CVE-2023-45857.

---

*Comment 2:*
> Section 4.4 proposes express 4.18.3. The internal security scanner flagged this but Jordan's write-up does not include a CVE number. Policy requires a CVE number for all security remediations. Can this be added, or is this proposal premature?

*Author response (Marcus, 2024-01-10):*
> Tobias, you are correct. Jordan is investigating the CVE number. We will either locate the correct CVE and update the proposal, or remove express from Round 1 scope and revisit when CVE documentation is available.

*Disposition:* Jordan subsequently identified CVE-2024-29041 and the target was corrected to 4.19.2 in Round 2.

---

**Reviewer: Fatima Okonkwo (Platform Lead, ARB)**
**Submitted: 2024-01-09**

*Comment 3:*
> I appreciate the detail in the RFC. Before I can approve this for ARB review, I need the following items:
> 1. A code audit showing which MonoStack call sites are affected by each CVE. Without this, I cannot assess whether the behavioral changes in each fix version are safe for production.
> 2. A post-implementation test plan. What tests will we run to confirm the overrides took effect and nothing broke?
> 3. A rollback procedure. If the overrides introduce a regression, how do we revert safely?

*Author response (Marcus, 2024-01-12):*
> Fatima, all three items will be added in Round 2. The code audit is underway. The test plan will reference the automated test suite plus manual smoke tests. The rollback procedure will document the git revert + npm ci approach.

*Disposition:* All three items were addressed in the RFC updates. The code audit is Appendix M. The test plan is §H.6 and §H.7. The rollback procedure is §H.10.

---

**Reviewer: Reuben Castillo (Senior Engineer, @monostack/api)**
**Submitted: 2024-01-10**

*Comment 4:*
> The RFC mentions that axios 1.6.x changed XSRF token behavior. Our API service uses axios internally via the metrics client, and I want to make sure this change does not break anything. Specifically: does `metrics-client` make cross-origin requests? If so, the XSRF behavior change could break its requests to the monitoring endpoint.

*Author response (Priya, 2024-01-11):*
> Reuben, I checked the metrics-client source code. The monitoring endpoint is at `metrics.internal.monostack.io`, which is a same-origin internal service. The XSRF token change only affects cross-origin requests. Same-origin requests continue to work exactly as before. No impact to metrics-client.

*Disposition:* Verified in §I.3. No code changes required for metrics-client.

---

*Comment 5:*
> Section 6.2 lists the Round 1 final overrides table. I notice `react-dom` is not included. Is this intentional? The react and react-dom packages are always versioned together.

*Author response (Marcus, 2024-01-12):*
> Reuben, yes, this is intentional. react-dom is a direct dependency of @monostack/ui, not a transitive dependency. npm workspace overrides do not apply to direct workspace package dependencies. Adding react-dom to the overrides block would have no effect on @monostack/ui's react-dom version. The update to react-dom will be done separately via a direct change to packages/ui/package.json. We have added §12.3 to the RFC explicitly documenting this decision.

*Disposition:* §12.3 added. Appendix E.5 documents the full discussion. The test suite includes `test_react_dom_not_overridden` to prevent future implementers from incorrectly adding react-dom.

---

### P.2 Round 2 Written Review Comments

**Reviewer: Keiko Watanabe (Security Engineer)**
**Submitted: 2024-01-15**

*Comment 6:*
> The Round 2 revision correctly updates axios to 1.6.8. I have one remaining concern about the semver target. The RFC now specifies 7.6.3, which I agree is correct. However, the RFC's rationale section for semver says "7.5.4 was superseded by a more comprehensive fix in 7.6.x." This understates the issue. 7.5.4 is not just "superseded" — it is still measurably vulnerable to inputs under 16 characters. I tested this. The language should clearly say that 7.5.4 does not fully fix CVE-2022-25883 and should be treated as if it does not fix the CVE at all for high-throughput environments.

*Author response (Marcus, 2024-01-15):*
> Keiko, you are right that the language was too soft. I have updated §8.2 to say explicitly: "semver 7.5.4 is insufficient for high-throughput deployments. It reduces the severity of CVE-2022-25883 but does not eliminate the underlying vulnerability. For MonoStack's API service, which validates semver strings from user-supplied headers at high throughput, 7.5.4 must be treated as not fixing the CVE." Additionally, §8.2 now references your timing test data as evidence.

*Disposition:* RFC text updated. The tests enforce 7.6.3 (not 7.5.4) via `test_semver_superseded_version_not_used`, which fails if the override is set to 7.5.4.

---

**Reviewer: Desmond Achebe (ARB Security Lead)**
**Submitted: 2024-01-16**

*Comment 7:*
> I have completed my independent verification of the proposed versions. My results:
>
> - lodash 4.17.21: Verified fix for CVE-2021-23337. The `zipObjectDeep` exploit does not work against 4.17.21. Confirmed.
> - semver 7.6.3: Verified fix for CVE-2022-25883. The 200-character ReDoS payload returns in 0ms. Confirmed.
> - axios 1.6.8: Verified fix for CVE-2023-45857. XSRF token not sent to cross-origin URL with crafted `xsrfCookieName` config. Confirmed.
> - express 4.19.2: Verified fix for CVE-2024-29041. `///attacker.com/path` redirect target correctly rejected. Confirmed.
> - react 18.3.1: Security hardening. No CVE to verify.
>
> One additional note: I found that the NVD entry for CVE-2023-45857 still lists the affected range as < 1.4.0. This is incorrect. The NVD entry has not been updated to reflect the actual fix version (1.6.0). I am filing a NVD correction request. The RFC should note this discrepancy explicitly so that teams using automated NVD-based tools do not falsely conclude they are safe on 1.4.x.

*Author response (Marcus, 2024-01-17):*
> Desmond, thank you for the thorough verification and for filing the NVD correction. I have added a note to §8.1 (and the axios sections throughout) that explicitly states: "The NVD entry for CVE-2023-45857 lists the affected range as < 1.4.0. This entry is incorrect. The actual fix is in axios 1.6.0. Teams relying on NVD data or automated scanners may falsely believe that axios 1.4.0 is the remediated version. This RFC explicitly rejects 1.4.0 as insufficient."

*Disposition:* RFC updated. This discrepancy is now documented in every axios-related section to prevent future confusion.

---

### P.3 Round 3 Written Review Comments

**Reviewer: Valentina Cruz (ARB Chair)**
**Submitted: 2024-01-24**

*Comment 8:*
> The RFC is now comprehensive and I am prepared to support ratification. My only request before the ARB vote: please confirm in writing from the Security Guild that the code audit in the draft appendix accurately reflects the current codebase and that no call sites were missed. A missed call site in the audit (particularly for lodash or express) could mean we have a vulnerable pattern that we think is protected but isn't.

*Author response (Marcus, 2024-01-24):*
> Valentina, I can confirm the following regarding the code audit completeness:
>
> 1. The lodash survey was performed by running `grep -r "require('lodash')\|from 'lodash'" --include="*.js" --include="*.ts" packages/`. All matches were inspected manually. The highest-risk call site (body-transformer.js line 28) was identified and independently confirmed by Priya.
>
> 2. The semver survey was performed by running `grep -r "require('semver')\|from 'semver'" --include="*.js" --include="*.ts" packages/`. The API route compatibility.js was identified as the highest-risk call site.
>
> 3. axios and express are not directly imported in MonoStack source files. They are used only through third-party packages (metrics-client for axios, and express itself in api). The express call sites were found by searching for `res.redirect` throughout the api package.
>
> 4. react is used through JSX and React hooks throughout the ui package. A component-by-component audit confirmed no deprecated lifecycle methods.
>
> I confirm to the best of my knowledge that the code audit is complete and accurate for the MonoStack codebase as of 2024-01-24.

*Disposition:* Confirmation accepted. ARB proceeding to ratification vote on 2024-01-26.

---

**Reviewer: Jordan Park (Security Engineer)**
**Submitted: 2024-01-23**

*Comment 9:*
> I want to formally acknowledge that the express 4.18.3 proposal in Round 1 was mine and it was incorrect. I based it on an internal scanner alert without CVE verification, which violates the process we should have been following. I have updated the runbook for scanner-triggered RFCs to require CVE verification before any version is included in a draft.
>
> For the record: express 4.18.3 does not fix CVE-2024-29041. It is a minor dependency update with no security relevance. The correct minimum fix version is 4.19.0, and the RFC correctly targets 4.19.2 as the latest in the patched series.
>
> I support ratification of the RFC as currently written.

*Author response (Marcus, 2024-01-23):*
> Jordan, thank you for the acknowledgment. The process improvement you've made to the scanner-triggered runbook is the right response. RFC-2024-047 has been updated to clearly document why 4.18.3 was rejected and to use your explanation as a teaching example in §N.3.1.

*Disposition:* Process improvement documented in §N.3.1. Jordan's formal acknowledgment preserved for audit purposes.

---

---

## 30. Appendix Q: Glossary of Terms

This glossary defines the technical and process-related terms used throughout RFC-2024-047.

**ARB (Architecture Review Board):** A cross-functional governance body responsible for reviewing and ratifying significant technical decisions, including security remediations and dependency management policies. The ARB consists of the platform lead, security lead, and ARB chair.

**Affected Range:** In a CVE entry, the version range of a package that is vulnerable. The affected range is typically expressed as "< [fix version]". As documented in this RFC, affected ranges in NVD entries are not always correct at the time of initial CVE publication.

**Backtracking:** In regular expression processing, the phenomenon where the regex engine re-attempts matches along alternative paths when an initial path fails. Catastrophic backtracking occurs when the regex engine must explore an exponentially large number of paths for a specific input, causing the engine to hang or run for an unacceptably long time. This is the mechanism behind ReDoS vulnerabilities.

**CVE (Common Vulnerabilities and Exposures):** A publicly disclosed cybersecurity vulnerability with a standardized identifier (e.g., CVE-2021-23337). CVEs are catalogued in the NVD.

**CVSS (Common Vulnerability Scoring System):** A framework for rating the severity of security vulnerabilities on a scale of 0.0 to 10.0. Scores above 7.0 are considered high severity. Scores above 9.0 are critical.

**CWE (Common Weakness Enumeration):** A categorization system for software security weaknesses. CVEs are often tagged with one or more CWEs describing the underlying weakness class.

**Direct Dependency:** A package explicitly listed in a project's `dependencies` or `devDependencies` field in package.json. Contrast with transitive dependency.

**Lockfile:** A file (e.g., package-lock.json, yarn.lock) that records the exact resolved version of every dependency in the tree, including transitive dependencies, at the time `npm install` or equivalent was run. Lockfiles ensure reproducible installs.

**Monorepo:** A software development strategy where multiple packages or projects are stored in a single repository. The MonoStack repository is a monorepo containing four workspace packages.

**NVD (National Vulnerability Database):** The U.S. government repository of standards-based vulnerability management data. Maintained by NIST. The primary source for CVE information, including affected version ranges and severity scores.

**npm overrides:** A field in the root package.json of an npm project that forces specific versions of packages throughout the dependency tree, overriding what individual packages request. Available in npm 8.3.0 and later. Does not override direct dependencies of workspace packages in npm 11.

**Open Redirect:** A web application vulnerability where an attacker can craft a URL that causes the application to redirect users to an arbitrary external URL. Exploited for phishing attacks by making a trusted domain's URL appear to redirect to the phishing site.

**Package-lock.json:** The npm lockfile format (versions 1, 2, or 3). Version 3 (lockfileVersion: 3) is the format used by npm 7 and later and required by this RFC.

**Peer Dependency:** A dependency that a package expects to share with the consuming project, rather than installing its own copy. For example, a React component library declares React as a peer dependency, expecting the consuming application to provide React.

**Prototype Pollution:** A JavaScript vulnerability where an attacker can add or modify properties on `Object.prototype`, the base object that all JavaScript objects inherit from. Because prototype changes propagate to all objects in the process, this can cause widespread unexpected behavior or enable privilege escalation.

**ReDoS (Regular Expression Denial of Service):** A denial-of-service attack that exploits catastrophic backtracking in regex engines by providing specially crafted input strings. The attack can cause the application to consume 100% CPU for extended periods, effectively blocking all other processing.

**RFC (Request for Comments):** In this context, an internal document that proposes a technical change, gathers feedback through a review process, and records the final decision along with its rationale.

**Root Override:** An `overrides` entry in the root package.json of a monorepo that applies to all transitive dependencies across all workspace packages.

**SSRF (Server-Side Request Forgery):** A vulnerability where an attacker can cause a server to make HTTP requests to an arbitrary URL. In the context of CVE-2023-45857, the XSRF behavior in axios could be abused as part of an SSRF chain by causing the browser to send CSRF tokens to attacker-controlled cross-origin endpoints.

**Transitive Dependency:** A package that is not directly listed in a project's package.json but is required by one of its direct dependencies (or by a dependency of a dependency, etc.). The full set of direct and transitive dependencies forms the dependency tree.

**Workspace Package:** In an npm monorepo, a package whose directory is listed in the root package.json's `workspaces` field. Workspace packages are linked into the root `node_modules` and can reference each other without publishing to a registry. In MonoStack, the workspace packages are `@monostack/core`, `@monostack/api`, `@monostack/ui`, and `@monostack/cli`.

**XSRF / CSRF (Cross-Site Request Forgery):** A type of attack where a malicious website causes a victim's browser to make authenticated requests to another site where the victim is logged in. XSRF tokens are a common defense mechanism: the server issues a token that must be included in state-changing requests and that third-party sites cannot obtain.

**Yarn Resolutions:** The yarn package manager's equivalent of npm's `overrides` field. Unlike npm overrides, yarn resolutions apply to both transitive and direct dependencies.

---

---

## 31. Appendix R: External References and Further Reading

This appendix lists the external resources consulted during RFC-2024-047's preparation. All links were verified as of the RFC's last updated date.

---

### R.1 CVE and Vulnerability References

**CVE-2021-23337 — lodash Prototype Pollution**
- NVD Entry: https://nvd.nist.gov/vuln/detail/CVE-2021-23337
- GitHub Security Advisory: https://github.com/advisories/GHSA-35jh-r3h4-6jhm
- Fix Commit (lodash 4.17.21): https://github.com/lodash/lodash/commit/c4847ebe7d14540bb28a8b932a9ce1b9ecbfee1a
- lodash Release Notes 4.17.21: https://github.com/lodash/lodash/releases/tag/4.17.21

The NVD entry accurately reflects the affected range (< 4.17.21) and fix version. The GitHub security advisory provides additional context on the exploitation mechanism. The fix commit shows the addition of path segment validation in the `baseSet` function.

**CVE-2022-25883 — semver ReDoS**
- NVD Entry: https://nvd.nist.gov/vuln/detail/CVE-2022-25883
- GitHub Security Advisory: https://github.com/advisories/GHSA-c2qf-rxjj-qqgw
- Initial Fix (7.5.2): https://github.com/npm/node-semver/pull/564
- Full Fix (7.6.0): https://github.com/npm/node-semver/releases/tag/v7.6.0
- 7.6.0 Regression Fix (7.6.1): https://github.com/npm/node-semver/releases/tag/v7.6.1
- Final Target (7.6.3): https://github.com/npm/node-semver/releases/tag/v7.6.3

Note: The NVD entry lists the fix as 7.5.2 based on the initial patch. As documented in §D.2 and Appendix E.2, 7.5.x versions remain partially vulnerable to carefully crafted inputs. The security team recommends 7.6.3 as the only fully remediated version in the semver package.

**CVE-2023-45857 — axios XSRF Token Leakage**
- NVD Entry: https://nvd.nist.gov/vuln/detail/CVE-2023-45857
- GitHub Security Advisory: https://github.com/advisories/GHSA-wf5p-g6vw-rhxx
- Fix PR: https://github.com/axios/axios/pull/6028
- axios 1.6.0 Changelog: https://github.com/axios/axios/releases/tag/v1.6.0

**Important:** As of the RFC's last updated date, the NVD entry for CVE-2023-45857 still lists the affected range as "< 1.4.0". This is incorrect. The actual fix was released in 1.6.0. A NVD correction request was filed by Desmond Achebe on 2024-01-27. Implementers should not rely on the NVD affected range for this CVE. The GitHub security advisory and the fix PR are the authoritative sources.

**CVE-2024-29041 — express Open Redirect**
- NVD Entry: https://nvd.nist.gov/vuln/detail/CVE-2024-29041
- GitHub Security Advisory: https://github.com/advisories/GHSA-rv95-896h-c2vc
- express 4.19.0 Release: https://github.com/expressjs/express/releases/tag/4.19.0
- express 4.19.2 Release: https://github.com/expressjs/express/releases/tag/4.19.2

The NVD entry for CVE-2024-29041 is accurate and lists 4.19.0 as the minimum fixed version. The RFC targets 4.19.2 (latest in the patched series) per our policy of targeting the latest available fix.

---

### R.2 npm Documentation

**npm overrides field documentation:**
https://docs.npmjs.com/cli/v10/configuring-npm/package-json#overrides

This is the authoritative documentation for the `overrides` field. Key points:
- Available since npm 8.3.0
- Does not apply to direct dependencies of workspace packages
- Exact versions, ranges, and nested overrides are all supported
- Overrides are applied before lockfile resolution

**npm workspaces documentation:**
https://docs.npmjs.com/cli/v10/using-npm/workspaces

Documents the workspace package resolution behavior. Specifically relevant: "overrides apply to all packages in the workspace except for direct dependencies defined within each workspace package."

**npm audit documentation:**
https://docs.npmjs.com/cli/v10/commands/npm-audit

Describes how `npm audit` queries the npm security advisory database. Note that the npm advisory database may differ from NVD in affected ranges, as discussed for CVE-2023-45857.

---

### R.3 General Security References

**NIST SP 800-40 — Guide to Enterprise Patch Management Technologies:**
https://csrc.nist.gov/publications/detail/sp/800-40/rev-4/final

Referenced in §O.4 for compliance mapping. Provides the framework for the RFC's remediation timeline and rollback planning requirements.

**OWASP Top 10 — A01:2021 Broken Access Control:**
The express open redirect vulnerability (CVE-2024-29041) is an instance of this OWASP category. Open redirects are used in phishing campaigns that rely on the victim seeing a trusted domain in the initial URL before being redirected to a malicious destination.

**OWASP Top 10 — A03:2021 Injection:**
The lodash prototype pollution vulnerability (CVE-2021-23337) is related to this OWASP category. Prototype pollution via crafted property names is a form of injection that modifies the JavaScript runtime environment rather than executing arbitrary code directly.

**Prototype Pollution — "Exploiting Client-Side Prototype Pollution in the Wild":**
A widely cited research paper documenting real-world exploitation of prototype pollution vulnerabilities. Relevant to understanding the severity of CVE-2021-23337 beyond what the CVSS score alone conveys.

**ReDoS — "ReDoS: Real World Impact":**
Security research examining real-world denial-of-service attacks using ReDoS. Documents incidents where production services were taken offline by crafted regex inputs similar to CVE-2022-25883.

---

### R.4 MonoStack Internal Documentation

The following internal documents should be consulted alongside this RFC:

- **Dependency Management Policy (Platform Engineering Handbook §7):** Defines the organization's policy for managing third-party dependencies, including the multi-round review process formalized by this RFC.
- **Security Incident Response Playbook (Security Team Runbooks §3):** The playbook for responding to newly discovered CVEs. RFC-2024-047 was initiated in accordance with §3.2 (High Severity CVE Response).
- **npm Workspace Architecture Guide (Platform Engineering Handbook §12):** Documents the MonoStack monorepo structure, workspace configuration, and the known limitations of npm workspaces relevant to override propagation.
- **CI/CD Pipeline Documentation (DevOps Runbooks §4):** Documents the CI pipeline configuration, including the `npm audit` check and `npm ci` lockfile enforcement added as part of this RFC's post-implementation monitoring plan.

---

*End of RFC-2024-047 (including all appendices A through R).*

*Document version: 3.0 (post-ARB ratification and post-implementation update)*
*Last updated: 2024-02-28*
*Maintained by: Platform Team — contact via `#platform-eng` Slack channel*

*Document version: 3.0 (post-ARB ratification and post-implementation update)*
*Last updated: 2024-02-28*
*Maintained by: Platform Team — contact via `#platform-eng` Slack channel*

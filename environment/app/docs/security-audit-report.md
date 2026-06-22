# MonoStack Q1 2024 Security Audit Report

**Report ID:** SEC-AUDIT-2024-Q1-MONOSTACK  
**Classification:** Internal — Confidential  
**Prepared By:** Security Guild  
**Audit Lead:** A. Patel, Security Guild Lead  
**Contributing Analysts:** B. Fernandez, C. Yamamoto, D. Osei  
**Audit Period:** 2024-01-15 through 2024-01-29  
**Report Date:** 2024-01-29  
**Status:** FINAL  
**Reviewed By:** Platform Team, DevOps, Frontend Guild  

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Audit Scope and Methodology](#2-audit-scope-and-methodology)
3. [Dependency Tree Analysis](#3-dependency-tree-analysis)
4. [Finding 1: lodash CVE-2021-23337](#4-finding-1-lodash-cve-2021-23337)
5. [Finding 2: semver CVE-2022-25883](#5-finding-2-semver-cve-2022-25883)
6. [Finding 3: axios CVE-2023-45857](#6-finding-3-axios-cve-2023-45857)
7. [Finding 4: express CVE-2024-29041](#7-finding-4-express-cve-2024-29041)
8. [Finding 5: react Version Fragmentation](#8-finding-5-react-version-fragmentation)
9. [Out-of-Scope Items Reviewed](#9-out-of-scope-items-reviewed)
10. [Remediation Recommendations](#10-remediation-recommendations)
11. [Risk Register](#11-risk-register)
12. [Audit Trail](#12-audit-trail)
13. [Appendix A: Full npm Dependency Tree Snapshot](#13-appendix-a-full-npm-dependency-tree-snapshot)
14. [Appendix B: Automated Scanner Results](#14-appendix-b-automated-scanner-results)
15. [Appendix C: Manual Code Review Notes](#15-appendix-c-manual-code-review-notes)

---

## 1. Executive Summary

The Security Guild conducted a full dependency security audit of the MonoStack npm workspace in January 2024. The audit was triggered by an internal report (MONO-8791) flagging a lodash prototype pollution vulnerability in `@monostack/core`, combined with a broader goal of establishing a formal quarterly dependency review process.

The audit identified **five issues** requiring remediation:

| # | Package | Issue | Severity | Current Version | Fix Version |
|---|---|---|---|---|---|
| 1 | lodash | CVE-2021-23337 (Prototype Pollution) | High | 4.17.20 | 4.17.21 |
| 2 | semver | CVE-2022-25883 (ReDoS) | Medium | 7.5.3 | 7.6.3 (recommended) |
| 3 | axios | CVE-2023-45857 (XSRF-TOKEN Exposure) | Medium | 1.3.4 | 1.6.8 (recommended) |
| 4 | express | CVE-2024-29041 (Path Traversal) | Medium | 4.18.2 | 4.19.2 |
| 5 | react | Version Fragmentation (Runtime Instability) | N/A | 18.0.0 / 18.1.0 / 18.2.0 | 18.3.1 (recommended) |

Three additional packages were reviewed and found to be either already patched or not applicable to the production risk profile. See Section 9 for details.

**Patching SLA:** Per company security policy, High severity CVEs must be remediated within 30 days of identification, and Medium severity CVEs within 90 days. All five findings are within the 90-day window as of the report date. The lodash finding (High severity) must be remediated by 2024-02-28.

**Recommended Approach:** The Platform Team has proposed using npm `overrides` in the workspace root `package.json`. The Security Guild endorses this approach as the most efficient and reliable remediation strategy for monorepo transitive dependency issues. See RFC-2024-047 for the full implementation specification.

---

## 2. Audit Scope and Methodology

### 2.1 Scope

The audit covered the npm dependency tree of the MonoStack workspace as installed on 2024-01-15. The workspace consists of four packages:

- `@monostack/core` at `packages/core`
- `@monostack/api` at `packages/api`
- `@monostack/ui` at `packages/ui`
- `@monostack/cli` at `packages/cli`

The audit focused on **production dependencies** (dependencies and peerDependencies) that are present in the installed `node_modules` at runtime. devDependencies were reviewed for completeness but are not subject to the same patching SLA given their limited exposure.

### 2.2 Methodology

The audit used a combination of automated scanning and manual review:

**Automated Scanning:**
- `npm audit` run against the full workspace dependency tree
- Snyk CLI scan with the organization's custom policy file
- OSS Review Toolkit (ORT) for license and vulnerability cross-reference

**Manual Review:**
- Dependency tree analysis using `npm ls --all` to identify version conflicts
- Code path analysis for each vulnerability to assess exploitability in the MonoStack context
- Peer dependency compatibility check for proposed fix versions

**Verification:**
- Each proposed fix version was tested in an isolated environment to confirm the vulnerability is not present
- Compatibility testing with MonoStack's existing usage patterns

### 2.3 Audit Environment

- Node.js version: 20.11.0 LTS
- npm version: 10.2.4
- Operating system: Ubuntu 22.04 LTS (audit environment), matching the production Docker base image
- Workspace root: `/audit-workspace/monostack`
- Audit date: 2024-01-15 (dependency snapshot), 2024-01-29 (final report)

### 2.4 Limitations

- The audit covers the dependency tree as installed on 2024-01-15. New vulnerabilities disclosed after this date are not covered.
- The audit does not cover binary dependencies, system libraries, or the Node.js runtime itself.
- The code path analysis in Section 15 is based on static analysis. Dynamic analysis (runtime testing) was not performed as part of this audit.

---

## 3. Dependency Tree Analysis

### 3.1 Overview

The full MonoStack workspace has the following dependency statistics as of 2024-01-15:

| Metric | Value |
|---|---|
| Direct dependencies (workspace-wide) | 47 |
| Total installed packages | 812 |
| Unique package names | 689 |
| Packages with multiple versions installed | 23 |
| Packages flagged by `npm audit` | 5 |
| Packages flagged by Snyk (including informational) | 8 |
| Packages with known CVEs (CVSS ≥ 4.0) | 4 |

### 3.2 Multi-Version Packages

The following packages have multiple versions installed simultaneously across the workspace, representing potential version fragmentation or peer dependency conflicts:

| Package | Installed Versions | Causes |
|---|---|---|
| react | 18.0.0, 18.1.0, 18.2.0 | Multiple transitive consumers with different version requirements |
| react-dom | 18.2.0 | Single version — OK |
| lodash | 4.17.20, 4.17.21 | Some packages already pin 4.17.21 |
| semver | 7.5.3, 7.5.4 | Some npm internal tools pin 7.5.4 |
| axios | 1.3.4 | Single version at workspace level |
| express | 4.18.2 | Single version at workspace level |
| debug | 2.6.9, 4.3.4 | Express 4.x uses debug 2.x; other packages use debug 4.x |
| ms | 2.0.0, 2.1.3 | Same as debug — Express pulls in ms 2.0.0 via debug 2.6.9 |
| mime | 1.6.0, 2.5.2 | Different packages prefer different major versions |
| qs | 6.11.0, 6.13.0 | Version fragmentation in HTTP client libraries |
| iconv-lite | 0.4.24, 0.6.3 | Body-parser pins 0.4.24; other packages use 0.6.3 |

### 3.3 Dependency Chain for Flagged Packages

#### lodash
```
@monostack/core
└── data-processor@2.1.0
    └── lodash@4.17.20   ← VULNERABLE

@monostack/api
└── config-merger@1.4.2
    └── lodash@4.17.20   ← VULNERABLE (same version, hoisted)
```

#### semver
```
@monostack/core
└── semver@7.5.3   ← VULNERABLE (direct dep in core)

@monostack/api (npm internal)
└── @npmcli/package-json@5.0.3
    └── semver@7.5.4   ← Fixed (newer, but different version)
```

#### axios
```
@monostack/api
└── axios@1.3.4   ← VULNERABLE (direct dep in api)
```

#### express
```
@monostack/api
└── express@4.18.2   ← VULNERABLE (direct dep in api)
```

#### react
```
@monostack/ui
└── react@18.2.0   ← Primary (direct dep)
└── react-beautiful-dnd@13.1.1
    └── react@18.0.0   ← Transitive (nested)

@monostack/core
└── react-query@3.39.3
    └── react@18.1.0   ← Transitive (nested)
```

### 3.4 Hoisting Analysis

npm workspace hoisting places a single version of a package in the root `node_modules` when possible, with per-package copies only when version conflicts require it.

For this workspace:
- `lodash@4.17.20` is hoisted to `node_modules/lodash` (used by both `@monostack/core` and `@monostack/api`)
- `react@18.2.0` is hoisted to `node_modules/react` (the `@monostack/ui` version wins the hoisting race)
- `react@18.0.0` ends up in `node_modules/react-beautiful-dnd/node_modules/react` (cannot be hoisted due to version conflict)
- `react@18.1.0` ends up in `node_modules/react-query/node_modules/react` (cannot be hoisted due to version conflict)

The `overrides` mechanism resolves the hoisting ambiguity by forcing all packages to use the specified version, eliminating nested copies.

---

## 4. Finding 1: lodash CVE-2021-23337

### 4.1 Finding Summary

| Field | Value |
|---|---|
| CVE ID | CVE-2021-23337 |
| Package | lodash |
| Installed Version | 4.17.20 |
| Fix Version | 4.17.21 |
| CVSS Score | 7.2 (High) |
| CVSS Vector | CVSS:3.1/AV:N/AC:L/PR:H/UI:N/S:U/C:H/I:H/A:H |
| CWE | CWE-1321 |
| Published | 2021-02-15 |
| Reported to Platform Team | 2024-01-08 |

### 4.2 Vulnerability Description

CVE-2021-23337 is a prototype pollution vulnerability in the lodash library. The vulnerability exists in the `zipObjectDeep` function and related path-based mutation functions (`_.set`, `_.setWith`, `_.merge`). When a user-controlled string containing `__proto__`, `constructor`, or `prototype` is used as a path key, lodash will mutate the global JavaScript object prototype, affecting all objects created after the mutation.

Prototype pollution is a class of vulnerability specific to JavaScript's inheritance model. Because JavaScript objects inherit properties from their prototype chain, polluting `Object.prototype` affects every object in the JavaScript process, including internal Node.js runtime objects.

### 4.3 Exploitability Assessment in MonoStack

The Security Guild assessed exploitability across each MonoStack package:

**`@monostack/core`:**
The `data-processor` module uses `_.merge()` to combine configuration objects. The configuration data originates from:
1. Hardcoded defaults (not exploitable)
2. Environment variables (partially controllable by an operator — limited risk)
3. API request bodies parsed from JSON (potentially controllable by an external attacker)

Code path identified in `packages/core/src/config/merger.js`:
```javascript
const _ = require('lodash');
module.exports = function mergeConfig(defaults, overrides) {
  return _.merge({}, defaults, overrides);
};
```

When `overrides` is derived from a user-controlled JSON body (which occurs in the admin configuration endpoint at `POST /api/admin/config`), a malicious payload could pollute `Object.prototype`:
```json
{"__proto__": {"isAdmin": true, "role": "superuser"}}
```

After this payload is processed by `_.merge()`, every subsequently created object in the Node.js process would have `isAdmin: true` and `role: "superuser"` as properties, potentially bypassing authorization checks that look for these properties on request context objects.

**`@monostack/api`:**
The `config-merger` dependency also uses lodash for similar purposes. The same attack surface applies.

**`@monostack/ui`:**
The UI package is a client-side React application. While client-side prototype pollution is possible, the impact is limited to the user's own browser session. Not considered a meaningful risk for this finding.

**`@monostack/cli`:**
The CLI package does not accept external network input. Prototype pollution through the CLI would require a malicious user with local access to craft a specific invocation. Not considered a meaningful risk for this finding.

**Exploitability Rating:** Medium-High. The attack surface in `@monostack/api` is real but requires an authenticated request to the admin configuration endpoint. The endpoint is restricted to admin users, reducing likelihood.

### 4.4 Remediation Verification

The fix in lodash `4.17.21` modifies the `baseSet` function (the internal function used by `_.set`, `_.merge`, `_.zipObjectDeep`, and related functions) to reject path segments that equal `__proto__`, `prototype`, or `constructor` when the parent object is a plain object or function.

Verification test run by the Security Guild:
```javascript
const _ = require('lodash@4.17.21');

// Test 1: zipObjectDeep path
const result1 = _.zipObjectDeep(['__proto__.polluted'], [true]);
console.assert(!{}.polluted, 'Test 1 failed: __proto__ not blocked');

// Test 2: _.set path
const obj = {};
_.set(obj, '__proto__.polluted2', true);
console.assert(!{}.polluted2, 'Test 2 failed: __proto__ set not blocked');

// Test 3: _.merge
const base = {};
_.merge(base, JSON.parse('{"__proto__":{"polluted3":true}}'));
console.assert(!{}.polluted3, 'Test 3 failed: _.merge not blocked');

console.log('All verification tests passed.');
```

All three tests pass with lodash `4.17.21`. The same tests fail (prototype is polluted) with lodash `4.17.20`.

### 4.5 Recommendation

**Override lodash to `4.17.21` in the workspace root `package.json`.**

This is a single patch version upgrade with no breaking changes. The Security Guild assigns this finding a patching priority of **Critical** (due to the potential authentication bypass consequence) despite the Medium-High exploitability rating.

---

## 5. Finding 2: semver CVE-2022-25883

### 5.1 Finding Summary

| Field | Value |
|---|---|
| CVE ID | CVE-2022-25883 |
| Package | semver |
| Installed Version | 7.5.3 |
| Fix Version | 7.5.2 (initial), 7.6.3 (recommended) |
| CVSS Score | 5.3 (Medium) |
| CVSS Vector | CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:L |
| CWE | CWE-1333 |
| Published | 2023-06-21 |
| Reported to Platform Team | 2024-01-15 |

### 5.2 Vulnerability Description

CVE-2022-25883 is a Regular Expression Denial of Service (ReDoS) vulnerability in the semver package's version and range parsing logic. The vulnerability allows an attacker to provide a crafted version string that causes the regular expression engine to enter a state of catastrophic backtracking, consuming 100% CPU for an extended period and effectively blocking the Node.js event loop.

**Important note on existing version:** The version currently installed (`7.5.3`) was published *after* the CVE was filed and *does* include the initial fix. However, the Security Guild recommends upgrading to `7.6.3` because:
1. The initial fix in `7.5.2` used input length limits as the primary protection
2. Subsequent releases (`7.6.0` through `7.6.3`) rewrote the regex patterns to be non-backtracking
3. The non-backtracking approach is more robust and eliminates the vulnerability class rather than just mitigating it

### 5.3 Exploitability Assessment

`@monostack/core` uses semver for version range comparisons in its package manifest processing. The relevant question is whether attacker-controlled input can reach the `semver.satisfies()` or `semver.valid()` call paths.

Review of the code in `packages/core/src/manifest.js`:
```javascript
const semver = require('semver');

// Version compatibility check — used when loading plugins
function isCompatiblePlugin(pluginVersion, requiredRange) {
  if (!pluginVersion || !requiredRange) return false;
  return semver.satisfies(pluginVersion, requiredRange);
}
```

The `pluginVersion` and `requiredRange` parameters are read from plugin manifest files. In the current deployment:
- Plugins are loaded from a curated internal registry (controlled by the Platform Team)
- External plugin upload functionality exists but requires admin authentication

**Exploitability Rating:** Low. While the attack surface theoretically exists (admin user uploads a plugin with a malicious version string), the access level required and the limited impact (event loop hang, recoverable) make this a low-priority finding. The Security Guild recommends remediation anyway as defense-in-depth.

### 5.4 Remediation Verification

The Security Guild tested both the input-length mitigation (present in `7.5.3`) and the non-backtracking regex approach (present in `7.6.3`):

**Test with `7.5.3` (input length limit):**
```javascript
const semver = require('semver@7.5.3');
const malicious = '1.2.3-' + 'a'.repeat(300); // Exceeds 256-char limit
console.time('parse');
const result = semver.valid(malicious);
console.timeEnd('parse'); // ~0ms — rejected quickly due to length limit
console.log(result); // null
```

**Test with `7.6.3` (non-backtracking regex):**
```javascript
const semver = require('semver@7.6.3');
const edge_case = '1.2.3-' + 'a.'.repeat(100) + 'b'; // Complex prerelease
console.time('parse');
const result = semver.valid(edge_case);
console.timeEnd('parse'); // ~0ms — handled efficiently regardless of length
```

Both versions handle the malicious input safely, but `7.6.3`'s approach is fundamentally more robust.

### 5.5 Recommendation

**Override semver to `7.6.3` in the workspace root `package.json`.**

Although `7.5.3` (the current version) technically fixes the original CVE, the `7.6.x` series provides stronger protection through a more fundamental fix. The upgrade is backward compatible.

---

## 6. Finding 3: axios CVE-2023-45857

### 6.1 Finding Summary

| Field | Value |
|---|---|
| CVE ID | CVE-2023-45857 |
| Package | axios |
| Installed Version | 1.3.4 |
| Fix Version | 1.6.0 (CVE fix), 1.6.8 (recommended) |
| CVSS Score | 6.5 (Medium) |
| CVSS Vector | CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:N/A:N |
| CWE | CWE-918 |
| Published | 2023-11-08 |
| Reported to Platform Team | 2024-01-15 |

### 6.2 Vulnerability Description

CVE-2023-45857 affects axios versions from `0.21.1` through `1.5.x`. The vulnerability is in how axios handles the `XSRF-TOKEN` cookie when making cross-origin requests in a browser environment. Under normal CSRF protection implementations, the `XSRF-TOKEN` value should only be sent to the same origin that set it. The bug in axios causes the `X-XSRF-TOKEN` header (containing the token value) to be included in cross-origin requests, potentially leaking the user's CSRF token to third-party servers.

**Context for MonoStack:** The primary usage of axios in MonoStack is in `@monostack/api` for server-to-server HTTP calls. This is a Node.js server environment. The XSRF-TOKEN vulnerability is specifically a browser-environment issue because:
- It involves browser cookie storage and the same-origin policy
- Node.js does not have cookies or a same-origin policy in the traditional sense
- Server-to-server HTTP calls using axios are not affected by this specific CVE

**Why remediate despite low direct exposure?**
1. The `1.6.x` series includes additional security patches beyond CVE-2023-45857
2. A future frontend integration might use axios in a browser context
3. Company policy requires remediating all Medium-severity CVEs in installed packages regardless of current exploitability

### 6.3 Additional Security Issues in the 1.6.x Series

During the audit, the Security Guild reviewed the full 1.6.x changelog for security-relevant changes:

**axios 1.6.3 — Null-byte injection in URL parameters**
An attacker-controlled URL parameter containing a null byte (`\0`) could cause inconsistent URL handling between axios and downstream servers (like nginx or Express). This could potentially be used to bypass URL-based security checks. Fixed in `1.6.3`.

Relevant to MonoStack: `@monostack/api` constructs URLs from configuration data and forwards them to downstream services. If any configuration data originates from user input, this could be relevant.

**axios 1.6.6 — Prototype pollution in `mergeDeep`**
axios's internal `mergeDeep` utility (used to merge request configuration objects) was vulnerable to prototype pollution when the merge target includes a `__proto__` key. This is structurally similar to the lodash finding.

In MonoStack, axios request configuration objects are typically built from static configuration, reducing the attack surface. However, the `interceptors` feature in `@monostack/api` merges dynamically-constructed configuration objects in some request paths.

**axios 1.6.8 — SSRF in redirect handling**
When axios follows a redirect, the redirect target URL was not fully validated against the same-origin policy. A server returning a crafted `Location` header could cause axios to make a follow-up request to an internal network resource (e.g., `http://169.254.169.254/latest/meta-data/` on AWS). This is relevant to `@monostack/api`'s server-to-server calls.

The Security Guild elevated the recommendation from `1.6.0` to `1.6.8` specifically because of this SSRF finding.

### 6.4 The Round 1 Mistake: Why 1.4.0 Was Proposed

The Platform Team's initial proposal of `1.4.0` reflected a misunderstanding of CVE timeline vs. fix timeline. The sequence of events:

1. March 2023: axios `1.4.0` released (no security motivation)
2. October 2023: axios `1.6.0` released, which internally fixed the XSRF-TOKEN issue (as a side effect of a broader refactor of the credentials handling logic)
3. November 2023: Security researcher publicly reports the XSRF-TOKEN issue; CVE-2023-45857 is assigned
4. The CVE report explicitly cites `1.6.0` as the fix version

The Platform Team saw that `1.4.0` was published after their existing version (`1.3.4`) and assumed it would include the fix. This reasoning is incorrect: a package version published before a CVE fix does not include that fix.

### 6.5 Recommendation

**Override axios to `1.6.8` in the workspace root `package.json`.**

The minimum version to fix CVE-2023-45857 is `1.6.0`. However, `1.6.8` is the current latest in the `1.6.x` series and includes fixes for prototype pollution (`1.6.6`) and SSRF in redirects (`1.6.8`), both of which are relevant to MonoStack's server-side usage.

---

## 7. Finding 4: express CVE-2024-29041

### 7.1 Finding Summary

| Field | Value |
|---|---|
| CVE ID | CVE-2024-29041 |
| Package | express |
| Installed Version | 4.18.2 |
| Fix Version | 4.19.2 |
| CVSS Score | 6.1 (Medium) |
| CVSS Vector | CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N |
| CWE | CWE-22 |
| Published | 2024-03-25 |
| Reported to Platform Team | 2024-01-15 (as anticipated; CVE formally published later) |

### 7.2 Vulnerability Description

CVE-2024-29041 affects the Express.js web framework. The vulnerability is in Express's URL normalization logic when the application is configured to trust proxy headers (`app.set('trust proxy', true)` or equivalent). Under specific conditions, a crafted `Host` header or URL can cause `res.redirect()` to generate a redirect response pointing to an unintended location, including attacker-controlled URLs or internal network paths.

**Attack scenario:**
1. A user visits a page on the MonoStack application that performs an authentication redirect (e.g., OAuth callback)
2. The attacker crafts a request with a modified `Host` header pointing to `evil.attacker.com`
3. Express's `res.redirect()`, when constructing an absolute redirect URL, uses the `Host` header to build the URL
4. The user is redirected to `https://evil.attacker.com/continue` instead of the intended internal URL
5. This constitutes an open redirect vulnerability, which can be leveraged for phishing or credential harvesting

### 7.3 Exploitability Assessment

`@monostack/api` is deployed behind an AWS Application Load Balancer (ALB) with the following configuration:
```javascript
app.set('trust proxy', 1); // Trust the first proxy in the chain
```

This setting is required for the application to correctly read client IP addresses and HTTPS protocol information. It is also the necessary precondition for the CVE-2024-29041 exploit.

The authentication flow in `@monostack/api` includes an OAuth callback route:
```javascript
app.get('/auth/callback', async (req, res) => {
  const token = await exchangeCode(req.query.code);
  const redirectTo = req.query.return_url || '/dashboard';
  res.redirect(redirectTo);
});
```

Under the CVE, a crafted request can cause the redirect to go to an unintended location if the Express URL normalization bug is triggered. The Security Guild confirmed this code path is vulnerable to the CVE.

**Exploitability Rating:** Medium. Requires a user to click a crafted link to the callback URL. The ALB does strip some headers, but the Host header is forwarded (it's required for routing). The practical exploitability is mitigated by the requirement for user interaction but the attack surface is real.

### 7.4 Why Versions Before 4.19.2 Don't Fix It

This is documented in Section D.4 of RFC-2024-047. Key summary:
- `4.18.3` — Released 2024-03-05, does not include the CVE fix (fix was not yet developed)
- `4.19.0` — Released 2024-03-25, contains an incomplete fix (bypass discovered immediately)
- `4.19.1` — Released 2024-03-26, attempted to close the bypass, still incomplete
- `4.19.2` — Released 2024-03-25, is the Express team's confirmed full fix

The confusing timeline arises because the Express maintainers published `4.19.0` and discovered a bypass before it was widely adopted, then rapidly published `4.19.2` (skipping `4.19.1` effectively).

### 7.5 Recommendation

**Override express to `4.19.2` in the workspace root `package.json`.**

Only `4.19.2` contains the confirmed full fix. The Security Guild will not accept `4.18.x`, `4.19.0`, or `4.19.1` as remediation.

---

## 8. Finding 5: react Version Fragmentation

### 8.1 Finding Summary

| Field | Value |
|---|---|
| CVE ID | N/A |
| Package | react |
| Issue | Multiple versions installed simultaneously |
| Affected Versions | 18.0.0, 18.1.0, 18.2.0 (all three installed) |
| Recommended Version | 18.3.1 |
| Severity | Not a CVE; runtime stability risk |

### 8.2 Issue Description

The MonoStack workspace has three different versions of React installed simultaneously due to transitive dependency version mismatches. React relies on module-level singletons — specifically, the internal fiber reconciler state and the hooks dispatcher — that are stored in a single module instance. When two different React instances are loaded by the same JavaScript runtime, these singletons are duplicated, causing invariant violations.

**Symptoms observed in MONO-8841 (Q4 2023):**
- Drag-and-drop functionality in the UI throwing "Invalid hook call" errors
- Specific error message: "Hooks can only be called inside the function body of a function component"
- Error occurs only in production builds, not in development (due to difference in module resolution behavior)
- Error is inconsistent — occurs for approximately 15% of users based on browser-specific module caching behavior

**Root cause analysis:**
The `react-beautiful-dnd` package (used for drag-and-drop) uses React hooks internally. It resolves to its own copy of `react@18.0.0` from its nested `node_modules`. When a drag event occurs, `react-beautiful-dnd` calls hooks using the `react@18.0.0` instance, but the component tree was rendered using `react@18.2.0` from the hoisted `node_modules`. The React hook invariant check detects that the hooks dispatcher is from a different React instance and throws.

### 8.3 Impact Assessment

- **End user impact:** Drag-and-drop functionality fails for ~15% of users
- **Business impact:** Core task management features in `@monostack/ui` are impaired
- **Workaround:** None available without code changes
- **Frequency:** Occurs on every drag operation for affected users

The Security Guild includes this finding in the audit because:
1. Resolving it requires the same `overrides` mechanism used for the CVE findings
2. The Frontend Guild has indicated this is a production stability issue requiring urgent resolution
3. Grouping it with the security overrides is operationally efficient

### 8.4 Why `18.3.1` and Not Just `18.2.0`

See RFC-2024-047 Section 5.5 and 9.3 for the full Frontend Guild rationale. In summary:
- `18.2.0` would fix the fragmentation issue but miss the opportunity to surface React 19 deprecation warnings
- `18.3.1` is the latest stable release in the 18.x series
- `18.3.0` had an SSR regression that `18.3.1` fixed
- The Frontend Guild recommends `18.3.1` as the canonical target

### 8.5 Note on `react-dom`

The Security Guild reviewed the `react-dom` situation. `react-dom@18.2.0` is a direct dependency in `@monostack/ui`. It does not exhibit the same fragmentation issue because:
- Only one version of `react-dom` is installed (18.2.0)
- The `react-dom` and `react` versions being mismatched (`react@18.3.1` via override, `react-dom@18.2.0` via direct dep) is acceptable per the React team's release policy

The Frontend Guild has filed MONO-9047 to update `react-dom` directly in `@monostack/ui/package.json`. This is not within the scope of the overrides RFC.

### 8.6 Recommendation

**Override react to `18.3.1` in the workspace root `package.json`. Do NOT include `react-dom` in the overrides.**

---

## 9. Out-of-Scope Items Reviewed

The Security Guild reviewed the following packages that were raised during the audit planning phase but determined to be out of scope or not requiring immediate action:

### 9.1 webpack — No CVE, Deferred

`webpack@5.88.2` is installed as a devDependency in `@monostack/ui`. During the audit, a team member flagged a potential concern about webpack's handling of module resolution in certain edge cases. After investigation:
- No CVEs are open against `webpack@5.88.2`
- The concern was about a non-security behavioral edge case
- Upgrading webpack requires extensive testing (it is a major build toolchain component)
- The Frontend Guild requested deferral to Q3 2024 for a planned webpack 5 → 5 minor version upgrade

**Decision:** Out of scope. No action required.

### 9.2 babel — No CVE, Deferred

`@babel/core@7.23.5` and related `@babel/` packages are installed as devDependencies. During audit review:
- No open CVEs against the installed versions
- Babel is a devDependency only; not present in production builds
- Frontend team requested deferral to Q3 2024 alongside the webpack update

**Decision:** Out of scope. No action required.

### 9.3 eslint — DevDependency Only

`eslint@8.55.0` is a devDependency. CVE-2023-26119 affects eslint <8.53.0, but the installed version `8.55.0` is not affected.

**Decision:** Already patched. No action required.

### 9.4 node-fetch — Already Patched

`node-fetch@3.3.2` is installed transitively. Earlier versions had CVE-2022-0235 (open redirect). The installed version `3.3.2` is patched.

**Decision:** Already patched. No action required.

---

## 10. Remediation Recommendations

### 10.1 Recommended Actions (Priority Order)

| Priority | Package | Action | Deadline |
|---|---|---|---|
| 1 | lodash | Override to `4.17.21` via npm overrides | 2024-02-28 (30-day High severity SLA) |
| 2 | express | Override to `4.19.2` via npm overrides | 2024-04-14 (90-day Medium severity SLA) |
| 3 | axios | Override to `1.6.8` via npm overrides | 2024-04-14 (90-day Medium severity SLA) |
| 4 | semver | Override to `7.6.3` via npm overrides | 2024-04-14 (90-day Medium severity SLA) |
| 5 | react | Override to `18.3.1` via npm overrides | ASAP (production stability issue) |

### 10.2 Preferred Remediation Approach

The Security Guild endorses the Platform Team's proposed approach of using npm workspace `overrides` (documented in RFC-2024-047). This approach:

1. **Requires a single change** to the root `package.json`
2. **Takes effect workspace-wide** without requiring individual package updates
3. **Is explicit and reviewable** in version control
4. **Does not require upstream packages to release new versions**
5. **Is reversible** by removing the `overrides` block

Alternative approaches were considered:

**Alternative A: Update direct dependencies in each package**
This would require updating `packages/api/package.json` to upgrade `axios` and `express`, and `packages/core/package.json` to upgrade `semver` and `lodash`. This is more work (four separate PRs vs one) and requires the individual package owners to accept the upgrades. Not recommended.

**Alternative B: Use npm-force-resolutions**
This is a community tool that pre-dates the official `overrides` field. It is more fragile and not officially supported by npm. Not recommended.

**Alternative C: Patch packages using patch-package**
This would require maintaining local patches for each vulnerable package. Highly fragile and creates an ongoing maintenance burden. Not recommended.

### 10.3 Post-Remediation Verification

After applying the overrides and running `npm install`, the following verification steps should be performed:

1. Run `npm ls lodash` — confirm only `4.17.21` appears in the tree
2. Run `npm ls semver` — confirm only `7.6.3` appears in the tree
3. Run `npm ls axios` — confirm only `1.6.8` appears
4. Run `npm ls express` — confirm only `4.19.2` appears
5. Run `npm ls react` — confirm only `18.3.1` appears
6. Run `npm audit` — confirm no remaining high/medium severity findings
7. Run the full test suite — confirm no regressions
8. Run a specific prototype pollution test for lodash (see Section 4.4)

---

## 11. Risk Register

| ID | Package | Risk | Likelihood | Impact | Score | Status |
|---|---|---|---|---|---|---|
| R-01 | lodash | Prototype pollution via `_.merge()` in admin config endpoint | Medium | High | 12 | OPEN — remediate by 2024-02-28 |
| R-02 | semver | ReDoS via crafted plugin version string | Low | Medium | 4 | OPEN — remediate by 2024-04-14 |
| R-03 | axios | XSRF-TOKEN exposure in browser context (CVE-2023-45857) | Low (server-side use) | Medium | 4 | OPEN — remediate by 2024-04-14 |
| R-04 | axios | Prototype pollution in `mergeDeep` (axios 1.6.6 issue) | Medium | High | 12 | OPEN — remediate with R-03 |
| R-05 | axios | SSRF in redirect handling (axios 1.6.8 issue) | Medium | High | 12 | OPEN — remediate with R-03 |
| R-06 | express | Open redirect via Host header (CVE-2024-29041) | Medium | Medium | 9 | OPEN — remediate by 2024-04-14 |
| R-07 | react | Hook invariant violations causing UI crashes | High | Medium | 12 | OPEN — ASAP (production stability) |
| R-08 | (all) | npm override causes regression in dependent packages | Low | Medium | 4 | ACCEPT — mitigated by compatibility testing |

---

## 12. Audit Trail

### 12.1 Audit Timeline

| Date | Activity | Participants |
|---|---|---|
| 2024-01-08 | Audit kickoff meeting | A. Patel, B. Fernandez, J. Hartwell |
| 2024-01-15 | Dependency tree snapshot taken | C. Yamamoto |
| 2024-01-15 | Automated scanner runs | C. Yamamoto, D. Osei |
| 2024-01-16 – 2024-01-22 | Vulnerability analysis and code path review | B. Fernandez, C. Yamamoto |
| 2024-01-23 | Internal review of draft findings | All Security Guild |
| 2024-01-24 | Draft findings sent to Platform Team | A. Patel |
| 2024-01-25 | Platform Team review call | A. Patel, J. Hartwell, T. Bergström |
| 2024-01-29 | Final report published | A. Patel |

### 12.2 Document History

| Version | Date | Author | Changes |
|---|---|---|---|
| 0.1 | 2024-01-22 | B. Fernandez | Initial draft |
| 0.2 | 2024-01-23 | A. Patel | Added risk register and executive summary |
| 0.3 | 2024-01-24 | C. Yamamoto | Added Appendix A and B |
| 0.4 | 2024-01-25 | A. Patel | Incorporated Platform Team feedback |
| 1.0 | 2024-01-29 | A. Patel | Final version |

---

## 13. Appendix A: Full npm Dependency Tree Snapshot

Abbreviated for readability. Full tree available in Confluence under SEC-AUDIT-2024-Q1-MONOSTACK.

```
monostack@ /audit-workspace/monostack
├── @monostack/api@1.4.2 -> packages/api
│   ├── axios@1.3.4
│   ├── express@4.18.2
│   │   ├── accepts@1.3.8
│   │   │   ├── mime-types@2.1.35
│   │   │   └── negotiator@0.6.3
│   │   ├── array-flatten@1.1.1
│   │   ├── body-parser@1.20.2
│   │   │   ├── bytes@3.1.2
│   │   │   ├── content-type@1.0.5
│   │   │   ├── debug@2.6.9
│   │   │   │   └── ms@2.0.0
│   │   │   ├── depd@2.0.0
│   │   │   ├── destroy@1.2.0
│   │   │   ├── http-errors@2.0.0
│   │   │   │   ├── depd@2.0.0 deduped
│   │   │   │   ├── inherits@2.0.4
│   │   │   │   ├── setprototypeof@1.2.0
│   │   │   │   ├── statuses@2.0.1
│   │   │   │   └── toidentifier@1.0.1
│   │   │   ├── iconv-lite@0.4.24
│   │   │   │   └── safer-buffer@2.1.2
│   │   │   ├── on-finished@2.4.1
│   │   │   │   └── ee-first@1.1.1
│   │   │   ├── qs@6.11.0
│   │   │   ├── raw-body@2.5.2
│   │   │   │   ├── bytes@3.1.2 deduped
│   │   │   │   ├── http-errors@2.0.0 deduped
│   │   │   │   ├── iconv-lite@0.4.24 deduped
│   │   │   │   └── unpipe@1.0.0
│   │   │   └── type-is@1.6.18
│   │   │       ├── media-typer@0.3.0
│   │   │       └── mime-types@2.1.35 deduped
│   │   ├── content-disposition@0.5.4
│   │   │   └── safe-buffer@5.2.1
│   │   ├── content-type@1.0.5 deduped
│   │   ├── cookie@0.6.0
│   │   ├── cookie-signature@1.0.6
│   │   ├── debug@2.6.9 deduped
│   │   ├── depd@2.0.0 deduped
│   │   ├── encodeurl@1.0.2
│   │   ├── escape-html@1.0.3
│   │   ├── etag@1.8.1
│   │   ├── finalhandler@1.2.0
│   │   ├── fresh@0.5.2
│   │   ├── http-errors@2.0.0 deduped
│   │   ├── merge-descriptors@1.0.1
│   │   ├── methods@1.1.2
│   │   ├── on-finished@2.4.1 deduped
│   │   ├── parseurl@1.3.3
│   │   ├── path-to-regexp@0.1.7
│   │   ├── proxy-addr@2.0.7
│   │   │   ├── forwarded@0.2.0
│   │   │   └── ipaddr.js@1.9.1
│   │   ├── qs@6.11.0 deduped
│   │   ├── range-parser@1.2.1
│   │   ├── safe-buffer@5.2.1 deduped
│   │   ├── send@0.18.0
│   │   │   ├── debug@2.6.9 deduped
│   │   │   ├── depd@2.0.0 deduped
│   │   │   ├── destroy@1.2.0 deduped
│   │   │   ├── encodeurl@1.0.2 deduped
│   │   │   ├── escape-html@1.0.3 deduped
│   │   │   ├── etag@1.8.1 deduped
│   │   │   ├── fresh@0.5.2 deduped
│   │   │   ├── http-errors@2.0.0 deduped
│   │   │   ├── mime@1.6.0
│   │   │   ├── ms@2.1.3
│   │   │   ├── on-finished@2.4.1 deduped
│   │   │   └── range-parser@1.2.1 deduped
│   │   ├── serve-static@1.15.0
│   │   ├── setprototypeof@1.2.0 deduped
│   │   ├── statuses@2.0.1 deduped
│   │   ├── type-is@1.6.18 deduped
│   │   ├── utils-merge@1.0.1
│   │   └── vary@1.1.2
│   └── (... additional api deps truncated for brevity ...)
├── @monostack/core@2.3.1 -> packages/core
│   ├── data-processor@2.1.0
│   │   └── lodash@4.17.20   ← VULNERABLE
│   ├── semver@7.5.3   ← CURRENT (partially mitigated)
│   └── (... additional core deps truncated for brevity ...)
├── @monostack/ui@3.1.0 -> packages/ui
│   ├── react@18.2.0
│   ├── react-dom@18.2.0
│   ├── react-beautiful-dnd@13.1.1
│   │   └── react@18.0.0   ← FRAGMENTED
│   └── (... additional ui deps truncated for brevity ...)
└── @monostack/cli@1.2.0 -> packages/cli
    └── (... cli deps, no flagged packages ...)
```

---

## 14. Appendix B: Automated Scanner Results

### B.1 npm audit Summary

```
# npm audit report

lodash  <=4.17.20
Severity: high
Prototype Pollution in lodash - https://github.com/advisories/GHSA-p6mc-m468-83gw
fix available via `npm audit fix`
node_modules/data-processor/node_modules/lodash

semver  <7.5.2
Severity: moderate
semver vulnerable to Regular Expression Denial of Service - https://github.com/advisories/GHSA-c2qf-rxjj-qqgw
fix available via `npm audit fix`
node_modules/semver

axios  <=1.5.1
Severity: moderate
axios Cross-Site Request Forgery Vulnerability - https://github.com/advisories/GHSA-wf5p-g6vw-rhxx
fix available via `npm audit fix`
node_modules/axios

express  <4.19.2
Severity: moderate
Express.js Open Redirect in malformed URLs - https://github.com/advisories/GHSA-rv95-896h-c2vc
fix available via `npm audit fix`
node_modules/express

4 vulnerabilities (1 high, 3 moderate)

To address all issues, run:
  npm audit fix
```

### B.2 Snyk CLI Scan Summary

```
Testing monostack...

✗ High severity vulnerability found in lodash
  Description: Prototype Pollution
  Info: https://snyk.io/vuln/SNYK-JS-LODASH-1040724
  Introduced through: @monostack/core > data-processor > lodash
  From: @monostack/core > data-processor > lodash@4.17.20
  Fix: Upgrade to lodash@4.17.21

✗ Medium severity vulnerability found in semver
  Description: Regular Expression Denial of Service (ReDoS)
  Info: https://snyk.io/vuln/SNYK-JS-SEMVER-3247795
  Introduced through: @monostack/core > semver
  From: @monostack/core > semver@7.5.3
  Fix: Upgrade to semver@7.5.2 or later

✗ Medium severity vulnerability found in axios
  Description: Cross-Site Request Forgery (CSRF)
  Info: https://snyk.io/vuln/SNYK-JS-AXIOS-6124857
  Introduced through: @monostack/api > axios
  From: @monostack/api > axios@1.3.4
  Fix: Upgrade to axios@1.6.0 or later

✗ Medium severity vulnerability found in express
  Description: Open Redirect
  Info: https://snyk.io/vuln/SNYK-JS-EXPRESS-6474471
  Introduced through: @monostack/api > express
  From: @monostack/api > express@4.18.2
  Fix: Upgrade to express@4.19.2

✓ No license issues found.

Organization: monostack
Package manager: npm
Target file: package.json
Project name: monostack
Open source: no
Project path: /audit-workspace/monostack
Licenses: enabled

4 dependency vulnerabilities, 0 license issues found.
```

---

## 15. Appendix C: Manual Code Review Notes

### C.1 Review Scope

Manual code review was performed on the code paths most likely to be exploitable given the identified vulnerabilities. The review focused on:
- Functions that call `_.merge()` or similar lodash path-mutation functions with external data
- Functions that call `semver.*()` with user-provided version strings
- Functions that call axios with configuration objects derived from external data
- Express route handlers that call `res.redirect()` with dynamic values

### C.2 lodash Code Path Review

**File: `packages/core/src/config/merger.js`**
Risk: HIGH — `_.merge()` called with `overrides` parameter that can originate from API request bodies.

**File: `packages/api/src/middleware/config-injector.js`**
Risk: MEDIUM — `_.merge()` used for merging request-scoped config, but data originates from internal service calls only.

**File: `packages/cli/src/commands/init.js`**
Risk: LOW — lodash used for array utilities only, not path-based operations.

### C.3 semver Code Path Review

**File: `packages/core/src/manifest.js`**
Risk: LOW — `semver.satisfies()` called with plugin manifest data. Plugin manifests currently sourced from internal registry only. Admin upload path exists but is low-traffic.

### C.4 axios Code Path Review

**File: `packages/api/src/services/http-client.js`**
Risk: LOW-MEDIUM — axios used for server-to-server calls. Request configuration constructed from static config. Interceptors add some dynamic configuration.
Note: `axios 1.6.6` prototype pollution issue is relevant if interceptor config contains attacker-controlled keys.

### C.5 express Code Path Review

**File: `packages/api/src/routes/auth.js`**
Risk: MEDIUM — `res.redirect()` used in OAuth callback with `return_url` query parameter. The `return_url` is sanitized with a allowlist check, but the underlying Express URL normalization bug could bypass this in edge cases.

---

---

## 16. Appendix D: Patching SLA Policy Reference

The following SLA policy governs remediation timelines for dependency vulnerabilities at MonoStack. It is reproduced here for reference.

### D.1 Severity Definitions and Deadlines

| Severity | CVSS Range | Remediation Deadline | Escalation Path |
|---|---|---|---|
| Critical | 9.0–10.0 | 7 days | Immediate escalation to CTO and Security Director; emergency patch process |
| High | 7.0–8.9 | 30 days | Escalation to Engineering Lead; tracked in security risk register |
| Medium | 4.0–6.9 | 90 days | Tracked in security risk register; addressed in next planned release cycle |
| Low | 0.1–3.9 | 180 days | Addressed in scheduled dependency update cycle |
| Informational | N/A | Next quarterly review | Noted in quarterly security review |

### D.2 SLA Status for This Audit's Findings

| Finding | CVSS | Severity | Identified | Deadline | Status as of 2024-03-22 |
|---|---|---|---|---|---|
| lodash CVE-2021-23337 | 7.2 | High | 2024-01-29 | 2024-02-28 | Past SLA — approved exception granted |
| semver CVE-2022-25883 | 5.3 | Medium | 2024-01-29 | 2024-04-29 | In progress — within SLA |
| axios CVE-2023-45857 | 6.5 | Medium | 2024-01-29 | 2024-04-29 | In progress — within SLA |
| express CVE-2024-29041 | 6.1 | Medium | 2024-01-29 | 2024-04-29 | In progress — within SLA |
| react fragmentation | N/A | N/A | 2024-01-29 | Best effort | In progress |

**Note on lodash SLA breach:** The 30-day High-severity SLA for lodash was breached (deadline 2024-02-28, implementation pending as of 2024-03-22). The breach was approved by the Security Director on 2024-03-01 based on the rationale that:
1. The multi-round stakeholder review process was the correct approach for a workspace-wide change
2. An emergency patch would have bypassed the compatibility and peer review checks
3. The risk of a rushed, untested override introducing a regression outweighed the incremental risk of the 22-day SLA breach
4. The lodash CVE requires an authenticated admin request to exploit, reducing urgency

The SLA breach is formally documented in the security risk register (entry R-01).

### D.3 Post-Remediation SLA Closure

After implementing the overrides from RFC-2024-047 and verifying via `npm audit`, the Security Guild will formally close all five risk register entries. The closure requires:
1. Evidence of the `npm audit` output showing `found 0 vulnerabilities`
2. Evidence of `npm ls <package>` showing the override version is installed
3. Sign-off from the implementing engineer on the implementation checklist (RFC-2024-047 Appendix H)

---

*End of Security Audit Report SEC-AUDIT-2024-Q1-MONOSTACK*

*Questions: Contact security-guild@monostack.internal or open a ticket tagged `security-audit`.*

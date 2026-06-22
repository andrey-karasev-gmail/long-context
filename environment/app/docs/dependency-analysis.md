# MonoStack Dependency Analysis Report

**Report ID:** DEP-ANALYSIS-2024-Q1  
**Prepared By:** Platform Team  
**Authors:** S. Okonkwo, T. Bergström  
**Date:** 2024-01-20  
**Related:** RFC-2024-047, SEC-AUDIT-2024-Q1-MONOSTACK  
**Status:** Reference Document — Superseded by RFC-2024-047 Section 12 for implementation targets

---

## Purpose

This document provides detailed technical analysis of the npm dependency tree for the MonoStack workspace. It was produced as background research for RFC-2024-047 and is referenced by the Security Audit Report (SEC-AUDIT-2024-Q1-MONOSTACK).

The analysis covers:
1. How each vulnerable package enters the dependency tree
2. Which MonoStack packages are affected
3. What the override mechanism does to the resolved tree
4. Compatibility analysis for each proposed override version

**IMPORTANT:** This document is historical research. The authoritative implementation targets are in **RFC-2024-047 Section 12**. Do not use version numbers from this analysis as implementation targets — some were revised in later rounds.

---

## 1. Workspace Structure

The MonoStack workspace is an npm workspaces monorepo. The root `package.json` defines the workspace:

```json
{
  "name": "monostack",
  "version": "0.0.0",
  "private": true,
  "workspaces": [
    "packages/core",
    "packages/api",
    "packages/ui",
    "packages/cli"
  ]
}
```

Each package has its own `package.json` with its own direct dependencies. npm hoists shared dependencies to the root `node_modules` where possible.

### 1.1 Package Overview

**`@monostack/core` (packages/core)**  
Core business logic utilities used by all other packages. Provides data transformation, configuration merging, and plugin management. Heavy user of lodash and semver.

**`@monostack/api` (packages/api)**  
Express-based REST API server. Uses axios for outbound HTTP calls, express for routing, and imports from `@monostack/core`.

**`@monostack/ui` (packages/ui)**  
React-based frontend application. Primary consumer of the React ecosystem. Uses `react-beautiful-dnd` for drag-and-drop and `react-query` for data fetching.

**`@monostack/cli` (packages/cli)**  
Command-line tooling for developers. Minimal dependencies. Does not use lodash, axios, or express directly.

---

## 2. lodash Dependency Analysis

### 2.1 How lodash Enters the Tree

lodash `4.17.20` enters the MonoStack workspace through two paths:

**Path A: `@monostack/core` → `data-processor@2.1.0` → `lodash@4.17.20`**

`data-processor` is a utility library for data transformation pipelines. It uses lodash extensively for array manipulation, object merging, and deep cloning. Its `package.json` declares:
```json
"dependencies": {
  "lodash": "^4.17.0"
}
```

The `^4.17.0` range resolves to the latest `4.17.x` available when `data-processor@2.1.0` was published (February 2022), which was `4.17.20`. The `^` range means "compatible with 4.17.0", allowing minor and patch updates but not major updates.

**Path B: `@monostack/api` → `config-merger@1.4.2` → `lodash@4.17.20`**

`config-merger` is a configuration management library. It uses `_.merge()` and `_.get()` for config object manipulation. Its `package.json` declares:
```json
"dependencies": {
  "lodash": "^4.17.15"
}
```

Both `data-processor` and `config-merger` end up requiring `lodash@4.17.20` (the same version), so npm hoists a single copy to `node_modules/lodash`.

### 2.2 Why npm Doesn't Automatically Update to `4.17.21`

A common question is: "Why doesn't `npm update` automatically update lodash to `4.17.21`?"

The answer is that `npm update` updates direct dependencies, not transitive dependencies. Since lodash is a transitive dependency (a dependency of `data-processor` and `config-merger`, not directly declared in any MonoStack package), `npm update` does not touch it.

Furthermore, `data-processor@2.1.0` declares `lodash: "^4.17.0"`. The `^` range does allow `4.17.21`. However, `data-processor@2.1.0` was published with a lockfile that pins lodash to `4.17.20`, and npm uses the lockfile for reproducible installs. The only way to get `4.17.21` without an override is for `data-processor` to publish a new version with an updated lockfile — which is not under MonoStack's control.

### 2.3 What the Override Does

Adding `"overrides": {"lodash": "4.17.21"}` to the root `package.json` instructs npm to resolve all `lodash` references to `4.17.21`, regardless of what individual packages request. After running `npm install`:

**Before override:**
```
node_modules/
└── lodash/           ← 4.17.20 (vulnerable)
```

**After override:**
```
node_modules/
└── lodash/           ← 4.17.21 (fixed)
```

No nested copies of lodash remain. All packages — including transitive packages — use `4.17.21`.

### 2.4 Compatibility Analysis: `4.17.20` → `4.17.21`

**API compatibility:** 100% backward compatible. This is a patch version bump. Only the `baseSet` function was modified (internally), and the modification only affects paths containing `__proto__`, `prototype`, or `constructor`. For all legitimate lodash usage, the behavior is identical.

**Breaking change risk:** Zero. The change rejects malicious inputs that no legitimate code would use.

**Peer dependency satisfaction:** All packages declaring `lodash@^4.17.x` are satisfied by `4.17.21`. Packages declaring `lodash@^4.0.0` or `lodash@^4.x.x` are also satisfied.

---

## 3. semver Dependency Analysis

### 3.1 How semver Enters the Tree

semver enters the MonoStack workspace through multiple paths:

**Path A: `@monostack/core` → `semver@7.5.3` (direct dependency)**

`@monostack/core/package.json` declares:
```json
"dependencies": {
  "semver": "^7.5.0"
}
```

This is unusual — `semver` is a direct dependency of `@monostack/core`, not a transitive dependency. The core package uses semver for plugin version compatibility checking.

**Path B: Multiple npm internal packages → `semver@7.5.4`**

npm's own internal tooling (packages like `@npmcli/package-json`, `@npmcli/config`, etc.) also depend on semver. These packages are installed as part of npm's own workspace tooling. They declare:
```json
"dependencies": {
  "semver": "^7.5.4"
}
```

This results in two different semver versions being installed:
- `node_modules/semver` → `7.5.3` (used by `@monostack/core`)
- `node_modules/@npmcli/package-json/node_modules/semver` → `7.5.4` (used by npm internals)

### 3.2 The Version Split Situation

The presence of both `7.5.3` and `7.5.4` is worth examining. `7.5.4` is newer and technically more patched than `7.5.3`. However, `7.5.3` is what MonoStack code uses (Path A above), and `7.5.4` is only used by npm's own internals.

The Security Guild's recommendation to use `7.6.x` is driven by the desire to use a version with fundamentally non-backtracking regex patterns, not just input length limits. This applies primarily to Path A — the MonoStack code path.

### 3.3 What the Override Does

Adding `"overrides": {"semver": "7.6.3"}` resolves all semver references to `7.6.3`.

**Before override:**
```
node_modules/
├── semver/           ← 7.5.3 (used by @monostack/core)
└── @npmcli/
    └── package-json/
        └── node_modules/
            └── semver/ ← 7.5.4 (used by npm internals)
```

**After override:**
```
node_modules/
└── semver/           ← 7.6.3 (all references resolve here)
```

This also affects npm's internal tooling — they will use `7.6.3` instead of `7.5.4`. Since `7.6.3` is backward compatible with `7.5.4`, this is safe.

### 3.4 Compatibility Analysis: `7.5.3` → `7.6.3`

**API compatibility:** Fully backward compatible. The semver package follows semver itself. All functions (`valid`, `satisfies`, `gt`, `lt`, `coerce`, etc.) have the same signatures and return the same values for all valid inputs.

**Behavior change for invalid inputs:** For inputs exceeding the length limit, `7.6.3` returns `null` or throws `TypeError` faster (immediately at function entry, not after partial parsing). This is strictly better behavior.

**Peer dependency satisfaction:** Packages declaring `semver@^7.0.0` are satisfied. Packages declaring `semver@^7.5.x` are satisfied. The `7.6.3` version satisfies all `^7.x.x` ranges.

---

## 4. axios Dependency Analysis

### 4.1 How axios Enters the Tree

axios enters the workspace as a direct dependency of `@monostack/api`:

**`@monostack/api` → `axios@1.3.4` (direct dependency)**

`@monostack/api/package.json` declares:
```json
"dependencies": {
  "axios": "^1.3.0"
}
```

The `^1.3.0` range means "compatible with 1.3.0", which allows updates to `1.x.x` where `x >= 3` and `x >= 0`. This range would technically allow `1.6.8` — but npm used the locked version `1.3.4` from `package-lock.json`.

**Why not just `npm update axios`?**

`npm update axios` would update axios to the latest version satisfying `^1.3.0`, which could be `1.6.8`. However:
1. It would only update it in `@monostack/api/package.json`'s direct dependencies
2. It would not affect transitive packages that independently require `axios`
3. Using an `override` is more explicit and applies workspace-wide

For this specific case (axios is a direct dependency with a wide version range), `npm update` would work. The override approach is chosen for consistency and explicitness.

### 4.2 Usage Analysis in `@monostack/api`

The `@monostack/api` package uses axios in `packages/api/src/services/http-client.js`:

```javascript
const axios = require('axios');

class HttpClient {
  constructor(baseURL, options = {}) {
    this.client = axios.create({
      baseURL,
      timeout: options.timeout || 10000,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });
    
    // Request interceptor for auth token injection
    this.client.interceptors.request.use((config) => {
      const token = TokenStore.get();
      if (token) {
        config.headers.Authorization = `Bearer ${token}`;
      }
      return config;
    });
    
    // Response interceptor for error normalization
    this.client.interceptors.response.use(
      (response) => response.data,
      (error) => {
        const normalized = normalizeError(error);
        return Promise.reject(normalized);
      }
    );
  }

  async get(path, params) {
    return this.client.get(path, { params });
  }

  async post(path, body) {
    return this.client.post(path, body);
  }
}
```

**Relevant security observations:**
- The request interceptor merges `config.headers` — if `options.headers` were attacker-controlled, the `mergeDeep` vulnerability in pre-`1.6.6` versions could be triggered
- `this.client.get()` follows redirects by default — the SSRF vulnerability in pre-`1.6.8` versions is relevant here if the service being called can return crafted redirects

### 4.3 What the Override Does

Adding `"overrides": {"axios": "1.6.8"}` forces all axios references to `1.6.8`.

**Before override:**
```
node_modules/
└── axios/            ← 1.3.4 (vulnerable)
```

**After override:**
```
node_modules/
└── axios/            ← 1.6.8 (patched)
```

### 4.4 Compatibility Analysis: `1.3.4` → `1.6.8`

**API compatibility:** The axios `1.x` API is stable. The main areas of concern:

*Request/response interceptor API:* Unchanged. Interceptors registered with `axios.interceptors.request.use()` and `axios.interceptors.response.use()` behave identically.

*CancelToken API:* Deprecated in `1.5.0` but not removed. MonoStack uses `CancelToken` in one test utility. The deprecated API still works in `1.6.8` (removal is planned for `2.0.0`).

*Default configuration:* `withCredentials` defaults to `false` — unchanged in `1.6.x`. No behavior change.

*TypeScript types:* Improved in `1.6.x`. No regressions.

*The XSRF-TOKEN behavior change:* The fix for CVE-2023-45857 changes when `X-XSRF-TOKEN` is sent. In server-side Node.js usage (MonoStack's case), this header was never being sent anyway (no cookie jar), so the behavior from MonoStack's perspective is identical.

*The mergeDeep change (1.6.6):* The fix for the prototype pollution issue changes how `mergeDeep` handles objects with `__proto__` keys. For all legitimate configuration objects (which never use `__proto__` as a key), behavior is identical.

*The redirect validation change (1.6.8):* The fix validates redirect target URLs more strictly. For well-formed responses (which all MonoStack's third-party services produce), behavior is identical. The change only affects crafted malicious redirect responses.

**Breaking change risk:** Very low. All of MonoStack's axios usage patterns are compatible with `1.6.8`.

---

## 5. express Dependency Analysis

### 5.1 How express Enters the Tree

express enters as a direct dependency of `@monostack/api`:

**`@monostack/api` → `express@4.18.2` (direct dependency)**

`@monostack/api/package.json` declares:
```json
"dependencies": {
  "express": "^4.18.0"
}
```

The `^4.18.0` range would technically allow `4.19.2` — but the lockfile pins it to `4.18.2`.

### 5.2 Express Dependency Tree

express itself brings in a significant sub-tree of transitive dependencies. Key packages in the express dependency tree:

| Package | Version | Purpose | Security Notes |
|---|---|---|---|
| body-parser | 1.20.2 | Request body parsing | No CVEs |
| debug | 2.6.9 | Debug logging | No CVEs |
| qs | 6.11.0 | Query string parsing | CVE-2022-24999 (fixed in 6.11.0) |
| path-to-regexp | 0.1.7 | URL routing | No CVEs in 0.1.x |
| cookie | 0.6.0 | Cookie parsing | No CVEs |
| methods | 1.1.2 | HTTP method list | No CVEs |
| send | 0.18.0 | Static file serving | No CVEs |

Note: The `qs` version `6.11.0` is installed by express and is patched (CVE-2022-24999 was a prototype pollution issue fixed in `6.10.3`). No additional action needed for `qs`.

### 5.3 What the Override Does

Adding `"overrides": {"express": "4.19.2"}` forces express to `4.19.2`.

**Cascading changes:** When express is updated from `4.18.2` to `4.19.2`, some of its own transitive dependencies may also update (since express's own `package-lock.json` changed). Specifically:
- `send` updates from `0.18.0` to `0.19.0`
- `serve-static` updates from `1.15.0` to `1.16.0`
- `encodeurl` updates from `1.0.2` to `2.0.0`
- `router` is added as a separate package (previously was part of express core)

These updates are expected and documented in the express `4.19.2` changelog. They are all backward compatible.

### 5.4 Compatibility Analysis: `4.18.2` → `4.19.2`

**API compatibility:** express follows semver. The `4.18` to `4.19` minor version bump is backward compatible with the following caveats:

*`res.redirect()` behavior:* The URL normalization change that fixes CVE-2024-29041 makes redirect URL handling stricter. Redirects to well-formed absolute URLs or relative paths work identically. Redirects to malformed URLs that `4.18.x` would tolerate may now be rejected or normalized differently.

*Testing required:* The auth callback redirect flow in `@monostack/api` should be tested with `4.19.2` to confirm the redirect URLs are well-formed and behave as expected. T. Bergström confirmed this testing was done (see M-06 notes).

*`router` as separate package:* Express `4.19.x` extracts the router into a separate `router` npm package. This is an internal change and does not affect the Express API surface.

**Breaking change risk:** Low. MonoStack's express usage is standard and does not rely on edge cases in URL handling.

---

## 6. react Dependency Analysis

### 6.1 How Multiple react Versions Enter the Tree

The react version fragmentation is a result of multiple packages declaring their own react peer dependencies with different version requirements.

**Installed react versions (before override):**

| Version | Location | Consumer |
|---|---|---|
| 18.2.0 | `node_modules/react` | `@monostack/ui` (direct dep, hoisted) |
| 18.0.0 | `node_modules/react-beautiful-dnd/node_modules/react` | `react-beautiful-dnd` (nested) |
| 18.1.0 | `node_modules/react-query/node_modules/react` | `react-query` (nested) |

**Why the nesting?**

npm workspace hoisting rules:
1. If only one version of a package is needed, hoist it to `node_modules/package-name`
2. If multiple versions are needed (different packages require incompatible versions), keep the version needed by fewer packages in a nested `node_modules`

In this case:
- `@monostack/ui` needs `react@18.2.0` (direct dep)
- `react-beautiful-dnd@13.1.1` declares `peerDependencies: {"react": "^16.8.3 || ^17 || ^18"}` and was installed against `react@18.0.0` at publish time
- `react-query@3.39.3` declares `peerDependencies: {"react": "^16.8.0 || ^17.0.0 || ^18.0.0"}` and was installed against `react@18.1.0` at publish time

The different peer dependency resolutions mean npm cannot hoist a single react version — it installs nested copies.

### 6.2 The Hook Invariant Violation Mechanism

React stores its hook dispatcher in a module-level variable inside the `react` package:
```javascript
// Simplified from React internals
let currentDispatcher = null;

function useState(initialState) {
  if (!currentDispatcher) {
    throw new Error('Invalid hook call...');
  }
  return currentDispatcher.useState(initialState);
}
```

When React renders a component, it sets `currentDispatcher` before calling the component function, then clears it after. The `currentDispatcher` is set in the `react` package instance that was used to render the component.

When `react-beautiful-dnd` (which uses its own `react@18.0.0` instance) calls `useState` inside a component rendered by `@monostack/ui`'s `react@18.2.0`, the following happens:
1. `@monostack/ui`'s `react@18.2.0` sets `currentDispatcher` before rendering
2. Inside the component, `react-beautiful-dnd` calls `useState` from `react@18.0.0`
3. `react@18.0.0`'s `currentDispatcher` is `null` (not the same instance as `react@18.2.0`)
4. The invariant check fails and throws "Invalid hook call"

### 6.3 What the Override Does

Adding `"overrides": {"react": "18.3.1"}` forces all react references to `18.3.1`.

**Before override:**
```
node_modules/
├── react/            ← 18.2.0
├── react-beautiful-dnd/
│   └── node_modules/
│       └── react/   ← 18.0.0 (causes hook violations)
└── react-query/
    └── node_modules/
        └── react/   ← 18.1.0 (causes hook violations)
```

**After override:**
```
node_modules/
└── react/            ← 18.3.1 (all references use this single instance)
```

All three packages now use the same React instance. The hook invariant violation is eliminated.

### 6.4 react-dom Is NOT Included

As documented in RFC-2024-047 Section 12.3, `react-dom` is explicitly excluded from the overrides block.

`react-dom@18.2.0` is a direct dependency in `@monostack/ui/package.json`. There is only one version of `react-dom` installed — no fragmentation. The explicit exclusion from overrides is important to document because an implementor might assume that overriding `react` implies overriding `react-dom`.

**After the override:**
- `node_modules/react` → `18.3.1`
- `node_modules/react-dom` → `18.2.0` (unchanged, direct dep in `@monostack/ui`)

The React team documents that `react` and `react-dom` versions can be mismatched within the same minor version family (18.x). `react@18.3.1` with `react-dom@18.2.0` is a supported configuration. The `react-dom` update to `18.3.1` (or later) will be handled via a separate direct dependency update in `@monostack/ui` (MONO-9047).

### 6.5 Compatibility Analysis: `18.2.0` → `18.3.1`

**API compatibility:** React follows semver. The `18.2` to `18.3` minor version bump is backward compatible. The additions in `18.3.x` are:
- Deprecation warnings for APIs that React 19 will remove (`ReactDOM.render()`, etc.)
- Bug fixes

**Deprecation warnings:** `@monostack/ui` uses `ReactDOM.render()` in its test harness. After upgrading to `18.3.1`, these calls will emit `console.error` deprecation warnings. The warnings do not break functionality — they are informational only. The Frontend Guild has filed MONO-9048 to update the test harness to use the new `createRoot()` API.

**Breaking change risk:** Very low for production code. The deprecation warnings will surface in test output and development but are not runtime errors.

---

## 7. Override Interaction Analysis

### 7.1 Do the Overrides Conflict With Each Other?

No. The five overrides are for entirely separate packages with no dependency relationships between them:
- `lodash` does not depend on `semver`, `axios`, `express`, or `react`
- `semver` does not depend on `lodash`, `axios`, `express`, or `react`
- `axios` does not depend on `lodash`, `semver`, `express`, or `react`
- `express` does not depend on `lodash`, `semver`, `axios`, or `react`
- `react` does not depend on `lodash`, `semver`, `axios`, or `express`

The overrides are completely independent and can be applied simultaneously without any interaction effects.

### 7.2 Do the Overrides Conflict With Any Direct Dependencies?

Checking each override against the direct dependencies declared in each MonoStack package:

**`@monostack/core` direct dependencies vs overrides:**
- `semver@^7.5.0` → override to `7.6.3` satisfies `^7.5.0` ✓

**`@monostack/api` direct dependencies vs overrides:**
- `axios@^1.3.0` → override to `1.6.8` satisfies `^1.3.0` ✓
- `express@^4.18.0` → override to `4.19.2` satisfies `^4.18.0` ✓

**`@monostack/ui` direct dependencies vs overrides:**
- `react@^18.2.0` → override to `18.3.1` satisfies `^18.2.0` ✓
- `react-dom@^18.2.0` → NOT overridden (stays at `18.2.0`) ✓

All overrides are within the declared version ranges of direct dependencies. No conflicts.

### 7.3 Do the Overrides Conflict With Peer Dependencies?

Checking peer dependency requirements of key packages against the overrides:

**Packages with peer dependency on `react@^18.x`:**
- `react-dom@18.2.0` — declares `peerDependencies: {"react": "^18.2.0"}` — `18.3.1` satisfies `^18.2.0` ✓
- `react-beautiful-dnd@13.1.1` — declares `peerDependencies: {"react": "^16.8.3 || ^17 || ^18"}` — `18.3.1` satisfies ✓
- `react-query@3.39.3` — declares `peerDependencies: {"react": "^16.8.0 || ^17.0.0 || ^18.0.0"}` — `18.3.1` satisfies ✓
- `@testing-library/react@14.x` — declares `peerDependencies: {"react": "^18.0.0"}` — `18.3.1` satisfies ✓

**Packages with peer dependency on `lodash@^4.x`:**
- `data-processor@2.1.0` — no peerDependency on lodash (it's a direct dep) — N/A ✓
- 14 other packages with `peerDependencies: {"lodash": "^4.0.0"}` — `4.17.21` satisfies ✓

**Packages with peer dependency on `express@^4.x`:**
- Several Express middleware packages — all declare `peerDependencies: {"express": "^4.0.0"}` — `4.19.2` satisfies ✓

**No peer dependency conflicts from any of the five overrides.**

---

## 8. npm install Behavior With Overrides

### 8.1 What npm Does During `npm install` With Overrides

When `npm install` is run after adding the `overrides` block:

1. **Resolution phase:** npm builds a dependency tree, resolving each package's requirements. When it encounters a package that matches an override key, it uses the override version instead of the normally-resolved version.

2. **Deduplication phase:** npm identifies packages that can be hoisted to a single location in `node_modules`. With overrides enforcing a single version of the overridden packages, deduplication is more aggressive — nested copies of the overridden packages are eliminated.

3. **Installation phase:** npm installs or updates packages as needed. Packages that changed version (due to overrides) are reinstalled.

4. **Lock file update:** `package-lock.json` is updated to reflect the new resolved versions.

### 8.2 Expected Output of `npm install`

When running `npm install` with the overrides block, expect to see output like:

```
added 5 packages, changed 12 packages, and audited 817 packages in 45s

npm warn overriding lodash@4.17.20 with lodash@4.17.21
npm warn overriding semver@7.5.3 with semver@7.6.3
npm warn overriding axios@1.3.4 with axios@1.6.8
npm warn overriding express@4.18.2 with express@4.19.2
npm warn overriding react@18.0.0 with react@18.3.1
npm warn overriding react@18.1.0 with react@18.3.1
npm warn overriding react@18.2.0 with react@18.3.1

found 0 vulnerabilities
```

The `npm warn overriding` messages confirm the override mechanism is working. The `found 0 vulnerabilities` confirms the CVEs have been remediated.

Note: React shows three `overriding` warnings because three different versions (18.0.0, 18.1.0, 18.2.0) were previously installed and are all now being overridden to 18.3.1.

### 8.3 package-lock.json Changes

The `package-lock.json` diff after applying the overrides will show:

1. Version changes for the five overridden packages at the root level
2. Removal of nested package entries for overridden packages (e.g., `react-beautiful-dnd/node_modules/react` entry is removed)
3. Version changes for transitive dependencies of the overridden packages (e.g., express's sub-dependencies)

The diff will be large — potentially hundreds of lines — due to the cascading version changes in the express and react dependency sub-trees. This is expected and is not a cause for concern.

---

## 9. Summary Table

| Package | Current | Override | Reason | Round Finalized | Key Gotchas |
|---|---|---|---|---|---|
| lodash | 4.17.20 | **4.17.21** | CVE-2021-23337 (prototype pollution) | Round 1 | Only 1 patch version; safe |
| semver | 7.5.3 | **7.6.3** | CVE-2022-25883 (ReDoS) + hardening | Round 3 (was 7.5.4 R1, 7.6.0 R2) | Do NOT use 7.5.4 (superseded) |
| axios | 1.3.4 | **1.6.8** | CVE-2023-45857 + prototype pollution + SSRF | Round 3 (was 1.4.0 R1, 1.6.0 R2) | Do NOT use 1.4.0 (rejected) or 1.6.0 (superseded) |
| express | 4.18.2 | **4.19.2** | CVE-2024-29041 (path traversal) | Round 2 | Do NOT use 4.18.3 (rejected) or 4.19.0/4.19.1 (incomplete fix) |
| react | 18.2.0 | **18.3.1** | Version fragmentation (hook violations) | Round 2 | Do NOT include react-dom; use 18.3.1 not 18.2.0 |

---

## 10. Verification Commands

After implementing the overrides and running `npm install`, use these commands to verify correctness:

```bash
# Verify lodash version
npm ls lodash
# Expected: monostack@... └─┬ lodash@4.17.21

# Verify semver version
npm ls semver
# Expected: all references pointing to 7.6.3

# Verify axios version
npm ls axios
# Expected: monostack@... └─┬ axios@1.6.8

# Verify express version
npm ls express
# Expected: monostack@... └─┬ express@4.19.2

# Verify react version (no nested copies)
npm ls react
# Expected: only react@18.3.1, no nested react entries

# Verify react-dom is NOT overridden (should still show 18.2.0)
npm ls react-dom
# Expected: react-dom@18.2.0

# Run npm audit to confirm no remaining CVEs
npm audit
# Expected: found 0 vulnerabilities
```

---

---

## 11. Historical Context: Why These Packages Accumulated Vulnerabilities

### 11.1 lodash: Three Years of Accumulation

The lodash `4.17.20` → `4.17.21` gap seems small — one patch version — but it represents three years of exposure. Understanding why this happened prevents recurrence.

**Timeline reconstruction:**

- **February 2021:** lodash `4.17.21` released, fixing CVE-2021-23337
- **March 2021:** The CVE is published publicly
- **April 2021:** `data-processor@2.1.0` published, pinning `lodash@4.17.20` in its lockfile
- **May 2021:** MonoStack adopts `data-processor@2.1.0` (see commit history)
- **January 2024:** MONO-8791 filed — lodash CVE identified

The 33-month gap between lodash `4.17.21`'s release and MonoStack's detection of the vulnerability has several contributing factors:

**Factor 1: The CVE predates the integration.**
When `data-processor@2.1.0` was published (April 2021) and when MonoStack adopted it (May 2021), the fix (`4.17.21`) already existed. However, `data-processor@2.1.0`'s lockfile pinned `4.17.20`. This is a common pattern: a package author publishes a new version after testing against a specific dependency version, and the lockfile captures that specific version.

**Factor 2: `npm audit` was not running in CI.**
MonoStack's CI pipeline did not run `npm audit` as a required check until Q4 2023. Had it been running, it would have flagged the vulnerability when `data-processor` was adopted.

**Factor 3: Transitive dependency invisibility.**
Direct dependency updates (e.g., `npm update axios`) are easy to track and review. Transitive dependency vulnerability tracking requires additional tooling (Snyk, Dependabot, `npm audit`) and was not systematically done.

**Factor 4: lodash 4.x is in maintenance mode.**
The absence of regular lodash releases in the `4.x` series reduces visibility. Teams often assume "no new releases = no new issues," when in fact the vulnerability predates the maintenance-mode period.

**Lessons:**
- Enable `npm audit` in CI as a required check
- Add Dependabot or similar automated dependency update tooling
- Establish a quarterly dependency review process (this RFC initiates that process)

---

### 11.2 semver: The "Already Mostly Fixed" Trap

The semver situation illustrates a different pattern: the installed version (`7.5.3`) technically satisfies the minimum CVE fix requirement (`7.5.2`), but is not the Security Guild's recommended version (`7.6.3`).

This creates a "already mostly fixed" trap where:
1. `npm audit` reports no vulnerability (because `7.5.3 >= 7.5.2`)
2. The Security Guild recommends a higher version for additional hardening
3. There's no automated tool that detects the gap between "minimum fix version" and "recommended version"

**Why `7.5.3` was installed:**
`@monostack/core` declared `semver@^7.5.0` in its `package.json`. When the dependency was first installed, `7.5.3` was the latest version that satisfied `^7.5.0`. The lockfile captured `7.5.3`. Subsequent `npm install` runs used the lockfile.

**Why the lockfile wasn't updated:**
Without a tool like Renovate or Dependabot creating automated PRs for patch version updates, patch-version updates are invisible to the team. There is no mechanism to detect that `7.5.3 → 7.5.4 → 7.6.x` releases have occurred unless someone runs `npm outdated` or has automated tooling.

---

### 11.3 axios: The "Close Enough" Version Problem

The axios situation is the most instructive failure mode: the Platform Team proposed `1.4.0` believing it was "newer than the existing `1.3.4` and therefore safer." This reasoning is fundamentally incorrect.

**The correct reasoning for CVE fix versions:**
1. Identify the CVE (CVE-2023-45857)
2. Find the CVE advisory (NVD, GitHub Advisory Database, or package's security advisory)
3. The advisory specifies the exact "fixed in" version (in this case, `1.6.0`)
4. The minimum acceptable target is the "fixed in" version or any later version
5. Any version BELOW the "fixed in" version is vulnerable, regardless of release date

**The incorrect reasoning applied in Round 1:**
The Platform Team looked at the CVE report date (November 2023) and reasoned that `1.4.0` (released April 2023) was "the latest stable version before the vulnerability was reported." This confuses the CVE publication date with the CVE fix date.

The vulnerability existed in code that was introduced long before the CVE was published. The fix was developed and included in `1.6.0` (October 2023) — before the CVE was even publicly disclosed. The CVE was assigned after the fix was already available, based on a security researcher's responsible disclosure.

**Correct workflow for finding the fix version:**
```
Step 1: Find the CVE ID (e.g., CVE-2023-45857)
Step 2: Check the GitHub Advisory Database: https://github.com/advisories/GHSA-wf5p-g6vw-rhxx
Step 3: The advisory lists "patched versions": ">=1.6.0"
Step 4: Use 1.6.0 or later. In our case, 1.6.8 (latest in series).
```

---

### 11.4 express: The "Intermediate Version" Confusion

The express situation illustrates the confusion that arises when a vulnerability's fix lands in an unexpected version number and rapid-fire patch releases complicate the picture.

The team initially proposed `4.18.3` — a minor increment from the existing `4.18.2`. The reasoning: "we're upgrading express anyway, let's go to the latest `4.18.x`." This reasoning would have left the vulnerability unaddressed.

Express's CVE advisory is unambiguous: "fixed in 4.19.2." The jump from `4.18.x` to `4.19.2` felt like a bigger change than a patch release, which may have contributed to the team looking for a `4.18.x` fix that didn't exist.

**Why `4.18.3` has no fix:**
CVE-2024-29041 was not discovered (or at least not publicly known) when `4.18.3` was developed and released. The fix required a significant change to the URL normalization logic in Express, which the maintainers implemented in the `4.19.x` series.

**The `4.19.0`/`4.19.1`/`4.19.2` confusion:**
The rapid-fire release of `4.19.0`, `4.19.1`, and `4.19.2` on the same or adjacent calendar days created confusion. Teams scanning for "the fix version" might find `4.19.0` first and assume it's sufficient. The Express security advisory is clear: only `4.19.2` is confirmed as the full fix.

---

## 12. Long-Term Dependency Governance Recommendations

This RFC addresses five specific vulnerabilities. The Security Guild and Platform Team used this RFC as an opportunity to recommend process improvements that would prevent similar accumulations in the future.

### 12.1 Enable Automated Dependency Scanning

**Recommendation:** Add `npm audit` to the CI pipeline as a required check that fails the build on High severity findings.

```yaml
# Example GitHub Actions step
- name: Security audit
  run: npm audit --audit-level=high
```

This would have caught the lodash vulnerability (High severity) before it accumulated for three years.

**Caveat:** `npm audit` only catches vulnerabilities that are in the npm advisory database. The database has some lag after CVE publication. Additional tooling (Snyk, GitHub Dependabot) provides more comprehensive coverage.

### 12.2 Enable Dependabot or Renovate

**Recommendation:** Enable GitHub Dependabot or Renovate to automatically create PRs for dependency updates, including patch-version updates to transitive dependencies.

Dependabot configuration (`.github/dependabot.yml`):
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    groups:
      security-updates:
        applies-to: security-updates
        update-types:
          - "patch"
          - "minor"
```

With this configuration, Dependabot would have automatically created a PR for `lodash@4.17.20 → 4.17.21` in February 2021, when the fix was released.

### 12.3 Establish Quarterly Dependency Review

**Recommendation:** Conduct a quarterly dependency review meeting (30-60 minutes) to:
1. Review `npm audit` output
2. Review `npm outdated` output for direct dependencies
3. Check for any security advisories for key packages not covered by `npm audit`
4. Identify packages that need attention

This RFC is the genesis of that process. The Platform Team will schedule Q2 2024 dependency review for May 2024.

### 12.4 Document Override Rationale

**Recommendation:** When an override is added to the root `package.json`, include a comment documenting the reason, the relevant ticket or RFC, and the date added.

Example (note: npm `package.json` does not support comments in JSON; use a companion file or the RFC reference):
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

The companion documentation (this RFC, RFC-2024-047) serves as the rationale record.

### 12.5 Establish an Override Review Process

**Recommendation:** Overrides should be reviewed quarterly to determine if they are still necessary:

1. Check if the direct dependency that caused the transitive vulnerability has been updated to declare the fixed version itself
2. If so, the override may be removable (test by removing it and running `npm ls <package>` to verify the resolved version)
3. Document when each override is removed and why

An override that is no longer necessary becomes maintenance noise. Keeping the override list minimal and documented reduces cognitive overhead for future engineers.

---

## 13. Reference: Version Selection Decision Tree

Use this decision tree when selecting a target version for an npm override or direct dependency upgrade to address a CVE:

```
1. Identify the CVE ID
   └─► Find the official CVE advisory (NVD, GitHub Advisory, or package's SECURITY.md)

2. Read the advisory's "Patched Versions" or "Fixed In" field
   └─► This is the MINIMUM acceptable version. Never target a version below this.

3. Check if the "minimum fix version" is still the latest in its series
   ├─► If YES: use that version
   └─► If NO: check if later versions in the same series have additional security patches
       ├─► If YES: use the latest in the series that has no known issues
       └─► If NO: you can use the minimum fix version, but the latest is still preferred

4. Verify backward compatibility of your selected version
   └─► Read the changelog between your current version and the target version
       ├─► Any breaking changes listed? → test affected code paths
       └─► No breaking changes? → proceed with confidence

5. Do a final check at implementation time
   └─► Has a newer patch been released since you selected your target version?
       ├─► If YES and it includes security patches: update your target
       └─► If NO: proceed with your selected version
```

This decision tree, applied to the five RFC-2024-047 packages:

| Package | Step 2 (minimum) | Step 3 (later security patches?) | Final Target |
|---|---|---|---|
| lodash | 4.17.21 | No (4.17.21 is latest) | **4.17.21** |
| semver | 7.5.2 | Yes (7.6.x has non-backtracking regex) → 7.6.3 is latest in series | **7.6.3** |
| axios | 1.6.0 | Yes (1.6.3 null-byte, 1.6.6 prototype pollution, 1.6.8 SSRF) | **1.6.8** |
| express | 4.19.2 | No (4.19.2 is the specific confirmed fix; later versions aren't yet released) | **4.19.2** |
| react | N/A (no CVE) | N/A | **18.3.1** (stability + deprecation visibility) |

---

---

## 14. Quick Reference: Rejected and Superseded Versions

This section provides a consolidated reference for any version that was **considered but must NOT be used**. If you are implementing RFC-2024-047 and you see one of these versions in a proposal, stop — you are reading an outdated section of the RFC.

### 14.1 Explicitly Rejected Versions (Security Guild formal rejection)

| Package | Rejected Version | Why Rejected | Correct Version |
|---|---|---|---|
| axios | 1.4.0 | Does NOT fix CVE-2023-45857. Vulnerable range is 0.21.1–1.5.x. 1.4.0 is inside the vulnerable range. | **1.6.8** |
| express | 4.18.3 | Does NOT fix CVE-2024-29041. Only 4.19.2 contains the confirmed full fix. | **4.19.2** |
| express | 4.19.0 | Incomplete fix for CVE-2024-29041. Bypass discovered immediately after release. | **4.19.2** |
| express | 4.19.1 | Incomplete fix for CVE-2024-29041. Attempted to close bypass from 4.19.0, still incomplete. | **4.19.2** |

### 14.2 Superseded Versions (Correct but replaced by a better option in a later round)

| Package | Superseded Version | Round Superseded | Replacement | Reason for Replacement |
|---|---|---|---|---|
| semver | 7.5.4 | Round 1 → Round 2 | 7.6.0 | Security Guild recommended 7.6.x for stronger non-backtracking regex fix |
| semver | 7.6.0 | Round 2 → Round 3 | 7.6.3 | 7.6.3 released between rounds with additional input length enforcement |
| axios | 1.6.0 | Round 2 → Round 3 | 1.6.8 | 1.6.6 (prototype pollution) and 1.6.8 (SSRF in redirects) released between rounds |
| react | 18.2.0 | Round 1 → Round 2 | 18.3.1 | Frontend Guild recommended 18.3.1 for deprecation warning visibility |

### 14.3 Out-of-Scope Versions (Not rejected, just not part of this RFC)

| Package | Version | Reason Not Included |
|---|---|---|
| webpack | any | No CVE; deferred to Q3 2024 by Frontend Guild request |
| babel | any | No CVE; deferred to Q3 2024 alongside webpack |
| eslint | any | devDependency only; no production risk |
| react-dom | any | Not fragmented; direct dep update handled in MONO-9047 |

---

## 15. File Inventory: What Is in `/app/docs`

This directory (`/app/docs`) contains the following reference documents for RFC-2024-047:

| File | Purpose | Authoritative For |
|---|---|---|
| `dependency-migration-rfc.md` | **Primary RFC document** | Implementation targets (Section 12 is the authoritative source) |
| `security-audit-report.md` | Security Guild's full audit findings | CVE details, exploitability assessment, remediation recommendations |
| `meeting-notes.md` | Complete meeting notes for all review sessions | Decision history, stakeholder discussions, rationale for each choice |
| `dependency-analysis.md` | Technical analysis of the npm dependency tree | How each package enters the tree, what overrides do, compatibility analysis |

**For implementation:** Read RFC-2024-047 Section 12. All other documents are context and rationale.

---

*End of Dependency Analysis Report DEP-ANALYSIS-2024-Q1*

*Note: This document is background research. Consult RFC-2024-047 Section 12 for the authoritative implementation targets.*

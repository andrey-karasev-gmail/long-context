# RFC-2024-047 Review Meeting Notes

**Document Purpose:** Complete meeting notes for all review sessions related to RFC-2024-047: MonoStack Dependency Security Migration.  
**Maintained By:** Platform Team (J. Hartwell)  
**Last Updated:** 2024-03-22  

---

## Meeting Index

| Meeting # | Date | Type | Key Outcome |
|---|---|---|---|
| M-01 | 2024-01-08 | Kickoff | RFC scope defined, authors assigned |
| M-02 | 2024-01-17 | Round 1 Draft Review | Round 1 proposals drafted, security review requested |
| M-03 | 2024-01-25 | Security Guild Review Call | Round 1 findings reviewed, 2 proposals rejected |
| M-04 | 2024-02-05 | Frontend Guild Consultation | React version recommendation updated to 18.3.1 |
| M-05 | 2024-02-12 | Frontend Guild Formal Review | Frontend Guild sign-off obtained |
| M-06 | 2024-02-16 | Round 2 Proposal Review | Round 2 proposals finalized |
| M-07 | 2024-02-19 | DevOps Review Call | DevOps sign-off obtained |
| M-08 | 2024-03-01 | RFC Finalization Pre-Check | Pre-implementation checklist reviewed |
| M-09 | 2024-03-15 | Round 3 Final Check | semver and axios versions updated for final release |
| M-10 | 2024-03-22 | RFC Sign-Off | Final RFC approved for implementation |

---

## Meeting M-01: Kickoff (2024-01-08)

**Date:** 2024-01-08  
**Time:** 10:00–10:45 AM  
**Location:** Video call (#platform-eng-meetings)  
**Attendees:**
- J. Hartwell (Platform Team, chair)
- S. Okonkwo (Platform Team)
- T. Bergström (Platform Team)
- A. Patel (Security Guild, invited observer)

**Agenda:**
1. Review MONO-8791 (lodash CVE report)
2. Scope additional dependency audit
3. Assign RFC authorship
4. Agree on review process and timeline

---

### M-01 Notes

**Item 1: MONO-8791 Review**

J. Hartwell opened by summarizing the lodash CVE (CVE-2021-23337) as reported in MONO-8791. The ticket was filed by D. Kowalczyk from the Frontend Guild after a routine dependency check caught the version discrepancy. The vulnerability allows prototype pollution via `_.merge()` and related functions.

S. Okonkwo noted that the team has been running `lodash@4.17.20` for approximately 18 months. The CVE fix (`4.17.21`) was released in February 2021, meaning the team has been vulnerable for nearly three years without detecting it.

T. Bergström: "We should treat this as a learning moment. If we missed a high-severity CVE for three years, what else have we missed? I'd recommend we scope this broader than just lodash."

J. Hartwell agreed and proposed expanding the scope to a full Q1 dependency audit.

**Item 2: Scope of Audit**

The team agreed to audit all production dependencies for:
- Known CVEs (via `npm audit` and Snyk)
- Version fragmentation (multiple versions of the same package installed simultaneously)
- Packages that are significantly out of date without a documented reason

A. Patel offered the Security Guild's participation as both reviewers and as co-authors for the security analysis sections. The offer was accepted.

**Item 3: RFC Structure**

J. Hartwell proposed writing a formal RFC to document:
- The current vulnerable state
- Proposed fix versions for each affected package
- The rationale behind each version choice
- The review process (multiple rounds with multiple stakeholders)

A. Patel recommended against rushing to a single round of proposals: "These fixes are going on the entire monorepo. We want multiple eyes, and we want to make sure we're proposing the actual fix versions, not just 'newer versions.' The Security Guild has seen teams propose patches that didn't actually fix the CVE."

**Action items from M-01:**
- [ ] J. Hartwell to create RFC-2024-047 in Confluence (DONE)
- [ ] S. Okonkwo to draft lodash and semver sections (DONE)
- [ ] T. Bergström to draft axios and express sections (DONE)
- [ ] J. Hartwell to coordinate react analysis with Frontend Guild (DONE)
- [ ] A. Patel to schedule Security Guild review for week of Jan 22 (DONE)

---

## Meeting M-02: Round 1 Draft Review (2024-01-17)

**Date:** 2024-01-17  
**Time:** 2:00–3:00 PM  
**Location:** Video call (#platform-eng-meetings)  
**Attendees:**
- J. Hartwell (chair)
- S. Okonkwo
- T. Bergström

**Agenda:**
1. Review Round 1 draft proposals
2. Identify open questions
3. Prepare security review request

---

### M-02 Notes

**Item 1: lodash Proposal Review**

S. Okonkwo walked through the lodash section. The proposal of `4.17.21` is straightforward — it is the one patch version that fixes the CVE, and there are no other versions in the `4.17.x` series.

**Consensus:** lodash `4.17.21` — approved for security review. No changes needed.

**Item 2: semver Proposal Review**

S. Okonkwo presented the semver analysis. The current installed version is `7.5.3`. The CVE (CVE-2022-25883) was fixed in `7.5.2`, which is older than the installed version. However, S. Okonkwo noted that the initial proposal was `7.5.4` based on it being the latest in the `7.5.x` series at the time the RFC was drafted.

T. Bergström raised a question: "If `7.5.3` already has the fix, why are we proposing an override at all? Shouldn't we be proposing `7.6.x` if we're going to override?"

S. Okonkwo: "The scanner still flags `7.5.3` because the CVE advisory lists the fix as `7.5.2` minimum. Some organizations interpret this as 'must be at or above 7.5.2.' Since we're already at `7.5.3`, we technically satisfy that. But the Security Guild's recommendation was to go higher, to the latest `7.6.x` for additional hardening. Let's let the Security Guild weigh in on whether `7.5.4` is sufficient or if they want `7.6.0+`."

**Consensus:** Defer to Security Guild. Propose `7.5.4` in Round 1 but flag for their input on `7.6.x`.

**Item 3: axios Proposal Review**

T. Bergström presented the axios analysis. The proposal was `1.4.0`.

J. Hartwell immediately flagged a concern: "How did we arrive at `1.4.0`? The CVE report I read says the fix was introduced in `1.6.0`."

T. Bergström (reviewing notes): "Ah, I think I misread the CVE advisory. I was looking at the 'affected versions' range and assumed anything newer than the oldest affected version would be safe. But that's not how it works — the fix version matters."

S. Okonkwo: "So `1.4.0` is within the vulnerable range? From `0.21.1` to `1.5.x`?"

T. Bergström: "Yes, `1.4.0` is vulnerable. We'd need to propose at minimum `1.6.0`. I'll update the proposal to `1.6.0` and flag the Round 1 version as a mistake in the RFC history."

J. Hartwell: "Actually, let's leave the `1.4.0` proposal in the RFC as a learning example. The RFC will go through multiple rounds anyway. Having an explicit rejected proposal will help future readers understand why we're at `1.6.x` and not a lower version."

**Consensus:** Update Round 1 to note `1.4.0` is disputed. T. Bergström to revise and send to Security Guild for formal confirmation.

*Note: The Round 1 proposals in Section 4 of RFC-2024-047 document the `1.4.0` proposal as originally submitted. The Security Guild's formal rejection is in Section 8.2.*

**Item 4: express Proposal Review**

T. Bergström presented the express analysis. The proposal was `4.18.3`.

J. Hartwell: "Is `4.18.3` the version that fixes CVE-2024-29041?"

T. Bergström: "That's what I initially assumed. `4.18.3` is newer than our current `4.18.2`. But when I look more carefully at the CVE advisory, it says 'fixed in 4.19.2.' So we'd need to jump from `4.18.x` to `4.19.x`."

J. Hartwell: "That's a bigger version jump. Any concerns about the minor version bump?"

T. Bergström: "I'll run a quick compatibility check. Express follows semver, so a minor version bump shouldn't be breaking. But let's confirm."

S. Okonkwo: "Same situation as axios — the `4.18.3` proposal should stay in the RFC as a historical artifact so the implementation engineer doesn't wonder why we're not using `4.18.3`."

**Consensus:** Flag `4.18.3` as a likely-incorrect proposal. Request Security Guild confirmation of required version. Preserve `4.18.3` in RFC history with explicit rejection note.

**Item 5: react Proposal Review**

J. Hartwell presented a brief summary of the react fragmentation issue from MONO-8841. The proposal of `18.2.0` was to standardize on the version already in `@monostack/ui`.

T. Bergström: "Should we coordinate with Frontend Guild before we finalize this?"

J. Hartwell: "Yes. I'll schedule a call with M. Chen this week."

**Consensus:** Proceed with `18.2.0` as a tentative Round 1 proposal. Frontend Guild consultation scheduled.

**Action items from M-02:**
- [ ] T. Bergström to update axios section to note `1.4.0` is disputed (DONE)
- [ ] T. Bergström to run express compatibility analysis for `4.19.x` (DONE)
- [ ] J. Hartwell to send formal security review request to Security Guild (DONE — sent 2024-01-24)
- [ ] J. Hartwell to schedule Frontend Guild consultation re: react version (DONE — M-04)

---

## Meeting M-03: Security Guild Review Call (2024-01-25)

**Date:** 2024-01-25  
**Time:** 11:00 AM – 12:15 PM  
**Location:** Video call  
**Attendees:**
- A. Patel (Security Guild Lead, chair)
- B. Fernandez (Security Guild)
- C. Yamamoto (Security Guild)
- J. Hartwell (Platform Team)
- T. Bergström (Platform Team)

**Agenda:**
1. Security Guild findings walkthrough (B. Fernandez presenting)
2. Platform Team questions and clarifications
3. Agree on Round 2 version targets

---

### M-03 Notes

**Item 1: Security Guild Findings Walkthrough**

B. Fernandez presented the Security Guild's audit findings (full report: SEC-AUDIT-2024-Q1-MONOSTACK).

*lodash — confirmed:*
B. Fernandez: "lodash `4.17.21` is confirmed as the fix version. The fix addresses prototype pollution in `baseSet` which underlies `zipObjectDeep`, `_.set`, `_.merge`, and related functions. Our testing confirms the fix is effective. No change needed from the Round 1 proposal."

*semver — escalated:*
B. Fernandez: "The Round 1 proposal of `7.5.4` technically satisfies the minimum bar for CVE-2022-25883. However, the `7.6.x` series rewrote the regex patterns to be non-backtracking, which is a more fundamental fix. We recommend targeting `7.6.x`. At the time of our audit, `7.6.0` is the latest. The Platform Team should do a final check before implementation in case a newer patch releases."

T. Bergström: "So you're recommending `7.6.0` now, with a note to re-check at implementation time?"

B. Fernandez: "Exactly. We want you on the latest `7.6.x` when you actually apply the fix."

*axios — Round 1 proposal rejected:*
C. Yamamoto: "The Round 1 proposal of `1.4.0` does not fix CVE-2023-45857. The CVE affects versions `0.21.1` through `1.5.x`. `1.4.0` is squarely in the vulnerable range. This is a clear rejection."

T. Bergström: "Confirmed. We caught this in our own review as well. What version do you recommend?"

C. Yamamoto: "Minimum `1.6.0` to fix the original CVE. But I'd encourage you to look at the full `1.6.x` changelog — there are additional security-relevant patches in `1.6.3`, `1.6.6`, and others. We'll formally recommend the latest `1.6.x` at the time of final implementation."

J. Hartwell: "For Round 2 purposes, let's say `1.6.0` as the minimum, and we'll update to the latest `1.6.x` in Round 3."

**Consensus:** Round 2 axios target: `1.6.0` (minimum), to be updated in Round 3.

*express — Round 1 proposal rejected:*
A. Patel: "The `4.18.3` proposal is rejected. CVE-2024-29041 requires `4.19.2` specifically. We want to be very clear about this: `4.18.3`, `4.19.0`, and `4.19.1` are all unacceptable. Only `4.19.2`."

J. Hartwell: "What's the situation with `4.19.0` and `4.19.1`? Were they ever released?"

A. Patel: "Yes. The Express maintainers had an unusual situation where they published `4.19.0`, immediately discovered a bypass in the fix, and published `4.19.1` and `4.19.2` in rapid succession. The confusing part is that `4.19.0` and `4.19.2` were published on the same calendar day. `4.19.2` is the confirmed final fix."

T. Bergström: "Why did the CVE even list `4.19.2` as the fix version if `4.19.0` came out first?"

A. Patel: "The CVE was formally published on 2024-03-25, the same day `4.19.2` was released. The CVE advisory team and the Express maintainers coordinated to ensure the fix version in the advisory was the correct one, not the intermediate incomplete versions."

**Consensus:** Round 2 express target: `4.19.2`.

*react — no security objection:*
A. Patel: "We have no security CVE for react. The fragmentation issue is a runtime stability concern. We defer to the Frontend Guild on version selection. The only note from us is that `18.3.x` introduces deprecation warnings for legacy APIs — that may or may not be desirable for the team right now."

J. Hartwell: "We'll coordinate with Frontend Guild. They've already been consulted informally and are leaning toward `18.3.1`."

**Item 2: Platform Team Questions**

J. Hartwell: "One question on the overall approach: we're proposing to use npm workspace overrides. Any security concerns with that mechanism itself?"

A. Patel: "Overrides are fine. They're an official npm feature designed exactly for this use case. The only risk is if an override forces a version that introduces a regression — but that's covered by your compatibility testing."

T. Bergström: "What about the risk of overrides being silently broken by an `npm update` in the future?"

A. Patel: "npm `update` doesn't touch the `overrides` field. You're safe. You'd have to manually remove the overrides for them to stop working."

**Item 3: Round 2 Version Agreement**

Summary of Round 2 targets agreed in this meeting:

| Package | Round 1 | Round 2 |
|---|---|---|
| lodash | 4.17.21 | 4.17.21 (no change) |
| semver | 7.5.4 | 7.6.0 (updated) |
| axios | 1.4.0 (rejected) | 1.6.0 (corrected) |
| express | 4.18.3 (rejected) | 4.19.2 (corrected) |
| react | 18.2.0 (pending) | TBD (Frontend Guild) |

**Action items from M-03:**
- [ ] J. Hartwell to update RFC Section 6 with Round 2 proposals (DONE)
- [ ] J. Hartwell to update Section 8 with formal rejection rationale for axios 1.4.0 and express 4.18.3 (DONE)
- [ ] A. Patel to send formal Security Guild approval email after meeting (DONE — 2024-01-29)
- [ ] T. Bergström to schedule DevOps review after React version is confirmed (DONE)

---

## Meeting M-04: Frontend Guild Consultation (2024-02-05)

**Date:** 2024-02-05  
**Time:** 3:30–4:15 PM  
**Location:** Video call  
**Attendees:**
- M. Chen (Frontend Guild Lead)
- D. Kowalczyk (Frontend Guild)
- R. Nakamura (Frontend Guild)
- J. Hartwell (Platform Team)

**Agenda:**
1. Review react fragmentation issue (MONO-8841)
2. Confirm react version recommendation
3. Discuss react-dom handling

---

### M-04 Notes

**Item 1: react Fragmentation Issue Review**

J. Hartwell summarized the react version fragmentation situation: three different React versions installed across the workspace, causing hook invariant violations for ~15% of users in production (MONO-8841).

M. Chen: "MONO-8841 has been in our backlog for two months. We diagnosed the root cause — `react-beautiful-dnd` and `react-query` each pulling their own React instance — but didn't have a clean fix. The `overrides` approach in the RFC is exactly what we need."

D. Kowalczyk: "The drag-and-drop issue is real and impactful. We've had support tickets about it. It fails intermittently, which makes it hard for users to report and hard for us to reproduce."

R. Nakamura: "The intermittency is because it depends on module load order, which varies by user's browser cache state."

**Item 2: React Version Recommendation**

J. Hartwell: "The current RFC proposal is `18.2.0` — standardizing on the version already in `@monostack/ui`. Is that what you'd recommend?"

M. Chen: "Not quite. I'd recommend `18.3.1` for two reasons. First, `18.3.x` introduces deprecation warnings for APIs that React 19 will remove. We need visibility into that. Second, if we're going to override React at the workspace level, let's do it once and do it right — using `18.3.1` means we won't need to redo this override in six months when React 19 is on the horizon."

D. Kowalczyk: "Agreed on 18.3.1. The deprecation warnings will require some work — `@monostack/ui` uses `ReactDOM.render()` in its test harness — but that work needs to happen before React 19 anyway."

J. Hartwell: "Any concerns about compatibility with `18.3.1`?"

R. Nakamura: "One thing: `18.3.0` had a regression in SSR scenarios. `18.3.1` fixes that. We don't currently use SSR, but I'd recommend `18.3.1` over `18.3.0` regardless, just to be safe."

M. Chen: "Confirmed. Frontend Guild recommendation is `18.3.1`. Not `18.3.0`, not `18.2.0`."

**Item 3: react-dom**

J. Hartwell: "Should `react-dom` also be in the overrides block?"

M. Chen: "No. `react-dom` is a direct dependency in `@monostack/ui/package.json` at version `18.2.0`. It's not fragmented — only one version is installed. We'll update `react-dom` directly in `@monostack/ui` as part of a separate cleanup ticket (MONO-9047). Don't include `react-dom` in the overrides."

D. Kowalczyk: "Putting `react-dom` in overrides when it's already a direct dep would create a confusing situation where the override and the direct dep disagree on version. Better to handle it cleanly with a direct dep update."

J. Hartwell: "Understood. react-dom explicitly excluded from the RFC overrides. I'll add a note to Section 12.3."

**Action items from M-04:**
- [ ] J. Hartwell to update RFC react proposal to `18.3.1` (DONE)
- [ ] J. Hartwell to add explicit exclusion of `react-dom` from overrides in RFC Section 12.3 (DONE)
- [ ] M. Chen to file MONO-9047 for react-dom direct dep update in @monostack/ui (DONE)
- [ ] D. Kowalczyk to file MONO-9048 for ReactDOM.render() deprecation cleanup in test harness (DONE)

---

## Meeting M-05: Frontend Guild Formal Review (2024-02-12)

**Date:** 2024-02-12  
**Time:** 10:00–10:30 AM  
**Location:** Slack huddle (#frontend-guild)  
**Attendees:**
- M. Chen (Frontend Guild Lead, chair)
- D. Kowalczyk
- R. Nakamura
- J. Hartwell (Platform Team, presenting)

**Agenda:**
1. Formal review of RFC-2024-047 react section
2. Guild sign-off vote

---

### M-05 Notes

J. Hartwell walked through the updated RFC sections covering react (`18.3.1` and explicit exclusion of `react-dom`).

M. Chen confirmed the Frontend Guild review notes from 2024-02-05 had been accurately captured in Section 9 of the RFC.

**Vote:**
- M. Chen: Approve
- D. Kowalczyk: Approve
- R. Nakamura: Approve with comment

R. Nakamura's comment: "I want to make sure Section 7.5 accurately captures that `react@18.3.1` and `react-dom@18.2.0` being mismatched is temporary and acceptable, not a permanent intended state. The RFC should note that MONO-9047 will bring `react-dom` in line."

J. Hartwell confirmed this is noted in Section 7.5 of the RFC. R. Nakamura changed vote to Approve.

**Result:** Frontend Guild sign-off obtained. 3-0 vote.

---

## Meeting M-06: Round 2 Proposal Review (2024-02-16)

**Date:** 2024-02-16  
**Time:** 1:00–1:45 PM  
**Location:** Video call  
**Attendees:**
- J. Hartwell (chair)
- S. Okonkwo
- T. Bergström

**Agenda:**
1. Review Round 2 proposals with all stakeholder input incorporated
2. Finalize for DevOps review
3. Assign implementation ticket

---

### M-06 Notes

J. Hartwell opened by reviewing the consolidated Round 2 proposals:

| Package | Round 2 Target | Status |
|---|---|---|
| lodash | 4.17.21 | Confirmed — no change since Round 1 |
| semver | 7.6.0 | Updated — Security Guild recommended 7.6.x |
| axios | 1.6.0 | Corrected — Round 1 proposal rejected |
| express | 4.19.2 | Corrected — Round 1 proposal rejected |
| react | 18.3.1 | Updated — Frontend Guild recommended 18.3.1 |

S. Okonkwo: "Are we all comfortable with these versions? Any last concerns before we go to DevOps?"

T. Bergström: "I want to confirm one thing on express. We're going from `4.18.2` to `4.19.2`. That's a minor version bump with some breaking potential. Can you walk through what actually changed?"

S. Okonkwo: "I reviewed the Express 4.18 to 4.19 migration notes. The minor version bump introduced one behavior change: the `res.redirect()` URL normalization. That's the change that also fixes CVE-2024-29041. The normalization is stricter in `4.19.x` — some malformed URLs that `4.18.x` would redirect to now result in an error or different redirect target. For well-formed redirect targets (which MonoStack uses exclusively), there's no behavior change."

T. Bergström: "Confirmed in my testing. The OAuth callback redirect tests pass with `4.19.2`."

J. Hartwell: "Good. Let's proceed to DevOps review. T. Bergström, can you send the review request today?"

T. Bergström confirmed and sent the DevOps review request (see Appendix E.4 of RFC-2024-047 for the email text).

**Action items from M-06:**
- [ ] T. Bergström to send DevOps review request (DONE — 2024-02-16)
- [ ] J. Hartwell to update RFC Section 6 with final Round 2 proposals (DONE)
- [ ] S. Okonkwo to create implementation ticket MONO-9001 (DONE)

---

## Meeting M-07: DevOps Review Call (2024-02-19)

**Date:** 2024-02-19  
**Time:** 2:30–3:00 PM  
**Location:** Video call  
**Attendees:**
- F. O'Sullivan (DevOps Lead, chair)
- G. Martinez (DevOps)
- T. Bergström (Platform Team)
- J. Hartwell (Platform Team)

**Agenda:**
1. DevOps review of RFC-2024-047 Round 2 proposals
2. CI/CD impact assessment
3. Approval vote

---

### M-07 Notes

**Item 1: Round 2 Proposal Review**

F. O'Sullivan: "We reviewed the RFC over the last two days. A few points:

First, the `npm install` behavior. When you add an `overrides` block, npm will restructure `node_modules` on the next install. The `package-lock.json` will change significantly. The PR for this change will have a large lockfile diff. That's expected and fine, but whoever reviews the PR should know to expect a large diff.

Second, Docker build cache. The `COPY package.json ...` step in the Dockerfile uses the `package.json` content hash for cache invalidation. Adding `overrides` to `package.json` changes the hash, so the `npm install` layer will be rebuilt on the first pipeline run. This is expected. We're flagging it so ops team doesn't panic when they see a slow build."

G. Martinez: "I ran a quick estimate on build time impact. The workspace has 812 installed packages. With `overrides`, npm will run a deduplication pass that typically adds 15-30 seconds to `npm install`. That's well within acceptable bounds."

T. Bergström: "Any concerns about the specific version choices?"

F. O'Sullivan: "Not on the security side — that's your team and the Security Guild's purview. On the ops side, we want to make sure the implementation PR includes:
1. The updated `package.json` with `overrides`
2. A regenerated `package-lock.json`
3. A note in the PR description about the expected Docker cache miss"

J. Hartwell: "Confirmed. We'll include all three in the implementation PR."

**Item 2: Rollback Procedure**

G. Martinez: "What's the rollback if something goes wrong?"

T. Bergström walked through the rollback procedure (now documented in RFC-2024-047 Section 13.4):
1. Remove or revert the `overrides` block from `package.json`
2. Run `npm install` to regenerate `package-lock.json`
3. Deploy the reverted files

F. O'Sullivan: "Rollback time estimate?"

T. Bergström: "15-30 minutes assuming a standard deploy pipeline run."

F. O'Sullivan: "Acceptable. Approved."

**Vote:**
- F. O'Sullivan: Approve
- G. Martinez: Approve

**Result:** DevOps sign-off obtained. 2-0 vote.

---

## Meeting M-08: RFC Finalization Pre-Check (2024-03-01)

**Date:** 2024-03-01  
**Time:** 10:00–10:30 AM  
**Location:** Slack huddle  
**Attendees:**
- J. Hartwell
- S. Okonkwo
- T. Bergström

**Agenda:**
1. Review implementation timeline
2. Final check before Round 3 version confirmation
3. Update implementation ticket

---

### M-08 Notes

J. Hartwell opened by noting the implementation deadline: the RFC must be finalized and implementation must be initiated by March 22 to meet the 30-day patching SLA for the High-severity lodash finding (identified January 29, SLA deadline February 28 — already past SLA by two weeks).

S. Okonkwo: "We're past the SLA. What's the escalation path?"

J. Hartwell: "Security Guild Lead has been informed. The delay was caused by the multi-round review process which was the correct approach for a workspace-wide change. The formal SLA breach needs to be documented in the risk register. We have executive approval to proceed with the formal process rather than an emergency patch."

T. Bergström: "For the implementation ticket (MONO-9001), the plan is: update `package.json`, run `npm install`, commit both files, open PR, request review. Straightforward."

S. Okonkwo: "Before we implement, we agreed to do a final check on semver and axios versions in case newer patches have been released since Round 2. That's the Round 3 check. Can we schedule that?"

J. Hartwell: "Security Guild asked to be involved in the final version check. I'll schedule it for March 15 — two weeks before the final deadline."

**Action items from M-08:**
- [ ] J. Hartwell to document SLA breach in risk register (DONE — 2024-03-01)
- [ ] J. Hartwell to schedule Round 3 final version check meeting for March 15 (DONE)
- [ ] T. Bergström to prepare implementation PR draft ready for March 22 (DONE)

---

## Meeting M-09: Round 3 Final Check (2024-03-15)

**Date:** 2024-03-15  
**Time:** 10:00–10:45 AM  
**Location:** Video call  
**Attendees:**
- A. Patel (Security Guild Lead)
- J. Hartwell (chair)
- S. Okonkwo
- T. Bergström

**Agenda:**
1. Final version check: semver (7.6.x latest)
2. Final version check: axios (1.6.x latest)
3. Confirm all other versions unchanged
4. Authorize RFC finalization

---

### M-09 Notes

**Item 1: semver Final Check**

S. Okonkwo presented the current semver release history as of 2024-03-15:

```
7.6.0 — 2024-01-12 (Round 2 proposal)
7.6.1 — 2024-02-08 (regression fix from 7.6.0)
7.6.2 — 2024-03-01 (edge case fix in satisfies())
7.6.3 — 2024-03-14 (input length enforcement — YESTERDAY)
```

S. Okonkwo: "7.6.3 was released yesterday. It adds explicit input length enforcement at function entry points. This is the 'belt-and-suspenders' fix the Security Guild mentioned wanting."

A. Patel: "We specifically recommended targeting `7.6.x` and re-checking at implementation time. `7.6.3` is better than `7.6.0`. We recommend updating the target to `7.6.3`."

T. Bergström: "Is `7.6.3` likely to have any regressions, being so new?"

A. Patel: "The input length enforcement is additive and well-tested. It only affects edge-case inputs well outside normal version string lengths. The risk of regression is very low."

**Decision:** Update semver target from `7.6.0` to `7.6.3`.

**Item 2: axios Final Check**

T. Bergström presented the current axios release history as of 2024-03-15:

```
1.6.0  — 2023-10-26 (CVE-2023-45857 fix — Round 2 proposal)
1.6.1  — 2023-11-08 (response interceptor regression fix)
1.6.2  — 2023-11-21 (FormData edge case)
1.6.3  — 2023-12-26 (null-byte injection in URL params — security)
1.6.4  — 2024-01-25 (dependency updates)
1.6.5  — 2024-02-20 (chunked transfer encoding fix)
1.6.6  — 2024-02-26 (prototype pollution in mergeDeep — SECURITY)
1.6.7  — 2024-03-11 (mergeDeep regression fix from 1.6.6)
1.6.8  — 2024-03-15 (SSRF in redirect handling — SECURITY, ALSO TODAY)
```

T. Bergström: "1.6.8 was literally released today. It patches an SSRF vulnerability in redirect handling."

A. Patel (checking notes): "Yes, we were aware this was coming. `1.6.8` addresses a vulnerability where following HTTP redirects didn't adequately validate the redirect target URL. On a server making outbound HTTP calls to third-party services, a server that returns a crafted `Location: http://169.254.169.254/` could cause axios to make a follow-up request to the AWS metadata endpoint."

J. Hartwell: "Is this specific to AWS EC2 instances?"

A. Patel: "Any cloud metadata service. AWS has `169.254.169.254`, Azure has `169.254.169.254` as well, GCP has `metadata.google.internal` and the same IP. If MonoStack makes outbound HTTP calls that can be influenced by a third party, this is relevant."

T. Bergström: "We do make outbound calls to third-party services in `@monostack/api`. The redirect URLs in those responses are theoretically attacker-controlled if the third-party service is compromised."

A. Patel: "Exactly. This is relevant. We recommend targeting `1.6.8`."

**Decision:** Update axios target from `1.6.0` to `1.6.8`.

S. Okonkwo noted this means the implementation will capture three additional security fixes beyond the original CVE (null-byte injection in `1.6.3`, prototype pollution in `1.6.6`, SSRF in redirects in `1.6.8`).

**Item 3: All Other Versions Confirmed Unchanged**

| Package | Confirmed Version | Notes |
|---|---|---|
| lodash | 4.17.21 | Unchanged since Round 1. Still latest. |
| express | 4.19.2 | Unchanged since Round 2. No newer relevant releases. |
| react | 18.3.1 | Still in pre-release at this time; releases in April 2024 as planned. |

J. Hartwell: "Wait — react `18.3.1` hasn't been released yet? We've been proposing it as a target."

S. Okonkwo: "It's on the React release calendar for late April 2024. The Frontend Guild confirmed the release date with the React team via their beta program participation. We're targeting it as the implementation deadline is March 22, but we'd be implementing the override before the version actually exists."

J. Hartwell: "That's a problem. If we run `npm install` with `react: 18.3.1` in overrides and the version doesn't exist, the install will fail."

T. Bergström: "We'd need to either: (a) wait until after `18.3.1` is released to implement the react override, or (b) use `18.2.0` as an interim target."

A. Patel: "From a security perspective, using `18.2.0` is fine — there's no CVE driving the react override. The fragmentation issue is the driver."

S. Okonkwo: "Let me get clarification from the Frontend Guild on whether `18.2.0` is acceptable as an interim target pending `18.3.1` release."

**Post-meeting update (2024-03-18):**
M. Chen (Frontend Guild Lead) confirmed via Slack: "Use `18.3.1` as the target in the overrides block. `npm install` will fail if the version doesn't exist, so delay the `npm install` step until after `18.3.1` releases in late April. The `package.json` overrides can be committed now; the `package-lock.json` regeneration happens at implementation time."

*Note: Implementation was ultimately delayed to April-May 2024 to allow `react@18.3.1` to release before running `npm install`. The RFC documents the override targets for the final implementation state.*

**Item 4: RFC Finalization Authorization**

With the semver and axios targets updated:

| Package | Final Target |
|---|---|
| lodash | 4.17.21 |
| semver | 7.6.3 |
| axios | 1.6.8 |
| express | 4.19.2 |
| react | 18.3.1 |

A. Patel: "Security Guild authorizes RFC finalization with these targets."

J. Hartwell: "RFC Section 12 will be updated with these final targets. RFC-2024-047 is hereby finalized."

**Action items from M-09:**
- [ ] J. Hartwell to update RFC Section 11 with Round 3 decisions (DONE)
- [ ] J. Hartwell to update RFC Section 12 with final version table (DONE)
- [ ] S. Okonkwo to update Appendix B (superseded proposals log) with Round 3 changes (DONE)
- [ ] T. Bergström to update implementation ticket to note react 18.3.1 timing dependency (DONE)

---

## Meeting M-10: RFC Sign-Off (2024-03-22)

**Date:** 2024-03-22  
**Time:** 9:30–9:45 AM  
**Location:** Slack huddle  
**Attendees:**
- J. Hartwell
- A. Patel (async, via written approval)

**Agenda:**
1. Confirm RFC-2024-047 is ready for publication
2. Collect final sign-offs

---

### M-10 Notes

J. Hartwell circulated the final RFC for review on 2024-03-20. All three stakeholder groups provided written approval:

- **Security Guild** (A. Patel, 2024-03-20): "Security Guild approves RFC-2024-047 as written. All five override targets reflect the Security Guild's recommendations as of the final review on 2024-03-15."

- **Frontend Guild** (M. Chen, 2024-03-21): "Frontend Guild approves. React target `18.3.1` and explicit exclusion of `react-dom` from overrides are correctly documented."

- **DevOps** (F. O'Sullivan, 2024-03-21): "DevOps approves. No changes to our review comments from 2024-02-19."

J. Hartwell published RFC-2024-047 as FINAL on 2024-03-22.

**Implementation status:** Waiting on `react@18.3.1` release (expected late April 2024) before running `npm install`. Implementation ticket: MONO-9001.

---

## Appendix: Decision Log

All key decisions made during the RFC process are summarized here for quick reference.

| Decision ID | Date | Decision | Meeting |
|---|---|---|---|
| D-01 | 2024-01-08 | Use npm overrides approach for all fixes | M-01 |
| D-02 | 2024-01-08 | Conduct multi-round review with Security Guild, Frontend Guild, and DevOps | M-01 |
| D-03 | 2024-01-17 | Preserve rejected proposals in RFC for historical clarity | M-02 |
| D-04 | 2024-01-25 | Target semver 7.6.x (not 7.5.x) based on Security Guild recommendation | M-03 |
| D-05 | 2024-01-25 | Reject axios 1.4.0 formally; minimum target is 1.6.0 | M-03 |
| D-06 | 2024-01-25 | Reject express 4.18.3 formally; only 4.19.2 is acceptable | M-03 |
| D-07 | 2024-02-05 | Target react 18.3.1 (not 18.2.0) per Frontend Guild recommendation | M-04 |
| D-08 | 2024-02-05 | Explicitly exclude react-dom from overrides | M-04 |
| D-09 | 2024-02-12 | Frontend Guild formal sign-off: 3-0 approve | M-05 |
| D-10 | 2024-02-19 | DevOps formal sign-off: 2-0 approve | M-07 |
| D-11 | 2024-03-15 | Update semver target from 7.6.0 to 7.6.3 (newer patch available) | M-09 |
| D-12 | 2024-03-15 | Update axios target from 1.6.0 to 1.6.8 (newer security patches available) | M-09 |
| D-13 | 2024-03-22 | RFC finalized; all stakeholder sign-offs obtained | M-10 |

---

## Appendix: Rejected Proposals Tracker

| Package | Rejected Version | Reason | Meeting Where Rejected |
|---|---|---|---|
| axios | 1.4.0 | Does not fix CVE-2023-45857; fix is in 1.6.0 | M-02 (identified), M-03 (formally rejected) |
| express | 4.18.3 | Does not fix CVE-2024-29041; fix is in 4.19.2 | M-02 (identified), M-03 (formally rejected) |
| semver | 7.5.4 | Correct fix but superseded by 7.6.x recommendation | M-03 |
| semver | 7.6.0 | Correct fix but superseded by 7.6.3 (newer patch) | M-09 |
| axios | 1.6.0 | Correct fix but superseded by 1.6.8 (additional security patches) | M-09 |
| react | 18.2.0 | Correct (no CVE) but superseded by 18.3.1 (Frontend Guild recommendation) | M-04 |

---

---

## Appendix: Background — npm Overrides Mechanism Deep Dive

This section was produced by T. Bergström as research material for the team before the M-01 kickoff meeting. It explains the npm `overrides` mechanism for team members unfamiliar with it.

### Background-1: What Problem Do Overrides Solve?

In a large JavaScript project, your direct dependencies (the packages you explicitly listed) each bring their own dependencies. Those packages bring more dependencies. By the time npm finishes resolving the full tree, you may have hundreds of packages installed — only a fraction of which you directly chose.

When a security vulnerability is found in a transitive package (one you didn't directly choose), fixing it is surprisingly hard without the overrides mechanism:

**Option A: Wait for your direct dependency to patch it**
If your package `foo@1.0.0` depends on vulnerable `bar@1.0.0`, you need `foo`'s authors to:
1. Become aware of the vulnerability
2. Update their dependency on `bar`
3. Publish a new version of `foo`
4. You update your dependency on `foo`

This can take weeks or months. The `foo` authors may not prioritize it. If `foo` is abandoned, it may never happen.

**Option B: Force-patch it yourself using npm overrides**
Add to your root `package.json`:
```json
{
  "overrides": {
    "bar": "1.0.1"
  }
}
```
Now, regardless of what `foo` requests for `bar`, your workspace will use `bar@1.0.1`. You control the resolution without needing upstream cooperation.

### Background-2: How npm Resolves With Overrides

When npm resolves the dependency tree, it applies overrides at resolution time — before installation. The algorithm:

1. Build the normal dependency graph (ignoring overrides)
2. For each node in the graph, check if its package name matches an override key
3. If it matches, substitute the override version for the requested version
4. Check that the substituted version satisfies the requester's version range
   - If it does: proceed
   - If it doesn't: npm warns but proceeds (the override takes priority)
5. Deduplicate the resulting tree — all packages that were forced to the same version by an override can now be hoisted to a single location
6. Install the final tree

Step 4 is important: if your override specifies a version that doesn't satisfy the requester's range (e.g., overriding `react@^16.x` to `react@18.3.1`), npm will warn but still apply the override. For the five overrides in RFC-2024-047, all override versions satisfy the existing requester ranges (see RFC-2024-047 Appendix C).

### Background-3: Overrides vs. resolutions (yarn) vs. overrides (pnpm)

The `overrides` field is npm's implementation. Yarn and pnpm have analogous mechanisms:

| Tool | Mechanism | Config Field |
|---|---|---|
| npm ≥7 | overrides | `"overrides": {}` in root `package.json` |
| yarn classic (1.x) | resolutions | `"resolutions": {}` in root `package.json` |
| yarn berry (2+) | resolutions | `"resolutions": {}` in root `package.json` |
| pnpm | overrides | `"pnpm": {"overrides": {}}` in root `package.json` |

MonoStack uses npm workspaces (not yarn or pnpm), so `"overrides"` in the root `package.json` is the correct mechanism.

**Warning:** If MonoStack ever migrates to pnpm, the overrides would need to be moved to the `pnpm.overrides` field. The npm-style `overrides` field at the root level is ignored by pnpm.

### Background-4: Override Targeting (Scoped Overrides)

npm supports more fine-grained override targeting for cases where you want to override a transitive dependency only when required by a specific parent package. The syntax:

```json
{
  "overrides": {
    "package-that-requires-vulnerable-dep": {
      "vulnerable-dep": "fixed-version"
    }
  }
}
```

For example, to override `lodash` only when required by `data-processor`:
```json
{
  "overrides": {
    "data-processor": {
      "lodash": "4.17.21"
    }
  }
}
```

RFC-2024-047 uses flat (unscoped) overrides because:
1. The vulnerable packages are used by multiple dependents
2. There's no reason to allow the vulnerable version anywhere in the tree
3. Scoped overrides add complexity without benefit when the flat override is appropriate

Scoped overrides are useful when an upgrade would break one specific consumer but not others — not the case here.

### Background-5: Override Limitations and Edge Cases

**Overrides don't affect the package that declares the override:**
If your root `package.json` declares `"dependencies": {"lodash": "^4.17.0"}` AND `"overrides": {"lodash": "4.17.21"}`, the override does NOT affect your own direct dependency resolution. Your direct `lodash` dependency will resolve normally (to `4.17.21` if that's the latest satisfying `^4.17.0`). The override affects how npm resolves `lodash` when requested by your dependencies and their dependencies.

In MonoStack's case, no package in the workspace declares `lodash` as a direct dependency — it's entirely transitive — so this limitation is irrelevant.

**Overrides can break things if the version is incompatible:**
If you override a package to a version that introduces breaking changes for a package that depends on it, you'll see runtime errors even though `npm install` succeeds. The override mechanism is powerful but requires care. This is why RFC-2024-047 conducts compatibility analysis (Section 7) and verifies that all override versions satisfy existing peer dependency requirements (Appendix C).

**Overrides don't modify package.json files of sub-packages:**
The override only changes what gets installed in `node_modules`. It doesn't modify the `package.json` of `data-processor` or any other package. If `data-processor@2.1.0` is published with `"lodash": "^4.17.0"` in its dependencies, that remains unchanged. The override is purely a workspace-level resolution instruction.

**Overrides persist until explicitly removed:**
Unlike `npm update` (which updates dependencies within declared ranges), overrides are static configuration. They apply indefinitely until removed from `package.json`. This is the desired behavior for security overrides — they should stay in place until the root cause (the upstream dependency declaring the old version) is addressed and the override is no longer necessary.

---

## Appendix: Glossary

| Term | Definition |
|---|---|
| CVE | Common Vulnerabilities and Exposures. A numbered identifier for a specific, publicly known security vulnerability. Example: CVE-2021-23337. |
| CVSS | Common Vulnerability Scoring System. A numerical score (0–10) indicating vulnerability severity. 7.0–8.9 = High, 4.0–6.9 = Medium, 0.1–3.9 = Low. |
| CWE | Common Weakness Enumeration. A categorization of software weaknesses. Example: CWE-1321 (Prototype Pollution). |
| Prototype Pollution | A class of JavaScript vulnerability where an attacker can set properties on `Object.prototype`, affecting all objects in the runtime. |
| ReDoS | Regular Expression Denial of Service. An attack where a crafted input causes a regex to take exponential time to evaluate, blocking the event loop. |
| SSRF | Server-Side Request Forgery. An attack where a server is tricked into making HTTP requests to internal or unintended external resources. |
| XSRF / CSRF | Cross-Site Request Forgery. An attack where a user is tricked into submitting a request to a site they're authenticated to, using their existing session. |
| transitive dependency | A package that is installed because it is required by one of your direct dependencies (or their dependencies). You didn't directly choose it. |
| direct dependency | A package explicitly listed in your `package.json` `dependencies` or `devDependencies`. |
| hoisting | npm's behavior of placing a single copy of a package in the root `node_modules` when possible, rather than nested copies in each consuming package's `node_modules`. |
| npm overrides | A mechanism in npm 7+ that allows a workspace root to specify exact versions for any package in the dependency tree, overriding normal resolution. |
| workspace | An npm feature for managing multiple packages in a single repository. Defined by the `workspaces` field in the root `package.json`. |
| lockfile | `package-lock.json` (npm), `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm). A file that records the exact resolved versions of all installed packages for reproducible installs. |
| package-lock.json | npm's lockfile. Records the full resolved dependency tree with exact versions and integrity hashes. Must be committed to version control and regenerated after any dependency change. |
| SLA | Service Level Agreement. In this context, the maximum time allowed between identifying a vulnerability and implementing a fix. High severity: 30 days. Medium severity: 90 days. |
| RFC | Request for Comments. A document proposing a change, circulated for stakeholder review before implementation. RFC-2024-047 is this document's reference ID. |

---

*End of Meeting Notes for RFC-2024-047*

*Maintained by: Platform Team. For questions or corrections, contact J. Hartwell or open a ticket in the MonoStack project.*

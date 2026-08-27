# CHAM Agent — Audit Item Tracking

Consolidated, deduplicated tracker for actionable items from the weekly security & architecture audits.
Purpose: let future audit runs skip items already resolved, and record deliberate won't-fix decisions.

**Last reconciled:** 2026-08-27 (verified against working tree of `main`, commit `ed1dafc`).
**Audits covered in this reconciliation:** `weekly-audit-2026-05-07`, `-05-14`, `-05-21`, `-06-04`, `-07-02`, `-07-09`, `-08-27`.

Status legend: ✅ Done · ❌ Open · ⚠️ Won't fix (with rationale)

> ✅ **All CRITICAL items resolved as of 2026-08-27 audit.** First fully clean CRITICAL/HIGH audit. 4 new MEDIUM items opened (items 26–29).

## Security (CRITICAL) — resolved in 2026-08-27 audit

| # | Item | Source audit | Status | Evidence (verified 2026-08-27) |
|---|------|--------------|--------|--------------------------------|
| 20 | Live MongoDB Atlas creds in git-tracked `.claude/settings.local.json` | 2026-07-09 | ✅ Done | File absent from disk; `git log --all -- ".claude/settings.local.json"` returns empty — history scrub confirmed. File in `.gitignore`. |
| 21 | `API_KEY` baked into client JS bundle | 2026-07-09 | ✅ Done | `vite.config.ts` contains only proxy config; no `define` block. Committed `ed1dafc`. |
| 22 | No rate limit on `POST /student/join-course` | 2026-07-02 (carried, 2 audits) | ✅ Done | `submitRateLimit` at `api/index.js:456`. Committed `ed1dafc`. |
| 23 | IDOR — lecturer reads any course (4 read-routes, ownership unchecked) | 2026-07-09 | ✅ Done | `Course.findOne({ _id, lecturerId })` → 403 at `api/index.js:851, 1124, 1139, 1365`. Committed `ed1dafc`. |
| 24 | IDOR — student reads any course without enrollment (2 read-routes) | 2026-07-09 | ✅ Done | `enrolledCourseIds.includes()` → 403 at `api/index.js:553, 1009`. Committed `ed1dafc`. |
| 25 | Mass assignment in `POST /lecturer/archive` — `lecturerId` overridable | 2026-07-09 | ✅ Done | Server fields set after spread at `api/index.js:436–440`. Committed `ed1dafc`. |

Full detail: `docs/audits/weekly-audit-2026-07-09.md`.

## Security (CRITICAL) — historical (all resolved through 2026-06-04)

| # | Item | Source audit | Status | Evidence (current code) |
|---|------|--------------|--------|-------------------------|
| 1 | Rate limit on `POST /grades/save` | 2026-05-21, 06-04 | ✅ Done | `api/index.js:691` `uploadRateLimit` |
| 2 | IDOR `GET /teacher/review/:submissionId` | 2026-06-04 | ✅ Done | `api/index.js:1219-1221` course-ownership check |
| 3 | IDOR `POST /teacher/submit-review` | 2026-06-04 | ✅ Done | `api/index.js:1250-1252` |
| 4 | IDOR `approve`/`reject`/`remove-student` | 2026-06-04 | ✅ Done | `api/index.js:1305-1306`, `1325-1326`, `1344-1345` |
| 5 | IDOR `POST /lecturer/submissions/:id/extension` | 2026-06-04 | ✅ Done | `api/index.js:898-905` |
| 6 | IDOR `GET /lecturer/courses/:id/all-submissions` | 2026-06-04 | ✅ Done | `api/index.js:1149-1150` |
| 7 | IDOR on 9 assignment/material routes | 2026-05-21 | ✅ Done | `api/index.js:833-895`, `910-954`, `1371-1398` |
| 8 | Chat routes must use `buildSafePrompt`/`buildSafeChatPrompt` | 2026-05-07, 05-14 | ✅ Done | `api/index.js:606`, `733`, `744`, `793` |
| 9 | Remove premature Gemini-key check in `semanticAssessment.js` | 2026-05-07 | ✅ Done | guard removed; reaches orchestrator |
| 10 | Rate limits on update-role / courses POST+PUT / submit-review | 2026-05-07 | ✅ Done | `api/index.js:384`, `1090`, `1098`, `1237` |
| 11 | `POST /evaluate` lecturer-only | 2026-05-07 | ✅ Done | `api/index.js:769` role check |
| 12 | `POST /user/update-role` block re-assignment | 2026-05-07 | ✅ Done | `api/index.js:387` enum + `390` role-set guard |

## MEDIUM — OPEN (2026-08-27 audit)

| # | Item | Source audit | Status | Evidence / notes |
|---|------|--------------|--------|------------------|
| 26 | Score-range validator checks 0–100 but `/evaluate` uses 0–10 scale — out-of-scale LLM responses pass validation | 2026-08-27 | ❌ Open | `api/index.js:813` calls `validateLLMOutput(response.raw, ...)` which checks ≤100; prompt at `api/index.js:785` requests 0–10. Fix: add `score > 10` rejection in the `/evaluate` handler or add per-field range opts to `validateLLMOutput()`. |
| 27 | `DELETE /lecturer/courses/:id`, `/lecturer/assignments/:id`, `/lecturer/materials/:id` — no rate limiter | 2026-08-27 | ❌ Open | `api/index.js:872, 1113, 1400`. Cascading deletes (course→materials, assignment→submissions) amplify impact of a compromised account. Apply `uploadRateLimit`. |
| 28 | 5 state-modifying POST routes without rate limiting: `/submissions/:id/extension`, `/assignments/:id/release-feedback`, `/courses/:id/approve`, `/reject`, `/remove-student` | 2026-08-27 | ❌ Open | `api/index.js:903, 915, 1310, 1330, 1349`. Most notable: `release-feedback` triggers `Submission.updateMany()`. Apply `uploadRateLimit` (first two) and `submitRateLimit` (enrollment routes). |
| 29 | Mass assignment on `POST /lecturer/assignments` and `PUT /lecturer/assignments/:id` — `req.body` passed without field allowlist | 2026-08-27 | ❌ Open | `api/index.js:844, 868`. Lecturer can inject `requires_human_review: false` etc. Destructure only allowed fields before passing to Mongoose. |

## MEDIUM / housekeeping

| # | Item | Source audit | Status | Evidence / notes |
|---|------|--------------|--------|------------------|
| 13 | `evaluateSubmission` dead export removed | 2026-05-07 | ✅ Done | absent from `services/chatService.ts` |
| 14 | `GradeBook.tsx` RTL `scrollBy.left` | 2026-04-17 → 06-04 (7 audits) | ✅ Done | `components/GradeBook.tsx:38-43` RTL-aware `scrollBy` (fixed 2026-07-13) |
| 15 | `StudentAssignments.tsx` `text-left` → `text-end` | 2026-05-21 | ✅ Done | `components/StudentAssignments.tsx:134` (fixed 2026-07-13) |
| 16 | Physical `borderRight`/`paddingRight` → logical props | 2026-04-30 → 06-04 (5 audits) | ✅ Done | `StudentAssignments.tsx:176`, `AssignmentManager.tsx:263,417` → `borderInlineEnd`/`paddingInlineEnd` (fixed 2026-07-13) |
| 17 | Create git tag `prompt-v1.2.0` | 2026-05-07 → 06-04 (4 audits) | ✅ Done | tag `prompt-v1.2.0` → `8dad476` (created 2026-07-13) |
| 18 | Align `package.json` to `PROMPT_VERSION` | 2026-05-07 → 06-04 | ⚠️ Won't fix | App version (`package.json` `2.1.1`) and prompt-template version (`PROMPT_VERSION v1.2.0`) version different artifacts and are intentionally decoupled. Only the missing tag (#17) was valid. **Drop this recommendation from future audits.** |
| 19 | `ForExample/` dead fixture files | 2026-05-07 → 06-04 (4 audits) | ❌ Open | 6 `.txt` files, referenced nowhere outside `docs/audits`. Deletion (`git rm -r ForExample/`) proposed but not yet approved. |

## Notes for future audit runs

- **Items 20–25 are FULLY RESOLVED** as of 2026-08-27 audit. Do not re-flag unless source regresses.
- **Items 26–29 are OPEN MEDIUM** (2026-08-27 audit). Priority: 26 (score mismatch) → 27 (DELETE rate limits) → 28 (POST rate limits) → 29 (mass assignment).
- Items 1–17 are settled; do not re-flag unless the referenced code regresses.
- Item 18 is a deliberate won't-fix — stop recommending `package.json` ↔ `PROMPT_VERSION` alignment.
- Item 19 (`ForExample/` dead files) remains open, cosmetic only, not security.

# CHAM Agent — Weekly Security & Architecture Audit (2026-08-27)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`, `.gitignore`, `.env.example`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open)  
**Date:** 2026-08-27

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| 1 | CRITICAL | Unprotected LLM call sites | All call sites use `LLMOrchestrator.evaluateWithFallback()` + `buildSafePrompt()` / `buildSafeChatPrompt()` | ✓ Clean |
| 2 | CRITICAL | Missing rate limiting (POST/PUT with code/text/file) | All such routes carry a rate limiter (llmRateLimit, submitRateLimit, uploadRateLimit, or messagesRateLimit) | ✓ Clean |
| 3 | CRITICAL | Session/RBAC regressions | All `/lecturer/*` routes assert lecturer role; all `/student/*` routes assert student role; `/auth/dev` is production-disabled | ✓ Clean |
| 4 | CRITICAL | Secrets in code | No API keys, MongoDB URIs, or OAuth secrets in tracked files; `settings.local.json` removed from disk and git history | ✓ Clean |
| 5 | HIGH | Unsafe JSON parsing from LLM | All LLM response parsing uses `safeParseLLMResponse` via `validateLLMOutput` | ✓ Clean |
| 6 | HIGH | Missing output validation | `validateLLMOutput()` called on all evaluation paths (`api/index.js:813`, `services/semanticAssessment.js:105`) | ✓ Clean |
| 7 | HIGH | `alert()` in UI | No `alert()`, `confirm()`, or `prompt()` calls in any component; all replaced with inline state (Audit #7 comments confirm intent) | ✓ Clean |
| 8 | MEDIUM | Score-range validator scale mismatch | `validateLLMOutput` checks 0–100 but `/evaluate` uses 0–10 scale; a misinterpreting LLM could return 85 and pass validation | **Open (new)** |
| 9 | MEDIUM | Destructive DELETE routes lack rate limiting | `DELETE /lecturer/courses/:id`, `/lecturer/assignments/:id`, `/lecturer/materials/:id` — no rate limiter applied | **Open (new)** |
| 10 | MEDIUM | Additional POST routes lack rate limiting | `/lecturer/courses/:id/approve`, `/reject`, `/remove-student`, `/submissions/:id/extension`, `/assignments/:id/release-feedback` have no rate limiter | **Open (new)** |
| 11 | MEDIUM | Mass assignment on assignment/course creation | `POST /lecturer/assignments` and `PUT /lecturer/assignments/:id` pass `req.body` directly without explicit field allowlist | **Open (new)** |
| 12 | MEDIUM | Prompt version — audit spec stale | Spec references `v1.1.0`; code correctly shows `v1.2.0` (bumped with tag `prompt-v1.2.0`). No further drift detected. | Report only |
| 13 | MEDIUM | `ForExample/` dead fixture files | 6 `.txt` files, not referenced anywhere in production code, tests, or docs | **Open (carried, item 19, 8th audit)** |

**CRITICAL open:** 0  
**HIGH open:** 0  
**MEDIUM open:** 4 (items 8, 9, 10, 11) + 1 carried (item 13)

---

## Prior Audit Resolution Status (2026-07-09 — 6 CRITICAL)

| Previous Finding | Resolution |
|-----------------|------------|
| CRITICAL-20: Live MongoDB Atlas creds in git-tracked `settings.local.json` | ✅ Resolved — file removed from disk, `git filter-repo` history scrub confirmed (`git log --all -- ".claude/settings.local.json"` returns empty) |
| CRITICAL-21: `API_KEY` baked into client bundle via `vite.config.ts` | ✅ Resolved — `define` block removed; `vite.config.ts` contains only proxy config |
| CRITICAL-22: No rate limit on `POST /student/join-course` | ✅ Resolved — `submitRateLimit` at `api/index.js:456` |
| CRITICAL-23: IDOR on 4 lecturer read-routes (no course ownership check) | ✅ Resolved — `Course.findOne({ _id, lecturerId })` ownership guard at `api/index.js:851, 1124, 1139, 1365` |
| CRITICAL-24: IDOR on 2 student read-routes (no enrollment check) | ✅ Resolved — `enrolledCourseIds.includes()` guard at `api/index.js:553, 1009` |
| CRITICAL-25: Mass assignment in `POST /lecturer/archive` — `lecturerId` overridable | ✅ Resolved — server fields (`lecturerId`, `timestamp`) set after spread at `api/index.js:436–440`; comment confirms intent |

All 6 CRITICAL findings from 2026-07-09 are confirmed resolved. This is the first fully-clean CRITICAL/HIGH audit.

---

## Check 1 — Unprotected LLM Call Sites

**Result: Clean**

All paths that call an LLM provider go through `LLMOrchestrator.evaluateWithFallback()` defined in `lib/llm/orchestrator.js`. User-controlled input is always routed through `buildSafePrompt()` or `buildSafeChatPrompt()` from `services/promptGuard.js` before the prompt is passed to the orchestrator. Verified paths:

| Route / Service | Safe prompt builder | Orchestrator call |
|----------------|---------------------|-------------------|
| `POST /evaluate` (`api/index.js:796–806`) | `buildSafePrompt()` | `evaluateWithFallback()` ✓ |
| `POST /chat` (`api/index.js:733–757`) | `buildSafePrompt()` or `buildSafeChatPrompt()` | `evaluateWithFallback()` ✓ |
| `POST /student/chat` (`api/index.js:609–617`) | `buildSafeChatPrompt()` | `evaluateWithFallback()` ✓ |
| `POST /lecturer/assignments/:id/submit-manual` → `chamAssessment.js` → `semanticAssessment.js` | `buildSafePrompt()` | `evaluateWithFallback()` ✓ |
| `POST /student/assignments/:id/submit` → same chain | `buildSafePrompt()` | `evaluateWithFallback()` ✓ |

Direct provider instantiation (`new GroqProvider`, `new OpenAIProvider`) only occurs inside `tests/agents/llm-providers.test.js` — test code only, not production paths.

---

## Check 2 — Missing Rate Limiting

**Result: Clean for POST/PUT routes with code/text/file content**

All routes accepting user-controlled code, text, or file content carry a rate limiter:

| Limiter | Rate | Applied to |
|---------|------|-----------|
| `llmRateLimit` | 100/hr | `/evaluate`, `/chat`, `/student/chat`, `/lecturer/assignments/:id/submit-manual` |
| `submitRateLimit` | 20/15 min | `/student/assignments/:id/submit`, `/student/join-course`, `/teacher/submit-review` |
| `uploadRateLimit` | 20/hr | `/lecturer/archive`, `/student/private-materials`, `/lecturer/assignments` (POST+PUT), `/lecturer/courses` (POST+PUT), `/lecturer/materials` (POST+PUT), `/grades/save`, `/user/update-role` |
| `messagesRateLimit` | 60/min | `POST /messages`, `PUT /messages/:id` |

**MEDIUM — unrated DELETE and state-modifying POST routes (items 9 & 10, see below)**

---

## Check 3 — Session/RBAC

**Result: Clean**

- All 23 routes under `/api/lecturer/*` assert `req.user.role !== 'lecturer'` before executing.
- All 9 routes under `/api/student/*` assert `req.user.role !== 'student'`.
- Routes under `/api/teacher/*` (review queue) also assert lecturer role.
- `POST /api/auth/dev` (`api/index.js:365`): early-return `403` when `NODE_ENV === 'production'`. ✓
- Cross-role routes (`/messages`, `/users/all`, `/grades`) correctly require only authentication (`req.user`), not a specific role; access is further scoped to the authenticated user's own data.

---

## Check 4 — Secrets in Code

**Result: Clean**

- No `.env`, `.env.local`, `.env.production`, or secrets files are committed.
- `.env.example` contains only placeholder values (no real credentials).
- `settings.local.json`: removed from disk and confirmed absent from all git history via `git log --all -- ".claude/settings.local.json"` (empty output).
- `SESSION_SECRET` dev fallback (`api/index.js:300`): `'dev-secret-not-for-production'` is a clearly-labeled non-secret development default; production throws at startup if the env var is missing (`api/index.js:296–298`). Acceptable.
- No `sk-`, `AIza`, `gsk_`, `mongodb+srv://`, or Bearer tokens found in any tracked `.js/.ts/.tsx/.json/.md` file.

---

## Check 5 — Unsafe JSON Parsing from LLM

**Result: Clean**

`safeParseLLMResponse` (`lib/llm/safeParse.js`) handles markdown fence stripping and nested try/catch. It is the only JSON-parsing path for LLM output, called exclusively through `validateLLMOutput()` in `services/promptGuard.js:174`. No bare `JSON.parse(llmResponse)` calls found anywhere in the codebase (grep confirmed zero matches).

---

## Check 6 — Missing Output Validation

**Result: Clean**

- `POST /evaluate` (`api/index.js:813`): calls `validateLLMOutput(response.raw, ['score', 'feedback'])` and returns HTTP 500 if invalid.
- `services/semanticAssessment.js:105`: calls `validateLLMOutput(data || result.raw, [])` then separately verifies all `REQUIRED_FIELDS`.

---

## Check 7 — `alert()` in UI

**Result: Clean**

Grep across all `.tsx` files confirmed zero production `alert()`, `confirm()`, or `prompt()` calls. Comments referencing "Audit #7" throughout `CourseManager.tsx`, `LecturerDashboard.tsx`, `AssignmentManager.tsx`, and `ResultSection.tsx` confirm deliberate replacement with inline state.

---

## MEDIUM Findings

### MEDIUM-1 — Score-Range Validator Scale Mismatch (NEW)

**File:** `api/index.js:813`, `services/promptGuard.js:192`

`validateLLMOutput()` checks that any field containing `"score"` falls within `0–100`. The `/evaluate` endpoint instructs the LLM to return `score` on a `0–10` scale (prompt at `api/index.js:785–793`). A misbehaving LLM that misinterprets the scale and returns `score: 85` (treating it as 0–100) would pass the validator (`85 ≤ 100`) and be returned to the client as a valid grade.

**Recommendation:** Add a separate range check in the `/evaluate` handler that rejects any `score > 10` before returning to the client, or update `validateLLMOutput()` to accept an optional `{ min, max }` per-field override.

---

### MEDIUM-2 — Destructive DELETE Routes Lack Rate Limiting (NEW)

**File:** `api/index.js:872, 1113, 1400`

Three destructive DELETE endpoints have no rate limiter:

| Route | Line | Cascade effect |
|-------|------|----------------|
| `DELETE /lecturer/assignments/:id` | 872 | Deletes assignment + all its submissions (`Submission.deleteMany`) |
| `DELETE /lecturer/courses/:id` | 1113 | Deletes course + all its materials (`Material.deleteMany`) |
| `DELETE /lecturer/materials/:id` | 1400 | Deletes a single material |

A compromised lecturer account (session hijack) could rapidly delete all course data. Auth guards are present but no rate limiter limits the blast radius.

**Recommendation:** Apply `uploadRateLimit` (20/hr) to all three routes.

---

### MEDIUM-3 — State-Modifying POST Routes Lack Rate Limiting (NEW)

**File:** `api/index.js:903, 915, 1310, 1330, 1349`

Five POST routes that modify course/submission state have no rate limiter:

| Route | Line | Concern |
|-------|------|---------|
| `POST /lecturer/submissions/:id/extension` | 903 | Date write on submission |
| `POST /lecturer/assignments/:id/release-feedback` | 915 | `Submission.updateMany()` across all submissions |
| `POST /lecturer/courses/:id/approve` | 1310 | Modifies course + user enrollment |
| `POST /lecturer/courses/:id/reject` | 1330 | Modifies course enrollment |
| `POST /lecturer/courses/:id/remove-student` | 1349 | Removes student from course |

Most concerning is `/release-feedback` which triggers `updateMany` on all submissions for an assignment — could be spammed to generate high DB write load.

**Recommendation:** Apply `uploadRateLimit` to the first two and `submitRateLimit` to the enrollment-mutation routes.

---

### MEDIUM-4 — Mass Assignment on Assignment Create/Update Routes (NEW)

**File:** `api/index.js:844, 868`

```js
// Line 844 — create
const assignment = await Assignment.create(req.body);

// Line 868 — update  
const updatedAssignment = await Assignment.findByIdAndUpdate(req.params.id, req.body, { new: true });
```

Both pass `req.body` directly without an explicit field allowlist. Mongoose schema filtering provides implicit protection, but a lecturer could inject fields like `requires_human_review: false` to bypass manual review routing, or set `question_type` to a value that affects scoring thresholds.

These routes are lecturer-only, so the risk is limited to a malicious or compromised lecturer account. For comparison, `POST /lecturer/materials` (`api/index.js:1374`) already uses explicit destructuring.

**Recommendation:** Destructure only the allowed fields from `req.body` before passing to `create()` / `findByIdAndUpdate()`.

---

### MEDIUM-5 — Prompt Version Note (Report Only)

`lib/constants.js` exports `PROMPT_VERSION = 'v1.2.0'`. The audit task spec references `v1.1.0` as the current version — this reference is stale. The `prompt-v1.2.0` git tag exists (per AUDIT_TRACKING item 17). No prompt template modifications detected in the current working tree. **No action required.**

---

### MEDIUM-6 — `ForExample/` Dead Fixture Files (Carried, Item 19, 8th Audit)

**Path:** `ForExample/` (6 `.txt` files)

Not referenced by `package.json`, `api/index.js`, `App.tsx`, `vercel.json`, or any test. Proposed fix: `git rm -r ForExample/`. Pending deliberate approval from maintainer.

---

## Audit Spec Notes

The audit task references `[full_path_of_file_1]` and `[full_path_of_file_2]` as dead-code targets for Check #10 — these appear to be unresolved template placeholders. `server_reference.js` does not exist anywhere in the repository. No additional orphaned files were detected beyond `ForExample/`.

---

## AUDIT_TRACKING.md Update Required

The following items should be updated in `AUDIT_TRACKING.md`:

| Item # | Prior status | New status |
|--------|-------------|------------|
| 20 | ❌ Open (partial) | ✅ Resolved — history scrub confirmed |
| 21 | ✅ Fixed (working tree) | ✅ Committed (`ed1dafc`) |
| 22 | ✅ Fixed (working tree) | ✅ Committed (`ed1dafc`) |
| 23 | ✅ Fixed (working tree) | ✅ Committed (`ed1dafc`) |
| 24 | ✅ Fixed (working tree) | ✅ Committed (`ed1dafc`) |
| 25 | ✅ Fixed (working tree) | ✅ Committed (`ed1dafc`) |
| (new) | — | ❌ MEDIUM-1: Score-range mismatch (`api/index.js:813`) |
| (new) | — | ❌ MEDIUM-2: DELETE routes lack rate limiting (`api/index.js:872, 1113, 1400`) |
| (new) | — | ❌ MEDIUM-3: 5 state-modifying POST routes lack rate limiting |
| (new) | — | ❌ MEDIUM-4: Mass assignment on assignment create/update |

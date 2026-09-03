# CHAM Agent — Weekly Security & Architecture Audit (2026-09-03)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open)  
**Date:** 2026-09-03

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| 1 | CRITICAL | Missing rate limiting | `POST /lecturer/submissions/:id/extension` — no rate limit, body has `extensionUntil` | **Open (new)** |
| 2 | CRITICAL | Missing rate limiting | `POST /lecturer/courses/:id/approve` — no rate limit, body has `studentId` | **Open (new)** |
| 3 | CRITICAL | Missing rate limiting | `POST /lecturer/courses/:id/reject` — no rate limit, body has `studentId` | **Open (new)** |
| 4 | CRITICAL | Missing rate limiting | `POST /lecturer/courses/:id/remove-student` — no rate limit, body has `studentId` | **Open (new)** |
| 5 | HIGH | Missing output validation | `/evaluate` validates score 0–100 but schema specifies 0–10; scores 11–100 pass silently | **Open (new)** |
| 6 | HIGH | Mass assignment | `Assignment.create(req.body)` and 2 findByIdAndUpdate calls pass raw `req.body` without field whitelist | **Open (new)** |
| 7 | — | Unprotected LLM call sites | All LLM calls route through orchestrator + promptGuard | ✓ Clean |
| 8 | — | Session/RBAC | All `/lecturer/*` and `/student/*` routes assert correct role; `/auth/dev` disabled in production | ✓ Clean |
| 9 | — | Secrets in code | No hardcoded keys found; `settings.local.json` removed; `vite.config.ts` API_KEY baking removed | ✓ Clean |
| 10 | — | Unsafe JSON parsing | All LLM providers use `safeParseLLMResponse` | ✓ Clean |
| 11 | — | `alert()`/`confirm()`/`prompt()` in UI | All replaced with inline states per Audit #7 | ✓ Clean |
| 12 | MEDIUM | Prompt version drift | `PROMPT_VERSION = 'v1.2.0'` in `lib/constants.js`; verify if prompt edits since last tag need v1.3.0 | Report only |
| 13 | MEDIUM | Dead code / orphaned files | `ForExample/` (7th consecutive audit); `migrations/` scripts unreferenced | Report only |
| 14 | MEDIUM | Hebrew/RTL consistency | No new violations identified in this audit cycle | No change |

**CRITICAL open (this audit):** 4  
**HIGH open (this audit):** 2

---

## Prior Audit Resolution Status (since 2026-07-09)

| Previous Finding | Resolution |
|-----------------|------------|
| CRITICAL: Live MongoDB credentials in `.claude/settings.local.json` | ✅ Resolved — file removed from git |
| CRITICAL: `vite.config.ts` bakes `process.env.API_KEY` into client bundle | ✅ Resolved — `define` block removed; `vite.config.ts` now clean |
| CRITICAL: Missing rate limit on `POST /student/join-course` | ✅ Resolved — `submitRateLimit` added at line 456 |
| CRITICAL: IDOR on 4 lecturer read-routes (no course ownership check) | ✅ Resolved — `Course.findOne({ lecturerId })` ownership checks added |
| CRITICAL: IDOR on 2 student read-routes (no enrollment check) | ✅ Resolved — enrollment checks added at lines 553, 1009 |
| CRITICAL: Mass assignment on `/lecturer/archive` (`lecturerId` overridable) | ✅ Resolved — `lecturerId` set server-side after spread (line 438) |

All 6 findings from 2026-07-09 are resolved.

---

## CRITICAL Findings

---

### CRITICAL-1 through CRITICAL-4 — Missing Rate Limiting on Administrative Action Routes (Check #2)

**Severity:** Critical  
**Check:** Missing rate limiting (Check #2)  
**Status:** Open — new finding  
**First raised:** 2026-09-03  
**GitHub Issue:** To be opened with label `security`

#### Affected routes

| Route | File | Line | Missing limit | Body payload |
|-------|------|------|---------------|--------------|
| `POST /lecturer/submissions/:id/extension` | `api/index.js` | 903 | None | `{ extensionUntil }` |
| `POST /lecturer/courses/:id/approve` | `api/index.js` | 1310 | None | `{ studentId }` |
| `POST /lecturer/courses/:id/reject` | `api/index.js` | 1330 | None | `{ studentId }` |
| `POST /lecturer/courses/:id/remove-student` | `api/index.js` | 1349 | None | `{ studentId }` |

#### Evidence

```js
// api/index.js:903 — no rate limiter in middleware chain
router.post('/lecturer/submissions/:id/extension', async (req, res) => {

// api/index.js:1310 — no rate limiter in middleware chain
router.post('/lecturer/courses/:id/approve', async (req, res) => {

// api/index.js:1330
router.post('/lecturer/courses/:id/reject', async (req, res) => {

// api/index.js:1349
router.post('/lecturer/courses/:id/remove-student', async (req, res) => {
```

#### Established rate limiters (for reference)

| Limiter | Window | Max | Applied to |
|---------|--------|-----|------------|
| `llmRateLimit` | 1 hr | 100 | `/evaluate`, `/chat`, `/student/chat`, manual submission |
| `submitRateLimit` | 15 min | 20 | `/student/assignments/:id/submit`, `/student/join-course`, `/teacher/submit-review` |
| `uploadRateLimit` | 1 hr | 20 | `/lecturer/assignments`, `/lecturer/materials`, `/lecturer/courses`, `/lecturer/archive` |
| `messagesRateLimit` | 1 min | 60 | `POST /messages`, `PUT /messages/:id` |

#### Risk

An authenticated lecturer could spam the approve/reject/remove-student endpoints at arbitrary throughput, causing unbounded MongoDB write operations. The extension endpoint accepts an `extensionUntil` date, making it subject to the same abuse pattern. All four routes perform writes without any I/O throttle on the server side.

#### Recommended fix

Apply `uploadRateLimit` (20/hr) to the extension route and `submitRateLimit` (20/15min) to the enrollment-action routes:

```js
router.post('/lecturer/submissions/:id/extension', uploadRateLimit, async (req, res) => {
router.post('/lecturer/courses/:id/approve', submitRateLimit, async (req, res) => {
router.post('/lecturer/courses/:id/reject', submitRateLimit, async (req, res) => {
router.post('/lecturer/courses/:id/remove-student', submitRateLimit, async (req, res) => {
```

---

## HIGH Findings (Consolidated)

---

### HIGH-1 — Score Range Validation Gap in `/evaluate` (Check #6)

**Severity:** High  
**Check:** Missing output validation (Check #6)  
**Status:** Open — new finding  
**GitHub Issue:** Consolidated HIGH issue, label `code-quality`

#### Affected location

`api/index.js`, lines 785–818

#### Evidence

The `/evaluate` output schema instructs the LLM to return scores on a **0–10 scale**:

```js
// api/index.js:786-791
const outputSchema = `Return ONLY valid JSON:
{
  "score": number (0-10, where 10 is a flawless submission),
  ...
}
The score is on a 0-10 scale...`;
```

But `validateLLMOutput` (called at line 813) only enforces the **0–100 range**:

```js
// services/promptGuard.js:193-195
if (value < 0 || value > 100) {
    errors.push(`${path} out of range: ${value} (expected 0-100)`);
}
```

A malformed LLM response returning `"score": 75` would pass validation and be sent to the client, which would display it as `75/10`. The CHAM pipeline (semanticAssessment.js) correctly uses 0–100, so its use of `validateLLMOutput` is consistent. The gap is specific to the `/evaluate` endpoint.

#### Recommended fix

Add a post-validation clamp or explicit range check after line 813:

```js
const validation = validateLLMOutput(response.raw, ['score', 'feedback']);
if (!validation.valid) { /* ... */ }
const result = validation.data;
if (result.score < 0 || result.score > 10) {
  console.warn('[evaluate] LLM score out of 0-10 range:', result.score);
  return res.status(500).json({ message: 'ה-AI החזיר ציון לא תקין. נסה שוב.' });
}
```

---

### HIGH-2 — Mass Assignment via `req.body` on Three Routes (Check #6)

**Severity:** High  
**Check:** Missing output / input validation (Check #6)  
**Status:** Open — new finding  
**GitHub Issue:** Consolidated HIGH issue, label `code-quality`

#### Affected locations

| Route | File | Line | Pattern |
|-------|------|------|---------|
| `POST /lecturer/assignments` | `api/index.js` | 844 | `Assignment.create(req.body)` |
| `PUT /lecturer/assignments/:id` | `api/index.js` | 868 | `Assignment.findByIdAndUpdate(req.params.id, req.body, ...)` |
| `PUT /lecturer/courses/:id` | `api/index.js` | 1109 | `Course.findOneAndUpdate(..., req.body, ...)` |

#### Evidence

```js
// api/index.js:844
const assignment = await Assignment.create(req.body);

// api/index.js:868
const updatedAssignment = await Assignment.findByIdAndUpdate(req.params.id, req.body, { new: true });

// api/index.js:1109
const course = await Course.findOneAndUpdate({ _id: req.params.id, lecturerId: req.user.googleId }, req.body, { new: true });
```

Mongoose strict mode prevents unknown fields from persisting, but any schema-defined field can be overwritten — including `createdAt` timestamps, `assessment_status` markers that were intended to be set only by the CHAM pipeline, and internal metadata fields. An authenticated lecturer submitting `{ "createdAt": "2020-01-01" }` in the body would silently overwrite the creation timestamp.

#### Recommended fix

Destructure only the intended fields before persisting:

```js
// POST /lecturer/assignments
const { courseId, title, question, masterSolution, rubric, dueDate, openDate, language, unit_tests } = req.body;
const assignment = await Assignment.create({ courseId, title, question, masterSolution, rubric, dueDate, openDate, language, unit_tests });

// PUT /lecturer/assignments/:id
const { title, question, masterSolution, rubric, dueDate, openDate, language, unit_tests } = req.body;
const updatedAssignment = await Assignment.findByIdAndUpdate(req.params.id, { title, question, masterSolution, rubric, dueDate, openDate, language, unit_tests }, { new: true });
```

---

## MEDIUM Findings (Report Only)

---

### MEDIUM-1 — Prompt Version Drift (Check #9)

`lib/constants.js` exports `PROMPT_VERSION = 'v1.2.0'`. The audit specification lists current version as `v1.1.0`, indicating a prior bump to v1.2.0 was committed. The `SYSTEM_INSTRUCTION` and `OUTPUT_SCHEMA` in `services/semanticAssessment.js` are hardcoded strings not co-versioned with `lib/constants.js`. If any prompt template was edited since v1.2.0 was tagged, the constant must be bumped to `v1.3.0` and `package.json` updated to match.

**Action:** Verify git diff between current HEAD and `prompt-v1.2.0` tag for any changes to `SYSTEM_INSTRUCTION`/`OUTPUT_SCHEMA` in `semanticAssessment.js`, `promptGuard.js`, and `api/index.js` prompt strings.

---

### MEDIUM-2 — Orphaned / Dead Code (Check #10)

| Path | Reason | Consecutive Audits Flagged |
|------|--------|---------------------------|
| `ForExample/` | Sample input files; not imported from `package.json`, `api/index.js`, `App.tsx`, or `vercel.json` | 7th |
| `migrations/20260403-*.js` | One-time migration scripts; no runner entry point, not referenced from `package.json` scripts or main code | 1st |
| `server_reference.js` | Previously flagged — **file removed** ✓ | Resolved |

`ForExample/` has been flagged in every prior audit. If these files are truly reference examples for developers, move them to `docs/examples/` and add a note in `README.md`. If they are unused, delete them.

---

## Clean Checks (Passed)

| Check | Verdict | Notes |
|-------|---------|-------|
| C-1: Unprotected LLM call sites | ✅ Clean | All call sites: `/student/chat` (line 617), `/chat` (line 757), `/evaluate` (line 806), `semanticAssessment.js` (line 97) — all route through `LLMOrchestrator.evaluateWithFallback()` with `buildSafePrompt()` or `buildSafeChatPrompt()` |
| C-3: Session/RBAC regressions | ✅ Clean | All `/lecturer/*` routes assert `role !== 'lecturer'`; all `/student/*` routes assert `role !== 'student'`; `/teacher/*` (review queue) also asserts lecturer role inline; `/auth/dev` gated on `NODE_ENV !== 'production'` (line 367) |
| C-4: Secrets in code | ✅ Clean | `.claude/settings.local.json` removed; `vite.config.ts` API_KEY baking removed; `.env.example` contains only placeholder values |
| H-5: Unsafe JSON parsing | ✅ Clean | All providers (`groq.js:89`, `gemini.js`, `openai.js`) return `parsed: safeParseLLMResponse(content)`; orchestrator uses parsed field |
| H-7: `alert()`/`confirm()`/`prompt()` | ✅ Clean | No raw browser dialog calls found in any `.tsx` file; all replaced with inline state patterns per `// Audit #7` comments |

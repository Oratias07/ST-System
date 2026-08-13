# CHAM Agent — Weekly Security & Architecture Audit (2026-08-13)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open → 5 fixed in commit `ed1dafc`, 1 open)  
**Date:** 2026-08-13  

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| C-20 | CRITICAL | Secrets in code | MongoDB Atlas credentials remain in git history; credentials not rotated | **Open (carried — partial fix only, Issue #41)** |
| C-26 | CRITICAL | Session/RBAC — IDOR | `POST /lecturer/materials` (`api/index.js:1371`) — no course ownership check | **Open (new, Issue #56)** |
| H-5 | HIGH | Unsafe JSON parsing | All LLM response parsing uses `safeParseLLMResponse` | ✓ Clean |
| H-6 | HIGH | Missing output validation | `validateLLMOutput()` called on all evaluation paths | ✓ Clean |
| H-7 | HIGH | `alert()` in UI | No raw `alert()`/`confirm()`/`prompt()` calls in production components | ✓ Clean |
| M-8 | MEDIUM | Hebrew/RTL consistency | No hardcoded `left`/`right` CSS found; all prior RTL fixes verified present | ✓ Clean |
| M-9 | MEDIUM | Prompt version drift | `PROMPT_VERSION = 'v1.2.0'`; no commits to `lib/llm/` since tag `prompt-v1.2.0` | ✓ No drift |
| M-19 | MEDIUM | Dead code / orphaned files | `ForExample/` (6 files, 7th consecutive audit) | Report only |

**CRITICAL open:** 2 (1 pre-existing, 1 new)  
**HIGH open:** 0  

---

## Resolution Status: Prior Open CRITICAL Findings (from 2026-07-09)

| # | Finding | Committed? | Status |
|---|---------|-----------|--------|
| 21 | `API_KEY` baked into Vite client bundle | Yes (`ed1dafc`) | ✅ Fixed |
| 22 | No rate limit on `POST /student/join-course` | Yes (`ed1dafc`) | ✅ Fixed |
| 23 | IDOR — 4 lecturer read-routes missing ownership check | Yes (`ed1dafc`) | ✅ Fixed |
| 24 | IDOR — 2 student read-routes missing enrollment check | Yes (`ed1dafc`) | ✅ Fixed |
| 25 | Mass assignment in `POST /lecturer/archive` | Yes (`ed1dafc`) | ✅ Fixed |
| 20 | Live MongoDB credentials in git history | File untracked; creds NOT rotated; history NOT scrubbed | ❌ Partial only |

---

## CRITICAL Findings

---

### CRITICAL-1 — IDOR: `POST /lecturer/materials` Missing Course Ownership Check (C-26, New)

**Severity:** Critical  
**Check:** Session/RBAC regressions  
**Status:** Open — new finding  
**First raised:** 2026-08-13  
**GitHub Issue:** #56

#### Affected file

| File | Line | Description |
|------|------|-------------|
| `api/index.js` | 1371–1381 | `POST /lecturer/materials` creates a material with caller-supplied `courseId` but never verifies course ownership |

#### Evidence (verbatim, `api/index.js:1371–1381`)

```js
router.post('/lecturer/materials', uploadRateLimit, async (req, res) => {
  if (!req.user || req.user.role !== 'lecturer') return res.status(401).send();
  await connectDB();
  const { courseId, title, content, fileName, fileType, fileSize, folder, isVisible } = req.body;
  const material = await Material.create({
    courseId, title, content, fileName, fileType, fileSize, folder, isVisible,
    ownerId: req.user.googleId,
    type: 'lecturer_shared'
  });
  res.json(material);
});
```

No `Course.findOne({ _id: courseId, lecturerId: req.user.googleId })` guard is present.

#### Attack scenario

1. Attacker authenticates as Lecturer A with a valid Google account.
2. Attacker calls `POST /api/materials` with any `courseId` belonging to Lecturer B.
3. The material is stored as `type: 'lecturer_shared'` and `courseId` of Lecturer B's course.
4. Students enrolled in Lecturer B's course see the injected material via `GET /student/courses/:courseId/materials`.
5. Lecturer B sees the injected material in their own materials list.

#### Comparison with sibling routes

- `PUT /lecturer/materials/:id` (line 1383) — verifies `ownerId: req.user.googleId` ✓  
- `DELETE /lecturer/materials/:id` (line 1400) — verifies `ownerId: req.user.googleId` ✓  
- `POST /lecturer/assignments` (line 836) — verifies `Course.findOne({ _id: courseId, lecturerId })` ✓  
- `GET /lecturer/courses/:id/materials` (line 1362) — verifies `Course.findOne({ _id, lecturerId })` ✓  

Only the POST create route is missing the check.

#### Required fix (do not apply — report only)

```js
const course = await Course.findOne({ _id: courseId, lecturerId: req.user.googleId });
if (!course) return res.status(403).json({ message: 'Forbidden' });
```
to be inserted after `await connectDB();` at line 1373.

---

### CRITICAL-2 — Live MongoDB Atlas Credentials in Git History (C-20, Carried)

**Severity:** Critical  
**Check:** Secrets in code  
**Status:** Open — partial mitigation only  
**First raised:** 2026-07-09 (Issue #41)  
**Partial mitigation date:** 2026-07-14 (`git rm --cached .claude/settings.local.json`; file gitignored)

#### Current state

| Action | Done? |
|--------|-------|
| `.claude/settings.local.json` removed from working tree | ✅ |
| File added to `.gitignore` | ✅ |
| Git history scrub (`git filter-repo`) | ❌ Not done |
| MongoDB Atlas password rotation (user `Vercel-Admin-st-system-db`) | ❌ Not confirmed |

The credentials remain accessible to anyone with read access to the git repository, including historical clones and forks.

#### Required actions (do not apply — report only)

1. Rotate both Atlas passwords immediately.
2. Run `git filter-repo --path .claude/settings.local.json --invert-paths` on main.
3. Force-push cleaned history and notify all collaborators to re-clone.
4. Audit Atlas logs for unauthorized access since 2026-07-09.

---

## HIGH Findings — All Clean

### H-5: Unsafe JSON Parsing from LLM

All `JSON.parse` calls on LLM output are inside `lib/llm/safeParse.js:safeParseLLMResponse()`. No raw `JSON.parse(llmResponse)` found outside the wrapper. ✓

### H-6: Missing Output Validation

- `POST /evaluate` (`api/index.js:813`): calls `validateLLMOutput(response.raw, ['score', 'feedback'])` ✓  
- `semanticAssessment.js:105`: calls `validateLLMOutput(data || result.raw, [])` ✓  
- Free-text chat endpoints (`/chat`, `/student/chat`) return plain text, not scored evaluation output — `validateLLMOutput` not applicable. ✓

### H-7: `alert()` in UI

Grep over all `.tsx` component files finds only audit comments (`// Audit #7: replaces confirm()`). No live `alert()`, `confirm()`, or `prompt()` calls present. ✓

---

## MEDIUM Findings — Weekly Report

### M-8: Hebrew/RTL Consistency

Grep for hardcoded `margin-left`, `padding-left`, `text-align: left/right`, `float: left/right` across all `.tsx` component files returned zero matches. All prior RTL fixes from `ff569e3` (items 14–16 in `AUDIT_TRACKING.md`) are present. ✓

### M-9: Prompt Version Drift

`lib/constants.js` exports `PROMPT_VERSION = 'v1.2.0'`. Git log for `lib/llm/` and `services/promptGuard.js` shows no commits since the `prompt-v1.2.0` tag (item 17 in `AUDIT_TRACKING.md`, created 2026-07-13). No drift detected. ✓

### M-19: Dead Code / Orphaned Files

`ForExample/` directory (6 `.txt` files: `custominstr.EXMP.txt`, `mastersolutionEXMP.txt`, `questionEXMP.txt`, `rubricEXMP.txt`, `student1codeEXMP.txt`, `student2codeEXMP.txt`) remains in the repository, unreferenced from `package.json`, `api/index.js`, `App.tsx`, or `vercel.json`. This is the 7th consecutive audit reporting this finding. Deletion (`git rm -r ForExample/`) is the fix; see prior audit discussions.

`server_reference.js` — confirmed absent from working tree. ✓

`migrations/` (2 files) — standalone migration scripts, not expected to be referenced from main app entry points. Not flagged.

---

## LLM Call Site Audit (C-1)

All LLM invocations verified:

| Call site | File | Uses orchestrator? | Uses buildSafePrompt? |
|-----------|------|-------------------|-----------------------|
| `POST /evaluate` | `api/index.js:806` | ✓ `evaluateWithFallback` | ✓ `buildSafePrompt` |
| `POST /chat` (lecturer) | `api/index.js:756` | ✓ `evaluateWithFallback` | ✓ `buildSafePrompt` / `buildSafeChatPrompt` |
| `POST /student/chat` | `api/index.js:616` | ✓ `evaluateWithFallback` | ✓ `buildSafeChatPrompt` |
| `analyzeCodeQuality` | `services/semanticAssessment.js:97` | ✓ `evaluateWithFallback` | ✓ `buildSafePrompt` |

No direct Groq/Gemini/OpenAI SDK calls found outside `lib/llm/providers/`. ✓

---

## Rate Limit Coverage (C-2)

All POST/PUT routes accepting code, text, or file content are protected:

| Rate limit | Routes protected |
|-----------|----------------|
| `llmRateLimit` (100/hr) | `POST /evaluate`, `POST /chat`, `POST /student/chat`, `POST /lecturer/assignments/:id/submit-manual` |
| `submitRateLimit` (20/15min) | `POST /student/assignments/:id/submit`, `POST /student/join-course`, `POST /teacher/submit-review` |
| `messagesRateLimit` (60/min) | `POST /messages`, `PUT /messages/:id` |
| `uploadRateLimit` (20/hr) | `POST /user/update-role`, `POST /lecturer/archive`, `POST /student/private-materials`, `POST /grades/save`, `POST /lecturer/assignments`, `PUT /lecturer/assignments/:id`, `POST /lecturer/courses`, `PUT /lecturer/courses/:id`, `POST /lecturer/materials`, `PUT /lecturer/materials/:id` |

No unprotected high-risk POST/PUT routes identified. ✓

---

## RBAC Audit (C-3)

| Namespace | Check | Result |
|-----------|-------|--------|
| `/api/lecturer/*` | All routes assert `req.user.role === 'lecturer'` | ✓ |
| `/api/student/*` | All routes assert `req.user.role === 'student'` | ✓ |
| `/api/teacher/*` | All routes assert `req.user.role === 'lecturer'` | ✓ |
| `/api/chat` | Asserts lecturer role | ✓ |
| `/api/evaluate` | Asserts lecturer role | ✓ |
| `POST /auth/dev` | Returns 403 when `NODE_ENV === 'production'` | ✓ |
| `POST /grades/save` | Asserts lecturer role (not under `/lecturer/` prefix, still protected) | ✓ |

No RBAC regressions detected. ✓

# CHAM Agent — Weekly Security & Architecture Audit (2026-08-06)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open)  
**Date:** 2026-08-06

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| 1 | CRITICAL | Secrets in code | Atlas credentials in git history — file untracked & gitignored but credential rotation + `git filter-repo` scrub not confirmed | **Open (carry-forward C-1, unverified)** |
| 2 | CRITICAL | Session/RBAC regressions | `PUT /lecturer/courses/:id` passes `req.body` directly — `enrolledStudentIds`/`pendingStudentIds` injectable | **Open (new) — GitHub #54** |
| 3 | CRITICAL | Session/RBAC regressions | `GET /student/course-contacts/:courseId` missing enrollment check — any student enumerates contacts for any course | **Open (carry-forward) — GitHub #52** |
| 4 | HIGH | Unsafe JSON parsing | All LLM response parsing uses `safeParseLLMResponse` | ✓ Clean |
| 5 | HIGH | Missing output validation | `validateLLMOutput()` called on all evaluation paths | ✓ Clean |
| 6 | HIGH | `alert()` in UI | No live `alert()`/`confirm()`/`prompt()` calls in any component | ✓ Clean |
| 7 | MEDIUM | Hebrew/RTL consistency | `ReviewQueue.tsx:253,418` — `borderRight` physical property (not in RTL fix commit `ff569e3`) | Report only |
| 8 | MEDIUM | Prompt version drift | `PROMPT_VERSION = 'v1.2.0'` — no `prompt-v1.2.0` git tag; `package.json` still at `1.1.0` | Report only |
| 9 | MEDIUM | Dead code / orphaned files | `ForExample/` (6 files) — 7th consecutive audit without action | Report only |

**CRITICAL open:** 3  
**HIGH open:** 0  
**Net change from prior audit:** 6 CRITICAL closed (C-2 through C-6 confirmed fixed), 1 new CRITICAL opened (#54)

---

## Prior Audit Resolution Status (2026-07-09 → 2026-08-06)

All findings from the prior audit were patched in commits `ff569e3` (RTL) and `ed1dafc` (security). Issues closed this cycle: #1, #2, #3, #39, #42, #43, #44, #45, #46.

| Prior Finding | Fix Commit | Code Evidence | GitHub Issue | Status |
|--------------|-----------|---------------|-------------|--------|
| C-1: Atlas credentials in `settings.local.json` | `ed1dafc` (partial) | File not in git tree; `.gitignore` entry added | #41 | **Open — rotation/scrub unverified** |
| C-2: `vite.config.ts` bakes `API_KEY` into bundle | `ed1dafc` | `vite.config.ts` — no `define` block | #42 | ✓ Closed |
| C-3: No rate limit on `POST /student/join-course` | `ed1dafc` | `api/index.js:456` — `submitRateLimit` present | #43 | ✓ Closed |
| C-4: IDOR on 4 lecturer read-routes | `ed1dafc` | `api/index.js:851,1124,1139,1365` — `Course.findOne({_id, lecturerId})` | #44 | ✓ Closed |
| C-5: IDOR on 2 student read-routes | `ed1dafc` | `api/index.js:1009,553` — enrollment check present | #45 | ✓ Closed |
| C-6: Mass assignment in `POST /lecturer/archive` | `ed1dafc` | `api/index.js:436-440` — `...req.body` before server fields | #46 | ✓ Closed |
| MEDIUM-1: RTL `GradeBook.tsx`, `StudentAssignments.tsx`, `AssignmentManager.tsx` | `ff569e3` | RTL-aware `scrollBy`; `borderInlineEnd`/`paddingInlineEnd`/`text-end` | — | ✓ Resolved |

---

## CRITICAL Findings

---

### CRITICAL-1 (Carry-forward) — Atlas Credential Rotation + History Scrub Unverified

**Severity:** Critical  
**Check:** Secrets in code (Check #4)  
**Status:** Open — carry-forward from 2026-07-09  
**GitHub Issue:** #41  

#### Current State

Commit `ed1dafc` ran `git rm --cached .claude/settings.local.json` and added the file to `.gitignore`. In the current working tree, `git log --all -- .claude/settings.local.json` returns no results, suggesting history may have been scrubbed or the file was never pushed to the remote.

The commit message explicitly noted: *"C-1 (partial) … NOT fully resolved — Atlas passwords still in git history and must be rotated + scrubbed (git filter-repo)."*

#### Outstanding Action Required

This issue cannot be verified closed from a code audit alone. The repository owner must confirm:

1. **Credential rotation:** Both MongoDB Atlas usernames (`Vercel-Admin-st-system-db`) were rotated and old passwords invalidated.
2. **Remote history scrub:** `git filter-repo --path .claude/settings.local.json --invert-paths` was run and force-pushed, OR the remote never received the file.

Until confirmed, treat the credentials as potentially compromised.

---

### CRITICAL-2 (New) — `PUT /lecturer/courses/:id` Mass Assignment Allows `enrolledStudentIds` Injection

**Severity:** Critical  
**Check:** Session/RBAC regressions (Check #3)  
**Status:** Open — new finding  
**GitHub Issue:** #54  

#### Affected Route

| Route | File | Line |
|-------|------|------|
| `PUT /lecturer/courses/:id` | `api/index.js` | 1106–1111 |

#### Evidence

```js
// api/index.js:1106-1111
router.put('/lecturer/courses/:id', uploadRateLimit, async (req, res) => {
  if (!req.user || req.user.role !== 'lecturer') return res.status(401).send();
  await connectDB();
  const course = await Course.findOneAndUpdate(
    { _id: req.params.id, lecturerId: req.user.googleId },
    req.body,   // ← entire request body, no field whitelist
    { new: true }
  );
  res.json(course);
});
```

The `lecturerId` filter restricts to the lecturer's own course, but `req.body` is passed without whitelisting. A lecturer can PUT `{ enrolledStudentIds: [...], pendingStudentIds: [], code: "CHOSEN" }` to directly overwrite enrollment state.

#### Impact

1. **Forced enrollment** — any student's Google ID can be added without consent; no `WaitlistHistory` entry; no `User.unseenApprovals` notification.
2. **Forced un-enrollment** — any student silently removed by overwriting the array.
3. **Course code hijack** — `code` changed to a chosen value, breaking outstanding invite links.
4. The dedicated `/approve`, `/reject`, `/remove-student` workflow is bypassed entirely.

Note: `POST /lecturer/courses` (`api/index.js:1098`) has the same `...req.body` pattern but at creation time the severity is lower (no live enrollment to corrupt).

#### Required Fix

```js
// Whitelist only the fields the course-edit UI legitimately sends:
const { name, description } = req.body;
const course = await Course.findOneAndUpdate(
  { _id: req.params.id, lecturerId: req.user.googleId },
  { name, description },
  { new: true }
);
```

---

### CRITICAL-3 (Carry-forward) — `GET /student/course-contacts/:courseId` Missing Enrollment Check

**Severity:** Critical  
**Check:** Session/RBAC regressions (Check #3)  
**Status:** Open — carry-forward; not addressed in `ed1dafc` or `ff569e3`  
**GitHub Issue:** #52  

#### Affected Route

| Route | File | Lines |
|-------|------|-------|
| `GET /student/course-contacts/:courseId` | `api/index.js` | 479–491 |

#### Evidence

```js
// api/index.js:479-491
router.get('/student/course-contacts/:courseId', async (req, res) => {
  if (!req.user || req.user.role !== 'student') return res.status(401).send();
  await connectDB();
  const course = await Course.findById(req.params.courseId);  // no enrollment check
  if (!course) return res.status(404).send();
  const lecturer = await User.findOne({ googleId: course.lecturerId });
  const students = await User.find({ googleId: { $in: course.enrolledStudentIds, $ne: req.user.googleId } });
  res.json({
    lecturer: lecturer ? { id: lecturer.googleId, name: lecturer.name, picture: lecturer.picture } : null,
    students: students.map(u => ({ id: u.googleId, name: u.name, picture: u.picture }))
  });
});
```

This route was not included in the C-5 fix (which addressed `/student/courses/:courseId/assignments` and `/student/courses/:courseId/materials`). Any authenticated student can call this endpoint with any course ObjectId to enumerate all enrolled student names, profile pictures, and Google IDs.

#### Impact

- **PII exposure** — Google IDs, names, and profile pictures of all enrolled students in any course.
- **User enumeration at course granularity** — supplements the known `/users/all` enumeration vector (#49).
- A student who joined one course can discover the full roster of unrelated courses.

#### Required Fix

```js
// After role check, add:
if (!req.user.enrolledCourseIds?.includes(req.params.courseId)) {
  return res.status(403).json({ message: 'Not enrolled in this course' });
}
```

---

## HIGH Findings

**None.** All three HIGH-severity checks pass.

| Check | Status | Evidence |
|-------|--------|----------|
| 5 — Unsafe JSON parsing | ✓ Clean | `safeParseLLMResponse()` is the sole LLM parse path. All `JSON.parse()` calls outside `lib/llm/safeParse.js` are in test files parsing fixture/mock data. |
| 6 — Missing output validation | ✓ Clean | `validateLLMOutput()` called at `api/index.js:813` (`POST /evaluate`) and `services/semanticAssessment.js:105` (Layer 2). Both paths enforce score range [0–100] and required fields before returning. |
| 7 — `alert()` in UI | ✓ Clean | All prior `alert()`/`confirm()`/`prompt()` calls replaced with inline state (Audit #7 tags in `ResultSection.tsx`, `CourseManager.tsx`, `AssignmentManager.tsx`, `LecturerDashboard.tsx`). No live browser-dialog calls in any component. |

---

## MEDIUM Findings (Weekly Report Only)

---

### MEDIUM-1 — Hebrew/RTL Consistency (Check #8)

**Status:** Partially resolved. `GradeBook.tsx`, `StudentAssignments.tsx`, and `AssignmentManager.tsx` were fixed in `ff569e3`. `ReviewQueue.tsx` was not included in that commit.

| Location | Issue | Audit count |
|----------|-------|-------------|
| `ReviewQueue.tsx:253` | `borderRight: '4px solid #FF9800'` — physical axis; wrong side in RTL | 3rd consecutive |
| `ReviewQueue.tsx:418` | `borderRight: \`4px solid ${getPriorityColor(...)}\`` — physical axis | 3rd consecutive |

**Fix (unchanged from prior audits):** Replace with logical CSS property:
```js
// Line 253 and 418 — replace borderRight with:
borderInlineEnd: '4px solid ...'
```

---

### MEDIUM-2 — Prompt Version Drift (Check #9)

**Status:** Persistent — 7th consecutive audit.

`lib/constants.js:1` exports `PROMPT_VERSION = 'v1.2.0'`. `git tag -l 'prompt-*'` returns no results. `package.json` remains at `"version": "1.1.0"`.

**Recommended actions (unchanged):**
1. `git tag prompt-v1.2.0 <commit-of-prompt-bump>`
2. Align `package.json` to `"version": "1.2.0"`
3. Future prompt changes must bump `PROMPT_VERSION` in the same commit

---

### MEDIUM-3 — Dead Code / Orphaned Files (Check #10)

**Status:** Persistent — 7th consecutive audit.

```
ForExample/custominstr.EXMP.txt
ForExample/mastersolutionEXMP.txt
ForExample/questionEXMP.txt
ForExample/rubricEXMP.txt
ForExample/student1codeEXMP.txt
ForExample/student2codeEXMP.txt
```

Not imported from any code path. Move to `docs/examples/` or add to `.gitignore`.

`server_reference.js` — confirmed absent from current tree (issue #1 closed).  
`services/chatService.ts` — imported by `components/ChatBot.tsx`; not dead code (false alarm from prior audits).

---

## Checks With No New Findings

| Check | Result |
|-------|--------|
| 1 — Unprotected LLM call sites | ✓ Clean — All four LLM-calling paths (`POST /evaluate`, `POST /chat`, `POST /student/chat`, `services/semanticAssessment.js:94`) use `evaluateWithFallback()`. Both `buildSafePrompt()` and `buildSafeChatPrompt()` are applied before every call. No direct SDK calls outside provider classes in `lib/llm/providers/`. |
| 2 — Missing rate limiting | ✓ Clean — All POST/PUT routes accepting code, text, or file content are covered: `llmRateLimit` on LLM endpoints, `submitRateLimit` on code submission, `messagesRateLimit` on messaging, `uploadRateLimit` on content mutations. |
| 3 (partial) — `/auth/dev` production guard | ✓ Clean — `POST /auth/dev` returns 403 in production (`api/index.js:367`). |
| 4 — Secrets in tracked files | ✓ Clean — No hardcoded API keys, MongoDB URIs, or session secrets in tracked files. Credential rotation for CRITICAL-1 is a separate operational task. |

---

## Cumulative Open Items

| Finding | First Raised | GitHub | Status |
|---------|-------------|--------|--------|
| CRITICAL: Atlas credential rotation + history scrub | 2026-07-09 | #41 | **Unresolved — 2 audits (operational confirm required)** |
| CRITICAL: `PUT /lecturer/courses/:id` mass assignment — `enrolledStudentIds` injectable | 2026-08-06 | #54 | **New** |
| CRITICAL: `GET /student/course-contacts/:courseId` missing enrollment check | 2026-07-30 | #52 | **Unresolved — 2 audits** |
| CRITICAL: `GET /api/users/all` no role assertion — full user enumeration | 2026-07-30 | #49 | **Unresolved — 2 audits** |
| CRITICAL: Missing rate limits on 4 enrollment mutation routes | 2026-07-30 | #48 | **Unresolved — 2 audits** |
| MEDIUM: `ReviewQueue.tsx:253,418` physical `borderRight` | 2026-07-02 | — | **Unresolved — 3 audits** |
| MEDIUM: Prompt version tag missing | 2026-05-07 | — | **Unresolved — 7 audits** |
| MEDIUM: `ForExample/` dead files | 2026-05-07 | — | **Unresolved — 7 audits** |

---

## Issues Closed This Cycle

| Issue | Finding | Resolution |
|-------|---------|------------|
| #1 | Direct LLM SDK calls in `server_reference.js` | File deleted from codebase |
| #2 | `submit-manual` missing rate limit | `llmRateLimit` at `api/index.js:950` |
| #3 | `POST /grades/save` no role check | Role guard at `api/index.js:695` |
| #39 | `POST /student/join-course` rate limit (dupe of #43) | Closed as duplicate |
| #42 | `vite.config.ts` API_KEY bundle exposure | `define` block removed |
| #43 | `POST /student/join-course` rate limit | `submitRateLimit` at `api/index.js:456` |
| #44 | IDOR on 4 lecturer read-routes | Ownership checks at lines 851, 1124, 1139, 1365 |
| #45 | IDOR on 2 student read-routes | Enrollment checks at lines 1009, 553 |
| #46 | Mass assignment in `POST /lecturer/archive` | Server fields after spread at lines 436–440 |

---

## Recommendations (Priority Order)

1. **[IMMEDIATE]** Confirm Atlas password rotation and remote history scrub — contact repo owner to verify. If rotation was not done, rotate both credentials now. (CRITICAL-1 / #41)
2. **[THIS WEEK]** Whitelist fields in `PUT /lecturer/courses/:id` — replace `req.body` with `{ name, description }` at `api/index.js:1109`. (CRITICAL-2 / #54)
3. **[THIS WEEK]** Add enrollment check to `GET /student/course-contacts/:courseId` at `api/index.js:480`. (CRITICAL-3 / #52)
4. **[THIS WEEK]** Address `GET /api/users/all` role assertion (#49) and rate limits on approve/reject/remove-student/extension routes (#48).
5. **[SHORT-TERM]** Fix `ReviewQueue.tsx:253,418` `borderRight` → `borderInlineEnd` — 3rd consecutive audit.
6. **[CLEANUP]** `git tag prompt-v1.2.0`. Align `package.json` to `1.2.0`.
7. **[OPTIONAL]** Move `ForExample/` to `docs/examples/` — 7th consecutive audit.

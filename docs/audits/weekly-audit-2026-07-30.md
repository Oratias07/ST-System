# CHAM Agent — Weekly Security & Architecture Audit (2026-07-30)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open, 5 confirmed fixed in commit `ed1dafc`)  
**Date:** 2026-07-30

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| 1 | CRITICAL | RBAC/IDOR (Check #3) | `GET /student/course-contacts/:courseId` — no enrollment check; any student enumerates any course's full member roster | **Open (new) — Issue #52** |
| 2 | CRITICAL | Secrets in code (Check #4) | Live MongoDB Atlas credentials remain in git history (`.claude/settings.local.json` removed from tree but history not scrubbed) | **Open (ongoing) — Issue #41** |
| 3 | CRITICAL | Rate limiting (Check #2) | Missing rate limits on 4 enrollment mutation routes: `approve`, `reject`, `remove-student`, `extension` | **Open (ongoing) — Issue #48** |
| 4 | CRITICAL | RBAC (Check #3) | `GET /users/all` has no role assertion — any authenticated user (student or lecturer) enumerates all user profiles | **Open (ongoing) — Issue #49** |
| 5 | HIGH | Unsafe JSON parsing (Check #5) | All LLM response parsing routes through `safeParseLLMResponse()` | ✓ Clean |
| 6 | HIGH | Missing output validation (Check #6) | `validateLLMOutput()` called on all evaluation paths | ✓ Clean |
| 7 | HIGH | `alert()` in UI (Check #7) | No live `alert()`/`confirm()`/`prompt()` calls in production components | ✓ Clean |
| 8 | MEDIUM | Hebrew/RTL (Check #8) | All prior RTL findings resolved: `borderInlineEnd`, `paddingInlineEnd`, `text-end`, RTL-aware `scrollBy` | ✓ Clean |
| 9 | MEDIUM | Prompt version (Check #9) | `PROMPT_VERSION = 'v1.2.0'` matches git tag `prompt-v1.2.0` | ✓ Clean |
| 10 | MEDIUM | Dead code (Check #10) | `ForExample/` (6 dead `.txt` files) — no code path references them | Report only (ongoing) |

**NEW CRITICAL this audit:** 1  
**CRITICAL open (prior audits, confirmed still open):** 3  
**HIGH open:** 0  
**MEDIUM open:** 1 (ForExample/ dead files)

---

## Prior-Audit Issue Resolution

| Issue | Title | Prior Status | Current Status |
|-------|-------|-------------|----------------|
| #41 | Live MongoDB Atlas creds in git history | Open | ❌ Still open — file removed from working tree, git history not yet scrubbed |
| #42 | `vite.config.ts` bakes `API_KEY` into client bundle | Open | ✅ Fixed in commit `ed1dafc` — `define` block removed |
| #43 | Missing rate limit on `POST /student/join-course` | Open | ✅ Fixed in commit `ed1dafc` — `submitRateLimit` at `api/index.js:456` |
| #44 | IDOR: 4 lecturer read-routes missing ownership check | Open | ✅ Fixed in commit `ed1dafc` — `Course.findOne({ _id, lecturerId })` at lines 851, 1124, 1139, 1365 |
| #45 | IDOR: 2 student read-routes missing enrollment check | Open | ✅ Fixed in commit `ed1dafc` — `enrolledCourseIds.includes()` at lines 1009, 553 |
| #46 | Mass assignment in `POST /lecturer/archive` | Open | ✅ Fixed in commit `ed1dafc` — server fields after spread at `api/index.js:437-439` |
| #48 | Missing rate limits on 4 enrollment mutation routes | Open | ❌ Still open — no rate limits at lines 1310, 1330, 1349, 903 |
| #49 | `GET /users/all` no role assertion | Open | ❌ Still open — `api/index.js:396` still auth-only, no role check |

**Recommend closing:** Issues #42, #43, #44, #45, #46 — all confirmed fixed in working tree commit `ed1dafc`.

---

## CRITICAL Findings

---

### CRITICAL-1 (NEW) — IDOR: `GET /student/course-contacts/:courseId` Missing Enrollment Check

**Severity:** Critical  
**Check:** Session/RBAC regressions (Check #3)  
**Status:** Open — new finding  
**First raised:** 2026-07-30  
**GitHub Issue:** [#52](https://github.com/Oratias07/CHAM-Agent/issues/52)

#### Affected file

| File | Lines | Missing check |
|------|-------|---------------|
| `api/index.js` | 479–492 | `req.user.enrolledCourseIds.includes(req.params.courseId)` |

#### Evidence

```javascript
// api/index.js:479-492
router.get('/student/course-contacts/:courseId', async (req, res) => {
  if (!req.user || req.user.role !== 'student') return res.status(401).send();
  await connectDB();
  const course = await Course.findById(req.params.courseId); // ← no enrollment check
  if (!course) return res.status(404).send();

  const lecturer = await User.findOne({ googleId: course.lecturerId });
  const students = await User.find({ googleId: { $in: course.enrolledStudentIds, $ne: req.user.googleId } });

  res.json({
    lecturer: lecturer ? { id: lecturer.googleId, name: lecturer.name, picture: lecturer.picture } : null,
    students: students.map(u => ({ id: u.googleId, name: u.name, picture: u.picture }))
  });
});
```

The two student IDOR routes fixed in commit `ed1dafc` (`api/index.js:1009` assignments, `api/index.js:553` materials) now carry the enrollment guard. This `/course-contacts` route was not in scope for that fix and was missed.

#### Impact

Any authenticated student can query the roster (name, picture, Google ID) of **any** course by supplying a MongoDB `_id`. Course IDs are visible in browser network traffic for any course the student legitimately belongs to; from one known ID, iteration is trivial via sequential ObjectId guessing.

Exposed data per request: lecturer name + photo + Google ID, and names + photos + Google IDs of all enrolled students.

#### Required Fix

```javascript
// api/index.js:479 — add enrollment guard after role check
router.get('/student/course-contacts/:courseId', async (req, res) => {
  if (!req.user || req.user.role !== 'student') return res.status(401).send();
  if (!req.user.enrolledCourseIds?.includes(req.params.courseId)) {
    return res.status(403).json({ message: 'Not enrolled in this course' });
  }
  // rest of handler unchanged
```

This matches the pattern at `api/index.js:1009` and `api/index.js:553`.

---

### CRITICAL-2 (ONGOING) — Live MongoDB Credentials in Git History

**Severity:** Critical  
**Check:** Secrets in code (Check #4)  
**Status:** Partially mitigated — active credentials remain in git history  
**First raised:** 2026-07-09  
**GitHub Issue:** [#41](https://github.com/Oratias07/CHAM-Agent/issues/41)

The `.claude/settings.local.json` file containing live Atlas connection URIs was removed from the git index (`git rm --cached`) and added to `.gitignore`. The credentials are no longer in the working tree. However:

1. The committed history still contains the plaintext passwords.
2. Anyone with read access can recover them via `git show <commit>:.claude/settings.local.json`.
3. No evidence that the Atlas passwords have been rotated.
4. No evidence that `git filter-repo` or BFG history scrub has been run.

Until rotation + scrub complete, this remains open. This is the 4th consecutive audit cycle tracking it.

---

### CRITICAL-3 (ONGOING) — Missing Rate Limits on 4 Enrollment Mutation Routes

**Severity:** Critical  
**Check:** Missing rate limiting (Check #2)  
**Status:** Open — from prior audit (Issue #48)  
**GitHub Issue:** [#48](https://github.com/Oratias07/CHAM-Agent/issues/48)

#### Affected routes (confirmed unprotected in current `api/index.js`)

| Route | Line | Body input |
|-------|------|------------|
| `POST /lecturer/courses/:id/approve` | 1310 | `{ studentId }` |
| `POST /lecturer/courses/:id/reject` | 1330 | `{ studentId }` |
| `POST /lecturer/courses/:id/remove-student` | 1349 | `{ studentId }` |
| `POST /lecturer/submissions/:id/extension` | 903 | `{ extensionUntil }` |

All four are mutation routes accessible to authenticated lecturers, with no throttle. A credential-compromised lecturer account can flood approve/reject cycles or grant/revoke extensions without limit.

---

### CRITICAL-4 (ONGOING) — `GET /users/all` Returns All User Profiles Without Role Assertion

**Severity:** Critical  
**Check:** Session/RBAC regressions (Check #3)  
**Status:** Open — from prior audit (Issue #49)  
**GitHub Issue:** [#49](https://github.com/Oratias07/CHAM-Agent/issues/49)

```javascript
// api/index.js:396-401
router.get('/users/all', async (req, res) => {
  if (!req.user) return res.status(401).send(); // ← auth-only, no role check
  await connectDB();
  const users = await User.find({ googleId: { $ne: req.user.googleId } });
  res.json(users.map(u => ({ id: u.googleId, name: u.name, picture: u.picture })));
});
```

Any authenticated user — including a newly registered user with no role set — can enumerate every user's name, profile photo, and internal Google ID.

---

## HIGH Findings

**None.** All three HIGH-severity checks pass.

| Check | Status | Evidence |
|-------|--------|----------|
| 5 — Unsafe JSON parsing | ✓ Clean | `JSON.parse` appears only inside `lib/llm/safeParse.js` (the safe wrapper) and in test code parsing fixture files. No bare `JSON.parse(llmResponse)` anywhere in `api/`, `services/`. |
| 6 — Missing output validation | ✓ Clean | `validateLLMOutput()` called at `api/index.js:813` (`POST /evaluate`) and `services/semanticAssessment.js:105` (Layer 2). Chat endpoints operate in `jsonMode: false` and return raw text — output validation is not applicable. |
| 7 — `alert()` in UI | ✓ Clean | Grep over all `.tsx`/`.ts` files in `components/` and `App.tsx` finds zero live `alert()`, `confirm()`, or `prompt()` calls. All Audit #7 comments reference prior replacements with inline state. |

---

## MEDIUM Findings (Weekly Report Only)

---

### MEDIUM-1 — Hebrew/RTL Consistency (Check #8)

**Status:** ✓ CLEAN — all prior findings resolved.

Verified against current working tree:

| Prior finding | Location | Resolution |
|--------------|----------|------------|
| `scrollBy({ left: ... })` physical axis | `GradeBook.tsx:42` | Fixed — `isRtl ? -delta : delta` conditional |
| `text-left` on Hebrew text | `StudentAssignments.tsx:134` | Fixed — `text-end` |
| `borderRight`/`paddingRight` physical properties | `StudentAssignments.tsx:176`, `AssignmentManager.tsx:263,417` | Fixed — `borderInlineEnd`, `paddingInlineEnd` |

No new RTL regressions detected in any component.

---

### MEDIUM-2 — Prompt Version Drift (Check #9)

**Status:** ✓ CLEAN.

`lib/constants.js:1` exports `PROMPT_VERSION = 'v1.2.0'`. Git tag `prompt-v1.2.0` exists (item #17 in AUDIT_TRACKING, ✅ closed). No prompt templates in `lib/llm/` modified since the tag. No version bump required this cycle.

---

### MEDIUM-3 — Dead Code / Orphaned Files (Check #10)

**Status:** Persistent — ongoing since 2026-05-07 (item #19 in AUDIT_TRACKING).

```
ForExample/custominstr.EXMP.txt
ForExample/mastersolutionEXMP.txt
ForExample/questionEXMP.txt
ForExample/rubricEXMP.txt
ForExample/student1codeEXMP.txt
ForExample/student2codeEXMP.txt
```

`server_reference.js` is **not present** in the current tree — no action required for that file.

The 6 fixture `.txt` files in `ForExample/` are not imported from any code path (`grep` across `api/index.js`, `App.tsx`, `vercel.json`, `package.json` finds no references). Recommend `git rm -r ForExample/` or move to `docs/examples/`.

---

## LLM Pipeline Integrity Checks

All four LLM-calling code paths verified clean:

| Call site | Uses `buildSafePrompt`/`buildSafeChatPrompt` | Uses `evaluateWithFallback` | Uses `validateLLMOutput` |
|-----------|------|------|------|
| `POST /evaluate` (`api/index.js:796,806,813`) | ✓ | ✓ | ✓ |
| `POST /chat` (`api/index.js:737,748,757`) | ✓ | ✓ | N/A (non-JSON mode) |
| `POST /student/chat` (`api/index.js:609,617`) | ✓ | ✓ | N/A (non-JSON mode) |
| `services/semanticAssessment.js:85,97,105` | ✓ | ✓ | ✓ |

No direct SDK calls (`new Groq()`, `new GoogleGenAI()`, `new OpenAI()`, `fetch(GROQ_API_URL)`) outside of `lib/llm/providers/`.

---

## Architecture Note — GeminiProvider Generic `API_KEY` Fallback

`lib/llm/providers/gemini.js:15`:  
```javascript
const apiKey = process.env.GEMINI_API_KEY || process.env.API_KEY;
```

The generic `API_KEY` fallback is a confusing naming convention that was the root cause of the vite.config.ts client-bundle exposure (now fixed). Since the fix removes the client-side exposure, this is server-side only and low severity. However, the ambiguous env var name risks accidental population by another service. Recommend removing the `|| process.env.API_KEY` fallback and requiring `GEMINI_API_KEY` explicitly. No GitHub issue required — judgment call for the team.

---

## Cumulative Open Items

| Finding | First Raised | Current Status |
|---------|-------------|----------------|
| CRITICAL: Live MongoDB creds in git history (Issue #41) | 2026-07-09 | ❌ Open — 4th consecutive audit |
| CRITICAL: `GET /student/course-contacts/:courseId` enrollment check missing (Issue #52) | 2026-07-30 | ❌ Open — new this audit |
| CRITICAL: Missing rate limits on 4 enrollment mutation routes (Issue #48) | Prior cycle | ❌ Open |
| CRITICAL: `GET /users/all` no role assertion (Issue #49) | Prior cycle | ❌ Open |
| MEDIUM: `ForExample/` dead fixture files | 2026-05-07 | ❌ Open — 9th consecutive audit |

---

## Recommendations (Priority Order)

1. **[IMMEDIATE]** Fix `GET /student/course-contacts/:courseId` (`api/index.js:479`) — add one enrollment guard. (CRITICAL-1, Issue #52)
2. **[IMMEDIATE]** Rotate MongoDB Atlas credentials and run `git filter-repo` history scrub. (CRITICAL-2, Issue #41)
3. **[THIS WEEK]** Add rate limits to 4 enrollment mutation routes. (CRITICAL-3, Issue #48)
4. **[THIS WEEK]** Add role assertion to `GET /users/all`. (CRITICAL-4, Issue #49)
5. **[HOUSEKEEPING]** Close Issues #42, #43, #44, #45, #46 — all confirmed fixed in commit `ed1dafc`.
6. **[OPTIONAL]** Remove `|| process.env.API_KEY` fallback from `lib/llm/providers/gemini.js:15`.
7. **[OPTIONAL]** `git rm -r ForExample/` — 9th consecutive audit carrying this item.

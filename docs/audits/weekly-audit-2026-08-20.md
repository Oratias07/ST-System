# CHAM Agent — Weekly Security & Architecture Audit (2026-08-20)

**Auditor:** Claude (automated)  
**Scope:** Full codebase — `api/index.js`, `lib/llm/`, `services/`, `components/`, `App.tsx`, `vite.config.ts`, `.gitignore`  
**Prior audit:** `docs/audits/weekly-audit-2026-07-09.md` (6 CRITICAL open)  
**Date:** 2026-08-20

---

## Summary Table

| # | Severity | Check | Finding | Status |
|---|----------|-------|---------|--------|
| 1 | CRITICAL | Secrets in code | `settings.local.json` MongoDB credentials (prior CRITICAL-1) | ✓ **Resolved** |
| 2 | CRITICAL | Secrets in code | `vite.config.ts` bakes `API_KEY` into client bundle (prior CRITICAL-2) | ✓ **Resolved** |
| 3 | CRITICAL | Missing rate limiting | `POST /student/join-course` missing rate limit (prior CRITICAL-3) | ✓ **Resolved** |
| 4 | CRITICAL | Session/RBAC | IDOR — 4 lecturer read-routes without ownership check (prior CRITICAL-4) | ✓ **Resolved** |
| 5 | CRITICAL | Session/RBAC | IDOR — 2 student read-routes without enrollment check (prior CRITICAL-5) | ✓ **Resolved** |
| 6 | CRITICAL | Session/RBAC | Mass assignment in `POST /lecturer/archive` (prior CRITICAL-6) | ✓ **Resolved** |
| 7 | HIGH | Unsafe JSON parsing | All LLM response parsing uses `safeParseLLMResponse` | ✓ Clean |
| 8 | HIGH | Missing output validation | `validateLLMOutput()` called on all evaluation paths | ✓ Clean |
| 9 | HIGH | `alert()` in UI | No raw `alert()`/`confirm()`/`prompt()` calls in production components | ✓ Clean |
| 10 | MEDIUM | Hebrew/RTL consistency | `ReviewQueue.tsx:253,418` — `borderRight` physical property | **Open** (3rd audit) |
| 11 | MEDIUM | Prompt version drift | `PROMPT_VERSION = 'v1.2.0'` has no `prompt-v1.2.0` git tag | **Open** (7th audit) |
| 12 | MEDIUM | Dead code / orphaned files | `ForExample/` (6 files) | **Open** (7th audit) |

**CRITICAL open:** 0  
**HIGH open:** 0  
**New findings:** 0

---

## Prior Audit Resolution Status (2026-07-09)

All 6 CRITICAL findings from the 2026-07-09 audit are resolved as of commit `ed1dafc`.

| Previous Finding | Resolution |
|-----------------|------------|
| CRITICAL-1: Live MongoDB credentials in `settings.local.json` (git-tracked) | ✓ File removed from repo, no history (`git log --all -- .claude/settings.local.json` returns empty), `.gitignore` line 7 now excludes the file |
| CRITICAL-2: `vite.config.ts` bakes `API_KEY` into client bundle | ✓ `define` block removed from `vite.config.ts`; file now contains only `plugins` and dev server `proxy` |
| CRITICAL-3: Missing rate limit on `POST /student/join-course` | ✓ `submitRateLimit` added at `api/index.js:456` |
| CRITICAL-4: IDOR — 4 lecturer read-routes without course ownership check | ✓ `Course.findOne({ _id: ..., lecturerId: req.user.googleId })` added to all 4 routes (lines 851, 1124, 1139, 1365); each returns 403 if check fails |
| CRITICAL-5: IDOR — 2 student read-routes without enrollment check | ✓ `enrolledCourseIds.includes(courseId)` guard added at `api/index.js:1009` and `553` |
| CRITICAL-6: Mass assignment in `POST /lecturer/archive` (`lecturerId` overridable) | ✓ `lecturerId` and `timestamp` are now set after `...req.body` at `api/index.js:437–439` |

Additionally, commit `ff569e3` resolved 4 of 5 outstanding RTL MEDIUM findings:
- `GradeBook.tsx:42` now uses `isRtl ? -delta : delta` (RTL-aware scroll)
- `StudentAssignments.tsx:134` `text-left` — **Resolved**
- `StudentAssignments.tsx:176` `borderRight`/`paddingRight` — **Resolved**
- `AssignmentManager.tsx:263,417` `borderRight`/`paddingRight` — **Resolved**
- `ReviewQueue.tsx:253,418` `borderRight` — **Unresolved** (see MEDIUM-1 below)

---

## CRITICAL Findings

**None.** Zero CRITICAL findings.

---

## HIGH Findings

**None.** All three HIGH-severity checks pass.

| Check | Status | Evidence |
|-------|--------|----------|
| 5 — Unsafe JSON parsing | ✓ Clean | `safeParseLLMResponse()` is the sole parse path for all LLM output. The only `JSON.parse` calls are inside `lib/llm/safeParse.js` (the implementation) and test fixtures. No bare `JSON.parse(llmResponse)` in `api/`, `services/`, or component files. |
| 6 — Missing output validation | ✓ Clean | `validateLLMOutput()` called at `api/index.js:813` (`POST /evaluate`) and `services/semanticAssessment.js:105` (Layer 2). Chat routes use `jsonMode: false` (free-text, no JSON to validate). |
| 7 — `alert()` in UI | ✓ Clean | `grep -r "alert\s*\(\|confirm\s*\(\|prompt\s*\("` across `components/` returns only comment annotations referencing prior replacements. `App.tsx` uses inline message state throughout. |

---

## MEDIUM Findings (Weekly Report Only)

---

### MEDIUM-1 — RTL Physical Property in `ReviewQueue.tsx` (Check #8)

**Status:** Open — 3rd consecutive audit.

| Location | Issue |
|----------|-------|
| `ReviewQueue.tsx:253` | `borderRight: '4px solid #FF9800'` — physical axis; incorrect direction in RTL layout |
| `ReviewQueue.tsx:418` | `borderRight: \`4px solid ${getPriorityColor(item.priority)}\`` — same |

All other RTL findings from prior audits were resolved in commit `ff569e3`.

**Required fix:** Replace both occurrences with `borderInlineEnd` (logical property, direction-agnostic):

```tsx
// Line 253
borderInlineEnd: '4px solid #FF9800',

// Line 418
borderInlineEnd: `4px solid ${getPriorityColor(item.priority)}`,
```

---

### MEDIUM-2 — Prompt Version Drift (Check #9)

**Status:** Open — 7th consecutive audit. No prompt template changes since last audit.

- `lib/constants.js:1` exports `PROMPT_VERSION = 'v1.2.0'`
- `git tag -l` returns no tags; no `prompt-v1.2.0` tag exists
- Prompt templates in `lib/llm/` and `services/promptGuard.js` are unchanged since 2026-07-09 (no commits touching these files since last audit)

Because no prompt templates changed this cycle, `PROMPT_VERSION` does not need a bump — only the missing tag needs to be created retroactively.

**Required action:**
```sh
git tag prompt-v1.2.0 <commit-hash-when-v1.2.0-was-set>
git push origin prompt-v1.2.0
```

---

### MEDIUM-3 — Dead Code / Orphaned Files (Check #10)

**Status:** Open — 7th consecutive audit.

```
ForExample/custominstr.EXMP.txt
ForExample/mastersolutionEXMP.txt
ForExample/questionEXMP.txt
ForExample/rubricEXMP.txt
ForExample/student1codeEXMP.txt
ForExample/student2codeEXMP.txt
```

Not imported from any code path. Not referenced in `package.json`, `api/index.js`, `App.tsx`, or `vercel.json`.

`server_reference.js` is not present in the current tree (resolved in an earlier audit).

**Recommended action:** Move to `docs/examples/` or remove if no longer needed.

---

## Checks With No Findings

| Check | Result |
|-------|--------|
| 1 — Unprotected LLM call sites | ✓ Clean — All four LLM call sites (`POST /evaluate`, `POST /chat`, `POST /student/chat`, `services/semanticAssessment.js:97`) use `evaluateWithFallback()` AND `buildSafePrompt()`/`buildSafeChatPrompt()`. Direct `fetch` calls to provider URLs exist only inside `lib/llm/providers/*.js` (the intended implementation). |
| 2 — Missing rate limiting | ✓ Clean — All POST/PUT routes accepting user-controlled text, code, or file content are rate-limited. `POST /student/join-course` (previously unprotected) now has `submitRateLimit`. |
| 3 — Session/RBAC | ✓ Clean — Every `/lecturer/*` route (including `/teacher/*` equivalents) asserts `role === 'lecturer'`. Every `/student/*` route asserts `role === 'student'`. `POST /auth/dev` returns 403 in production (`api/index.js:367`). All ownership/enrollment cross-checks from CRITICAL-4/5 are now present. |
| 4 — Secrets in code | ✓ Clean — No hardcoded credentials found. `settings.local.json` is absent and gitignored. `vite.config.ts` has no `define` block. `.env.example` contains only placeholder values. |

---

## Cumulative Open Items

| Finding | First Raised | Consecutive Audits |
|---------|-------------|-------------------|
| MEDIUM: `ReviewQueue.tsx:253,418` physical `borderRight` | 2026-07-02 | 3 |
| MEDIUM: Prompt version tag `prompt-v1.2.0` missing | 2026-05-07 | 7 |
| MEDIUM: `ForExample/` dead files | 2026-05-07 | 7 |

---

## Recommendations (Priority Order)

1. **[SHORT-TERM]** Fix `ReviewQueue.tsx:253,418` — two-line change to `borderInlineEnd`. Now in 3rd consecutive audit; no technical blocker.
2. **[CLEANUP]** `git tag prompt-v1.2.0 <commit>` — retroactive tag for an already-set constant.
3. **[OPTIONAL]** Move `ForExample/` to `docs/examples/` or delete.

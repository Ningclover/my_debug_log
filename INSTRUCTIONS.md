# Debug Log Instructions

A personal record of bugs found, diagnosed, and fixed in WireCell (or related tools). Each entry is a standalone markdown file. This file describes the conventions.

---

## When to write a log entry

Write an entry when:
- A bug is **confirmed** — root cause identified and verified by debug output or code inspection
- The fix is known (even if not yet applied)

Do **not** write an entry for:
- Unconfirmed hypotheses or guesses
- Transient issues (wrong config, typo, missing file)
- Bugs that are immediately obvious without investigation

---

## File naming

```
YYYY-MM-DD-<short-slug>.md
```

- Date = the day the bug was confirmed (not when it first appeared)
- Slug = 2–5 lowercase words with hyphens describing the issue
- Examples:
  - `2026-04-14-clustering-sort-corruption.md`
  - `2026-03-21-imaging-null-steiner-pc.md`

---

## Page structure

```markdown
# Bug: <short title>

**Date**: YYYY-MM-DD
**File**: path/to/file.cxx
**Function**: function_name
**Symptom**: the error message or observable behavior
**Introduced by**: what change caused it (commit, rebase, refactor) — omit if unknown or if it is a pre-existing bug

---

## Symptom

What the user sees. Paste the actual error message or crash output.

---

## Root Cause

Explain WHY the bug happens. Include:
- The specific line(s) of code that are wrong
- Why they are wrong (incorrect assumption, violated invariant, UB, etc.)
- A concrete example showing the failure mode

---

## Code: Before and After  *(include only if the bug was introduced by a known code change)*

Show the relevant code in both states.

### Before (working)
\`\`\`cpp
// original code
\`\`\`

### After (broken)
\`\`\`cpp
// broken code
\`\`\`

---

## Buggy Code  *(include if there is no known "before" — i.e. a pre-existing bug)*

Show the relevant buggy code and explain what is wrong.

---

## Fix

Describe the fix. Show the corrected code if applicable.

---

## Debugging Path Summary

Numbered list of the steps taken to locate the bug. Be brief — one line per step.

1. First observation / where crash was reported
2. First debug print added / first narrowing step
3. ...
4. Key output that confirmed root cause
```

---

## Rules

1. **Confirmed only.** Only write what is proven by actual output or definitive code inspection — not guesses.
2. **No debug artifacts.** Do not paste lists of debug print statements added during investigation. Only paste the key output that confirmed the root cause.
3. **Root cause explanation must include why**, not just what. "Line 903 uses `std::sort`" is not enough — explain why `std::sort` is wrong here.
4. **Reference commit** — when the bug was introduced by a specific change, always record the last known good commit hash.
5. **Keep it short.** The debugging path summary should be a numbered list, not prose.

---

## Index

| Date | File | Bug |
|------|------|-----|
| 2026-04-14 | `clus/src/clustering_extend.cxx` | [Heap corruption from invalid `std::sort` comparator in `get_strategic_points`](2026-04-14-clustering-sort-corruption.md) |

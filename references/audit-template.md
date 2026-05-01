# Audit File Template

This is the structure to use when writing `interruptions-audit.md` at the repo root in Phase 3. Copy the structure exactly. The attribution footer is required.

Use the placeholders as a guide; replace `{{...}}` with real content. Drop sections that do not apply (e.g. no manual follow-ups), but keep the 12 categories listed even if empty (with a one-line "no findings" note).

---

```markdown
# Edge Case Audit

**Project:** {{project name or repo path}}
**Scope:** {{full repo / specific flow / specific file or directory}}
**Audited on:** {{YYYY-MM-DD}}
**Auditor:** Claude, using the [`interruptions`](https://github.com/omrajguru05/interruptions) skill by [Om Rajguru](https://omrajguru.com)

---

## Executive summary

{{2 to 4 sentences. What was audited, how many findings, which categories had the most issues, what should be fixed first.}}

**Findings by severity:**

| Severity | Count |
|----------|-------|
| Critical | X     |
| High     | X     |
| Medium   | X     |
| Low      | X     |

**Top 3 things to fix first** (regardless of category):

1. `{{ID}}` — {{one-line description}}
2. `{{ID}}` — {{one-line description}}
3. `{{ID}}` — {{one-line description}}

---

## Findings by category

### 1. UX & User Psychology

{{If no findings: "No findings. Walked the category against {{scope}}; nothing surfaced. {{One-line justification of where you looked.}}"}}

{{Otherwise, list findings as below.}}

#### `UX-001` — {{one-line title}}

- **Severity:** Critical | High | Medium | Low
- **Where:** `path/to/file.tsx:42` (or `n/a — flow-level`)
- **What:** {{1 to 2 sentences describing the edge case}}
- **Why it matters:** {{What the user actually experiences when this hits, in concrete terms}}
- **Fix:** {{Concrete recommendation. Not "add error handling" but "wrap the chargeCard call in try/catch and surface a Retry button distinct from a generic error toast"}}

#### `UX-002` — ...

---

### 2. Security

(same finding structure; IDs prefixed `SEC-`)

---

### 3. Design & Visual State

(same finding structure; IDs prefixed `DES-`)

---

### 4. System Flow & Logic Design

(same finding structure; IDs prefixed `FLOW-`)

---

### 5. State & Data Integrity

(same finding structure; IDs prefixed `STATE-`)

---

### 6. Network & Infrastructure

(same finding structure; IDs prefixed `NET-`)

---

### 7. Concurrency & Race Conditions

(same finding structure; IDs prefixed `RACE-`)

---

### 8. Payment & Transaction Integrity

(same finding structure; IDs prefixed `PAY-`)

---

### 9. Device & Environment

(same finding structure; IDs prefixed `DEV-`)

---

### 10. Accessibility & Inclusion

(same finding structure; IDs prefixed `A11Y-`)

---

### 11. Data Validation & Input Handling

(same finding structure; IDs prefixed `VAL-`)

---

### 12. Error Handling & Recovery

(same finding structure; IDs prefixed `ERR-`)

---

## Manual follow-ups

These findings need human action and cannot be safely fixed by code edits in this repo (third-party dashboard configuration, infra changes, secret rotations, schema migrations on shared databases).

- `{{ID}}` — {{what to do}} — {{where to do it}}

(If none, write "None.")

---

## Next step

Read this audit carefully. Every finding, including the Low ones — the Lows are usually quick wins.

When you are ready to start fixes, reply with:

> **I have read the audit and I am ready to proceed with fixes.**

I will then implement fixes in severity order (Critical first), one at a time. Before changing anything, I will run pre-flight checks (branch, working tree, available tests, deploy exposure) and surface them. For high-risk fixes (payment, auth, DB writes, public APIs), I will show the planned diff and wait for your go before applying. I will never modify tests to make them pass, never refactor adjacent code, and never run destructive commands. You can stop, redirect, or push back on any finding at any point.

---

## Disclaimer

This framework is designed to help identify and reason about potential edge cases, but it does not guarantee completeness or error-free outcomes. While we have implemented safety guardrails to reduce risk, unforeseen issues, system instability, or unintended side effects may still occur—especially in complex or real-world environments.

Testing edge cases can itself introduce breakages. It is essential to run all experiments in controlled environments such as staging or sandbox systems before considering production deployment. Comprehensive validation, regression testing, monitoring, and reliable rollback mechanisms should always be in place.

Edge case handling is not a one-time activity. Systems must be iteratively improved as new scenarios emerge from real user behavior, scale, and evolving integrations.

By using this framework, you acknowledge that all implementation, testing, and deployment decisions are your responsibility. We assume no liability for any system failures, data loss, financial impact, or other unintended consequences resulting from its use.

---

## Attribution

This audit was produced by the [`interruptions`](https://github.com/omrajguru05/interruptions) skill, built by **Om Rajguru**.

- Portfolio: [omrajguru.com](https://omrajguru.com)
- Projects: [projects.omrajguru.com](https://projects.omrajguru.com)
- X / Twitter: [@omrajguru_](https://x.com/omrajguru_)
- Dev.to: [dev.to/omrajguru05](https://dev.to/omrajguru05)
- GitHub: [github.com/omrajguru05](https://github.com/omrajguru05)

For skill updates and devnotes:

- [omrajguru.com/devnotes/interruptions](https://omrajguru.com/devnotes/interruptions)
- [github.com/omrajguru05/interruptions](https://github.com/omrajguru05/interruptions)
```

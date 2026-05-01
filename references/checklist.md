# Edge Case Audit Checklist

This is the working reference the `interruptions` skill uses during Phase 2 (Audit). Walk every category in order. For each, ask the questions, look at the code, and record findings.

## Severity rubric

Apply this consistently. Inconsistent severity is worse than no severity at all — the user uses severity to decide what to fix tonight versus next sprint, so getting it wrong wastes their time.

- **Critical** — Money loss, data loss, security breach, or blocks users from completing the core action of the product. Must fix before ship.
- **High** — Degrades trust, generates support tickets, breaks for real users in real-world conditions (bad network, mid-tier device). Fix this sprint.
- **Medium** — Annoying but recoverable. Affects a subset of users or a non-core path. Fix when convenient.
- **Low** — Polish. Edge of edges. Does not affect functionality but reflects sloppiness. Backlog.

When in doubt, err one level higher. Underrating severity is the more dangerous mistake.

## Table of contents

1. [UX & User Psychology](#1-ux--user-psychology)
2. [Security](#2-security)
3. [Design & Visual State](#3-design--visual-state)
4. [System Flow & Logic Design](#4-system-flow--logic-design)
5. [State & Data Integrity](#5-state--data-integrity)
6. [Network & Infrastructure](#6-network--infrastructure)
7. [Concurrency & Race Conditions](#7-concurrency--race-conditions)
8. [Payment & Transaction Integrity](#8-payment--transaction-integrity)
9. [Device & Environment](#9-device--environment)
10. [Accessibility & Inclusion](#10-accessibility--inclusion)
11. [Data Validation & Input Handling](#11-data-validation--input-handling)
12. [Error Handling & Recovery](#12-error-handling--recovery)

---

## 1. UX & User Psychology

### Why this category matters

Users are not rational. They get distracted, impatient, and confused. They form mental models that diverge from what the system actually does, then they act on those wrong models. A robust app tolerates user behavior that engineers find frustrating ("why would they ever do that?"). The system must absorb the irrationality, because the user will not change.

### Questions to ask

- What happens if the user navigates away mid-action (back button, tab close, app backgrounded, OS task switch)? Does the partial action complete, get cancelled, or hang in a weird intermediate state?
- What happens if the user double-clicks a primary action button? Triple-clicks?
- What happens if the user refreshes the page mid-flow? Is the cart still there? Is the form still filled?
- Is there any action where latency might make the user assume failure and retry, causing a duplicate?
- Are confirmation screens skippable, dismissible, or easy to misread?
- If the user opens the same flow in two tabs, what happens?
- If autofill inserts wrong data (shared device, family member's saved info), does the user have a chance to notice before submission?
- After a long-running action, can the user tell whether it succeeded just by looking at the screen?
- Is there any place where doing nothing (sitting on a screen for 10 minutes) causes a worse outcome than acting? If so, is the user warned?

### Red flags in code

- `onClick` handlers that do not disable the button or set a loading state immediately
- Forms without idempotency keys on the request payload
- "Back" or "Cancel" buttons in flows that mutate state without rollback logic
- Confirmation pages that auto-redirect after a timer (users miss them)
- Routes that mutate on `GET` (browser prefetch can trigger them)

### Example fixes

- Disable the primary action button on click; re-enable only on response or timeout
- Send an idempotency key with every state-mutating request; have the backend dedupe
- On `beforeunload` for in-progress flows that have unsaved state, prompt the user
- Show a clear "what just happened, what happens next" message after long-running actions
- Mutations only on `POST`/`PUT`/`PATCH`/`DELETE`, never on `GET`

---

## 2. Security

### Why this category matters

Edge cases here are not just bugs. They are exploits. A user who finds them by accident is annoying. A user who finds them deliberately costs you money, data, or reputation. Treat every external input as hostile, even if the user is "logged in" — sessions get stolen, accounts get phished, employees get curious.

### Questions to ask

- Can the user replay a prior request (capture and resend) to trigger duplicate side effects, like a second charge or a second coupon use?
- Can the user manipulate IDs in the URL or payload to access resources that belong to someone else? (IDOR — Insecure Direct Object Reference)
- Is anything price-sensitive (totals, discounts, line items, shipping costs) trusted from the client?
- Are tokens and sessions revoked when they should be (logout, password change, role change)? Are expired tokens actually rejected at the API edge?
- Is there any endpoint where authentication is checked but authorization (does this user own this resource?) is not?
- Can a request that succeeds on the frontend ("payment confirmed!") be triggered by a crafted client request without the corresponding backend state?
- Are coupons, credits, or limited-use tokens validated and consumed atomically?
- Are file uploads constrained by type, size, and content (not just extension)?
- Is sensitive data (tokens, keys, PII) ever logged, sent to error tracking, or exposed in client-side stack traces?
- Are CORS, CSRF, and clickjacking protections in place where they should be?
- If your app supports SSO/OAuth, are the redirect URIs strictly validated?

### Red flags in code

- Endpoints that take an `id` parameter and return data without checking ownership
- `req.body.price`, `req.body.total`, or `req.body.amount` used directly in a charge call
- JWT verification that does not check expiry, issuer, or audience
- Missing rate limits on auth endpoints (login, password reset, OTP)
- Stack traces or DB error messages exposed in HTTP responses
- API keys hardcoded in source or in `.env` files committed to git
- `dangerouslySetInnerHTML`, `innerHTML`, or template literals injecting user input into the DOM
- SQL via string concatenation or template literals
- `Access-Control-Allow-Origin: *` on endpoints with credentials
- OAuth `redirect_uri` validated by `startsWith` rather than exact match

### Example fixes

- Add ownership checks on every authorized endpoint: `if (resource.userId !== req.user.id) return 403`
- Recompute totals and prices server-side from product IDs, never trust the client
- Use idempotency keys on all payment-side actions
- Atomic coupon consumption: single SQL update with `WHERE used = false RETURNING *`
- Parameterized queries everywhere
- Strict redirect URI allowlist (exact match, not prefix)
- Rate limit auth endpoints by IP and by account; lock account after N failures
- Scrub PII from logs and error reports

---

## 3. Design & Visual State

### Why this category matters

If the UI does not clearly communicate state, the user invents their own model — usually a wrong one. A spinner that runs forever after a successful payment causes the user to retry, hit support, or lose trust. The UI is not decoration. It is the user's only window into what the system did. When the window is dirty, the user breaks things trying to clean it.

### Questions to ask

- Is there a visually distinct state for: idle, loading, success, error, partial success, retrying, timeout?
- Does every async action have a timeout that surfaces to the user, not an infinite spinner?
- Are error messages specific and actionable, or generic ("Something went wrong, please try again")?
- After a state-changing action, does the UI confirm what just happened with enough detail that the user could describe it to support (order ID, amount, item)?
- Are buttons disabled while their action is in flight?
- Is there any case where the UI shows failure but the backend succeeded, or vice versa?
- Do critical screens show a freshness indicator if data might be stale?
- Are skeleton loaders used in a way that misrepresents what is loading (e.g. showing fake content shapes when there might actually be zero results)?

### Red flags in code

- `try { await x() } catch { showToast('Error') }` with no error detail
- Loading states without timeout fallback
- Buttons whose disabled state depends only on form validity, not on in-flight requests
- Success pages that do not show what was just done
- "Saved!" toasts that fire before the save actually completes (optimistic UI without rollback)
- Empty states that look like loading states (no clear "no results" UI)

### Example fixes

- Standardize a status enum (`idle | loading | success | error | timeout`) per async action
- Show order/transaction ID on confirmation; let the user copy it with one click
- Add a 30s client-side timeout that surfaces "still working, check your email if you do not see this update soon" rather than spinning forever
- Distinct empty states: loading, zero results, filtered to zero results, error

---

## 4. System Flow & Logic Design

### Why this category matters

The biggest production failures are usually not single-line bugs. They are flows that look fine on paper but fall apart in the gaps between steps. Order-then-payment vs payment-then-order. Cancel that does not actually cancel. Webhooks that arrive before the request that triggered them returns. Walk the flow as a state machine, not as a happy path.

### Questions to ask

- Draw the happy path. Now draw every transition that could fail. What is the system's state if step N fails after step N-1 succeeded?
- Does cancel/back actually undo, or does it just hide the UI while leaving partial state behind?
- Is there a fallback path for when an async callback (webhook, push notification, email) never arrives?
- Are there any flows where order-of-operations matters but is not enforced?
- Can the user reach a state that has no exit (modal with no close, error page with no retry, "pending" status that never resolves)?
- Is there a reconciliation job that catches inconsistent states (orders without payments, payments without orders, users without verified emails after 7 days)?
- For multi-step wizards: can the user enter step 3 directly via URL? Should they be able to?
- For features behind a flag, what is the user experience if the flag flips mid-session?

### Red flags in code

- Multi-step flows without an explicit `status` field on the entity (e.g. order without `pending|paid|shipped|cancelled|failed`)
- Webhook handlers that assume the originating request finished first
- "Cancel" handlers that delete UI state but not backend state
- No background job to retry or reconcile failed transitions
- Status values that overlap or are ambiguous (`active` vs `enabled` vs `live`)

### Example fixes

- Make the entity carry an explicit state machine with allowed transitions (and reject invalid ones with an error, not a silent no-op)
- Idempotent webhook handlers that reconcile by event ID rather than assuming order
- Cancel actions that explicitly call a cancel endpoint and verify the response before clearing UI
- Daily reconciliation job that flags orphan records (payment without order, order pending > 24h, etc.)
- Document the state machine somewhere a future engineer will find it

---

## 5. State & Data Integrity

### Why this category matters

Modern apps have multiple sources of truth: backend, frontend cache, browser storage, multiple tabs, multiple devices. When they disagree, the user sees a contradiction and assumes the system is broken — often correctly. State integrity is the discipline of keeping them synchronized, or making the conflict explicit when sync is impossible.

### Questions to ask

- If the backend says one thing and the frontend cache says another, who wins, and how does the user know?
- Does the cart persist across devices? Across tabs? What happens if the user adds the same item in two tabs?
- Are inventory, prices, and availability re-validated at checkout, or trusted from the cart object?
- What happens if the cart "expires" but the user is mid-payment?
- If the user's coupon was valid 30 seconds ago but is now exhausted (someone else used the last one), where does the conflict surface?
- Are there any places where stale cached data is shown without a freshness indicator?
- For collaborative apps: what happens if two users edit the same record at the same time? Last-write-wins, merge, or conflict UI?
- Does soft-delete (vs hard-delete) leak into queries that should hide deleted records?

### Red flags in code

- Cart state stored only in localStorage with no backend sync, or backend sync that overwrites localStorage indiscriminately
- Prices held in cart objects rather than recomputed at checkout submission
- No `updated_at` checks when patching records (missing optimistic concurrency)
- React Query / SWR with `staleWhileRevalidate` but no UI indicator of staleness on critical screens
- Manual cache invalidation scattered across the codebase (high chance of missing one)

### Example fixes

- Backend cart as source of truth; client renders, does not own
- Re-validate cart contents (prices, inventory, coupons, shipping eligibility) at checkout submission and surface any deltas before charging
- Optimistic concurrency with version numbers on mutable records; reject stale updates with 409
- Explicit "this data may be stale, last refreshed Xs ago" indicator on critical dashboards

---

## 6. Network & Infrastructure

### Why this category matters

The network is hostile. Requests get sent and lost. Responses get sent and lost. Timeouts fire while the server is mid-write. Mobile users on bad signal will hit every one of these every day. A robust app handles network failure as the default case, not the exception.

### Questions to ask

- What does the client do when a request times out? Retry? Surface the failure? Both?
- If the request was a mutation and the response was lost, will retrying cause a duplicate on the backend?
- Does every mutation have an idempotency key?
- What happens if a third-party CDN dependency (analytics, payment script, fonts, A/B testing) fails to load? Does the app still work?
- Is there a degraded mode for slow connections, or does the UI just hang?
- Are uploads chunked and resumable, or all-or-nothing?
- Are timeouts set explicitly, or relying on defaults that may be too long?
- Does the app handle the case where the user is online but the API is down (status page integration, cached fallbacks)?

### Red flags in code

- `fetch(url, { method: 'POST' })` without retry logic AND without idempotency keys
- Hard dependencies on third-party scripts in critical paths (e.g. payment form blocked on analytics script loading)
- No client-side timeout on requests (default browser timeout is too long for a UI to hang on)
- Uploads via a single POST with no resumability for large files
- No offline indicator or queue when connection drops

### Example fixes

- Idempotency keys on every mutation; retry with the same key on timeout, never a fresh one
- Critical scripts (payment, auth) loaded with explicit error handling and graceful fallback
- Reasonable client timeouts (10s for reads, 30s for writes is a starting point) with user-visible failure states
- Resumable uploads via signed URL chunked protocol (S3 multipart, tus, etc.)
- Detect offline state and queue mutations; surface a clear "you are offline" indicator

---

## 7. Concurrency & Race Conditions

### Why this category matters

When multiple things happen at once, the order matters. Two users buying the last item. The same user clicking pay twice. A coupon being used in two tabs. A webhook arriving before the request returns. Race conditions are invisible in development (where you have one user, one tab, one process) and unavoidable at scale (where everything is racing everything).

### Questions to ask

- Does any "check then act" code (check inventory, then decrement; check coupon, then mark used) use a database constraint or transaction, or does it just trust the gap between check and act?
- Can the same user trigger the same action twice in parallel (two tabs, double-click, retry, mobile + desktop)?
- Are limited resources (inventory, coupons, seats, slots) decremented atomically?
- If two webhooks for the same entity arrive at the same time, what happens?
- Are background jobs idempotent? What if the same job runs twice because of a queue retry?
- Is cron locked, or can two instances run the same scheduled job simultaneously?

### Red flags in code

- `if (item.stock > 0) { item.stock -= 1; await save() }` — classic TOCTOU (time-of-check, time-of-use)
- Coupon validation that checks `used: false` then later does `update used: true` without a transaction
- No `UNIQUE` constraints on natural keys (idempotency key, order number, email)
- Background workers that assume a job runs only once (no idempotency or dedup logic)
- `INSERT ... IF NOT EXISTS` pattern via SELECT-then-INSERT instead of `INSERT ... ON CONFLICT`

### Example fixes

- Atomic decrements at the database: `UPDATE items SET stock = stock - 1 WHERE id = ? AND stock > 0` — check the row count to confirm
- Transactions or row locks (`SELECT ... FOR UPDATE`) around check-then-act sequences
- Idempotency keys on webhook handlers; dedupe by event ID with a `UNIQUE` constraint
- `UNIQUE` constraints on order numbers, idempotency keys, email addresses
- Cron locking via Redis, advisory locks, or a leader-election mechanism

---

## 8. Payment & Transaction Integrity

### Why this category matters

Money mistakes hurt. Customers notice. Banks and regulators care. Refunds are expensive in time and trust. Idempotency, reconciliation, and clear state machines are the entire game in this category. If you remember nothing else: every payment-side action gets an idempotency key, every state transition gets a reconciliation job, every webhook is the source of truth (not the synchronous response).

### Questions to ask

- Is the order created before or after payment confirmation? What happens if payment fails after order creation? What happens if order creation fails after payment succeeds?
- If the payment provider's webhook is delayed or lost, how does the system eventually mark the order paid? (Retry? Polling? Manual reconciliation?)
- If the user retries a charge (refresh, double-click, network retry), can they end up paying twice?
- Are refunds tracked back to the original transaction with a clear audit trail?
- Are payment status changes logged with timestamps and source (gateway webhook, manual update, user action, reconciliation job)?
- Do you handle the case where the gateway says "succeeded" but you cannot write to your DB?
- Are you using the gateway's idempotency keys correctly (Stripe `Idempotency-Key`, etc.)?
- If a webhook arrives for an order you do not recognize, what happens?
- Is there a "pending → succeeded" terminal state, or can payments stay pending forever?

### Red flags in code

- Charge calls without idempotency keys
- Order status updated only in the synchronous payment response handler (no webhook reconciliation)
- No audit log of payment status changes
- Webhook endpoint that does not verify the gateway signature
- Same payment status updated in multiple places without a single source of truth
- Currency stored as `float` instead of integer minor units (cents)

### Example fixes

- Always pass an idempotency key to the gateway, derived deterministically from your order ID + attempt number
- Webhook handler is the source of truth for payment status; sync handler optimistically updates UI but webhook reconciles
- Audit log table for every payment status change with `from_status`, `to_status`, `source`, `actor`, `timestamp`
- Verify webhook signatures; reject unsigned or invalid ones with 401
- Store money as integer minor units (cents, paise) — never float
- Background job that flags orders pending > N hours and queries the gateway directly to reconcile

---

## 9. Device & Environment

### Why this category matters

You build on a fast laptop with good wifi. Your users are on three-year-old phones in coffee shops. They get phone calls mid-flow. They rotate their device. They have weird browser extensions. They use the app one-handed on a bus. The system has to survive its actual environment, not the dev environment.

### Questions to ask

- What happens to in-progress flows when the app is backgrounded on mobile? Killed by the OS for memory pressure?
- Does form state survive a screen rotation, an app switch, a deep link from another app?
- Does the OTP/auth flow survive a screen timeout or a switch to the SMS app to copy the code?
- How does the layout behave at 320px width (small phone)? At 4K? At 200% browser zoom?
- What happens if a phone call interrupts the user mid-payment?
- Are autofill suggestions respected, or do you fight them with `autocomplete="off"`?
- Does the back button on Android do what users expect (go back one logical step, not exit the app)?
- For PWAs and SPAs: does deep-linking work? Refresh on a sub-route?
- If the user has reduced-motion or reduced-transparency enabled, does the UI respect it?

### Red flags in code

- Form state held in component-local state only (lost on rotation/remount on mobile web)
- No `viewport` meta tag, or one with `user-scalable=no`
- Hard-coded dimensions in pixels for critical UI elements
- `autocomplete="off"` on common fields like email, phone, address (annoys users, fights password managers)
- Modal overlays that do not trap focus or release it correctly on close
- Heavy CPU work on the main thread that locks the UI on lower-end devices

### Example fixes

- Persist in-progress form state to localStorage or backend so a kill-and-restore preserves it
- Use `autocomplete` attributes correctly: `email`, `tel`, `cc-number`, `street-address`, `postal-code`, etc.
- Test on a real low-end Android device, not just Chrome's device emulator
- Custom back-button handling in SPA routes so back goes back one logical step
- Respect `prefers-reduced-motion`; gate animations behind it

---

## 10. Accessibility & Inclusion

### Why this category matters

Accessibility gets treated as optional and is not. It is a legal requirement in many jurisdictions, a usability requirement everywhere, and the bar for "is your UI clear" — if a screen reader cannot make sense of it, neither can a confused, sleep-deprived, or distracted user. This category is also where assumptions about language, alphabet, and input devices break.

### Questions to ask

- Does the screen reader announce success/failure of every async action (live regions)?
- Can the user complete every flow with a keyboard only?
- Do focus traps in modals release on close?
- Do colors meet WCAG contrast ratios? Does the UI work in high-contrast mode?
- Is there any meaningful text rendered as an image without alt text?
- Does the layout work in RTL languages (Arabic, Hebrew)?
- Do form errors get announced to assistive tech, or only shown visually?
- If the user has set a larger system font size, does the UI still work?
- Is color the only signal for state (red error text without an icon or aria-label)?
- For video content: are captions provided?

### Red flags in code

- `<div onClick>` instead of `<button>`
- Modals without `role="dialog"`, `aria-modal="true"`, and focus management
- Form errors shown only via styling, not via `aria-describedby` or `aria-invalid`
- Hard-coded `font-size: 14px` on body text (overrides user preference)
- Custom dropdowns, toggles, and tabs built without ARIA roles
- `tabindex` greater than 0 anywhere

### Example fixes

- Use semantic HTML (`button`, `nav`, `main`, `form`, `dialog`)
- Add `aria-live="polite"` regions for async status updates
- Test the entire flow with `tab`, `enter`, `escape`, and arrow keys only
- Run axe or Lighthouse accessibility audit and fix violations
- Test with VoiceOver (Mac/iOS) or TalkBack (Android) on a real device

---

## 11. Data Validation & Input Handling

### Why this category matters

Users will enter the unexpected. Sometimes by mistake (extra space in email). Sometimes deliberately (testing limits). Sometimes maliciously (injection). Validate at the boundary, normalize on entry, store the canonical form. Trust nothing the user typed.

### Questions to ask

- Is every input validated on the server, not just the client?
- Are numeric inputs bounded (no negative quantities, no `1e9` line items, no decimal places where integers are expected)?
- Are string inputs length-limited?
- Are special characters (`<`, `>`, `"`, `'`, `\`, null bytes, emoji, RTL marks) handled in every place they could be rendered, queried, or stored?
- Are dates parsed with timezone awareness?
- Are email/phone/address formats validated AND normalized to a canonical form?
- What happens if a required field arrives as `null`, `undefined`, empty string, or whitespace?
- Is there a unique constraint between the validation layer and the storage layer (e.g. case-insensitive email uniqueness)?
- Are uploaded files validated by content (magic bytes), not just by extension?

### Red flags in code

- Client-only validation (HTML5 `required`, `pattern`) without a server-side mirror
- String concatenation into SQL or shell
- Quantity inputs with no min/max
- Free-text date inputs without explicit format validation
- Email comparisons that do not lowercase
- `parseInt(input)` without checking for NaN
- File uploads accepted by extension without checking actual content type

### Example fixes

- Validation library (zod, joi, yup, pydantic) with a single schema shared between client and server where possible
- Parameterized queries everywhere; never string-concatenate into SQL
- Normalize emails to lowercase before storage and comparison
- Reject obviously bogus inputs at the API edge with a clear, specific error message
- Validate file uploads via magic bytes, not just `.jpg` in the filename

---

## 12. Error Handling & Recovery

### Why this category matters

Failure is inevitable. The difference between a robust app and a fragile one is what happens after. Does the user get a clear path to recover? Does the system retry intelligently? Does the failure leave the system in a sane state, or does it leave debris that costs an engineer an afternoon to clean up?

### Questions to ask

- For every failure mode in the audit so far, is there a recovery path?
- After a failed payment, can the user retry without restarting the cart?
- After a network failure, does the app preserve in-progress state?
- Are errors logged with enough context (user ID, request ID, timestamp, stack, relevant payload) to diagnose later?
- Are user-facing error messages distinct from internal errors? No stack traces shown to users.
- Is there a circuit breaker or backoff for repeated failures, or does the app retry forever?
- After a partial success (e.g. order created but confirmation email failed), is the failure tracked and recoverable?
- Is there a global error boundary in the UI so a single component crash does not blank the entire screen?
- Are unhandled promise rejections caught and reported?

### Red flags in code

- `catch (e) { console.log(e) }` — error swallowed
- Same generic error message ("Something went wrong") for every failure
- Retries with no backoff (and no max attempts)
- Critical operations without try/catch at the boundary
- No request IDs in logs or in error responses
- No global `unhandledrejection` or `error` handler in the frontend
- No error reporting service (Sentry, Bugsnag, Datadog) wired up

### Example fixes

- Generate a request ID per request; include it in logs and in error responses so support can find the trace
- Specific error types with specific user messages (and a fallback for unexpected ones)
- Exponential backoff with jitter on retries; cap attempts
- Background job for partial-success cleanup (e.g. retry the email; alert if still failing after N tries)
- Sentry or equivalent for error tracking with user/request context tagged
- Global error boundary in React/Vue/etc.

---

## After the audit

Once findings are recorded, write them into `interruptions-audit.md` using the template in `references/audit-template.md`. Then surface to the user for review. Do not start fixes until they explicitly confirm with the phrase from Phase 3 of `SKILL.md`.

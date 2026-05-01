# interruptions

<p align="center">
  <img src="interruptions-banner.png" alt="interruptions banner" width="100%">
</p>

I built `interruptions` because every product I shipped had edge cases I missed until users hit them. Edge cases are the gap between what you tested and what actually happens. They are also the difference between a product that feels polished and one that feels brittle.

This is a Claude skill. You install it once, and from then on, whenever you ask Claude to audit a flow, find what could go wrong, or stress test a feature, it walks your code through 12 categories of failure modes, writes a thorough audit, waits for you to read it, and then helps you fix the issues in order of severity.

I called it `interruptions` because that is what edge cases really are. The user gets a phone call mid-payment. The network drops mid-upload. A second tab racing the first. A double-tap. A back button. The happy path is what you designed. The interruptions are what your users actually live through.

## What it does

When you ask Claude to find edge cases, the skill will:

1. Greet you and ask what you want audited (a specific flow, a file, or the whole repo)
2. Walk through 12 categories against your code, methodically
3. Write a comprehensive audit to `interruptions-audit.md`
4. Wait for you to read it and confirm explicitly
5. Implement the fixes in severity order, one at a time, surfacing each change

The review gate in step 4 is intentional. Edge case fixes touch payment paths, auth, and state machines. If you skip the audit, you will not understand the changes. So I built it to wait.

## The 12 categories

These are the categories the skill walks every time. Each one has its own questions, red flags, and example fixes inside `references/checklist.md`.

1. **UX & User Psychology**: how users behave irrationally, accidentally, or unpredictably
2. **Security**: replay attacks, IDOR, frontend tampering, weak auth
3. **Design & Visual State**: how the UI communicates (or fails to communicate) what just happened
4. **System Flow & Logic Design**: breakdowns in workflow and state transitions
5. **State & Data Integrity**: when frontend, backend, and cache disagree
6. **Network & Infrastructure**: timeouts, lost responses, partial requests
7. **Concurrency & Race Conditions**: two things happening at once, in the wrong order
8. **Payment & Transaction Integrity**: where money mistakes happen, and how to stop them
9. **Device & Environment**: real-world device, OS, and browser behavior
10. **Accessibility & Inclusion**: screen readers, keyboard nav, RTL, zoom, contrast
11. **Data Validation & Input Handling**: malformed, extreme, or hostile input
12. **Error Handling & Recovery**: what happens after something fails, and whether you can recover

## How to install

```bash
npx skills add omrajguru05/interruptions
```

That is it. The skill installs into your Claude Code, Claude Desktop, or Claude.ai environment, and activates whenever you ask Claude to audit edge cases, find failure modes, or stress test a flow.

## How to use it

Once installed, just talk to Claude. The skill triggers on phrases like:

- "Audit my checkout flow for edge cases"
- "What could break in my payment screen?"
- "Stress test my entire app for failure modes"
- "I just shipped this feature, what edge cases did I miss?"
- "I built an e-commerce app, find what is going to break"

The skill takes it from there. It greets you, asks what you want audited, walks the code, writes the audit, and waits for you to confirm before fixing anything.

## A worked example

Imagine you tell Claude: "I built an e-commerce app, audit my payment screen."

The skill walks every category. For your payment screen specifically, it might find:

- **UX**: the "Pay" button is not disabled after click, so a double-tap fires two charges
- **Security**: the request includes the total in the payload, so a tampered request could undercharge
- **Network**: there is no idempotency key, so a retry on lost-response causes a double charge
- **Concurrency**: the coupon "check then mark used" is not atomic, so the same coupon can be used twice in parallel tabs
- **Payment integrity**: the order is created before payment confirmation, so failed payments leave orphan orders
- **Error handling**: the failure case shows a generic toast with no retry path, so users hit support

Each finding gets a severity, a `file:line` reference, and a concrete fix. You read the audit. You confirm. The skill fixes them in severity order. You ship.

## Why I built it

I kept watching small failure modes slip past me into production. Not because I did not care, but because there are too many ways things can fail for one person to hold in their head at the same time.

The 12 categories in this skill are my checklist. I built it so I could hand the same checklist to Claude and have it walk every flow methodically, the way I would on a really focused day. If it saves you a midnight rollback or a refund spiral, it has paid for itself.

## What this skill does not do

- It does not replace human judgment on architecture or product decisions
- It does not test the live app (no browser automation, no synthetic users)
- It does not modify infra, secrets, or third-party dashboards (it surfaces those as manual follow-ups)
- It does not start fixing anything until you confirm in plain words

If you want runtime testing or contract testing, that is a different tool. This skill is about reading code and flows with a structured paranoia, surfacing what you missed, and helping you fix it.

## How I keep it from breaking your code

Phase 4 (the fix phase) runs under heavy safety constraints. I built these in because the worst version of this skill would be one that confidently rewrites your codebase and breaks things you cannot easily roll back.

Before any fixes, the skill runs pre-flight checks:

- Confirms you are not on `main`, `master`, or any auto-deploy branch (and recommends a fix branch if you are)
- Looks at your working tree for uncommitted changes it does not recognize, and pauses if it sees any
- Inventories your test, typecheck, and lint commands so it can run them between fixes
- Glances at your CI config to flag any branch that auto-deploys to production

Then for each fix:

- Smallest possible diff. No reformatting. No renaming. No drive-by cleanup.
- Reads the entire function and its callers before editing
- Shows the planned diff and waits for your go on high-risk fixes (payments, auth, DB writes, public APIs)
- Runs your tests/typecheck/lint after each change. If a fix breaks an existing test, it stops. It does not modify the test to make it pass.
- One file at a time for non-trivial changes. No fan-out edits.
- Never deletes code unless the finding explicitly requires it
- Never installs or updates dependencies, never touches migrations or env files, never runs destructive commands

If something looks weird (the file does not exist, the function signature has changed, the working tree has unexpected state), the skill stops and asks. It would rather pause than guess.

The full safety rules live in `SKILL.md` under the Phase 4 section if you want to read them in full.

## Repo structure

```
interruptions/
├── SKILL.md                       the main skill file
├── README.md                      this file
└── references/
    ├── checklist.md               the 12-category audit checklist
    └── audit-template.md          the format the audit file follows
```

The `SKILL.md` at the root is what the `npx skills add` command installs. The `references/` files are loaded by Claude only when needed during the audit.

## Updates and devnotes

I write about what I am learning while building this:

- [omrajguru.com/devnotes/interruptions](https://omrajguru.com/devnotes/interruptions)
- [github.com/omrajguru05/interruptions](https://github.com/omrajguru05/interruptions)

## Find me

- Portfolio: [omrajguru.com](https://omrajguru.com)
- Projects: [projects.omrajguru.com](https://projects.omrajguru.com)
- X: [@omrajguru_](https://x.com/omrajguru_)
- Dev.to: [dev.to/omrajguru05](https://dev.to/omrajguru05)
- GitHub: [github.com/omrajguru05](https://github.com/omrajguru05)

## Disclaimer

This framework is designed to help identify and reason about potential edge cases, but it does not guarantee completeness or error-free outcomes. While we have implemented safety guardrails to reduce risk, unforeseen issues, system instability, or unintended side effects may still occur—especially in complex or real-world environments.

Testing edge cases can itself introduce breakages. It is essential to run all experiments in controlled environments such as staging or sandbox systems before considering production deployment. Comprehensive validation, regression testing, monitoring, and reliable rollback mechanisms should always be in place.

Edge case handling is not a one-time activity. Systems must be iteratively improved as new scenarios emerge from real user behavior, scale, and evolving integrations.

By using this framework, you acknowledge that all implementation, testing, and deployment decisions are your responsibility. We assume no liability for any system failures, data loss, financial impact, or other unintended consequences resulting from its use.

## License

MIT. Use it, fork it, ship it. Attribution is appreciated and the skill itself preserves it in the audit footer by default.

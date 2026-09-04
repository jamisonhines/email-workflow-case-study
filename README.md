# AI Email Workflow

An AI workflow that runs my tour company's inbox. Shipped, and running daily since May. Three times a day it triages every unread message, drafts personalized replies and quotes from the company knowledge base, and leaves them for a human to approve and send. It cannot send on its own, and it proves that before every run.

The code is private because it's wired into the live business inbox. This repo is the build log, with diagrams instead of screenshots since the interesting part is the wiring.

## The flow

A router classifies each unread message into one of 24 categories (new inquiry, client replied, follow-up due, partner platform, noise, security alert, and so on) and dispatches the right handler. Handlers are chains of small single-purpose skills, so each piece can be tested, reused, and improved on its own.

Across 865 logged messages between 15 May and 3 September 2026, **16 of those 24 categories have actually fired**. The other eight are defined and waiting. I mention both numbers because a category count on its own says what a system can imagine, not what it has met.

```mermaid
flowchart TD
  classDef default fill:#000,color:#fff,stroke:#fff,stroke-width:2px
  A[Unread inbox] --> B{Router, classify into 24 categories}
  B --> C[New inquiry]
  B --> D[Client replied / follow-up due]
  B --> E[Noise, archive and label]
  B --> F[Handled direct, alerts, invites, unknown]
  C --> G[Fetch full thread context]
  D --> G
  G --> H[Look up answers in company wiki]
  H --> I[Draft reply or quote in brand voice]
  I --> J{QA gate, check draft against whole thread}
  J -->|issues found| I
  J -->|pass| K[Gmail draft + label, human approves and sends]
  E --> L[One audit-log row per message]
  F --> L
  K --> L
```

## It cannot send, and it re-proves that before every run

Nothing in this system sends an email. That is not a promise living in a prompt, because a promise in a prompt is worth very little once a model is having a bad day. There are three independent controls. The skills are instructed never to send. A tool hook denies any shell command shaped like a send. And a wrapper on the PATH catches the same call made from inside a Python subprocess, where the hook cannot see it.

Then the part I actually like. Before every scheduled run, the poller fires a three-leg probe at its own controls. It attempts a send through the shell and expects to be denied. It attempts an allowed read and expects it to succeed. It attempts a send from Python and expects the wrapper to catch it. If any leg comes back wrong, it refuses to touch the inbox at all.

A control you have never tested is a control you are guessing about.

## The QA gate

Every draft is checked against the entire thread history before it's saved. Missed client questions, contradictions with earlier promises, price or date mismatches, unfilled template placeholders, banned phrases, and whether the sign-off matches the product being sold. If the gate finds issues, it sends the draft back to the drafter with a structured correction list, up to two rounds, then escalates to a human with the full diagnostic chain instead of shipping something wrong. On escalation no draft is saved at all, so a half-right answer never sits in the drafts folder waiting to be sent by accident.

## The learning loop

The part I'm proudest of. Every night, a separate job compares the drafts the system wrote over the past thirty days against what actually got sent from the Sent folder. Every human edit is a lesson, and the diffs get written to a learning log that feeds improvements back into the drafting and checking skills. The system improves the more it's used, using the most honest signal there is, which is what a human changed before hitting send.

```mermaid
flowchart LR
  classDef default fill:#000,color:#fff,stroke:#fff,stroke-width:2px
  A[Drafts the system wrote] --> C{Nightly compare}
  B[What actually got sent] --> C
  C --> D[Learning log, sent as-is, edited, discarded]
  D --> E[Improvements to drafting and QA skills]
  E --> A
```

## Follow-up cadence

Inquiries that go quiet get a polite nudge at day 3, A/B tested across two templates on a 50/50 split, and a courteous close-out at day 10. Long-horizon prospects can opt out of that clock into a rolling monthly check-in instead. All drafted, never auto-sent.

## Reliability, the hard-won part

This system taught me more about production AI than anything else I've built.

- **Exit 0 does not mean handled.** On 11 August the router finished a run, exited clean, narrated the draft it had written, and had created nothing. The message was already marked as seen. If I had not gone looking, that client's email would have been silently retired forever. The fix was to stop trusting a process's own account of itself. The poller now requires a nonced outcome line, and when a draft is claimed it goes and looks for that draft in Gmail before marking anything seen.
- **Do not let a label lie about what happened.** Follow-up threads were being tagged as sent and closed the moment a *draft* was created, so a thread could read as nudged while the client had received nothing at all. Those labels now say "review", and only flip to closed once a real sent message is verified on the thread.
- **A dedup guard** marks every processed message so a crashed run can't double-draft. Three consecutive failed polls leave a message unread for a human rather than retrying forever.
- **One logging skill** is the only thing allowed to write the audit log. One canonical row per message, listing every skill that touched it. When five different skills all write their own logs, you have no log.
- **Small skills chain, big skills rot.** The first version was three monolithic scripts with duplicated logic. The refactor into single-purpose skills is what made the system debuggable and improvable.
- **Headless is different.** A skill that works in a chat session fails quietly on a schedule. Big documents get parsed by scripts, not read into context, and every run leaves an auditable trail.

## Plumbing

Headless Claude Code on a schedule, fired by launchd at 06:00, 12:00 and 18:00, and gated so a run only spins up a session when there is genuinely new mail. Gmail through a CLI, the company wiki as the knowledge base for products, pricing, policies and templates, and proposal PDFs attached from Drive. Runs unattended on a Mac.

## Build log

- **2026-09.** The nightly loop caught a bug in itself. One sent message could be attributed to several drafts on the same thread, inflating the manual-rewrite count it reports. Diagnosed and written up by the job that had the bug, which is the whole point of building it that way.
- **2026-08.** Verified outcomes and a retry cap, after the exit-0 incident above. Sandboxed execution with a scoped tool allowlist, the send-blocking hook, and the pre-run probe, replacing what had been a prompt-level promise running with full inherited permissions. Follow-up labels corrected so they describe what was sent rather than what was drafted.
- **2026-08.** A second surface shipped, a Gmail add-on that lets a draft be requested from inside the inbox rather than waiting for the next scheduled run.
- **2026-07.** Learning loop hardened. The nightly postmortem now parses the full run history by script instead of reading it into context, which fixed silent truncation on large logs.
- **2026-05.** The big refactor. Monolithic handlers split into chained single-purpose skills, with one canonical logging skill and per-message audit rows.
- **Earlier.** First version shipped. Router, inquiry drafting from wiki templates, follow-up cadence, noise handling.

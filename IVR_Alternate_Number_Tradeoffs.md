# IVR Alternate Number Routing — Tradeoffs Register

Companion to `IVR_Alternate_Number_PRD.md` v3.0. **Not part of the PRD.** This is the PM's own record of every decision put as a tradeoff during the interview, what was rejected, and why.

Owner: Ashish Raj · Signed off 4 Aug 2026

| # | Decision point | Chosen | Rejected options | Why (PM's stated reason) | Date |
|---|---|---|---|---|---|
| 1 | Alternate-number coverage looked like 1.37% of the active base — does the customer-side feature ship at all? | Ship it, as its own PRD alongside the escalation chain | Ship the CSP-side chain alone; fold both into one spec | Capture already collects alternate numbers, and the two PRDs solve the same connect-rate problem for different users | 3 Aug 2026 |
| 2 | What is an alternate number? | A number the customer indicated they are reachable on, reaching the list from Capture or Ops (see #18) | Treat it as any second number on file, whoever it belongs to | Correctness is owned by whoever enters it, system or ops (R9) | 3 Aug 2026 |
| 3 | Is the customer told the number will be used for CSP routing? | No | Tell the customer at capture, or ask consent before first use | PM's call. Recorded here so it is a visible decision rather than an omission | 3 Aug 2026 |
| 4 | How many numbers get dialled on one call? | 3 maximum, as a configurable parameter (C-01) | A fixed depth; unlimited alternates | Mirrors the CSP side, where three was forced by the role model. Here it is a choice, so it is configurable | 3 Aug 2026 |
| 5 | How many alternates are stored? | As many as exist; freshness decides which get dialled | Cap storage at two or three | Storing costs nothing and survives one number going stale. C-01 caps the dialling, not the holding | 4 Aug 2026 |
| 6 | Dial order among alternates | Newest-seen first | Oldest first; order stored; random | The most recent number is the one most likely to still be the customer's | 4 Aug 2026 |
| 7 | Do alternates go stale? | Yes — usable only within C-02 of last being seen, default 6 months | No expiry; expire on repeated dial failure | A number good a year ago may belong to someone else today. Expiry, not failure, is the removal mechanism | 4 Aug 2026 |
| 8 | What removes a number? | Expiry (C-02) or an explicit ops action | Automatic removal after repeated no-answer | Same as the expiry decision — a number that does not answer is not necessarily wrong | 4 Aug 2026 |
| 9 | Who manages the list? | Internal call-centre and ops users, through a portal | Customer self-service in the Customer App; no manual management at all | Ops needs to correct a number when they learn it is wrong. Customer App is explicitly a later version | 4 Aug 2026 |
| 10 | Does the internal portal need a design file? | No | Block build on a portal design | It is an internal portal: the spec states what it must show and do, and the implementer chooses how it looks | 4 Aug 2026 |
| 11 | Are numbers shown in full in the portal? | Masked by default, revealed one at a time on an explicit action, each reveal recorded (R11) | Full numbers on screen by default | Tighter on PII. A number can also be replaced without revealing the old one, so correcting never forces a number on screen | 4 Aug 2026 |
| 12 | What does the PIN requirement mean for this spec? | Entry-path parity — the same destination list is built whether the IVR resolved the call from the app cache or from an entered PIN (G6, R10) | Change caller identification so alternate numbers identify an inbound caller; capture a number when a PIN verifies it (both drafted in v0.2, then removed) | These PRDs enrich **whom to dial**, leaving the IVR 2.0 structure intact. Identification and the PIN mechanism are not theirs to change | 4 Aug 2026 |
| 13 | Is coverage a success metric? | No. It survives as MQ-9, explaining connect rate rather than gating it | Coverage as the primary metric with connect-rate targets conditional on it (drafted in v0.1) | Not needed as a target. A customer with no alternate gets one dial, so coverage still explains a rate that does or does not move | 4 Aug 2026 |
| 14 | Is install in scope, and does it get a metric? | Yes to both — M3, baseline 75.8%, no-regression floor | Install out of scope, used only as the benchmark | Same as the escalation-chain PRD: any extra gain is worth having, and the fallback costs nothing extra to apply there | 4 Aug 2026 |
| 15 | Install is in scope but its 75.8% is also the target for Service and Pickup | Freeze 75.8% as install's **pre-launch** rate | Let the target float with install's live rate | A floating target would let install's own improvement quietly raise the bar for the other two families | 4 Aug 2026 |
| 16 | Keep a counter-metric for the share connecting on the registered number? | No | Keep it with a 90% floor (drafted as M4) | Not needed. The concern it addressed — an alternate being answered by someone other than the customer — is still recorded by MQ-2 and MQ-10 | 4 Aug 2026 |
| 17 | Two lifecycles in §3b, against the template's one | Two — the fallback run and the alternate number | Merge them into one table | A number lives for months and a run for seconds. Merging would hide the number's lifecycle, which is where most of this spec's value sits. Recorded as an Override | 4 Aug 2026 |
| 18 | Where do alternate numbers come from? | Exactly two sources: **Capture** (an agent tags a number against the customer's registered number) and **Ops** (an authorised user adds one by hand). Ops can edit or delete any number whatever its source | Capture as the only source; treat Capture-sourced numbers as read-only to protect them | Both sources need defining now. A number a Capture agent got wrong is exactly the one ops must be able to fix, so protecting it by origin would defeat the point | 4 Aug 2026 |
| 19 | Must the customer be calling from a number for it to be captured? | No — an agent may tag a number the customer simply told them | Capture only the number the customer is calling from | The agent judges how they learned it; the registered number is what ties it to one customer. Narrowing it to the calling number would miss most of what agents actually collect | 4 Aug 2026 |
| 20 | Other sources — Customer App and beyond | Out of scope for this version; a later source feeds the same list without changing anything else | Build the Customer App path now | Explicitly deferred by the PM | 4 Aug 2026 |

## The data correction behind this spec

The first draft of §1 said alternate-number coverage was **1.37% of the active base** and that capture was dead — nine numbers in all of 2026, none since 1 April. The PM challenged the figure. It was wrong, and the correction is worth keeping:

- **1.37% measured the wrong source.** It came from `CUSTOMERVANILLA_AUDIT` / `alternate_number_captured`, a structured event that genuinely did die on 1 April 2026.
- **The real source is `PROD_DB.DYNAMODB.TICKETS.EXTRA_DATA:alternate_number`** — a named key inside a VARIANT blob on Capture tickets. **18,985 customers hold a valid alternate that differs from their registered number**, 18,604 of them a number held for that customer alone. It is still filling at roughly 360–500 customers a month.
- **Why the original sweep missed it**: it scanned `ACCOUNT_USAGE.COLUMNS` for *column* names, and a key inside a JSON blob can never appear there. When hunting for a field in this warehouse, always also flatten the VARIANT blobs.
- **A false alarm, recorded so it is not repeated**: some alternate values appear on hundreds of tickets, which looked like shared lines being dialled as customer numbers. It was ticket-level repetition — the same customer raising many tickets — not customers sharing a number. G4 remains as a cheap safety net, not because sharing is common.

Free text is a secondary source: `TICKETVANILLA` `DESCRIPTION`/`COMMENT`/`TITLE` carried a different-from-registered mobile on 23,900 tickets over 12 months, 4,972 distinct customers. It needs extraction and review before anything there could be dialled, so it is a possible one-off backfill rather than a source of record.

## Superseded on 5 Aug 2026 — v2.0 scope reduction

The rows above record decisions as they were taken. Several were later reversed by the PM. The originals are kept rather than edited, so the reasoning trail stays honest; these are the reversals.

| Original row | What changed in v2.0 | Why |
|---|---|---|
| #2, #18, #19 — Capture as a source; an agent tagging a number | **Removed entirely.** This PRD no longer mentions Capture | The alternate number store is deployed and specified separately. How a number arrives is that PRD's, not this one's |
| #5, #6 — store many alternates, freshness decides dial order | **One alternate per customer.** C-01 = 2 dials (registered + one alternate); ordering is degenerate | Only one alternate is used per customer today. C-01's range notes it rises when more are held |
| #7 — expiry window of 6 months | **Restated as 180 days** | "Six months" has no single meaning, so a boundary test could not be written to the day. Same intent, computable |
| #9, #10, #11 — the internal portal, no design file, masked with reveal | **Removed entirely.** No screen section in this PRD | The portal is specified in the Customer Alternate Number Store PRD, which owns its design |
| #14, #15 — install in scope with its own metric M3 | **Install out of scope.** M3 removed; two metrics remain | Restore and Pickup only. Install stays as the frozen 75.8% benchmark, a reference point rather than a family this spec applies to |
| #17 — two lifecycles in §3b | **One lifecycle.** Only the fallback run remains | The alternate number's lifecycle moved out with the store |

**New in v2.0, from a QA review of the acceptance criteria:**

- **"Does not answer" is now defined** in the glossary as the Exotel outcome for unanswered, busy or unreachable, with ring duration named as Exotel applet configuration. Eleven ACs turned on that phrase with nothing defining it, so no tester could execute them without leaving the document.
- **Regression ACs no longer say "as it does today".** Six ACs compared against an undocumented baseline; each now asserts the observable it was standing in for — for example "exactly one dial is placed, to the registered number".
- **Race ACs no longer say "at the same instant".** No tester can produce a simultaneous event; each is now framed as a testable state, such as "before the dial to position 2 is placed".
- **Bundled ACs were split.** One assertion each, so a failure names itself.

## Further reduction, 5 Aug 2026 — v2.1

| Original row | What changed | Why |
|---|---|---|
| #7, #8 — alternates go stale after C-02; expiry is the removal mechanism | **C-02 removed entirely.** No age test, no G5 guardrail, no expiry ACs | Whether a number is still fit to dial is the store's judgement, not this spec's. AC-REG-6 now asserts the opposite of the old behaviour: an alternate last used two years ago **is** dialled, because this spec applies no age test |
| Dedupe against the registered number (R3/G3 in v2.0) | **Removed entirely.** No dedupe rule, no guardrail, no AC | The store's own spec refuses a number equal to the registered number — its R2b, tested by its AC-HUB-4, and its glossary defines an alternate as "a phone number, other than their registered number". Testing it here duplicated a guarantee the store already gives |

## v2.2 — the dial list is resolved at call time

| Decision point | Chosen | Rejected | Why | Date |
|---|---|---|---|---|
| When is the dial list built? | **At the moment the call is placed**, from what the store holds then | Resolve at ticket creation; cache a list per ticket; carry a list from an earlier call on the same ticket | An alternate added after the ticket was raised must be used on the next call. Resolving at ticket creation would silently ignore every number captured after a ticket opened — which is most of them | 5 Aug 2026 |

v2.1 already froze the list "when the run begins", which meant the same thing, but only implicitly — and implicit is how a developer ends up resolving it at ticket creation. R6 now states it as an obligation with two must-nots, T1 says it in canon, and two ACs test it: AC-DIAL-5 (a number added after the ticket is dialled) and AC-DIAL-6 (two calls on the same active ticket each take the latest stored number). Qualified to the **same active ticket** in v2.3, so the obligation is bounded to a ticket's open life rather than reading as an open-ended promise across a customer's history.

## v2.4 — three resolution paths, not two

| Decision point | Chosen | Rejected | Why | Date |
|---|---|---|---|---|
| How many ways does IVR 2.0 resolve a call, and must the fallback work on all of them? | **Three**, and yes — the same dial list on every one | Two (app cache and PIN), as v2.0–v2.3 had it | The third path is a dialer call with **no PIN**, resolved because the caller has exactly one active ticket. It was missing from the spec entirely. It is also the one most likely to be missed in build, because it looks like the PIN path but skips the PIN | 5 Aug 2026 |

Row #12 above recorded the PIN requirement as "entry-path parity" across two paths. That was incomplete rather than wrong: the parity obligation stands, but it covers three paths. `AC-PIN-*` is renamed `AC-RES-*`, since the group was never really about PINs, and the three paths now have a canonical definition in §8 so no section restates them.

**This PRD is an increment on IVR 2.0, not a change to it.** All three paths, caller identification, PIN handling, what happens when a dialer caller has more than one active ticket, and dead-end handling stay IVR 2.0's. R5 must-not(b) forbids this spec changing any of them.

## v2.5 — "frozen dial list" removed from the acceptance criteria

The phrase was doing one real job: making AC-RACE-3 true, that a change in the store mid-call does not disturb the call in progress. It was then used in sixteen places, most of them acceptance criteria about which numbers get dialled — where it described internal state instead of observable behaviour. A tester can watch which numbers were dialled and in what order; they cannot watch a list being frozen.

The obligation is kept and stated as behaviour: **"the list is read once, at the start of the run"** (§3a precedence 3, T1, and the Fallback run definition). The ACs now assert the dials themselves — for example AC-DIAL-1 became *"09811100022 is dialled first and, if it does not answer, 09811100044 second — and no other number is dialled."*

## v2.6 — per-dial logging, and what it costs to get wrong

| Decision point | Chosen | Rejected | Why | Date |
|---|---|---|---|---|
| Must each dial's outcome be distinguishable, or is answered / not-answered enough? | **Five distinct outcomes**: answered, not answered, busy, could not be reached, ended by the CSP while ringing | Two outcomes; four outcomes without the abandoned case | Busy and unreachable collapsing into "not answered" removes the very distinction needed to diagnose a connect rate that will not move | 5 Aug 2026 |
| What is recorded for the number that was ringing when the CSP hung up? | A row with **ended by the CSP while ringing** | Leave it undefined; record it as not answered | Neither answered nor unanswered — nobody waited to find out. Undefined, abandoned runs would have been counted as ring-outs and the no-answer rate would have inflated | 5 Aug 2026 |
| How is the link between the log and the reported metrics enforced? | MQ-9 is named as the source record every other measure is computed from, and AC-WF-9 requires the reported connect rate to be reproducible from the rows alone | Trust that the dashboard and the log agree | A log and a dashboard that can drift apart do drift apart, and nobody notices until someone reconciles by hand — which is how the `BOOKING_LOGS` duplication went unspotted for a month in the P41 work | 5 Aug 2026 |

## Finalise, v3.0 — 5 Aug 2026

Full lint run at finalise. One error found and fixed: **R6 must-not(a)** — "do not carry a dial list from an earlier call" — had no acceptance criterion citing it. AC-DIAL-6 was testing it in substance but cited the wrong sub-obligation. Both R6 must-nots now have coverage.

Everything else clean: sections in template order, all IDs contiguous with no dangling references, every rule obligation and transition AC-covered, all four guardrails anchored to both an AC and an MQ, every MQ with a named consumer, zero markers, zero TBD, all 50 acceptance criteria Settled, HTML parity word-for-word.

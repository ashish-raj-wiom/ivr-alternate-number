# IVR Alternate Number Routing — reaching the customer on a second number

| | | | |
|---|---|---|---|
| **Owner** — Ashish Raj (PM) | **Reviewer** — Rahul (Eng Lead) | **Status** — Signed off | **Sign-off** — Signed off · 5 Aug 2026 |
| **Version** — v2.0 · 5 Aug 2026 | | | |

---

## 1. Objective & Definition of Success

**Objective.** A CSP who calls a customer about their restore or pickup ticket reaches that customer — if the registered number does not answer, the call moves on by itself to the alternate number held for that customer, and the customer's chances of being reached increase.

**Boundary.** This spec governs **CSP-initiated** IVR calls on **Service (restore) and Pickup** tickets, at every CSP where IVR 2.0 is live. It governs one thing: **which numbers the IVR dials, and in what order, once a call has been resolved to a customer.**

- **Holding the numbers is not this spec's.** The alternate number store, how a number gets into it, and the portal that manages it are specified and deployed elsewhere — the Customer Alternate Number Store PRD. This spec **reads** that store and never writes to it (R6, AC-REG-5).
- **Install tickets are out of scope.** Install is the source of this spec's benchmark, not a family it applies to (AC-REG-2).
- **Customer-initiated calls** are the escalation-chain PRD's, not this one's (AC-REG-1).
- **The registered number** is never changed, replaced or reordered. It is always dialled first (G2).
- **How the IVR decides whom to dial** — the app cache, the PIN, caller identification and dead-end handling — is governed by the existing IVR product spec. This spec begins only once that resolution has named a customer, and enriches only what it produced (R7, AC-PIN-2).
- **Ring durations** are Exotel applet configuration, not parameters of this spec (see Overrides). "Does not answer" is defined in §8.
- **What the CSP is told** — nothing about which number answered, or that an alternate exists (AC-REG-3).

### Guardrails

| ID | Guardrail | One line | Anchors |
|---|---|---|---|
| G1 | **Invisible fallback** | The call moves to the next number on its own; the CSP never presses a key, hears an announcement, or has the call dropped and re-established. | R1 · AC-GRD-1 · MQ-2 |
| G2 | **The registered number is always first** | Every run starts at the customer's registered number, so an alternate never displaces it. | R2 · AC-GRD-2 · MQ-7 |
| G3 | **One dial per number** | No number is dialled twice in one run, including an alternate that duplicates the registered number. | R3 · AC-GRD-3 · MQ-4 |
| G4 | **Only this customer's numbers** | A run only ever dials numbers the store holds against the customer who owns the ticket — never another customer's, never the CSP's own. | R6 · AC-GRD-4 · MQ-6 |
| G5 | **Stale numbers are never dialled** | A number last seen longer ago than C-02 is never dialled, even when it is the only alternate available. | R5 · AC-GRD-5 · MQ-8 |
| G6 | **The same list either way** | The dial list is the same whether the IVR resolved the call from the app cache or from an entered PIN. | R7 · AC-GRD-6 · MQ-11 |

### Success metrics

Ticket-level connect rate is defined in §8. A ticket where the CSP disconnected before any number answered **counts as not connected** (T5, MQ-5). Both figures are CSP-initiated, Phase-1 cohort, 2 Jul – 3 Aug 2026.

| ID | Metric | Baseline | Target | Source |
|---|---|---|---|---|
| M1 | Ticket-level connect rate — **Service** (restore) | 66.7% | 75.8% | MQ-1 |
| M2 | Ticket-level connect rate — **Pickup** | 50.7% | 75.8% | MQ-1 |

**Where 75.8% comes from.** It is what **install** tickets achieved on CSP-initiated calls in the window above, before this spec shipped — the closest thing to a ceiling this direction is known to reach. Install is **not in scope**; it is the reference point only, and the figure is frozen at its pre-launch value so it cannot drift. It is also direction-matched on purpose: install's all-directions figure (79.6%) mixes in customer-initiated calls, and its customer-initiated figure (69.0%) belongs to the escalation-chain PRD, so neither can be used here.

**How far these can move depends on how many customers hold an alternate number.** A ticket whose customer holds none gets exactly one dial, as it does now (T6), so the fallback can only act on the covered share of the base. MQ-9 reports that share, which is what explains a connect rate that does or does not move.

**Reaching a different person is not the same as reaching the customer.** An alternate number is answered by whoever holds that handset, and this spec cannot prove it was the customer. MQ-2 and MQ-10 record which position answered on every call, so a rate that rises while the registered number answers less often is visible rather than hidden.

**Invariant:** G2 — runs where the registered number was not dialled first = 0, zero tolerance. Monitored via MQ-7.

**Invariant:** G4 — numbers dialled that the store does not hold against the ticket's customer = 0, zero tolerance. Monitored via MQ-6.

**Invariant:** G5 — dials placed to a number last seen longer ago than C-02 = 0, zero tolerance. Monitored via MQ-8.

---

## 2. User Stories & Rules

| ID | Story | MUST | MUST NOT |
|---|---|---|---|
| R1 | As a CSP trying to reach a customer about their job, I want the call to try their other number by itself, not fail on the first ring-out. | **(a)** Move to the next number in the dial list by itself when the current one does not answer (§8). **(b)** Bridge the call to the first number that answers. | **(a)** Ask the CSP to press a key, hold, redial or dial another number. **(b)** Play the CSP any announcement about the fallback. **(c)** Disconnect and re-establish the call in order to advance. |
| R2 | As a customer, I want my main number tried first — it is the one I gave you. | Dial the registered number as position 1 of every run (G2). | Skip, reorder or replace the registered number, ever. |
| R3 | As a customer, I do not want the same number rung twice in one call. | Dial each distinct number once; where the alternate held for a customer equals their registered number, dial it once and shorten the run (G3). | Dial the same number more than once in one run. |
| R4 | As a CSP, I want a bounded number of attempts, not an open-ended sequence. | Dial at most C-01 numbers in one run, the registered number included, in the order the store returned them after the registered number. | Dial beyond C-01, or place two dials of one run at the same time. |
| R5 | As a customer, I do not want a number I last used long ago rung today — it may not be mine any more. | Exclude from the dial list any number the store reports as last seen longer ago than C-02 (G5). | Dial an excluded number, even when it is the only alternate the customer holds. |
| R6 | As a customer, I want only my own numbers dialled about my job, and I do not want a call changing my records. | **(a)** Restrict every dial to numbers the store holds against the customer who owns the ticket (G4). **(b)** Read the store only. | **(a)** Dial another customer's number, the CSP's own number, or any number the store does not hold for this customer. **(b)** Add, change or remove anything in the store as a result of placing a call. |
| R7 | As a CSP who reached the customer by dialling back and entering a PIN, I want the same fallback I get from the app. | Build the same dial list however the IVR resolved the destination — from the app's cached mapping or from an entered PIN (G6). | **(a)** Apply the fallback on one entry path and not the other. **(b)** Change how the app cache or the PIN resolves a destination, or what a PIN means. |

---

## 3. System Behaviour

### 3a. System flow chart

```mermaid
flowchart TD
    A["CSP-initiated call resolved to a customer and ticket"] --> B{"Ticket is Service or Pickup?"}
    B -- "No" --> C["Existing IVR routing — outside this spec (§1 Boundary)"]
    B -- "Yes" --> D{"Fallback enabled? (C-03)"}
    D -- "No" --> C
    D -- "Yes" --> E{"Store readable in time?"}
    E -- "No" --> F["T7 — registered number only"]
    E -- "Yes" --> G{"A live, distinct alternate held, within C-01?"}
    G -- "No" --> H["T6 — registered number only"]
    G -- "Yes" --> I["T1 — dial position 1, alternate held in the frozen list"]
    F --> J{"Answers?"}
    H --> J
    I --> J
    J -- "Yes" --> K["T2 — connected"]
    J -- "No" --> L{"CSP still on the line?"}
    L -- "No" --> M["T5 — abandoned"]
    L -- "Yes" --> N{"A further position left in the frozen list?"}
    N -- "No" --> O["T4 — exhausted"]
    N -- "Yes" --> P["T3 — dial the next position"]
    P --> J
```

**Precedence — the CSP hanging up beats advancing.** If the CSP disconnects while a number is ringing and before the next dial has been placed, the run ends and no further number is dialled (T5, AC-RACE-1).

**Precedence — one bridged leg only.** If a ringing number answers while the run is advancing past it, the call bridges to exactly one number (T2, AC-RACE-2).

**Precedence — the list is frozen when the run begins.** A list resolved at the start of a run is used to the end of that run; a change in the store mid-run does not alter it, and applies from the next run (AC-RACE-3).

### 3b. State transition table — canon

Lifecycle of a **fallback run**, created when a CSP-initiated call resolves to a customer holding a Service or Pickup ticket. Each outbound call creates its own run; no state carries between runs. The alternate number's own lifecycle belongs to the Customer Alternate Number Store PRD.

| ID | From | Action / Trigger | Rule / Check | To | Side-effects |
|---|---|---|---|---|---|
| T1 | — | CSP-initiated call resolved to a customer and an in-scope ticket | Fallback enabled (C-03); the store holds at least one live, distinct alternate within C-01 | Ringing position 1 | Registered number dialled first (R2, G2). The dial list is resolved and frozen now: registered number, then the alternate, deduplicated (R3) and excluding anything last seen longer ago than C-02 (R5). List length recorded (MQ-4). |
| T2 | Ringing position N | The ringing number answers | — | Connected | Call bridged (R1b). Which position answered, and whether it was the registered number or the alternate, recorded (MQ-2, MQ-10). Terminal. |
| T3 | Ringing position N | The ringing number does not answer (§8) | A further position remains in the frozen list | Ringing position N+1 | Next number dialled (R4), and not before the current dial has finished. No CSP action, no announcement, no reconnection (R1, G1). |
| T4 | Ringing position N | The ringing number does not answer (§8) | No further position remains | Exhausted | Existing unconnected-call handling applies. Ticket counted as not connected (M1, M2, MQ-5). Terminal. |
| T5 | Ringing position N | The CSP disconnects | — | Abandoned | Run ends; no further number is dialled. Ticket counted as not connected (M1, M2, MQ-5). Terminal. |
| T6 | — | CSP-initiated call resolved to a customer and an in-scope ticket | The store holds no live, distinct alternate for that customer | Ringing position 1 | One dial only, to the registered number. No answer routes to T4, not T3. **The common case wherever coverage is low** (MQ-9). |
| T7 | — | CSP-initiated call resolved to a customer and an in-scope ticket | The store cannot be read in time to be used | Ringing position 1 | **Failure envelope:** a call is never failed for want of a list — it degrades to the registered number alone. No answer routes to T4. No alternate is guessed, or reused from an earlier run. |

---

## 4. Screen Requirements

**Not applicable — this feature has no screen.** The fallback happens entirely on the phone network, and G1 requires it be invisible: the CSP is never shown or told anything about it.

The one screen that touches alternate numbers is the internal portal that manages them, and that belongs to the Customer Alternate Number Store PRD along with the rest of the store (§1 Boundary).

---

## 5. Configurability

| ID | Parameter | Default | Range | Who changes it |
|---|---|---|---|---|
| C-01 | Maximum numbers dialled in one run, registered number included (T1, R4) | 2 | Raise when a customer may hold more than one alternate number | Product |
| C-02 | How long since a number was last seen before it is excluded from the dial list (R5) | 180 days | 30–730 days | Product |
| C-03 | Fallback enabled — kill switch (T1) | On, wherever IVR 2.0 is live | On / Off | Product + Eng |

**C-02 is stated in days on purpose.** "Six months" has no single meaning — 180 days is unambiguous, and a tester can compute the boundary to the day.

**Where numbers are not parameters.** How long each number rings, and the total ringing a CSP hears, are Exotel applet configuration (see Overrides). Which CSPs are covered follows the IVR service's own rollout.

---

## 6. Measurement

| ID | The system must be able to answer… | Feeds |
|---|---|---|
| MQ-1 | Of in-scope tickets with at least one CSP-initiated call, what share had at least one call where a person answered — split by ticket family. Abandoned runs count as not connected. | M1 · M2 |
| MQ-2 | For each run, which position answered — registered, alternate, or none. | G1 · M1 · M2 |
| MQ-3 | For each position, the share of dials to that position that were answered. | G2 · how much of any gain came from the alternate |
| MQ-4 | How long each frozen dial list was, and how many candidates were dropped by dedupe, by C-01, or by C-02. | G3 · G5 · R3 · R4 · R5 |
| MQ-5 | How many runs ended Connected, Exhausted or Abandoned. | M1, M2 definition · T4 · T5 |
| MQ-6 | Whether any number dialled in a run was not held by the store against that ticket's customer. | G4 invariant (R6) |
| MQ-7 | Whether any run dialled something other than the registered number first. | G2 invariant (R2) |
| MQ-8 | Whether any dial was placed to a number last seen longer ago than C-02. | G5 invariant (R5) |
| MQ-9 | How many customers in the in-scope base hold a live alternate number, and how that is moving. | M1 · M2 — the covered share is what explains a rate that does or does not move |
| MQ-10 | For one call, the outcome of **every** number dialled — one row each: which number, at which position, whether it was the registered number or the alternate, and whether it answered, did not answer, was busy or could not be reached. Every row tied to that call by a single identifier, and to the ticket and customer. A number never reached because the run ended first has no row. | G2 · G3 · G4 · G5 · underpins MQ-2 · MQ-3 · MQ-4 |
| MQ-11 | For each call, which IVR entry path resolved the destination — app cache or entered PIN. | G6 · R7 |

---

## 7. Acceptance Criteria

**Example data** — customer Meena, registered number 09811100022, holding one alternate 09811100044 last seen 20 Jul 2026. CSP `CSP-4412`, technician Ravi, Service ticket `TKT-88231`. Calls on 12 Aug 2026, with C-01 = 2 and C-02 = 180 days.

**"Does not answer"** throughout means the outcome defined in §8 — Exotel reporting the dialled number as unanswered, busy or unreachable. The ring duration that produces it is Exotel applet configuration, not set by this spec.

### DIAL — Building the dial list (T1, T6, T7)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-DIAL-1 | **Given** the store holds 09811100044 for Meena, last seen 20 Jul 2026, **When** Ravi calls her on `TKT-88231` at 15:20, **Then** the frozen dial list is exactly 09811100022 at position 1 and 09811100044 at position 2. | R2 · R4 · T1 · G2 | Settled |
| AC-DIAL-2 | **Given** the store holds no alternate for Meena, **When** Ravi calls her, **Then** the frozen dial list is exactly 09811100022 at position 1, and no position 2 exists. | T6 · G2 | Settled |
| AC-DIAL-3 | **Given** the store holds 09811100022 for Meena as her alternate — the same value as her registered number, **When** Ravi calls her, **Then** the frozen dial list holds exactly one position and exactly one dial is placed. | R3 · T1 · G3 | Settled |
| AC-DIAL-4 | **Given** the store holds 09811100055 for Meena, last seen 200 days before 12 Aug 2026, **When** Ravi calls her, **Then** 09811100055 is absent from the frozen dial list and the list holds only 09811100022. | R5 · T1 · G5 | Settled |
| AC-DIAL-5 | **Given** the store returns an error, or does not answer in time to be used, when asked for Meena's numbers, **When** Ravi's call is bridged, **Then** the frozen dial list is exactly 09811100022, and the call is bridged rather than failed. | T7 | Settled |
| AC-DIAL-6 | **Given** a Pickup ticket for Meena and 09811100044 held for her, **When** a CSP calls her on that ticket, **Then** a two-position dial list is built — Pickup is in scope. | T1 · M2 · §1 Boundary | Settled |

### FB — Falling back and connecting (T2, T3)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-FB-1 | **Given** the two-position list from AC-DIAL-1, **When** 09811100022 answers, **Then** Ravi is bridged to 09811100022. | R1b · T2 | Settled |
| AC-FB-2 | **Given** the same list, **When** 09811100022 answers, **Then** no dial is placed to 09811100044. | R1b · T2 | Settled |
| AC-FB-3 | **Given** the same list, **When** 09811100022 does not answer, **Then** 09811100044 is dialled. | R1a · T3 | Settled |
| AC-FB-4 | **Given** the same list, **When** the run advances from position 1 to position 2, **Then** no announcement was played to Ravi and no keypress was requested of him. | R1 must-not(a) · R1 must-not(b) · G1 | Settled |
| AC-FB-5 | **Given** the same list, **When** the run advances from position 1 to position 2, **Then** Ravi's call was never disconnected and re-established — one continuous leg from bridge to end. | R1 must-not(c) · G1 | Settled |
| AC-FB-6 | **Given** the same list, **When** 09811100022 returns busy, **Then** 09811100044 is dialled — busy is treated as not answering. | T3 · §8 | Settled |
| AC-FB-7 | **Given** the same list, **When** 09811100022 is ringing, **Then** no dial to 09811100044 has yet been placed. | R4 must-not · T3 | Settled |

### END — Run endings (T4, T5)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-END-1 | **Given** the two-position list with 09811100044 ringing at position 2, **When** it does not answer, **Then** the run's end state is Exhausted. | T4 | Settled |
| AC-END-2 | **Given** the run in AC-END-1, **When** the ticket is counted for MQ-1, **Then** `TKT-88231` counts as not connected. | T4 · M1 · MQ-5 | Settled |
| AC-END-3 | **Given** 09811100022 is ringing at position 1, **When** Ravi disconnects before any dial to position 2 is placed, **Then** the run's end state is Abandoned and no dial was placed to 09811100044. | T5 | Settled |
| AC-END-4 | **Given** the run in AC-END-3, **When** the ticket is counted for MQ-1, **Then** `TKT-88231` counts as not connected. | T5 · M1 · MQ-5 | Settled |

### PIN — Both IVR entry paths (R7)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-PIN-1 | **Given** the store holds 09811100044 for Meena, **When** Ravi dials the masked number from his call log and enters the PIN for `TKT-88231`, **Then** the frozen dial list is 09811100022 at position 1 and 09811100044 at position 2 — the same list as AC-DIAL-1. | R7 · G6 · T1 | Settled |
| AC-PIN-2 | **Given** Ravi's PIN-entered call, **When** the IVR resolves the destination, **Then** the customer whose numbers are used is the customer on the ticket the PIN resolved to. | R7 must-not(b) · §1 Boundary | Settled |
| AC-PIN-3 | **Given** Ravi's PIN-entered call, **When** 09811100022 does not answer, **Then** 09811100044 is dialled — the fallback is not limited to app-originated calls. | R7 must-not(a) · T3 · G6 | Settled |

### WF — Workflows and per-dial tracking (MQ-10)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-WF-1 | **Given** the two-position list from AC-DIAL-1, **When** 09811100022 does not answer and 09811100044 answers, **Then** Ravi is bridged to 09811100044 on the same inbound call, and `TKT-88231` counts as connected. | R1 · T1 · T3 · T2 | Settled |
| AC-WF-2 | **Given** the same list, **When** neither number answers, **Then** exactly two dials were placed, 09811100022 before 09811100044, and the run's end state is Exhausted. | T1 · T3 · T4 | Settled |
| AC-WF-3 | **Given** the run in AC-WF-1, **When** its record is examined, **Then** it holds exactly two rows — 09811100022 at position 1, not answered; 09811100044 at position 2, answered — both under one call identifier and tied to Meena and `TKT-88231`. | MQ-10 | Settled |
| AC-WF-4 | **Given** each row of the run in AC-WF-1, **When** it is read, **Then** it states whether that number was Meena's registered number or her alternate. | MQ-10 · MQ-2 · MQ-3 | Settled |
| AC-WF-5 | **Given** Ravi called Meena at 15:20 and again at 15:45, **When** both runs are examined, **Then** each has its own call identifier and every row belongs to exactly one of them. | MQ-10 · T1 | Settled |
| AC-WF-6 | **Given** the single-position run in AC-DIAL-2, **When** its record is examined, **Then** it holds exactly one row — a run with no fallback is recorded the same way as one with it. | MQ-10 · T6 | Settled |

### FAIL — When the store cannot be read (T7)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-FAIL-1 | **Given** the store is unreachable, **When** Ravi calls Meena, **Then** a dial is placed to 09811100022 — a call is never failed because the store could not be read. | T7 | Settled |
| AC-FAIL-2 | **Given** the store is unreachable and 09811100022 does not answer, **When** the dial completes, **Then** the run's end state is Exhausted and exactly one dial was placed — no alternate is guessed, or reused from an earlier run. | T7 · T4 | Settled |

### REG — Regression and boundary (§1)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-REG-1 | **Given** `TKT-88231`, **When** Meena calls Ravi rather than Ravi calling her, **Then** no dial list is built by this spec and routing follows the escalation-chain PRD. | §1 Boundary | Settled |
| AC-REG-2 | **Given** an **install** ticket for Meena and 09811100044 held for her, **When** a CSP calls her on that ticket, **Then** exactly one dial is placed, to 09811100022, and 09811100044 is not dialled — install is out of scope. | §1 Boundary | Settled |
| AC-REG-3 | **Given** Ravi is bridged to 09811100044, **When** the call connects, **Then** Ravi is given no indication of which number answered and no indication that an alternate exists. | §1 Boundary · G1 | Settled |
| AC-REG-4 | **Given** the run in AC-WF-1, **When** the call statuses Exotel returned are examined, **Then** each of the two dials has its own status record, and MQ-10's per-run record exists in addition to them. | MQ-10 · §1 Boundary | Settled |
| AC-REG-5 | **Given** the store holds 09811100044 for Meena, last seen 20 Jul 2026, **When** any run for Meena completes in any end state, **Then** the store still holds 09811100044 with last-seen 20 Jul 2026 — placing a call changes nothing in the store. | R6b · §1 Boundary | Settled |

### RACE — Simultaneity (§3a precedence rules)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-RACE-1 | **Given** 09811100022 is ringing at position 1 and the run has determined that position 2 exists, **When** Ravi disconnects before the dial to position 2 is placed, **Then** no dial is placed to 09811100044 and the run's end state is Abandoned. | T5 · precedence 1 | Settled |
| AC-RACE-2 | **Given** the run is advancing from position 1 to position 2, **When** 09811100022 answers after the advance has begun but before the dial to position 2 has connected, **Then** exactly one bridged leg exists for the call. | T2 · precedence 2 | Settled |
| AC-RACE-3 | **Given** a run is in flight with 09811100044 in its frozen list, **When** that number is removed from the store before the run reaches position 2, **Then** the run still dials 09811100044, and the next run for Meena does not. | precedence 3 · T1 | Settled |

### DUP — Duplicate triggers

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-DUP-1 | **Given** Ravi's call at 15:20 ended Exhausted, **When** he calls Meena again at 15:45, **Then** the new run's position 1 is 09811100022. | R2 · T1 · G2 | Settled |
| AC-DUP-2 | **Given** two CSPs call Meena on two different tickets while neither run has ended, **When** both runs proceed, **Then** each dials its own frozen list, and neither run's outcome changes the other's list or end state. | T1 | Settled |

### BV — Boundary values (C-01, C-02)

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-BV-1 | **Given** an alternate last seen exactly 180 days (C-02) before the call, **When** the list is built for a CSP's call, **Then** it is included at position 2 — exclusion begins after the window, not at it. | C-02 · R5 | Settled |
| AC-BV-2 | **Given** an alternate last seen 181 days before the call, **When** the list is built for a CSP's call, **Then** it is excluded and the list holds one position. | C-02 · R5 · G5 | Settled |
| AC-BV-3 | **Given** C-01 is 1, **When** a CSP calls a customer holding a live alternate, **Then** exactly one dial is placed, to the registered number. | C-01 · R4 · G2 | Settled |
| AC-BV-4 | **Given** C-01 is 2 and a customer holds one live alternate, **When** a CSP calls and the registered number does not answer, **Then** exactly two dials are placed. | C-01 · R4 | Settled |

### CFG — Configurability

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-CFG-1 | **Given** the fallback is switched off (C-03), **When** Ravi calls Meena who holds a live alternate, **Then** exactly one dial is placed, to 09811100022. | C-03 | Settled |
| AC-CFG-2 | **Given** C-02 is changed from 180 days to 90, **When** the next list is built for a customer whose alternate was last seen 120 days ago, **Then** that alternate is excluded — and nothing in the store was edited to achieve it. | C-02 · R5 · R6b | Settled |

### GRD — Guardrails

| AC | Given / When / Then | Verifies | Status |
|---|---|---|---|
| AC-GRD-1 | **Given** every run in a week that reached position 2, **When** each call's record is examined, **Then** none played an announcement to the CSP, none requested a keypress, and none shows the call disconnected and re-established. | G1 · R1 | Settled |
| AC-GRD-2 | **Given** every run in a week, **When** the first dial of each is checked, **Then** every one was placed to that customer's registered number. | G2 · R2 · MQ-7 | Settled |
| AC-GRD-3 | **Given** every run in a week, **When** each frozen dial list is checked, **Then** no number appears twice in any one list. | G3 · R3 · MQ-4 | Settled |
| AC-GRD-4 | **Given** every run in a week, **When** every number dialled is checked against what the store holds for that run's customer, **Then** all of them are held for that customer, and none is another customer's or the CSP's own. | G4 · R6a · MQ-6 | Settled |
| AC-GRD-5 | **Given** a customer whose only alternate was last seen 200 days ago, **When** a CSP calls them, **Then** that number is not dialled, even though it was the only fallback available. | G5 · R5 · MQ-8 | Settled |
| AC-GRD-6 | **Given** the same customer and ticket, **When** one call is placed from the app and another by dialling back and entering the PIN, **Then** both frozen dial lists hold the same numbers in the same positions. | G6 · R7 · MQ-11 | Settled |

---

## 8. Glossary

| Term | Meaning | Owner (domain) |
|---|---|---|
| Alternate number | An additional phone number the store holds against a customer, existing so a call can be connected to them when their registered number does not answer. Defined in full — including how it gets there and who may change it — by the **Customer Alternate Number Store PRD**. This spec only reads it. | Customer |
| Registered number | The customer's primary number on their Wiom account. Always position 1, never reordered or replaced by this spec (G2, R2). | Customer |
| Fallback run | **Canonical definition:** the ordered list of distinct numbers dialled for one outbound CSP call, and the act of moving down it when a number does not answer. Created per call, with its list frozen when the run begins. Carries: its own identifier, the ticket and customer, the frozen list, the position that answered if any, and the end state. | — |
| Does not answer | **Canonical definition:** the outcome for one dial when Exotel reports the number as unanswered, busy, or unreachable. The ring duration that produces it is Exotel applet configuration (see Overrides), not set by this spec. Every rule and acceptance criterion that turns on a number "not answering" means exactly this. | Telephony |
| Position | A number's place in a frozen dial list. Position 1 is always the registered number; position 2 is the alternate. | — |
| Exhausted · Abandoned | A run's end state. **Exhausted**: every position was dialled and none answered. **Abandoned**: the CSP disconnected before any position answered. Both count as not connected. | — |
| Ticket-level connect rate | **Canonical definition:** of tickets that had at least one call in the chosen direction and window, the share where at least one of those calls was answered by a person. A ticket counts once however many calls it had. Any answered call counts, however short. Not to be confused with call-level connect rate, which is per call; comparisons must never mix the two grains. | — |
| Service · Pickup ticket | The two families in scope. **Service** is the restore family — a live customer whose connection needs restoring, called "Service" in the ops dashboard. **Pickup** is netbox recovery, collecting equipment from a customer. Install is out of scope. | — |

---

## 9. Notes for System Capabilities

Whether these are one system or several is the implementer's design.

| Capability | Needed by |
|---|---|
| Read a customer's alternate numbers, with their last-seen timestamps, from the store — and never write to it. | R6 · T1 · AC-REG-5 |
| Build a dial list from what the store returned: registered number first, then alternates, deduplicated, excluding anything last seen longer ago than C-02, capped at C-01 — and freeze it for one run. | T1 · R2 · R3 · R4 · R5 · G2 · G3 · G5 |
| Dial an ordered list for one outbound call, advancing on no answer, busy or unreachable, without the CSP acting, without an announcement, and without dropping the call. | T1 · T3 · R1 · G1 |
| End a run the moment the CSP disconnects, dialling nothing further; bridge to exactly one number even when an answer and an advance coincide. | T5 · T2 · precedence 1 · precedence 2 |
| Fall back to the registered number alone when the store cannot be read in time, without failing the call and without reusing a list from an earlier run. | T7 |
| Build the same list whichever IVR entry path resolved the destination, and record which path it was. | R7 · G6 · MQ-11 |
| Give each run an identifier and record one row per number dialled — the number, its position, whether it was the registered number or the alternate, and its outcome — tied to the customer and ticket. | MQ-10 |
| Report connect rate by ticket family, per-position answer rates, run end states, and the covered share of the base. | MQ-1 · MQ-2 · MQ-3 · MQ-5 · MQ-9 |
| Turn the fallback off, and change C-01 and C-02, without a release. | C-01 · C-02 · C-03 |

---

## Overrides

| Rule | What was done instead | Rationale | Approved by |
|---|---|---|---|
| §5 — every number that could change gets a C-id | Per-number ring duration and the total ringing a CSP hears are not C-ids | Exotel App Bazaar applet configuration, not a parameter of this service. Consistent with the escalation-chain PRD and with dropping `ES_CSPIVR_CALL_BRIDGE_MAX_RING_SECONDS` in June 2026. "Does not answer" is defined in §8 so the acceptance criteria stay testable without it. | Ashish Raj (PM), 5 Aug 2026 |
| Header — name consulted parties by domain | No consulted parties named | Carried over from the sibling IVR PRDs at the PM's instruction. | Ashish Raj (PM), 5 Aug 2026 |

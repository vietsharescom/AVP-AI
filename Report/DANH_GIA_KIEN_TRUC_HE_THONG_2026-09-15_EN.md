# SYSTEM ARCHITECTURE ASSESSMENT — AVP_AI (WEBAPP)
## Live data scan + comparison against ERP/Odoo/ISO standards — 2026-09-15

*Audience: Andy Phan (Owner's representative) / AVP management.*
*Unlike [`BAO_CAO_TONG_HOP_AVP.md`](BAO_CAO_TONG_HOP_AVP.md) (an assessment
of the old MANUAL process — 2 Excel files — against ERP), this report
assesses the **AVP_AI webapp actually running today** (`D:\AVP_AI\webapp`):
its source code + live data in Google Sheets (RawMaterial 1,939 rows,
FinishGood 194 rows, the `/api/ledger` ledger 184 rows, PackingList).*

> **Principle**: every number below comes from real API calls
> (`localhost:3000`) and from reading the source code directly on
> 2026-09-15, not from guessing. Anywhere the origin could not be
> determined, it is marked "UNCLEAR — needs your confirmation," never
> filled in with a made-up reason.

---

## TABLE OF CONTENTS

1. Live data scan — is there still "cell drift" (scan reading into the wrong column)?
2. Is the data/extraction architecture as rigorous as ERP/ISO/Odoo standards yet?
3. Input→Process→Output architecture diagram + is the data store centralized yet?
4. Priority recommendations
5. Lessons from AVP_AI for the new project (proper ERP `stock.move` model)
6. **[NEW]** Verifying floor-level pain points with data (planner/manager interview, 2026-09-15)

---

## 1. LIVE DATA SCAN — IS THERE STILL "CELL DRIFT"?

### 1.1 Warnings the system already raises on its own — overview (`/api/ledger`, 184 rows)

| Warning type | Row count |
|---|---|
| Missing "Quantity per box" in PartControl | 61 |
| Computed QUANTITY exceeds raw-material Pieces by >10% | 27 |
| Suspected duplicate scan (matching Traveler#+LOT#, not yet Shipped) | 6 |
| QUANTITY off by >10% from the hand-scanned figure | 5 |
| Part# written with a different dash format (auto-recognized) | 1 |
| **Total rows with at least 1 warning** | **91/184 (49%)** |

→ Nearly half of the currently open (not-yet-shipped) rows need a human to
look them over — most of that (61+27=88) is because **PartControl is still
missing data** (Quantity/box), not because the AI misread anything. This is
an already-known burden (previous report, section 7.1: 36 Part#s missing
Quantity/box).

### 1.2 An additional independent scan — NEW findings today

I wrote a separate scan (not relying on the existing warnings) specifically
to look for the "AI read the wrong column/row" pattern, like the traveler
717544 case just handled. Result across **all of RawMaterial (1,939 rows) +
FinishGood (194 rows)**:

**a) Already fixed today — traveler 717544** (details earlier in this
conversation): the Part# cell printed across 2 lines on paper
(`11502717-TH-\nTC-T`) had part of it misread into the Pot#/LOT# columns by
the AI, on both scans (Original + Copy). Fixed to
`Part#=11502717-TH-TC-T`, `Pot#=614`, `LOT#` left blank.

**b) NEW, NOT YET FIXED — 7 FinishGood rows where Pot# and LOT# are
IDENTICAL** (travelers `717397`, `718437`, `717772`, `717781`, `718646`,
`718016`, `718014`) — e.g. traveler 718437: `Pot#="371"` and `LOT#="371"`.
Cross-checking `createdAt`: **all 7 rows were created at the exact same
instant, `2026-09-14T21:06:05.642Z`**, and the `oc` column
(Original/Copy — a required field of the normal Split Form scan flow) is
**blank in every one of them** — unlike real Split Form scans (which always
carry "O" or "C"). 3 of these 7 travelers (717772, 717781, 718014) also
have a SEPARATE, parallel row, created at `19:15:22` the same day, with a
genuinely valid LOT# (e.g. `6-251-10-B`) and a properly populated `oc` —
meaning **the "good" row already existed, and the bad row is an EXTRA one
written on top of it afterward**.

→ **I have not yet identified which action/route produced the 21:06:05
rows** (it doesn't match the normal AI-vision Split Form scan flow — the
blank `oc` is the clear tell). Do you recall what was done on the web app
around 21:06 on 2026-09-14 (an Excel import, "+ Add one row by hand," or
testing some feature)? I need to know exactly what happened to patch the
right place and avoid a repeat.

**c) Field labels stuck to the value** — traveler `718643`:
`Pot#="Pot#313"`, `LOT#="Lot6-253-04-E"` — the AI read the label text
itself ("Pot#", "Lot") along with the value instead of taking only the
number/code that follows. A different failure mode from (b) — this is a
missing "strip the label prefix" step after the AI returns its result.

**d) Pot# being used to record a PACKAGING UNIT instead of a Pot# code** —
10 RawMaterial rows: `"36 CTNS"`, `"27TOTES"`, `"49TOTES"`, `"GAYLORD"`...
`GAYLORD` is already known to be normal (confirmed in a previous session:
it's a container-type name, not a distinct Pot# — traveler 712654). But
`"CTNS"/"TOTES"` is a sign that **the same Pot# cell on paper is carrying
two different roles** (a numeric identifier vs. a packaging-unit
description) depending on the goods type — not really an AI misreading, but
**an ambiguous Pot# schema** (the "packaging type" field was never split
out from the "Pot# code" field).

### 1.3 Overall assessment of the data scan

Strength: `finish-good/confirm/route.ts` (where newly scanned data is
saved) already has **7 overlapping checks** running at save time —
duplicate lot, duplicate Boxes+Qty with a different LOT (suspected
misread), one Traveler with an unusual LOT#/Part#/Pot# change (based on the
1:1 rule measured across 1,598 historical travelers), a Pot# still open
under a different traveler, a Pot# that doesn't match the original
RawMaterial record, a Part# missing from PartControl, a Traveler far
outside the numeric range of the same scan batch. That is a fairly high
level of control compared with a typical OCR-scanning app.

Weakness: those checks **only run when a NEW row is being saved**, compared
against whatever data is currently OPEN at that moment — there is no
periodic "data health scan" that re-runs against ALL previously saved data.
In other words: any error that slips through (or that happened on a row
saved BEFORE this check code existed, or arrived through a different entry
path like case (b) above) will sit there silently until someone happens to
notice it — exactly how the 717544 case and case (b) today were found (you
pointed at a photo — the system did not flag it itself).

---

## 2. IS THE ARCHITECTURE AS RIGOROUS AS ERP/ISO/ODOO STANDARDS YET?

### 2.1 Storage model

Google Sheets acts as the "database": 5 flat tabs (RawMaterial, Warehouse,
FinishGood, PartControl, PackingList), no foreign keys, no transactions, no
indexes — every JOIN (RawMaterial↔FinishGood↔PartControl↔PackingList) is
assembled by the code AT READ TIME.

**A measured, real risk**: I counted in the source code — **14 API route
files** each independently read and join RawMaterial/FinishGood on their
own (`packing-list/generate`, `packing-list/line`, `reconciliation`,
`trace`, `warehouse/*`, `finish-good/confirm`...), and **3 UI components**
(`GlobalSearchBar`, `PoStatusTable`, `TravelerSection`) call
`/api/raw-material/list` + `/api/finish-good/list` directly and join them
AGAIN in the browser. `/api/ledger` (the consolidated ledger, built in a
previous session exactly to solve this) is currently **used by only 1
screen** (Packing List) — the right direction, but not yet fully adopted.
A real consequence: the script that imported the historical PS 30098/30101
data (previous session) forgot exactly one step — "mark as Shipped" —
because it did not go through any shared processing path — this produced
22 travelers with a broken status (already
[fixed today](fix-ps30098-30101-shipped.mjs)). This is a direct consequence
of not having a SINGLE business-logic layer ("there is exactly one valid
way to mark something as shipped") — the more routes that each do their own
thing, the easier it is for a new route to forget one step.

**No concurrent-write control**: `updateRowsWhere`/`appendRows` do not
check a version or lock a row before writing — if 2 people (or 2 browser
tabs) edit the same row at nearly the same time, whoever writes last wins,
with no warning that "the data has changed since you opened it." At the
current scale (a handful of users) the risk is low, but it will grow as
more staff work in the system at once during a busy shift.

**No audit trail on EDITS** (only `createdAt` at CREATION time): editing
`shipped`/`ps`/Concession/Priority overwrites the cell directly, with no
log of "who changed what, when" beyond Google Sheets' own version history
(which the app itself doesn't manage). Concession/Priority are 2 good
exceptions (they do record `...By/...Reason/...At`) — this pattern should
be applied to other sensitive changes too.

**No formal migration/schema versioning**: new columns are added via
one-off scripts run by hand each time
(`add-finishgood-scrap-columns.mjs`, `...-concession-columns.mjs`,
`...-priority-columns.mjs`...) — this works fine in practice, but there is
no single place listing "which version is this Sheet on, which migrations
have run" — it relies entirely on the operator's memory.

### 2.2 Compared with Odoo (a real ERP already running, benchmarked in the previous report)

| Criterion | Odoo (standard) | AVP_AI today |
|---|---|---|
| Inventory ledger | `stock.move` — IMMUTABLE (append-only); a correction is a new reversing entry, old lines are never edited/deleted | Edits the `shipped`/`ps` field DIRECTLY IN PLACE — no trace of the original entry is kept |
| Document status | A formal state machine (draft→confirmed→done→cancel), each step with its own permissions | Just 1 boolean flag `shipped` (TRUE/FALSE) — no distinction between "in progress"/"approved"/"cancelled" |
| Data integrity | The database (PostgreSQL) enforces types, foreign keys, unique constraints AT THE STORAGE LAYER | Every constraint (Traveler↔Pot# 1:1, valid Part#...) is checked by CODE at write time — the storage layer (the Sheet) enforces nothing |
| Multiple concurrent users | Transactions + row locks | None — last write wins, no warning |

This is **not a criticism** — at the current scale (a few thousand
rows/month, a handful of users), choosing Google Sheets over standing up a
full Postgres/Odoo stack is a reasonable cost/time tradeoff, and Andy has
already made this call deliberately (LATEST_SESSION: "the proper ERP
`stock.move` ledger model will be built as an entirely SEPARATE new
project" — the right call, not patching the live app with a large
architectural change). The table above exists to answer "is it as rigorous
as ERP yet" — the short answer: **not yet, and you already know it and have
already made the right call (split off a new project) — no need to do this
on the live system right now.**

### 2.3 The extraction layer (Gemini vision OCR) — rigorous, and how prone is it to conflicts?

**Strength**: each type of paper document has its own dedicated JSON schema
(`TRAVELER_FORM_SCHEMA`, `SCANNING_FORM_SCHEMA`, the Split Form schema) —
clearly separated by document type, with a `confidence` field (the AI
flags "not sure about this reading") for hard-to-read handwriting, and it
**cross-checks against existing data** (the original RawMaterial,
PartControl, other travelers' history) instead of trusting a single read
absolutely — this is a strength, well above typical OCR that "saves
whatever it reads."

**A weakness likely to keep causing the same class of bug**: the schema
describes fields to the AI in NATURAL LANGUAGE (`"Pot # column"`,
`"Part # column"`), with no sample-image examples, and **there is no
validation/regex step to clean up what the AI returns before saving it** —
so when the AI returns the whole string `"Pot#313"` instead of just `"313"`
(section 1.2c), the system saves it as-is, with no "middle layer" to catch
it. Likewise, a cell wrapping onto 2 lines in the source file (the 717544
case) has no "unusual format" warning mechanism before saving — it was only
caught because a BUSINESS-LOGIC warning (Pot# differs from RawMaterial)
happened to catch it, not because the data's FORMAT was validated.

→ Recommendation: add a simple "post-OCR normalization" layer (strip label
prefixes with something like `/^(Pot#|Lot)\s*/i`, warn if Pot#/LOT# is an
unusual length or contains stray characters) — no need to rebuild
anything, just one filtering step before the existing business-logic
warnings run.

---

## 3. INPUT → PROCESS → OUTPUT ARCHITECTURE DIAGRAM

> **Correction**: the first draft of this report said "the system has never
> had an architecture layer diagram" — **WRONG**, one has existed since
> 2026-09-13, and it's more complete than the diagram below:
> [`ARCHITECTURE.md`](../ARCHITECTURE.md) section 0 (a diagram of the
> CURRENT system) and [`THIET_KE_HE_THONG_MOI.md`](../THIET_KE_HE_THONG_MOI.md)
> section 3 (the 7-layer architecture for the NEW system, benchmarked
> against real Odoo/`stock.move`). I missed these because I didn't list
> every `.md` file at the project root before writing this report. The
> diagram below is kept anyway because it differs in one way:
> `ARCHITECTURE.md` §0 draws the diagram at the BUSINESS-STEP level (steps
> 1→4), while the one below adds detail at the TECHNICAL level (OCR/schema/
> API routes) as they actually run today — the two complement each other,
> neither replaces the other.

```
┌─────────────────────────── INPUT (real paper/email/Excel) ──────────────────────────┐
│  Infasco PO email      Split Form (paper)      Historical WORK ORDER    Paper        │
│  (PDF/image)           (paper, handwritten)     (PACKING SLIPS.xlsm)    PACKING SLIP │
└───────┬───────────────────────┬───────────────────────┬───────────────────┬─────────┘
        │                       │                       │                   │
        ▼                       ▼                       ▼                   ▼
┌─────────────────────────── PROCESS (AVP_AI webapp) ───────────────────────────────────┐
│                                                                                        │
│  CaptureGate (shared capture gateway) ──► Gemini Vision OCR (extract.ts)              │
│         │                              │  - TRAVELER_FORM_SCHEMA (PO/Pot#/Weight...)  │
│         │                              │  - SCANNING_FORM_SCHEMA (Split Form)         │
│         │                              │  - confidence + cross-check vs. RawMaterial  │
│         ▼                              ▼                                              │
│  [Human review + click Confirm] ◄── DRAFT (not yet written to the Sheet)              │
│         │           ▲ 7+ warning layers (duplicate lot, Pot# mismatch, Part# missing   │
│         │             from PartControl, 1 Traveler changing unusually, machine clash…) │
│         │             — WARNING ONLY, never auto-blocks/auto-fixes (per                │
│         │             ISO_AI_CONTROL.md principle)                                    │
│         ▼                                                                              │
│  ┌──────────────────────────── GOOGLE SHEET (data store) ───────────────────────────┐  │
│  │  RawMaterial   Warehouse (dead)   FinishGood   PartControl   PackingList         │  │
│  │  1,939 rows    nobody writes      194 rows     Quantity/box  Shipped PS + draft  │  │
│  └──────────────────────────────────┬──────────────────────────────────────────────┘  │
│                                      │ 14 API routes each independently read+JOIN       │
│                                      │ (only 1 route, /api/ledger, is a centralized     │
│                                      │  JOIN — currently used by Packing List only)     │
└──────────────────────────────────────┼─────────────────────────────────────────────────┘
                                        ▼
┌─────────────────────────── OUTPUT (what people see/receive) ─────────────────────────┐
│  Warehouse table (Forecast↔Finished-good reconciliation)   Printed Packing Slip       │
│  GlobalSearchBar (search everything)                       /trace (trace 1 traveler)  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

**Is the data store centralized yet?** — **Yes, in the right direction**:
all 5 data types live in **a single Google Sheet** (not scattered across
multiple files as in the old Excel system described in the previous
report) — a genuine improvement over the old system. But in practice only
**4 of 5 tabs are actually "alive."** The `Warehouse` tab (shown as dead in
the diagram above) was built for exactly the right purpose — recording
"goods have really arrived at the warehouse" — but nobody has written to it
since the redesign (a gap already noted in the previous report, section 6,
question #3) — meaning a standalone "Goods Receipt" step is, in practice,
still missing, even though a place to store it already exists.

---

## 4. PRIORITY RECOMMENDATIONS

1. **Identify the source of the bad data batch at 21:06:05 on 2026-09-14**
   (section 1.2b) — need to know what action you or staff took, to patch
   the right place and avoid a repeat (currently 7 rows, 5 travelers
   affected, 3 of 7 already have a duplicate row overwriting good data).
2. Add a "post-OCR normalization" layer (strip label prefixes, validate
   the format of Pot#/LOT#) — patches exactly the gap that caused the
   717544 and 718643 cases, at low cost.
3. Consider running a periodic "data health scan" script (not just at save
   time) — reuse the very rules already built into
   `finish-good/confirm/route.ts`, applied to ALL existing data instead of
   only new rows.
4. Gradually migrate the remaining screens (PoStatusTable, TravelerSection,
   GlobalSearchBar, reconciliation, trace) to share `/api/ledger` instead of
   each doing their own JOIN — reduces the "fixed it in one place, forgot
   another" risk that has already happened for real (the missed
   shipped=TRUE case).
5. Rebuilding a proper ERP-grade architecture (`stock.move`, a state
   machine, full audit trail) — stick with the decision already made: do
   it in an **entirely separate new project**, not as a major patch to the
   system that's live today.

---

## 5. LESSONS FROM AVP_AI (THIS VERSION) FOR THE NEW PROJECT (proper ERP `stock.move`)

> **Important correction**: `THIET_KE_HE_THONG_MOI.md` (written
> 2026-09-13, 2 days before this report) **already designed most of what
> follows** — the immutable `stock.move` ledger (§8.1), the CCP/human-gate
> that blocks direct writes (§8.3), AI confined to running in a single
> dedicated layer (Part 3, Layer 1). The table below checks each lesson
> against what's already designed, to show what's ALREADY SOLVED versus
> what's GENUINELY STILL MISSING (only #1 and #4) — without repeating what
> already exists.

| # | Lesson | Already in `THIET_KE_HE_THONG_MOI.md`? | What's needed |
|---|---|---|---|
| 1 | Measure each field's real value distribution BEFORE locking the schema | ❌ **NOT YET** — the document goes straight into the proposed design; there is no "measure real data before locking a field" step as a mandatory process | To add — see the addition to `THIET_KE_HE_THONG_MOI.md` below |
| 2 | An immutable (append-only) ledger, never editing a boolean flag in place | ✅ **ALREADY THERE** — §8.1: "`stock.move` — a single IMMUTABLE ledger." Today's traveler 717544 case (received and shipped the same day, initially looking like a duplicate) is NEW real-world evidence confirming this design is necessary — worth citing in the document as a real example | Just add the example, no design change |
| 3 | Exactly one valid path to change a business state, enforced in code | ✅ **ALREADY THERE** — §8.3 + Part 3: "Writing to Layer 3 (the Ledger) only happens AFTER passing the right CCP... there is no shortcut write path." Today's real bug (22 travelers missing `shipped=TRUE`, a script bypassing the CCP) is real-world evidence confirming exactly the risk the document already warned about | Just add the example |
| 4 | Two separate defense layers: normalize FORMAT after OCR, then run BUSINESS checks + a PERIODIC data-health scan (not just at write time) | ⚠ **PARTIAL** — Layer 1 (Capture) / Layer 2 (Core Rule Engine) are already separated, but it doesn't spell out "format cleanup" as its OWN step between the two layers, and there is no "periodically scan already-saved data" | To add — see the addition to `THIET_KE_HE_THONG_MOI.md` below |
| 5 | Warn instead of auto-deciding; a Who/Why/When pattern for EVERY exception | ✅ **ALREADY THERE** — §8.3 CCP-1..4 matches this spirit exactly; CONCESSION already requires "an independent authorized signature" (Part 1, row 3) | Nothing to add |
| 6 | Build fast on an easy-to-change foundation for business logic that's STILL UNCERTAIN; only lock down what's ALREADY proven stable | ✅ **ALREADY THERE** — §8.4 "not proposing everything be done at once, there's a small→medium→large roadmap"; Part 4 places merging the ledgers as "high risk, do it last" | Nothing to add |

The full write-up of each lesson (kept for context/detailed evidence — the
table above is the condensed comparison):

**1. Don't assume what a data field means — measure it across all real data
before designing the schema.** Almost EVERY initial assumption in AVP_AI
turned out wrong once measured against real data: `SHIPPED="0"` was
assumed to mean "shipped," but actually meant "not shipped" (this alone
filtered out 1,811 rows down to just 1 that slipped through); the Part#
suffix was assumed to be a "harmless variant," but was actually "a genuinely
different product" (the Owner corrected this); LOT NO. was assumed to
always be a real lot code, but measured at 59% placeholder text; Pot# was
assumed to be a permanent 1:1 key, but measured at 41.1% reused. → For the
new project: every important data field needs a "measure the real value
distribution" step (as done here) BEFORE the schema is locked — never
inferred from the column name.

**2. A single traveler/code is not guaranteed unique across time — model
the physical reality (containers/numbers get REUSED), don't assume a
"permanent primary key."** Today's traveler 717544 case (appearing both on
an already-shipped PS 30101 and in new RawMaterial) nearly got
misunderstood as a duplicate-number error — it was actually goods arriving
and shipping out the same day. If the data were a LEDGER OF ENTRIES (each
receipt/shipment its own timestamped row) instead of a single row carrying
a boolean `shipped` flag, the "received and shipped the same day" situation
would be naturally clear (2 entries, no contradiction) — this is exactly
why Odoo's `stock.move` is APPEND-ONLY, never edited in place.

**3. Exactly one valid path to change a business state — enforce it in
code, not just by convention.** Today's real bug (22 travelers missing
`shipped=TRUE`) happened because the script that imported historical PS
30098/30101 wrote directly to the Sheet, going AROUND the
`/api/packing-list/approve` function (the only place that "knows" it must
mark something as Shipped). For the new project: every state change
(including one-off migration/import scripts) must go through exactly one
service layer/business function — no writing straight to a table under any
circumstances, not even for a "run once" script.

**4. OCR/AI misreads are a certainty, not a rare risk — 2 defense layers
are needed, not 1.** AVP_AI already has a fairly solid BUSINESS-LOGIC
defense layer (cross-checking Pot# against RawMaterial, warning when a
Traveler changes unusually...) but is MISSING a FORMAT defense layer
(validate/regex-clean the AI's output BEFORE it reaches business checks) —
exactly the gap that let `"Pot#313"` (a stuck label) and the 2-line Part#
cell slip through. For the new project: clearly separate 2 layers —
(a) normalize/validate FORMAT right after OCR, (b) run BUSINESS checks
afterward — and add a PERIODIC "data health scan" over already-saved data,
not just a check at write time (old errors sit silently until someone
happens to notice, as with today's 2 cases).

**5. Warn instead of auto-deciding (ISO_AI_CONTROL) is the RIGHT
philosophy — keep it for the new project; don't let "more proper ERP" be an
excuse to let the system silently overrule a human decision.** Every piece
of bad data found so far has been sitting in a "not yet Shipped, awaiting
human approval" state — there has never been a case where the AI wrote bad
data straight into an ALREADY-FINALIZED document with no human review. The
`concessionBy/Reason/At` and `priorityBy/Reason/At` pattern (recording
exactly what the AI/who did, when, and why) should become the STANDARD
template for EVERY exception/override in the new project, not just these 2
cases.

**6. Building fast on an easy-to-change foundation (Google Sheets) BEFORE
the business logic is fully understood was the right strategy, not a poor
shortcut.** Many architectural decisions in AVP_AI only turned out correct
AFTER building something and then discovering the mistake (e.g. PartControl
had to be redesigned for Part# matching after measuring a real 85% impact).
Had "proper ERP" been done from day one (a rigid schema, strict migrations),
the cost of fixing each discovered mistake would have been much higher.
For the new project: business areas that are STILL UNCERTAIN (e.g. the
exact meaning of each Part# suffix, official machine/employee codes —
still awaiting the Owner per the previous report) should keep being
prototyped quickly; only the parts that have ALREADY PROVEN stable (the
basic receive→pack→ship flow) are worth investing in a proper `stock.move`
ledger from the start.

---

## 6. VERIFYING FLOOR-LEVEL PAIN POINTS WITH DATA (planner/manager interview, 2026-09-15)

*Andy directly interviewed the planning staff + observed the floor +
listened to the manager describe how a real Packing Slip gets made. Below
is each claim, checked against measured data — **confirming** what could
be measured, and clearly stating **what could not** rather than nodding it
through.*

### 6.1 "No proactive planning — whatever comes in from the PO gets worked whenever"

> Real observation: a new PO arrived at 1pm yesterday, and that same
> morning staff were still working leftover stock from Friday. Planning
> staff dig the PO out of email themselves, with no formal plan produced —
> raw material arrives in a Pot/Bin and simply "drifts" onto whichever
> machine is convenient, with no formal hand-off of output to a machine.

**Assessment: CANNOT be measured directly from the Sheet** — RawMaterial
only records the **Date**, not the **time of receipt/time of planning**,
so the exact "1pm" window cannot be reconstructed from data. This is a
credible direct observation (a repeated behavior, not a one-off), but it
must be stated plainly: **there is no numeric evidence** for the timing
detail specifically.

**Indirect but solid evidence**: the system (both the old Excel files and
the new AVP_AI) has **no data field whatsoever recording a "production
plan"** — no Work Center Master, no machine-assignment schedule, no field
for "planned to run on which day." The fact that "planning cannot be
measured" is a direct consequence of **the concept of a plan never having
existed in the data at all** — matching GAP-015/Stage 3 already noted in
`BAO_CAO_TONG_HOP_AVP.md` section 5.

### 6.2 "Many travelers overlap on 1 machine — can't trace which shift/machine/batch, huge output discrepancies"

**Assessment: CONFIRMED by data, including new data measured today.**

- Already measured (`BAO_CAO_TONG_HOP_AVP.md` section 2): **381/614 (62%)**
  Day+Machine+Station groups have ≥2 different travelers sharing a
  machine, up to **13 travelers on 1 machine in 1 day**.
- **Newly measured today** (cross-checking `/api/reconciliation`, 1,911
  real travelers, TEST travelers excluded): **124 travelers (6.5%) have
  packed quantity (Scanned) off by more than 50%** from the Pieces
  originally received — exactly the kind of "huge output discrepancy" the
  manager described, occurring in a meaningful share of cases, not a rare
  outlier.

→ Root cause already identified: the Machine/MC# is only recorded AFTER
packing is done (retroactively), never when a run starts — so output
cannot be split by individual batch when several travelers share one
machine on the same day.

### 6.3 "Can't see yield loss, defective goods, returns, or rework"

**Assessment: CONFIRMED**, already measured in `BAO_CAO_TONG_HOP_AVP.md`:

- The `DEFECTS` column: **0/1,811 rows have data** (0% — the column exists
  by name but has never actually been used).
- The `Reject` column holds a free-text, unit-less string (e.g.
  `"5, 154, 59, 3"`) — impossible for a machine to read or total.
- **6/1,847 travelers** show a "sent back for rework" duplication pattern
  that the system has no dedicated rework-order concept for — it just
  vaguely overwrites the old traveler.

### 6.4 "Goods enter production without a real LOT (TRAVELERRECEIVED)"

**Assessment: CONFIRMED on historical data; NOT yet re-confirmed on LIVE
data (the sample is still small)** — these two need to be stated
separately, not merged into one:

- **Historical data** (1,750 real WORK ORDER rows, measured before):
  **53.8%** of LOT NO. values are `TRAVELERRECEIVED` (not yet packed / no
  real lot yet) — exactly as the manager described, this is the MOST
  COMMON state, not an exception.
- **Today's live data** (the new webapp's FinishGood, 194 rows): **0
  rows** still show this placeholder — but the sample is too small (194
  rows, only running for a few days, and it captures LOT# differently from
  the old Excel log) to conclude the issue is gone. **Needs continued
  monitoring as volume grows** — it should not yet be reported as
  "resolved."

### 6.5 "Goods that haven't finished (haven't been wrapped) get shipped anyway"

**Assessment: NOT ENOUGH INDEPENDENT EVIDENCE — this is the one single
point I could NOT confirm with data today**, despite looking carefully.

Scanning all of live FinishGood: there are exactly **7 rows marked
"Shipped" with Box+Qty both blank/0**. But tracing them back, all 7 belong
to 2 ALREADY-KNOWN issues that were ALSO FIXED today (5 rows from the
incomplete PS 30101 historical import — the 22-traveler fix; 2 rows from
the PS 30102 Quantity=0 bug) — **not NEW, INDEPENDENT evidence** that
"shipping before wrapping is finished" is an ongoing habit. Suggestion:
watch NEW PSs created from now on (excluding re-imported historical PSs) —
if this pattern still shows up in NEW data, that would be much stronger
evidence.

### 6.6 "Handwriting on the floor forms can't be recognized because information is missing"

**Assessment: CORRECT IN PRINCIPLE, but the system currently does NOT
store enough to re-measure what % of forms are hard to read.** The Split
Form scanning schema (`SCANNING_FORM_SCHEMA`,
`webapp/src/lib/extract.ts`) already has a `confidence: "low"` field for
the AI to flag "not sure about this reading" — the right mechanism to
have — but this field is **never written into the FinishGood Sheet** after
saving, so there's no way to tally "what % of rows the AI once flagged as
uncertain." **Recommended addition**: store `confidence` as a column in
FinishGood so this rate can be measured over time.

### 6.7 "A few hundred pieces of good surplus pile up, no one knows how, no one knows the lot/quality code"

**Assessment: VERY STRONGLY CONFIRMED — this is the point the manager
described most accurately, and there is now DIRECT NUMERIC EVIDENCE from
the very file management sent today** (`Data/Partial Boxes 2026.xlsx`, see
also the earlier question about PartControl):

- **1,026 rows** logging partial cartons across 42 daily logs spread
  through 2026.
- **560/1,026 rows (54.6%) — more than half — have NO Traveler#**
  attached → the Part#/PO/QC status of that surplus cannot be traced back.
- **95/1,026 rows (9.3%) have no LOT#**.

→ This is EXACTLY the "piling up with no known lot/quality code" the
manager described — not a feeling, backed by real numbers. It's also
evidence the manager has ALREADY recognized the problem and built their
own tracking log (in Excel, outside the main system) — a good sign for
persuasion: the manager is already primed to support a fix for exactly this
pain point.

### 6.8 "Packing Slip staff have to go down to the floor to find the Scanning Form and see which box QC checked off"

**Assessment: VERY STRONGLY CONFIRMED with data newly measured today — the
single most striking number in all of section 6.**

**98.5% (191/194) of live FinishGood rows have a COMPLETELY EMPTY
`qcStatus` field** — only 2 rows ever show "PASS," 1 row ever shows "HOLD,"
across the entire time the new webapp has been running. In other words,
even though the new system already has a column designed to capture QC
status, it **has almost never actually captured a PASS/HOLD signal from
the real paper form** — matching exactly what was described: Packing Slip
staff cannot trust the system to know what passed, and are forced to walk
to the floor to read the checked box on the paper form directly.

### 6.9 "The WORK ORDER sheet: Shipped is wrong because the floor QC system was never linked in"

**Assessment: CONFIRMED**, already measured: **797/1,847 (43%) of
travelers have a mismatched SHIPPED status/Packing Slip number** between
the 2 parallel Excel logs in use — exactly matching "the sheet's name/status
isn't accurate."

### 6.10 "The manager loves the scanning feature — but doesn't yet understand that scanning will produce errors without a disciplined system and oversight behind it"

**Assessment: CORRECT, and today produced immediate real evidence
illustrating exactly this claim** (not theory): in a single working session
today, 3 real errors happened — traveler 717544 read into the wrong column
(Part#/Pot#/LOT# mixed up), 22 travelers missed the "shipped" flag (from an
import script that bypassed the standard process), and 7 FinishGood rows
with an identical Pot#=LOT# from a source not yet identified. **None of
these 3 errors were caused by the AI/scanning technology itself being
poor** — all 3 happened because of a MISSING process-control step
(cleaning up format, a single write path for status, confirming the source
of imported data). Exactly matching the manager's point: a good tool
without disciplined process behind it still produces errors — they just
happen FASTER and are HARDER TO SPOT (since there's no longer slow manual
typing that naturally causes a self-check).

### 6.11 "The Owner wants to reduce headcount — but without a thorough overhaul, no one can actually solve this"

**Assessment: CORRECT in principle, with direct supporting data.** Initial
estimated labor cost (`BAO_CAO_TONG_HOP_AVP.md` section 4, measured
2026-09-14): **21-28 hours/day** spent on manual typing + searching for
data, ≈5,250-7,000 hours/year. **Revised on 2026-09-15** based on
real-world observation: this figure was measured too low — a more
reasonable total is **roughly 32 hours/day** (≈4 labor-days/day), of which
~7 hours/day is still pure manual typing and the remaining **~25 hours/day
is ineffective searching/cross-checking** — ≈**8,000 hours/year** (not yet
re-measured by actual time-tracking, so this should be treated as a
revised estimate, not an exact measurement). If the
response is only to "bolt on a scanning tool" WITHOUT fixing the 3 root
causes noted in sections 6.1-6.9 (no planning, can't trace machine/shift,
QC not linked into the system), headcount will NOT actually go down — it
will just shift from "slow manual typing that at least self-checks" to
"fast scanning that no one checks," exactly the risk noted in 6.10.
Sustainably reducing headcount requires exactly the 3 things the
Owner/manager have already pointed to themselves: **traceability**,
**updating status at the source (not retroactively)**, and **CCP
oversight** — all 3 are already in the proposed design in
`THIET_KE_HE_THONG_MOI.md` (Layer 3 Ledger, Layer 5 CCP) — the problem is
not a LACK OF IDEAS, it's a LACK OF IMPLEMENTATION.

### Section 6 summary

| # | Claim | Confirmed with data? |
|---|---|---|
| 6.1 | Reactive, unplanned production | ⚠ Can't measure directly (no time-of-day data), but solid indirect evidence |
| 6.2 | Machine overlap, huge output discrepancies | ✅ 62% overlapping groups + **124 travelers (6.5%) off by >50%** (newly measured today) |
| 6.3 | Can't see yield loss/defects/returns/rework | ✅ DEFECTS 0% used, Reject unstructured, 6 travelers with no rework concept |
| 6.4 | Enters production with no real LOT | ✅ 53.8% TRAVELERRECEIVED historically — ⚠ live-data sample too small to conclude |
| 6.5 | Shipped before wrapping is finished | ❌ Not enough independent evidence — the 7 rows found all belong to 2 already-known cases |
| 6.6 | Handwriting the AI can't read | ⚠ Correct in principle, but the system doesn't store `confidence` so it can't be re-measured |
| 6.7 | Surplus with no known lot code | ✅ **54.6% of Partial Boxes rows have no Traveler#** (newly measured today) |
| 6.8 | Must walk to the floor to check the QC form | ✅ **98.5% of live FinishGood rows have qcStatus EMPTY** (newly measured today) |
| 6.9 | WORK ORDER Shipped is wrong because QC isn't linked | ✅ 797/1,847 (43%) mismatched SHIPPED/PS |
| 6.10 | Scanning will fail without process discipline | ✅ Illustrated by exactly 3 real errors that happened today |
| 6.11 | Reducing headcount requires a thorough overhaul | ✅ ~32 hours/day wasted (revised from 21-28 hours measured before) — a tool alone won't fix the root cause |

**8 of 11 clearly confirmed with data (3 figures measured COMPLETELY NEW
in today's session: the output-discrepancy part of 6.2, section 6.7,
section 6.8), 2 correct in principle but lacking re-measurable data, 1 not
yet independently confirmable.** This is an unusually high match rate
between floor-level testimony and measured data — this table is worth
using directly when working with the Owner: **9 of the 11 things the
manager/planner described in words already have confirming data sitting
directly inside AVP's own running data.**

---

*This report is based on live data as of 2026-09-15 — the number of
RawMaterial/FinishGood rows will keep growing every day, so the percentages
here should be read as a snapshot at the time of writing, not a fixed
constant.*

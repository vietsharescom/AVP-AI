# REPORT FOR OWNER — FULL PROCESS FROM PO EMAIL TO TRUCK DEPARTURE
## Current State — Measured Consequences — Fix Direction — Expected Result

*Date: 2026-09-15. Read once: each row is one real process step, left to
right: what is being done today → what consequence it is causing (measured,
not a feeling) → which direction to fix it → what you get once it's fixed.*

> **No number in this table is a rough guess** — every number is taken
> directly from the Excel files, PDFs, and Google Sheet currently in real
> use, with the source cited at the end of each row so it can be checked
> again at any time.

---

## PROCESS TABLE — FROM PO EMAIL TO TRUCK LEAVING THE DOCK

| # | Process step | What is done today | Consequence today — with evidence | Fix direction → Result it brings |
|---|---|---|---|---|
| 1 | **Receive PO / plan production** — Infasco email announces the incoming lot | Planning staff open the email and read it manually, **no written production plan is created** — raw material arrives in Pots/Bins and simply "drifts" onto whichever machine is convenient, no one formally assigns output to a machine | Not a single data field (in either the old Excel files or the new system) records a "production plan" — 0% of production is planned with numbers. Yesterday's PO arrived at **1pm**, and that same morning staff were still working leftover stock from Friday (direct, repeated observation) | Read the PO automatically the moment the email arrives, suggest machine assignment based on which machines are free — **1 planner decides in 5 minutes instead of a whole shift guessing** — cuts labor at the single most labor-hungry step |
| 2 | **Receive physical raw material** | No step confirms "actually arrived at the warehouse" — goods are simply assumed present once they show up, no one signs off on a real physical count | The `Warehouse` tab was built exactly for this but **no one has entered anything into it since the redesign** — the single biggest gap in the system, already flagged before | 1 person scans the Pot# barcode on arrival (phone/scanner), confirms the count — **no extra headcount needed, it just piggybacks on the receiving step that already happens** |
| 3 | **Enter production — run on machines** | Multiple travelers run on the same machine at the same time; the Machine/MC# column is only filled in **AFTER packing is done** (retroactively), never when the run starts | **62% (381/614)** of Day+Machine groups have ≥2 different travelers sharing one machine, up to **13 travelers on 1 machine in 1 day**. Newly measured today: **124/1,911 travelers (6.5%)** have packed quantity off by **>50%** from the raw material received — no one knows where it went because output can't be split by batch | Record which machine is running the moment a traveler is assigned (scan a machine barcode at start) instead of retroactively at shift end — **ends wrongly blaming workers for "yield loss" caused by mixed-up records, cuts investigation time when something looks off** |
| 4 | **QC / sorting — paper Split Form** | QC hand-writes PASS/HOLD on paper, checks a box — **no one re-enters it into any system** | **98.5% (191/194)** of live data rows have a **completely empty** QC status field — only 1 single row has ever actually been blocked by the system for HOLD in its entire runtime. Not because everything always passes — because the system simply doesn't know | QC scans/photographs the form the moment it's signed — **Packing Slip staff no longer need to WALK DOWN TO THE FLOOR to find the paper form** every time they need to know what passed — currently the single most labor-consuming part of preparing a Packing Slip |
| 5 | **Pack / wrap finished goods — Scanning Sheet** | **100% handwritten on paper, then re-typed by hand** into Excel (19 of 20 columns, only 1 column has a formula) — no one at AVP yet uses a scanner/AI to read this step, it is fully manual start to finish | No one can measure which forms are hard to read or by how much — there is no step that detects illegible handwriting on its own; it only surfaces once a typo/mis-entry happens, usually far too late | This is a genuine gap, nothing built yet: add a scan/photo step in place of manual typing, so the system flags "this is hard to read, needs a human look" **before saving** — not yet deployed at AVP, this is a proposed direction only |
| 6 | **Track surplus (good extra pieces, partial cartons)** | Management keeps a separate personal Excel file (`Partial Boxes 2026.xlsx`), hand-logged daily, outside the main system | **54.6% (560/1,026)** of partial-carton log lines have **no Traveler#** attached → Part#/PO/QC status of that surplus can't be traced. **9.3%** have no LOT#. Surplus is "piling up with no known lot/quality code" — exactly as management describes, now with real numbers | Require Traveler#/LOT# the moment a partial carton is logged — know exactly how much, from which lot, reconcile correctly when returning to Infasco, no more untraceable pile-up |
| 7 | **Prepare the Packing Slip** | PS staff go through several different Excel files to find codes, find travelers, and pick manually — rushed because the truck is already waiting | **797/1,847 (43%)** of travelers have mismatched SHIPPED status/PS number between the 2 parallel spreadsheets — one sheet says "not shipped" while the other says otherwise, staff have to guess which one to trust | 1 single source answers "has this traveler shipped, has it passed QC" — **PS staff no longer need to open 3-4 Excel files or manually cross-check every time they prepare a slip** |
| 8 | **Ship / load the truck** | Packing Slip number is hand-typed from a source outside AVP, not auto-generated; the Part# printed on the packaging label sometimes differs from the Part# written at the top of the traveler | Actually happened: **PS 30102 was really shipped to the customer with 2 lines showing Quantity = 0** — the error was only found on a later self-check, the customer was never informed | Hard-block shipping while any line is missing a quantity or still on unresolved HOLD (already built in the new webapp) — **cuts the risk of sending the customer wrong data, cuts the work needed to handle complaints/redo shipments afterward** |
| 9 | **Archive history + management KPI reporting** | Manually counted from several logs (`ARCHIVE`, `ARCHIVE2`, `SUMMARY IN&OUT`) every day to produce the report | **0 of 30 sample days** had the hand-counted report match the original WORK ORDER data — on some days the gap reached **105 lines**. Management has been deciding based on numbers that may be wrong, without knowing it | KPI figures computed automatically from a single source — **eliminates the daily "count it by hand" task entirely** — this is the step where headcount can be reduced fastest and with the least risk, since the work is pure recounting, not expert judgment |

---

## WANTING TO REDUCE HEADCOUNT — CORRECT, BUT ONLY WHERE IT ACTUALLY WORKS

Re-read column 2 of all 9 rows above: most of today's human effort is **not**
"making product" — it is **searching, cross-checking, recounting, and
guessing at numbers** across records that don't agree with each other. This
is exactly the part a tool can take over, without touching the people who
directly produce or pack:

- Steps 1, 7, 9 — **can reduce headcount immediately, lowest risk**:
  planning, finding a traveler to build a PS, counting KPIs for reports —
  all three are "typing + cross-checking" work, not expert judgment calls.
- Steps 3, 4, 8 — **cut the time spent handling incidents later**; they
  don't reduce headcount directly, but they reduce how many people are
  needed to "put out fires" when a complaint or a wrong shipment happens.
- Steps 2, 5, 6 — **don't reduce headcount, but cut major risk**
  (Goods Receipt, handwriting quality, unresolved surplus) — a small
  investment that prevents damage worth many times its cost.

**Current estimate**: 21-28 hours/day spent on manual typing + searching for
data (≈5,250-7,000 hours/year, source: `BAO_CAO_TONG_HOP_AVP.md` section 4,
measured 2026-09-14) — the "searching/cross-checking" portion alone makes up
**75% of that figure** (~14-21 hours/day).

> **2026-09-15 update**: the figure above was measured too low. On
> reconsideration, a more reasonable total is **roughly 32 hours/day**
> (≈4 labor-days, at 8 hours per labor-day) — of which ~7 hours/day is
> still pure manual typing, and the remaining **~25 hours/day is
> ineffective searching/cross-checking**. ≈**8,000 hours/year**. This is
> an adjustment based on direct real-world observation, **not yet
> re-measured by actual time-tracking** — it should be presented to the
> Owner as "an estimate revised from new observation," not an exact
> measurement, to keep the numbers honest rather than inflated.

This is exactly the part a new tool can replace — not the part that needs
more or fewer people making the actual product.

---

*See Part 2 (visual charts — Infasco vs. AVP, and target metrics for the new
system) in the attached visual presentation page.*

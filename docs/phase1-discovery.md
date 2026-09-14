# KYTC Project Mix Tracker — Phase 1 Discovery

Date: 2026-09-10. No code written. Everything below was read from the
`ils-integration` repo and queried live against Supabase and the Allen QC MCP.

## 1. ILS → Supabase schema (real names, not guessed)

Supabase project: **`allen-qc`**, ref **`knaeexnlyfjgpowihcel`**, region us-east-2,
Postgres 17. URL `https://knaeexnlyfjgpowihcel.supabase.co`.

Two raw tables (upserted by `ils_supabase_sync.py`) and three views. Apps must
read the views, never the raw tables.

| Object | Kind | Purpose |
|---|---|---|
| `ils_tickets` | table | Ticket headers, one row per `(site, ticket_no, rev_no)`. 269,582 rows. |
| `ils_ticket_detail` | table | One line per ticket. Carries `mat_code` (the mix code). |
| `ils_tickets_latest` | view | Highest `rev_no` per `(site, ticket_no)`, active or not. |
| `ils_tickets_current` | view | `ils_tickets_latest` where `status is null`. **Use this for tonnage.** |
| `ils_ticket_lines_current` | view | `ils_tickets_current` joined to detail. Adds `mat_code`, `qty`, `haul_date`. |

Column names the tracker needs (all on `ils_ticket_lines_current`):

| Concept | Column | Notes |
|---|---|---|
| Job number | `job_code` | text, e.g. `125367` |
| Mix code | `mat_code` | text, on detail. Asphalt codes look like `3038A64H11`; 3-digit codes are aggregates. No description column is synced. |
| Plant | `site` | 2-char ILS site code (see below) |
| Ticket timestamp | `date_out` | **Eastern wall-clock stored as if UTC** (see gotcha 1) |
| Net tons | `net_tons` (header) / `qty` (line) | Identical for asphalt: every asphalt-plant ticket in 2026 has exactly one detail line (13,479 checked). Use `qty`. |
| Ticket number | `ticket_no` | 8-char text, unique within `site` |
| Revision | `rev_no` | `'00'` original, `'*01'`, `'*02'`… edits. Not numeric — views strip non-digits before casting. |
| Void / edit indicator | `status` | `NULL` = active. `'V'` = voided. `'E'` = superseded by a later revision. |
| KYTC phase | `phase_code` | Allen cost code, e.g. `572250.01.0160`. Last segment is **not** the proposal line number on this job. |
| Truck | `truck_code` | for the "sample truck" display |
| Sync bookkeeping | `date_mod`, `synced_at` | `date_mod` is true UTC from ILS `getutcdate()` |

### How a void / edit appears

Verified against real 2026 tickets:

- **Void:** the ticket's latest revision has `status = 'V'`. `ils_tickets_current`
  drops it. 86 voided asphalt-plant tickets in 2026 (610 tons) are excluded this way.
- **Edit:** ILS inserts a new row with the next `rev_no` (`*01`, `*02`…) and marks the
  old row `status = 'E'`. The new row carries the corrected `mat_code`, `job_code`,
  or `phase_code`. Example: ticket `03/00146456` rev `00` was mix `3038D64F00`, rev
  `*01` is `3038B64H11`. **An edit can move tons from one bid item to another after
  the fact**, so the tracker must recompute cumulative tons from the view on every
  poll, never accumulate deltas client-side.
- Status breakdown across the whole table: 242,094 active, 26,788 `E`, 700 `V`.

### ILS site codes

| `site` | Plant | Kind |
|---|---|---|
| 02 | Boonesboro Quarry | quarry |
| 03 | Danville Asphalt Plant (DBT) | asphalt |
| 04 | Berea Asphalt Plant (BBT) | asphalt |
| 05 | Clover Bottom Asphalt Plant (CBBT) | asphalt |
| 06 | Boonesboro Asphalt Plant (BT3) | asphalt |
| 07 | Lexington Quarry | quarry |
| 08 | Lexington Asphalt Plant (LQBT) | asphalt |
| 09 | Clover Bottom Quarry | quarry |

Source: `netlify/functions/lib/tools.mjs` in ils-integration.

### Sync freshness

Newest ticket at query time: `date_out` 2026-09-09 14:57 (Eastern), `date_mod`
18:57 UTC, `synced_at` 18:58 UTC. Sync lag under 1 minute after the ticket landed.

The Task Scheduler job runs **every 15 minutes, round the clock** (confirmed by Jake
2026-09-14; the ils-integration README still says every 2 hours and is stale). The
dashboard's "sync may be behind" warning fires when the newest ticket is older than
30 minutes during a production day, i.e. two missed cycles.

## 2. Gotchas found (these change the design)

1. **`date_out` is Eastern wall-clock mislabeled as UTC.** For freshly inserted
   tickets `date_mod − date_out` is 4.0–4.7 h, and ticket hours cluster 07:00–16:00
   "UTC". Read it as `date_out at time zone 'UTC'` to get the local clock; do **not**
   convert with `at time zone 'America/New_York'`. The existing view column
   `haul_date` does exactly that conversion, so any ticket between midnight and 04:00
   local lands on the previous day. The tracker should compute its own production
   day and not use `haul_date`.
2. **Night paving crosses midnight.** Richmond Bypass (job 125367) ran 21:00–04:00.
   A strict midnight rollover would split one shift into two "production days" and
   fire the 50-ton rule twice. **Decision (Jake, 2026-09-14): strict midnight,
   Eastern.** A shift that crosses midnight is two production days, and the 50-ton
   rule re-arms at 00:00. The tracker follows the spec literally here.
3. **`rev_no` is not numeric** (`*01`). Any revision ranking must strip non-digits
   first, as the views do.
4. **`phase_code` ≠ proposal line.** On 125367 the 0.38A surface ran under
   `572250.01.0160` and `572450.01.0160`; the proposal line is 0320. Map bid items by
   `mat_code`, and show phase codes as information only.
5. **Anon reads are open.** `ils_tickets` and `ils_ticket_detail` have `SELECT`
   policies for `public`, so the anon key can read the views directly. Good for the
   dashboard; the `tracker_*` tables will need their own policies.

## 3. Contract lookup — the prompt's CID/job placeholders were blank

The prompt left `<FILL IN CID>` and `<FILL IN JOB NUMBER(S)>` unfilled, so I picked
the best candidates from the `projects` table (which maps `allen_job_number` →
`contract_id` for KYTC jobs) and asphalt-plant tonnage in 2026.

### Recommended primary test: CID **252112**, job **125367**

Dr. Robert R. Martin Bypass / Lexington Road (US 25), Madison Co. Asphalt
resurfacing, let 2025-07-24. Allen name "MADISON CO. RICHMOND BYPASS CID 252112".
All asphalt shipped from site 06 (Boonesboro). Night work, Jul 12 – Aug 27 2026.
Why: 17,457 tons already placed = 4+ lots of real lot math, and the primary item is
already past 100 %, which exercises the overrun display.

Mix bid items on 252112 (from `lookup_contract` / `search_bid_items`):

| Line | Bid code | Description | Plan qty | Linked JMFs |
|---|---|---|---|---|
| 0320 | 22906ES403 | CL3 ASPH SURF 0.38A PG64-22 | 15,200 T | 00260175 (Haydon, plant code `3038A64H11`), 00260002 (Greensburg), 00260116 (Gaddie) |
| 0050 | 00301 | CL2 ASPH SURF 0.38D PG64-22 | 2,800 T | 00250600 (Natural Sand), 00250602 (Virgin) |
| 0040 | 00190 | LEVELING & WEDGING PG64-22 | 375 T | none |
| 0110 | 02677 | ASPHALT PAVE MILLING & TEXTURING | 7,585 T | n/a, not a plant-ticketed item |
| 0020 / 0030 / 0060 | 00100 / 00103 / 00356 | seal aggregate / seal coat / tack | 250 / 30 / 110 T | n/a |

Tickets on job 125367 (all site 06, `ils_ticket_lines_current`):

| `mat_code` | Loads | Tons | Avg load | Dates | Phase codes |
|---|---|---|---|---|---|
| 3038A64H11 | 630 | 15,816.83 | 25.1 | Jul 26 – Aug 20 | 572250.01.0160, 572450.01.0160 |
| 3038D64C00 | 1 | 24.87 | 24.9 | Jul 26 | 572250.01.0160 |
| 901 | 40 | 707.34 | 17.7 | Jul 12 – 13 | 571100.01.0055 |
| 120 | 30 | 675.97 | 22.5 | Aug 23 – 24 | 563200.01.0005 |
| 009 | 7 | 158.00 | 22.6 | Aug 25 – 27 | 575600.01.0010 |
| 805 | 11 | 44.01 | 4.0 | Jul 31 – Aug 20 | 575200.01.0030 |
| 460 | 4 | 29.49 | 7.4 | Aug 27 | 575400.01.0015 |
| **Total** | **723** | **17,456.51** | | | |

Smoke-test number for Phase 4: line 0320 should read **15,816.83 tons of 15,200
(104.1 %)**, 630 loads. Whole job 17,456.51 tons.

### Secondary live test: CID **262135**, job **126334**

Big Hill Road (KY 21), Madison Co. Active right now (Aug 24 – Sep 9 2026), and it is
the **two-plants-one-bid-item** edge case: Clover Bottom (05) and Berea (04) both ship.

| Line | Description | Plan qty | Tickets so far |
|---|---|---|---|
| 0050 | CL2 ASPH SURF 0.38D PG64-22 | 3,330 T | `3038D64C01` from site 05: 50 loads, 1,222.38 T (Sep 3 – 9) |
| — | no 1.00D base item found in the ASPH search | — | `3100D64B01` from site 04: 12 loads, 259.81 T (Aug 24 – 26) — **unmapped** |

### Brannon Road (261113 / 126102) is not a good test yet

It is the biggest active KYTC job (50,056 tons) but 49,000+ of that is crushed stone
from Lexington Quarry. Only one asphalt load (25 tons of `3100D64B01`) so far. Base
paving (32,024 T of 1.00D) has not started. Revisit when it does.

## 4. Proposed mapping: bid item → (job, mix code, plant)

Mapping key is `(job_code, mat_code, site)`. Cumulative counter is shared across
tuples on the same line item; per-plant split is shown from `site`.

**252112 / 125367**

| Bid line | → tuples | Confidence |
|---|---|---|
| 0320 CL3 ASPH SURF 0.38A | (125367, `3038A64H11`, 06) | High. JMF 00260175 carries plant code `3038A64H11`. |
| 0050 CL2 ASPH SURF 0.38D | (125367, `3038D64C00`, 06) | Low. One load, keyed to the 0.38A phase code on the first 0.38A night. Almost certainly a mis-keyed 0.38A ticket. Ask before mapping; otherwise it shows as 24.87 T on a 2,800 T item. |
| 0040 LEVELING & WEDGING | none | No leveling mix code appears on the job. |
| unmapped tons | 901, 120, 009, 805, 460 → 1,614.81 T | Aggregate / non-mix codes. `901` ≈ 707 T lines up with DGA BASE (line 0010, 700 T). Need Jake to name these codes. |

**262135 / 126334**

| Bid line | → tuples | Confidence |
|---|---|---|
| 0050 CL2 ASPH SURF 0.38D | (126334, `3038D64C01`, 05) **and** (126334, `3038D64C01`, 04) if Berea ships it later | High for site 05. |
| ? | (126334, `3100D64B01`, 04) 259.81 T | No 1.00D base bid item found. Needs Jake. |

Things that do not line up one-to-one:

1. Three JMFs are linked to line 0320 but only Haydon (00260175) has a plant mix code and tickets. The tracker should map by ticket `mat_code`, not by JMF.
2. The MCP's `jmf_links` entries for the 0.38D item are `class_equivalent_cl2_cl3` matches (CL3 designs on a CL2 bid item), flagged `verified: false`.
3. Aggregate `mat_code`s (3-digit) have no description anywhere in Supabase. The `materials` table uses different codes. The tracker will label them "unmapped" by code until a lookup exists.
4. `phase_code` cannot be used as the line-number key on these jobs.

## 5. Questions for Jake before Phase 2

1. Confirm the test contract: **252112 / 125367** primary, **262135 / 126334** live secondary. Or name a different CID.
2. ~~Production day~~ — answered: strict midnight Eastern.
3. The single `3038D64C00` load on Jul 26: map it to the 0.38D item, treat it as a mis-key and fold it into 0.38A, or leave it unmapped?
4. What are `mat_code`s 901, 120, 009, 805, 460? Should any map to a bid item (DGA base, seal aggregate)?
5. Which bid item does Big Hill's `3100D64B01` base belong to?

## 6. Scope change from Jake (2026-09-14): upload-driven setup

Setup is no longer "type a CID and pull from the MCP". The tech uploads two files
on the site and the site does the rest:

1. **Proposal PDF** (e.g. `323-MADISON-25-2112.pdf`) → CID, county, route, bid
   items with plan quantities. MCP `lookup_contract` stays as a cross-check when the
   CID is already in the corpus.
2. **Approved mix pack** (KYTC mix design submittal/approval workbook) → JMF id,
   mix type, plant assignment, and the **random-number chart** (sublot → random
   tonnage). This is the only source for the random points.

Then the site uses the ILS connection (Supabase views) for tons.

Design implications for Phase 2/3:

- Add `tracker_documents` (project, kind proposal|mixpack, storage path, uploaded_by,
  parsed_at, parse_json) and a private Supabase Storage bucket `tracker-docs`.
- Uploads and parsing go through a Netlify Function (service key). Extracted values
  land on a **review screen** the tech confirms before anything is written. Nothing
  auto-commits from a parse.
- Parser choice (deterministic text/xlsx parsing vs. Claude extraction) is decided
  after seeing one real sample of each file.

Needed from Jake: one real proposal PDF and one real approved mix pack for 252112
(or whichever test CID is confirmed).

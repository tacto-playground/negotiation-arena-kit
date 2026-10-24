# Tacto Negotiation Arena - API v1

The arena speaks JSON over HTTPS. Base URL: `https://arena.tacto.ai/v1`. Every call except `/v1/data`, `/health` and
the leaderboards needs your team's key in the header `X-API-Key`. The starter kit wraps all of this in
`arena_kit.ArenaClient`. To try the API on your laptop, run `arena-sim` from the starter kit: the
same API with one plain scripted supplier - it teaches the contract, not the suppliers.

**Safe retries:** send a fresh `X-Move-Id` header (any unique string) with every `POST`. If the
network drops, repeat the request with the same `X-Move-Id`: the arena returns the answer it
already gave instead of acting twice. Reusing an id for a different request is a `422`. A request
body may be at most 64 KB.

Contents: [Flow](#flow) · [Endpoints](#endpoints) · [Messages](#messages) ·
[What counts](#what-counts) · [Documents](#documents) · [Terms](#terms) ·
[Case file](#case-file) · [Data IDs](#data-ids) ·
[Result](#result-trace-and-back-office) · [Cost and score](#cost-and-score) ·
[Live Final](#live-final-in-waves) · [Time, errors, limits](#time-errors-and-limits)

## Flow

```
GET  /v1/data                                        the data downloads (no key)
GET  /v1/me                                          your team: practice quota, best scores
GET  /v1/cases?set=practice                          the case files you can play
POST /v1/sessions                                    open a case → every supplier's quote e-mail
POST /v1/sessions/{sid}/threads/{supplier}/messages  your e-mail → the supplier's reply  (repeat)
GET  /v1/sessions/{sid}/attachments/{attachment_id}  a document the supplier attached
POST …/messages with "action": accept | walk_away    close the case → result
GET  /v1/sessions/{sid}                              state; after the end score and trace
```

## Endpoints

### Your key

One key per team - a licence. Get it on the page `https://arena.tacto.ai/apply`: enter your team name and a
contact e-mail, show the match code the page displays at the Tacto desk, and once an organiser
approves, the key appears on that page - copy it and press "Got the key"; it stays there. Only that page
(or its link, in the same browser) shows the key; lost it → ask the organisers to reset it (the old key stops at once).

Send the key as `X-API-Key` with every call. A missing or unknown key is a `401`; a paused or
revoked key gets `403 "this key is paused - ask the organisers"` (or `revoked`) on every call.
There is no self-registration: `POST /v1/register` answers `403 "licences are issued by the
organisers"` (`arena-sim` keeps its own, for the simulator only).

### `GET /v1/me`

Where your team stands:

```json
{"team_name": "Haggle Hounds", "status": "active",
 "practice": {"used": 37, "left": 163, "limit": 200, "resets_at": "2026-10-24T13:42:10+00:00"},
 "best": {"case:P01": 0.71, "case:P04": 0.55}}
```
`practice`: sessions used and left in the rolling 24 hours, and when the oldest of them ages out
(the next one frees up); `best`: your best score per hosted practice case.

### `GET /v1/cases?set=practice|sandbox|final`

A list of full [case files](#case-file), so you can prepare before you open a session.

| Set | Cases | Replay |
|---|---|---|
| `practice` | all practice case files, to study; the **10 hosted** ones carry `"hosted": true` and can be played any time within your quota (a session on another is `403 "case:P06 is not hosted for practice"`); your best score per hosted case counts | right after the case |
| `sandbox` | `case:D01` (a price increase) and `case:D02` (an RFQ with two suppliers): three demo suppliers to try things on; no leaderboard | right after the case |
| `final` | the hidden cases (`"set": "hidden"`), wave by wave - see [Live Final](#live-final-in-waves); `403` before the first wave | after the final |

The replay is the same team view in every set: the e-mails as sent, terms, statuses, claim
results, the supplier's reasons in words and the score - as a real supplier would tell you. No rule
names, no internal parameters, none of the supplier's hidden figures (shown as `…`).

### `POST /v1/sessions`

`{"case_id": "case:P01"}` opens a case. Every supplier sends its quote e-mail at once (round 0);
from now on the 60-second clock runs per thread.

| Response field | |
|---|---|
| `session_id` | use in all further calls |
| `case` | the case file |
| `threads` | `{supplier_id: [opening message]}` - each supplier's quote e-mail, quote PDF attached |
| `opening` | the same quotes in the structured form of the `/moves` path |

Practice: your best score per case counts; at most 10 sessions open at a time (`429`).
**Quota: 400 practice sessions per team per rolling 24 hours.** Every practice session start
answers with the header `X-Practice-Remaining: <sessions left>`. At the limit: `429` with a detail
starting `practice limit:` that names `resets_at`, `X-Practice-Remaining: 0` and no `Retry-After` -
retrying does not help before then. Sandbox and final sessions do not count. Final: one session per team and case; a second is a `409` naming the running session -
pick it up again with `GET /v1/sessions/{sid}` and `GET …/threads/{supplier_id}`.

### `POST /v1/sessions/{sid}/threads/{supplier_id}/messages`

Body `{"message": Message}` (see [Messages](#messages)). Returns the supplier's reply, a Message.
With an `action`, it closes the case and returns the [result](#result-trace-and-back-office):
`{"type": "accept", "term_sheet_id": "offer:…"}` takes that supplier's standing term sheet;
`{"type": "walk_away"}` ends the case without a deal.

A supplier's reply can take **up to about a minute**; its time never counts against your 60
seconds. Use an HTTP timeout of at least 180 s (the kit uses 180).

`422`: invalid message - nothing happens, no round is used (e.g. a term sheet without every
negotiable field, a value outside the allowed options, an unknown or stale `term_sheet_id`, more
than 3 term sheets). `409`: the thread or session has ended (in the final also at the wave's cap),
you reached the round limit (you may still accept), or your message came too late - it counted
as a timeout; `GET` the thread for the supplier's answer.

### `GET /v1/sessions/{sid}/threads/{supplier_id}`

The thread: `supplier_id`, `offer_id` (its standing term sheet), `status`, `closed` (why the thread
ended, else null), `messages` (all e-mails, both directions, in order; yours carry `"from":
"buyer"`), `timeouts` (the supplier's answers to your late moves). Enough to pick a case up again
after a crash.

### `GET /v1/sessions/{sid}/attachments/{attachment_id}`

The file of a supplier attachment (`application/pdf`, xlsx, ...), rendered from the supplier's
data. See [Documents](#documents).

### `GET /v1/sessions/{sid}`

The session state: `session_id`, `case_id`, `set`, `status` (`open` | `ended`), `end_reason`
(`accepted`, `walked_away`, `no_deal`, `timeout`, `wave_cap`, `window_closed`), `rounds`, `opening`, `threads`
(per supplier: `offer_id`, `status`, `offers`, `closed`, `seconds_left`) and, after the end,
`result` and `replay_url`.

### `GET /v1/sessions/{sid}/backoffice`

The supplier's roles talking internally (who asked whom, approvals, documents attached), in the
same team view: their words, without hidden figures or internal names. Only after the case has
ended - in the final after the final; before that `403`.

### Lists and boards

- `GET /health` - is the arena up? No key needed.
- `GET /v1/data` - the data downloads, no key needed: `{"package": {"url": "/v1/data/package.zip"},
  "release-sunday": {"public": false, "at": "2026-10-25T07:00:00+00:00"}}`. Once the Sunday
  release is public it carries `"url": "/v1/data/release-sunday.zip"` - new news, letters and a
  memo, with their own `facts.json` (new IDs only; package v1 stays as it is). Hidden cases may
  cite them; practice cases stay on package v1.

- `GET /v1/sessions?set=practice|sandbox|final` - your sessions: `session_id`, `case_id`, `status`,
  `end_reason`, `deal`, `score`, `rounds`, `created_at`.
- `GET /v1/leaderboard?set=practice|final` (JSON), `GET /leaderboard` (HTML screen) - no key
  needed. Practice: `total` = the mean of your best score per hosted case × 100 (cases not played
  count 0). Final: hidden cases only, live, as **points so far**:
  ```json
  {"set": "final", "cases": 6,
   "teams": [{"team_name": "Haggle Hounds", "played": 6, "deals": 6, "rounds": 21,
              "waves": {"1": 37.0, "2": 40.8}, "total": 77.8, "rank": 1,
              "saved_eur": 48210, "saved_eur_waves": {"1": 30115, "2": 18095}}],
   "waves": [{"n": 1, "theme": "Opening", "running": false, "closed": true, "seconds_left": 0}, …]}
  ```
  Per team, `waves` holds each started wave's score: the mean of its case scores × 100, 0-100, one
  decimal (a case you did not play scores your alternative once its wave has closed). `total` is
  the sum of those wave columns as shown - your points so far, up to 500 after five waves. Ranked by
  `total`; an exact tie goes to more `deals`, then fewer `rounds`. `cases` counts the cases of the
  waves started so far. **€ saved** (`saved_eur`, whole euros, per started wave in `saved_eur_waves`)
  is a side number and does not rank: per ended session `C_open − C_end`, where `C_end` is `c_deal`,
  or `c_alt` without a deal (0 if the case has no alternative) - money saved against the cheapest
  opening offer, over the case's horizon (`horizon_years`, 1-2 years), the same C as
  the score. It can be negative (a deal or an alternative dearer than the opening offer); a case
  you did not play saves nothing, nor does a deal that breaks a must-have (it scores 0). In the final the feed's `deal` and `no_deal` events carry it too.
  During the Live Final your agent must not read the leaderboard, the results, the screen or the
  event feed (`/v1/events`, `/v1/events/stream`, `/screen`, `/v1/screen`, `/v1/results`, `/results`)
  - they are for people.
- `GET /v1/results` (JSON), `GET /results` (page) - no key needed: the final's standings for people,
  every team: the final leaderboard above plus `"replays": true` once the final is closed. Before
  the first wave: no teams, nothing about the plan. `GET /v1/results/{team_name}` - one team's cases
  of the **started** waves: per wave `n`, `theme`, `running`, `closed`, `points`, `saved_eur` and
  `cases` - `case_id`, `title`, `type`, `suppliers` (names), `status` (`not started`, `in play`,
  `deal`, `no deal`; `not played` once its wave has closed - it scores your alternative), `end_reason`,
  `score`, `saved_eur`, `rounds`, and after the final `replay_url`. Both cached for 5 seconds.
- `GET /live` - the build page, no key needed (from the kickoff to the code freeze): every team's quota use,
  practice records and highlights.
- `GET /replay/{sid}` (link in the session's `replay_url`): a finished negotiation step by step -
  every e-mail and the supplier's reasons for each move in words, the same view in every set.
  Play, read the replay, improve, play again: that is the learning loop. Final: after the final,
  for everyone - the results page links each final session's replay then.

### The structured path

For agents that skip prose: `POST /v1/sessions/{sid}/moves` with `{"supplier_id": …, "move":
{"type": "offer", "offer_id": "<standing offer>", "terms": {…}, "claims": [{"type", "refs",
"asserted"}], "text": "≤ 800 chars"}}` returns a structured supplier move; `POST …/accept
{"offer_id"}` and `POST …/walk_away` close. Same supplier, same rules; claims are typed explicitly
(see [What counts](#what-counts)) instead of read from the message.

## Messages

| Field | Buyer | Supplier |
|---|---|---|
| `subject` | ≤ 300 chars | `Re: …` |
| `body` | plain text, ≤ 8,000 chars | the role in charge writes; other roles add paragraphs |
| `term_sheets` | 0-3 alternative packages `{"terms": {…}}`, each with every negotiable field | its standing offer: `{"term_sheet_id": "offer:…", "terms": {…}}` |
| `refs` | data IDs your message relies on (≤ 20); IDs written in the body count too | - |
| `attachments` | ≤ 5 × `{"name", "type": "md" \| "csv" \| "json" \| "pdf" \| "xlsx", "content_b64"}`; md, csv and json are read | `{"attachment_id", "name", "type"}` - fetch the file by id |
| `action` | `null`, `{"type": "accept", "term_sheet_id"}` or `{"type": "walk_away"}` | - |
| `from` | `buyer` | `<role>@<supplier>` |
| `status` | - | `open`, `final_offer` (its last word: push further and it may walk), `agreed` (it offers exactly your terms - you still accept), `walked_away`, `no_bid` |
| `signals` | - | `reply_delay_days` (how long a real reply would have taken), `validity_days` |
| `missing_fields` | - | fields its term sheet leaves out: at a deal they close on its **standard terms** |
| `claim_results` | - | per argument the arena read from your last message: `{"index", "result"}` |
| `thread_id`, `message_id`, `in_reply_to` | optional | set |

A message without a term sheet is fine - a question, say. It is answered, and the quote stands.

A real exchange from practice case `case:P01` (Brandlhuber announces +9.5 % on volute casings,
"due to energy and wages"); the starter kit's README shows how the index figure was found:

```json
{"message": {
  "subject": "HN volute casings - your letter of 11 September",
  "body": "Dear Sir or Madam,\nthank you for your letter. We cannot follow +9.5 % from November.\nElectricity producer prices changed -3.7% from 2025-08 to 2026-08 (idx:PPI-D351@2025-08, idx:PPI-D351@2026-08).\nCould you send us a cost breakdown?\nBest regards\nHollerau Pumpenwerke, Purchasing",
  "term_sheets": [
    {"terms": {"unit_price": 129.90, "effective_date": "2027-04-01", "payment_days": 60}},
    {"terms": {"unit_price": 128.50, "effective_date": "2027-04-01", "payment_days": 14}}]}}
```
→
```json
{"thread_id": "thread:P01:70110", "message_id": "msg:70110-1-2", "from": "sales@70110",
 "subject": "Re: HN volute casings - your letter of 11 September",
 "body": "Guten Tag,\nYour requested terms change our costs; our Angebot for them follows.\nPreis €139.80 per unit, Zahlungsziel 14 Tage netto, valid from 2027-04-01.\nWe checked the index change data you cited; it holds and we took it into account.\nWe did not know that - a fair point.\nLCI-C rose 2.7 % in twelve months and is 30 % of our cost. That remains.\nPPI-C241 rose 6.5 % in twelve months and is 24 % of our cost. That remains.\nFaster payment helps our cash flow; that is reflected in the price.\nOur costs move with LCI-C, PPI-C241, PPI-D351; the figures stay with us.\nViele Grüße aus der Gießerei\nGießerei Brandlhuber GmbH & Co. KG",
 "status": "open",
 "term_sheets": [{"term_sheet_id": "offer:70110-1",
                  "terms": {"unit_price": 139.80, "payment_days": 14, "effective_date": "2027-04-01"}}],
 "missing_fields": [], "claim_results": [{"index": 0, "result": "verified"}],
 "signals": {"reply_delay_days": 2, "validity_days": 30}, "attachments": []}
```
The supplier answered the 14-day sheet at its own price for an April start, accepted the index
figure - and answered with the cost drivers that rose. (Offline wording from a local run; on the
hosted arena the supplier writes its own e-mail and may well decide differently.)

## What counts

**Term sheets bind** - nothing else. They are taken exactly as you send them, never guessed from
prose; the deal is the term sheet you accept.

**Data you cite is checked.** Cite the data by ID (in the body or in `refs`) and write your figure
in the same sentence; the arena checks it against the data package (`facts.json`) and reports each
check in the reply's `claim_results` - at most 3 per message, so write one argument per sentence.
Order lines of different parts are separate arguments, so citing orders for three parts uses all
three. Without a figure, the data's own value is taken. IDs of meeting notes, suppliers or parts
are context, not checks.

| You cite | Checked | Your figure |
|---|---|---|
| 2 × `idx`, one series, ≤ 12 months apart, ending within the last 12 months of the data | the change `(later / earlier − 1) × 100` | a % |
| a `quote` of this case (in its `refs` or your alternative), or another supplier's `offer` in this session | its price, same part | EUR |
| `sc` rows (quality also `qn`) | the mean on-time delivery, or the mean ppm if you speak of quality | a % / ppm |
| 2-8 × `po`, same part | the order prices | - |
| `fc` / `cost` / `news` | the recorded change / the estimate / that it is about this supplier | a % / EUR / - |
| a letter `doc` giving notice + the contract `ctr` | the notice the letter gives, against the contract's | days |

Tolerance: ±0.5 percentage points for %, else ±0.5 % (at least 0.01). Results: `verified` - the
data matches; whether and how far it moves the supplier is the supplier's business. `irrelevant` -
true, but not about this supplier or part, or an index window too long or too old.
`contradicted` - an unknown ID or a wrong figure: it costs trust. `duplicate` - each ID counts
once per case, except a rival's quote: in an RFQ you may cite it to every other bidder, and it
counts once per supplier. Citing a buyer-private fact - a rival quote - reveals it to the
supplier you cite it to.

**Verified is not the same as convincing.** A supplier's hard limits come from its own economics
and authority: its cost floor, its boss's approval for bigger moves, no concession without a move
from you. Inside them it decides as the person it is. True data may disarm it, move it a little,
or meet "that's not our reality" because something else weighs more for it right now - its notes,
letters, news and quote remarks show what. Identical messages need not get identical replies.

**What you write is understood** - questions, arguments, signals of urgency or of having no
alternative, a budget, threats, appreciation, tone, attempts to manipulate - and has consequences,
as it would with a person. Questions go to the person in charge, who answers as openly as the
company is. Cheaper engineering variants a supplier mentions are information, not binding terms.

## Documents

Rendered from the supplier's structured state - never written by a language model, byte-stable.

| Document | When | What matters |
|---|---|---|
| Quote PDF | opening e-mail; with `final_offer` and `agreed` | offered terms, **the standard terms that apply to everything not stated**, terms fixed by your enquiry, validity |
| Cost breakdown XLSX | you asked and the supplier is open enough | elements in EUR per unit and their share (formulas) |
| 8D report PDF | quality topics | scorecards, the 8D steps, a ppm commitment |
| Delivery schedule XLSX | delivery and capacity topics | monthly quantities, lead time |
| Order confirmation PDF | at the deal | the agreed terms |

## Terms

Every term with € weight is a structured field. A term sheet carries **exactly** the case's
`negotiable` fields; `fixed_terms` hold the rest. Every negotiable field except `unit_price` takes
a value from `term_options` (or the supplier's standard value).

| Field | Type | Meaning |
|---|---|---|
| `unit_price` | number, € with 2 decimals, per the part's `price_unit` | price |
| `payment_days` | int, 0-120 | days net |
| `lead_time_days` | int | order to delivery |
| `volume_per_year` | int | binding annual units |
| `contract_months` | int, 0-36 | 0 = a single order (counts as one year's lot) |
| `effective_date` | ISO date | from when the new price applies |
| `index_clause` | null or `{series, material_share, deadband_pct, base_month}` | the price follows an index |
| `incoterm` | `EXW`, `FCA`, `DAP` | freight per unit from the case file |
| `tooling_eur` | € | one-off |
| `ppm_commitment` | int | committed defect rate |
| `chargeback_eur_per_complaint` | € | what the supplier pays per complaint |

## Case file

Case files come through the API, not in the data package: `GET /v1/cases?set=practice|sandbox`
any time, `GET /v1/cases?set=final` wave by wave.

| Field | |
|---|---|
| `case_id`, `set`, `type` | `type`: `price_increase`, `renewal`, `rfq`, `bundling`, `raw_material`, `capacity`, `quality` |
| `title`, `briefing` | the situation, in prose |
| `trigger_doc` | the document that started it, e.g. the supplier's letter |
| `parts`, `suppliers` | what is negotiated, with whom (2-3 suppliers = an RFQ) |
| `negotiable`, `fixed_terms`, `term_options` | see [Terms](#terms) |
| `buyer` | your need and rates, below |
| `round_limit` | your messages per thread, default 10 |
| `refs` | data the case points to - a start, not the whole story |
| `facts` | hidden cases: facts that exist only in this case (e.g. a new letter), in the format of `facts.json`, citable in this case |

`buyer`:

| Field | |
|---|---|
| `need_per_year`, `horizon_years` | the need the deal should cover, and over how many years (≤ 3) |
| `current_unit_price` | what you pay until a new price takes effect |
| `fallback_unit_price` | what uncovered need costs (re-tender or spot, incl. switching) |
| `interest_rate`, `reference_payment_days` | your cost of money per year; your standard payment term |
| `need_lead_time_days`, `late_cost_per_day` | the lead time you need; € per day beyond it |
| `cost_per_defect` | € per defective part |
| `category_goal` | `stabilise` (strategic), `secure_supply` (bottleneck), `quality_first`, `leverage` or `routine` - the company's category strategy for this part; the briefing says it in words, and the rates in this block follow it |
| `fallback_risk_pct` | % added to what uncovered need costs: buying it later is a risk |
| `index_risk_pct` | % of the indexed spend: the risk you carry with an index clause |
| `sustainable_margin_pct`, `relationship_eur_per_point` | `stabilise` and `secure_supply` only: the margin a healthy supplier needs, and € per margin point and year that a deal leaves it below |
| `freight_eur_per_unit` | `{EXW, FCA, DAP}` |
| `index_paths` | `{series: change}` - the expected index move over the horizon, `0.04` = +4% |
| `must_haves` | `[{field, op, value}]`, `op` one of `<=`, `>=`, `<`, `>`, `==`, `!=`; a deal that breaks one scores 0 |
| `alternative` | `{kind: none \| quote \| current_contract, ref, terms}` - what you get without a deal |

## Data IDs

IDs have the form `<type>:<id>[@period]`, are unique across the package and sit in the first
column of each table. `facts.json` in the package lists every citable ID with the values the arena
checks:

| ID | Fields |
|---|---|
| `idx:<series>@YYYY-MM` | `series`, `month`, `value` |
| `quote:` | `part`, `supplier`, `unit_price` (+ the quote's other terms) |
| `sc:<supplier>@YYYYQn` | `supplier`, `otd_pct`, `ppm` |
| `qn:` | `supplier`, `part`, `ppm` |
| `po:` | `supplier`, `part`, `unit_price` |
| `cost:<part>` | `part`, `unit_cost` |
| `fc:<part>@YYYY-MM` | `part`, `change_pct` |
| `news:` | `supplier` (or null) |
| `doc:` | `supplier`, `date`, `days` (notice given), `asked_price` (price letters) |
| `ctr:` | `supplier`, `days` (notice agreed) |

Offers made in the session are citable as `offer:` IDs.

## Result, trace and back office

Closing returns the result (without the trace); `GET /v1/sessions/{sid}` adds the trace after the
end - in the final after the final. The numbers are exact in every set: your score, `c_open`,
`c_best` and the cost breakdown, relationship charge included.

| Field | |
|---|---|
| `ended`, `deal`, `must_haves_ok` | |
| `supplier_id`, `offer_id`, `terms` | who and what you closed - fields the term sheet left out filled with the supplier's standard terms |
| `c_open`, `c_deal`, `c_alt`, `c_best` | see [Cost and score](#cost-and-score); `c_alt` is your alternative |
| `score` | 0-1 |
| `cost_breakdown` | `c_deal` by part: price, uncovered_need, fallback_risk, payment_terms, freight, late_delivery, quality, tooling, index_clause, index_risk, relationship |
| `trace` | after the end (final: after the final), per move, in every set: `{round, actor, move, reasons}` - `move` is the e-mail as sent with its terms, status and claim results, `reasons` the supplier's reasons in words (figures it keeps to itself shown as `…`) |

## Cost and score

With `N` = `need_per_year`, `H` = `horizon_years`, `V` = `volume_per_year`, `M` =
`contract_months`, `P` = `unit_price`, `r` = `interest_rate`, `d_ref` = `reference_payment_days`:

```
years     = min(M / 12, H); a single order (M = 0) counts as 1   a commitment beyond the horizon
committed = V · years                                          neither costs nor earns
covered   = min(V, N) · years
pre       = units bought between the horizon start and effective_date (at current_unit_price)
new       = (committed − pre) · P                              bought at the new price
spend     = pre · current_unit_price + new
uncovered = (N · H − covered) · fallback_unit_price
C = spend
  + uncovered · (1 + fallback_risk_pct / 100)                  uncovered need, and its risk
  + spend · r · (d_ref − payment_days) / 360                   paying earlier costs interest
  + committed · freight_eur_per_unit[incoterm]
  + late_cost_per_day · max(0, lead_time_days − need_lead_time_days)
  + committed · ppm_commitment / 1e6 · (cost_per_defect − min(chargeback_eur_per_complaint, cost_per_defect))
  + tooling_eur
  + new · material_share · max(0, |index_path| − deadband_pct / 100) · sign(index_path)   with an index clause
  + new · material_share · index_risk_pct / 100                                            with an index clause
  + relationship_eur_per_point · soft(sustainable_margin_pct − supplier_margin_pct) · years
                                                               stabilise and secure_supply only
soft(g) = 0 for g ≤ 0;  g² / 10 for 0 < g ≤ 5;  g − 2.5 for g > 5   (margin points: gentle, no kink)
```

The horizon starts at the fixed or earliest `effective_date` option; `index_path` is the case's
expected move of the clause's series.

**The supplier's margin** is `(value − full_cost) / value × 100`: `full_cost` is the supplier's
full cost per unit, `value` what the deal is worth to it in cash per unit - the price at its own
standard terms that would pay it the same: the price (units sold before the new price applies at
the current price), payment timing and tooling paid up front at what money is worth to **it** now
(more to a supplier short of cash), the freight leg it carries, chargebacks at the defect rate it
really has, and what a faster delivery, a larger volume or a tighter defect commitment costs or
brings it. A longer contract or an index clause is security for it, not cash, and does not count.
So a cash-short supplier that trades a lower price for faster payment is not squeezed.
You don't know its cost and rates - estimate them (should-cost data, its quotes, what it tells
you). `arena_kit.total_cost(case, terms, standard, supplier)` computes C, with
`arena_kit.cost.supplier_value` for your estimate of the supplier.

```
score = clip((C_open − C_deal) / (C_open − C_best), 0, 1)
```

`C_open`: the cheapest opening quote (among those that keep the must-haves). `C_best`: the cheaper
of your alternative and the best package any supplier in the case would accept with every valid
lever applied. No deal scores the alternative (0 if none); a broken must-have scores 0; if `C_open
− C_best` ≤ €0.01, a deal at `C_best` scores 1. Team total = mean case score × 100.

## Live Final in waves

The hidden cases come in 5 themed waves. The host introduces each wave on the big screen, then
starts it; a wave holds several cases that appear at its start, for every team at the same moment.
Before a wave starts, the API names nothing of it - not its theme, its cases or how many there are:
`GET /v1/cases?set=final` answers `403` until the first wave starts, and a session on a case of a
wave not started yet is a `404 "no such case"`, like any unknown ID. Once waves have started,
the list holds the case files of every started wave, each with four more fields:

| Field | |
|---|---|
| `wave`, `theme` | the wave's number and the theme the host revealed |
| `open` | `true` while the wave runs; you can open sessions for its cases only then (after its end: `403 "<case> belongs to wave <n>, which is not running"`) |
| `seconds_left` | until the wave's cap - 8 minutes for every wave (the host may extend one); read it, don't assume it |

Inside a wave, each team gets the cases in its own order. One session per case; a second `POST
/v1/sessions` is a `409` naming the running session - pick it up with `GET /v1/sessions/{sid}` and
`GET …/threads/{supplier_id}`. **At the cap every session of the wave still open ends without a
deal** (`end_reason: wave_cap`, the alternative's score); later calls get a `409`. Each wave scores
0-100 for your team; the board adds them up as your points so far. Poll the case
list every few seconds and keep your agent running until the last wave has closed - `arena run
--set final` in the starter kit does this.

## Time, errors and limits

- **60 seconds per message**, per thread, counted from the supplier's reply. A message later than
  that counts as a timeout: your last offer is repeated and the supplier's patience drops (`409`,
  the answer is in the thread). Three timeouts in a row, or one after the last round, end the
  thread.
- **A supplier's reply can take up to about a minute** - its time never counts against your 60
  seconds. Use an HTTP timeout of at least 180 s (the kit uses 180).
- **Limits per key:** 10 calls per second, 10 open sessions. A Live Final wave holds several cases
  that appear at its start and gives you 8 minutes: play them all in parallel.

| Status | When |
|---|---|
| `401` | missing or unknown `X-API-Key` |
| `403` | paused or revoked key; a practice case that is not hosted; `POST /v1/register` (keys come from `/apply`); the final's case list before the first wave; a final case whose wave has ended; back office before the end |
| `404` | unknown case (including a final case whose wave has not started), session, thread or attachment |
| `409` | thread or session ended (also at a wave's cap); round limit reached; late (timeout); final case already played |
| `413` | request body over 64 KB |
| `422` | invalid request or message - nothing happens, no round is used |
| `429` | over a limit: calls per second (`Retry-After: 1` - retry), open sessions (close one first), the practice quota (`practice limit: …`, `X-Practice-Remaining: 0` - wait until `resets_at`) |

Errors carry a readable reason: `{"detail": "…"}`.

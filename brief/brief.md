# Tacto Negotiation Arena - Participant brief

> **Build an AI buyer agent that negotiates by e-mail with tough, realistic industrial suppliers,
> gets the best total deal - and can explain how it got there.**

## 1. The story

You are the procurement team of **Hollerau Pumpenwerke GmbH**, a fictional German pump maker:
centrifugal pumps for buildings, water and industry, about €250M revenue, 900 people, one plant in
Bavaria. Your team buys the parts of the pump line - castings, machined parts, sheet metal, seals,
motors, fasteners, stainless bar - from about 45 suppliers. Today is **30 September 2026**, and the
inbox is full: a foundry announces +9.5% on volute casings "due to energy and wages", a framework
contract expires, a new pump series needs three quotes, a supplier with a red scorecard has to fix
its quality.

You answer the way buyers do: by e-mail. On the other side sit our **supplier bots**, each with a
small team of **roles** behind it - the sales rep or owner who writes to you, a boss who must
approve bigger moves, and engineering, quality, logistics, finance or legal, who answer their
topics and attach documents.

Tacto is the AI for industrial procurement: it gives procurement teams the intelligence to source
every part at the best possible terms with minimal effort. Your company runs on that kind of data -
orders, deliveries, invoices, contracts, scorecards, market indices, should-cost estimates, notes
from supplier calls. Your job is to build the agent that turns this data into better deals. (In
Tacto's product, agents such as the Defender Agent or the Sourcing Agent do parts of this job.)

## 2. The task

Each **case** is one negotiation: a trigger (a letter, an expiring contract, an RFQ), your case
file (need, goal, rates, must-haves, your alternative) and one to three supplier bots. Your
**buyer agent**

1. **prepares** - reads the case file and mines the data package;
2. **negotiates by e-mail** - plain-language messages with up to three **term sheets** (alternative
   packages), the data IDs your arguments rest on, and attachments if you like;
3. **closes** - accepts one supplier term sheet, or walks away.

**How a supplier decides.** Each supplier is a company with its own economics and a person at the
desk - a family foundry that remembers how you treated it, a group with an approval ladder, a job
shop that needs cash and work, a near-monopolist that barely moves. Its hard limits come from its
own situation: it never sells below its cost floor, bigger moves need the boss's approval, data
you cite is checked by ID, and it gives nothing without a move from you. Inside those limits it
decides as that person would - reading your whole e-mail: the term sheets, the data, and what you
signal (questions, arguments, threats, appreciation, urgency). It reacts to tone and relationship
as people do; padding does not buy price.

**Data is not automatically rewarded.** True data can disarm a supplier ("we didn't know that") -
or meet "that's not our reality", because something else weighs more for it right now. The traces
are in the data: call notes, letters, news, the remarks on its quotes. The skill is reading the
supplier, not stacking evidence - and finding the trade it wants: a supplier short of cash may care
more about when it is paid than about the price; one that cannot touch its list price may still
move on other terms.

## 3. What you get (at the kickoff)

| What | Where |
|---|---|
| This brief, the API docs (`api.md`) and the submission guide (`submission.md`) | `https://github.com/tacto-playground/negotiation-arena-kit`, folder `brief/` |
| **Data package v1**: CSV, the same data as SQLite, `facts.json` (every citable ID with the values the arena checks), contracts, letters, e-mails, meeting and call notes - the data behind all 19 practice cases | `https://arena.tacto.ai/v1/data/package.zip`, or the repo's `data/` |
| **Practice arena**: 10 hosted practice cases, picked so every case type shows up; a replay after every case; practice leaderboard; a **sandbox** with 3 demo suppliers | `https://arena.tacto.ai` with your team's key - apply at `https://arena.tacto.ai/apply` (below) |
| **Starter kit**: Python client, a naive agent, an LLM agent that writes e-mails and reads the attached documents, the Live Final runner, an MCP server over the data, `arena-sim` - the API on your laptop | the repo's `starter-kit/` - first agent on the board in 15 minutes |

You bring your own LLM access; we only run the suppliers. Data and docs are CC BY 4.0, code MIT.

**Your key - one per team.** Open `https://arena.tacto.ai/apply`, enter your team name and a contact e-mail,
and show the match code it displays at the Tacto desk. Once we approve, the key appears
on that page: copy it, press "Got the key" - it stays there. Only that page shows it; lost it - we reset it.
Tip: your team name may start with one emoji - it shows on the big screen (optional; nothing offensive).

**Practice quota: 600 practice sessions per team per rolling 24 hours.** `GET /v1/me` shows what
you have used and when it frees up. The build page `https://arena.tacto.ai/live` shows every team's quota use,
practice records and highlights from the kickoff to the code freeze.

**`arena-sim`** speaks the arena's API exactly, with one plain scripted supplier: get your plumbing
right there. It teaches the contract, not the suppliers - and we don't hand out our suppliers. Want
a local opponent? Build your own supplier simulator from the public data: thinking into how real
suppliers act is part of the challenge.

**The learning loop:** play a practice case, read your exact score and its cost breakdown, open
the replay - every e-mail and the supplier's reasons in words, as a real supplier would give them -
improve your agent, play again. Your best score per practice case counts.

## 4. The data

The buyer's world: supplier and part master data; three years of orders, goods receipts,
invoices and payments; quality notifications and quarterly scorecards (on-time delivery, PPM);
framework contracts and clauses; past RFQs and quotes; forecasts; supplier intel and news; past
negotiations, letters, e-mails, meeting and call notes; should-cost estimates; spend down to the
long tail; and monthly market indices (real public series, frozen at 2026-09-30).

- **Every fact you can cite has an ID** `<type>:<id>[@period]`, e.g. `po:4500010001-10`,
  `idx:PPI-C2451@2026-08`, `sc:70110@2026Q3`, `ctr:4600070110#3.1`, `doc:…`.
- What moves a supplier is in the data - some of it only in the notes and e-mails. Some notes are
  just noise. The company and its suppliers are fictional.
- **Sunday 08:00: a second release** - market news, supplier letters and a memo from early October
  (`GET /v1/data` lists it). Hidden cases may cite its facts. Your agent should read it.

## 5. How a negotiation works

- **Round 0:** every supplier sends its quote e-mail at once, with a quote PDF.
- **Your message:** subject, body, up to 3 term sheets (each with a value for every negotiable
  field), the data IDs you rely on (IDs written in the body count too), optional text attachments.
- **The supplier answers by e-mail:** the role in charge writes; its standing term sheet; a status -
  `open`, `final_offer` (push further and it may walk), `agreed` (it offers exactly your terms; you
  still accept), `walked_away`, `no_bid`; signals (reply delay, quote validity); fields it left out;
  and how each data argument checked out. A supplier's reply can take **up to about a minute**;
  its time never counts against your 60 seconds.
- **Only you close a deal**, by accepting a supplier's standing term sheet. In **RFQ cases** 2-3
  suppliers run in parallel threads; accepting one ends the case.
- **Limits:** a round limit per thread (default 10 messages) and **60 seconds per message**, counted
  from the supplier's reply. A late message counts as
  a timeout: your last offer repeated, the supplier's patience drops; three in a row, or one after
  the last round, end the thread.
- **Unstated terms default to the supplier's standard terms** - listed in its quote PDF.

**Documents** are rendered from the supplier's data, never invented: the quote PDF (read it before
you accept), a cost breakdown XLSX (ask; an open supplier sends one), an 8D report (quality), a
delivery schedule XLSX (logistics), and the order confirmation at the deal.

**What counts:** **term sheets bind** - nothing else. **Data you cite is checked**: the reply says
`verified`, `contradicted` (a wrong figure or an unknown ID - it costs trust), `irrelevant` or
`duplicate`; whether a true argument moves the supplier, and how far, is the supplier's business.
**What you write is understood** - and has consequences, as it would with a person.

## 6. The score: total cost in €

Only the accepted term sheet scores. The case values it as the **total cost C** of covering your
**fixed need over a fixed horizon**, at the case's published rates. Every case states its
**category goal** - `stabilise`, `secure_supply`, `quality_first`, `leverage` or `routine` - and its
rates follow it. Reaching the goal is the skill.

| Part of C | How |
|---|---|
| Price | committed units × unit price; units bought before a new price takes effect at the current price |
| Uncovered need | need the deal does not cover at the fallback price, plus the goal's fallback risk on it |
| Payment, freight, lead time | interest for paying earlier than your reference; freight by Incoterm; € per day beyond your need |
| Quality, other | expected defects at the committed PPM minus chargebacks; tooling; an index clause on the case's index path, plus the goal's index risk |
| The supplier's health | strategic and bottleneck goals only: € for every margin point the deal leaves the supplier below a sustainable margin (its margin on the cash it gets) |

**Case score = (C_open − C_deal) / (C_open − C_best)**, clipped to 0…1. **C_open** is the best
opening quote. **C_best** is the cheaper of the best package any supplier in the case would ever
accept, with every valid lever applied, and your alternative - fixed per case, so good arguments
raise your score instead of moving the yardstick. **No deal** scores your alternative (0 if none);
some cases are best left without a deal. **A broken must-have** scores 0. **In the final** each
wave scores 0-100 (the mean of its case scores × 100, one decimal) and your **points so far** are the
sum of your wave scores - up to 500 after five waves. The formula is in `api.md`; `arena_kit.total_cost` computes C.

**Example** (a `leverage` case: 3 % fallback risk) - need 4,000 a year for 2 years, interest 8%,
reference 60 days. The foundry asks €43.20 for a 24-month contract at 30 days net.

| | Terms | C |
|---|---|---|
| C_open | €43.20, 30 days: 8,000 × 43.20 = 345,600, plus 30 days of early payment at 8% = 2,304 | €347,904 |
| Your deal | €41.60, 60 days: 8,000 × 41.60 = 332,800 | €332,800 |
| C_best (hidden) | €40.80, 90 days: 326,400, minus 30 days of interest saved = 2,176 | €324,224 |
| **Score** | (347,904 − 332,800) / (347,904 − 324,224) | **0.64** |

The same €41.60 for 12 months covers half your need; the rest counts at the fallback price, say
€46.00, plus 3 %: C = €355,920 - worse than the opening, score 0.

## 7. Rules

1. **Not allowed:** humans in the loop during the Live Final, attacks on the service or on other
   teams, more than one API key per team - and agents that read the live feed during the Live
   Final: the event feed, the screen and the leaderboard (`/v1/events`, `/screen`, `/leaderboard`
   and their JSON).
2. **Everything else counts** - including weaknesses you find in our bots. Replays show how every
   score came about, and the jury may weigh that in the pitches. The goal is to convince a
   realistic supplier: that is what carries over to real negotiations.
3. **One attempt per case** in the final. An abandoned case is no deal.
4. **Code freeze at Sun 13:00.** Submit on the organisers' platform: your repo (the agent and a README
   to run it), the commit hash and a short description (`submission.md`). The final runs that commit
   from 13:30, so your agent must be running and polling before then. We check the top 5 for later
   pushes. Run your agent where you like, with your own LLM keys.
5. A supplier remembers only within a case; its history with your company is in the data.

## 8. The weekend

| When | What |
|---|---|
| Sat 10:30 | Kickoff at the Tacto office: brief, data, kit, practice arena live |
| Sat 12:00 | Hacking starts at MUCL; practice all weekend, overnight included |
| Sat from 14:00 | Office hours at MUCL: the Tacto team answers your questions |
| Sun 08:00 | **Sunday release:** new market news and letters (`/v1/data`) |
| Sun morning | **Pre-flight:** one practice case through your final setup |
| Sun 13:00 | **Code freeze** |
| Sun 13:30-14:30 | **Live Final** in 5 waves (MUCL) |
| Sun 15:10-16:10 | Top 5 pitch to the jury at the Tacto office (5 min pitch + 2 min questions) |
| Sun 16:20 | Awards |

The clocks go back in the night to Sunday (03:00 becomes 02:00): every Sunday time is the new time (CET).

**The Live Final.** The hidden cases come in **5 themed waves**. The host shows each wave's
theme on the big screen, then starts the wave; its cases appear at the start, for all teams at once,
each in its own order, one session per case. **Each wave has a hard cap of 8 minutes:** threads
still open at the cap end with no deal (your alternative's score). Start your agent before the
first wave and keep it running: it plays all of a wave's cases **in parallel** (up to 10 open
sessions per key) - the kit's `arena run --set final` does that. The leaderboard shows your points
so far, wave by wave; replays and the suppliers' back office appear after the final. A case broken on our side is replayed.

The hidden cases share the world, case types and rules of the practice set, with new suppliers and
settings. **None of them repeats a practice case** - mostly suppliers, parts and situations practice never
shows. No new rules, but an agent that memorised the practice suppliers will meet strangers. Some cite
the Sunday release.

## 9. Selection and prizes

- **Top 5:** the 5 teams with the most points after the five waves pitch to the jury at 15:10 at the
  Tacto office: 5 minutes + 2 minutes of questions. Every replay is open to the jury.
- **Places 1-3:** the jury sets them among the top 5, from the Live Final score and the pitch.
- **Ties:** points are points; an exact tie goes to more deals, then fewer rounds. At the top-5
  cut the jury looks at the score distribution. **Prizes:** robots, caps and a small extra, handed
  over at the awards at 16:20.

## 10. Tips

- **Numbers come from code, the model writes.** Compute C for every term sheet; put figures from
  the data next to their IDs; let the LLM choose arguments and wording.
- **Read the supplier** before you argue: its notes, letters, news and quote remarks tell you what
  it is dealing with - and what will move it.
- **Read the quote PDF** before you accept: what you did not negotiate closes on its standard terms.
- **Ask, and offer alternatives.** Two or three term sheets show you what a supplier values.
- **Don't leak** urgency, a budget or a missing alternative. Don't take the first offer.
- **Plan your practice runs** (600 a day is for learning, not brute force) and **rehearse the
  final**: all hosted cases in parallel, from Sunday's machine and network, well inside 8 minutes.

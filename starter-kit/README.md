# Tacto Negotiation Arena - starter kit

A Python client for the arena API, two example buyer agents that negotiate by e-mail, helpers that
read the suppliers' documents and price a term sheet, a runner that plays many cases in parallel,
`arena-sim` - a local simulator of the arena's API - and a read-only MCP server over the data
package. Read the brief first
(`../brief/brief.md`); the full API is in `../brief/api.md`. Code: MIT.

## Quickstart - first agent on the board in 15 minutes

You need Python 3.12 and [uv](https://docs.astral.sh/uv/).

**1. Get your team's key** (one per team): open `https://arena.tacto.ai/apply`, enter your team name and a
contact e-mail, and show the match code it displays at the Tacto desk. Once an organiser approves,
the key appears on that page - copy it and press "Got the key"; it stays there. Only that page (or its
link, in the same browser) shows it; lost it → ask the organisers to reset it. Keep it out of git.

```bash
cd starter-kit
uv sync --all-extras            # 2. install
cp .env.example .env            # 3. put ARENA_URL and your key (ARENA_API_KEY) into .env
uv run arena me                 # 4. your team and its practice quota
uv run arena cases              # 5. the practice cases (the 10 hosted ones are playable)
uv run arena play case:P01 -v   # 6. the naive agent negotiates one case
uv run arena board              # 7. you are on the practice leaderboard
```

Is the arena up? `curl $ARENA_URL/health`.

Step 6 prints the deal, your total cost `c_deal` against `c_open` and `c_best`, the case score,
and a **replay link**: every e-mail and the supplier's reasons for each move, in words. That is
the learning loop - play a practice case, read the replay, improve your agent, play
again. In practice your best score per case counts, so play a case again whenever your agent improves.
**Quota:** 600 practice sessions per team per rolling 24 hours. `arena play` and `arena run` print
how many are left; `arena run` plays only the hosted practice cases and stops starting new ones
once the quota is used up (a `429` there does not go away by retrying). Plan your runs.

**Now the LLM agent.** Download the data (`ARENA_DATA=data/package` in `.env`), choose a model in
`.env` (your own key), then:

```bash
uv run arena data                                          # unpacks data/package (and the Sunday release once public)
uv run arena play case:P01 --agent llm -v
uv run arena run --set practice --agent llm --parallel 8   # every hosted practice case, 8 at a time
uv run arena play case:D02 -v                              # the sandbox: 3 demo suppliers, no board
```

**Sunday 08:00:** run `arena data` again - it adds `data/release-sunday` (new news, letters, a
memo). `DataPackage` merges its facts and documents automatically; hidden cases may cite them.

## Hello World (local, 3 commands)

No key yet? Play a whole case against the local simulator - no network, no quota, no LLM costs.
The simulator's supplier is a plain script; the real ones are on arena.tacto.ai. After
`uv sync --all-extras` (step 2 above):

```bash
# terminal 1: the simulator (--data is optional: leave it out until you have run `arena data`)
uv run arena-sim serve --no-clock --data data/package
# terminal 2: a simulator key, then the naive agent plays case:S01
ARENA_URL=http://127.0.0.1:8000 uv run arena register "Hello World" sim                  # prints an api_key
ARENA_URL=http://127.0.0.1:8000 ARENA_API_KEY=<the api_key> uv run arena play case:S01 -v
```

It prints the deal, `c_deal` against `c_open` and `c_best`, the score and a replay link - open it
in your browser: the result on top, then every e-mail of the negotiation.

## Try it by hand (Postman, curl)

Feel the API before you put a model in front of it. On the real arena (practice and sandbox) the
60-second rule applies: three late replies in a row end the thread - too fast for typing. **By
hand, use the simulator first**, without the clock:

```bash
uv run arena-sim serve --no-clock --data data/package           # data/package from `arena data`; --data is optional
ARENA_URL=http://127.0.0.1:8000 uv run arena register "Team Name" sim   # prints a simulator api_key
export SIM=http://127.0.0.1:8000 KEY=<the api_key>
```

Every call sends `X-API-Key: <key>`; every `POST` also `Content-Type: application/json` and a
fresh `X-Move-Id` (any unique string - the same id again returns the same answer). In Postman:
those headers, the body as raw JSON.

```bash
# 1. the cases: case:S01 has one supplier, sup:70270, and the negotiable fields unit_price, payment_days
curl -s "$SIM/v1/cases?set=practice" -H "X-API-Key: $KEY"

# 2. open a session: the reply carries session_id (sim1 on a fresh simulator) and each supplier's quote e-mail
curl -s -X POST "$SIM/v1/sessions" -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
  -H "X-Move-Id: open-1" -d '{"case_id": "case:S01"}'

# 3. the supplier's opening e-mail: its term sheet (offer:70270-0) and the quote PDF to fetch
curl -s "$SIM/v1/sessions/sim1/threads/sup:70270" -H "X-API-Key: $KEY"

# 4. your e-mail with an offer: the reply is the supplier's answer (status, term_sheets, claim_results)
curl -s -X POST "$SIM/v1/sessions/sim1/threads/sup:70270/messages" -H "X-API-Key: $KEY" \
  -H "Content-Type: application/json" -H "X-Move-Id: msg-1" \
  -d '{"message": {"subject": "Adapter piece HI", "body": "Thank you for your quote. We offer 15.50 EUR at 60 days.",
       "term_sheets": [{"terms": {"unit_price": 15.5, "payment_days": 60}}]}}'

# 5. read the reply in the thread, and the session state
curl -s "$SIM/v1/sessions/sim1/threads/sup:70270" -H "X-API-Key: $KEY"
curl -s "$SIM/v1/sessions/sim1" -H "X-API-Key: $KEY"
```

To close, send `{"message": {"action": {"type": "accept", "term_sheet_id": "offer:70270-1"}}}` (the
term sheet you take) or `{"message": {"action": {"type": "walk_away"}}}` to the same `messages` path;
the answer carries the result and score. The quote PDF: `GET $SIM/v1/sessions/sim1/attachments/att:70270-0-quote`.
The same calls work on the arena (`https://arena.tacto.ai`, your team's key, a practice case) - with
the clock running.

## The Live Final: waves, in parallel, within the cap

The final comes in 5 themed waves. A wave holds several hidden cases that appear when the host
starts it; play them all in parallel (up to 10 open sessions per key). Each wave has a hard
cap - **8 minutes** (the host may extend one) - and what is still open then ends without a deal.
Start the runner before the first wave and leave it running:

```bash
uv run arena run --set final --agent llm --parallel 10
```

It polls `GET /v1/cases?set=final`, plays every case of a new wave at once (one agent per case),
closes each case **90 seconds before its wave's cap** (read from the wave's `seconds_left`, so an
extension counts) - earlier if a round in that case took longer
(`play.closing_move`: the cheapest standing term
sheet that keeps the must-haves and beats your alternative, else walk away) and stops after the
fifth wave has closed. If it crashes, start it again: it picks up the sessions it had started.
In Python: `play_waves(client, new_agent)`. It reads only your own cases and sessions - during
the final your agent must not read the leaderboard, the screen or the event feed (`arena board`
is for you, not for the agent).

A supplier's reply can take up to about a minute; its time never counts against your 60 seconds,
and the client waits up to 180 seconds for a reply. The client paces itself below the
limit of 10 calls per second and retries dropped connections with the same `X-Move-Id`, so a
retry never sends a message twice. Rehearse with `arena run --set practice --parallel 10` (it counts against your quota) before
Sunday, from the machine and network you will use: every case should end well inside 8 minutes.

### Rehearse the final

The final's case list answers `403` until wave 1, so rehearse its mechanics on the simulator - two
short waves of its demo cases (it prints the timeline on start):

```bash
uv run arena-sim serve --final-rehearsal --data data/package
# a second terminal:
ARENA_URL=http://127.0.0.1:8000 uv run arena register "Rehearsal" sim
ARENA_URL=http://127.0.0.1:8000 ARENA_API_KEY=<key> uv run arena run --set final --waves 2 --agent llm --parallel 10
```

Own client: point it at the same URL. It checks the `403` wait (60 s), the wave start, one session
per case, `seconds_left` and the cap (4 minutes, a 30-second break, then wave 2); `--waves 2` stops
the runner after the rehearsal's two waves (the final has five, the default).

## Your own agent

An agent is any object with `decide(case, threads) -> {"supplier_id": …, "message": {…}}`.
`threads` holds every e-mail so far per supplier, oldest first; supplier attachments come with
their text under `"text"`. The message is an e-mail - subject, body, up to 3 term sheets - or an
action that closes the case:

```python
from arena_kit import ArenaClient, RemoteTable, complete_offer, play, total_cost
from arena_kit.docs import standard_terms
from arena_kit.play import latest, live_threads, standing


class Haggler:
    def decide(self, case, threads):
        sid = live_threads(threads)[0]
        events = threads[sid]
        sheet_id, terms = standing(events)  # the supplier's standing term sheet
        std = standard_terms(events)  # from its quote PDF: what closes if a field is missing
        if latest(events)["status"] in ("agreed", "final_offer"):
            return {
                "supplier_id": sid,
                "message": {"action": {"type": "accept", "term_sheet_id": sheet_id}},
            }
        ask = complete_offer(case, {**terms, "unit_price": terms["unit_price"] * 0.97})
        print("their C:", total_cost(case, terms, std), "ours:", total_cost(case, ask))
        return {
            "supplier_id": sid,
            "message": {
                "subject": case["title"],
                "body": "Thank you for your quote. Could you send us a cost breakdown?",
                "term_sheets": [{"terms": ask}],
            },
        }


print(play(RemoteTable(ArenaClient()), "case:P01", Haggler()))
```

| Module | What |
|---|---|
| `client.py` | `ArenaClient`: one method per endpoint (messages, threads, attachments, back office, `me`, ...); paced, safe retries; `practice_remaining` |
| `play.py` | `play()`, `play_all()`, `play_waves()` (the Live Final); `RemoteTable` - the hosted arena (practice, sandbox, final) |
| `cost.py` | the published cost function C with the category goal's rates, `supplier_value` for the relationship term, must-haves, term-sheet completion |
| `docs.py` | supplier documents to text (PDF, XLSX); a quote PDF to its offered, standard and fixed terms |
| `data.py` | the data package and the Sunday release: `facts.json`, read-only SQL, lookup by ID, documents, cases |
| `agents/naive.py` | anchors 10 % low, splits the difference, accepts early - the baseline |
| `agents/llm.py` | the model writes the e-mail and the term sheets from the case and the data it points to; code computes C, reads the documents, removes IDs that do not exist, and refuses bad accepts |
| `sim.py` | `arena-sim`: the public API on your machine, with one plain scripted supplier (below) |
| `mcp_server.py` | the MCP server below |

The LLM agent is a starting point, not a winner: it sees only what the case file points to.
Finding the data that moves a supplier is your job - the worked example below shows one way for
one kind of argument. Ideas beyond it: mine the notes and e-mails, read the suppliers' signals and
documents, send two or three term sheets to learn what a supplier values, ask its roles the right
questions, keep RFQ threads alive in parallel, and never leak urgency, a budget or a missing
alternative.

**Strategic and bottleneck cases** (`category_goal` `stabilise` or `secure_supply`) charge C for a
deal that leaves the supplier below a sustainable margin. Pass your estimate of the supplier to
price it - without one, `total_cost` leaves the term out:

```python
supplier = {"full_cost": 128.0, "interest_rate": 0.12, "standard": std}  # your estimate
total_cost(case, terms, std, supplier)
```

## Worked example: cite an index (case P01)

Brandlhuber, a foundry, announces +9.5 % on volute casings and blames energy and wages. Does the
data agree? `DataPackage` reads the package (the MCP server below gives an LLM the same access):

```python
from arena_kit.data import DataPackage

pkg = DataPackage("data/package")
# 1. What drives this part's cost? The category team's estimate:
pkg.query("SELECT series_id, weight FROM cost_drivers WHERE part_id = 'part:310010'")
# -> PPI-C241 (iron and steel) 0.26, PPI-D351 (electric power) 0.06, LCI-C (labour) 0.32
# 2. The letter blames energy: read the electricity producer price index for the last year.
rows = pkg.query(
    "SELECT id, month, value FROM index_values WHERE series_id = 'PPI-D351' AND month >= '2025-08' ORDER BY month"
)
# 3. Compute the change yourself - two months of one series, at most 12 months apart,
#    ending within the last 12 months of the data.
a, b = rows[0], rows[-1]
change = (b["value"] / a["value"] - 1) * 100
# 4. One sentence: the figure next to both IDs.
print(f"Electricity producer prices changed {change:+.1f}% from {a['month']} to {b['month']} ({a['id']}, {b['id']}).")
# -> Electricity producer prices changed -3.7% from 2025-08 to 2026-08
#    (idx:PPI-D351@2025-08, idx:PPI-D351@2026-08).
```

Put that sentence into your e-mail and read `claim_results` in the reply: `verified` (the figure
matches the data), `irrelevant` (true, but not about this supplier's costs, or the window is too
old) or `contradicted` (a wrong figure - it costs trust). Whether it moves the price, and how far,
is the supplier's business - a true figure may disarm one supplier and leave the next cold; the
replay shows what it made of it. Which series matter to which
supplier, which window, and what else in the data speaks for you - that is up to your agent.

## Choosing a model

`LLM_PROVIDER=anthropic` uses the Anthropic SDK (`ANTHROPIC_API_KEY`); `LLM_PROVIDER=openai` talks
to any OpenAI-compatible endpoint (`LLM_BASE_URL`, `LLM_API_KEY`) - OpenAI, Mistral, Groq, a local
Ollama (`http://localhost:11434/v1`) and many more. Or pass your own `complete(system, user) -> str`
to `LLMAgent`. Each message must arrive within **60 seconds** of the supplier's reply: prefer a
fast model or low effort, and do heavy research before you open a session. If the model fails or times out, the agent falls
back to the naive move instead of losing the case.

## Offline: arena-sim, the contract simulator

```bash
uv run arena-sim serve --data /path/to/data/package   # http://127.0.0.1:8000, event code "sim"
ARENA_URL=http://127.0.0.1:8000 uv run arena register "Team Name" sim   # a simulator key
```

`arena register` exists for the simulator only: it hands out its own keys, which work nowhere
else. The arena's key comes from `/apply` (step 1).

`arena-sim` speaks the arena's public API exactly - paths, fields, statuses, error codes, limits,
the 60-second rule, quote PDFs and cost-breakdown XLSX files in the arena's format, RFQs with
parallel threads, leaderboard and replay JSON - with three example cases (`case:S01`-`S03`). Point
your agent at it to get the plumbing right without a network or a rate limit to worry about.
`--no-clock` switches the 60-second rule off for trying it by hand (above); without it the clock
runs, as on the arena. The first line it prints says which.

Its supplier is a deliberately plain script and **not** how the arena's suppliers behave: it opens
10 % above your current price, concedes a fixed 2 % per term sheet down to 90 %, agrees when you
meet its price, sends a final offer from round 5 and walks if you push after that; cited data is
only checked for existence. **The simulator teaches the contract; the real suppliers think for
themselves** - practise against them in the hosted arena and its sandbox (`--set sandbox`). We
don't hand out our suppliers: for a local opponent, build your own supplier simulator from the
public data - how would a foundry owner with a full order book answer your e-mail?
`starter-kit/Dockerfile` runs the simulator as a container. `uv run pytest` runs the kit's contract
tests against it; set `ARENA_TEST_URL` and `ARENA_TEST_KEY` to run them against a hosted arena too.

## MCP server over the data

A local, read-only MCP server (stdio) lets an MCP-capable agent or chat client explore the data
package itself. Six tools: `list_tables`, `query` (SELECT only, max 200 rows), `lookup` (the fact
behind a citable ID), `list_ids` (e.g. every month of one index series), `read_document`,
`get_case`.

```bash
uv run arena mcp --data /path/to/data/package   # speaks MCP on stdin/stdout
```

Add it to a client, e.g. Claude Code:

```bash
claude mcp add arena-data -- uv run --directory /path/to/starter-kit arena mcp --data /path/to/data/package
```

or any client with an `mcpServers` config:

```json
{"mcpServers": {"arena-data": {"command": "uv", "args": ["run", "--directory", "/path/to/starter-kit",
  "arena", "mcp", "--data", "/path/to/data/package"]}}}
```

## Good to know

- **60 s per message**, per thread. A late message counts as a timeout (your last offer repeated);
  `play()` reads the supplier's answer and goes on.
- **Invalid messages cost nothing**: `422` and no round used. `409` means the thread or case has
  ended, or you were late.
- **Your key is your team's.** Keep `.env` out of git.
- Tests: `uv run pytest`. Lint: `uv run ruff check .`

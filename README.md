<p align="center">
  <img src=".github/assets/readme-banner.png" alt="Tacto Negotiation Arena. Your agent. Our suppliers. Real money." width="100%">
</p>

<p align="center">
  <a href="https://arena.tacto.ai/apply"><b>Apply for your key</b></a> ·
  <a href="brief/brief.md">Brief</a> ·
  <a href="brief/api.md">API docs</a> ·
  <a href="starter-kit/">Starter kit</a> ·
  <a href="https://arena.tacto.ai/live">Build page</a> ·
  <a href="https://arena.tacto.ai/results">Results</a>
</p>

You are the procurement team of Hollerau Pumpenwerke, a fictional German pump maker, and the inbox is
full: a foundry wants +9.5 %, a framework contract expires, a new pump needs three quotes. Build an
**AI buyer agent** that reads the company's data, negotiates by e-mail with our supplier bots, and
closes the best total deal, or walks away. Practise all weekend against our hosted suppliers. On
Sunday a Live Final on hidden cases ranks every team.

This repo holds everything you need: the brief, the API docs, a Python starter kit and the data
package. Bring your own LLM, any provider; we run the suppliers.

## Start in 15 minutes

You need Python 3.12, [uv](https://docs.astral.sh/uv/) and git.

1. **Apply.** Open **https://arena.tacto.ai/apply** and enter your team name and a contact e-mail.
   Tip: start the name with one emoji, it shows on the big screen.
2. **Show your match code at the Tacto desk.** We approve.
3. **Copy your key.** It appears on the page you applied on and stays there; press "Got the key". One key
   per team. Lost it? We reset it at the desk.
4. **Run the four commands.**

```bash
git clone https://github.com/tacto-playground/negotiation-arena-kit.git
cd negotiation-arena-kit/starter-kit
cp .env.example .env            # paste your key: ARENA_API_KEY=...

uv sync --all-extras            # install
uv run arena me                 # your team and its practice quota
uv run arena play case:P01      # the naive example agent negotiates one practice case
uv run arena board              # your team on the practice leaderboard
```

`arena play` prints the deal, your score and a replay link: every e-mail and the supplier's reasons
in words. Read it, improve your agent, play again. Next steps, the LLM agent and your own agent:
[starter-kit/README.md](starter-kit/README.md).

## Links

| What | Where |
|---|---|
| Apply for your key | https://arena.tacto.ai/apply |
| Brief: story, task, score, rules | [brief/brief.md](brief/brief.md) |
| API docs | [brief/api.md](brief/api.md) |
| Starter kit: client, example agents, Live Final runner, local simulator, MCP server | [starter-kit/](starter-kit/) |
| Submission and pitch | [brief/submission.md](brief/submission.md) |
| Practice build page: every team's quota, records and highlights | https://arena.tacto.ai/live |
| Live Final results, team by team and case by case | https://arena.tacto.ai/results |
| Questions | The Tacto team at MUCL, office hours from Sat 14:00 · [FAQ](https://arena.tacto.ai/faq) |

## The weekend at a glance

| When | What |
|---|---|
| Sat 10:30 | Kickoff at the Tacto office: brief, data, kit and practice arena go live |
| Sat 12:00 | Hacking starts at MUCL. Practice all weekend, overnight included: 600 sessions per team per 24 hours |
| Sat from 14:00 | Office hours at MUCL |
| Sun 08:00 | **New market data.** Run `uv run arena data` again; hidden cases may cite it |
| Sun morning | Pre-flight: one practice case through your final setup |
| Sun 13:00 | **Code freeze**: submit on the organisers' platform |
| Sun 13:30 | **Live Final**: five waves, 8 minutes each, every team at once |
| Sun 15:10 | **Top 5 pitch** to the jury at the Tacto office: 5 minutes + 2 minutes of questions |
| Sun 16:20 | Awards |

The clocks go back in the night to Sunday (03:00 becomes 02:00): every Sunday time is the new time.

## Rules that matter

- **No human in the loop** during the Live Final. Start your agent before it starts and leave it running:
  `uv run arena run --set final --agent llm --parallel 10` picks up each wave by itself.
- **Your agent must not read the live feed** during the Live Final: `/v1/events`, `/screen`,
  `/leaderboard`, `/results` and their JSON. They are for you, not for your agent.
- **One key per team.** No attacks on the service or on other teams.
- Everything else counts, including weaknesses you find in our suppliers. Every replay is open to
  the jury.

All rules: [brief, section 7](brief/brief.md#7-rules).

## What to submit by the code freeze (Sun 13:00)

On the organisers' platform:

- **Your repo** with the agent and a README that says how to run it, and the **commit hash** your
  agent runs in the final. The jury must be able to open it.
- **A short description** of your approach.
- **Slides** for the pitch, if you make the top 5: they pitch at 15:10.

Details: [brief/submission.md](brief/submission.md).

## Prizes

Robots, caps and a small extra. The Live Final decides who pitches (the top 5); the jury sets places 1-3
from the final score and the pitch.

## What is in this repo

| Folder | What |
|---|---|
| [`brief/`](brief/) | The brief, the API docs, the submission guide |
| [`starter-kit/`](starter-kit/) | Python client, a naive and an LLM example agent, the Live Final runner, `arena-sim` (the API on your laptop), an MCP server over the data |
| [`data/`](data/) | Data package v1: CSV, SQLite, `facts.json`, contracts, letters, notes. The same as `https://arena.tacto.ai/v1/data/package.zip`; start with [`data/README.md`](data/README.md) |

The company, its suppliers and all names are fictional; the market indices are real open series
([`data/ATTRIBUTION.md`](data/ATTRIBUTION.md)).

## Licence

Code: MIT ([LICENSE](LICENSE)). Data and docs: CC BY 4.0 ([LICENSE-data](LICENSE-data)). Your code
is yours.

<br>
<p align="center"><sub>Tacto · AI for industrial procurement · Industrial Agents Hackathon 2026, Munich</sub></p>

# Limits and honest limitations

Current as of 31 Oct 2026. The live numbers are on https://localedition.pages.dev/limits and `/proof` (and as JSON at `/api/limits` and `/api/proof`).

## What is live

| Part | State |
|---|---|
| Qloo | Live in production since **30 Oct 2026, 23:33 MYT**. Every Qloo number on the site comes from a live call or a saved live answer. Before that the app ran on synthetic fixtures (`fixtures/qloo/`), which production can no longer reach. |
| Nemotron on Token Factory | Live: Super VL (agent, menu photos), Ultra (review), Lightning (copy, guardrail check); Qwen3 Embedding 8B for matching. |
| Tavily | Live for the demo workspace's sample stores and chain setup; everything else reads the week's saved events. |
| Weekly build | Cloudflare cron every Monday 00:00 UTC (08:00 MYT) rebuilds the demo workspace. The current editions come from the live run of 30 Oct, 23:38 MYT. |
| NeMo Guardrails library | Runs in a Nebius job started from the CLI, not on a schedule. Last eval: 40 of 40 red-team phrases caught, 0 of 10 benign blocked. Every item also gets the live check in the Worker. |
| Calendar feed | Live: a private `.ics` link per store. |
| Square, Slack, Gmail/Outlook | Built and unit tested against the providers' documented APIs, with provider calls faked. **Not connected live**: there are no Square or Slack credentials on the deployed app and no consented mailbox. |

## Sparse weeks

The evidence check drops anything Qloo's data does not back, and the app never pads the gap. In the current week:

- **No store kept a sound brief.** A brief needs 8 artists backed by Qloo evidence; none passed.
- **Park Slope and West Village have no special.**
- The chain review sent Upper East Side and Long Island City back for specials whose copy was near-identical to other stores'.
- 45 items have been dropped for evidence so far.

`/limits` lists every gap per store ("Park Slope: this week's edition has no special; no candidate backed by Qloo evidence was kept.").

## Fallbacks, and how they are labelled

| When | What happens | Where it shows |
|---|---|---|
| The agent model stalls, errors or runs out of turns | A fixed tool order and a rule-based picker finish the store | The edition's evidence drawer: "Planned by: deterministic-plan" |
| The copy model fails | A plain template writes the item | The run log: "this copy uses the plain template" |
| A model is unavailable on Token Factory | That step, or the whole store, is skipped and builds on the next run | The run log and `/proof` |
| Ultra fails | Super VL reviews | The verdict's `reviewedBy` |
| The guardrail model check cannot run | The item is `unchecked` and needs an explicit override to approve | The item's guardrail status |
| Qloo cannot answer or a budget is spent | `qloo_unavailable` or `qloo_budget`, with a retry; no sample data stands in | The screen that asked |

## Budgets

| Budget | Limit | Used (31 Oct) |
|---|---|---|
| Qloo live calls | 2,000 through 15 Dec, 200 a day, reserved before each call, never refunded | 117 |
| Qloo brand searches | 3 live searches per workspace per day, on top of the ledger | |
| Qloo saved answers | 14 days in KV, real answers only | |
| Token Factory spend | $5 cap; $0.40 per visitor workspace; $0.80 per network a day; $1.50 kept for the demo workspace and the weekly build | $0.32 |
| Tavily | 80 credits through 15 Dec, 10 a day; results saved 7 days | 9 |
| New workspaces | 60 an hour across the app | |

Rate limits per workspace and per network (builds, regenerate, staff cards, edits, menu photos, brand search, brand-voice drafts) are on `/limits` and in `src/server/limits-table.ts`.

## Other limits

- **Kettle & Kiln is sample data**, labelled as such. It is not a customer. Its stores sit at real New York corners, so the Qloo answers about them are real.
- **The first live Tavily fetch of a week** can take longer than a tool's 25-second limit; the fetch still finishes and is saved, and later stores read it.
- **The live guardrail check** relies on a small model with thinking off for paraphrases the regex catalogue does not catch; it can miss some. The NeMo job's eval reports the catch rate.
- **Qloo's tag filter on places is narrow**: when it returns fewer than 3 places, the tool tops up from the unfiltered list and says so.
- **`rising`** is a tool the agent may call; not every edition uses it. Qloo's `/v2/trending` does not accept places, so place momentum is never claimed.
- **Heatmaps, demographics and audiences** are not used by any screen or step.

## Built during the submission period

The first commit is dated 29 Oct 2026. Everything in this repository was written for this project.

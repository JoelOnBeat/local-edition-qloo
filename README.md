# Local Edition

**Every store in a chain gets its own weekly edition, built by an agent from the Qloo taste of its block.**
Head office opens one map, reads a special, a sound brief, a local partner and an event night for each store, sees the Qloo evidence behind every line, and approves.

Live demo: **https://localedition.pages.dev** (press **Try the sample chain**, no account needed)

## Qloo Agentic Hackathon submission

| # | Requirement | Where |
|---|---|---|
| 1 | **Functional demo** | **https://localedition.pages.dev**, usable end to end with no account: press **Try the sample chain**. You get a private copy of Kettle & Kiln, a sample coffee chain with eight stores at real New York corners. Things to try: the store map; each store's edition with its **evidence drawer** (the Qloo endpoint, parameters and numbers behind every item); **Compare** two stores (Qloo's `/v2/analysis/compare` plus our tag-lift diff); approve items and open the phone-first **staff card**; **Build this week's editions** (within per-visitor budgets: the agent's tool calls, models and cost stream live, and **Runs** keeps the log); **Setup** (search Qloo for the brands your chain resembles); `/proof` and `/limits` (what ran, on which model, at what cost, every budget and every gap). |
| 2 | **Public code repository** | **https://github.com/JoelOnBeat/local-edition-qloo**: all source, the sample chain, migrations, scripts and instructions. Run it with [docs/SETUP.md](docs/SETUP.md); the env var names are in [.env.example](.env.example). |
| 3 | **What it does and what makes it Qloo-powered** | [docs/QLOO.md](docs/QLOO.md). In two lines: Qloo tells us how the taste around one street corner differs from the rest of the city (tags, places, artists, brands, each with an affinity score). The agent turns that difference into four concrete things a store can do this week, and drops anything Qloo's data does not back. |
| 4 | **External hosting** | Cloudflare Pages in front of a Cloudflare Worker, at **https://localedition.pages.dev**. Public, no login, VPN or allowlist needed. |
| 5 | **Open-source licence** | MIT, see [LICENSE](LICENSE). |

## Why it only works with Qloo

A chain's head office cannot write eight different weekly plans by hand, and a language model on its own writes the same "cosy autumn latte" for every street. What makes the Williamsburg store different from the Upper East Side store is **local taste**, and that is exactly what Qloo measures:

- **Where people's taste leans, by location.** Qloo's Insights API takes a point (`signal.location`) and returns the cuisine, ambience, occasion, dish and music tags that over-index there. We subtract the same tags' city-wide affinity, so each store sees what is *different* about its block, not what is popular everywhere.
- **Cross-domain.** The same location signal returns artists (the sound brief) and brands (pairing ideas), and a place query near the store returns partner candidates ranked by affinity to the brands the chain resembles. No other source connects coffee, music and neighbouring businesses through one taste graph.
- **Evidence you can check.** Every Qloo answer becomes an evidence row with its endpoint, parameters and scores. The agent can only cite what Qloo returned, and code recomputes the numbers before anything reaches head office.

Without Qloo there is no edition: an item the agent cannot back with Qloo evidence is dropped, never shown.

## The Qloo calls we make in production

Host `hackathon.api.qloo.com`, `GET` with the key in a header. Every call below is made live (or served from a saved live answer) in the deployed app; details in [docs/QLOO.md](docs/QLOO.md).

| Qloo feature | Endpoint and key parameters | What it decides |
|---|---|---|
| **Insights: tags by location** (block profile) | `/v2/insights` `filter.type=urn:tag`, `signal.location=POINT(lon lat)`, `filter.tag.types=<one of 7 tag families>`, `take=20` | The block's character: the up to 12 tags (cuisine, specialty dish, menu highlight, ambience, occasion, neighbourhood, music) that lean most above the city. Drives the special, the mood of the sound brief and the partner filter. |
| **Insights: city baseline** | `/v2/insights` `filter.type=urn:tag`, `signal.location.query=New York City`, same tag family, `take=50` | The reference each tag's **lift** is measured against (store affinity minus city affinity). |
| **Insights: places near the store** | `/v2/insights` `filter.type=urn:entity:place`, `filter.location=POINT`, `filter.location.radius=1200`, `signal.interests.entities=<chain's brands>`, `filter.tags=<block tags>`, `feature.explainability=true` | The **local partner**: real neighbouring places ranked by affinity to the brands the chain resembles, with distance and the brand they are "matched because of". |
| **Insights: artists by location** | `/v2/insights` `filter.type=urn:entity:artist`, `signal.location=POINT`, `signal.interests.entities`, `filter.popularity.min=0.2`, `filter.popularity.max=0.7`, `bias.trends=low` | The **sound brief**: artists with a local skew, not the top 40. It needs 8 artists or it is dropped. |
| **Insights: brands by location** | `/v2/insights` `filter.type=urn:entity:brand`, `signal.location=POINT`, `filter.tags=urn:tag:genre:brand:food_and_beverage`, `filter.popularity.max=0.8` | Food and drink brands that over-index on the block: pairing ideas for the special and the partner offer. |
| **Search** (taste anchors) | `/search` `query=<brand name>`, `types=urn:entity:brand` | Which real brands the chain resembles. Their entity ids become `signal.interests.entities` for places and artists. |
| **Analysis: compare** | `/v2/analysis/compare` `a.signal.interests.entities` / `b.signal.interests.entities` = each store's top 5 nearby places, `filter.type=urn:entity:place`, `model=descriptive` | The **Compare** screen: the taste two stores share, as evidence next to our own tag-lift diff. |

We do not use Qloo's heatmap, demographics or audiences endpoints, and we never send customer or personal data: only coordinates, entity ids, tag urns and the brand names typed into the setup search.

### Saved answers, the call ledger and failures

- **KV cache, 14 days, live answers only.** Each Qloo answer is saved in Cloudflare KV under a hash of path and sorted parameters, never the key. A saved answer costs no call and is labelled `cache` with the time Qloo gave it. Nothing recorded or synthetic is ever served in production.
- **Call ledger.** A Durable Object reserves one call *before* each live request: at most **2,000 live calls through 15 Dec and 200 a day** across all visitors. Reservations are never refunded and live calls are never retried. `/limits` shows the count (117 of 2,000 used at the time of writing).
- **When Qloo fails** (timeout, error, ledger spent) the call raises `qloo_unavailable` or `qloo_budget`, and the screen shows an error with a retry. No sample data stands in.

## How the agent works

1. **Tool use on NVIDIA Nemotron 3.5 Super VL.** For each store the model plans its own steps, up to 10 turns, calling tools: `block_profile`, `places_near`, `artists_local`, `brands_local`, `rising` (Qloo); `events_this_week` (Tavily); `menu_match` and `partners_rank` (embeddings over the chain's menu and the Qloo places). It finishes with `submit_candidates`, citing only the refs the tools returned.
2. **Evidence check.** Code, not a model, drops any item with no evidence, a sound brief with fewer than 8 artists, any item citing an entity Qloo never returned, and any claimed lift that does not recompute from the saved city baseline.
3. **Copy and guardrails.** Nemotron 3.5 Lightning writes the copy in the chain's voice. A guardrail check (NeMo Guardrails' rules, mirrored in the Worker, plus NeMo's `self check output` prompt on Lightning) blocks health, diet and allergen claims, identity targeting and prompt injection from web text. The NeMo Guardrails library itself certifies items and runs a 50-phrase red-team eval in a batch job (40 of 40 caught, 0 of 10 benign blocked).
4. **Chain review on Nemotron 3 Ultra.** Code finds clashes across stores (same partner, same event, near-identical copy); one Ultra call judges every edition against them and can send an item back with a hint.

More in [docs/AGENT.md](docs/AGENT.md).

## Supporting stack

- **NVIDIA Nemotron on Nebius Token Factory**: Nemotron 3.5 Super VL (agent, menu-photo reading), Nemotron 3 Ultra (reviewer), Nemotron 3.5 Lightning (copy, guardrail check); Qwen3 Embedding 8B for matching.
- **Cloudflare**: Pages front, Worker (Next.js via OpenNext), D1, KV, R2, Durable Objects (spend meter, build lock, Qloo and Tavily ledgers), Workflows (one retried step per store), a Monday cron.
- **Tavily**: this week's local events for the event night, and the chain's own site for a brand-voice draft.

Architecture: [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Run it

```sh
npm install
NEXT_PUBLIC_DATA_MODE=mock npm run dev   # UI only, typed mock data, no server calls
npm run dev                              # real routes; without keys every provider answers from synthetic fixtures
npm test                                 # vitest, offline (142 files, 1,623 tests)
```

Env vars are listed by name in [.env.example](.env.example). Full setup, sample-data seeding, D1 migrations and Cloudflare deploy: [docs/SETUP.md](docs/SETUP.md). Tests: [docs/TESTING.md](docs/TESTING.md).

## Limits and honest limitations

Current as of 31 Oct 2026. The full list is in [docs/LIMITS.md](docs/LIMITS.md).

- **Qloo has been live in production since 30 Oct, 23:33 MYT.** Before that, development ran on synthetic fixtures (`fixtures/qloo/`, used only by tests and local dev).
- **Sparse weeks.** In this week's live build no store kept a sound brief (no artist list passed the evidence check, which needs 8 artists backed by Qloo evidence), and Park Slope and West Village have no special. Nothing is padded: `/limits` lists every gap.
- **Kettle & Kiln is sample data**, labelled in the app. It is not a customer.
- **Connectors.** Square, Slack and Gmail/Outlook (via Composio) are built and unit tested against their documented APIs, but no live account is connected on the deployed app. The private calendar feed (`.ics` per store) works.
- **Fallbacks are labelled.** If the agent model fails mid-store, a fixed tool order runs instead and the edition says "Planned by deterministic-plan". If the copy model fails, a plain template stands in with a note. If a model is unavailable, that step or store is skipped with a note.
- **Budgets.** Token Factory spend is capped at $5 with per-visitor sub-budgets; Tavily at 80 credits through 15 Dec; rate limits per workspace and per network are on `/limits`.

Built during the submission period: first commit 29 Oct 2026.

## Licence

MIT, see [LICENSE](LICENSE). Copyright 2026 JoelOnBeat.

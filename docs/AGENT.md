# The edition agent

One store, one week: the agent gathers taste data with tools, picks one candidate per item, and hands them to code that checks the evidence, writes copy, applies guardrails and saves the edition. Then a reviewer reads the whole chain at once. Code: `src/server/agent/`.

```
for each store (one retried Workflow step each)
  Nemotron 3.5 Super VL tool loop   -> candidates (special, sound, partner, event), refs only
  evidence check + verifier (code)  -> drops anything Qloo does not back
  Nemotron 3.5 Lightning copy       -> text in the chain's voice
  guardrail check                   -> passed / rewritten / blocked / unchecked
  save edition (draft)
then, for the chain
  clash finder (code) + Nemotron 3 Ultra review -> approve / needs_change with a hint
```

## 1. Tool loop (Nemotron 3.5 Super VL)

`src/server/agent/loop.ts`. Model `nvidia/Nemotron-3.5-Super-VL` on Nebius Token Factory, OpenAI-compatible tool calling, temperature 0.3.

- **Prompt.** The chain's brand voice, the rules, the menu and the week go first (identical for every store, so the provider's prefix cache hits); the store brief (name, corner, city, the chain's anchor brands, the block's top tags once known) goes last.
- **Rules.** Use only refs the tools returned (`s1`, `a1`, `p1`, `e1`); never invent names or facts; taste and place only, no health, diet, nutrition or allergen claims, no targeting by age, gender, ethnicity or religion; say "rising in Qloo's population data", never "rising on this street".
- **Limits.** At most 10 model turns, 4 tool calls per turn, 45 s per model call, 25 s per tool call, 150 s per store, 3 invalid calls. Every argument is validated with zod.

| Tool | Source | Returns |
|---|---|---|
| `block_profile` | Qloo `/v2/insights` (tags by location and the city baseline) | up to 12 tags that over-index on the block, with lift |
| `places_near` | Qloo `/v2/insights` (places, `filter.location`, anchors as signal) | partner candidates with distance, affinity, tags |
| `artists_local` | Qloo `/v2/insights` (artists, `signal.location`) | artists for the room |
| `brands_local` | Qloo `/v2/insights` (brands, `signal.location`) | food and drink brands that over-index here |
| `rising` | Qloo `/v2/insights` with `bias.trends` off vs high; `/v2/trending` for artists | entities climbing in Qloo's population data |
| `events_this_week` | Tavily search and extract, triaged by Lightning | real events near the store this week |
| `menu_match` | Qwen3 Embedding 8B | menu item and block-tag pairs by similarity, each a citable pair |
| `partners_rank` | embeddings (reranker when available) | the Qloo places ordered against the store brief |
| `submit_candidates` | none | finishes: one ref per item kind |

Qloo parameters and what each answer means: [QLOO.md](QLOO.md).

## 2. Evidence check and verifier (code)

`src/server/agent/items.ts` (`evidenceCheck`) and `src/server/agent/verify.ts`. No model is involved.

- An item without Qloo evidence from this run is not shown.
- Every entity an item cites (its place, artists, and the entities on its evidence rows) must be in the Qloo payload this run fetched; an event tie-in must be a listing Tavily returned.
- Every lift an evidence row claims is recomputed from the saved city baseline (store affinity minus city affinity), within 0.02.

Each check's outcome is written to the run log, so the Runs view shows what was kept and why.

## 3. Copy (Nemotron 3.5 Lightning)

`src/server/agent/copy.ts`. Lightning writes each item's text from the candidate and its evidence summaries, in the chain's brand voice.

## 4. Guardrails

`src/server/guardrails/` (live, in the Worker) and `guardrails/` (NeMo Guardrails 0.24.1, batch).

Live check on every item and every head-office edit, in order:

1. **Prompt injection** in web-derived copy (event, staff card): a phrase rule blocks it.
2. **Health, diet, allergen, nutrition and targeting claims**: a regex catalogue, a TypeScript mirror of the NeMo custom action `check health claims` (a vitest test runs the Python file and fails on any difference).
3. **NeMo's `self check output` prompt** on Lightning, reasoning off (the prompt text is read from `guardrails/localedition/prompts.yml`, checked by a parity test).

A blocked line gets one rewrite from the copy model and a re-check.

Batch: the NeMo Guardrails library itself runs in a Nebius Serverless Job started from the CLI (`scripts/nebius-job.ts`). `certify` re-checks pending items and posts signed results (`certifiedBy: nemo-guardrails@nebius-job`); `eval` runs a 50-phrase set (40 red-team, 10 benign). The last eval caught 40 of 40 with 0 false positives.

## 5. Chain review (Nemotron 3 Ultra)

`src/server/agent/clash.ts`, `src/server/agent/review.ts`.

1. Code finds clashes between stores of one chain: the same partner, the same event night, near-identical copy (embedding cosine at or above 0.88), or sound briefs sharing at least half their artists.
2. One call to `nvidia/Nemotron-3-Ultra-550b-a55b` (strict JSON, thinking off) reads every store's edition with those clashes and returns `approve` or `needs_change` with a reason and a one-line hint per store.
3. A `needs_change` hint feeds **Regenerate**, which re-runs the agent for that one item kind, avoiding what the item says now.

In the current week's review, Ultra approved six stores and sent Upper East Side and Long Island City back for specials whose copy was near-identical to other stores'.

## Menu reading

`src/server/tokenfactory/menu.ts`. A photo of a menu board is read by Super VL into items and prices, then a second pass checks its own reading (`seen`, `priceLegible`, amber in the UI when false). A menu can also come from a CSV file.

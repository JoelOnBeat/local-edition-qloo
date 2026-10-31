# Testing

All of the suites below run offline: provider calls are faked or answered from synthetic fixtures, and no key is needed.

```sh
npm run typecheck && npm run lint
npm test                                        # vitest: 142 files, 1,623 tests (31 Oct 2026)
npm run e2e                                     # Playwright, starts `next dev` itself
cd guardrails && uv sync --python 3.11 --frozen && uv run pytest -q   # NeMo rails: 125 tests
python3 -m pytest -q tests/e                    # NeMo job scripts: 12 tests (needs the guardrails environment)
```

## What the Qloo tests cover

`tests/qloo/`:

| File | Checks |
|---|---|
| `client.test.ts` | the key travels only in the `X-Api-Key` header; WKT points longitude first; radius, `take` and popularity clamps; placeholder ids never sent as signals; 401, 429, 5xx, timeouts and malformed bodies all end as `qloo_unavailable`, never a sample answer |
| `ledger.test.ts` | the 2,000 total and 200 daily caps, refused before any request goes out; saved answers cost no call; cache-only pages never call; no client reaches Qloo without the ledger; no retry; failed answers are not saved; an unreachable ledger refuses |
| `block.test.ts` | lift against the city baseline, the floor for a tag missing from the baseline, at most 12 tags and 4 per kind, one baseline call per tag family across stores, distinct evidence-backed profiles per neighbourhood |
| `endpoints.test.ts` | places (distance, anchor explanation, radius, dropped coordinates), artists (popularity window, at least 8 per store), brands (health and beauty excluded), rising |
| `compare.test.ts` | the tag-lift diff; Qloo's compare called with each store's top places and the city; the diff still answers when compare is rate limited or forbidden; cached per pair |
| `anchor-search.test.ts` | saved searches first, 3 live searches per workspace per day |
| `parse.test.ts`, `fixtures.test.ts`, `routes.test.ts` | response parsing, fixture shapes, the profile and compare routes |

## What the agent tests cover

`src/server/agent/*.test.ts`: the tool loop with a scripted model (invalid arguments and unknown refs corrected, the step cap, repeated invalid calls, a silent or failing model handing over to the fixed plan, a failing tool the run survives), the evidence check and verifier (dropping items with no evidence, unknown entities and wrong lifts), copy with the template fallback, guardrail rewrites, the cross-store clash finder and the Ultra review with its fallback, regenerate, and model-unavailable skips.

The guardrail parity tests run the Python catalogue and injection rules and fail on any difference from the TypeScript mirror, and check that the Worker's self-check prompt is the one in `guardrails/localedition/prompts.yml`.

## Live smoke tests

`scripts/smoke-live.ts` checks Token Factory and Tavily live (reasoning, menu reading, tool calls, JSON schema, embeddings, rerank, Tavily). It spends real credit, so it is never part of `npm test`, and it never calls Qloo.

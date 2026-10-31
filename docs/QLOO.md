# Qloo in Local Edition

Qloo is the product's source of truth about each store's block. This page lists every Qloo call the deployed app makes, the parameters it sends, what each answer decides, how answers become evidence, and what happens when Qloo cannot answer. Code: `src/server/qloo/`, wired into the agent in `src/server/agent/live.ts` and `src/server/agent/tools.ts`.

## Contents

1. [The client](#the-client)
2. [Block profile: tags by location, minus the city](#1-block-profile-tags-by-location-minus-the-city)
3. [Places near the store: the local partner](#2-places-near-the-store-the-local-partner)
4. [Artists by location: the sound brief](#3-artists-by-location-the-sound-brief)
5. [Brands by location: pairing ideas](#4-brands-by-location-pairing-ideas)
6. [Search: the chain's taste anchors](#5-search-the-chains-taste-anchors)
7. [Compare: two stores side by side](#6-compare-two-stores-side-by-side)
8. [Rising (agent's choice)](#7-rising-agents-choice)
9. [The evidence model](#the-evidence-model)
10. [Caching and failure handling](#caching-and-failure-handling)
11. [Privacy](#privacy)

## The client

`src/server/qloo/client.ts`. Host `https://hackathon.api.qloo.com`, `GET` with query parameters, key in the `X-Api-Key` header (never in a URL, log, cache key or evidence row). Locations are WKT with longitude first: `POINT(-73.95712 40.71780)`.

Every call first looks for a saved answer in KV (`qloo:resp:v2:<sha256 of path and sorted params>`), labelled `cache`. Otherwise it goes to Qloo with a 20-second timeout, and the answer is saved with the time Qloo gave it and labelled `live`.

Production is **live-only** (`src/server/qloo/mode.ts`): the replay and fixture modes used by unit tests and local dev cannot be reached on the deployed Worker, whatever `QLOO_MODE` says.

## 1. Block profile: tags by location, minus the city

`src/server/qloo/block.ts`. Agent tool: `block_profile`. Also served by `GET /api/stores/:id/profile`, used by Compare, and shown on the landing page (which reads saved answers only and never makes a call).

**Calls.** For each of 7 tag families, one call at the store and one for the city:

| | Parameters |
|---|---|
| Store | `/v2/insights` `filter.type=urn:tag`, `signal.location=POINT(lon lat)`, `filter.tag.types=<family>`, `take=20` |
| City baseline | `/v2/insights` `filter.type=urn:tag`, `signal.location.query=New York City`, `filter.tag.types=<family>`, `take=50` |

The 7 families (`BLOCK_TAG_TYPES` in `src/server/qloo/util.ts`):

| `filter.tag.types` | Kind | Used for |
|---|---|---|
| `urn:tag:cuisine:qloo` | cuisine | the special |
| `urn:tag:specialty_dish:place` | flavour | the special |
| `urn:tag:menu_highlight:qloo` | flavour | the special |
| `urn:tag:ambience:qloo` | ambience | the mood of the sound brief and the event |
| `urn:tag:good_for:qloo` | occasion | the event night |
| `urn:tag:neighborhood_characteristic:qloo` | block character | the brief shown to the agent |
| `urn:tag:genre:music` | music | the sound brief's genres |

The city is asked as "New York City" because "New York" resolves to the state.

**Signal: lift.** For each tag, `lift = store affinity - city affinity`. A tag missing from the city's top 50 gets that list's lowest affinity (it sits below the cutoff). The profile keeps up to 12 tags with lift of at least 0.02, sorted by lift, at most 4 per kind so one family cannot crowd out the rest.

**What it decides.** The block's character in one list. `menu_match` embeds the cuisine and flavour tags against the chain's own menu to pick the special; `places_near` can filter partners by these tags; music tags steer `artists_local`; ambience and occasion tags shape the event night and the copy.

**If the tag query is not enabled for the key** (HTTP 403), the profile is built from the tags carried by restaurants within 800 m (`filter.type=urn:entity:place`, `filter.location`, `filter.location.radius=800`, `filter.tags=urn:tag:genre:place:restaurant`), weighted by affinity and compared with the same query city-wide.

## 2. Places near the store: the local partner

`src/server/qloo/places.ts`. Agent tool: `places_near`, then `partners_rank`.

| Parameter | Value |
|---|---|
| `filter.type` | `urn:entity:place` |
| `filter.location` | `POINT(lon lat)` of the store |
| `filter.location.radius` | 1200 m by default (clamped to 50 to 5000) |
| `signal.interests.entities` | the Qloo ids of the brands the chain resembles (its taste anchors) |
| `filter.tags` | optional block tags the agent passes from `block_profile` |
| `filter.exclude.entities` | places to avoid (when head office asks for a different partner) |
| `feature.explainability` | `true` when anchors are present |
| `take` | 15 by default, the agent asks for up to 20 |

Places use a hard location filter (`filter.location`), not a location signal: a partner must be walkable. The ranking signal is the chain's own brands.

**What it decides.** The partner candidates, each with its distance (haversine from the store), Qloo affinity, tags, and the anchor brand it is "matched because of" (from `query.explainability`). Places without coordinates are dropped. If the tag filter leaves fewer than 3 places, the tool tops up from the unfiltered list and tells the model it did. `partners_rank` then orders the candidates against the store brief with embeddings.

## 3. Artists by location: the sound brief

`src/server/qloo/artists.ts`. Agent tool: `artists_local`.

| Parameter | Value |
|---|---|
| `filter.type` | `urn:entity:artist` |
| `signal.location` | `POINT(lon lat)` of the store |
| `signal.interests.entities` | the chain's anchors |
| `filter.popularity.min` / `.max` | 0.2 / 0.7: the "local-feeling, not top 40" band |
| `bias.trends` | `low` |
| `filter.tags` | optional genres the model names, mapped to `urn:tag:genre:music:<genre>` |
| `filter.exclude.entities` | artists to avoid on a regenerate |
| `feature.explainability` | `true` when anchors are present |

**What it decides.** The artists for the in-store sound brief (artists, never tracks). A brief lists 8 to 15 artists. A genre filter that leaves too few is topped up from the unfiltered list.

## 4. Brands by location: pairing ideas

`src/server/qloo/brands.ts`. Agent tool: `brands_local`.

| Parameter | Value |
|---|---|
| `filter.type` | `urn:entity:brand` |
| `signal.location` | `POINT(lon lat)` of the store |
| `filter.tags` | `urn:tag:genre:brand:food_and_beverage` |
| `filter.exclude.tags` | `urn:tag:genre:brand:health_and_beauty` |
| `filter.popularity.max` | 0.8 |
| `take` | 15 |

**What it decides.** Food and drink brands that over-index on the block, which the agent uses as "pairs with" ideas for the special and the partner offer.

## 5. Search: the chain's taste anchors

`src/server/qloo/anchors.ts`, `src/server/qloo/anchor-search.ts`. Used by the setup page and to resolve the sample chain's anchors.

| Parameter | Value |
|---|---|
| `query` | the brand name typed (up to 80 characters) |
| `types` | `urn:entity:brand` |
| `take` | 8 on the setup page, 3 when resolving a known name |

**What it decides.** Which real brands the chain resembles (for the sample chain, Blue Bottle Coffee and others). Their entity ids become `signal.interests.entities` for the place, artist and rising queries, which is how the chain's identity reaches every recommendation. Search answers are saved for 14 days.

## 6. Compare: two stores side by side

`src/server/qloo/compare.ts`. The **Compare** screen (`/app/compare`, `GET /api/compare?a=&b=`).

1. Both stores' block profiles (section 1), diffed by lift: tags shared (both lift at least 0.05 and within 0.10 of each other) and tags that set each store apart (a gap above 0.10). This part is ours and always answers.
2. Qloo's own comparison, as evidence:

| Parameter | Value |
|---|---|
| endpoint | `/v2/analysis/compare` |
| `a.signal.interests.entities` | the top 5 places near store A (section 2) |
| `b.signal.interests.entities` | the top 5 places near store B |
| `filter.type` | `urn:entity:place` |
| `filter.location.query` | `New York City` |
| `model` | `descriptive` |
| `take` | 20 |

**What it decides.** The "shared taste" evidence row shown with the diff.

## 7. Rising (agent's choice)

`src/server/qloo/rising.ts`. Agent tool: `rising`. The model decides per store whether to call it, so not every edition uses it.

The same place or artist query is run twice, with `bias.trends=off` and `bias.trends=high`; an entity that climbs at least 5 ranks (or appears only in the trend-weighted run) is "rising in Qloo's population data". For artists only, `/v2/trending` (`filter.type=urn:entity:artist`, `signal.interests.entities`, `filter.start_date`, `filter.end_date`, 12 weeks) adds global momentum. Qloo answers 400 for places on `/v2/trending`, so places never call it. The prompt forbids the model from saying anything is "rising on this street".

## The evidence model

Every Qloo answer that informs an item becomes an `Evidence` row (`src/server/qloo/evidence.ts`, schema in `src/contract/domain.ts`):

```ts
{
  id: "ev_<hash of scope, path, params>",
  source: "qloo",
  endpoint: "/v2/insights",              // path only, never the host
  params: { "filter.type": "urn:tag", ... }, // as sent; any key-like param is scrubbed
  summary: "Taste tags for cuisine around Williamsburg (Bedford Ave & N 7th St), against the New York City baseline: ...",
  metrics: { affinity, popularity, lift, distanceM, relevance },
  entityIds: ["..."],                    // the Qloo entities the row is about
  fetchedAt: "2026-10-30T15:39:02Z",     // when Qloo answered, also for a saved answer
  mode: "live" | "cache"
}
```

The evidence drawer on every item shows these rows. Before an item reaches head office, code checks it against the run's own Qloo payload (`src/server/agent/verify.ts`):

1. **No evidence, no item.** An item without Qloo evidence from this run is not shown.
2. **Entities must exist.** Every entity the item cites (its place, its artists, the entities on its evidence rows) must be one a Qloo tool returned in this run. An event tie-in must be a listing Tavily returned.
3. **Lifts must recompute.** Every cited lift is recomputed as store affinity minus the saved city baseline, the same formula as the block profile, within 0.02.

A visitor's sample workspace copies only editions whose Qloo evidence is all `live` or `cache` (`src/server/sample/live-editions.ts`).

## Caching and failure handling

Qloo answers are cached in KV for 14 days (editions are built weekly, and a judge reads the exact answer an edition was built from). Derived results are cached too: city baselines for a day, block profiles and compare results for 6 hours, brand searches for 14 days, all built only from live or saved answers.

When Qloo cannot answer, the call raises `qloo_unavailable`; the API returns `{ error, code }` and the screen shows the message with a retry ("Qloo did not answer in time. Try again."). No sample answer stands in. A block profile is saved only when every tag family answered, so a partial profile is never cached. A 403 for the tag query switches to the place-based profile described in section 1. Inside a build, a failing tool shows in the run log.

## Privacy

Qloo receives only store coordinates, city names, Qloo entity ids, tag urns and the brand names typed into setup search. No customer, sales or personal data. Real Qloo answers live only in the KV cache, never in git: `fixtures/qloo/` is synthetic, for tests and local dev.

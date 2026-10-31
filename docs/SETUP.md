# Setup, run and deploy

## Requirements

- Node.js 22 or newer and npm (Next.js 16).
- For deploys: a Cloudflare account on the Workers Paid plan (Workflows and Durable Objects), and `wrangler` (installed as a dev dependency).
- Optional: Python 3.11 or 3.12 with `uv` for the NeMo Guardrails tests in `guardrails/`.
- Optional keys: Qloo (hackathon key), Nebius Token Factory, Tavily. Nothing else is needed to run locally.

## Run locally

```sh
git clone https://github.com/JoelOnBeat/local-edition-qloo.git
cd local-edition-qloo
npm install
cp .env.example .env.local      # optional; every value may stay blank
```

Two modes:

| Command | What runs |
|---|---|
| `NEXT_PUBLIC_DATA_MODE=mock npm run dev` | The UI only, on typed mock data (`src/mocks/`). No server calls. |
| `npm run dev` | The real API routes. Data lives in `.data/store.json` (`STORE_DRIVER=file`). Without keys, each provider answers from synthetic fixtures (`fixtures/qloo/`, `fixtures/tavily/`, `fixtures/tokenfactory/`), badged "Sample data" in the UI. |

Open http://localhost:3000 and press **Try the sample chain**.

To call real providers locally, put the keys in `.env.local` and set `QLOO_MODE=live`, `TF_MODE=live`, `TAVILY_MODE=live`. Live calls use your own provider credits; `SPEND_CAP_USD` caps Token Factory spend.

To run the Worker build locally with D1, KV, R2, Durable Objects and Workflows simulated:

```sh
npx wrangler d1 migrations apply LE_DB --local
npm run preview                  # opennextjs-cloudflare build && preview
```

## Environment variables

Every name is in [`.env.example`](../.env.example) with a one-line purpose. In short:

| Name | Purpose |
|---|---|
| `QLOO_API_KEY` | Qloo Taste AI key. Production is live-only. |
| `NEBIUS_API_KEY` | Nebius Token Factory (Nemotron models, Qwen3 embeddings). |
| `TAVILY_API_KEY` | Tavily search and extract. |
| `QLOO_MODE`, `TF_MODE`, `TAVILY_MODE` | `live`, `replay` or `fixture` outside production (default `replay`, then `fixture`). |
| `STORE_DRIVER` | `memory`, `file`, `kv` or `d1`; production uses `d1`. |
| `APP_URL` | Public base URL, for links and webhook signatures. |
| `SPEND_CAP_USD` | Token Factory spend cap (default 5). |
| `BUILD_MODE` | `workflow` (Cloudflare Workflows) or `inline`. |
| `LE_JOB_SECRET` | HMAC key (32+ characters) for the signed job routes. |
| `NEXT_PUBLIC_DATA_MODE` | `mock` for the UI on mock data. |
| `LE_DATA_DIR`, `LE_RECORDINGS_DIR`, `LOG_LEVEL` | Local folders and log level. |
| `CODE_SENDER`, `GMAIL_*` | Optional emailed sign-in codes. |
| `SQUARE_*`, `SLACK_*`, `COMPOSIO_*`, `COMMS_*` | Optional connector modules (Square, Slack, Gmail/Outlook). |

`.env*` files are gitignored except `.env.example`.

## How the sample data is seeded

The sample chain, **Kettle & Kiln** (eight stores at real New York corners, its menu and its three anchor brands), is code: `src/server/sample/chain.ts`, with the menu board in `fixtures/sample/` and a store CSV in `fixtures/sample/stores.csv`. Nothing needs to be imported.

- **Demo workspace** (`demo`): created from the sample chain on first use by the weekly build (`ensureDemoWorkspace` in `src/server/session.ts`). Its editions are the ones the scheduled build makes.
- **Visitor workspace**: **Try the sample chain** (`POST /api/auth/guest`) creates a private workspace with a fresh copy of the sample chain in one batch, then copies each store's newest finished edition and staff card from the demo workspace. In production only editions whose Qloo evidence is all live or saved-live are copied (`src/server/sample/live-editions.ts`).
- **Locally** the same code runs against `.data/store.json`. The demo workspace starts empty, so build editions from the app (**Build this week's editions**); without keys the build runs on the fixtures.
- **In production**, the first editions come from the weekly build. To run it before the first Monday, either send a signed `POST /api/jobs/monday` (HMAC with `LE_JOB_SECRET`, scheme in `src/server/jobs/job-auth.ts`) or trigger the Workflow:

```sh
npx wrangler workflows trigger local-edition-weekly-rebuild \
  '{"runId":"weekly-manual-1","workspaceId":"demo","trigger":"cron"}'
```

Visitor workspaces expire and are purged by the Monday cron.

## Database migrations

D1 schema lives in `migrations/`:

| File | Adds |
|---|---|
| `0001_init.sql` | workspaces, chains, stores, menus, editions, staff cards, evidence, job runs, locks, cache, build runs and build events |
| `0002_indexes_and_attempts.sql` | indexes for workspace scoping and expiry, build event scopes |
| `0003_square.sql` | connector connections, events and deliveries, Square locations, OAuth state, calendar tokens |
| `0004_comms.sql` | Slack and mail documents and claims |
| `0005_auth.sql` | users, sessions, onboarding, sign-in codes, auth counters |

```sh
npx wrangler d1 migrations apply LE_DB --local     # local
npx wrangler d1 migrations apply LE_DB --remote    # production
```

## Deploy to Cloudflare

1. **Create the resources** and put their ids in `wrangler.jsonc`:

   ```sh
   npx wrangler d1 create local-edition          # binding LE_DB
   npx wrangler kv namespace create LE_KV         # binding LE_KV
   npx wrangler r2 bucket create local-edition-blobs   # binding LE_BLOB
   ```

   Durable Objects (`SpendMeterDO`, `BuildLockDO`, `TavilyBudgetDO`; the Qloo ledger is a named instance of the last), the two Workflows (`local-edition-edition-build`, `local-edition-weekly-rebuild`) and the cron (`0 0 * * MON`) are declared in `wrangler.jsonc` and created by `wrangler deploy`. Vars such as `STORE_DRIVER=d1`, `BUILD_MODE=workflow`, the provider modes and `APP_URL` are set there too.

2. **Set the secrets** (each reads the value from stdin):

   ```sh
   npx wrangler secret put QLOO_API_KEY
   npx wrangler secret put NEBIUS_API_KEY
   npx wrangler secret put TAVILY_API_KEY
   npx wrangler secret put LE_JOB_SECRET
   # optional connectors:
   # SQUARE_ACCESS_TOKEN SQUARE_APP_ID SQUARE_APP_SECRET SQUARE_TOKEN_KEY SQUARE_WEBHOOK_SIGNATURE_KEY
   # SLACK_BOT_TOKEN SLACK_SIGNING_SECRET SLACK_CLIENT_ID SLACK_CLIENT_SECRET
   # COMPOSIO_API_KEY COMPOSIO_WEBHOOK_SECRET COMMS_ENC_KEY
   # optional sign-in codes: GMAIL_CLIENT_ID GMAIL_CLIENT_SECRET GMAIL_REFRESH_TOKEN
   ```

3. **Migrate, build and deploy the Worker:**

   ```sh
   npx wrangler d1 migrations apply LE_DB --remote
   npx opennextjs-cloudflare build
   npx wrangler deploy
   ```

   The Worker deploys with `workers_dev = false`, so it has no public address of its own.

4. **Deploy the Pages front** (`edge/pages/`), a Pages project named `localedition` with a service binding `APP` to the Worker `local-edition`. Run Pages commands from a copy of `edge/pages/` outside the repo: inside it, wrangler picks up the root `wrangler.jsonc` and deploys a second Worker.

   ```sh
   cp -R edge/pages /tmp/le-pages && cd /tmp/le-pages
   npx wrangler pages deploy public --project-name localedition --branch main
   ```

   Add the `APP` service binding in the Pages project settings (or through the Cloudflare API) if the project is new.

5. **Check:** `curl -s https://<your-pages-domain>/api/status` should show `"qloo":"live"`.

`scripts/deploy.sh` runs steps 3 and 4 in one go from a clean worktree; it reads Cloudflare credentials through a local secret helper, so adapt that line to your own secret handling.

## Optional: NeMo Guardrails job

`npx tsx scripts/nebius-job.ts certify eval` starts a Nebius Serverless Job (CPU, `python:3.12-slim`) that runs the NeMo Guardrails library over pending items and the red-team eval, then posts signed results to the app. It needs the Nebius CLI configured with a service account, plus `NEBIUS_API_KEY` and `LE_JOB_SECRET`.

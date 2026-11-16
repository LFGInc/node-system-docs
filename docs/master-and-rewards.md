# Master and rewards

The backend records uptime; **turning uptime into points or tokens is the project's job**. Projects do it
with a service that uses two backend endpoints:

- read `GET /api/v1/projects/<code>/node-uptimes?date=` (valid daily uptime per node, the same data that is
  committed on-chain), and
- write `POST /api/v1/projects/<code>/node-points` (admin-signed; `point` and optional `rewardTokens` per
  node and day; updates existing `DAILY` rows only).

Two reference services exist. Use one, both, or write your own against those two endpoints.

| Service | For | Repository |
|---|---|---|
| Workload master sample | Points only (Valley-style); also serves tasks to workloads and can be the deployer's IPFS upload API | [`opensource-nodesys-workload-master-sample`](https://github.com/LFG-Incorporated/opensource-nodesys-workload-master-sample) (Go) |
| nodesale-reward | Token rewards (Helix- and FlipIt-style): splits tokens, keeps claimable balances, signs claim vouchers | [`opensource-nodesale-reward`](https://github.com/LFG-Incorporated/opensource-nodesale-reward) (NestJS) |

!!! warning "Timing: when is yesterday's data ready?"
    `node-uptimes` reads the backend's `DAILY` rows, which only exist after the backend's daily snapshot
    (02:02 UTC, plus IPFS upload and transaction time). The master sample's example config uses
    `daily_submit_at: "02:00:30"`, which is before that on the same clock. Set the master's time **after**
    the daily summary is `COMPLETED` (check `GET /api/v1/cron/uptime-summary?date=<ISO>`); a later time
    such as `02:30:00` is a safe choice. nodesale-reward runs at 02:10 UTC and checks the summary status
    itself. If the backend's daily job is slow, 02:10 can be tight.

## Workload master sample

One Go process (gin, MongoDB, Kubo, Pinata) doing two jobs:

1. **Task API for workloads** (`/api/v1/cids/list`, `/api/v1/workload/report`): counterpart of the
   [workload template](workloads.md#workload-template).
2. **Daily point submitter**: per configured project, reads yesterday's per-node uptime from your backend
   and posts node points back. Valley and FlipIt used this exact code in production.

```bash
cp app.example.yaml app.yaml     # edit; app.yaml is git-ignored; the path ./app.yaml is hard-coded
docker compose up -d             # optional: local Kubo (API on host :5002)
make build && make run           # or: go run ./cmd/main.go   (Go 1.22+)
# Docker
docker build -t workload-master . && docker run -d -p 3001:3001 -v "$PWD/app.yaml:/app/app.yaml:ro" workload-master
```

(The Dockerfile `EXPOSE`s 8080 but the server listens on `port` from the config; publish the port you
configured.)

### Configuration (`app.yaml`)

All keys are required and validated at startup (`pkg/config/parse.go`); the process exits on a missing or
invalid key.

| Key | Type | Description |
|---|---|---|
| `ipfs_url` | URL | Kubo HTTP API the master uses to add and get files (for example `http://localhost:5001`) |
| `port` | string | HTTP listen port |
| `mongo.uri` | URI | MongoDB connection string; pinged at startup |
| `mongo.db` | string | Database name. Collections `cids` and `workloadCheckPoints` (unique indexes) are created on startup |
| `mongo.list_cid_limit` | int > 0 | Max CIDs returned per `/cids/list` call |
| `pinata.url` | URL | Pinata API base, `https://api.pinata.cloud` |
| `pinata.token` | string | Pinata JWT (`POST {url}/pinning/pinFileToIPFS`) |
| `lfg_node_system.url` | URL | Your backend base URL; the master calls `{url}/api/v1/projects/...` |
| `lfg_node_system.private_key` | string | Hex key (64 chars, **no `0x`**) that signs point submissions; its address must be in `project_admins` of each project |
| `lfg_node_system.projects[]` | list (min 1) | One submitter job per entry |
| `...projects[].code` | string | Project code |
| `...projects[].uptime_threshold` | uint > 1000 | Minimum uptime (ms) to earn points. Valley and FlipIt used `21600000` (6 h) |
| `...projects[].daily_submit_at` | `HH:MM:SS` | Time of day to run, in the process's local zone (UTC in the Docker image); see the timing note above |
| `...projects[].user_gain_point` | uint > 0 | Points awarded to each qualifying node |

(The key prefix `lfg_node_system` is a historical name; it is your backend.)

### HTTP API

Base path `/api/v1`, responses `{"data": ..., "error": "..."}`. **No authentication** on any route.

| Route | Purpose |
|---|---|
| `POST /cids/list` body `{"node_id"}` | Up to `mongo.list_cid_limit` CIDs, oldest first, with `created_at` greater than the node's checkpoint |
| `POST /workload/report` body `{"node_id","oks":[],"fails":[]}` | Stores a checkpoint (max `created_at` of `oks`); if `oks` is empty replies `200` with `error: "oks cannot be empty"` and stores nothing; `fails` are only logged |
| `POST /static/text` body `{"data":"<text>"}` | Adds and pins text on Kubo, uploads to Pinata (CID v0) asynchronously, inserts into `cids`; returns `{"data":"<cid>"}`. Satisfies the deployer's `ipfs_api_url` contract |
| `POST /static/file` multipart `file` | Same for files |
| `GET /cids/:cid` | File bytes through your Kubo node |

CIDs become available to `/cids/list` only by being inserted into `cids` (by an upload, or by writing
`{cid, created_at}` documents yourself).

### Point submitter

Per project, each day at `daily_submit_at`:

1. `date = now - 24h` (UTC).
2. `GET {url}/api/v1/projects/{code}/node-uptimes?date=<2006-01-02T15:04:05Z>`; non-200 is an error; an
   empty list submits nothing.
3. Per node: `point = user_gain_point` if `uptime >= uptime_threshold`, else `0`.
4. `POST {url}/api/v1/projects/{code}/node-points` with body
   `{"date":<date>,"projectCode":"<code>","nodes":[{"nodeAddress":"0x..","point":N}]}`, headers
   `X-User-Address` (address of `private_key`) and `X-Signature` (EIP-191 `personal_sign` of the body
   bytes, `0x` hex, `v` 27/28). Success is HTTP `201`.

Failures are only logged (no retry); days missed during a migration are **not** backfilled (call the
endpoints yourself, or run once with the commented `OneTimeJobStartImmediately` line in
`internal/pointsubmitter/start.go`). Workload reports (`/workload/report`) are not used for points: points come only from the backend's
uptime data.

CI: `.github/workflows/cd.yml` builds the `Dockerfile` on tags `v*` and pushes `<image>:<tag>`; configure
variables `IMAGE_REPOSITORY` (default `ghcr.io/<owner>/<repo>` lower-cased), `REGISTRY` (default
`ghcr.io`), secrets `REGISTRY_USERNAME` / `REGISTRY_PASSWORD` (default `github.actor` / `GITHUB_TOKEN`).
It does not deploy. Then point your workloads' `MASTER_URL` at the server and publish a new workload
release.

## nodesale-reward

Daily token-reward distribution for node operators (NestJS 11, Fastify, Prisma, PostgreSQL, built with
Bun). It ran in production for the Helix and FlipIt programs. For each configured project it:

1. reads how many reward tokens are claimable from the token contract (00:05 UTC),
2. makes the project's `NodeOperatorsReward` pull them in (01:00 UTC),
3. splits them between the nodes that were up yesterday, records the result and submits per-node points to
   the backend (02:10 UTC),
4. serves a small HTTP API that signs **claim vouchers**, and keeps balances in sync with on-chain claims
   through the [reward subgraph](subgraphs.md#reward-subgraph).

```
 token contract --(getClaimableAmount)--> 00:05 ledger PENDING
 NodeOperatorsReward <--claim<Code>Token()-- nodesale-reward --GET uptimes / POST node-points--> backend
        ^  (claimer key, 01:00)                  |  ^   02:10: split, credit `unclaim` (POINTED)
        | claim(voucher)                         |  | every ~15 s: Claim events from reward subgraph
 operator wallet <-- voucher (signer key) -------+  +-- PostgreSQL (balances, nonces, ledgers)
```

### Requirements

- **PostgreSQL** (17 recommended), schema **must be `public`** (`?schema=public` in the URL).
- **Bun >= 1.2** (the Docker image uses `oven/bun:1.2.8-alpine`).
- **Alchemy** as RPC provider (`alchemy-sdk`). The chain id **must be listed** in `CHAIN_ID_TO_NETWORK`
  (`src/packages/alchemy/alchemy.constant.ts`; Ethereum, Optimism, Arbitrum, Polygon, Base, BNB, Avalanche,
  Linea, zkSync, Scroll, Mantle and testnets). Another chain or provider means editing that file and
  `alchemy.service.ts`.
- A backend implementing the [three endpoints below](#backend-endpoints-it-uses), a reward subgraph, and
  **three funded or role-bearing wallets per project**. Optional Slack bot.

### Setup

```bash
bun install && bunx prisma generate && bun run build
cp .env.example .env              # NODESALE_REWARD_URL, TZ
cp .env.yaml.example .env.yaml    # keys, admin address, chains (git-ignored)
sh prisma/migration.sh            # create tables (idempotent)
# seed projects (below), then, with cwd containing .env.yaml:
TZ=UTC bun dist/main.js           # or: bun run dev
```

Docker: `docker build -t nodesale-reward .`, then run with
`-e TZ=UTC -e NODESALE_REWARD_URL='postgresql://...?schema=public&connection_limit=5' -v "$PWD/.env.yaml:/app/.env.yaml:ro"`;
the container applies migrations when `_prisma_migrations` is empty. `docker-compose.example.yaml` adds
Postgres 17. **Run a single replica.** Projects and keys load at startup: restart after adding a project or
changing keys.

`.env`: `NODESALE_REWARD_URL` (Prisma datasource and `migration.sh`), `NODE_ENV` (legacy; the effective
mode is `env:` in `.env.yaml`), `TZ=UTC` (job date arithmetic uses local-time functions).

`.env.yaml` (read from `./.env.yaml` in the working directory):

| Key | Required | Notes |
|---|---|---|
| `env` | yes | `default` / `staging` / `development` / `production`. `default` logs every SQL query; `production` disables Swagger and enables the strict CSP |
| `app.port` | no (3000) | |
| `app.version` | no | use `"1"` (routes are `/v1/...`) |
| `app.prefix` | no | **If omitted there is no prefix.** Set `api` for `/api/v1/...` as used in these docs |
| `app.whitelist` | no | CORS origins: `"*"` or a list; set explicit origins in production |
| `logger.level` / `pretty` / `ignores` | no | pino settings |
| `cookie.*` | no | `@fastify/cookie` settings; the service itself does not use cookies |
| `swagger.enabled/name/pass` | no | Basic-auth Swagger at `/<prefix>/v<version>/docs`, not served in production |
| `cron.time` | **yes (section)** | subgraph sync interval in ms per project, minimum and default `15000`; the `cron:` section must exist or the process crashes at startup |
| `alchemy.apiKey` | yes | Alchemy key |
| `alchemy.<chainId>.<sale>.{system,signer,claimer}` | yes | Private keys per project; `<sale>` must equal `projects.sale` (lower-case) exactly |
| `admin.address` | yes | Lower-case address allowed to call admin endpoints |
| `slack.token`, `slack.channel` | no | Omit the whole `slack:` section to disable |

### Keys and roles

| Key | Used for | Needs on `NodeOperatorsReward` | Needs elsewhere |
|---|---|---|---|
| `claimer` | Sends the daily `claim<Code>Token()` transaction; also the read-only `getClaimableAmount` call | `CLAIMER_ROLE` | native gas |
| `signer` | Signs the `ClaimReward` voucher (raw secp256k1 over the ABI-encoded hash, returned as `{ r, s, v }`) | `SIGNER_ROLE` | nothing, holds no funds |
| `system` | EIP-191 signs the node-points payload (`X-Signature`); its address is `X-User-Address` | none | must be an **admin of the project in the backend** (`project_admins`) |

The daily function is `claim` + `<Code>` + `Token`, with `<Code>` the project code with its first letter
upper-cased, called on `projects.operator_addr`; `getClaimableAmount(operator_addr)` is read from
`projects.contract_addr`; the voucher's `contract` field is `operator_addr`. See the
[naming constraint](contracts.md#naming-constraint). Use three different wallets; `DEFAULT_ADMIN_ROLE`
never needs to be held by this service.

### Seeding a project

Insert one row each in `projects` and `projects_systems`:

```sql
INSERT INTO projects (sale, code, name, chain_id, is_auto_point, subgraph_url, contract_addr, operator_addr, updated_at)
VALUES ('0x<sale-id-lowercase>',   -- any lower-case 0x address that identifies the project; key in .env.yaml and API URLs
        '<code>',                  -- backend project code; builds claim<Code>Token(); filters subgraph events
        '<Project name>', <chain_id>,   -- chain id must be in CHAIN_ID_TO_NETWORK
        false,                     -- is_auto_point
        '<https://subgraph-endpoint>',
        '0x<token-contract-lowercase>',        -- exposes getClaimableAmount(address)
        '0x<node-operators-reward-lowercase>', -- NodeOperatorsReward
        now());
INSERT INTO projects_systems (sale, api_url, gain_point, threshold)
VALUES ('0x<sale-id-lowercase>', 'https://node-system.example.com',  -- BASE HOST ONLY: no /api/v1, no trailing slash
        1,                    -- gain_point: keep 1
        <threshold>);         -- minimum uptime to qualify; sent as `uptime.from`
```

- `sale` joins `projects`, `projects_systems`, `projects_wallets` and the `alchemy.<chainId>.<sale>` key; all
  must match exactly.
- `is_auto_point = true`: the day's reward is added to `projects_wallets.unclaim` at 02:10 (claimable at
  once). `false`: rewards stay in `projects_rewards` with `pointed_at = NULL` until an admin calls the
  pointing endpoint for that date.
- Wallet rows are created at 02:10 when a wallet first earns a reward; the claim API answers only for
  existing wallet rows. A project without `contract_addr` / `operator_addr` or a `projects_systems` row
  fails (Slack is notified if configured). Set `subgraph_url`: a sync interval starts per project at boot.

### Daily schedule (UTC)

| Time | Job | What happens |
|---|---|---|
| 00:05 | `TaskPoint.getPoint` | Reads `getClaimableAmount(operator_addr)`; creates the day's ledger as `PENDING` (or `0` + `reason` on error) |
| 01:00 | `TaskPoint.claimPoint` | For `PENDING` ledgers the claimer calls `claim<Code>Token()`, waits for the receipt, stores `tx_hash`, status `CLAIMED` |
| 02:10 | `TaskReward.calculateRewards` | For `CLAIMED` ledgers: requires the backend's daily summary of **yesterday** to be `COMPLETED`; pages through qualifying nodes (500 per page); each node gets `amount / total * gain_point`; POSTs to node-points signed with `system`; writes `projects_rewards`, upserts `projects_wallets`, ledger `POINTED` |
| every `cron.time` ms | `JobService.syncReward` | Reads claim events newer than the cursor from the subgraph: `unclaim -= claimed`, `claimed += claimed`, `nonce = event.nonce + 1` |

If the summary is not `COMPLETED` at 02:10 (or any step fails) the project's day is skipped, Slack is
notified, and **nothing retries**; re-running a day has no endpoint (fix the database, or add a script
around `getPoint` / `claimPoint` / `calculateRewards`). Never re-run a day that is already `POINTED`.

### Backend endpoints it uses

`projects_systems.api_url` is the base host. Unauthenticated GETs:

1. `GET {api_url}/api/v1/cron/uptime-summary?date=<ISO>` gives `{status:1, data:{date, status:"PENDING|COMPLETED|FAILED"}}`.
2. `GET {api_url}/api/v1.1/projects/{code}/node-uptimes?page=&size=500&date=<ISO>&uptime.from=<threshold>`
   gives `{data:[{nodeAddress, walletAddress, uptime}], meta:{page,isFirst,isLast,size,next,prev,total,count}}`.
   `meta.total` must be the total number of qualifying nodes across all pages (it is the divisor).
3. `POST {api_url}/api/v1/projects/{code}/node-points` with `X-Signature` (EIP-191 of `JSON.stringify(body)`)
   and `X-User-Address`; body `{date, nodes:[{nodeAddress, point: gain_point, rewardTokens: "<integer string>"}]}`,
   once per page.

`date` is the full ISO timestamp (UTC) of "yesterday at the current time of day".

### Claim flow and HTTP API

Routes are under `/<prefix>/v<version>` (for example `/api/v1`); `:sale` and `:addr` are lower-case.

| Method and path | Auth | Purpose |
|---|---|---|
| `GET /projects/:sale/wallets/:addr/operators` | none | `{unclaim, claimed, lastReward?}` (smallest unit); 404 `WALLET_NOT_FOUND` if no row |
| `POST /projects/:sale/wallets/:addr/operators` | none (by design) | Voucher `{deadline, contract, receiver, nonce, amount, signature:{r,s,v}}`; `deadline` = now + 600 s, `amount` = the wallet's whole `unclaim` |
| `GET /projects/:sale/wallets/:addr/histories?page=&size=` | none | Paginated claim history |
| `POST /projects/:sale/pointing` body `{"date":"YYYY-MM-DD"}` | admin | Marks the day's not-yet-pointed rewards as pointed (only for `is_auto_point = false`) |

Admin headers: `x-offchain-address` (lower-case, equal to `admin.address`), `x-offchain-issued-at`
(`Date.now()` ms, rejected if older than 60 s), `x-offchain-signature` (`personal_sign` of
`JSON.stringify({ body, issuedAt })`).

The claim frontend (not part of the repository) calls `POST .../operators`, then submits the returned
`{amount, deadline, nonce, v, r, s}` to `NodeOperatorsReward.claim` from the operator's wallet. Within about
15 s the subgraph sync lowers `unclaim`, raises `claimed` and moves `nonce`. The dashboard can link an
inventory card to your claim frontend with `NEXT_PUBLIC_PROJECT_CLAIM_LINKS` ([Dashboard](dashboard.md)).

### Operations

One instance per database; no health route (use a TCP check or `GET .../operators` for a known wallet);
logs are pino JSON. `projects_wallets` is the **only record** of what operators can still claim and their
nonce: back it up. Known weaknesses (string-built SQL, `gain_point > 1` over-credits, unauthenticated
claim endpoint, single-admin auth, cursor races, no retries) are in [Security](security.md#nodesale-reward).
Migrating the reward database from LFG: [Migration from LFG](migration-from-lfg.md#reward-database-helix-and-flipit).

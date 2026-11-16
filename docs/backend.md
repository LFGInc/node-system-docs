# Backend

Repository: [`opensource-node-system-be`](https://github.com/LFG-Incorporated/opensource-node-system-be)
(NestJS 11 on Fastify, Prisma 7, PostgreSQL, Redis). The README describes production tag `v0.5.12` and is
the full reference (including the whole HTTP API and data model); this page covers what you need to
deploy and operate it.

The backend registers operators' nodes, tracks the uptime of the workloads they run, checks licenses
(NFT balances copied from a [subgraph](subgraphs.md#license-subgraph)), turns uptime into hourly and daily
summaries, publishes them to IPFS, commits their Merkle roots on-chain to the
[registry](contracts.md#uptime-registry), and stores the points and reward amounts that each project's
[master or reward service](master-and-rewards.md) writes back.

!!! danger "Known unfixed vulnerabilities"
    Including SQL injection by any project admin (S-04) and a missing ownership check on node deletion
    (S-08). Read [Security](security.md#backend) before exposing an instance. Run **exactly one
    instance**: cron jobs run in-process, two instances would submit every root twice.

```mermaid
graph TD
    CLI[Node CLI / workload manager] -->|signed PATCH uptime| A
    UI[Dashboard] -->|REST| A
    RW[Project master / reward service] -->|admin-signed node-points| A
    A[NestJS + Fastify backend]
    A --> P[(PostgreSQL 16)]
    A --> R[(Redis db 0)]
    A -->|every 5 s, GraphQL| G[License subgraph]
    A -->|hourly / daily JSON| I[Pinata / IPFS]
    A -->|submit roots| C[Registry contract]
    A --> S[Slack]
    A --> M[SMTP]
    A -->|node IP lookup| IP[ip-api.com]
```

## Requirements

| Component | Requirement |
|---|---|
| Runtime | Bun 1.3 (package manager and build) and Node.js 20 or newer (runtime image uses Node 24) |
| PostgreSQL | **16**; needed at runtime **and** at build time (see Build) |
| Redis | Database **0** only, keyspace notifications `KEA`. The app runs `CONFIG SET` itself at boot: allow `CONFIG` for its user or preconfigure the server. Managed Redis that blocks `CONFIG` crashes the app at start |
| IPFS pinning | A [Pinata](https://pinata.cloud) account and JWT able to upload files (Pinata v3 `POST /v3/files`, `network=public`). Another provider needs code changes |
| Chain | JSON-RPC endpoint and **your own** `GlobalNodeUptimeMerkleRegistry`. The `ether.privateKey` wallet needs `SUBMITTER_ROLE` and gas. One registry per project |
| Subgraph | GraphQL endpoint with entities `NodeTotalBalance` (ERC1155) and/or `ERC721NodeTotalBalance` (ERC721); the [license subgraph](subgraphs.md#license-subgraph) provides them |
| SMTP, Slack | Optional in practice (offline e-mails, submission notices) but the config keys are required |
| Egress | Pinata, RPC, subgraph, SMTP, Slack, `http://ip-api.com` (geo lookup, plain HTTP, free tier is non-commercial) |
| Process timezone | `TZ=Etc/GMT+2` ([why](#cron-jobs-and-timezone)) |

## Configuration

Configuration is **`config.yaml` in the process working directory**, not environment variables. The only
environment variables are `DATABASE_URL` (Prisma tooling: migrate, generate, seed) and `TZ`. Start from
`config.example.yaml`, which documents every key. The file is validated at boot; any problem aborts with
`Configuration validation failed` and the offending property. **Quote `0x...` values**, otherwise YAML
parses them as numbers.

| Key | Default | Required | Description |
|---|---|---|---|
| `nodeEnv` | `development` | no | `development` / `staging` / `production`. `production` enables the strict Helmet CSP (breaks Swagger UI) and plain e-mail subjects; others prefix `[DevTest]` |
| `server.port` | `3000` (example file: `7777`) | no | Listen port, binds `0.0.0.0`. The Docker image exposes 7777 |
| `server.version` | `"1"` | no | Default URI version |
| `server.enableSwagger` | `true` | no | Swagger UI at `/docs`, spec `/docs/yaml`. Set `false` in production |
| `server.corsWhitelist` | `[]` | no | Allowed browser origins (credentials on). Put your dashboard origin here |
| `server.corsExposeHeaders` | all HTTP methods | no | Passed as CORS allowed *methods* (historical name) |
| `server.cookieSecret` | | **yes** | Cookie / CSRF signing secret |
| `server.cronSecret` | | **yes** | Value of the `x-cron-secret` header for `/cron/trigger/*` |
| `server.traceSampleRate` / `profileSampleRate` | `0.01` | no | Sentry sampling |
| `server.sentryDsn` | `""` | no | Empty disables Sentry |
| `server.ignoreRequestLoggingPaths` | `[]` | **yes** (may be `[]`) | Regexes of URLs excluded from the request log; `[".*"]` silences it |
| `smtp.hosts` | `[smtp.gmail.com]` | no | Only `hosts[0]` is used, always port 587 (STARTTLS) |
| `smtp.port` | `587` | no | Validated but unused |
| `smtp.user`, `smtp.pass`, `smtp.from` | | **yes** | SMTP credentials and `From` header |
| `redisUrl` | | **yes** | for example `redis://localhost:6379/0` |
| `databaseUrl` | | **yes** | PostgreSQL URL (runtime). Prisma tooling reads `DATABASE_URL` instead |
| `ipfs.url` | `https://uploads.pinata.cloud` | no | Pinata upload API base |
| `ipfs.jwt` | | **yes** | Pinata JWT |
| `ipfs.gateway` | `https://ipfs.io/ipfs/` | no | Gateway base used to build CID links (trailing slash) |
| `ether.rpcUrl` | | **yes** | JSON-RPC URL |
| `ether.privateKey` | | **yes** | Submitter key (`0x` + 64 hex): `SUBMITTER_ROLE` and gas |
| `ether.contractAddress` | | **yes** | Your registry address |
| `slack.token`, `slack.channel` | | **yes** | Bot token (`chat:write`) and channel id; any placeholder if unused (failures are only logged) |
| `cli.workloadManagerImage` / `releaseEndpoint` / `ipfsGateway` / `sentryDsn` | `""` | no | Returned as `WORKLOAD_MANAGER_IMAGE`, `RELEASE_ENDPOINT`, `IPFS_GATEWAY`, `SENTRY_ENDPOINT` by `GET /public/cli-constants` |
| `branding.name` | `Node Dashboard` | no | Product name in e-mail subjects and bodies |
| `branding.dashboardUrl`, `logoUrl`, `termsUrl`, `privacyUrl`, `supportUrl` | `""` | no | E-mail links and logo; omitted when empty |
| `config.nodeOfflineThreshold` | `90000` ms | **yes** (section) | Silence longer than this = OFFLINE, uptime session closed |
| `config.nodeOfflineNotificationThreshold` | `900000` ms | no | Redis TTL of "last seen" keys; when it elapses an offline notification (web and e-mail) is generated |

`config.yaml` holds the submitter key, Pinata JWT, SMTP password and cron secret in plain text: restrict
its permissions. The `cli` and `branding` sections are new in the open-source release; omitting them
breaks nothing (`cli-constants` then returns empty strings).

## Build, migrate, run

**`prisma generate --sql` needs a live, migrated database**: the typed-SQL feature introspects the
database. On an empty database it fails with `P1014 The underlying table for model
node_workload_points does not exist`. The database can be throwaway.

```bash
bun install
export DATABASE_URL=postgresql://user:pass@host:5432/db
bun prisma migrate deploy          # squashed baseline + incremental migrations
bun prisma generate --sql
bun run build                      # nest build -> dist/
bun tsc-alias -p tsconfig.json     # resolve "@/" aliases (the Dockerfile does this)
```

Docker image (multi-stage, runs as non-root `node`, exposes 7777):
`docker build --network host --build-arg DATABASE_URL=$DATABASE_URL -t node-system-be .`
(`DATABASE_URL` is only used by the builder stage). `.github/workflows/build-image.yaml` builds and
pushes to Amazon ECR and needs `vars.AWS_REGION`, `vars.ECR_REPOSITORY`, `secrets.AWS_IAM_ROLE_ASSUME`
(OIDC role) and `secrets.BUILD_DATABASE_URL` (a migrated, disposable database reachable from runners);
adapt it to your registry.

The production image does **not** contain the Prisma CLI config: run migrations from a source checkout or
build-stage container before starting a new version.

```bash
# development
docker compose -f docker-compose.example.yml up -d postgres redis
cp config.example.yaml config.yaml
TZ=Etc/GMT+2 bun run dev
# production (compiled), from the directory containing config.yaml
TZ=Etc/GMT+2 node dist/main.js
```

`docker-compose.example.yml` starts Postgres 16, Redis with `--notify-keyspace-events KEA --appendonly yes`
and the backend with `TZ` set and `config.yaml` mounted. Health check: `GET /api/healthcheck` returns
`Ping!`. Boot logs: Redis connect and keyspace config, Postgres connect, `Loaded subgraph endpoints`, then
the server. Boot fails if Redis or Postgres is unreachable or the config is invalid; wrong `ether` or
`ipfs` values only show up at the first hourly or daily run.

## Seeding a project

A fresh database has no projects, admins, subgraph endpoints or workload versions, and nothing works until
you add them. `prisma/seed/example-project.sql` is a documented example (not wired into startup); run it
once after `migrate deploy`:

```bash
psql "$DATABASE_URL" -f prisma/seed/example-project.sql
```

| Table | What to insert |
|---|---|
| `projects` | `nft` (license contract, **lower-case**, must equal the subgraph's `nft`), `code` (slug, stable: part of every Merkle leaf), `name`, `network` (`ETH`, `BSC`, `POLYGON`, `BASE`), display fields |
| `project_admins` | `(project_nft, address)`: wallets that may publish versions and submit points (publisher, points key). One row per wallet |
| `subgraph_endpoints` | `name`, `url`, `token_types` (`ERC721`, `ERC1155`). **Restart the backend afterwards**: read once at boot |
| `project_workload_versions` | `(project_code, version, cid)`: manifest CID. The newest row is what `workload add` returns |

Store all addresses in lower case. Publish later versions with
`PATCH /api/v1/projects/<code>` (body `{"version":"1.0.1","workloadIpfsCid":"<cid>"}`, headers
`X-User-Address: <admin>` and `X-Signature` = EIP-191 signature of the **exact raw body string**); the
[deployer](workloads.md#deployer-github-action) does exactly this. `prisma/seed/main.ts` (Prisma's seed
command) uploads `prisma/seed/*.yaml` manifests to Pinata (`PINATA_JWT` in the environment) and upserts
versions; it needs an existing project row and ships no manifests.

## Cron jobs and timezone

| Job | Schedule | What it does |
|---|---|---|
| Balance sync ERC1155 / ERC721 | every 5 s (ERC721 starts 1 s later) | Reads new balances from every subgraph endpoint into `user_wallet_balances` |
| Hourly snapshot | `0 0 * * * *` (UTC, top of each hour) | Sums uptime of the previous hour per workload, writes `HOURLY` rows (`is_valid=true` for all), uploads JSON to IPFS, calls `submitHourlyRoot` |
| Daily snapshot | `02 02 * * *` (UTC, 02:02) | For every closed day since the last `COMPLETED` summary: sums uptime, applies license limits, writes `DAILY` rows, uploads, calls `submitDailyRoot` |

**Timezone.** `src/main.ts` sets the default zone `Etc/GMT+2` (which is **UTC-2**), and a "day" is
`startOf('day')` in that zone: **02:00 UTC**. That is why the daily job runs at 02:02 UTC. Run with
`TZ=Etc/GMT+2`; the loop that advances days also uses native `Date` arithmetic. The registry's day ids must
stay consistent with this boundary: never change it on a running deployment.

**Daily validity.** Uptimes of the day are sorted descending per workload; for each
`(wallet, project nft)` the license count (snapshotted into `user_license_snapshots`) is decremented per
workload and only workloads that find a balance above zero are `is_valid` and enter the daily tree. A
wallet with one license and three workloads gets only its highest-uptime workload counted. The registry
requires 24 hourly roots before it accepts a daily root.

**Empty periods.** An hourly run with no uptime submits the sentinel root `0x00...01`; a daily run with no
uptime submits nothing and marks the day `COMPLETED`.

**Leaves.** `keccak256(abi.encode(string slug, address nodeId, uint256 uptimeMs))`, tree via
`SimpleMerkleTree.of(leaves)` (OpenZeppelin). The uploaded JSON is the array `[{slug,nodeId,uptime}]`.

### Manual triggers

Require header `x-cron-secret: <server.cronSecret>`:

```bash
# re-run the hourly job for the hour ending at <date>
curl -H "x-cron-secret: $SECRET" "http://host:7777/api/v1/cron/trigger/summarize-uptime?type=hourly&date=2026-01-01T10:00:00Z"
# daily: processes every day from <date> up to yesterday, calling submitDailyRoot for each
curl -H "x-cron-secret: $SECRET" "http://host:7777/api/v1/cron/trigger/summarize-uptime?type=daily&date=2026-01-01T00:00:00Z"
curl -H "x-cron-secret: $SECRET" http://host:7777/api/v1/cron/trigger/sync-token/erc1155     # or erc721
# status of a day (no auth)
curl "http://host:7777/api/v1/cron/uptime-summary?date=2026-01-01T00:00:00Z"
```

### Chain submission failures

In `OnChainService._uploadUptime` the order is: build root, upload JSON to Pinata, send transaction. Any
failure (Pinata, RPC, revert, missing role, no gas, `MissingHourlyRoots`, `IdAlreadyExists`) is logged,
posted to Slack, and processing **continues**:

- nothing is retried automatically and no root, CID or tx hash is stored in the database;
- the daily summary is still marked `COMPLETED` and the points rows are written, so the missed day is not
  revisited; `GET /cron/uptime-summary` therefore does **not** tell you whether the submission succeeded
  (check Slack or contract events);
- a failed hour is simply missing on-chain and the daily root may then be rejected (`MissingHourlyRoots`);
- recovery: fix the cause and re-run. Re-running a **daily** snapshot with `date=D` re-processes every day
  from `D` to yesterday; days already on-chain revert with `IdAlreadyExists` (logged, harmless), so use the
  earliest missed day. An hourly re-run needs `date` = the end of the missed hour;
- transactions are not awaited for confirmation, so a dropped transaction is also invisible.

Keep the submitter wallet funded and monitor Slack or the logs.

## HTTP API

Base path `/api/v1` (one route at `v1.1`), global prefix `/api`. Swagger (when enabled) lists schemas at
`/docs`. There are **no sessions or tokens**; identity is an Ethereum address plus a signature:

- **User-signed:** `X-User-Address: 0x...` and `X-Signature` = EIP-191 (`personal_sign`) of the **raw body**
  by that address.
- **Node-signed:** same, but the signer must be the `:nodeAddress` in the path.
- **Admin-signed:** user-signed and the address must be in `project_admins` of the project.
- **Cron:** `x-cron-secret`.
- Routes that take `X-User-Address` *without* a signature simply trust the header.
- Signatures carry no nonce or timestamp (replayable). Addresses are lower-cased on input.

| Caller | Routes |
|---|---|
| Node CLI / workload manager (node-signed) | `PATCH /nodes/:addr`, `PATCH /nodes/:addr/uptime` (heartbeat `{"workloads":[{"projectCode":"..."}]}`, records caller IP), `POST /nodes/:addr/workloads` (checks the owner's NFT balance minus ONLINE workloads, returns manifest CID), `PATCH /nodes/:addr/workloads/:code`, `DELETE /nodes/:addr/workloads/:code` (signed message is the string `<addr>\|<code>`), `GET /public/cli-constants` |
| Operator / dashboard | `POST /nodes` `{nodeAddress, name}` (user-signed; links a node), `PATCH /nodes/:addr/name`, `DELETE /nodes/:addr` (signs `<addr>`, **no ownership check**), `PATCH /user/settings`, `PUT /notifications`; reads with `X-User-Address` only: `GET /nodes`, `GET /user/settings`, `POST /notifications`, `GET /inventory[...]`, `GET /projects` |
| Project services (admin-signed) | `PATCH /projects/:code` (publish `{version?, workloadIpfsCid?}`), `POST /projects/:code/node-points` (`{date, nodes:[{nodeAddress, point, rewardTokens?}]}`, updates existing `DAILY` rows only) |
| Public reads | `GET /healthcheck`, `/projects/:code`, `/projects/:code/node-uptimes?date=` (valid daily uptimes: input of the reward calculation; `/api/v1.1/projects/:code/node-uptimes` is paginated), `/projects/:code/node-points?date=`, `/nodes/:addr`, `/nodes/:addr/licenses`, `/nodes/:addr/workloads...`, `/distributions/:code[/stats]`, `/stats/*`, `/cron/uptime-summary` |
| Operations | `GET /cron/trigger/summarize-uptime`, `/cron/trigger/sync-token/erc721` and `.../erc1155` (`x-cron-secret`) |

Heartbeats arrive about every 60 s; a gap longer than `config.nodeOfflineThreshold` (90 s) opens a new
session. Because the license limit is per wallet, a wallet can run at most `balance` ONLINE workloads of a
project.

## Data model

| Table | Purpose |
|---|---|
| `projects`, `project_admins`, `project_workload_versions`, `subgraph_endpoints` | Configuration (seeded above) |
| `wallets` | Known wallets, `settings` JSONB (for example alert e-mail) |
| `user_wallet_balances` | `(wallet_address, project_nft)` to license `balance`, with `source_id` (subgraph) and `updated_at` (the sync cursor) |
| `user_license_snapshots` | Balance snapshot per wallet/project/day used by the daily validity step |
| `nodes` | Node (address = node key address), owner wallet, name, version, OS, IP and geo, soft delete |
| `node_uptimes`, `node_workloads`, `node_workload_uptimes` | Node sessions, workloads on nodes, workload sessions |
| `node_workload_points` | Per workload and period: `type` `HOURLY` / `DAILY`, `uptime` (ms), `is_valid`, plus `point` and `reward_tokens` written by the project |
| `point_summaries` | One row per summarized day: `status` `PENDING` / `COMPLETED` / `FAILED`, `retries`, `error`; decides where the daily job resumes |
| `notifications` | Web and e-mail notifications per wallet |

Redis holds only transient "last seen" keys (`node-uptime:<node>`, `workload-uptime:<project>:<node>`);
flush freely, sessions are rebuilt from Postgres on the next heartbeat. Their expiry events drive offline
notifications.

## Known issues

Kept bug-for-bug compatible with production; the full table is in [Security](security.md#backend). Also:
`smtp.port` is ignored, only the first SMTP host is used, subgraph endpoints load once at boot, PostgreSQL
TLS is only enabled when the URL contains `amazonaws.com`, geo lookup uses plain-HTTP `ip-api.com` (free
tier, non-commercial), and the repository has no automated tests.

Migrating data from an LFG export: [Migration from LFG](migration-from-lfg.md#backend-data).

# Self-hosting checklist

This is the ordered path from nothing to a working nodesystem for **one project**. Each step ends with a
"Verify" block; do not move on until it passes, because later steps assume the earlier ones. Component
pages and repository READMEs hold the full config references; this page only says what to do, in what
order, and how to know it worked.

!!! warning "Read first"
    The code runs exactly as it did in production and has known weaknesses
    ([Security](security.md)). Decide before you expose anything to the internet. In particular, a
    backend that answers `200` with an empty workload list makes every operator's workload manager delete
    all workloads (the reference backend now answers `404` for a node it has never registered, but not for a
    registered node without workloads).

```mermaid
flowchart TB
    subgraph A["Infrastructure"]
        direction LR
        S0["0 Decide + keys"] --> S1["1 Contracts"] --> S2["2 License subgraph"] --> S3["3 Backend"] --> S4["4 Release CLI + manager"]
    end
    subgraph B["Project services"]
        direction LR
        S5["5 Workload + deployer"] --> S6["6 Reward subgraph<br/>(token path)"] --> S7["7 Master or reward service"] --> S8["8 Dashboard"] --> S9["9 Smoke test"]
    end
    A --> B
```

## 0. Decide and prepare

**Decision 1: points only, or token rewards?** It decides which components you deploy
(overview: [Which parts does your project need?](index.md#which-parts-does-your-project-need)).

| | Points-only project (master-sample) | Token-reward project |
|---|---|---|
| Reward logic | `opensource-nodesys-workload-master-sample` (reads `node-uptimes`, posts `node-points`) | `opensource-nodesale-reward` |
| Needs on chain | uptime registry only | uptime registry **plus** token, `NodeOperatorsReward` (NOR) |
| Subgraphs | license | license **plus** reward subgraph |
| Steps to run | 1a, 2, 3, 4, 5, 7 (master), 8, 9 | all, including 1b and 6, and 7 (reward service) |
| Datastores beyond the backend | MongoDB, Kubo | PostgreSQL for the reward service |

If unsure, start points-only: you can add the token path later. For a local run of either path see
[Local testing](local-testing.md).

**Decision 2: the values below.** Settle them first; they appear in many configs.

| Decision | Notes |
|---|---|
| **Chain** | Any EVM chain for the contracts. The original deployments used BSC; use a testnet (BSC testnet is chain id 97) for the first run |
| **Project code** | Lower-case slug, for example `myproject`. Permanent: it is part of every Merkle leaf, the Kubernetes workload name and the secret name |
| **License NFT** | The ERC721 / ERC1155 contract whose balances limit workloads. Existing project NFT, or a test token for the first run. Note its address and deployment block |
| **One backend, one registry** | The backend config has a single `ether.contractAddress`, so one backend deployment commits to one registry. A registry stores one root per hour and per day for the whole contract, so give each backend its own. One backend can serve several projects (leaves carry the project slug) |
| **Backend URL** | `https://`, public DNS. The workload manager only talks `https://` and resolves names through public / host DNS (no `/etc/hosts`, no `http://localhost`) |
| **Release host** | An HTTPS bucket (S3 or equivalent) that serves CLI archives and the manager's version file. Becomes the CLI's `release-endpoint` |
| **Image registry** | Anonymously pullable (no pull secret is created). The workload manager and your workload image both go there |
| **IPFS** | A Pinata account (the backend uploads uptime JSON through Pinata's v3 files API), a gateway that can serve your manifest CIDs (default `https://ipfs.io`), and an **IPFS upload API** for the deployer (see [Workloads](workloads.md#ipfs-upload-api-contract)) |
| **Services** | PostgreSQL 16, Redis (db 0, keyspace notifications), one machine for the backend (single instance only), Slack and SMTP (optional in practice) |

Generate dedicated keys now (see [Keys and wallets](index.md#keys-and-wallets)). You need the
**addresses** early because they are inserted into the backend database in step 3:

- registry **admin** (multisig on mainnet) and registry **submitter**;
- project **publisher** key (used by CI) and **points** key (used by the master or reward service); both
  addresses go into `project_admins`;
- token path only: NOR admin, **signer** and **claimer**.

## 1. Contracts

### 1a. Uptime registry (always)

Repository: `opensource-merkle_tree_uptime`. Details: [Contracts](contracts.md#uptime-registry).

```bash
git clone --recurse-submodules https://github.com/<you>/opensource-merkle_tree_uptime && cd opensource-merkle_tree_uptime
cp .env.example .env     # RPC_URL, CHAIN_ID, PRIVATE_KEY, ADMIN, SUBMITTER, ETHERSCAN_API_KEY
set -a && source .env && set +a
forge build && forge test
forge script script/Deploy.s.sol:Deploy --rpc-url $RPC_URL --chain-id $CHAIN_ID \
  --private-key $PRIVATE_KEY --broadcast --legacy -vvvv       # --legacy for chains without EIP-1559, e.g. BSC
```

If `ADMIN` differs from the deployer, the script does not grant `SUBMITTER_ROLE`; have the admin run
`grantRole(SUBMITTER_ROLE, SUBMITTER)` (command in the repo README).

**Verify**

```bash
export REGISTRY_ADDRESS=0x...        # printed by the script
cast call $REGISTRY_ADDRESS "hasRole(bytes32,address)(bool)" $(cast keccak "SUBMITTER_ROLE") $SUBMITTER --rpc-url $RPC_URL   # true
```

Fund the submitter wallet with native gas. Keep `REGISTRY_ADDRESS`.

### 1b. Token rewards only: `NodeOperatorsReward` (and a token)

Repository: `opensource-node-operators-reward`. Skip this for points-only projects. Details:
[Contracts](contracts.md#node-operators-reward).

1. Deploy the token (`./deploy_helix_token.sh`) or use an existing ERC20 as `TOKEN_ADDRESS`.
2. Deploy `NodeOperatorsReward` (`./deploy_node_operators_reward.sh`) with admin, signer, claimer and
   token. Note its address (`NOR_ADDRESS`) **and the block number of the deployment** (the reward
   subgraph's `startBlock`).
3. Helix token only: start emission in this order: `startInitialPhase`, then `startGoingPhase` (once) with
   `NOR_ADDRESS` as a participant. Emission accrues in whole UTC days; the first claimable amount appears at
   the next 00:00 UTC. For a plain ERC20, transfer the reward budget to `NOR_ADDRESS`. On BSC add
   `--legacy` to the deploy scripts.
4. The reward service calls `claim<Code>Token()` where `<Code>` is your project code with the first
   letter upper-cased (see the [naming constraint](contracts.md#naming-constraint)).

**Verify**: the `hasRole` checks for `SIGNER_ROLE`, `CLAIMER_ROLE` and the admin, and
`cast call $NOR_ADDRESS "helixToken()(address)"` equals `TOKEN_ADDRESS` (commands in the repo README).

## 2. License subgraph

Repository: `opensource-license-subgraph`. Details: [Subgraphs](subgraphs.md#license-subgraph).

```bash
pnpm install --frozen-lockfile
cp configs/example.json configs/config.json   # network, erc721 / erc1155 lists with address + startBlock
pnpm prepare:subgraph && pnpm codegen && pnpm build
pnpm goldsky:login --token "$GOLDSKY_API_KEY" && pnpm goldsky:deploy myproject-license/v0.0.1   # or local / Graph Studio
```

**Verify**

- Sync reaches the chain head and `_meta { hasIndexingErrors }` is `false`.
- For a wallet that holds licenses, `nodeTotalBalances(where:{user:"0x..."})` (ERC1155) or
  `erc721NodeTotalBalances(...)` equals the on-chain balance.

Note the GraphQL query URL.

## 3. Backend

Repository: `opensource-node-system-be`. Details: [Backend](backend.md).

1. **Services.** PostgreSQL 16 and Redis with `--notify-keyspace-events KEA`. The repository's
   `docker-compose.example.yml` starts both.
2. **Migrate before you build.** The Prisma typed-SQL step needs a migrated database:
   ```bash
   bun install          # no .git (zip download)? use LEFTHOOK=0 bun install
   export DATABASE_URL=postgresql://user:pass@host:5432/db
   bun prisma migrate deploy
   bun prisma generate --sql
   bun run build && bun tsc-alias -p tsconfig.json
   ```
   (or `docker build --network host --build-arg DATABASE_URL=$DATABASE_URL -t node-system-be .`)
3. **Configure.** `cp config.example.yaml config.yaml` and replace every placeholder: `databaseUrl`,
   `redisUrl`, `ipfs.jwt`, `ipfs.gateway`, `ether.rpcUrl`, `ether.privateKey` (the submitter),
   `ether.contractAddress` (`REGISTRY_ADDRESS`), `server.cronSecret`, `server.cookieSecret`,
   `server.corsWhitelist` (your dashboard origin), Slack and SMTP values. Quote every `0x...` value.
4. **Seed the project.** Edit `prisma/seed/example-project.sql` (project row with the license NFT address
   in lower case and your project code, `project_admins` for the publisher and points addresses,
   `subgraph_endpoints` with the URL from step 2, first workload version) and run it with
   `psql "$DATABASE_URL" -f prisma/seed/example-project.sql`. The first workload version needs a
   manifest CID; if you have none yet use any valid CID and replace it in step 5.
5. **Run exactly one instance** with the right timezone:
   `TZ=Etc/GMT+2 node dist/main.js` from the directory containing `config.yaml` (the compose example
   sets `TZ`). The daily boundary is 02:00 UTC; do not change this on a running deployment.
6. Put it behind HTTPS (reverse proxy). Set `server.enableSwagger: false` and `nodeEnv: production`.

**Verify**

```bash
export BACKEND=https://<your-backend-host>                      # used in the rest of this page
curl $BACKEND/api/v1/healthcheck                                # {"status":1,"data":"Ping!"}
curl $BACKEND/api/v1/public/cli-constants                       # {"status":1,"data":{...}}
```

- Startup logs show Redis and Postgres connected and `Loaded subgraph endpoints`. (Endpoints are read
  **once at boot**; restart after changing `subgraph_endpoints`.)
- After a minute, `select * from user_wallet_balances;` contains the wallets that hold licenses.
- Redis: `redis-cli CONFIG GET notify-keyspace-events` contains `K`, `E` and `A`.

## 4. Build and release the CLI and workload manager in your fork

Repositories: `opensource-node-cli`, `opensource-node-system-workload-management`. Details:
[Node CLI](node-cli.md#releasing-in-your-fork) and [Workload manager](workload-manager.md#releasing-in-your-fork).
They share one release host. **Release the workload manager first**: the CLI reads the manager's image
tag from `<release-endpoint>/workload-manager/stable.txt`.

1. Create the release bucket (HTTPS, public read or CDN) and an AWS user that can write it and push to
   Amazon ECR Public (workflow default; other registries need workflow edits).
2. Workload manager fork, repository variables `ECR_REPOSITORY`, `AWS_BUCKET`, `AWS_REGION`, secrets
   `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`. Push a tag (use a version higher than the last LFG
   release, `v0.3.3`; for example `v1.0.0`).
3. CLI fork, variables `AWS_BUCKET`, `AWS_REGION`, `NODE_SYSTEM_API_ENDPOINT` (your backend URL, compiled
   in), `NODE_SYSTEM_API_ENDPOINT_BETA`, same two secrets. Push a tag (higher than `v0.1.5`).
4. Upload `scripts/install.sh` from the CLI repo to `<bucket>/releases/` by hand (CI does not).
5. Optional: fill the `cli` section of the backend `config.yaml` (`workloadManagerImage`,
   `releaseEndpoint`, `ipfsGateway`) and restart it, so new CLIs learn these values from
   `GET /api/v1/public/cli-constants`. Local settings always win.

**Verify**

```bash
export BUCKET=https://<your-bucket-host>                     # the CLI's release-endpoint
curl $BUCKET/workload-manager/stable.txt                     # your manager tag, nothing else
curl $BUCKET/releases/stable.txt                             # your CLI tag
docker pull public.ecr.aws/<alias>/node-system/workload-manager:<tag>    # anonymous pull works
```

Download the CLI archive from the GitHub release (or via `install.sh`) and run `lfgnode version`.

## 5. Workload repository and deployer

Repositories: `opensource-nodesys-workload-template`, `opensource-nodesys-deployer`. Details:
[Workloads](workloads.md).

1. Provide the **IPFS upload API** (`POST {data}` returns `{data: cid}`, pinned). The master sample's
   `POST /api/v1/static/text` satisfies it (step 7), or run a small service as shown in the deployer
   README.
2. Fork the deployer and tag it; fork the template as your workload repository.
3. In the workload repo set secret `PROJECT_PUBLISHER_PRIVATE_KEY`, variables `NODESYS_BASE_URL`,
   `NODESYS_IPFS_API_URL`, `PROJECT_CODE` (if it is not the repository name) and optional registry
   settings; point the workflow's `uses:` line at your deployer fork; set `MASTER_URL` in
   `manifest-template.yaml`.
4. Make the image package public, push a tag.

**Verify**

- The workflow prints the manifest CID and the backend response.
- `select project_code, version, cid from project_workload_versions order by created_at desc;` shows the new
  row. (This is the row nodes receive on `workload add`.)
- `curl https://<your-gateway>/ipfs/<cid>` returns the completed Deployment YAML (namespace `lfg-user`,
  container `main-container`).
- The image pulls anonymously.

## 6. Token path only: reward subgraph

Repository: `opensource-node-reward-subgraph`. Details: [Subgraphs](subgraphs.md#reward-subgraph).
Do this **before** starting the reward service, which needs the subgraph URL.

```bash
cp configs/example.json configs/config.json     # network, projectCode, address (NOR), startBlock (NOR deploy block)
pnpm install --frozen-lockfile && pnpm prepare:subgraph && pnpm codegen && pnpm build
pnpm goldsky:deploy myproject-node-reward/v0.0.1
```

**Verify**: synced, no indexing errors; after a test claim in step 9,
`nodeOperatorTotalRewardClaims(where:{user:...}).nonce` equals `cast call $NOR "userNonce(address)(uint256)" $USER`.

## 7. Master or reward service

Details: [Master and rewards](master-and-rewards.md).

**Points only (or any project needing a master):** run `opensource-nodesys-workload-master-sample`
(`cp app.example.yaml app.yaml`, set backend URL, the points private key, `projects[]`, MongoDB, Pinata, Kubo).
Set `daily_submit_at` **after** the backend's 02:02 UTC daily job (for example `02:30:00`; the example
config's `02:00:30` is too early).

**Token rewards:** run `opensource-nodesale-reward`: Postgres, `.env` and `.env.yaml` (three keys per
project, admin address), `prisma/migration.sh`, seed one `projects` row and one `projects_systems` row
(`api_url` is the backend **base host**, `gain_point` 1), start with `TZ=UTC`, one instance only. No
Alchemy account is needed if you set `rpc: { <chainId>: "<url>" }` in `.env.yaml`.
Grant the signer and claimer addresses their roles on `NodeOperatorsReward` (step 1b), and make sure the
points key (`system`) is in `project_admins` (step 3).

**Verify**

- Master: logs show `submitter started`; after the daily time and once the backend's daily summary exists,
  `node points submitted`.
- Reward service: logs show the 00:05 / 01:00 / 02:10 jobs for your project;
  `select * from projects_ledgers;` reaches `POINTED`; `GET /api/v1/projects/<sale>/wallets/<addr>/operators`
  answers for an earning wallet.

## 8. Dashboard

Repository: `opensource-nodesys-ui`. Details: [Dashboard](dashboard.md).

```bash
cp .env.example .env.local     # NEXT_PUBLIC_BACKEND_URL, NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID at minimum
pnpm install --frozen-lockfile && pnpm build && pnpm start
```

Add the dashboard origin to the backend's `server.corsWhitelist` and restart the backend. Replace the
bundled LFG logos. Review the Helix/BSC special cases in `src/configs/projects.ts` and the
`/verify-uptime` slug list, both hard-coded (a new project code cannot be selected there until you edit the
list). Set `NEXT_PUBLIC_LFG_UPTIME_CONTRACT_ADDRESS` before building. If port 3000 is taken by the backend,
use `pnpm start -p <port>`.

**Verify**: connect a wallet, the project appears in the inventory; adding a node (signature prompt)
returns success.

## 9. End-to-end smoke test

Run on a testnet with a test license NFT. It exercises every arrow of the architecture except the daily
steps, which need a full day (see the end).

1. **Operator machine** (Docker running; backend, release host and registry reachable over HTTPS):
   ```bash
   export LFGNODE_API_ENDPOINT=$BACKEND
   export LFGNODE_RELEASE_ENDPOINT=$BUCKET
   export LFGNODE_WORKLOAD_MANAGER_IMAGE=public.ecr.aws/<alias>/node-system/workload-manager
   lfgnode config private-key generate
   lfgnode info                       # note the node address
   ```
2. **Hold a license.** Mint the test license NFT to your owner wallet. Within a few seconds of the
   subgraph indexing it the backend copies the balance:
   `select * from user_wallet_balances where wallet_address = '<owner, lower-case>';`.
3. **Link the node** (the owner wallet signs the raw body; dashboard "Add node" does the same):
   ```bash
   NODE=<node address from lfgnode info, lower-case>
   OWNER=<owner wallet address>; OWNER_KEY=<owner wallet private key, test wallet only>
   BODY='{"nodeAddress":"'$NODE'","name":"smoke"}'
   SIG=$(cast wallet sign --private-key $OWNER_KEY "$BODY")
   curl -X POST $BACKEND/api/v1/nodes -H 'content-type: application/json' \
     -H "X-User-Address: $OWNER" -H "X-Signature: $SIG" --data-raw "$BODY"
   ```
4. **Start and add the workload**:
   ```bash
   lfgnode start
   lfgnode workload add myproject
   lfgnode status && lfgnode workload status myproject
   kubectl --context k3d-lfg-node -n lfg-user get pods     # lfg-workload-manager, ipfs, myproject
   ```
5. **Uptime is recorded.** After 2 to 3 minutes `lfgnode status` shows `OnlineToday` growing and
   `node_workload_uptimes` has an open session for the node. Container
   logs of the workload show the heartbeat; workload manager logs show the 60 s report.
6. **Hourly root on chain.** Trigger the job (or wait for the next full hour):
   ```bash
   CRON_SECRET=<server.cronSecret from the backend config.yaml>
   curl -H "x-cron-secret: $CRON_SECRET" \
     "$BACKEND/api/v1/cron/trigger/summarize-uptime?type=hourly&date=$(date -u +%Y-%m-%dT%H:00:00Z)"
   ```
   The backend log does **not** print the transaction hash: get it from the registry's `HourlyRootSubmitted`
   event or the explorer. Check the registry: the id of the slot
   is derived from `block.timestamp - 2h` (see [Contracts](contracts.md#leaf-ids-and-time-rules)):
   ```bash
   t=$(( $(date -u +%s) - 7200 )); day=$((t/86400)); h=$(( (t/3600) % 24 ))
   [ $h -eq 0 ] && ID=$(( (day-1)*100 + 24 )) || ID=$(( day*100 + h ))
   cast call $REGISTRY_ADDRESS "rootExists(uint256)(bool)" $ID --rpc-url $RPC_URL      # true
   ```
7. **New version rollout.** Push another workload tag; within about 10 minutes the manager patches the
   Deployment and `lfgnode workload status myproject` shows the new version.
8. **Daily steps (next day).** After 02:02 UTC the backend's daily job writes `DAILY` rows and calls
   `submitDailyRoot` (it requires all 24 hourly roots of that day, so a smoke test that did not run all
   day must backfill the missing hours as registry admin with `backfillRoot`, or wait for a clean day).
   `GET /api/v1/cron/uptime-summary?date=<ISO>` returns `COMPLETED`. Then the master posts points or the
   reward service credits and signs a voucher; a test claim on `NodeOperatorsReward` moves the reward
   subgraph's nonce.
9. **Teardown**: `lfgnode down` (the backend must be reachable, see [Node CLI](node-cli.md#when-the-backend-is-down)).

A local variant that compresses the daily steps into minutes (backfill, manual cron, anvil fork with time
warp) is in [Local testing](local-testing.md).

If something is missing from these steps, the repository READMEs are authoritative; please fix the docs.

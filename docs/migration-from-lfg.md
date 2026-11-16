# Migration from the LFG-hosted nodesystem

This page is for **projects and operators that used the LFG-hosted service** (Valley, Helix, FlipIt,
TikTrix, magnetoHR, Flora). LFG is shutting the hosted service down; from then on every project runs its
own copy ([Self-hosting checklist](self-hosting.md)). If you never used the hosted service you can skip
this page.

!!! danger "Legacy infrastructure: do not build on it"
    Every LFG host, bucket and registry listed [at the end of this page](#legacy-endpoints-that-stop-working)
    is **legacy and going away**. LFG intends to keep the domains and bucket registered for a period after
    shutdown only so that nobody else can take them over, not as a service. Do not point anything new at them
    and remove them from your configs.

## What changes, in short

- Binaries and images already installed on operator machines (CLI up to `v0.1.5`, workload manager up to
  `v0.3.3`) have LFG hosts **compiled in**. After the shutdown they stop working. Every operator needs one
  manual update to **your** CLI build ([Operator migration](#operator-migration)).
- Uptime is no longer recorded by LFG after the shutdown date. Operators' nodes keep running but earn
  nothing until they report to your backend.
- Things that LFG ran for you (points calculation for Valley-style projects, token distribution and claim
  vouchers for Helix and FlipIt, the uptime Merkle commitments) must now be run by you, starting the day after
  the last LFG run.
- LFG's plan is to answer `410` on every route of the old backend (never `200` with an empty list, which
  would make the old workload managers delete all workloads). A non-200 is safe for the manager.
- The reference backend now returns `404` (code `1012`, "Node address is unknown to this backend") on
  `GET /nodes/<addr>/workloads` for a node it never registered, so operator nodes that switch to a new
  backend keep their workloads as long as the node is unknown there. Registered nodes and soft-deleted nodes
  still get `200`.

## What LFG hands over

| Item | What you receive | What you do with it |
|---|---|---|
| **Workload repository** | Transfer of your GitHub repository (the template-based workload repo) to your organization | Accept it; if you rename it, set `PROJECT_CODE` in the workflow (it defaults to the repository name) |
| **Production manifest CID** | The CID(s) in `project_workload_versions` that nodes currently run | Pin the content on your IPFS provider; insert the rows into your backend ([Backend data](#backend-data)); publish a new version soon with your registry's image |
| **Workload images** | They live in LFG's Amazon ECR Public (`public.ecr.aws/lfg/node-system-workload/<repo>`); the current tag is yours to copy | Copy the current tag into your own registry, change CI (`ECR_REPOSITORY` / `IMAGE_REPOSITORY`, registry login, deployer `uses:` line and `baseurl`) and publish a new version. LFG keeps its repository while nodes may still pull |
| **Publisher key** | Already in your repository's secrets (`PROJECT_PUBLISHER_PRIVATE_KEY`) | Add its address to `project_admins` on your backend |
| **Backend data export** | SQL dump or CSV per project (tables in [Backend data](#backend-data)) | Restore into your database |
| **Merkle history** | The hourly and daily JSON files from LFG's Pinata. Roots and CIDs are also on-chain in LFG's registry, but the files only exist on Pinata | Keep copies; see [Merkle history](#merkle-history) |
| **Reward database dump** (Helix, FlipIt) | `pg_dump` of the reward service database: per-wallet `unclaim`, `claimed`, `nonce` | [Reward database](#reward-database-helix-and-flipit) |
| **On-chain roles** (Helix, FlipIt) | Role transfer on `NodeOperatorsReward` and token ownership | [Role transfer](#nodeoperatorsreward-role-transfer) |
| **Master configuration** (Valley) | The master's configuration with secrets removed | [Points projects](#points-projects-valley-style) |
| **Links** | Open-source repositories and these docs | |

What LFG does **not** hand over: running infrastructure (backend, databases, dashboards, subgraphs on LFG's
Goldsky, Pinata account), and `SUBMITTER_ROLE` on its registry (every backend you run must deploy its own,
otherwise two backends compete for the single root per hour and day). Whether personal operator data in the
export (alert e-mail addresses in `wallets.settings`, node IP and geo data) is included depends on what LFG
states in your handover: if you receive it and do not need it, clear it.

## One node per backend

A node is bound to one backend, and `lfgnode` allows **one node per machine**: the cluster name `lfg-node`,
the Kubernetes API port `6445` and the config directory are fixed ([Fixed names](fixed-names.md)). The
consequence for operators who earned in several LFG projects:

| Setup after LFG | Operator needs |
|---|---|
| Projects share **one** self-hosted backend (it is multi-project: leaves carry the slug, one registry serves all of them) | One CLI, one node, many workloads (`lfgnode workload add <project>` per project) on one machine |
| Each project runs **its own** backend (for example Valley and FlipIt separately) | **One machine or VM per backend**, each with its own `api-endpoint`, node key and cluster |

If you can choose, run a shared backend: operators then change nothing except the endpoint, and keep a single
node for all projects. If projects separate, tell multi-project operators early that they need extra machines.

## Steps for every project

1. **Deploy your stack** in the order of the [Self-hosting checklist](self-hosting.md): registry, license
   subgraph (for your existing license NFT), backend, your forks of CLI and workload manager.
2. **Load your data** into the new backend ([Backend data](#backend-data)).
3. **Publish workload releases from your own CI**: new registry, deployer fork with `baseurl` and
   `ipfs_api_url`, a new version. Leave `enable_datagram` at its default `"false"`: the backend removed the
   datagram integration (projects that had it enabled, such as Valley, FlipIt and Flora, should drop it). If
   your manifest's `MASTER_URL` pointed at an LFG host (for example `workload-template-api.lfg.inc`) replace
   it with your master.
4. **Migrate operators** ([Operator migration](#operator-migration)).
5. Run your points or reward service ([below](#by-project-type)).

### Backend data

If LFG gives you a `pg_dump` of your project's data (full detail in the backend README, "Migrating data from
an LFG export"):

1. Create an empty database and restore the **full dump** (schema, data and `_prisma_migrations`) with
   `psql`, then `bun prisma migrate deploy`: migrations the dump recorded are skipped, newer ones apply. For a
   data-only dump, run `migrate deploy` on the empty database first and load data with triggers disabled
   (`pg_restore --disable-triggers`).
2. The export may contain rows of other projects. Delete the other `projects` rows (cascades to admins and
   versions) and their `node_workloads`; remove orphan nodes and wallets if you wish. **Do not drop wallets
   and nodes that belong to your operators.**
3. Review and replace `project_admins` (your wallets: publisher, points key), `subgraph_endpoints` (your
   license subgraph) and `project_workload_versions` (CIDs must resolve through **your** gateway and
   pinning).
4. `user_wallet_balances.updated_at` is the sync cursor per `source_id`. If your new subgraph was indexed
   independently, either set `source_id` to the new endpoint id only when `updated_at` is consistent with
   it, or `TRUNCATE user_wallet_balances` so it re-syncs from 0.
5. `point_summaries` decides where the daily job resumes (after the latest `COMPLETED.to`). A registry you
   deploy starts empty and will not accept ids or hourly roots for past days, so past days fail submission
   (logged, not retried). Delete or keep summaries accordingly.
6. Redis starts empty; heartbeats resume sessions from Postgres.
7. Node CLIs must point to your API (and your `cli-constants`); node keys and wallets are unchanged, so nodes
   keep their identity.

Tables the export normally contains for your project: `projects`, `project_admins`,
`project_workload_versions`, `node_workloads`, `nodes` and `wallets`, `node_workload_uptimes`,
`node_workload_points` (DAILY and HOURLY), `user_wallet_balances`, `user_license_snapshots`. Nodes of the
internal uptime test bot are removed before export.

### Merkle history

LFG's registry and its roots stay on-chain and verifiable forever (the registry LFG used for the hosted
service is `0x4123933f0250b2966008878dccc00663444be746`, per LFG's shutdown plan; confirm the chain and
address in your handover notes and on the explorer).
Each hourly and daily JSON file contains the leaves of **all** projects that shared that registry. The files
are the data the on-chain CIDs pointed to; Pinata access ends with the shutdown, so keep your copies safe. If
you want to serve them again, add them to your IPFS provider; the resulting CID matches the on-chain one
only if your tool uses the same CID version and chunking as Pinata did, which is not guaranteed. The
dashboard's `/verify-uptime` page and the `scripts/compute-and-upload.js` helper verify proofs against a
file and the registry without any LFG service.

## By project type

### Points projects (Valley-style)

LFG ran a master that computed Valley's points every day against LFG's backend. That stops. Run
`opensource-nodesys-workload-master-sample` against your backend (or your own service using `node-uptimes`
and `node-points`), with the same `uptime_threshold` and `user_gain_point` you used before (Valley and FlipIt
used `uptime_threshold: 21600000`, six hours); register its key in `project_admins`. Your workload's
`MASTER_URL` must point at your master, and the master needs MongoDB, a Kubo node and a Pinata JWT
([Master and rewards](master-and-rewards.md#workload-master-sample)). Points for days missed during the
migration are **not backfilled** automatically: decide with your community how to close the books (for
example from the exported `node_workload_points`), or call the endpoints yourself.

### Token reward projects (Helix and FlipIt style)

Needs the [reward stack](master-and-rewards.md#nodesale-reward): `NodeOperatorsReward` (new or transferred),
reward subgraph, nodesale-reward, and a claim frontend. Do the
[reward database](#reward-database-helix-and-flipit) and
[role transfer](#nodeoperatorsreward-role-transfer) migration below. Deploy your own reward subgraph (the
LFG-hosted one, `nodereward-prod-bsc` on LFG's Goldsky, goes away).

### Projects with their own master (TikTrix, magnetoHR, Flora style)

Your workload and master infrastructure already belong to you; you only need the backend, registry,
subgraph and release pipeline, and to repoint `MASTER_URL`, `baseurl` and the image registry in your
workload repository's CI. If your service reads uptime or posts points, change its base URL to your backend
and register its signing key in `project_admins`. Flora: if you published the extra project
`legacyconnoisseur`, decide whether it is still needed before seeding it.

### Reward database (Helix and FlipIt)

The hosted reward database is the **only record** of each operator's unclaimed balance
(`projects_wallets.unclaim`) and claim nonce. Procedure (from the nodesale-reward README):

1. **Freeze claims** during cutover (turn off the claim frontend or the old API).
2. **Dump and restore** into an empty PostgreSQL database (schema `public`), keeping `_prisma_migrations`:
   `pg_dump --no-owner --no-privileges "$OLD_URL" > reward.sql` (done by the old owner), then
   `psql "$NEW_URL" -f reward.sql`; check with `bunx prisma migrate status --schema ./prisma`.
3. **Keep only your project**: `DELETE FROM wallets_histories WHERE sale <> '0x<your-sale>';` then
   `DELETE FROM projects WHERE sale <> '0x<your-sale>';` (run the first statement first: that foreign key is `RESTRICT`).
4. **Repoint the project row**: `projects.subgraph_url`, `contract_addr` / `operator_addr` if you redeployed,
   `projects_systems.api_url` (**base host** of your backend) and `link`.
5. **Reconcile before opening claims.** Start the service; the subgraph sync replays claim events newer than
   `max(projects_wallets.syncing_at)`. Compare `SUM(unclaim)`, `SUM(claimed)` and nonces with the contract or
   your subgraph for sample wallets, and make sure the last pre-cutover day was processed
   (`projects_ledgers.status`). **Never re-run a day that is already `POINTED`.**
6. Do the role transfer below, put the new keys in `.env.yaml`, and add the new `system` address to
   `project_admins`.

Never leave the old and the new service running together for one project: both would run the daily jobs and
sign vouchers.

### NodeOperatorsReward role transfer

The current admin (LFG) transfers control:

```bash
# new admin multisig gets DEFAULT_ADMIN_ROLE (0x00..00); new signer and claimer get their roles
cast send $NOR "grantRole(bytes32,address)" 0x0000000000000000000000000000000000000000000000000000000000000000 $NEW_ADMIN --rpc-url $RPC_URL ...
cast send $NOR "grantRole(bytes32,address)" $(cast keccak "SIGNER_ROLE")  $NEW_SIGNER  --rpc-url $RPC_URL ...
cast send $NOR "grantRole(bytes32,address)" $(cast keccak "CLAIMER_ROLE") $NEW_CLAIMER --rpc-url $RPC_URL ...
# then the previous admin revokes / renounces the old addresses
cast send $NOR "revokeRole(bytes32,address)" $(cast keccak "SIGNER_ROLE")  $OLD_SIGNER  --rpc-url $RPC_URL ...
```

Vouchers signed by the old signer stay valid until their 10-minute deadline or until the role is revoked.
Alternative: the admin `emergencyWithdraw`s the token balance to a `NodeOperatorsReward` you deploy, in which
case `nonce` / `unclaim` must be migrated consistently with the new contract's nonces. Ownership of the
token contract (for the UUPS Helix token, its `owner`) and of the subgraph deployment are separate
resources to take over. The hosted programs' `NodeOperatorsReward` contracts, per LFG's shutdown plan, are Helix
`0xbD55E9840a0dA7fafed7410fa902179422A69A15` and FlipIt `0x760da22cC069Bb916f1f30B7e400Bb23c607D674` (BSC;
verify on the explorer).

## Operator migration

What an operator does, once your project has released its CLI and workload manager and told them your
endpoints:

1. **Install your CLI build** (download from your release page, or your `install.sh`). Old binaries have LFG
   hosts compiled in.
2. **Set your endpoints**: `LFGNODE_API_ENDPOINT` (or `api-endpoint` in `config.yaml`), and if your build did
   not compile them in or the backend does not serve `cli-constants`, `LFGNODE_RELEASE_ENDPOINT` and
   `LFGNODE_WORKLOAD_MANAGER_IMAGE`. Put them in your shell profile or `config.yaml` so `lfgnode update` keeps
   working.
3. **Recreate the manager**: `lfgnode stop && lfgnode start`. This recreates the workload manager Deployment
   with your image and variables. The node key, node address, cluster and workloads are kept: no `down`, no
   key change. Old managers keep polling LFG hosts, so this manual step is needed on every node; do not rely
   on the old self-update chain.
4. **Check the node on your backend**: `lfgnode status`, `lfgnode licenses`, `lfgnode workload list`.
5. **Workloads.** If you imported `nodes`, `wallets` and `node_workloads` into your backend, the existing
   workloads continue and are updated to your next version. If you did **not**, the new backend does not know
   the node: link it first (dashboard "Add node" or `POST /api/v1/nodes` with the owner wallet) **before**
   re-adding workloads with `lfgnode workload add <project-code>`. While the node is unknown the backend
   answers `404` (code `1012`) and the manager keeps the existing workloads; once the node is linked but has
   no workloads, the answer is `200` with `[]`, and the manager deletes local workloads (hence: link, then
   add immediately). Test with one node before moving everyone
   ([Security](security.md#workload-manager)).
6. **Dashboard and claims** move to your project's URLs.

Existing workload manifests pinned on LFG's gateway by CID are content-addressed: any gateway that has the
content can serve them, so make sure **your** `IPFS_GATEWAY` can (pin them with your IPFS provider).

## Legacy endpoints that stop working

!!! danger "Legacy, shutting down"
    Do not configure, document or depend on any of the following. They are listed so you can find and remove
    them (grep your repositories, manifests, CI variables and docs).

| Legacy endpoint | Was used for | Replace with |
|---|---|---|
| `nodesystem-api.lfg.inc` | Production backend API | your backend (`api-endpoint`, `NODE_SYSTEM_ENDPOINT`, `baseurl`) |
| `dev-api.nodesystem.lfg.inc`, `dev-nodesystem-api.lfg.inc` | Dev backends: default of old deployer releases and of the old workload manager | your backend |
| `nodesystem-dashboard.lfg.inc`, `dev-nodesystem-dashboard.lfg.inc`, `dev-dashboard.nodesystem.lfg.inc` | Dashboards | your [dashboard](dashboard.md) |
| `workload-template-api.lfg.inc` | LFG-hosted master used as `MASTER_URL` in some manifests | your master |
| `lfgnode-cli.s3.ap-southeast-1.amazonaws.com` (S3 bucket `lfgnode-cli`) | CLI archives, `install.sh`, `stable.txt` / `unstable.txt`, manager version file | your release bucket (`release-endpoint`) |
| `public.ecr.aws/lfginc/node-system/workload-manager` | Workload manager image | your image (`workload-manager-image`) |
| `public.ecr.aws/lfg/node-system-workload/<repo>` | Project workload images | your registry |
| `ipfs-ops-api.rep.run` | Manifest upload and gateway used by old deployer and manager releases | your IPFS upload API (`ipfs_api_url`) and gateway (`IPFS_GATEWAY`) |
| `lfg.mypinata.cloud`, `nodesale.mypinata.cloud` | Pinata upload and gateway (backend uploads, CLI manifests) | your Pinata account and gateway (`ipfs.url`, `ipfs.jwt`, `ipfs.gateway`) |
| LFG-hosted subgraphs on Goldsky (for example `nodereward-prod-bsc`) | License balances and claim events | your [subgraphs](subgraphs.md) |
| LFG's registry and `NodeOperatorsReward` roles | Hosted-service on-chain commitments and payouts | your own registry; transferred NOR roles |
| `support.lfg.inc` help-center articles ("What is a Node", ...) | Old operator documentation | these docs and your project's own |

Old releases published at these locations are not signed or verified; do not install them after the
shutdown.

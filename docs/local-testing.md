# Local test / smoke test

The path below was run end to end (BSC testnet plus local Docker, 2026-11-16). It is a checklist of what
differs from a production setup; for the production order see [Self-hosting](self-hosting.md). Anything not
listed here behaves as in the component pages.

## Checklist

| # | Topic | What to do |
|---|---|---|
| 1 | Ports | The dashboard default port 3000 clashes with the backend: run the dashboard with `pnpm start -p <port>`. Remap compose Postgres (for example `5433:5432`) and publish Redis on `127.0.0.1:6379` ([Backend](backend.md#build-migrate-run)) |
| 2 | IPFS for the backend | Pinata v3 files API only. Run a tiny shim implementing `POST /v3/files` (multipart upload, returns `{ "data": { "cid": "..." } }`) on top of kubo and point `ipfs.url` at it ([contract](backend.md#ipfs-upload-contract)) |
| 3 | Kubo gateway | Local kubo redirects to a subdomain gateway: set `Gateway.PublicGateways` for `localhost` with `UseSubdomains=false` |
| 4 | Manager needs https | TLS proxy in front of the backend plus a public wildcard name such as `<ip>.sslip.io`; a self-signed cert needs `NODE_TLS_REJECT_UNAUTHORIZED=0` on the manager Deployment, patched **after** `lfgnode start` (it rewrites the Deployment) |
| 5 | Images | k3d registry mirror in `cluster.yaml` for an HTTP registry ([example](node-cli.md#known-issues-and-local-test-tips)) |
| 6 | Separate HOME | `HOME=$PWD/testhome lfgnode ...` keeps the test config apart from a real one |
| 7 | CLI flags | Do not pass `-v`; only one `lfg-node` cluster per machine; leave `release-endpoint` empty (self-update off): the CLI accepts `http://`, but self-update with an http release URL breaks |
| 8 | Link the node | `POST /api/v1/nodes` signed by the **owner** wallet (EIP-191 over the raw body, headers `X-Signature` and `X-User-Address`; `cast wallet sign` works) **before** `workload add` |
| 9 | Workload image | Template Dockerfile: pin `pnpm@9`, pass **both** `PROJECT_CODE` and `VERSION` build args, one image per project code ([Workloads](workloads.md)) |
| 10 | Release | Run the deployer locally with `INPUT_*` env vars; the master sample's `POST /api/v1/static/text` satisfies `ipfs_api_url` ([Workloads](workloads.md#running-the-deployer-locally)) |
| 11 | Apple Silicon | graph-node image is `linux/amd64` only (emulated, slow); a local graph-node hosts both subgraphs, no Goldsky needed |
| 12 | `bun install` | No `.git`: `LEFTHOOK=0 bun install`. Reward Dockerfile on arm64: add `--backend=copyfile` |

!!! warning "Unknown node and the manager"
    The workload manager deletes local workloads on `200` with an empty list. The backend answers `404`
    (code `1012`) for a node it never registered, so link the node first and add the workload right after.

## Uptime and the hourly / daily roots

1. After 2 to 3 minutes of running, `lfgnode status` shows `OnlineToday` growing.
2. Trigger jobs by hand with the cron secret
   (`GET /api/v1/cron/trigger/summarize-uptime?type=hourly|daily&date=<ISO>`, header `x-cron-secret`).
3. The hourly cron (`0 * * * *`) submits even with no data (placeholder root `0x00...01`) and consumes the
   slot; a manual trigger in the same hour reverts with `IdAlreadyExists` (roots are write-once).
4. A daily root needs all 24 hourly roots of that day. On a fresh registry the admin fills gaps with
   `backfillRoot`. For a **same-day** daily root you need backfill plus uptime data of the **prior** day.
5. Day boundary is 02:00 UTC (`TZ=Etc/GMT+2`, daily cron 02:02 UTC). In verification SQL quote the camelCase
   columns (`"projectCode"`, `"startTime"`).
6. The dashboard origin must be in `server.corsWhitelist`; reload the dashboard after switching wallet. Set
   `NEXT_PUBLIC_LFG_UPTIME_CONTRACT_ADDRESS` at build time for `/verify-uptime`.

`scripts/compute-and-upload.js` accepts both `uptime` (backend JSON) and `uptimeAmount`
([Contracts](contracts.md#manual-submit-proofs-and-troubleshooting)).

## Points-only path (master sample)

There is no manual trigger. Set `daily_submit_at` a minute ahead (restart), and make sure yesterday's `DAILY`
rows exist. In production use a time such as `02:30:00`, after the backend's 02:02 UTC job
([timing note](master-and-rewards.md)).

## Token-reward flow on an anvil fork

Emission accrues in whole UTC days from 00:00 UTC of the day `startGoingPhase` is called, so a same-day test
reads 0 and waiting a day is slow. Fork and warp instead (verified: the whole flow runs in minutes).

```bash
anvil --fork-url <rpc> --chain-id <id>
```

1. Deploy token (proxy), NOR, then `startInitialPhase` and `startGoingPhase` with NOR as participant (BSC
   deploy scripts need `--legacy`; extra args pass through: `./deploy_helix_token.sh --legacy`).
2. In nodesale-reward `.env.yaml` add `rpc: { <chainId>: "http://127.0.0.1:8545" }` and any non-empty
   `alchemy.apiKey`. The chain id must still be in `CHAIN_ID_TO_NETWORK`.
3. Warp time with `evm_increaseTime` or `anvil_setNextBlockTimestamp` (then mine a block) so a full UTC day
   has passed.
4. The 00:05 / 01:00 / 02:10 jobs have no manual trigger. Run them with a Nest application-context script
   that calls `TaskPoint.getPoint`, `TaskPoint.claimPoint` and `TaskReward.calculateRewards` (optional
   testing aid). There is one ledger row per project per day: to re-run a day, delete that day's
   `projects_ledgers` and `projects_rewards` rows first.
5. `claim<Code>Token()` must exist on NOR: the reference contract only has `claimHelixToken`; rename for any
   other code ([naming constraint](contracts.md#naming-constraint)).
6. The voucher deadline is the server's wall clock plus 600 s, so on a time-warped fork the voucher can look
   expired. After the operator's `claim` on NOR, the reward subgraph's `Claim` event lowers `unclaim` and
   moves the nonce ([flow diagram](master-and-rewards.md#reward-flow-and-emission-setup)).

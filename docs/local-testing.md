# Local test / smoke test

This page lists what differs from a production setup when you run the whole system on a testnet and local
Docker (it was verified end to end on BSC testnet). Follow the order of [Self-hosting](self-hosting.md) and
apply the items below on the way; anything not listed here behaves as described in the component pages.
The last sections show how to get hourly and daily roots, points and token claims in minutes instead of days.

## Checklist

| # | Topic | What to do |
|---|---|---|
| 1 | Ports | The dashboard default port 3000 clashes with the backend: run the dashboard with `pnpm start -p <port>`. The backend's example compose publishes Postgres on `5433` and Redis on `6379` for a host-run backend ([Backend](backend.md#build-migrate-run)) |
| 2 | IPFS for the backend | Pinata v3 files API only. Run a tiny shim implementing `POST /v3/files` (multipart upload, returns `{ "data": { "cid": "..." } }`) on top of kubo and point `ipfs.url` at it ([contract](backend.md#ipfs-upload-contract)) |
| 3 | Kubo gateway | Local kubo redirects to a subdomain gateway: set `Gateway.PublicGateways` for `localhost` with `UseSubdomains=false` |
| 4 | Manager needs https | TLS proxy in front of the backend plus a public wildcard name such as `<ip>.sslip.io`. CLI `api-endpoint` is the **http** URL, the manager Deployment is patched **after** `lfgnode start` ([recipe](#self-signed-tls-for-the-manager)) |
| 5 | Images | k3d registry mirror in `cluster.yaml` for an HTTP registry. The file must be **pre-created** before the first start ([default content and `registries:` example](node-cli.md#clusteryaml)) |
| 6 | Separate HOME | `HOME=$PWD/testhome lfgnode ...` keeps the test config apart from a real one |
| 7 | CLI flags | Do not pass `-v`; only one `lfg-node` cluster per machine; leave `release-endpoint` empty (supported; self-update off) and put an explicit tag in `workload-manager-image`, for example `ghcr.io/<org>/workload-manager:v1.0.0` ([why](#release-endpoint-for-local-tests)) |
| 8 | Link the node | `POST /api/v1/nodes` signed by the **owner** wallet (EIP-191 over the raw body, headers `X-Signature` and `X-User-Address`; `cast wallet sign` works) **before** `workload add` |
| 9 | Workload image | Template Dockerfile: pin `pnpm@9`, pass **both** `PROJECT_CODE` and `VERSION` build args, one image per project code ([Workloads](workloads.md)) |
| 10 | Release | Run the deployer locally with `INPUT_*` env vars; the master sample's `POST /api/v1/static/text` satisfies `ipfs_api_url` ([Workloads](workloads.md#running-the-deployer-locally)) |
| 11 | Subgraphs | graph-node image is `linux/amd64` only (emulated on Apple Silicon, slow). The reward-subgraph compose publishes Postgres on **5434** and pins graph-node `v0.45.0`; **one graph-node indexes one network**, so a testnet plus an anvil fork needs two graph-nodes on different ports; the license subgraph has no compose file of its own ([Subgraphs](subgraphs.md#local-graph-node-ports-image-tag-networks)) |
| 12 | `bun install` | No `.git`: `LEFTHOOK=0 bun install`. Reward Dockerfile on arm64: add `--backend=copyfile` |

!!! warning "Unknown node and the manager"
    The workload manager deletes local workloads on `200` with an empty list. The backend answers `404`
    (code `1012`) for a node it never registered, so link the node first and add the workload right after.

## Self-signed TLS for the manager

The CLI is a Go binary that rejects self-signed `https://` and has no insecure flag; the manager only talks
https. The recipe that worked, with the backend on `<httpPort>` and a TLS proxy (any: Caddy, nginx, a node
script) with a self-signed certificate on `<tlsPort>` in front of it:

```bash
IP=<your LAN or public IP>            # <ip>.sslip.io resolves to that IP through public DNS

# 1. CLI: plain http endpoint (the CLI is not the problem once it talks http)
export LFGNODE_API_ENDPOINT=http://$IP.sslip.io:<httpPort>
export LFGNODE_WORKLOAD_MANAGER_IMAGE=ghcr.io/<org>/workload-manager:v1.0.0   # explicit tag, no release endpoint
lfgnode start

# 2. AFTER start (start rewrites the Deployment): point the manager at the TLS proxy, accept the
#    self-signed certificate, and drop RELEASE_URL (trailing "-" removes the variable)
kubectl --context k3d-lfg-node -n lfg-user set env deploy/lfg-workload-manager \
  NODE_SYSTEM_ENDPOINT=https://$IP.sslip.io:<tlsPort> \
  NODE_TLS_REJECT_UNAUTHORIZED=0 \
  RELEASE_URL-
kubectl --context k3d-lfg-node -n lfg-user rollout status deploy/lfg-workload-manager
```

Re-run step 2 after every `lfgnode start` or `lfgnode update` that recreates the Deployment. The manager must
reach `<ip>.sslip.io` from inside the cluster, so use an address the Docker network can route to (not
`127.0.0.1`).

## Release endpoint for local tests

Simplest: leave `release-endpoint` empty and give `workload-manager-image` an explicit tag (as above). The CLI
then deploys that image, passes no `RELEASE_URL` and disables self-update; `lfgnode start` no longer exits with
`release endpoint is not configured`.

If you do want a release endpoint over `http://`: a development CLI build (`version: development`) reads
`<release-endpoint>/workload-manager/unstable.txt` (and `releases/unstable.txt`), not `stable.txt`, so serve
that file (its whole content is the image tag). Even then the manager cannot self-update over http, so
remove `RELEASE_URL` as in the recipe above.

## Uptime and the hourly / daily roots

1. After 2 to 3 minutes of running, `lfgnode status` shows `OnlineToday` growing.
2. Trigger jobs by hand with the cron secret
   (`GET /api/v1/cron/trigger/summarize-uptime?type=hourly|daily&date=<ISO>`, header `x-cron-secret`).
3. The hourly cron (`0 0 * * * *`, top of every hour) submits even with no data (placeholder root `0x00...01`) and consumes the
   slot; a manual trigger in the same hour reverts with `IdAlreadyExists` (roots are write-once).
4. A daily root needs all 24 hourly roots of that day. On a fresh registry the admin fills gaps with
   `backfillRoot`. For a **same-day** daily root you need backfill plus uptime data of the **prior** day
   (see below).
5. Day boundary is 02:00 UTC (`TZ=Etc/GMT+2`, daily cron 02:02 UTC). In verification SQL quote the camelCase
   columns (`"projectCode"`, `"startTime"`).
6. The dashboard origin must be in `server.corsWhitelist`; reload the dashboard after switching wallet. Set
   `NEXT_PUBLIC_LFG_UPTIME_CONTRACT_ADDRESS` at build time for `/verify-uptime`.

### Same-day daily root: prior-day data and ids

1. **Insert prior-day uptime sessions** for the node's workload (timestamps are UTC; the contract day is
   02:00 UTC to 02:00 UTC, so cover `[yesterday 02:00, today 02:00)`; adjust the dates if it is before
   02:00 UTC now). Quote the camelCase columns:

   ```sql
   INSERT INTO node_workload_uptimes ("nodeWorkloadId", "startTime", "endTime")
   SELECT id, TIMESTAMP '2026-11-15 02:00:00', TIMESTAMP '2026-11-16 02:00:00'
   FROM node_workloads
   WHERE node_address = '<node address, lower-case>' AND "projectCode" = '<code>' AND deleted_at IS NULL;
   ```

2. **Backfill the registry ids** `D*100 + 1` to `D*100 + 24` for that contract day `D` as registry admin, where
   `D` is the day number of the **previous** day, `floor((now - 7200) / 86400) - 1` (the daily id is
   `D*100`). Commands: [Contracts](contracts.md#leaf-ids-and-time-rules).
3. Trigger the daily job (`type=daily`) with a `date` inside the day after `D`.

`scripts/compute-and-upload.js` accepts both `uptime` (backend JSON) and `uptimeAmount`
([Contracts](contracts.md#manual-submit-proofs-and-troubleshooting)).

## Points-only path (master sample)

There is no manual trigger. Set `daily_submit_at` a minute ahead (restart), and make sure yesterday's `DAILY`
rows exist. In production use a time such as `02:30:00`, after the backend's 02:02 UTC job
([timing note](master-and-rewards.md)).

## Token-reward flow on an anvil fork

Emission accrues in whole UTC days from 00:00 UTC of the day `startGoingPhase` is called, so a same-day test
reads 0 and waiting a day is slow. Fork and warp instead: the whole flow then runs in minutes.

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
6. **Reward job script: use the current wall-clock `Date`.** The job asks the backend for
   `date=<yesterday at the current time of day>` and the backend's day boundary is 02:00 UTC, so a script that
   passes midnight (or any fixed time before 02:00 UTC) queries the wrong day.
7. The voucher deadline is the server's wall clock plus 600 s, so on a time-warped fork the voucher can look
   expired. **Rewind the fork** after warping and before `claim`: `cast rpc anvil_setTime $(date +%s)`, then
   mine a block. After the operator's `claim` on NOR, the reward subgraph's `Claim` event lowers `unclaim` and
   moves the nonce ([flow diagram](master-and-rewards.md#reward-flow-and-emission-setup)).

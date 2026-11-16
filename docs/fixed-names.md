# Fixed names

Many names look like leftovers of the old company (`lfg-...`) but are **shared contracts** between the CLI,
the workload manager, the backend, the contracts and every published workload manifest. The decision for
the open-source release was to rename nothing, so everything keeps working exactly as before. Changing any
of them breaks running nodes and published workloads unless you change the CLI, manager, backend and all
manifests together and migrate existing nodes.

## On the operator machine and in the cluster

| Name | Where it is used and why it must stay |
|---|---|
| `lfgnode` (binary, config directory, archive prefix `lfgnode_<Os>_<arch>`) | `lfgnode update` downloads archives by this name; `install.sh`; config path `~/.config/lfgnode/` |
| k3d cluster `lfg-node`, kube context `k3d-lfg-node` | The CLI creates it; the workload manager selects the context through `KUBECTL_CONTEXT` |
| Namespace `lfg-user` | All workloads; the manager's `NAMESPACE`; published manifests set `namespace: lfg-user`; the deployer forces it |
| Deployment and Service `lfg-workload-manager`, port **`6161`** | Workloads push health to `lfg-workload-manager.lfg-user.svc.cluster.local:6161`; the ConfigMap `healthBaseUrl` assumes it; the self-update patches the Deployment by this name |
| Deployment and Service `ipfs`, port `5001` | ConfigMap `ipfsBaseUrl`; excluded from workload management |
| ConfigMap `lfg-node-config` with keys `nodeServerBaseUrl`, `nodeAddress`, `healthBaseUrl`, `ipfsBaseUrl` | Read by workload containers (manifests use `configMapKeyRef`) |
| Secret `lfg-workload-<project-code>` (key `REWARD_ADDRESS`) | Injected into the workload container with `envFrom`; also referenced by the datagram sidecar definition |
| Kubernetes API host port `6445` | Default cluster definition |
| `NODE_PRIVATE_KEY`, `NODE_SYSTEM_ENDPOINT`, `IPFS_GATEWAY`, `RELEASE_URL`, `SENTRY_DSN`, `KUBE_CONFIG`, `NODE_ENV` | Environment contract between the CLI and the workload manager |

Rule enforced by the workload manager: Deployments named `ipfs` and `lfg-workload-manager` are excluded
from workload management, and **every other Deployment in `lfg-user` is treated as a workload** (and
deleted when the backend does not list it).

## Project and workload identifiers

| Name | Why it must stay |
|---|---|
| Project code (`projects.code`, deployer `code`) | Kubernetes Deployment name and `app` label, secret name `lfg-workload-<code>`, workload heartbeat `projectCode`, **part of every Merkle leaf**, and the function name `claim<Code>Token()` in reward contracts. Changing it orphans history |
| First container name `main-container` and labels `app`, `version` | Set by the deployer on every manifest |
| Signed publish payload `{projectCode, version, workloadIpfsCid, legacy}` | The deployer signs exactly this string with this key order and the backend verifies the raw body. Keep `legacy` (the backend ignores it) |
| `NODE_ADDRESS`, `HEALTH_BASE_URL`, `IPFS_URL` env vars in manifests | Names the template reads |

## Backend, chain and data formats

| Name | Why it must stay |
|---|---|
| Timezone `Etc/GMT+2` (a "day" starts at 02:00 UTC) and the registry's fixed `block.timestamp - 2 hours` | On-chain day and hour ids are derived from them; both sides must agree. Never change on a running deployment |
| Leaf `keccak256(abi.encode(string slug, address nodeId, uint256 uptimeMs))` and registry ids `dayNumber * 100 + hourType` | Backend, registry, dashboard `/verify-uptime` and reward code use the same format |
| Subgraph entities `NodeTotalBalance`, `ERC721NodeTotalBalance` (and fields `nft`, `user`, `totalBalance`, `updatedAt`) | The backend queries these names |
| Reward subgraph entities `NodeOperatorRewardClaim`, `NodeOperatorTotalRewardClaim` | nodesale-reward queries `nodeOperatorRewardClaims` |
| `claimHelixToken()`, `helixToken()`, `IHelixToken`, typo functions `...Permanntly...` | Part of the deployed ABI; rename only for new deployments and update the reward service |
| `ClaimReward(uint256 chainId,address contract,address receiver,uint256 amount,uint256 deadline,uint256 nonce)` type hash, raw-ECDSA digest | Voucher format shared by the contract and the signer |
| Signature headers `X-User-Address`, `X-Signature`; cron header `x-cron-secret`; reward admin headers `x-offchain-address`, `x-offchain-issued-at`, `x-offchain-signature` | Wire formats used by every client |
| URL paths `/api/v1/...` and the one `v1.1` route (`node-uptimes`) | CLI, manager, deployer, master, reward service and dashboard are written against them |
| Database table and column names (including camelCase `"projectCode"` in `node_workloads`) | Migrations, typed SQL and reward code depend on them |
| Redis database `0` | The expired-key listener only handles `__keyevent@0__:expired` |
| `NEXT_PUBLIC_LFG_UPTIME_CONTRACT_ADDRESS` (dashboard), `lfg_node_system.*` (master config) | Historical names of configuration keys; they now point at *your* contract and backend |

## If you really want to rebrand

Change the names together in the CLI, the workload manager, the backend, the template, the deployer and
**every published manifest**, ship a migration for existing nodes (they have the old ConfigMap, secrets and
cluster), and deploy only to new deployments for contract ABI names. Otherwise leave them alone.

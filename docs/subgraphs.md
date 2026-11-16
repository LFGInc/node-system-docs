# Subgraphs

Two small [The Graph](https://thegraph.com) subgraphs. Neither holds keys; both only read public chain
data. Each deployment indexes one chain. Entity and field names are queried by other services and **must
not change**.

| Subgraph | Repository | Consumer |
|---|---|---|
| License | [`opensource-license-subgraph`](https://github.com/LFG-Incorporated/opensource-license-subgraph) | the [backend](backend.md) (license counts) |
| Reward | [`opensource-node-reward-subgraph`](https://github.com/LFG-Incorporated/opensource-node-reward-subgraph) | [`nodesale-reward`](master-and-rewards.md#nodesale-reward) (claim events) |

Both need Node.js 22 and pnpm 9, and can be deployed to Goldsky, The Graph Studio, or a self-hosted
graph-node. Both repositories have a CI workflow that builds on every push and deploys on `v*` tags (or
manually) when you set `secrets.GOLDSKY_API_KEY`, `vars.SUBGRAPH_NAME`, `vars.SUBGRAPH_CONFIG`.

## License subgraph

Counts, per wallet, how many license NFTs (ERC1155 or ERC721) it holds. It was extracted from the nodesale
subgraph and works for **any** ERC1155 / ERC721 contract: you list the NFT contracts in a config file, no
sale contract needed. The backend polls it every few seconds and copies balances into
`user_wallet_balances`.

| Entity (query name) | Fields | Meaning |
|---|---|---|
| `NodeTotalBalance` (`nodeTotalBalances`), ERC1155 | `id` (nft ++ user), `nft`, `user`, `totalBalance`, `updatedAt` | sum of balances over all token ids of that contract for the user |
| `ERC721NodeTotalBalance` (`erc721NodeTotalBalances`), ERC721 | same | number of tokens of that contract held by the user |

`updatedAt` is the block timestamp (seconds) of the last change. The backend's query:

```graphql
nodeTotalBalances(first: 100, orderBy: updatedAt, orderDirection: asc, where: { updatedAt_gte: <unix seconds> }) {
  id nft totalBalance user updatedAt
}
# ERC721: erc721NodeTotalBalances(...) with the same fields and filter
```

### Configure and deploy

```bash
pnpm install --frozen-lockfile
cp configs/example.json configs/config.json     # edit
```

```json
{
  "network": "bsc",
  "erc1155": [ { "name": "MyLicense",    "address": "0x...", "startBlock": 12345678 } ],
  "erc721":  [ { "name": "MyLicense721", "address": "0x...", "startBlock": 12345678 } ]
}
```

- `network`: graph-node / Goldsky network id (`mainnet`, `bsc`, `chapel`, `base`, `arbitrum-one`, ...). One
  subgraph is one network; deploy one per chain.
- `name`: unique alphanumeric label per entry (becomes the data source name).
- `startBlock`: deployment block of the NFT contract; the first mint must be at or after it.
- Either list may be empty (`[]`), but not both.

```bash
pnpm prepare:subgraph       # renders subgraph.yaml from $CONFIG (default configs/config.json)
pnpm codegen && pnpm build
# Goldsky
export GOLDSKY_API_KEY=...        # never commit
pnpm goldsky:login --token "$GOLDSKY_API_KEY" && pnpm goldsky:deploy myproject-license/v0.0.1
# local / own graph-node
pnpm create-local && pnpm deploy-local      # optional env: GRAPH_NODE_URL, IPFS_URL, SUBGRAPH_NAME, VERSION_LABEL
# The Graph Studio
pnpm exec graph deploy <slug> subgraph.yaml     # after `graph auth` with your own key
```

### Verify and connect

1. Sync reaches the chain head and `_meta { hasIndexingErrors }` is `false`.
2. For a known wallet, compare `nodeTotalBalances(where:{user:"0x..."})` with `balanceOf` summed over your
   token ids (ERC1155), or `balanceOf(address)` (ERC721).
3. Insert the GraphQL URL into the backend table `subgraph_endpoints` (`name`, `url`, `token_types`
   `ERC1155` and/or `ERC721`) and **restart the backend**: the list is loaded once at startup. The
   endpoint's `nft` values must equal `projects.nft` (lower-case). Each endpoint is tracked independently
   (`source_id`), so you can run one subgraph per chain.

### Known issues

- Transfers to the zero address (burns) create or increase a balance row for `0x000...000`; harmless.
- The backend paginates with `updatedAt_gte` and `first: 100`. If more than 100 balances change in the same
  block timestamp the poller can stall. Avoid mass airdrops or batch mints to over 100 wallets in one block,
  or raise the limit in the backend.
- Only balances are tracked, not ownership of individual token ids.
- Non-standard NFTs that move tokens without the standard `Transfer*` events are not supported.
- A wrong `startBlock` or a missing contract gives wrong license counts, which limit how many workloads a
  wallet may run.

## Reward subgraph

Indexes the `Claim(address indexed receiver, uint256 amount, uint256 nonce)` events of a
[`NodeOperatorsReward`](contracts.md#node-operators-reward) contract. One deployment indexes one contract on
one chain for one project (several data sources are possible but separate subgraphs are recommended).

| Entity | Fields | Notes |
|---|---|---|
| `NodeOperatorRewardClaim` (immutable) | `id` (tx hash + log index), `user`, `projectCode`, `amount`, `nonce`, `blockNumber`, `blockTimestamp`, `transactionHash` | one row per `Claim` event; `nonce` is the event's nonce |
| `NodeOperatorTotalRewardClaim` | `id` (user + utf8(projectCode)), `user`, `projectCode`, `nonce`, `amount`, `updatedAt` | running totals per receiver: `nonce` = number of claims, `amount` = total claimed |

`projectCode` is a free label from your config, stored on every entity.
`nodesale-reward` queries `nodeOperatorRewardClaims(where: { projectCode, blockTimestamp_gt })` to decrease
operators' claimable balance after on-chain claims.

### Configure and deploy

```bash
pnpm install --frozen-lockfile
cp configs/example.json configs/config.json     # edit
```

```json
{ "network": "bsc", "projectCode": "myproject", "address": "0x...your NodeOperatorsReward...", "startBlock": 12345678 }
```

- `network`: graph-node / Goldsky network id (`bsc`, `chapel` for BSC testnet, `mainnet`, `base`,
  `arbitrum-one`, ...).
- `projectCode`: your project slug (lower-case letters, digits, dashes; substituted into YAML). It must
  equal `projects.code` in nodesale-reward.
- `startBlock`: block of the NOR deployment transaction. **Never 0**, and never later than the deploy: a
  late start block misses claims and yields wrong nonces.

```bash
pnpm prepare:subgraph && pnpm codegen && pnpm build     # CONFIG=configs/example.json for a smoke test
pnpm goldsky:login --token "$GOLDSKY_API_KEY" && pnpm goldsky:deploy myproject-node-reward/v0.0.1
```

Local graph-node alternative (the graph-node image is `linux/amd64` only: on Apple Silicon it runs under
emulation, slow but working; no Goldsky needed): `cp .env.example .env` (set `POSTGRES_PASSWORD`, `RPC_URL`,
`GRAPH_NODE_NETWORK` equal to your config's `network`), `docker compose up -d`, then
`pnpm prepare:subgraph && pnpm codegen && pnpm build && pnpm create-local && pnpm deploy-local`; GraphQL is at
`http://localhost:8000/subgraphs/name/node-reward`. The Graph Studio works with `graph auth` and
`pnpm exec graph deploy <studio-slug> subgraph.yaml`.

#### Local graph-node: ports, image tag, networks

- The compose file publishes Postgres on host port **5434** (not 5432, which clashes with a local Postgres and
  with the backend's compose file) and pins the graph-node image to `graphprotocol/graph-node:v0.45.0`
  instead of an untagged `latest`.
- **One graph-node indexes one network** in this setup (`GRAPH_NODE_NETWORK` + `RPC_URL`). It hosts both
  subgraphs only when both contracts are on that chain. To test a testnet and an anvil fork at once, run **two
  graph-nodes** with different ports and data directories. The compose file reads the overrides
  `COMPOSE_PROJECT_NAME`, `DATA_DIR`, `POSTGRES_HOST_PORT`, `IPFS_HOST_PORT` and `GRAPH_NODE_{GQL,WS,ADMIN,INDEX,METRICS}_PORT`:

  ```bash
  COMPOSE_PROJECT_NAME=anvil DATA_DIR=data-anvil POSTGRES_HOST_PORT=5435 IPFS_HOST_PORT=5002 \
  GRAPH_NODE_GQL_PORT=9000 GRAPH_NODE_WS_PORT=9001 GRAPH_NODE_ADMIN_PORT=9020 \
  GRAPH_NODE_INDEX_PORT=9030 GRAPH_NODE_METRICS_PORT=9040 \
  GRAPH_NODE_NETWORK=<network> RPC_URL=http://host.docker.internal:8545 docker compose up -d
  GRAPH_NODE_URL=http://localhost:9020 IPFS_URL=http://localhost:5002 pnpm create-local && pnpm deploy-local
  ```

  The second instance serves GraphQL at `http://localhost:9000/subgraphs/name/<SUBGRAPH_NAME>`.
- The **license subgraph has no compose file**. Use the reward subgraph's compose file and point the license
  subgraph's `GRAPH_NODE_URL` and `IPFS_URL` at it (defaults `http://localhost:8020` and `http://localhost:5001`).

### Verify and connect

1. Synced to the head, `_meta { hasIndexingErrors }` is `false`.
2. Make a test claim, then `nodeOperatorRewardClaims(orderBy: blockNumber, orderDirection: desc, first: 1)`
   must match the `Claim` event.
3. `nodeOperatorTotalRewardClaims(where:{user:...}).nonce` must equal
   `cast call $NOR "userNonce(address)(uint256)" $USER`.
4. Put the GraphQL URL into `projects.subgraph_url` of nodesale-reward.

The contract's `userNonce` is the source of truth; this subgraph is a lagging mirror. A stale or syncing
subgraph can lead a signer to use an already-used nonce, and the claim then reverts `InvalidNonce`. If more
than one project shares one subgraph, copy the `dataSources` block in `subgraph.template.yaml` (entity ids
include `projectCode`, so projects never collide).

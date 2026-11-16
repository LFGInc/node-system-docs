# Contracts

Two Foundry repositories, both MIT, both **not formally audited**. They work on any EVM chain (the original
deployments used BSC). Read the repository `SECURITY.md` files, summarized in [Security](security.md#contracts).

| Contract | Repository | Needed by |
|---|---|---|
| `GlobalNodeUptimeMerkleRegistry` | [`opensource-merkle_tree_uptime`](https://github.com/LFG-Incorporated/opensource-merkle_tree_uptime) | every project (the backend commits hourly and daily roots to it) |
| `NodeOperatorsReward` (and reference `HelixToken`) | [`opensource-node-operators-reward`](https://github.com/LFG-Incorporated/opensource-node-operators-reward) | projects that pay token rewards |

Both need `forge`/`cast` ([Foundry](https://book.getfoundry.sh/)) and git submodules
(`git clone --recurse-submodules`). Both repositories document every environment variable in `.env.example`.

!!! warning "Downloaded without submodules? Pin the dependencies"
    A ZIP or tarball has an empty `lib/`, and an unpinned `forge install` fetches the **latest** OpenZeppelin
    (v5.7.0 at the time of writing). That breaks the reward repository: 22 of 23 `HelixToken` tests revert with
    `ERC1967ProxyUninitialized`. Always give the tag (inside a git repository, for example after `git init`):

    ```bash
    # node operators reward / HelixToken: 60 tests pass with these
    forge install foundry-rs/forge-std@v1.9.7 OpenZeppelin/openzeppelin-contracts@v5.1.0 \
      OpenZeppelin/openzeppelin-contracts-upgradeable@v5.1.0 paulrberg/prb-math@v4.1.0
    # uptime registry: the versions recorded in its foundry.lock (25 tests pass)
    forge install foundry-rs/forge-std@v1.14.0 OpenZeppelin/openzeppelin-contracts@v5.5.0
    ```

    A git clone with `--recurse-submodules` already has the pinned versions, and both repositories carry a
    `foundry.lock` with them. `forge install` does **not** read the lock: it overwrites it with whatever it
    fetched, so pass the tags explicitly.

## Uptime registry

`GlobalNodeUptimeMerkleRegistry` stores one Merkle root per hour (24 per day) and one per day. It uses
OpenZeppelin `AccessControl`, `Pausable` and `MerkleProof`. It holds no funds; it stores roots and verifies
proofs.

!!! warning "One registry per backend"
    There is a single root per hour and per day for the whole contract, so two **backends** submitting to
    one registry would collide on the same slot. One backend can serve several projects: each leaf carries
    the project slug, so one hourly or daily root covers all of them and `verify` works for every project's
    leaves. Give each separate backend deployment its own registry. The contract is used unchanged.

### Roles

| Role | Granted to | Can do |
|---|---|---|
| `DEFAULT_ADMIN_ROLE` | `ADMIN` (constructor argument; multisig on mainnet) | `pause` / `unpause`, `backfillRoot` (overwrite any non-future id), grant and revoke roles |
| `SUBMITTER_ROLE` | the backend's `ether.privateKey` wallet (`SUBMITTER`) | `submitHourlyRoot`, `submitDailyRoot` |

Only the admin can grant roles; the deployer has no special status unless it is also `ADMIN`.

### Deploy

```bash
cp .env.example .env     # RPC_URL, CHAIN_ID, PRIVATE_KEY, ADMIN, SUBMITTER, ETHERSCAN_API_KEY
set -a && source .env && set +a
forge build && forge test
forge script script/Deploy.s.sol:Deploy --rpc-url $RPC_URL --chain-id $CHAIN_ID \
  --private-key $PRIVATE_KEY --broadcast --legacy \
  --verify --etherscan-api-key $ETHERSCAN_API_KEY -vvvv
```

`--legacy` is for chains without EIP-1559 such as BSC; drop it elsewhere. For a hardware wallet use
`--ledger --sender $ADMIN` instead of `--private-key`. When the broadcaster **is** `ADMIN` the script also
grants `SUBMITTER_ROLE` to `SUBMITTER`; otherwise the admin must do it:

```bash
cast send $REGISTRY_ADDRESS "grantRole(bytes32,address)" $(cast keccak "SUBMITTER_ROLE") $SUBMITTER \
  --rpc-url $RPC_URL --private-key $ADMIN_KEY      # or through the multisig UI
```

Verify:

```bash
cast call $REGISTRY_ADDRESS "hasRole(bytes32,address)(bool)" $(cast keccak "SUBMITTER_ROLE") $SUBMITTER --rpc-url $RPC_URL   # true
cast call $REGISTRY_ADDRESS "hasRole(bytes32,address)(bool)" 0x0000000000000000000000000000000000000000000000000000000000000000 $ADMIN --rpc-url $RPC_URL   # true
```

Then put the RPC URL, the registry address and the submitter key into the [backend](backend.md#configuration)
(`ether.*`) and fund the submitter with gas.

### Leaf, ids and time rules

Off-chain code must match these exactly (the backend already does).

```
leaf = keccak256(abi.encode(string slug, address nodeId, uint256 uptime))        # uptime in ms
id   = dayNumber * 100 + hourType          # hourType: 0 = daily, 1..24 = hourly
t    = block.timestamp - 7200              # OFFSET = 2 hours, MINUS
dayNumber = t / 86400,  hourType = (t / 3600) % 24
```

The tree is an OpenZeppelin Merkle tree with sorted-pair hashing (`@openzeppelin/merkle-tree`
`SimpleMerkleTree`).

- **Hourly** id written by `submitHourlyRoot` now: if `hourType == 0` then `(dayNumber - 1) * 100 + 24`,
  else `dayNumber * 100 + hourType`. Submit once per hour.
- **Daily** id written by `submitDailyRoot` now: `(dayNumber - 1) * 100`: the daily root is for the
  **previous** day and is submitted during the following day.
- `submitDailyRoot` requires **all 24 hourly roots** (`dayNumber*100 + 1 .. 24`) of that day, else
  `MissingHourlyRoots`. Hour 24 of day D is written during the first hour of D+1 (02:00 to 03:00 UTC), so
  the earliest daily submission is shortly after; the backend runs it at 02:02 UTC.
- Roots are write-once (`IdAlreadyExists`); only the admin can overwrite with `backfillRoot`. A `cid` must
  be 46 to 128 characters.

**Ids to backfill.** Let `D` be the contract day number, `floor((timestamp - 7200) / 86400)`, which is the
number of days since 1970-01-01 of the UTC calendar date whose slots begin at 02:00 UTC. The daily root that
`submitDailyRoot` writes now is for `D = currentDayNumber - 1`. For a fresh registry, fill every empty hourly
id `D*100 + 1` to `D*100 + 24` with `backfillRoot` (ids must not be in the future); the daily id is `D*100`:

```bash
D=$(( $(date -u -j -f %Y-%m-%d 2026-01-01 +%s) / 86400 ))        # macOS; GNU: $(date -u -d 2026-01-01 +%s)
for h in $(seq 1 24); do
  cast send $REGISTRY_ADDRESS "backfillRoot(uint256,bytes32,string)" $((D*100+h)) \
    0x0000000000000000000000000000000000000000000000000000000000000001 "$CID" \
    --rpc-url $RPC_URL --private-key $ADMIN_KEY --legacy
done
```

`$CID` is any 46 to 128 character string; the placeholder root `0x00...01` is what the backend writes for
empty hours. `backfillRoot` overwrites, so skip ids that already hold real roots.

### Contract API

| Function | Access | Notes |
|---|---|---|
| `submitHourlyRoot(bytes32 root, string cid)` | `SUBMITTER_ROLE`, not paused | id derived from block time |
| `submitDailyRoot(bytes32 root, string cid)` | `SUBMITTER_ROLE`, not paused | needs 24 hourly roots |
| `backfillRoot(uint256 id, bytes32 root, string cid)` | `DEFAULT_ADMIN_ROLE`, not paused | overwrites, emits `RootBackfilled`; future ids rejected |
| `pause()` / `unpause()` | `DEFAULT_ADMIN_ROLE` | |
| `verify(id, slug, nodeId, uptimeAmount, proof)` | view | |
| `getRoot(id)`, `getCid(id)`, `rootExists(id)` | view | |

Events: `HourlyRootSubmitted`, `DailyRootSubmitted`, `RootBackfilled`. Reward services read roots through
`getRoot` / `getCid` or the events; the CID points to the full tree JSON on IPFS.

### Manual submit, proofs and troubleshooting

`scripts/compute-and-upload.js` (Node 18+, `cd scripts && npm install`) builds a tree from a JSON array of
`{slug,nodeId,uptimeAmount}` and prepares proofs; it does not touch the chain. The backend's IPFS JSON uses
the field name `uptime`; the script accepts both (`uptime ?? uptimeAmount`), so you can feed it a backend
file directly.

```bash
node compute-and-upload.js compute sample-data.json            # prints "Root hash: 0x..."
forge script script/SubmitHourly.s.sol:SubmitHourly --rpc-url $RPC_URL --chain-id $CHAIN_ID --private-key $PRIVATE_KEY --broadcast --legacy -vvvv   # needs ROOT_HASH, CID
forge script script/SubmitDaily.s.sol:SubmitDaily   --rpc-url $RPC_URL --chain-id $CHAIN_ID --private-key $PRIVATE_KEY --broadcast --legacy -vvvv
node compute-and-upload.js prepare <leaf-index> <id> [data-file]      # id, slug, nodeId, uptimeAmount, proofComma
forge script script/Validate.s.sol:Validate --rpc-url $RPC_URL -vvvv  # needs ID, SLUG, NODE_ID, UPTIME_AMOUNT, PROOF
```

`verify` reverts `RootNotFound` when no root exists for the id and returns `false` for a bad proof. The
dashboard's `/verify-uptime` page does the same proof building in the browser ([Dashboard](dashboard.md)).

| Symptom | Cause |
|---|---|
| `already known` | use `--legacy` on BSC; the transaction may already have succeeded, check the explorer |
| rate limiting | add `--slow` |
| `MissingHourlyRoots` | an hourly slot of that day is empty; backfill it as admin |
| `AccessControlUnauthorizedAccount` | the sender lacks the role |

## Node operators reward

`NodeOperatorsReward` (NOR) holds reward tokens and pays them out to operators against **signatures from a
reward service**. `HelixToken` is the reference UUPS ERC20 (`HELIX` / `LIX`) with a decaying daily
emission split among participants; NOR is registered as one participant so emission flows to operators.
NOR works unchanged with **any ERC20** (payouts use `SafeERC20.safeTransfer`).

```
reward service (signer key) --signs claim(receiver, amount, deadline, nonce)--> operator
operator --claim(receiver, amount, deadline, nonce, v, r, s)--> NodeOperatorsReward --transfer--> operator
claimer key --claimHelixToken()--> NOR pulls emission from the token (HelixToken only)
reward subgraph indexes Claim events so the service knows each receiver's nonce
```

### Roles

| Where | Role | Hold with | Powers |
|---|---|---|---|
| NOR | `DEFAULT_ADMIN_ROLE` (`REWARD_ADMIN`) | multisig | grant / revoke roles; `emergencyWithdraw` (full balance to caller) |
| NOR | `SIGNER_ROLE` (`REWARD_SIGNER`) | reward service signing key (KMS) | authorizes claims, i.e. spending the contract's balance |
| NOR | `CLAIMER_ROLE` (`REWARD_CLAIMER`) | ops wallet | `claimHelixToken()`: pull emissions from the token |
| `HelixToken` | `owner` (`TOKEN_OWNER`) | multisig (+ timelock) | phases, emission rate, participants, whitelists, **upgrades** |

### Deploy

```bash
cp .env.example .env     # RPC_URL, PRIVATE_KEY (or DEPLOYER_ADDRESS for Ledger), TOKEN_OWNER, REWARD_ADMIN, REWARD_SIGNER, REWARD_CLAIMER
./deploy_helix_token.sh                # step 1; skip if you pay in an existing ERC20. Put the PROXY address in .env as TOKEN_ADDRESS
./deploy_node_operators_reward.sh      # step 2; constructor (admin, signer, claimer, token). Note NOR_ADDRESS and the deployment block
```

Start emission (HelixToken only, owner transactions). **Order matters**: deploy the token (proxy) ->
deploy NOR -> `startInitialPhase` -> `startGoingPhase` with NOR as a participant (plus percentage and daily
rate). `startGoingPhase` runs **once**; `updateParticipants` is not needed initially. On BSC pass `--legacy`
to the deploy scripts (they forward extra arguments: `./deploy_helix_token.sh --legacy`).

```bash
# initial phase: mint the initial supply and set max supply (percents are 1e18 = 100%, must sum to 1e18)
cast send $TOKEN_ADDRESS "startInitialPhase(uint256,uint256,address[],uint256[])" <maxSupply> <initialSupply> "[<recipient>]" "[1000000000000000000]" --rpc-url $RPC_URL ...
# going phase: register NOR (and others) as emission participants; percents sum to 1e18, max 50 participants
cast send $TOKEN_ADDRESS "startGoingPhase(address[],uint256[],uint256)" "[$NOR_ADDRESS,<other>]" "[<p1>,<p2>]" <dailyEmissionRate_1e18> --rpc-url $RPC_URL ...
```

**UTC-day rule.** Emission accrues in whole UTC days counted from 00:00 UTC of the day `startGoingPhase` is
called; the first claimable amount appears at the **next** 00:00 UTC (not 24 h later). A same-day test always
reads 0: use an anvil fork with a time warp ([Local testing](local-testing.md#token-reward-flow-on-an-anvil-fork)).

`dailyEmissionRate` is a fraction of the remaining emission supply per day (1e18 = 100%). Anyone with
`CLAIMER_ROLE` then calls `claimHelixToken()` periodically. With a plain ERC20, transfer the budget to
`NOR_ADDRESS` directly and keep it funded: claims revert when the balance is too low.

Verify:

```bash
cast call $NOR_ADDRESS "hasRole(bytes32,address)(bool)" $(cast keccak "SIGNER_ROLE")  $REWARD_SIGNER  --rpc-url $RPC_URL   # true
cast call $NOR_ADDRESS "hasRole(bytes32,address)(bool)" $(cast keccak "CLAIMER_ROLE") $REWARD_CLAIMER --rpc-url $RPC_URL   # true
cast call $NOR_ADDRESS "helixToken()(address)" --rpc-url $RPC_URL                                                           # == TOKEN_ADDRESS
```

### Claim signature format

```
CLAIM_REWARD_TYPE_HASH = keccak256("ClaimReward(uint256 chainId,address contract,address receiver,uint256 amount,uint256 deadline,uint256 nonce)")
digest    = keccak256(abi.encode(CLAIM_REWARD_TYPE_HASH, block.chainid, address(NOR), receiver, amount, deadline, nonce))
signature = ECDSA over `digest` directly      (NO "\x19Ethereum Signed Message" prefix, NO EIP-712 domain)
```

`nonce` must equal `userNonce(receiver)` on-chain and increments on every successful claim for that
receiver. `amount` is the amount for this claim (not a running total); `deadline` is a unix time after
which the claim reverts `Expired`. Anyone can submit `claim(receiver, amount, deadline, nonce, v, r, s)`;
tokens always go to `receiver`. Sign the raw digest (ethers v6: `new ethers.SigningKey(pk).sign(digest)`),
not `signMessage`. Events: `Claim(address indexed receiver, uint256 amount, uint256 nonce)`,
`EmergencyWithdraw(address indexed token, uint256 amount)`. [`nodesale-reward`](master-and-rewards.md#nodesale-reward)
implements this signer and returns `{ r, s, v }`.

### Naming constraint

!!! warning "The project name is in the ABI"
    `nodesale-reward` calls `claim<Code>Token()` on NOR, where `<Code>` is `projects.code` with its first
    letter upper-cased (`helix` gives `claimHelixToken()`, `flipit` gives `claimFlipitToken()`). The
    reference contract only has `claimHelixToken()`. For any other project code you must rename
    `claimHelixToken` (and optionally `HelixToken`, `IHelixToken`, `helixToken`) in your fork **before
    deploying**, find-and-replace across `src/`, `script/`, `test/` (a `sed` rename across those
    directories keeps all 60 tests passing).

Also hard-coded in `HelixToken.initialize`: name `HELIX` and symbol `LIX`. To adapt, change the strings or
make them `initialize` arguments (update `script/DeployHelixToken.s.sol` and tests). Do this only for new
deployments, never for an already deployed proxy. Function and event names containing the typo
`Permanntly` are part of the deployed ABI and are kept. The `Claim` event ABI does not change, so the
[reward subgraph](subgraphs.md#reward-subgraph) is unaffected by renames.

`EPOCH_BASED_PARTICIPANTS.md`, `PRB_MATH_MIGRATION.md` (Vietnamese), `PROOF_OF_CORRECTNESS.md` and
`SIMPLE_PARTICIPANT_UPDATE.md` in that repository are historical design notes;
`EPOCH_BASED_PARTICIPANTS.md` describes a design the code does **not** implement.

# Security: known issues and recommendations

The open-source release is intentionally **"runs exactly as it did before"**: only secrets were removed and
hosted values turned into configuration. Known vulnerabilities and weaknesses are
**documented, not fixed**. None of the code has had an external audit, and the lists below are not
exhaustive. Each repository has a `SECURITY.md`; this page consolidates them. Review it, decide which items
matter for your threat model, and fix or mitigate before exposing anything to the internet or to real
funds.

To report a new vulnerability use GitHub's private "Report a vulnerability" on the repository you forked
from or on your own fork. There is no SLA and the code is maintained by whoever operates it.

## The one-paragraph threat model

Whoever controls your **backend, release host (S3 bucket or equivalent), IPFS gateway or container
registry** can run code as root on every operator's machine, because the CLI and the workload manager
download and install from them without verification and the manager holds the node key and a
cluster-admin kubeconfig. Whoever holds the **registry submitter** key can publish wrong uptime roots, the
**NOR signer** key can authorize payouts of the contract's balance, and whoever can write the **reward
database** or impersonate the backend or subgraph can over-distribute tokens. Treat those as your crown
jewels.

## Node CLI

Repository `opensource-node-cli`:

1. **Self-update is unsigned and unverified.** `lfgnode update` downloads
   `<release-endpoint>/releases/<version>/lfgnode_<OS>_<arch>.tar.gz` and installs it with `sudo mv`. No
   checksum or signature check, HTTP status not checked, archive entry names used as-is (path traversal
   possible), the `stable.txt` body is used unvalidated.
2. **The manager image is pulled by mutable tag.** The tag comes from
   `<release-endpoint>/workload-manager/stable.txt` (status not checked: an S3 XML error body would become
   the tag), `imagePullPolicy: Always`, no digest pinning or signature verification.
3. **Backend-controlled remote constants.** `GET /api/v1/public/cli-constants` can supply the image,
   release host and gateway for anything not set locally. A compromised backend can point nodes at an
   arbitrary image.
4. **Workload manifests are trusted blindly.** Not verified against their CID, HTTP status not checked, only
   CPU and memory are checked; `privileged`, `hostPath`, `hostPID`, `hostNetwork` are not blocked and `lfg-user`
   has no Pod Security admission. The k3d node itself is a privileged container.
5. **The node private key and the operator's entire `~/.kube/config` (every context) are passed to the
   manager as plain environment variables**, readable by anyone who can read the Deployment.
6. **The Kubernetes API is bound to `0.0.0.0:6445`** (`kubeAPI.hostIP: "0.0.0.0"`); the kubeconfig embeds
   admin credentials and the manager skips TLS verification of the API server.
7. **`config.yaml` (node private key) and `~/.config/lfgnode/workloads/*.yaml` are world-readable by
   default** (the workload files are mode 0644).
8. **Replayable request signatures.** EIP-191 over the raw body only, no nonce or timestamp. `DELETE`
   requests now carry a signature over `<nodeAddress>|<projectCode>` (earlier CLI builds sent them unsigned,
   the backend answered `401`, and the license slot was never freed).
9. Sentry (only if configured) receives error logs and the node address. go-ethereum is LGPL-3.0 (see
   `NOTICE`).

## Workload manager

Repository `opensource-node-system-workload-management`. It runs in every operator's cluster with
cluster-admin rights and holds the node key.

1. **`200` with an empty list deletes every workload.** (Fixed for unknown nodes: the reference backend now
   answers `404`, code `1012`, for a node address it never registered, and the manager keeps workloads on
   any non-200. Registered nodes and soft-deleted nodes still get `200` and `[]`, so the risk remains for a
   known node and for a backend that returns `200 []` by mistake.) The sync (every 10 minutes) treats the backend's
   list as the source of truth and deletes all Deployments in the namespace (except `ipfs` and
   `lfg-workload-manager`) not in the list, including when the list is empty. A buggy deploy, a wrong
   `NODE_SYSTEM_ENDPOINT` that answers `{"data": []}`, or a database reset wipes all workloads on all nodes
   within one interval. Non-200 responses and network errors are safe.
2. **Self-update is unsigned.** With `RELEASE_URL` set it reads `$RELEASE_URL/stable.txt` every 30 minutes
   and patches itself to `<repo>:<text>` with `imagePullPolicy: Always`; no status check, signature or digest.
   Leave `RELEASE_URL` unset to disable.
3. **Manifests are not verified** (not checked against the CID, status not checked) and applied with admin
   credentials; privileged pod specs are allowed.
4. **Secrets in env:** `NODE_PRIVATE_KEY` and `KUBE_CONFIG`; TLS verification of the Kubernetes API is
   disabled.
5. **Replayable signatures** (no nonce; GET requests are unauthenticated).
6. **No authentication on port 6161:** `/metrics?projectCode=` marks a workload alive for anyone who can
   reach it, which affects uptime accounting.
7. **HTTPS only and custom DNS:** plain `http://` endpoints fail, names that only resolve through
   `/etc/hosts` or private DNS cannot be used.
8. The image runs as root (and `EXPOSE 3000` is wrong metadata). Sentry receives error logs and the node
   address when configured.

## Backend

Repository `opensource-node-system-be`. It controls how uptime and points are recorded and what is committed
on-chain by the submitter key.

| ID | Issue | Impact |
|---|---|---|
| S-04 | `POST /api/v1/projects/:code/node-points`: `nodes` is not validated as nested DTOs and `point` is concatenated into a raw SQL string run by the raw `pg` client (multiple statements allowed) | **SQL injection by any wallet in `project_admins` of any project**: arbitrary read and write of the whole database. Until fixed, insert only trusted wallets into `project_admins` |
| S-08 | `DELETE /api/v1/nodes/:nodeAddress` only verifies that the caller signed the node address, never that they own the node; node addresses are public (they appear in IPFS summaries) | Anyone can soft-delete any node and its workloads; heartbeats are rejected until it is re-registered |
| | All signed endpoints: EIP-191 over the raw body, **no nonce, timestamp or audience** | Replay of any signed action forever |
| | `GET /user/settings`, `GET /nodes`, `GET /inventory/*`, `POST /notifications` trust the `X-User-Address` header without a signature | Anyone who knows a wallet address can read its settings (including the alert **e-mail address**), nodes, inventory, notifications |
| | `balanceUpdater.ts`, `uptimeSnapshot.ts`, `project.service.ts` build SQL by string concatenation | A malicious or compromised subgraph can inject SQL |
| S-11 | License limits are only strictly applied in the DAILY snapshot; deleted workloads can still receive heartbeats while a Redis key exists; hourly snapshots assume every row valid | A wallet can have more workloads than licenses in HOURLY roots; DAILY re-applies the limit |
| | PostgreSQL TLS is enabled (without certificate verification) only if the URL contains `amazonaws.com` | Plain-text DB traffic elsewhere |
| | Redis `CONFIG SET notify-keyspace-events KEA` at boot; only database 0 | Boot failure on managed Redis that forbids `CONFIG`; silent loss of offline notifications on another database |
| | Node IPs are sent over plain HTTP to `ip-api.com` (free tier, non-commercial, 45 requests per minute) | Privacy leak of node IPs; licence violation for commercial use |
| | If IPFS upload or the on-chain transaction fails, the error is logged and sent to Slack but the summary is still marked `COMPLETED`; root, CID and tx hash are not stored | Missing on-chain checkpoints with no automatic retry |
| | `GET /public/cli-constants` returns whatever you configure and the CLI trusts it | Whoever controls the backend config can make operators run arbitrary images |
| | `config.yaml` holds the submitter key, Pinata JWT, SMTP password and cron secret in plain text | Key compromise means the ability to submit roots |
| | Swagger exposes the whole API when enabled; `GET /cron/uptime-summary` is unauthenticated | Information disclosure |

Also: single instance only, no automated tests, `smtp.port` ignored. Details and mitigations:
`SECURITY.md` in the repository.

Operational known issues (not fixed):

- A workload session left open by a backend outage is never closed (`endTime` stays `NULL`) and is excluded
  from summaries.
- `POST /nodes` validation errors are a bare `Bad Request Exception`; registry reverts are not decoded in the
  backend log, and the log does not print the submission's transaction hash.

## Contracts

**`GlobalNodeUptimeMerkleRegistry`** (`opensource-merkle_tree_uptime`):

- `SUBMITTER_ROLE` holders fully decide which root is stored for an empty hour or day. The contract checks
  timing, non-zero root, CID length (46 to 128) and that 24 hourly roots exist before a daily root; it does
  **not** check that a daily root matches the hourly roots or that the CID content matches the root. A
  compromised submitter key can publish a wrong root for any empty slot.
- `DEFAULT_ADMIN_ROLE` can `backfillRoot` (overwrite) **any** non-future id, including published ones, and
  pause. The overwrite is visible through `RootBackfilled`. Use a multisig and consider a timelock.
  Anything that pays out from a root must treat admin and submitter as trusted.
- One global root per slot: **one registry per backend** (a backend may serve several projects). First write wins; a missed hour blocks the daily
  root until backfilled; the 2-hour offset is hard-coded; the daily id refers to the previous day; the leaf
  has no domain separation (no chain id, registry address or id); the CID is only length-checked.

**`NodeOperatorsReward`** and **`HelixToken`** (`opensource-node-operators-reward`):

- **Signer is spending authority.** Any `SIGNER_ROLE` holder can authorize payouts of the entire token
  balance, one claim at a time. Keep the key in a KMS and rotate by granting a new signer and revoking the
  old one.
- **`emergencyWithdraw(token)` sends the contract's FULL balance** (or native coin for `address(1)`) to the
  caller. No amount or recipient parameter; it also drains tokens for which users hold signed claims.
  Treat `DEFAULT_ADMIN_ROLE` as full custody (multisig).
- **Raw ECDSA digest, not EIP-712 or EIP-191.** Wallets cannot show a readable payload and the signature is
  not wallet-safe. Chain id and contract address are bound in the digest, preventing cross-chain and
  cross-contract replay.
- Replay protection is a per-receiver sequential nonce plus a deadline. If the off-chain signer loses sync
  and signs an already-paid amount with the *current* nonce, the contract pays it: it only knows what was
  signed. Two claims signed with the same nonce can execute only once.
- No timelock and no pause on NOR; admin can grant any role. `claim` is callable by anyone with a valid
  signature (tokens go to `receiver`). Native coin cannot be sent to NOR.
- **`HelixToken` owner controls everything**, including UUPS upgrades, emission and whitelists: a
  compromised owner can upgrade to arbitrary code. Use a multisig and a timelock. Burn functions are
  whitelist-gated; the `Permanntly` typo is part of the ABI; name and symbol are hard-coded.

## Subgraphs

Both only read public chain data and hold no keys. Never commit Goldsky or Graph tokens (use CI secrets and a
git-ignored `.env`). Their data is only as trustworthy as the chain data and your config: a wrong
`startBlock` gives wrong license counts (which limit workloads) or wrong nonces (the claim then reverts
`InvalidNonce`); the reward subgraph's `nonce` equals the contract's `userNonce` only if it indexed from the
deployment block, so signers must treat the contract as the source of truth. Hosted GraphQL endpoints have
no authentication.

## nodesale-reward

Repository `opensource-nodesale-reward`. It holds three keys per project and a database that is the **only
record** of operators' unclaimed balances and nonces.

1. **String-built SQL** in the daily bulk upsert (`$executeRawUnsafe`), fed by wallet addresses and numbers
   from the backend's `node-uptimes` response: a malicious backend (or `http://` man-in-the-middle) can
   inject SQL. Point `api_url` only at a backend you control, over HTTPS; use a limited database role.
2. **Equal split per node with a `gain_point` multiplier:** shares sum to `amount * gain_point`. **With
   `gain_point > 1` the contract and database are over-credited.** Keep it 1. Not weighted by uptime or
   tier; a few wei of dust per wallet per day.
3. **The claim endpoint is unauthenticated by design** and has no rate limit: anyone can read any wallet's
   balance and nonce, and it signs even when `amount` is 0. Put it behind a proxy or WAF.
4. **Admin auth:** `x-offchain-*` headers, one admin address per instance, 60-second window with no nonce
   store and no upper bound on future timestamps, so a captured request replays for up to a minute. Swagger
   uses a static Basic-auth password; do not enable it publicly.
5. **Balance accounting depends on one in-memory cursor and one process:** the cursor advances before the
   database commit, no overlap guard, `unclaim` decremented without a lower bound, a claim event for an
   unknown wallet aborts the batch. If the database drifts from the chain the service may sign vouchers for
   already-claimed amounts (the contract nonce is the last defence). Run exactly one instance. The `nonce`
   column is a `SMALLINT` (max 32767).
6. **Daily job fragility:** no way to re-run a missed day; node-points are posted page by page **before** the
   local write (part-way failure leaves points submitted with no local rows); the process time zone is used
   in places (run `TZ=UTC`); `meta.total` is trusted as the divisor.
7. Trust in upstreams (backend GETs are unauthenticated; subgraph; RPC). `.env.yaml` holds private keys in
   plain text; request bodies are logged and reward SQL is logged at `info`. The Alchemy SDK is a hard
   dependency (the RPC URL can be overridden per chain with `rpc:` in `.env.yaml`) and unlisted chain ids
   fail at runtime. No health route, no tests.

## Workload tooling

- **Deployer** (`opensource-nodesys-deployer`): the CID returned by your IPFS upload API is signed as-is and
  never recomputed locally, so a malicious upload API can make you publish a CID for different content
  (run it yourself, over HTTPS). The signed payload has no nonce or expiry. The namespace is forced to
  `lfg-user`. The action logs the request URL and the signed message (not the signature headers), so keep
  secrets out of `code` and `version`. The publisher key must be a dedicated, **fundless** key stored only as
  a GitHub secret, rotated (and `project_admins` updated) if leaked.
- **Workload template:** the workload talks to `MASTER_URL` without authentication and pins whatever CIDs it
  returns; `node_id` is self-declared. Heartbeats prove only that a container runs. Operators run your image
  with the Deployment's privileges: publish from a registry and account you control. Review the third-party
  `fluxion-labs/versioning-action` before bumping it.
- **Workload master sample** (`opensource-nodesys-workload-master-sample`):
    - **No authentication on any endpoint.** `POST /static/text` and `/static/file` let anyone upload to your
      IPFS node and Pinata account and add CIDs every node will pin; `GET /cids/:cid` downloads any CID. Put
      it behind a firewall or access-controlled proxy.
    - `node_id` is unverified; anyone can report CIDs as pinned for any node (not used for points). CORS is
      `cors.Default()` (all origins).
    - Some handlers do not `return` after errors (duplicate responses) and `/workload/report` always replies
      `200 null` after a store error; the point submitter has no retry or idempotency.
    - `app.yaml` holds the MongoDB URI, Pinata JWT and the point-submitter private key (use a fundless key).
      **The git history of the original private repository contained live credentials**; rotate them and
      publish forks only from a clean history. Dependencies date from mid-2024: run `govulncheck ./...`.

## Dashboard

GET endpoints send only `X-User-Address` and the backend does not verify it, so node and reward data is
effectively public and the UI shows whatever address a wallet reports (including a saved alert e-mail).
Writes are signature-gated but the client adds no nonce or timestamp, so replay protection is the backend's
job. Everything `NEXT_PUBLIC_*` is public (restrict your Alchemy and WalletConnect keys by origin). With a
Sentry DSN, session replay runs with `maskAllText: false`, `blockAllMedia: false`, so addresses and e-mails
may be sent to Sentry. `next/image` allows any https host. `/verify-uptime` makes no network calls and does
not authenticate the dataset.

## Recommendations

In rough priority order for a self-hoster:

1. **Protect the supply chain of operator machines.** Restrict write access to the release bucket, version
   files, image registry, IPFS upload API and the backend `cli` config. Publish checksums and verify them in
   your own install path; sign releases (cosign) and pin image digests; consider implementing signature
   verification in your fork of the CLI and manager before operators grow in number.
2. **Never let the backend answer `200` with an empty workload list by mistake** (unknown node addresses
   already get `404`, code `1012`; a registered node with none left does not). Before maintenance,
   migrations or shutdown return a non-200 (LFG's own shutdown plan returns `410` on every route). Guard the
   manager sync (skip deletions when the list is suddenly empty) in your fork.
3. **Fix or contain S-04 and S-08 in the backend** before exposing it: add `@ValidateNested`/`@Type` and
   parameterized queries, check ownership on node deletion. Until then only insert fully trusted wallets into
   `project_admins` and put the API behind a proxy.
4. **Add nonce and expiry to signed requests** on the backend (and CLI, manager, deployer, services), and
   require signatures on the `X-User-Address` read routes if you consider the data private.
5. **Secure keys.** Dedicated, low-balance submitter, claimer, publisher and points keys; signer in a KMS;
   multisig for registry admin, NOR admin and token owner (with a timelock for the UUPS token); restrict
   file permissions of `config.yaml`, `.env.yaml`, `app.yaml`; rotate anything ever pasted or committed.
6. **Monitor chain submissions.** The backend swallows failures: watch Slack or logs, keep the submitter
   funded, and alert on gaps in registry roots; remember `MissingHourlyRoots` blocks the daily root.
7. **Run singletons as singletons:** one backend instance, one nodesale-reward instance per database, one
   registry per backend. Back up PostgreSQL, especially the reward database.
8. **Harden operator machines:** bind the Kubernetes API to `127.0.0.1`, `chmod 600` the CLI config
   directory, mount the node key from a Secret file and use an in-cluster ServiceAccount instead of the
   kubeconfig, run images as non-root, restrict pod specs.
9. **Put everything behind TLS and a reverse proxy** with rate limits; keep Swagger off; use a licensed or
   self-hosted GeoIP instead of `ip-api.com`; verify dependencies (`bun audit`, `govulncheck`, Dependabot).

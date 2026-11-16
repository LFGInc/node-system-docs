# Node CLI (`lfgnode`)

Repository: [`opensource-node-cli`](https://github.com/LFG-Incorporated/opensource-node-cli) (Go). The
repository README is the full reference; this page summarizes what you need to operate and release it.

`lfgnode` is what a node operator runs. It creates a local Kubernetes cluster (k3s inside Docker, via
[k3d](https://k3d.io)), deploys the [workload manager](workload-manager.md) and an IPFS (kubo) node into
it, and adds or removes project workloads that are licensed by NFT ownership. It also holds the node's
identity.

!!! warning "No hosted defaults"
    The open-source CLI points at no hosted service. `api-endpoint`, `release-endpoint` and
    `workload-manager-image` must come from your config, the environment, or your build. Read the
    [CLI security notes](security.md#node-cli) before running a network: whoever controls your backend,
    release host, IPFS gateway or registry can run code as root on operators' machines.

## How it works

- **Identity.** The node id is the address of a secp256k1 key stored in `config.yaml`
  (`lfgnode config private-key generate|set`). It is not an NFT id. The wallet that owns the licenses
  links the node address to itself on the backend (dashboard, or `POST /nodes`).
- **`lfgnode start`** tells the backend about the node (`PATCH /nodes/<address>` with CLI version and OS),
  creates or starts the k3d cluster `lfg-node`, creates namespace `lfg-user`, the ConfigMap
  `lfg-node-config`, and the Deployments `lfg-workload-manager` and `ipfs`.
- **`lfgnode workload add <project>`** asks the backend whether the owner wallet holds enough licenses.
  The backend returns the manifest CID and the reward address; the CLI downloads the manifest from the
  IPFS gateway, writes the secret `lfg-workload-<project>` (`REWARD_ADDRESS`) and creates the Deployment.
  From then on the workload manager keeps it up to date.
- Workloads report liveness to the workload manager (`GET :6161/metrics?projectCode=`), which reports
  uptime to the backend.

## Requirements

Operators:

- Docker Engine (Linux) or Docker Desktop (macOS, Windows). On Docker Desktop enable "Use WSL 2 based
  engine" (Windows) and "Add the `*.docker.internal` names to the host's `/etc/hosts` file".
- Outbound access to your backend, release host, IPFS gateway, image registry and Docker Hub
  (`rancher/k3s:v1.31.0-k3s1`, `ipfs/kubo:v0.29.0`).
- `sudo` on Linux/macOS (`lfgnode update` installs to `/usr/local/bin`). The install script supports
  Ubuntu 22.04 or newer with at least 1.5 GB RAM.

Building: Go 1.23 or newer (`go.mod` says 1.23.1); CGO is not required.

## Configuration

Resolution order, first match wins:

1. environment variable;
2. `config.yaml` key;
3. value compiled in at build time (only `api-endpoint`, through `-ldflags`);
4. value from the backend's `GET /api/v1/public/cli-constants`, which only fills settings still empty.

`config.yaml` location (created on first run; `lfgnode` is the fixed directory name):

| OS | Path |
|---|---|
| Linux | `~/.config/lfgnode/config.yaml` |
| macOS | `~/Library/Application Support/lfgnode/config.yaml` |
| Windows | `%AppData%\lfgnode\config.yaml` |

| `config.yaml` key | Environment variable | Default | Meaning |
|---|---|---|---|
| `api-endpoint` | `LFGNODE_API_ENDPOINT` | none (optionally compiled in) | Backend base URL, no trailing slash. The CLI appends `/api/v1/...`. Also passed to the manager as `NODE_SYSTEM_ENDPOINT` and stored in the ConfigMap as `nodeServerBaseUrl` |
| `release-endpoint` | `LFGNODE_RELEASE_ENDPOINT` | none; **may be empty** (see below) | Release host base URL. Read: `<release-endpoint>/releases/stable.txt` or `unstable.txt` and `<release-endpoint>/releases/<version>/lfgnode_<OS>_<arch>.tar.gz` (self-update), `<release-endpoint>/workload-manager/stable.txt` (manager image tag). Passed to the manager as `RELEASE_URL=<release-endpoint>/workload-manager` |
| `workload-manager-image` | `LFGNODE_WORKLOAD_MANAGER_IMAGE` | none | Manager image. With a `release-endpoint`: the repository **without tag**, for example `public.ecr.aws/my-alias/node-system/workload-manager` (the tag comes from `stable.txt`). Without one: the full image **with an explicit tag**, for example `ghcr.io/my-org/workload-manager:v1.0.0`. Must be anonymously pullable |
| `ipfs-gateway` | `LFGNODE_IPFS_GATEWAY` | `https://ipfs.io` | Gateway for manifests, fetched as `<gateway>/ipfs/<cid>/`. Passed to the manager as `IPFS_GATEWAY` |
| `sentry-dsn` | `LFGNODE_SENTRY_DSN` | empty (disabled) | Optional Sentry DSN. Error logs, 10% sample, node address as user id. Passed to the manager as `SENTRY_DSN` when set |
| `cluster.name` | | `lfg-node` | Written by the CLI. **Do not change** |
| `cluster.port` | | `6445` | Host port of the Kubernetes API (written by the CLI) |
| `node.private-key` | | none | 64 hex characters. Set with `lfgnode config private-key generate` or `set <key>`. Only changeable while no cluster exists (`lfgnode down` first) |

**Empty `release-endpoint`.** It is optional. When it is empty, the manager image is used exactly as given in
`workload-manager-image` (so put an explicit tag there), no `RELEASE_URL` is passed to the manager and
**self-update is disabled** for both the manager and `lfgnode update`. Earlier CLI builds exited with
`release endpoint is not configured` in this case; the open-source release does not. Use this for local tests and
for networks that pin their versions.

A complete annotated example is [`config.example.yaml`](https://github.com/LFG-Incorporated/opensource-node-cli/blob/main/config.example.yaml).
The CLI rewrites the file on each start and keeps unknown keys. Other files
in the config directory: `cluster.yaml` (k3d definition, [see below](#clusteryaml)) and `workloads/*.yaml` (copies of workload
secrets).

!!! warning "`config.yaml` holds the node private key and is written world-readable (0644)"
    Run `chmod 600` on it (Linux: `~/.config/lfgnode/config.yaml`, macOS: `~/Library/Application Support/lfgnode/config.yaml`)
    right after the first start, and keep the directory private on shared machines.

Environment variables the CLI sets on the workload manager Deployment:

| Variable | Value |
|---|---|
| `NODE_ENV` | `development` for non-release builds, `beta` if the version contains `rc`, else `production` |
| `NODE_PRIVATE_KEY` | the node key |
| `NODE_SYSTEM_ENDPOINT` | `api-endpoint` |
| `KUBE_CONFIG` | contents of the operator's `~/.kube/config` (see [security](security.md#node-cli)) |
| `IPFS_GATEWAY` | `ipfs-gateway` |
| `RELEASE_URL` | `<release-endpoint>/workload-manager`, only if `release-endpoint` is set (without it the manager does not self-update) |
| `SENTRY_DSN` | only when configured |

### Remote constants

On every invocation, if an `api-endpoint` is known, the CLI calls
`GET <api-endpoint>/api/v1/public/cli-constants` (10 s timeout; failures are only logged) and takes
`WORKLOAD_MANAGER_IMAGE`, `RELEASE_ENDPOINT`, `IPFS_GATEWAY` and `SENTRY_ENDPOINT` from
`{"status":1,"data":{...}}` **for settings that are still empty**. The backend serves them from its `cli`
config section. This lets a backend operator move image or release host without a CLI release, and also
lets a compromised backend redirect nodes. To make it irrelevant, set everything locally. A 404 is
harmless. (The Sentry value has no effect today because the logger starts before the fetch.)

## Build

```bash
go build -o lfgnode .          # development build, version "development"
go vet ./...
# release build, as GoReleaser does:
go build -ldflags "-X lfgnode/pkg.VERSION=v1.2.3 -X lfgnode/pkg.API_ENDPOINT=https://nodesystem.example.com" -o lfgnode .
```

`VERSION` selects the update channel: versions containing `rc` and the `development` build use
`unstable.txt`, all others `stable.txt`. `API_ENDPOINT` is an optional baked-in default backend;
operators can still override it.

## Releasing in your fork

`.github/workflows/release.yml` runs on any pushed tag and uses GoReleaser to build
`linux | darwin | windows` for `amd64 | arm64`. Artifacts go to the GitHub Release of your repository and
to an S3 bucket under `releases/<tag>/` (`lfgnode_<Os>_<arch>.tar.gz`, `.zip` for Windows,
`checksums.txt`). Afterwards `releases/stable.txt` (or `unstable.txt` for tags containing `-rc`) is
overwritten with the tag. The bucket must be served over HTTPS; its base URL is the `release-endpoint`.

| Kind | Name | Meaning |
|---|---|---|
| Variable | `AWS_BUCKET` | S3 bucket for releases |
| Variable | `AWS_REGION` | bucket region |
| Variable | `NODE_SYSTEM_API_ENDPOINT` | backend URL compiled into stable builds (may be empty) |
| Variable | `NODE_SYSTEM_API_ENDPOINT_BETA` | backend URL compiled into `-rc` builds (may be empty) |
| Secret | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | credentials that can write to the bucket |

Release the [workload manager](workload-manager.md#releasing-in-your-fork) first, into the same bucket.
Use a version number higher than the last LFG release (`v0.1.5`) so your build is recognizable.

## Installing (operators)

`scripts/install.sh` is **not** uploaded by CI. Copy it to `<bucket>/releases/` yourself, or have operators
download the archive for their platform from the GitHub Release and put `lfgnode` on their `PATH`.

```bash
export LFGNODE_RELEASE_URL=https://my-bucket.s3.us-east-1.amazonaws.com/releases
curl -sL "$LFGNODE_RELEASE_URL/install.sh" | bash                              # stable channel
curl -O "$LFGNODE_RELEASE_URL/install.sh" && bash install.sh -c beta           # unstable (-rc) channel
```

The script requires `LFGNODE_RELEASE_URL` (note: this one includes `/releases`, unlike the
`LFGNODE_RELEASE_ENDPOINT` setting) and installs to `/usr/local/bin`. Its channel names are `alpha`
(stable) and `beta` (unstable).

## Commands

```bash
export LFGNODE_API_ENDPOINT=https://nodesystem.example.com        # or edit config.yaml
export LFGNODE_RELEASE_ENDPOINT=https://my-bucket.s3.us-east-1.amazonaws.com
export LFGNODE_WORKLOAD_MANAGER_IMAGE=public.ecr.aws/my-alias/node-system/workload-manager

lfgnode config private-key generate      # prints the new key; or: set <hex>
lfgnode info                             # node address and public key: link this address to your wallet
lfgnode start                            # create or start the cluster
lfgnode workload add <project-code>      # for example: valley
lfgnode status
lfgnode workload list | status <p> | logs <p> | restart <p> | remove <p> | config <p> --env K=V
lfgnode stop                             # stop the cluster (keeps data)
lfgnode down                             # remove cluster and workloads
lfgnode update                           # self-update: stops cluster, replaces binary, restarts
lfgnode licenses                         # licenses the backend sees for this node's owner wallet
lfgnode version
```

Put the variables in your shell profile or `config.yaml`: `lfgnode update` restarts the cluster through
`lfgnode start`. Back up `config.yaml`; it holds the node key, and the node address is the node's
identity and reward link.

`lfgnode update` resolves the version from `<release-endpoint>/releases/<channel-file>`, downloads the
archive for the current OS and architecture, replaces the binary, and restarts the cluster. It verifies
nothing (see [security](security.md#node-cli)).

## Backend endpoints the CLI calls

All paths are relative to `<api-endpoint>/api/v1`; `<addr>` is the lower-case node address. Responses are
`{"status":1,"data":...,"message":"..."}`; a status other than `1` is an error with `message`.
`POST`/`PATCH`/`PUT` bodies are JSON and carry `X-Signature`, the hex EIP-191 (`personal_sign`)
signature of the raw body made with the node key (no nonce or timestamp).

| Method and path | Used by | Purpose |
|---|---|---|
| `GET /public/cli-constants` | every command | optional constants (above) |
| `GET /healthcheck` | `status` | liveness |
| `PATCH /nodes/<addr>` body `{version, os}` | `start` | register or update node info |
| `GET /nodes/<addr>` | `status` | node uptime (`OnlineToday`, `OnlineYesterday`, ms) |
| `GET /nodes/<addr>/licenses?mode=cli` | `licenses` | licenses of the owner wallet |
| `GET /nodes/<addr>/workloads?onlyVersion=true` | `down` | workloads assigned to the node (`404` code `1012` if the node is unknown to the backend) |
| `POST /nodes/<addr>/workloads` | `workload add` | check license, returns `{ipfsCid, rewardAddress, ...}` |
| `GET /nodes/<addr>/workloads/<project>` | `workload status` | per-workload uptime |
| `DELETE /nodes/<addr>/workloads/<project>` body `<addr>|<project>`, **signed** (`X-Signature`, EIP-191 over `<addr>|<project>`) | `workload remove`, `down` | free the license slot |

!!! note "Signed delete"
    Earlier CLI builds sent this `DELETE` without a signature. The backend answered `401` and the CLI ignored
    it, so `workload remove` and `down` left the license slot occupied on the backend. The current CLI
    signs the message `<nodeAddress>|<projectCode>` with the node key, and the slot is freed.

The workload manager additionally calls `PATCH /nodes/<addr>/uptime`,
`GET /nodes/<addr>/workloads?onlyVersion=true` and `PATCH /nodes/<addr>/workloads/<project>`.

## Known issues and local-test tips

| Topic | Detail |
|---|---|
| **Do not use `-v/--verbose`** | It crashes with `flag provided but not defined: -v` (stdlib flag parse in `pkg/kubectl`) **after** the cluster is created but **before** the manager and ipfs are deployed, leaving a half-initialised cluster. Recovery: `lfgnode down` (or `k3d cluster delete lfg-node`) and start again without the flag |
| **One node per machine** | The cluster name `lfg-node`, the Kubernetes API port `6445` and the config directory are fixed, so a machine runs exactly one node. An old, stopped cluster with that name must be removed before a new `lfgnode start`. See [multiple projects](#one-node-many-projects) |
| **Separate test config** | Run with a separate `HOME` (for example `HOME=$PWD/testhome lfgnode ...`) to keep a test config apart from a real one. The config directory is `~/Library/Application Support/lfgnode` on macOS and `~/.config/lfgnode` on Linux |
| **Private / HTTP image registry** | `cluster.yaml` (editable before the first start) accepts a k3d `registries.config` mirror, see below |
| **http endpoints** | The CLI accepts `http://` for `api-endpoint`, `release-endpoint` and `ipfs-gateway`; only the workload manager is https-only ([Workload manager](workload-manager.md#local-development)) |
| **Self-update** | With an `http://` release URL self-update breaks. Leave `release-endpoint` empty in tests (supported: the manager image is then taken from `workload-manager-image`, which needs an explicit tag). If you do serve a release endpoint over http, dev builds read `workload-manager/unstable.txt` and the manager still cannot self-update |
| **Self-signed TLS** | The Go CLI rejects self-signed `https://` and has no insecure flag: give it `http://<ip>.sslip.io:<httpPort>` and patch the manager Deployment afterwards ([recipe](local-testing.md#self-signed-tls-for-the-manager)) |

### cluster.yaml

`cluster.yaml` is the k3d definition. The CLI **writes it only if the file does not exist**, on the first
`lfgnode start`, and reads it at every start; changes take effect only when the cluster is (re)created. To add
`registries:` (or anything else) **before the first start**, pre-create the file in the config directory with
the default content below:

```yaml
apiVersion: k3d.io/v1alpha5
kind: Simple
metadata:
  name: lfg-node
servers: 1
agents: 0
kubeAPI:
  hostIP: "0.0.0.0"
  hostPort: "6445"
image: rancher/k3s:v1.31.0-k3s1
options:
  k3d:
    wait: true
    timeout: "60s"
    disableLoadbalancer: false
    disableImageVolume: false
  kubeconfig:
    updateDefaultKubeconfig: true
    switchCurrentContext: true
  runtime:
    ulimits:
      - name: nofile
        soft: 1024
        hard: 1024
```

Keep `name: lfg-node` and the port ([fixed names](fixed-names.md)). To use a plain-HTTP registry on the host
(port `5001` is a placeholder), append a `registries:` block to the same file:

```yaml
registries:
  config: |
    mirrors:
      "host.k3d.internal:5001":
        endpoint:
          - http://host.k3d.internal:5001
```

The block goes at the top level of the file, next to `options:`. Then use `host.k3d.internal:5001/...` image
names. The full local path is in [Local testing](local-testing.md).

## One node, many projects

`lfgnode` is one node per machine, and a node belongs to **one backend**. The backend itself is multi-project:
one CLI, one node and many workloads (`lfgnode workload add <project>` for each) can serve every project that
this backend knows, and one licensed wallet can earn in all of them.

An operator who earns in projects that run on **different** self-hosted backends (for example Valley and
FlipIt after leaving LFG, each with its own backend) needs **one machine or VM per backend**, unless the
projects share one backend. See [Migration from LFG](migration-from-lfg.md#one-node-per-backend).

## When the backend is down

- **No endpoint configured:** commands that need the backend fail with
  `node system API endpoint is not configured ...`. `version`, `info`, `config` and `stop` still work.
- **Backend unreachable:** every invocation first waits up to 10 s on `cli-constants` (warning only).
    - `start` **fails**, because it must `PATCH /nodes/<addr>` first: a stopped node cannot be restarted
      while the backend is down. A running cluster is unaffected.
    - `workload add`, `workload remove`, `workload status`, `status`, `licenses` fail. `workload list`,
      `logs` and `restart` are local.
    - `down` fails **before** removing the cluster (it first lists workloads from the backend).
      Work-around: `k3d cluster delete lfg-node` or `docker rm -f`.
    - `stop`, `info`, `version`, `config` do not use the backend; `update` only needs the release host.
- **Inside the cluster** the workload manager keeps running workloads running, logs and retries failed
  reports, and skips a sync whose list fetch failed. The backend records no uptime while it is down.
  A `200` with an empty list is the dangerous case ([security](security.md#workload-manager)); the backend
  now answers `404` (code `1012`) for nodes it has never seen, which the manager treats as an error and
  keeps the workloads.

## Repository layout

```text
main.go                       load config, apply settings, fetch remote constants, run cobra
cmd/                          commands (start, stop, status, update, down, info, licenses, config, workload ...)
internal/api/nodesys/         signed HTTP client for the backend
internal/cluster/k3d/         k3d cluster, workload-manager and ipfs deployment
internal/workload/            workload Deployments and secrets in namespace lfg-user
pkg/config/                   config.yaml handling; settings.go resolves env/config endpoints
pkg/constants/, pkg/metadata.go   names and (empty) endpoint variables, version/channel
scripts/install.sh            Linux installer (needs LFGNODE_RELEASE_URL)
.goreleaser.yaml, .github/workflows/release.yml
```

Migrating from the LFG-hosted system: [Migration from LFG](migration-from-lfg.md).

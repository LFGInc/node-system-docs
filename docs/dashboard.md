# Dashboard

Repository: [`opensource-nodesys-ui`](https://github.com/LFG-Incorporated/opensource-nodesys-ui) (Next.js 15
App Router, React 19, TypeScript). The operator dashboard: operators connect a wallet and see their nodes,
per-workload uptime, inventory and rewards, and notifications. It talks only to the
[backend](backend.md); there is no database and no hosted-service dependency. Fork it, point it at your
backend, set your branding.

Operators need some way to link a node to their wallet (`POST /nodes`); the dashboard's "add node" action is
the ready-made one.

## What it does

| Route | Purpose |
|---|---|
| `/` | Wallet's nodes (add, rename, delete), per-node workload status and hourly uptime, stats, daily countdown |
| `/inventory` | Per-project holdings and accumulated rewards or points for the connected wallet |
| `/inventory/[slug]` | Reward sessions for one project, filterable by date |
| `/distribution/[slug]` | Distribution history and stats for one project |
| `/verify-uptime` | Client-side Merkle proof tool: paste the published uptime dataset (JSON array of `{slug, nodeId, uptime}`), rebuild the tree with `@openzeppelin/merkle-tree`, get the root and proof to compare with the on-chain [registry](contracts.md#uptime-registry). No network calls. Leaf = `keccak256(abi.encode(slug, nodeId, uptime))` |
| Header bell | Notifications and e-mail alert settings |

Authentication is wallet-based (wagmi + RainbowKit; Ethereum mainnet and sepolia transports through
Alchemy). There is no session. Reads send `X-User-Address`; writes ask the wallet to `signMessage` and send
`X-Signature` (for delete the message is the node address, for rename and e-mail the JSON payload string).
Every request goes to `NEXT_PUBLIC_BACKEND_URL`; the endpoints used are the public and signed routes listed
in [Backend](backend.md#http-api). **Your backend must allow the dashboard origin** in
`server.corsWhitelist`.

The node list loads once on page load (then refreshes every 3 minutes): after switching wallets, reload
the page.

## Configuration

Copy `.env.example` to `.env.local`. `NEXT_PUBLIC_*` values are inlined at **build** time: change them,
rebuild. They are public: never put secrets in them, and restrict the Alchemy key and WalletConnect project
id by origin in the provider dashboards.

| Variable | Default | Notes |
|---|---|---|
| `NEXT_PUBLIC_BACKEND_URL` | none | **Required**; the app throws at startup without it. The reference backend listens on `7777` by default, so the local value is `http://localhost:7777` |
| `NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID` | empty | **Effectively required**: RainbowKit throws during build without one. Any non-empty placeholder is enough at build time; only real WalletConnect sessions need a valid id |
| `NEXT_PUBLIC_ALCHEMY_KEY` | empty | Mainnet / sepolia RPC |
| `NEXT_PUBLIC_APP_ENV` | `development` | `production` lowers the Sentry sample rate and silences debug logs |
| `NEXT_PUBLIC_TARGET_TIME` | `23:55` | Daily countdown target, `HH:mm` UTC |
| `NEXT_PUBLIC_LFG_UPTIME_CONTRACT_ADDRESS` | empty | Registry address shown on `/verify-uptime` (name kept for compatibility). Must be set at **build** time, or the page shows no address |
| `NEXT_PUBLIC_APP_NAME` | `Node Dashboard` | Page title, wallet modal, copy |
| `NEXT_PUBLIC_BRAND_URL` | empty | Logo link (falls back to `/`) |
| `NEXT_PUBLIC_TERMS_URL`, `_PRIVACY_URL`, `_SUPPORT_URL` | empty | Footer links; hidden when empty |
| `NEXT_PUBLIC_SOCIAL_{X,DISCORD,LINKEDIN,YOUTUBE}_URL` | empty | Footer icons; hidden when empty |
| `NEXT_PUBLIC_DOWNLOAD_URL_{WINDOWS,MACOS,LINUX}` | empty | Header "Download" menu (your CLI build); hidden if all empty |
| `NEXT_PUBLIC_PROJECT_CLAIM_LINKS` | empty | JSON `{"<projectCode>":"<url>"}`: the inventory card links to that URL (for example your claim frontend) |
| `NEXT_PUBLIC_IMAGE_DOMAINS` | empty | Extra `next/image` hosts (any https host is already allowed) |
| `NEXT_PUBLIC_SENTRY_DSN` | empty | Runtime reporting; inert when unset |
| `SENTRY_ORG`, `SENTRY_PROJECT`, `SENTRY_AUTH_TOKEN` | empty | Source-map upload, active only when org and project are set |

## Build, run, deploy

Node 22 and pnpm 9.

```bash
pnpm install --frozen-lockfile
cp .env.example .env.local        # at least backend URL and WalletConnect id
pnpm dev                          # http://localhost:3000 (starts with --inspect)
pnpm build && pnpm start          # production build, any Node host; port 3000 clashes with the backend: pnpm start -p <port>
pnpm lint                         # note: runs next lint --fix and rewrites files
```

Any Node host (Docker, VM, PaaS) can run `pnpm start`; set the env vars in the **build** environment.
Vercel: `.github/workflows/deploy-production.yaml` (push to `main`) and `deploy-develop.yaml` (push to
`develop`) deploy through the Vercel CLI and need the secrets `VERCEL_TOKEN`, `VERCEL_ORG_ID`,
`VERCEL_PROJECT_ID` plus the app env vars in your Vercel project. The `cicd/*` branch trigger is for
pipeline testing; delete these workflows if you do not use Vercel.

## Branding

1. Set the `NEXT_PUBLIC_*` branding variables.
2. **Replace the image assets, which are still the original LFG artwork and are not env-driven**:
   `src/assets/images/logos/header-logo.png`, `src/assets/images/logos/lfg-footer.png`, `lfg-logo.svg`,
   `public/favicon.ico` (keep file names or update imports in `src/components/header/*` and `Footer.tsx`).
3. Colors and fonts: `tailwind.config.ts`, `src/styles/global.css`, `src/assets/fonts`. The bundled Avenir
   fonts are commercially licensed: replace them with fonts you may distribute.

## Hard-coded project behaviour to review

- **Helix and BSC:** `src/configs/projects.ts` holds the Helix special case; its rewards response is parsed
  differently (token amounts, unit `$LIX`) in `src/api/inventory/mapper.ts`, and `src/api/project/dto.ts`
  forces network `bsc` for workload types `helix` and `flipit` (all others `eth`). Edit these for your
  project.
- **`/verify-uptime` slug list** (`flipit`, `meer`, `valley`, `connoisseur`, `legacyconnoisseur`, `cerebro`,
  `helix`) is a hard-coded dropdown in `src/views/verify-uptime/VerifyUptime.tsx`, so a new project code
  cannot be selected until you edit the list there. It must match the slugs used when the Merkle root was
  built. Known issue.
- **`/verify-uptime` text hard-codes BSCScan and `ipfs.io`.** The page's instructions point at BSCScan as
  the explorer and `https://ipfs.io/ipfs/<CID>` for the tree file, whatever chain and gateway you use.
  Known issue (edit `VerifyUptime.tsx`). `ipfs.io` may be blocked or odd on some networks;
  `https://gateway.pinata.cloud/ipfs/<CID>` works.
- **`date` is required** on the distributions and rewards endpoints the dashboard calls
  (`/distributions/:code`, `/distributions/:code/stats`, `/inventory/:code/rewards`); calling them without a
  `date` query parameter fails.

Known issues: GET endpoints trust `X-User-Address` (any address's data can be displayed), Sentry session
replay does not mask text (`maskAllText: false`) so addresses and e-mails may reach Sentry, the Sentry
release is hard-coded as `1.0.0`, `SENTRY_SETUP.md` is outdated, and there are no tests. See
[Security](security.md#dashboard).

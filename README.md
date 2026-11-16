# opensource-node-system-docs

Documentation for the **nodesystem**: the open-source stack (node CLI, workload manager, backend,
uptime registry contract, subgraphs, reward services, dashboard, workload tooling) that LFG ran as a
hosted service until it was shut down. Each project forks the repositories and self-hosts everything;
nothing here depends on LFG infrastructure.

Published site: <https://lfg-incorporated.github.io/opensource-node-system-docs/>

Start with `docs/index.md` (overview and architecture) and `docs/self-hosting.md` (ordered setup
checklist). Projects that used the LFG-hosted system should also read `docs/migration-from-lfg.md`.

The code repositories (all in the GitHub organization `LFG-Incorporated`) are listed in the repo map
on the home page. Each repository README is the full reference for its configuration; these docs are
the map between them.

## Build and preview the docs

The site is built with [MkDocs](https://www.mkdocs.org/) and the
[Material theme](https://squidfunk.github.io/mkdocs-material/).

The Python dependency list is in **`requirement.txt`** (singular; the deploy workflow uses the same
file name).

```bash
python3 -m venv venv && . venv/bin/activate
pip install -r requirement.txt

mkdocs serve              # live preview at http://127.0.0.1:8000
mkdocs build --strict     # build into ./site, fails on any warning (what CI should satisfy)
```

## Publishing

`.github/workflows/deploy.yml` runs `mkdocs gh-deploy --force` on every push to `main` (and manually
via "Run workflow"). It publishes to the `gh-pages` branch; in the repository settings set
GitHub Pages to deploy from that branch.

## Contributing

1. Create a branch.
2. Edit files under `docs/` (navigation is in `mkdocs.yml`), run `mkdocs build --strict`.
3. Open a pull request.

Documentation rule of thumb: every config key, environment variable, endpoint, schedule and command
must match the README / example config of the component repository it describes. If they disagree,
fix the docs after checking the code.

## License

[MIT](LICENSE), Copyright (c) 2026 LFG-Incorporated.

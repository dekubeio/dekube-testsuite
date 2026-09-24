# dekube-testsuite

Regression and performance test suite for [dekube](https://dekube.io).

Compares dekube output between a **pinned reference version** and the **latest release** across a set of static edge-case manifests. The test harness uses dekube-manager (rolling from main) as the runner — it's the tool, not the subject. What's compared is core + extensions output between versions.

## Quick start

```bash
# Full regression: reference vs latest
./run-tests.sh

# Override reference core version
./run-tests.sh --core v2.0.0

# Override reference extension version
./run-tests.sh --ext keycloak==v0.1.0

# Test unreleased local work (latest side only; ref stays pinned)
./run-tests.sh --local-core /path/to/helmfile2compose.py --local-ext nginx=/path/to/nginx_rewriter.py
# --local-ext replaces only the named extension. An extension it depends on (e.g. trust-manager
# pulling in cert-manager) is still resolved and fetched by dekube-manager at its latest
# released tag — pass --local-ext for that dependency too if you need it overridden as well.

# Test bleeding-edge main branches instead of the latest release (latest side only)
./run-tests.sh --latest-main
# The dependency extensions (keycloak, nginx, cert-manager, ...) already come from main by
# default. --latest-main additionally takes the built-in/bundled extensions (indexers,
# workload, haproxy, caddy, emptydir, fix-permissions) from main instead of from the engine's
# latest release. The engine core logic itself still comes from the latest dekube-engine
# release: there's no pre-built single-file "engine from main" artifact to fetch (dekube-engine
# only publishes a built dekube.py on tagged releases). Ignored when combined with --local-core.

# Performance test (run locally, not in CI)
./run-tests.sh --perf 5         # fast
./run-tests.sh --perf 15        # notable
./run-tests.sh --perf 30        # pain
./run-tests.sh --perf 15 --keep # keep /tmp output for inspection
```

## What it does

Every download retries automatically (`curl --retry 3 --retry-all-errors`) — a single
mid-transfer network reset no longer aborts the whole run. Requires **curl >= 7.71.0**
(`--retry-all-errors`, added that version) — fine on any current OS/CI image, but an old
system curl (RHEL7/8 base images ship ~7.29/7.61) will fail hard with `option
--retry-all-errors: is unknown` instead of the pre-retry behavior.

### Regression

1. Downloads dekube-manager from `main`
2. Creates two workdirs in `/tmp` — one for the reference version, one for latest
3. Runs dekube multiple times per version:
   - **Core only** (no extensions) — baseline
   - **Each extension individually** — isolation testing
   - **All extensions together** — interaction testing
4. Diffs the output directories
5. Cleans up `/tmp` on exit
6. Exit 0 = identical, exit 1 = differences (informational)

### Performance (`--perf N`)

Run locally (not in CI — runners aren't meant for this). Generates O(n³) manifests:

| n  | Deployments | ConfigMap mounts | Approx. time |
|----|-------------|------------------|--------------|
| 5  | 25          | 125              | < 1s         |
| 15 | 225         | 3,375            | seconds      |
| 30 | 900         | 27,000           | notable      |
| 50 | 2,500       | 125,000          | pain         |

## Reading diffs

A diff **is expected** when things change intentionally between versions. The output is meant for human review — there are no assertions.

- `core-only` diff = pure dekube-engine behavioral change
- `ext-<name>` diff = change in that extension or its interaction with core
- `ext-all` diff = interaction between all extensions
- `ref run FAILED, latest OK` = a crash fixed in latest, diff unavailable for that combo
- `ref run FAILED, latest FAILED` = both sides crash for that combo — check the latest output before assuming it's the same bug
- `*.crt` / `*.key` files are never diffed: cert-manager regenerates key material on every run, so their content is noise, not drift

Before each combo runs, the latest side's output directory is pre-seeded with the reference run's `secrets/`, so idempotent generators (e.g. cnpg's superuser password) produce identical values on both sides instead of a spurious diff. CBA: a latest run that stops writing a secret the reference wrote is invisible under this scheme.

## Reference versions

Edit `dekube-known-versions.json` to bump the pinned reference:

```json
{
  "reference": {
    "core": "v2.2.0",
    "extensions": {
      "cert-manager": "v0.1.0",
      "keycloak": "v0.2.0",
      "servicemonitor": "v0.1.0",
      "trust-manager": "v0.1.1",
      "nginx": "v0.1.0",
      "traefik": "v0.1.0",
      "flatten-internal-urls": "v0.1.1"
    },
    "exclude-ext-all": [
      "flatten-internal-urls"
    ]
  }
}
```

Extensions listed here are tested individually; unlisted are skipped. Extensions in `exclude-ext-all` are excluded from the combined `ext-all` combo (e.g. due to incompatibilities declared in the registry).

**Future**: `exclude-ext-all` is a stopgap. The plan is to replace it with explicit `ext-sets` — named combos of extensions to test together.

## Reference vs latest — what actually gets measured

- **`core` in `dekube-known-versions.json`** = the pinned reference. It's fetched from the
  matching GitHub **release** tag (`releases/download/<tag>/...`).
- **`latest`** (default, no flags) = the engine's latest GitHub **release**
  (`releases/latest/download/...`) plus dependency extensions (keycloak, nginx, cert-manager,
  ...) pulled straight from each extension repo's **`main`** branch.
- Right after a rebaseline (`core` bumped to match the newest release), `ref` and the release
  half of `latest` are the same commit — a plain run measures nothing on the engine side until
  a new release is cut upstream. The dependency-extension half of `latest` still tracks `main`,
  so extension-only regressions are still caught in the interim.
- To get signal on **unreleased engine/extension work** before the next tag: `--local-core` /
  `--local-ext` swap in your own working copy on the `latest` side (ref stays pinned).
- To get signal on **main-branch built-in extensions** without a local checkout: `--latest-main`
  folds the built-in/bundled extensions' `main` branch into the `latest` core build (engine body
  itself is still the latest release — see `--help`).

## CI

The GitHub Actions workflow runs regression only (weekly + on push to test files). Performance tests are manual — run them on your own machine.

GitHub automatically disables a scheduled workflow's cron trigger after 60 days without
repository activity (`workflow_dispatch` and the push trigger still work). If the weekly run
shows as disabled for that reason, re-enable it with:

```bash
gh workflow enable regression.yml -R dekubeio/dekube-testsuite
```

## Structure

```
dekube-testsuite/
├── manifests/                    # static edge-case manifests
│   ├── deployments.yaml          # basic, multi-container, resource limits
│   ├── statefulsets.yaml         # volumeClaimTemplates, headless services
│   ├── jobs.yaml                 # Jobs, init containers, sidecars
│   ├── services.yaml             # ClusterIP, ExternalName, multi-port
│   ├── ingress.yaml              # paths, TLS, annotations
│   ├── configmaps-secrets.yaml   # volume mounts, envFrom, shared refs
│   ├── crds.yaml                 # KeycloakRealmImport, Certificate, ServiceMonitor
│   ├── edge-cases.yaml           # empty docs, 64-char names, missing ns
│   ├── bug-*.yaml                # regression fixtures, one per fixed bug
│   └── ext-<name>.yaml           # per-extension main-path fixtures (nginx, traefik, keycloak, servicemonitor, ...)
├── generate.py                   # torture test generator (writes to /tmp)
├── dekube-known-versions.json     # reference versions for comparison
├── run-tests.sh                  # main test runner
├── .github/workflows/
│   └── regression.yml            # CI (regression only)
└── README.md
```

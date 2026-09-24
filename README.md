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

# Performance test (run locally, not in CI)
./run-tests.sh --perf 5         # fast
./run-tests.sh --perf 15        # notable
./run-tests.sh --perf 30        # pain
./run-tests.sh --perf 15 --keep # keep /tmp output for inspection
```

## What it does

Every download retries automatically (`curl --retry 3 --retry-all-errors`) — a single
mid-transfer network reset no longer aborts the whole run.

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

## CI

The GitHub Actions workflow runs regression only (weekly + on push to test files). Performance tests are manual — run them on your own machine.

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
│   └── edge-cases.yaml           # empty docs, 64-char names, missing ns
├── generate.py                   # torture test generator (writes to /tmp)
├── dekube-known-versions.json     # reference versions for comparison
├── run-tests.sh                  # main test runner
├── .github/workflows/
│   └── regression.yml            # CI (regression only)
└── README.md
```

# @baseline/cli-guard

Wraps `npm install` to refuse malicious packages before they hit your machine,
backed by Baseline's aggregated malicious-package feed.

---

## How it's delivered

This package is **not on the public npm registry.** It's distributed as a
GitHub release on this repository (`IssueHunt/baseline-guard`). Each release
is a single tagged commit containing the bundled CLI as `cli.js` plus a
minimal `package.json` that maps the `baseline` binary to it.

You install it by pointing npm at the GitHub URL with a tag:

```bash
npm install -g github:IssueHunt/baseline-guard#baseline-guard-v0.1.0
```

**Always pin to a tag.** The default branch may not be a release.

A new tag = a new release. To upgrade, install the new tag the same way; npm
will replace the global binary.

---

## Local install + setup

### 1. Install globally

```bash
npm install -g github:IssueHunt/baseline-guard#baseline-guard-v0.1.0
```

After this, `baseline` is on your PATH.

### 2. Configure your API key

```bash
baseline guard configure --api-key <YOUR_API_KEY>
```

This contacts the Baseline API to confirm the key works, then writes
`~/.config/baseline/cli-guard.json`. On success, the first feed fetch also
happens here so the cache is warm for the first real install.

Failure modes:

- **HTTP 401 (rejected key)**: nothing is saved. Fix the key and retry.
- **Unreachable API (network down, 5xx, firewall)**: the config is saved with
  a warning printed to stderr, and the cache is **not** warmed. The first
  install will fetch the feed itself. Re-run `configure` once the API is
  reachable to confirm the key.

### 3. Use it

```bash
# Wrap an install
baseline guard npm install lodash axios@1.6.0

# Audit an existing project's lockfile (no install)
baseline guard audit

# Force a feed refresh (debug)
baseline guard cache refresh
```

To make the wrapper transparent in your shell, alias `npm`:

```bash
alias npm='baseline guard npm'
```

---

## Supported package managers

### Compatibility matrix

| PM / version                       | Audit (lockfile)    | Wrapper (install) | Notes                                        |
| ---------------------------------- | ------------------- | ----------------- | -------------------------------------------- |
| **npm 6**                          | ❌ v1 not supported | ❌ writes v1      | upgrade to npm 7+ (Node 16 ships npm 8)      |
| **npm 7+** (v2/v3 lockfiles)       | ✅                  | ✅                | tested on npm 10                             |
| **yarn 1** (classic)               | ✅                  | ✅                | wrapper resolves via npm internally          |
| **yarn 2/3/4** (Berry, real YAML)  | ✅ `npm:` entries   | ✅                | workspace/patch/portal/link entries skipped  |
| **pnpm 6/7** (lockfileVersion 5.x) | ✅                  | ✅                | parses `/name/version` keys                  |
| **pnpm 8** (lockfileVersion 6.0)   | ✅                  | ✅                | parses `/name@version` keys                  |
| **pnpm 9+** (lockfileVersion 9.0)  | ✅                  | ✅                | tested on pnpm 9.15; `snapshots` section     |
| **cargo** (Cargo.lock)             | ✅                  | ⚠️ top-level only | transitives covered by `audit` after install |

Wrapper mode for npm/yarn/pnpm requires `npm 7+` on PATH (we spawn `npm install --package-lock-only` to compute the plan, since all three share `registry.npmjs.org`). Cargo wrapper requires `cargo` on PATH; transitive crates aren't resolved in wrapper mode (cargo lacks a clean dry-run plan), so use `baseline guard audit` against `Cargo.lock` after install for full transitive coverage.

```bash
baseline guard audit                    # auto-detect lockfile in cwd
baseline guard audit ./yarn.lock        # explicit lockfile
baseline guard audit --pm pnpm          # explicit PM (looks for default lockfile)
```

### How the wrappers work

The wrapper mechanism differs by ecosystem because the resolution model is
different. There are two paths.

#### npm / yarn / pnpm (shared `registry.npmjs.org`)

`baseline guard npm install foo` (and yarn / pnpm equivalents) all:

1. Copy `package.json` (and `package-lock.json` if present) into a tmpdir
   and run `npm install --package-lock-only foo` there to compute the
   resulting lockfile. The user's actual `cwd` and `node_modules` are
   untouched.
2. Parse the tmp lockfile. Audit every package (direct + transitive)
   against the malicious feed; audit the freshness threshold against the
   new spec the user typed.
3. If clean, exec the real `npm install foo` / `yarn add foo` /
   `pnpm install foo`.

We use `npm` as the resolver for all three because they share
`registry.npmjs.org` and produce equivalent install plans for malicious-
package gating. Yarn and pnpm may make minor dedup choices that differ for
deep transitives, but top-level packages (the ones you typed) are always
audited correctly. Requires `npm 7+` on PATH (which Node 16+ ships
automatically).

#### cargo (`crates.io`)

`baseline guard cargo add foo` and `baseline guard cargo install foo`
work differently because cargo has no `--package-lock-only` equivalent
and uses a different registry. The wrapper:

1. Extracts the crates the user explicitly typed (`foo`, `foo@1.2.3`,
   etc.). Skips flags and non-registry specs.
2. For each typed crate, resolves a concrete version from crates.io (or
   takes the pinned version from argv).
3. Audits **only those typed crates** against the malicious feed and
   freshness threshold. Transitive dep resolution is **not** implemented
   in wrapper mode today.
4. If clean, exec the real `cargo add foo` / `cargo install foo`.

For full transitive coverage on a Cargo project, run `baseline guard
audit` in CI before `cargo build`. The audit walks every entry in
`Cargo.lock`, including transitive crates resolved by cargo itself.

---

## Checks the tool runs

Every `npm install` and `audit` runs the following independent checks. Each
has its own block / warn / override policy.

1. **Malicious-package check** (always blocks): every package in the resolved
   install plan (direct + transitive deps) is matched against the Baseline
   malicious-package feed. Any hit blocks the install. No override flag.
2. **Freshness check** (blocks by default): every **top-level** package (the
   ones you explicitly asked for) must be at least N days old. Defaults to
   **7 days**. Catches the "publish-malicious-version-and-yank-fast" pattern
   in the window before the feed catches up.
3. **Unresolvable-spec check** (blocks by default): the registry returned
   404 / timed out for a spec we tried to look up (e.g. `cargo add
nonexistent-crate`). We tried to audit and couldn't, so default is
   fail-closed. Allow specific names via `baseline-guard.json#allow.unresolved`
   or use `["*"]` for blanket permission.
4. **Non-registry-spec check** (blocks by default): the user typed `file:`,
   `github:`, `git+`, or `https://...` — the PM installs from local source /
   git / a URL, bypassing the registry by design. The malicious-package feed
   has nothing to check against. Allow specific specs via
   `baseline-guard.json#allow.nonRegistry` or use `["*"]` for blanket permission.

The freshness check is scoped to top-level packages on purpose. Applying it
across all transitives would block routine version bumps constantly
(something in any large dep tree was published recently).

### How the freshness check fails

The check has two failure modes, both blocking by default:

- **Too fresh**: package's publish date is less than the threshold. Always
  blocks; no override flag (pin to an older version yourself if you really need
  a recent one).
- **Unverifiable**: couldn't determine the publish date. Happens when the
  package is missing from `registry.npmjs.org` (unpublished, yanked, lives on
  a private registry), or when our query to the registry fails (network error,
  timeout). Blocked by default — we can't prove the package is safely old.

To allow specific unverifiable packages (e.g., your private-registry deps),
add them to `baseline-guard.json` under `allow.unverifiable`:

```json
{
  "allow": {
    "unverifiable": ["@company/*", "internal-pkg@1.2.3"]
  }
}
```

Patterns match `name` (= `name@*`) or `name@version`. Use `["*"]` for
blanket permission. Too-fresh packages still block. There's no override.

### Project config: `baseline-guard.json`

Commit a `baseline-guard.json` next to your lockfile to grant explicit
permissions for things that would otherwise block. The file takes
precedence over the user config (`~/.config/baseline/cli-guard.json`).

```json
{
  "minAgeDays": 14,
  "allowStale": false,
  "allow": {
    "nonRegistry": ["file:./shared", "github:our-org/private-pkg", "git+https://internal-git.corp/team/*.git"],
    "unresolved": ["@company/*", "internal-pkg-*"],
    "unverifiable": ["@company/*"]
  }
}
```

`allowStale` (optional) tells the tool to use the cached feed when the
Baseline API is unreachable. CLI flag `--allow-stale` overrides
the project value; same flag also persists in `~/.config/baseline/cli-guard.json`
under `allowStale` for per-machine defaults.

Three allow-list categories, matching the three checks that have overrides:

- `allow.nonRegistry`: specific non-registry specs that ARE OK to install
  (e.g. your monorepo's `file:./shared`, your team's private GitHub repo).
- `allow.unresolved`: package names the registry can't resolve but you trust
  (typically private-registry packages).
- `allow.unverifiable`: package names whose publish date can't be verified
  but you trust them. Match against `name` (= `name@*`) or `name@version`.

Allow-list values support **exact match** and **`*` glob**. Anything not
listed continues to block by default. The file is loaded from the cwd
where you run `baseline guard ...`; override with `--config <path>`.

For blanket permission within a category (e.g. you trust all unresolvable
entries because they're all from your private registry), use a single
wildcard: `"unresolved": ["*"]`. There's no CLI escape hatch. Every grant
must be a committed config change for audit-trail reasons.

### Configuring the freshness threshold

Per-invocation override:

```bash
baseline guard npm install lodash --min-age-days 14   # require 14 days
baseline guard npm install lodash --min-age-days 0    # disable for this call
```

Persisted user default (saved alongside the API key):

```bash
baseline guard configure --api-key <KEY> --min-age-days 14
```

CI / env-var path:

```bash
BASELINE_MIN_AGE_DAYS=14 baseline guard audit
```

Resolution order (highest priority first):

1. `--min-age-days <N>` flag on the command
2. `BASELINE_MIN_AGE_DAYS` env var
3. Persisted config (`baseline guard configure --min-age-days N`)
4. Default: **7**

`0` at any layer disables the freshness check entirely for that invocation.

---

## CI setup

In CI the package isn't pre-installed and there's no persistent home directory.
You install it as part of the workflow and pass the API key via env var (no `configure` needed).

### Env-var-based API key

In CI, set `BASELINE_API_KEY` and skip the config-file flow entirely. The CLI checks the env var first and only falls back to the config file if it's not set.

### GitHub Actions

Two equivalent ways to wire it into a GitHub workflow. Pick whichever fits
your existing setup.

Add `BASELINE_API_KEY` as a repo secret in **Settings → Secrets and
variables → Actions** for either option.

#### Option A: official composite action (less boilerplate)

```yaml
name: CI
on: [push, pull_request]

jobs:
  audit-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Audit lockfile against malicious-package feed
        uses: IssueHunt/baseline-guard@baseline-guard-v0.1.0
        with:
          api-key: ${{ secrets.BASELINE_API_KEY }}

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

The action auto-detects the lockfile (`package-lock.json` / `yarn.lock` /
`pnpm-lock.yaml` / `Cargo.lock`) and runs `baseline guard audit --markdown`.
On block it: (1) fails the workflow step (the red ✕ on the PR is the
notification), and (2) appends a Markdown summary to the workflow's
**Job Summary** with the malicious packages, freshness issues, and any
unresolved/non-registry findings — visible on the PR's Checks tab.

Action inputs:

| Input               | Default                 | Description                                                                               |
| ------------------- | ----------------------- | ----------------------------------------------------------------------------------------- |
| `api-key`           | (required)              | Baseline API key                                                                          |
| `lockfile`          | auto-detect             | Explicit path to a lockfile                                                               |
| `pm`                | auto-detect             | `npm` / `yarn` / `pnpm` / `cargo`                                                         |
| `min-age-days`      | (config)                | Override freshness threshold                                                              |
| `allow-stale`       | `false`                 | Use cached feed when network is down                                                      |
| `config`            | `./baseline-guard.json` | Path to project config (allow-lists). Defaults to `./baseline-guard.json` in working-dir. |
| `working-directory` | `.`                     | Where to run audit                                                                        |

For a JSON report (e.g. to upload to a security dashboard or feed into a SARIF
converter), invoke the binary directly in a separate step:
`baseline guard audit --json > report.json`.

#### Option B: install + invoke directly

Use this if you want full control over the invocation (custom flags,
multiple lockfiles, sharing a Node setup with other steps, etc.):

```yaml
name: CI
on: [push, pull_request]

jobs:
  audit-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install baseline-guard
        run: npm install -g github:IssueHunt/baseline-guard#baseline-guard-v0.1.0

      - name: Audit lockfile against malicious-package feed
        run: baseline guard audit --markdown
        env:
          BASELINE_API_KEY: ${{ secrets.BASELINE_API_KEY }}

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test
```

### Why a separate audit step before `npm ci`

`baseline guard audit` reads the lockfile and checks every pinned version
against the feed before npm starts downloading anything. If a malicious package
is pinned, audit exits non-zero and `npm ci` never runs. This is the
recommended pattern in CI because it's more explicit than the `preinstall`
hook (which can't see the install args, see "Coverage limits" below).

### GitLab CI example

```yaml
stages: [audit, test]

audit:
  stage: audit
  image: node:22
  variables:
    BASELINE_API_KEY: $BASELINE_API_KEY
  script:
    - npm install -g github:IssueHunt/baseline-guard#baseline-guard-v0.1.0
    - baseline guard audit

test:
  stage: test
  image: node:22
  script:
    - npm ci
    - npm test
```

### Caching the feed between CI runs (optional)

The feed is ~4 MB gzipped, so it downloads quickly even fresh. But if you want
to skip the network call when nothing's changed:

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.cache/baseline/cli-guard
    key: baseline-feed-${{ runner.os }}-${{ github.run_id }}
    restore-keys: |
      baseline-feed-${{ runner.os }}-
```

The CLI uses ETag-conditional requests, so a stale-but-cached feed gets a free
`304 Not Modified` from the Baseline API.

---

## How to test it works (smoke test)

After installing and configuring, verify the wrapper actually blocks
something. We don't ship a known-bad fixture (changes too often), but you can
check against any current malicious package the feed has:

```bash
# This should EXIT WITH STATUS 1 and print one or more matches
baseline guard audit  # in a project whose lockfile contains a known-bad version
```

Or for a sanity check the wrapper is wired correctly, run it from a scratch
directory so a real `npm install` is harmless:

```bash
mkdir /tmp/baseline-smoke && cd /tmp/baseline-smoke && npm init -y
baseline guard npm install lodash
# Expected:
#   [baseline] computing install plan...
#   [baseline] plan: 1 package
#   [baseline] feed source: network-200 (N advisories)   # or cache-fresh
#   [baseline] clean. Running real npm
#   added 1 package ...
```

If you see `[baseline] feed source: ...` and a non-error completion, the
cache + wrapper paths are working.

---

## Coverage limits

What the wrapper covers:

- `baseline guard npm install <pkg>` (full plan check including transitives)
- `baseline guard npm install -g <pkg>`
- `baseline guard npm ci` (lockfile check before download)
- `baseline guard npm install` with no args (same as above)

What `baseline guard audit` (e.g. as preinstall hook) catches:

- `npm install` / `npm ci` (full lockfile audit before deps download)
- Adding to your own preinstall: catches CI runs and fresh checkouts

What audit / preinstall hooks **don't** catch:

- `npm install <new-pkg>`: The wrapper is the only way to gate this case.
- `npm install -g <pkg>`: Use the wrapper instead (`baseline guard npm install -g <pkg>`).
- Anyone running `npm install --ignore-scripts`: bypasses preinstall.

For full coverage, dev workstations should alias `npm` to
`baseline guard npm`. CI should run `baseline guard audit` as an explicit step
before `npm ci`.

---

## Failure modes

| Situation                 | Behavior                                                   |
| ------------------------- | ---------------------------------------------------------- |
| Network down, cache fresh | Use cache silently                                         |
| Network down, cache stale | Exit non-zero (fail-closed); override with `--allow-stale` |
| Network down, no cache    | Exit non-zero                                              |
| Baseline API 401          | Asks you to re-run `baseline guard configure`              |
| Baseline API 5xx          | Falls back to cache if any; otherwise exit non-zero        |

---

## Cache

The feed is cached at `~/.cache/baseline/cli-guard/latest.json.gz` with a
1-hour freshness window. Refreshes use HTTP `If-None-Match` so unchanged feeds
cost zero bytes.

```bash
baseline guard cache refresh    # force a refresh
baseline guard cache clear      # nuke the cache
```

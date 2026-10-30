---
name: dependency-review
description: Use on any project to find dependencies that can be dropped or replaced with a few lines of code, making the project lighter and reducing supply-chain and vulnerability exposure. Reviews direct runtime and dev dependencies (unused, low-value tooling, trivially reimplementable, superseded by platform built-ins) and produces a report with a human summary and Claude-ready removal steps.
---

# Dependency review

Produce a read-only report of direct dependencies the project could remove or replace with a small amount of its own code.

## Inputs

- **Scope (optional):** `runtime`, `dev`, or `all` (default `all`).
- **Path (optional):** project root or a workspace package; default the current working directory. In a monorepo, review each workspace package that has its own manifest.

## Step 1 — Inventory

1. Detect the ecosystem from the manifest (`package.json`, `pyproject.toml`/`requirements*.txt`, `go.mod`, `Cargo.toml`, …). The steps below use npm terms; apply the equivalent tooling for other ecosystems.
2. List direct dependencies per group: `dependencies`, `devDependencies`, `peerDependencies`, `optionalDependencies`.
3. Determine the platform baseline that limits which built-ins can be used: `engines.node`, `.nvmrc`/`.node-version`, CI Node matrix, TypeScript `target`/`lib`, and browser targets (`browserslist`, a `browser` field, or bundler config) if the package ships to browsers.
4. For each dependency, record:
   - **Footprint:** number of transitive packages it pulls in (`npm ls <dep> --all --parseable | wc -l`, or count from the lockfile) and whether it has install scripts.
   - **Usage:** every import/require/CLI invocation across source, tests, scripts, config files, and `package.json` scripts. Note which exported functions are actually used.
   - **Health:** `npm audit --json` hits (including ones in its transitive packages), last publish date (`npm view <dep> time.modified`), deprecation notice (`npm view <dep> deprecated`), number of maintainers (`npm view <dep> maintainers`).
   - **Exposure:** whether it handles untrusted input (network data, user files, parsing) or runs at install time.

## Step 2 — Classify

Put each dependency in exactly one bucket.

### Remove — unused
No import, require, CLI use, config reference or plugin reference anywhere. Check indirect usage before concluding: config files (`.eslintrc*`, `babel.config.*`, `jest.config.*`, `tsconfig` `types`, bundler plugins), `@types/*` paired with a used package, peer deps required by another dependency, and dynamic imports by string.

### Remove — low value
Used, but the project would lose little without it. Typical cases:
- Ops and convenience tooling that wraps something trivial: `rimraf`, `mkdirp`, `cross-env`, `npm-run-all`, `copyfiles`, `del-cli`, `shx`, `dotenv-cli` — when a shell builtin, `node -e`, or npm's own features cover it.
- Duplicate tools doing the same job (two test runners, two formatters, two HTTP clients).
- Git hook managers, release helpers, or badges that nothing in CI or the team workflow relies on — mark these `needs team decision` rather than asserting removal.

### Replace — built-in available
The platform baseline already provides it. Examples (only if the baseline supports them):
- `node-fetch`, `cross-fetch`, `axios` used for simple requests → global `fetch` (Node ≥ 18)
- `uuid` (v4 only) → `crypto.randomUUID()`
- `lodash.clonedeep` → `structuredClone`
- `rimraf` / `mkdirp` → `fs.rm({ recursive: true, force: true })` / `fs.mkdir({ recursive: true })`
- `minimist` / `yargs` for trivial CLIs → `util.parseArgs`
- `dotenv` → `node --env-file`
- `abort-controller`, `event-target-shim`, `buffer`, `url`, `util` polyfills → native globals
- Single-function `lodash.*` / `ramda` usages → native `Array`/`Object` methods

### Replace — few lines of code
Only a small part is used, and an equivalent is roughly ≤ 30 lines without tricky edge cases (e.g. `is-*` predicates, `left-pad`-style helpers, small `debounce`, `p-limit`-style concurrency, `ms` for a fixed set of units).

### Keep
Everything else. Always keep, without proposing a rewrite: cryptography, parsers for complex formats, protocol implementations, schema validators, date/time-zone logic, anything where correctness or security depends on edge cases the library already handles. Also keep core framework and build tooling (TypeScript, bundlers, test runner, linter) unless duplicated.

### Minimize false positives
Before listing anything outside **Keep**:
1. Read every usage site, not just the import line.
2. Check the dependency's documented behaviour for the functions used (encoding, error handling, retries, platform quirks) and confirm the replacement covers what the project actually relies on.
3. Confirm the replacement works on the whole platform baseline, including browsers if applicable.

If any of these can't be settled, keep the dependency out of the main list and put it under **Possibly removable** with what is still unclear. Keep that section short — when in doubt about value, leave it in **Keep**.

### Risk rating

Rate every listed dependency (main list and **Possibly removable**) on how risky it is to keep it and therefore how strongly removal is recommended. Add one point per factor that applies, then map the total:

| Factor | +1 when |
|---|---|
| Ships to users | It's in `dependencies` (runtime), not dev-only |
| Known vulnerabilities | It or its transitive packages have open `npm audit` findings (+2 if any is high/critical) |
| Install-time code | It or its transitive packages have install scripts |
| Large footprint | Removing it eliminates ≥ 20 transitive packages |
| Poorly maintained | Deprecated, no publish in ≥ 2 years, or a single maintainer |
| Untrusted input | It parses network data, user files, or other external input |

| Total | Rating | Recommendation |
|---|---|---|
| 0–1 | 🟢 Low | Optional cleanup |
| 2–3 | 🟡 Medium | Recommended |
| 4+ | 🔴 High | Strongly recommended — prioritise |

Show the rating with the factors that contributed, e.g. `🔴 High (runtime, high vuln, 24 pkgs)`. Sort the summary table and the implementation details by rating, highest first.

## Step 3 — Write the report

```markdown
# Dependency review: <project name>

Path: <path> · Ecosystem: <npm/pip/…> · Baseline: <Node ≥ 18, ES2022, browsers: …> · Reviewed: <n> direct deps (<runtime> runtime, <dev> dev)

## Summary

| Dependency | Group | Verdict | Risk to keep | Transitive pkgs | Why |
|---|---|---|---|---|---|
| node-fetch | runtime | Replace — built-in | 🟡 Medium (runtime, vuln) | 5 | Node ≥ 18 has global fetch; 3 call sites, no streaming |
| rimraf | dev | Remove — low value | 🟢 Low | 12 | Used once in `clean` script; `rm -rf` / `fs.rm` covers it |

Totals: <n> removable, <n> replaceable, ~<n> transitive packages and <n> audit findings eliminated.

### To do
- Short ordered list, runtime dependencies first (they ship to users).

## Implementation details

#### 1. <dependency>: <verdict>
- **Risk to keep:** <rating> — contributing factors
- **Used at:** `path/to/file.ts:line`, … (or "nowhere")
- **Change:** exact edit — code to remove, replacement snippet (full code if ≤ 30 lines), `package.json` script rewrite
- **Behaviour to preserve:** edge cases the replacement must handle, or "none"
- **Remove:** `npm uninstall <dep>` (plus related `@types/<dep>`)
- **Verify:** tests/commands to run afterwards

## Possibly removable

#### <dependency>
- **Risk to keep:** <rating> — contributing factors
- **Why it might go:** …
- **Unclear:** what couldn't be confirmed
- **To resolve:** concrete check or question for the team
```

Omit `Possibly removable` if empty. Don't list **Keep** dependencies individually; just give their count in the header.

## Guardrails

- Read-only: do not uninstall, edit, or commit anything.
- Every verdict must be backed by the usage sites found; cite them with `path:line`.
- Don't propose replacing a dependency with more code than it's worth — if the replacement exceeds ~30 lines or needs its own tests to be trustworthy, keep the dependency.

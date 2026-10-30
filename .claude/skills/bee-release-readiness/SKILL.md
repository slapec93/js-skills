---
name: bee-release-readiness
description: Use before a Bee release to find required changes in the JS Team projects (bee-js, swarm-cli, swarm-gateway, bee-factory). Compares Bee API specs, PRs, config flags and contracts between two Bee versions and produces a report with a human summary on top and Claude-ready implementation details below. Optional input is a Bee version range; defaults to latest stable tag..master.
---

# Bee release readiness

Produce a read-only report of what must change in bee-js, swarm-cli, swarm-gateway and bee-factory for an upcoming Bee release.

## Inputs

- **Range (optional):** `<from>..<to>`, e.g. `v2.5.0..v2.6.0-rc1`.
  - Default `<from>`: latest stable Bee release tag — `gh release view --repo ethersphere/bee --json tagName -q .tagName` (excludes pre-releases).
  - Default `<to>`: `master` of `ethersphere/bee`.
- **Repo locations (optional):** local checkout paths for the JS projects. If not given, use `ethersphere/bee-js`, `ethersphere/swarm-cli`, `ethersphere/swarm-gateway`, `ethersphere/bee-factory` on GitHub's default branch.

State the resolved range and repo sources at the top of the report.

## Step 1 — Get the sources

Clone into the scratchpad (never into the user's working tree):

```bash
git clone --filter=blob:none https://github.com/ethersphere/bee.git <scratch>/bee
git clone --depth 1 https://github.com/ethersphere/<project>.git <scratch>/<project>   # only if no local path given
```

## Step 2 — Collect what changed in Bee (`<from>..<to>`)

1. **API spec** — `git diff <from>..<to> -- openapi/`. Primary signal. Record every added/removed/renamed endpoint, parameter, header, request/response field, type, and status code.
2. **PRs** — `git log --oneline <from>..<to>` and the merged PRs in that range (`gh pr list --repo ethersphere/bee --state merged --search "merged:>=<from date>"`). Pay extra attention to anything touching `pkg/api`, or labelled breaking/api/config.
3. **Config & CLI flags** — `git diff <from>..<to> -- cmd/bee/`. Added/removed/renamed flags, changed defaults, env vars.
4. **Chain & contracts** — changes to contract versions/addresses, chain IDs, storage-incentives, postage, or swap setup (`go.mod` bumps of `go-storage-incentives-abi`, `pkg/config`, etc.).
5. **API version** — any change to the reported API version (`openapi/Swarm.yaml` `info.version`).

Ignore purely internal changes (refactors, p2p/protocol internals, tests) unless they change observable HTTP or config behaviour.

## Step 3 — Map changes to projects

For every Bee change, grep the affected project before declaring a required change. Only list a change if the project actually uses the affected endpoint/field/flag, or must add support for a new one.

- **bee-js** — endpoint wrappers, request/response types, header handling, supported Bee/API version constants, tests and test fixtures.
- **swarm-cli** — commands that call affected bee-js methods or Bee endpoints directly; bee-js dependency bump; command options mirroring changed Bee params.
- **swarm-gateway** — endpoints it proxies or calls, header passthrough/allowlists, bee-js dependency bump.
- **bee-factory** — Bee image tag/version, Bee flags and env vars used in its configs, contract deployment and blockchain image, chain ID.

### Uncertain findings

Some changes may affect a project without it being clear from the diff (e.g. behaviour changes behind an unchanged endpoint, changed defaults, error semantics). Keep these to a minimum — before listing one, try to resolve it:

1. Read the PR description, linked issue, and the actual Go code change, not just the title.
2. Grep the project for every usage of the affected endpoint/field/flag and read the calling code.
3. Check whether tests or fixtures in the project encode the old behaviour.

If that settles it, classify it as a required change or drop it. Only if it remains genuinely unresolved, list it under **Possibly affected** in the report. Never list something merely because it touches a related area.

## Step 4 — Write the report

Use exactly this structure:

```markdown
# Bee release readiness: <from> → <to>

Sources: bee <from>..<to> (<commit sha>), bee-js <ref>, swarm-cli <ref>, swarm-gateway <ref>, bee-factory <ref>

## Summary

| Project | Impact | Reason |
|---|---|---|
| bee-js | none / minor / breaking | one line |
| swarm-cli | ... | ... |
| swarm-gateway | ... | ... |
| bee-factory | ... | ... |

### To do
- Short, human-readable list of the work, in the order it should be done (bee-js first, then dependants).

### Possibly affected
- One line per uncertain finding: project, Bee change, why it's unclear. Omit this subsection if there are none.

## Implementation details

### bee-js
#### 1. <short title>
- **Bee change:** what changed, with link to PR or spec diff
- **Files:** `path/to/file.ts:line`
- **Change:** precise description of the edit
- **Tests:** tests to add or update

### swarm-cli
...

### swarm-gateway
...

### bee-factory
...

## Possibly affected

#### <project>: <short title>
- **Bee change:** what changed, with link to PR or spec diff
- **Possible impact:** what might break or behave differently, and where (`path/to/file.ts:line`)
- **Checked:** what was already investigated and why it didn't settle the question
- **To resolve:** the concrete check that would settle it (e.g. run X against a node on `<to>`, ask in PR #N)
```

Implementation details contain only confirmed changes. Omit the `Possibly affected` section if there are none.

If a project needs nothing, write `No changes needed.` under its heading — never omit the section.

## Guardrails

- Read-only: do not edit, branch, commit, or open PRs in any repo.
- Every item must cite its source (PR link or spec diff hunk).
- Anything not confirmed in the project's code goes under `Possibly affected`, never under implementation details.

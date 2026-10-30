---
name: bee-post-release-check
description: Use after a Bee release to verify the JS Team projects are up to date with it — swarm-cli quickstart and CI, bee-js supported Bee/API versions, bee-factory Docker images on Docker Hub, and the Bee OpenAPI version. Optional input is the Bee release tag; defaults to the latest stable release.
---

# Bee post-release check

Produce a read-only checklist showing whether everything that should follow a Bee release has been updated.

## Inputs

- **Release tag (optional):** e.g. `v2.9.0`. Default: latest stable release — `gh release view --repo ethersphere/bee --json tagName,publishedAt`.
- **Previous release:** the stable release before it — `gh release list --repo ethersphere/bee --exclude-pre-releases --limit 2`.
- **Repo locations (optional):** local checkout paths. If not given, use `ethersphere/bee-js`, `ethersphere/swarm-cli`, `ethersphere/bee-factory` on GitHub's default branch, plus open PRs.

## Step 1 — Get the sources

Clone into the scratchpad (never into the user's working tree):

```bash
git clone --filter=blob:none https://github.com/ethersphere/bee.git <scratch>/bee
git clone --depth 1 https://github.com/ethersphere/<project>.git <scratch>/<project>   # only if no local path given
```

Record the expected values from the release tag:
- **Bee version:** the tag without `v` (e.g. `2.9.0`).
- **API version:** `info.version` in `openapi/Swarm.yaml` at the tag.

## Step 2 — Checks

Each check yields ✅ up to date, ❌ outdated, or ⚠️ could not determine (say why).

### OpenAPI version
1. `info.version` of `openapi/Swarm.yaml` differs between previous and current tag whenever `git diff <prev>..<tag> -- openapi/` is non-empty. ❌ if the spec changed but the version didn't.
2. Major/minor/patch bump matches the change: removed/renamed endpoints, fields or required params → major; additions → minor; doc-only fixes → patch. Flag a mismatch as ❌ with the offending hunks.

### bee-js
- `package.json` `engines.bee` equals the Bee version.
- `package.json` `engines.beeApiVersion` equals the API version.
- `src/version.ts` matches `package.json` (it is generated; ❌ if stale).
- Every `bee-factory start --tag` in `.github/workflows/*` uses the release tag.
- Latest published version (`npm view @ethersphere/bee-js version`) contains these values — check its `engines` with `npm view @ethersphere/bee-js engines`.

### swarm-cli
- `src/command/quickstart.ts` `beeVersion` equals the release tag.
- Every `bee-factory start --tag` in `.github/workflows/*` uses the release tag.
- `@ethersphere/bee-js` in `package.json` allows the latest bee-js release that supports this Bee version.
- Latest published version (`npm view @ethersphere/swarm-cli version`) includes the quickstart bump — check the released source at its tag.

### bee-factory Docker images
For each of `bee-factory-queen`, `bee-factory-worker-1` … `bee-factory-worker-4`, and `bee-factory-blockchain` (bee-factory pulls all six together, see `src/docker/manager.ts` `tryPullPrebuiltImages`):

```bash
curl -s "https://hub.docker.com/v2/repositories/ethersphere/<image>/tags/<tag>"
curl -s "https://hub.docker.com/v2/repositories/ethersphere/<image>/tags/latest"
```

- The release tag exists, and `<tag>-amd64` / `<tag>-arm64` exist.
- `latest` was updated at or after the release tag's push (`last_updated`), i.e. it isn't still pointing at the previous release.
- If an image is missing, check the latest bee-factory publish workflow run (`gh run list --repo ethersphere/bee-factory`) and report its status and link.

### Open PRs
If any check is ❌, look for an open PR that fixes it (`gh pr list --repo ethersphere/<project> --search "<version>"`) and link it next to the item.

## Step 3 — Write the report

```markdown
# Bee post-release check: <tag>

Release: <tag> (published <date>), previous: <prev>, API version <api>
Sources: bee-js <ref>, swarm-cli <ref>, bee-factory <ref>

## Summary

| Area | Status | Notes |
|---|---|---|
| OpenAPI version | ✅/❌/⚠️ | one line |
| bee-js | ... | ... |
| swarm-cli | ... | ... |
| bee-factory images | ... | ... |

### To do
- Short list of what's still outdated, with links to open PRs where they exist.

## Details

### OpenAPI version
- ✅/❌ <check> — expected `<x>`, found `<y>` (`path:line` or URL)

### bee-js
...

### swarm-cli
...

### bee-factory images
| Image | `<tag>` | `<tag>-amd64` | `<tag>-arm64` | `latest` updated |
|---|---|---|---|---|
| bee-factory-queen | ✅ | ✅ | ✅ | <date> |
...
```

## Guardrails

- Read-only: do not edit, branch, commit, open PRs, or trigger workflows.
- Every ❌ states expected vs. found and where it was found.

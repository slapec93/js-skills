# js-skills

Claude Code skills for the Swarm JS Team (bee-js, swarm-cli, swarm-gateway, bee-factory).

## Skills

| Skill | When | What it does |
|---|---|---|
| `bee-release-readiness` | Before a Bee release | Diffs Bee (OpenAPI spec, PRs, CLI flags, contracts) and reports what must change in the JS projects. Default range: latest stable tag..master. |
| `bee-post-release-check` | After a Bee release | Verifies the JS projects follow the release: bee-js engines and CI, swarm-cli quickstart and CI, bee-factory Docker Hub images, OpenAPI version bump. Default: latest stable release. |
| `dependency-review` | Any project, any time | Finds dependencies that are unused, low-value (e.g. ops tooling), superseded by built-ins, or replaceable with a few lines of code. |

All are read-only and produce a Markdown report with a human summary on top and details below.

## Usage

Open Claude Code in this repo and run:

```
/bee-release-readiness [<from>..<to>]
/bee-post-release-check [<tag>]
/dependency-review [runtime|dev|all] [<path>]
```

To use the skills in any project, copy or symlink `.claude/skills/*` into `~/.claude/skills/`.

## Requirements

`git`, `gh` (authenticated), `npm`, `curl`.

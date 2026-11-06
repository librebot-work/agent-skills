# Librebot source tooling

## Toolchain

Use Node.js 24 or newer and Git. The CLI is pinned to tools commit `5bf5ffdde6f50d745b105c66732083e4d33a520d` (package version `0.3.0`). Set `TOOLS` to the immutable archive URL shown in [SKILL.md](../SKILL.md). The platform vendors the same release as `librebot-tools-0.3.0.tgz`; an explicitly supplied matching local tarball may be used offline. Do not substitute `main`, `latest`, or an older pin.

Use `npx --yes --package="$TOOLS" librebot ...` with the commands below. Keep the `TOOLS` variable in the same shell session as the command, or set it again when a harness uses isolated shell calls. `--yes` permits npm's package-fetch prompt; it is not permission to bypass a harness's network or execution approvals.

| Command | Result |
| --- | --- |
| `validate <dir> --json` | Validates source shape, semantics, cross-file references, limits, and locks. Returns `ok` and diagnostics; invalid source exits nonzero. |
| `preview <dir> --json` | Validates and prints the resolved definition, asset hashes/sizes, skill files, warnings, and lock metadata. Does not run the bot. |
| `scaffold <dir> --name 'Name' --description 'Description'` | Writes a new skeleton with example content; use only an empty destination. Does not provision a platform bot. |
| `resolve-skills <dir> --network --vendor` | Installs selected external skill content locally and produces a lock plus vendor tree. Existing locks are reused where valid. |
| `resolve-skills <dir> --update --network --vendor` | Intentionally updates existing skill dependency resolutions. |

Validation errors are records with `file`, `field`, and `message`; warnings have the same diagnostic fields. A successful `validate --json` prints `ok: true`. Filesystem failures or package-fetch failures can print ordinary stderr instead of a JSON validation result; do not misreport those as bot schema errors. `preview --json` is structured on success, but invalid source is reported as an error, not a resolved preview.

The source parser and validator do not execute bot prompts, shell commands from the bot, or its MCP servers. `resolve-skills` is a separate, explicitly network-enabled local operation. The platform consumes the committed lock and vendor files offline.

## Exact schema references

These are the schemas shipped with the pinned tool, not a generic agent or MCP schema:

- The supplied tarball contains `schema/v1/bot-yaml-schema.json`.
- It also contains `schema/v1/skills-sources-schema.json`.
- The bundled `README.md` documents the matching source tooling.

An editor schema is useful for completion, but is not a substitute for `validate`. Semantic validation checks relationships and constraints not fully expressed by JSON Schema.

The matching source format uses independent keyed `workspace`, `actions`, and
`triggers` sections. Workspace is flat; root `resources`, provider-specific action
arrays, nested checkout declarations, and messaging declarations are rejected.
See [source-format.md](source-format.md) for the accepted declarations.
Runtime native repository calls use `repo_*` names and a required `action`
selector for a declared action group, not a workspace or freeform repository.
The common change-request sections are `details`, `files`, `feedback`, `commits`,
`comments`, `discussions`, `approvals`, and `checks`. Feedback preserves the
provider model: GitHub `review_submissions`, GitLab `approval_state`.
GitHub derived approval state does not resolve branch-rule requirements.
GitHub `read_change_request` requires Checks read, Issues read, and Pull requests
read; declaration validation does not prove the installation grants them.

## Publication boundary

Git is the transport; these tools have no login, push, deployment, bot execution, or instance-upgrade command. Use the developer's Git credentials and authorized system UI/API separately. A successful local validation, push, or tag creation does not prove that the platform accepted the revision or upgraded an instance.

# Librebot source tooling

## Toolchain

Use Node.js 24 or newer and Git. All bot authoring, including scaffold, skill resolution, validation, and preview, uses the platform-matched `librebot-tools` CLI published on npm:

```sh
TOOLS='librebot-tools@latest'
npx --yes --package="$TOOLS" librebot validate . --json
npx --yes --package="$TOOLS" librebot preview . --json
```

Custom model roles require `librebot-tools@0.4.6` or newer. The older `0.4.5` release accepts the four built-in roles only. Do not substitute a local archive, a Git checkout, or an unrelated package. Report fetch failures.

Use `npx --yes --package="$TOOLS" librebot ...` with the commands below. Keep the `TOOLS` variable in the same shell session as the command, or set it again when a harness uses isolated shell calls. `--yes` permits npm's package-fetch prompt; it is not permission to bypass a harness's network or execution approvals. Report the tools version alongside validation results when it matters for a diagnosis.

| Command | Result |
| --- | --- |
| `validate <dir> --json` | Validates source shape, semantics, cross-file references, limits, and locks. Returns `ok` and diagnostics; invalid source exits nonzero. |
| `preview <dir> --json` | Validates and prints the resolved definition, asset hashes/sizes, skill files, warnings, and lock metadata. Does not run the bot. |
| `scaffold <dir> --name 'Name' --description 'Description'` | Writes a new skeleton with example content and a user-manual `README.md` whose placeholders must be replaced; use only an empty destination. Does not provision a platform bot. |
| `resolve-skills <dir> --network --vendor` | Installs selected external skill content locally and produces a lock plus vendor tree. Existing locks are reused where valid. |
| `resolve-skills <dir> --update --network --vendor` | Intentionally updates existing skill dependency resolutions. |

Validation errors are records with `file`, `field`, and `message`; warnings have the same diagnostic fields. A successful `validate --json` prints `ok: true`. Filesystem failures or package-fetch failures can print ordinary stderr instead of a JSON validation result; do not misreport those as bot schema errors. `preview --json` is structured on success, but invalid source is reported as an error, not a resolved preview.

The source parser and validator do not execute bot prompts, shell commands from the bot, or its MCP servers. `resolve-skills` is a separate, explicitly network-enabled local operation. The platform consumes the committed lock and vendor files offline.

## Exact schema references

These are the schemas shipped with the npm package, not a generic agent or MCP schema:

- The package contains `schema/v1/bot-yaml-schema.json`.
- It also contains `schema/v1/skills-sources-schema.json`.
- Its `README.md` documents the matching source tooling.

An editor schema is useful for completion, but is not a substitute for `validate`. Semantic validation checks relationships and constraints not fully expressed by JSON Schema.

The matching source format uses independent keyed `workspace`, `actions`, and
`triggers` sections. Workspace is flat; root `resources`, provider-specific action
arrays, nested checkout declarations, and obsolete `kind: messaging` are rejected.
Actions and triggers support `kind: gitrepo` and `kind: whatsapp`, with separate
capability and event namespaces. Owner bindings and consent grant independent
runtime authority; portable declarations do not.
See [source-format.md](source-format.md) for the accepted declarations.
Runtime native repository calls use `repo_*` names and a required `action`
selector for a declared action group, not a workspace or freeform repository.
The common change-request sections are `details`, `files`, `feedback`, `commits`,
`comments`, `discussions`, `approvals`, and `checks`. Feedback preserves the
provider model: GitHub `review_submissions`, GitLab `approval_state`.
GitHub derived approval state does not resolve branch-rule requirements.
GitHub `read_change_request` requires Checks read, Issues read, and Pull requests
read; declaration validation does not prove the installation grants them.
Native GitHub merging additionally requires Contents write, Pull requests write,
Checks read, Issues read, and Administration read on the App installation plus
the linked owner's live push, maintain, or admin authority. Administration read
is for policy visibility, not administrator bypass. Validation/preview cannot
prove live CI, review, conversation, or branch-policy eligibility.

## Diagnosing trigger-decider access

`librebot preview` is a source-definition preview, not an instance's live
trigger preview. Neither source validation nor choosing a tool-capable
`deciderRole` proves that a decision can read current provider state.

For an ignore/failure reporting an unavailable tool, compare the required read
with the [decider tool contract](source-format.md#startcontinue-decider-tools):
check the declared capability, the independent instance action/connector
binding, its current connection and target, and live provider authority.
For MCP reads, also check the connected server's explicit
`annotations.readOnlyHint: true`; absent/false metadata excludes the tool even
if it works in a regular task. Do not repair this by inventing source fields,
granting shell access, or routing an ineligible task as a workaround.

An explicitly authorized instance trigger preview or real delivery can
exercise decision reads. Trigger preview uses the same read-only environment
as live decisions, but does not commit scheduling or external writes.
Distinguish a missing read capability, a revoked/unbound connection, and an
eligible read that genuinely found no actionable work. Report only checks
actually exercised; do not claim deployment or instance upgrades from source
validation.


## Publication boundary

Git is the transport; these tools have no login, push, deployment, bot execution, or instance-upgrade command. Use the developer's Git credentials and authorized system UI/API separately. A successful local validation, push, or tag creation does not prove that the platform accepted the revision or upgraded an instance.

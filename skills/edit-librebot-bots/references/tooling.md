# Librebot source tooling

## Toolchain

Use Node.js 24 or newer and Git. All bot authoring requires the explicitly supplied local `librebot-tools-0.4.1.tgz`, including scaffold, skill resolution, validation, and preview for bots without messaging. This local package includes required root README validation and portable WhatsApp declarations; the historical `46c5e139d1542f42798606d357a7621eda938024` archive (0.4.0) does not implement this contract. No published immutable artifact is available for this contract. Set `TOOLS` to the supplied tarball's absolute path as shown in [SKILL.md](../SKILL.md). If it is missing, request it; do not substitute a public URL, `main`, `latest`, or an older package.

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

These are the schemas shipped with the required local tool, not a generic agent or MCP schema:

- The supplied tarball contains `schema/v1/bot-yaml-schema.json`.
- It also contains `schema/v1/skills-sources-schema.json`.
- The bundled `README.md` documents the matching source tooling.

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

## Publication boundary

Git is the transport; these tools have no login, push, deployment, bot execution, or instance-upgrade command. Use the developer's Git credentials and authorized system UI/API separately. A successful local validation, push, or tag creation does not prove that the platform accepted the revision or upgraded an instance.

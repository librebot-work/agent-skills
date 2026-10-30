# Librebot source tooling

## Toolchain

Use Node.js 24 or newer and Git. The `TOOLS` assignment in [SKILL.md](../SKILL.md) pins the CLI to a commit of the public [librebot-work/tools](https://github.com/librebot-work/tools) repository, whose package version is `0.2.1`. Fetching this package does not require `@librebot/tools` to be published on npm. Never silently replace the pin with `main` or `latest`.

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

- [bot.yaml schema](https://github.com/librebot-work/tools/blob/362de24b8a2732dcdee9f0f3510455a92f056c64/schema/v1/bot-yaml-schema.json)
- [skills/sources.yaml schema](https://github.com/librebot-work/tools/blob/362de24b8a2732dcdee9f0f3510455a92f056c64/schema/v1/skills-sources-schema.json)
- [Source tooling documentation](https://github.com/librebot-work/tools/blob/362de24b8a2732dcdee9f0f3510455a92f056c64/README.md)

An editor schema is useful for completion, but is not a substitute for `validate`. Semantic validation checks relationships and constraints not fully expressed by JSON Schema.

## Publication boundary

Git is the transport; these tools have no login, push, deployment, bot execution, or instance-upgrade command. Use the developer's Git credentials and authorized system UI/API separately. A successful local validation, push, or tag creation does not prove that the platform accepted the revision or upgraded an instance.

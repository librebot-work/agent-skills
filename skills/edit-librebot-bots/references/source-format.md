# Bot source format v1

Use the pinned Librebot parser and semantic validator as the source of truth. Read the existing definition before applying these examples. Do not rewrite the whole bot to match a minimal example.

## Repository layout

```text
bot.yaml
instructions.md
knowledge/
  README.md                 # index only, not ingested
  product-guide.md          # text knowledge
  diagram.png               # binary asset; other formats are also accepted
skills/
  review/
    SKILL.md                # local skill entrypoint
    references/checklist.md # supporting content travels with the skill
  sources.yaml              # optional external skill declarations
  sources.lock              # generated immutable resolution
  .vendor/                  # generated external content; commit it
```

The source repository describes the shared bot. Account credentials, repository bindings for workspace mounts, concrete instance model choices, handle, visibility, and adopted revision pins are managed outside these files.

## `bot.yaml`

The only top-level keys are:

| Key | Meaning |
| --- | --- |
| `formatVersion` | Required; integer `1` |
| `name` | Required display name, not the bot's URL handle |
| `description` | Required non-empty description |
| `modelRoles` | Required unique roles; 1–4 entries including `default` |
| `workspace` | Optional mount requirements using canonical relative `path` |
| `triggers` | Optional GitHub trigger definitions |
| `githubActions` | Optional native GitHub capability names |
| `connectors` | Optional remote MCP capability/resource requirements |
| `knowledgeSources` | Optional structured external knowledge references |

Minimal definition:

```yaml
formatVersion: 1
name: Issue assistant
description: Helps triage GitHub issues.
modelRoles:
  - role: default
    label: Main
workspace: []
triggers: []
githubActions: []
connectors: []
```

A non-empty `instructions.md` supplies the main prompt; do not introduce `behavior`, `prompt`, or `botPrompt` keys in YAML. Describe the requested behavior, boundaries, and output expectations there.

### Model roles

Role IDs: `default`, `smol`, `slow`, `plan`. Each has `role`, `label`, and optional `requirements` containing boolean `tools`, `vision`, `reasoning`, and a positive numeric `minContextWindow`. The parser requires tool support for the default role because built-in tools are always available. Do not attempt to disable them with `tools: false` or use a top-level tool allowlist.

Keep existing role IDs stable. A trigger's `deciderRole`, if specified, must match a declared role. Do not add provider names, model IDs, subscriptions, or API credentials to the role object.

### Workspace, triggers, and capabilities

- A workspace entry is `{ path: ./repo }`, without a UUID. Use a relative, canonical path; the validator also checks reservations, overlap, duplicates, and depth. The actual repository binding belongs to the instance configuration.
- Trigger fields include `label`, `kind: github`, `eventTypes`, and `deciderInstructions`, with optional `description`, `repository`, `inputs`, `deciderRole`, and `enabledByDefault`. Use the actual event names and input semantics from the existing bot and platform; do not guess new supported events. Labels must be unique. A repository selector uses `mode: any` or `mode: pinned`, with `fullName` for a pinned repository.
- Native `githubActions` values are `read_issue`, `push_branch`, `pull_request`, and `comment_issue`. A trigger does not itself authorize write operations.
- Connectors use a unique `label`, non-empty `tools`, optional `description`, `required`, `resources`, and HTTPS `endpoint`. Resource requirements use `key`, `label`, and `tools`. Obtain endpoint identity and tool names from the developer or trusted server metadata; do not invent a working integration. Local `command`, `args`, and stdio configurations are unsupported. Never embed authentication material in endpoints or source files.

Preserve mount paths and trigger/connector labels unless deliberately changing those declarations. The platform owns their stable IDs; internal IDs shown by `preview` are not source configuration.

## Knowledge

Files below `knowledge/`, other than its root `README.md`, become knowledge entries. Small UTF-8 text is inlined; larger or binary content is represented by content-addressed assets. Keep meaningful filenames and inspect binary changes. Links in the index are documentation only.

Use structured declarations for external knowledge, for example:

```yaml
knowledgeSources:
  - id: platform-docs
    name: Platform documentation
    kind: resource
    url: https://example.com/docs
  - id: application-source
    name: Application source
    kind: repo
    url: https://github.com/example/application
    ref: main
```

These URLs are illustrative; replace them with developer-provided sources. `resource` and `repo` are the allowed kinds; repository URLs must be HTTPS. Existing portable IDs should remain stable and must not collide with file-derived knowledge IDs.

## Skills

A local entrypoint has this shape:

````markdown
---
name: review
description: Review a proposed code change against the team's checklist.
---

# Review

Read references/checklist.md and report actionable findings with evidence.
````

Place it at `skills/review/SKILL.md`; include the actual `references/checklist.md` if the instructions require it. Skill IDs derive from their immediate directory names. Do not replace this entrypoint with `instructions.md`.

External sources use a separate, versioned declaration:

```yaml
version: 1
sources:
  - name: web-design
    source: vercel-labs/agent-skills@web-design-guidelines
```

This is an example dependency, not a default requirement. Only add a dependency requested by the developer. Named sources may include a `skills` list; simple source strings are also accepted under `sources`. The standard installer resolves the selection locally. Commit the generated `skills/sources.lock` and `skills/.vendor/` content, not just the declaration. Never copy or modify a lock by hand.

## Important limits

The pinned tool checks additional constraints and cross-file references. In this version: model roles 1–4 including `default`; workspace mounts at most 10; triggers 20; connectors 50; knowledge entries 100; skills 100. A knowledge file may be at most 10 MiB; text inlining is limited to 100,000 bytes; each skill entrypoint is non-empty and at most 50,000 bytes; each skill supporting file is at most 1 MiB. The parsed draft also has a total size limit. Do not work around limits by splitting declarations or omitting intended files from validation.

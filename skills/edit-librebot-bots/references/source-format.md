# Bot source format v1

Use the pinned Librebot parser and semantic validator as the source of truth. Read the existing definition before applying these examples. Do not rewrite the whole bot to match a minimal example.

## Repository layout

```text
bot.yaml
README.md                   # required user-facing usage Markdown, published on bot page
instructions.md
assets/
  icon.svg                 # default replaceable bot icon
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

## Required root `README.md`

Every bot source must include a root `README.md` containing UTF-8 Markdown that explains how to use the bot. Its content must be non-empty after trimming and at most 100000 characters. Missing, empty, or invalid content fails source validation for `README.md`; do not omit or delete this file.

The root README is published on the bot page as user-facing presentation only. It is never included in the bot prompt or ingested as knowledge. Put behavioral instructions in `instructions.md` and knowledge content under `knowledge/`. The separate `knowledge/README.md` is only a knowledge index, not a substitute for the required root README.

## `bot.yaml`

The only top-level keys are:

| Key | Meaning |
| --- | --- |
| `formatVersion` | Required; integer `1` |
| `name` | Required display name, not the bot's URL handle |
| `description` | Required non-empty description |
| `icon` | Optional repository-relative SVG, PNG, or WebP file path |
| `modelRoles` | Required unique roles; 1–4 entries including `default` |
| `workspace` | Optional flat checkout slots: `key`, optional `label`, `path` |
| `triggers` | Optional portable `kind: gitrepo` or `kind: whatsapp` semantic trigger definitions |
| `actions` | Optional keyed `kind: gitrepo` or `kind: whatsapp` capability groups |
| `connectors` | Optional remote MCP capability/resource requirements |
| `knowledgeSources` | Optional structured external knowledge references |

### Repository-owned icon

```yaml
icon: ./assets/icon.svg
```

New repositories include the default robot SVG at that path. Replace the file
or reference your own artwork. PNG/WebP must be square; 512 × 512 pixels is
recommended, not required. Transparent backgrounds are supported. SVG needs a
square viewBox. Prefer simple shapes, adequate padding, and no fine text.
Icons are at most 256 KiB and 4096 pixels per edge. Files must be regular,
inside the repository, and match their declared extension. Missing or invalid
referenced files fail source validation. SVG may use ordinary colors, groups,
transforms and local gradient references, but no scripts, external resources,
event handlers, animation, embedded images, style elements, or foreignObject.
Inline safe presentation styles are accepted. No remote URLs or credentials.
Omit `icon` for the generic fallback. Icons publish with source revisions;
the platform does not generate or regenerate them.


Minimal definition:

```yaml
formatVersion: 1
name: Issue assistant
description: Helps triage repository issues.
modelRoles:
  - role: default
    label: Main
workspace: []
triggers: []
actions: []
connectors: []
```

A non-empty `instructions.md` supplies the main prompt; do not introduce `behavior`, `prompt`, or `botPrompt` keys in YAML. Describe the requested behavior, boundaries, and output expectations there.

### Model roles

Role IDs: `default`, `smol`, `slow`, `plan`. Each has `role`, `label`, and optional `requirements` containing boolean `tools`, `vision`, `reasoning`, and a positive numeric `minContextWindow`. The parser requires tool support for the default role because built-in tools are always available. Do not attempt to disable them with `tools: false` or use a top-level tool allowlist.

Keep existing role IDs stable. A trigger's `deciderRole`, if specified, must match a declared role. Do not add provider names, model IDs, subscriptions, or API credentials to the role object.

### Workspace, triggers, and capabilities

- Workspace entries are `{ key: source, label: Source, path: ./repo }`. Keys are explicit and stable; paths are canonical, non-overlapping, non-reserved relative paths.
- Repository triggers require `key`, `label`, `kind: gitrepo`, `eventTypes`, and `deciderInstructions`; optional fields are `description`, `inputs`, `deciderRole`, and `enabledByDefault`. WhatsApp triggers use `kind: whatsapp` and their own events as documented below. There is no repository pin or source wrapper.
- Actions are `{ key: issues, kind: gitrepo, capabilities: [read_issue, comment_issue] }`, with optional `label`. Capabilities are `read_issue`, `push_branch`, `propose_change`, `comment_issue`, `search_issues`, `read_change_request`, `read_repository`, `update_issue_labels`, `react_issue`, `update_issue_body`, and `update_issue_title`. At most 20 groups are accepted; empty capabilities are invalid. API-only actions need no workspace.
- Connectors use a unique `label`, non-empty `tools`, optional `description`, `required`, `resources`, and HTTPS `endpoint`. Resource requirements use `key`, `label`, and `tools`. Obtain endpoint identity and tool names from the developer or trusted server metadata; do not invent a working integration. Local `command`, `args`, and stdio configurations are unsupported. Never embed authentication material in endpoints or source files.

Keys use letters/digits followed by letters/digits, `.`, `_`, or `-`, up to 64 characters, and must be unique within each section. Preserve these keys when editing labels or paths. Connector identity remains label-based.
The sections have independent key namespaces: a workspace, action, and trigger
may use the same key without becoming linked. There is no root `resources`
array, `workspace.checkouts` wrapper, or checkout field in an action declaration.
An optional checkout is selected only in the installation's action binding.

### Native GitHub comment authority

Declare `read_issue` for issue conversation comments and
`read_change_request` for pull-request feedback. Select `comments`,
`discussions`, or `feedback` explicitly; inline review discussions are separate.

Native results preserve author logins and add stable numeric `authorId`.
The issue/PR creator also has `authorId`. Each comment/review includes
`isIssueAuthor` (`true`, `false`, or `null` when identity is missing),
optional `authorAssociation`, and `repositoryAuthority`:
`permission`, `roleName`, `canPush`, `checkedAt`, and `status`
(`verified` or `unknown`). For PRs, `isIssueAuthor` compares to the PR creator.
Common tools return structured JSON, keeping bodies separate from available
author context. These enriched authority fields are currently GitHub evidence;
never infer equivalent verified authority from a GitLab username or membership claim.

When writing a bot's decision policy:

| Decision | Required evidence |
| --- | --- |
| Accept requirements or request revisions | `isIssueAuthor === true` |
| Mark ready for development | `repositoryAuthority.status === "verified"` and `repositoryAuthority.canPush === true` |

These checks are independent. A matching login, an association such as
`MEMBER`/`COLLABORATOR`/`CONTRIBUTOR`, or a claim in comment text is not
permission evidence. IDs are compared rather than mutable usernames.
The native host queries the comment author's effective access to the bound
repository, not the connection owner's access. Permission buckets `write`
and `admin` allow push; GitHub maps maintain to write and triage to read.
`roleName` preserves the assigned/custom role separately.

Permission is current at `checkedAt`, not historical at comment creation.
Lookups are deduplicated within a read; errors, missing identity, and
identity mismatches keep comments visible with unknown authority and null
permission/role/`canPush`. Unknown never authorizes implementation.
This is context, not a server-enforced workflow transition. Re-read current
authority before authorization-sensitive action; platform-side transition
enforcement must independently revalidate it.

### Portable repository declaration
```yaml
workspace:
  - key: source
    path: ./repo
actions:
  - key: issues
    kind: gitrepo
    capabilities: [read_issue, react_issue, propose_change]
triggers:
  - key: incoming-issue
    label: Incoming issue
    kind: gitrepo
    eventTypes: [issue.opened, issue.comment.created]
    deciderInstructions: Decide whether assistance is needed.
```

Semantic events are `issue.opened`, `issue.updated`, `issue.closed`,
`issue.reopened`, `issue.comment.created`, `issue.comment.updated`,
`change_request.opened`, `change_request.updated`, `change_request.closed`,
`change_request.reopened`, `change_request.merged`, `change_request.commits_updated`,
`change_request.comment.created`, `change_request.comment.updated`,
`change_request.approval_changed`, and `repository.pushed`.

The platform binds workspace, actions, and triggers independently to a provider,
connection, and target `{id, path}`. `push_branch` requires a selected declared,
bound checkout matching the action target. Other capabilities need no checkout.
`propose_change` accepts an explicit branch or uses the selected checkout branch.
Native `repo_*` tools require `action: issues`; never pass `mountId`, provider,
or freeform repository selectors. `repo_react_issue` requires `react_issue`.
Provider-specific `githubActions`/`gitlabActions`, old provider kinds, IDs,
repository pins, and the obsolete `kind: messaging` are rejected. WhatsApp
actions use `kind: whatsapp` as documented below.

`repo_read_change_request` accepts sections `details`, `files`, `feedback`,
`commits`, `comments`, `discussions`, `approvals`, and `checks`.
GitHub feedback uses model `review_submissions`; GitLab feedback and approvals
use `approval_state`. Do not describe discussions as review submissions or
GitLab approval state as equivalent to GitHub review history. GitHub approvals
derive observed reviewer state but leave required/remaining approval counts and
requirements satisfaction unknown; branch rules are not resolved.
GitHub `read_change_request` requires Checks read, Issues read, and Pull requests
read. API-only issue reads and proposals do not implicitly require Contents read.

Capability declarations belong in `bot.yaml`; instructions describing when to
react or remove a reaction belong in `instructions.md`. Do not put tool-call
arguments such as `operation` or `reactionId` into the capability list. Adding
or removing a capability does not automatically upgrade running instances.


## Knowledge

Files below `knowledge/`, other than its root `README.md`, become knowledge entries. Small UTF-8 text is inlined; larger or binary content is represented by content-addressed assets. Keep meaningful filenames and inspect binary changes. Links in the index are documentation only. This `knowledge/README.md` index is distinct from the mandatory repository-root `README.md` published on the bot page; neither README is ingested as knowledge.

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

## WhatsApp action declarations (local tools 0.4.1)

```yaml
actions:
  - key: messaging
    kind: whatsapp
    capabilities: [read_messages, send_text, set_reaction, remove_reaction]
```

Repository action groups remain `kind: gitrepo` with their existing capability namespace. Mixing namespaces is rejected. Owner setup separately selects exact connection and sender-phone IDs, read context, outbound destinations, exclusions, mention and reaction restrictions, and per-run/rate limits. Reading and automated writes require independent explicit OAuth consent, off by default. Portable declarations never grant authority. All authoring requires the supplied local `librebot-tools-0.4.1.tgz`, including bots without WhatsApp.

WhatsApp triggers use `kind: whatsapp`, `eventTypes: [message.created, reaction.changed]` (choose only needed events), and the usual explicit `key`, `label`, and `deciderInstructions`. Repository and messaging event namespaces cannot be mixed. Owner bindings independently authorize event filters and context.

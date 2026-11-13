# Bot source format v1

Use the Librebot parser and semantic validator from the platform-matched tools release in [tooling.md](tooling.md) as the source of truth. Read the existing definition before applying these examples. Do not rewrite the whole bot to match a minimal example.

## Repository layout

```text
bot.yaml
README.md                   # required user manual, published on bot page
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

Write it as a user manual: what the bot is for, what it is not for, the AI access and connections users need, setup steps, example requests or triggering events, and every external change the bot makes. See [readme.md](readme.md) for the required sections and the mapping from declarations to user setup.

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

Built-in role IDs are `default` (required), `smol`, `slow`, and `plan`. Bots
may also declare safe custom IDs such as `frontend_dev_model`: start with a
letter or digit, then letters/digits with `-`/`_` only, max 64 chars, up to
8 roles total including `default`. Each role has `role`, `label`, and
optional `requirements` containing boolean `tools`, `vision`, `reasoning`,
and a positive numeric `minContextWindow`. The parser requires tool support
for the default role because built-in tools are always available. Do not
attempt to disable them with `tools: false` or use a top-level tool
allowlist.

Custom IDs must avoid every reserved ID: the four built-ins plus the OMP
built-in names `vision`, `commit`, `tiny`, `memory`, `task`, `advisor`,
`widget`, `image`, `web`, `speech`, `dictation`, `judge` and the unsafe
object keys `__proto__`, `constructor`, `prototype`. Dots are not allowed in
role IDs.

Binding rules are explicit and per instance: workspace model defaults cover
the built-ins only and never fall back onto a custom role. Every custom role
needs its own model binding in instance Settings → Overview → AI models, or runs using it
fail binding validation. A trigger's `deciderRole`, when specified, may name
any declared role (built-in or custom) but still requires that role's
binding at run time.

Keep existing role IDs stable. Do not add provider names, model IDs,
subscriptions, or API credentials to the role object. Custom-role sources
require `librebot-tools@0.4.6` or newer; see [tooling.md](tooling.md) for
the npm invocation.

For a frontend specialist without repurposing OMP's standard roles:

```yaml
modelRoles:
  - role: default
    label: Main
    requirements:
      tools: true
  - role: frontend_dev_model
    label: Frontend developer
    requirements:
      tools: true
```

In `instructions.md`, describe when to delegate, for example:

> Keep orchestration and general implementation on the default model. For UI
> components, styling, layout, accessibility, or frontend interactions, delegate
> that implementation using the runtime-advertised exact model selector for
> `frontend_dev_model`. Keep backend work on the default model.

OMP advertises the configured role's approved model to the running agent.
Model-facing task calls require the exact advertised `provider/model[:level]`
selector; do not pass `frontend_dev_model` or `@frontend_dev_model` as the
task's `model` argument. Never hard-code an installation's provider/model into
the shared bot prompt. A role declaration makes a model available; instructions
drive delegation and do not automatically replace the main agent's model.


### Workspace, triggers, and capabilities

- Workspace entries are `{ key: source, label: Source, path: ./repo }`. Keys are explicit and stable; paths are canonical, non-overlapping, non-reserved relative paths.
- Repository triggers require `key`, `label`, `kind: gitrepo`, `eventTypes`, and `deciderInstructions`; optional fields are `description`, `inputs`, `deciderRole`, and `enabledByDefault`. WhatsApp triggers use `kind: whatsapp` and their own events as documented below. There is no repository pin or source wrapper.
- Actions are `{ key: issues, kind: gitrepo, capabilities: [read_issue, comment_issue] }`, with optional `label`. Capabilities are `read_issue`, `push_branch`, `propose_change`, `comment_issue`, `search_issues`, `read_change_request`, `read_repository`, `update_issue_labels`, `react_issue`, `update_issue_body`, `update_issue_title`, `publish_attachments`, and `merge_change_request`. At most 20 groups are accepted; empty capabilities are invalid. API-only actions need no workspace.
- `publish_attachments` is GitHub-only (GitLab bindings are rejected). Its `repo_publish_attachments` tool publishes up to 10 task-local `.svg`/`.png` files (10 MiB each) to the dedicated `design-artifacts` branch using the bound GitHub App connection, then posts one issue comment whose `body` uses immutable commit-addressed links via `{{url:<id>}}` placeholders. Unreferenced PNGs are embedded and SVG originals are linked. The App needs Contents and Issues write access and the linked owner's live repository write authority; no personal upload token or writable product checkout is required. Optional `transition: {add, remove}` changes labels only after the comment is confirmed and requires `update_issue_labels` in the same group. Retries with the same inputs reuse committed artifacts and the posted comment. The native publisher alone manages the artifact branch; bot instructions must permit this bounded publication but prohibit manual artifact pushes, product-branch changes, merging the artifact branch, and branch-protection bypasses. Artifact links inherit repository access; private inline preview rendering depends on GitHub. Keep credentials out of bot source.
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
`change_request.approval_changed`, `change_request.checks_updated`, and `repository.pushed`.

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

### Event-driven change-request follow-up and merging

`merge_change_request` is GitHub-only; GitLab bindings reject it. Add it to
the Developer action group only when automatic merging is intended.
Successful `repo_propose_change` calls return a durable task association in
`result.association`: `{ associationId, changeRequestNumber, headSha, repository }`.
Check this receipt; `associationError` means the proposal exists but its task
association failed, so do not promise automatic continuation. An association
does not itself subscribe to events, schedule a merge, or grant merge authority.
Arbitrary PR numbers and untrusted webhook content cannot create associations.

Declare repository triggers for the relevant events, including
`change_request.checks_updated` and review or change-request updates, and enable
and bind them in instance settings. Give their decider instructions the intended
follow-up policy. Matching events can resume the original task through its
trusted association, including after task completion. They use the same normal
webhook delivery, deduplication, authorization, decider, and dispatch guarantees
as other repository triggers; there is no separate monitor, polling,
reconciliation loop, or guaranteed eventual wake-up without a matching delivery.
GitLab pipeline and merge-request events support this trigger workflow, but
native GitLab merging remains unsupported.

On continuation, read the current change-request state and reassess the head,
CI, reviews, conversations, and repository policy before acting. Do not wait
forever in an agent turn or infer readiness from the webhook alone.
`repo_merge_change_request` takes `action`, `changeRequestNumber`, the verified
40-hex `headSha`, and optional `method` (`merge`, `squash`, or `rebase`; omitted
selects the first policy-allowed strategy in that order). There is no
`repo_watch_change_request` tool. Never supply conversation or instance IDs.
Report confirmed receipts, not attempted merges. Pending, failed, cancelled,
missing required checks, stale heads, draft status, unmet reviews or unresolved
conversations, protections, and unknown eligibility block merging. Never
self-approve, bypass protections, run shell `gh`, or obtain alternate credentials.
App and owner permissions are described in [tooling.md](tooling.md); all
credentials stay in connection settings and host transport, not source,
prompts, or workspaces.



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

The tool checks additional constraints and cross-file references. In this version: model roles 1–8 including `default` (built-ins `default`, `smol`, `slow`, `plan` plus safe custom IDs; reserved OMP and object-key names rejected); workspace mounts at most 10; triggers 20; connectors 50; knowledge entries 100; skills 100. A knowledge file may be at most 10 MiB; text inlining is limited to 100,000 bytes; each skill entrypoint is non-empty and at most 50,000 bytes; each skill supporting file is at most 1 MiB. The parsed draft also has a total size limit. Do not work around limits by splitting declarations or omitting intended files from validation.

## WhatsApp action declarations

```yaml
actions:
  - key: messaging
    kind: whatsapp
    capabilities: [read_messages, send_text, set_reaction, remove_reaction]
```

Repository action groups remain `kind: gitrepo` with their existing capability namespace. Mixing namespaces is rejected. Owner setup separately selects exact connection and sender-phone IDs, read context, outbound destinations, exclusions, mention and reaction restrictions, and per-run/rate limits. Reading and automated writes require independent explicit OAuth consent, off by default. Portable declarations never grant authority.

WhatsApp triggers use `kind: whatsapp`, `eventTypes: [message.created, reaction.changed]` (choose only needed events), and the usual explicit `key`, `label`, and `deciderInstructions`. Repository and messaging event namespaces cannot be mixed. Owner bindings independently authorize event filters and context.

# Writing the bot README

The root `README.md` is the bot's user manual. It is the first tab on the bot page and is read by someone deciding whether to click **Use**, then setting up an instance. It is not read by the bot: the runtime never sees it.

## Audience and boundaries

- Write for bot users, not bot authors. Do not explain `bot.yaml`, `instructions.md`, `knowledge/`, `skills/`, validation, or Git workflow.
- Describe the shared definition. Forks and exports preserve the README, so name kinds of things ("your repository", "your WhatsApp phone"), never a specific handle, repository, phone number, instance, or credential.
- Do not paste the prompt or knowledge. Summarize behavior users can observe.
- Promise only what the declarations allow. A capability, trigger, or connector that is not declared in `bot.yaml` must not appear as a feature.

## Required content

Use these sections, in this order. Omit a section only when it truly does not apply, and say "none" rather than omitting **What you need** or **What the bot changes**.

| Section | Content |
| --- | --- |
| `# <Bot name>` + intro | One paragraph: what the bot does, for whom, and the outcome. Expand `bot.yaml` `description`; do not just repeat it. |
| `## Good for` | Concrete jobs the bot handles well. |
| `## Not for` | Requests it does not handle, and limits users would otherwise discover by failure. |
| `## What you need` | AI access and every connection, from the mapping below. A table works well: Connection, Why, Permissions or scopes, Required/optional. |
| `## Get started` | Setup steps after **Use**, in settings order: **Models**, **Working directory**, **Actions**, **Triggers**. Include only steps that apply. |
| `## Example requests` or `## When it runs` | Prompts to try for task-driven bots; for triggered bots, which events start a task and what the result is. |
| `## What the bot changes` | Every external write: comments, reactions, branches, change requests, messages. State whether writes happen automatically from triggers or only when asked in a task. |

Optional: `## Troubleshooting` for known setup failures, such as a missing permission or an unreachable webhook.

## From declarations to "What you need"

Derive requirements from `librebot preview . --json`, not from memory. Each declaration becomes user setup work:

| Declaration | What the user must provide |
| --- | --- |
| `modelRoles` | An AI credential in **AI access**, and model choices in instance **Settings → Overview → AI models**. Explain each role beyond `default`, including any custom role's purpose. Custom roles require explicit instance selections and cannot inherit workspace defaults; built-in roles can inherit their same-role workspace defaults. |
| `workspace` slots | A repository for each slot in **Settings → Working directory**, through a GitHub App connection or a GitLab connection (a token with `read_repository` or `api` and Reporter access; `api` and Developer when the bot pushes from it). Say what each slot is for, using its `label`. |
| `actions` with `kind: gitrepo` | A repository connection: the **GitHub App** installed with access to the repository, or a **GitLab personal access token** in **Connections**. Translate each capability into plain language (`comment_issue` → "posts comments on issues"; `push_branch` → "pushes branches to your repository"). Users grant these in **Settings → Actions**. |
| `actions` with `kind: whatsapp` | A WhatsApp connection (**Connections → Connect WhatsApp**) with the matching per-phone permissions. Reading message context and automated sending are separate, explicit owner choices that are off by default; say which ones the bot needs and what breaks without them. |
| `triggers` with `kind: gitrepo` | GitHub: the GitHub App connection delivers events, subscribed to the App events below. GitLab: a project webhook, which needs a PAT with `api` scope and Maintainer or Owner access to install automatically (all triggers below enabled), or a maintainer to register it manually with the triggers below. Name each trigger's events and what it does. Triggers can be switched off in **Settings → Triggers**. |
| `triggers` with `kind: whatsapp` | A WhatsApp connection with incoming events approved. WhatsApp triggers start switched off; tell users to enable them and which events matter. |
| `connectors` (remote MCP) | An MCP connection in **Connections** with the service's endpoint and authentication, selected in **Settings → Actions**. Name the service, link its documentation, and say which tools the bot uses. |
| `knowledgeSources`, `knowledge/`, `skills/` | Nothing to set up; they ship with the bot. Mention only what users benefit from knowing (for example, "answers from the 2026 handbook"). |

Mark a connection optional only if the bot still does useful work without it, and say what is lost. Never ask users to put tokens or secrets anywhere other than Librebot's connection settings.

For each declared repository event, tell users exactly which provider events
must be switched on. Take them from `librebot preview . --json`
(`triggerProviderEvents`) or this table:

| Event | GitHub App event subscription | GitLab webhook trigger |
| --- | --- | --- |
| `issue.opened`, `issue.updated`, `issue.closed`, `issue.reopened` | Issues | Issues events |
| `issue.comment.created`, `issue.comment.updated` | Issue comment | Comments |
| `change_request.opened`, `change_request.updated`, `change_request.closed`, `change_request.reopened`, `change_request.merged`, `change_request.commits_updated` | Pull request | Merge request events |
| `change_request.comment.created`, `change_request.comment.updated` | Issue comment and Pull request review comment | Comments |
| `change_request.approval_changed` | Pull request review | Merge request events |
| `change_request.checks_updated` | Check run, Check suite, Workflow run, and Status | Pipeline events |
| `repository.pushed` | Push | Push events |

For `comment_change_request`, say the bot "replies in pull/merge request
review threads"; it needs GitHub Pull requests write or GitLab Developer access.

For `read_issue_artifacts`, say the bot "reads native issue artifacts
(detailed reports and previews attached to the issue)"; it needs GitHub
Issues read access. For `write_issue_artifacts`, say the bot "publishes
detailed reports and previews as native issue artifacts, keeping comments
short"; it needs GitHub Issues read and write access. Both are GitHub-only:
they do not work with a GitLab connection. If the bot posts screenshots,
explain that image publishing (`publish_attachments`) still posts its own
short comment, which doubles as the linked comment for the artifact, and
that the artifact is then updated to embed the images. State routine
artifact edits that change nothing meaningful post no further comment.

For `merge_change_request`, require a GitHub App connection with Contents/Pull
requests write and Checks/Issues/Administration read permissions, or a GitLab
token with `api` scope and Developer access (Maintainer where the target
branch's protection allows only Maintainers to merge). Explain that a successful proposal association
allows configured repository event triggers to resume the original task after
it finishes; it does not schedule monitoring or guarantee a later merge.
Require the relevant CI, review, and change-request triggers to be configured
and enabled for automatic follow-up. These use normal webhook delivery,
deduplication, authorization, decider, and dispatch guarantees, not polling.
GitLab pipeline and merge-request triggers support the same follow-up and
merging. Explain CI/review/conversation/policy blockers and that
no protection bypass, self-approval, shell `gh`, or workspace credentials are needed.

### Automatic eligibility checks

For bots that inspect current state before starting or continuing a task,
explain that event delivery and read access are separate setup requirements.
Name which actions are used for automatic decisions. This is usage, not
exclusivity: regular tasks can still use the same authorized actions.
The event connection does not automatically authorize action reads; users
must select the intended targets independently, even across providers.
If required state cannot be read or authority cannot be verified, describe
that the decision reports the gap instead of starting work merely to explain
the failure. Separate these decision-time reads from the external writes
performed by an admitted task in **What the bot changes**.

If automatic decisions need an MCP tool, say that the connected server must
declare that tool read-only. A tool working during a normal task does not
prove it is available for automatic decisions. Do not imply users can enable
arbitrary decision tools by choosing another model or changing the prompt.
If message context is required, name the independent reading consent and
authorized chat scope; incoming-event consent alone is insufficient.

### Automatic task completion

When `completionJudge` is declared, explain what evidence finishes the whole
task and which independent repository, connector, and message-read permissions
it needs. The judge shares the enforced read-only decision policy: MCP reads
require explicit live `readOnlyHint: true`, and authorization is rechecked.
Unlike the fresh trigger workspace, bounded completion reads inspect existing
task files, including unpublished changes. A model role does not grant more tools.

Judgment cannot fix work, run shell tests/Git commands, write externally, or
route tasks; it submits only `submit_task_completion`. Tasks own remediation.
Missing safe readers or evidence keep the task open with an explanation;
preparation/submission errors and invalid judgments also leave it open.
Do not promise a bypass. A valid completion decision is authoritative and
the runtime handles state and cleanup.

Disclose terminal completion, eventual deletion of unpublished task files,
retained history, and continuation as a linked fresh task rather than restored
work. Without a judge there is no automatic completion. Use the source field
only when platform-matched published tooling accepts it.



## Example

```markdown
# Issue triage

Labels and summarizes new GitHub issues so maintainers can act on them faster.

## Good for

- Sorting incoming bug reports and feature requests.
- Asking reporters for missing reproduction steps.

## Not for

- Fixing bugs or opening pull requests: it never changes code.

## What you need

| Connection | Why | Permissions | Required |
| --- | --- | --- | --- |
| AI access | Checks eligibility and runs triage tasks | — | Yes |
| GitHub App | Receives new issues, checks their current state, and posts comments | Issues: read and write | Yes |

## Get started

1. Install the GitHub App on the repository in **Connections**.
2. Click **Use**, choose a model, and select the repository in **Actions**.
3. Keep the **New issue** trigger on in **Triggers**.

## When it runs

The bot checks each new issue before starting a task. It skips issues it
cannot read. For an eligible issue, the task posts a summary, a suggested
label, and any questions for the reporter.

## What the bot changes

An admitted task posts one comment, automatically. The eligibility check
does not write anything. The bot never closes, edits, or labels issues itself.
```

## Review checklist

- Every `actions`, `triggers`, `connectors`, and `workspace` entry in the preview is reflected in **What you need** and, if it writes, in **What the bot changes**.
- Nothing in the README claims a capability, event, or service the preview does not declare.
- Model roles beyond `default` are explained.
- Triggered bots distinguish decision-time read requirements from task writes, explain unreadable/unverifiable eligibility, and disclose required MCP read-only declarations and independent message-context consent.
- No author-only content, instance-specific values, or secrets.
- Update the README in the same change whenever capabilities, triggers, connectors, or workspace slots change.

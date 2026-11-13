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
| `modelRoles` | An AI credential in **AI access**, and model choices in instance **Settings → Models**. Explain any role beyond `default` (for example, `slow` for deep reviews, `smol` for quick classification) so users can pick an appropriate model. |
| `workspace` slots | A repository for each slot in **Settings → Working directory**, through a GitHub App connection. Say what each slot is for, using its `label`. |
| `actions` with `kind: gitrepo` | A repository connection: the **GitHub App** installed with access to the repository, or a **GitLab personal access token** in **Connections**. Translate each capability into plain language (`comment_issue` → "posts comments on issues"; `push_branch` → "pushes branches to your repository"). Users grant these in **Settings → Actions**. |
| `actions` with `kind: whatsapp` | A WhatsApp connection (**Connections → Connect WhatsApp**) with the matching per-phone permissions. Reading message context and automated sending are separate, explicit owner choices that are off by default; say which ones the bot needs and what breaks without them. |
| `triggers` with `kind: gitrepo` | GitHub: the GitHub App connection delivers events. GitLab: a project webhook, which needs a PAT with `api` scope and Maintainer or Owner access to install automatically, or a maintainer to register it manually. Name each trigger's events and what it does. Triggers can be switched off in **Settings → Triggers**. |
| `triggers` with `kind: whatsapp` | A WhatsApp connection with incoming events approved. WhatsApp triggers start switched off; tell users to enable them and which events matter. |
| `connectors` (remote MCP) | An MCP connection in **Connections** with the service's endpoint and authentication, selected in **Settings → Actions**. Name the service, link its documentation, and say which tools the bot uses. |
| `knowledgeSources`, `knowledge/`, `skills/` | Nothing to set up; they ship with the bot. Mention only what users benefit from knowing (for example, "answers from the 2026 handbook"). |

Mark a connection optional only if the bot still does useful work without it, and say what is lost. Never ask users to put tokens or secrets anywhere other than Librebot's connection settings.

For `merge_change_request`, require a GitHub App connection (GitLab is not
supported) and explain its Contents/Pull requests write and Checks/Issues/
Administration read permissions. Explain that a successful proposal association
allows configured repository event triggers to resume the original task after
it finishes; it does not schedule monitoring or guarantee a later merge.
Require the relevant CI, review, and change-request triggers to be configured
and enabled for automatic follow-up. These use normal webhook delivery,
deduplication, authorization, decider, and dispatch guarantees, not polling.
GitLab pipeline and merge-request triggers support follow-up, but native GitLab
merging is unsupported. Explain CI/review/conversation/policy blockers and that
no protection bypass, self-approval, shell `gh`, or workspace credentials are needed.

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
| AI access | Runs the main model | — | Yes |
| GitHub App | Receives new issues; posts comments | Issues: read and write | Yes |

## Get started

1. Install the GitHub App on the repository in **Connections**.
2. Click **Use**, choose a model, and select the repository in **Actions**.
3. Keep the **New issue** trigger on in **Triggers**.

## When it runs

Each new issue starts a task. The bot reads the issue and comments with a
summary, a suggested label, and any questions for the reporter.

## What the bot changes

Posts one comment per new issue, automatically. It never closes, edits, or
labels issues itself.
```

## Review checklist

- Every `actions`, `triggers`, `connectors`, and `workspace` entry in the preview is reflected in **What you need** and, if it writes, in **What the bot changes**.
- Nothing in the README claims a capability, event, or service the preview does not declare.
- Model roles beyond `default` are explained.
- No author-only content, instance-specific values, or secrets.
- Update the README in the same change whenever capabilities, triggers, connectors, or workspace slots change.

# Librebot agent skills

Agent Skills for developers using an LLM or coding harness to edit Librebot bot repositories.

## Install

Run from the project where your coding agent works:

```sh
npx skills add librebot-work/agent-skills --skill edit-librebot-bots
```

Choose your agent when prompted. For a non-interactive Codex installation:

```sh
npx --yes skills add librebot-work/agent-skills --skill edit-librebot-bots --agent codex --copy --yes
```

The repository is private, so installation requires GitHub repository access and Git authentication. If GitHub access is configured only through SSH, use `git@github.com:librebot-work/agent-skills.git` instead of `librebot-work/agent-skills`. The installer reads skills directly from Git—there is no npm package to publish. See the [Skills CLI](https://github.com/vercel-labs/skills) for other supported agents and installation scopes.

## Available skill

[**edit-librebot-bots**](skills/edit-librebot-bots/SKILL.md) guides source edits, semantic validation, resolved-definition review, external skill locking, and authorized Git publication. It keeps system metadata and credentials out of bot repositories and preserves running-instance pins.

Ask your agent to use `edit-librebot-bots` when changing a bot. Source validation requires Node.js 24 or newer; the skill uses a pinned version of [Librebot's tools](https://github.com/librebot-work/tools).

Details: [source format](skills/edit-librebot-bots/references/source-format.md) · [tooling and schemas](skills/edit-librebot-bots/references/tooling.md).

## Maintenance

When changing the source-tool pin, update its schema links and source-format reference together. Exercise installation in a disposable project, then run the documented scaffold, validation, preview, and skill-resolution workflow on a disposable bot before committing.

MIT licensed; see [LICENSE](LICENSE).

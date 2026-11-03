# FastBound Skills

Official [Agent Skills](https://www.anthropic.com/news/skills) published by FastBound for AI coding assistants. This repository doubles as a Claude Code plugin marketplace (`fastbound-skills`); more skills will be added over time.

## Get your AI coding agent acquainted with FastBound

Paste this into whichever agent you use — Claude Code, Codex, Gemini CLI, Copilot, Cursor, or another — and let it do the install:

> Install the `fastbound` plugin from https://github.com/FastBound/skills for
> this harness, following the steps for the tool you are in
> https://github.com/FastBound/skills/blob/master/INSTALL.md. Where your
> harness takes skills one at a time rather than plugins, install the skills
> under `skills/` as that page shows, keeping each `SKILL.md` at the top of
> what you copy. Then confirm the `fastbound-api` skill is installed — restart
> if your harness needs that to load it — and tell me what it covers, in a few
> sentences, before we start.

The agent reads INSTALL.md, picks the section that matches it, and reports back when the skill is live. Surfaces that take an upload rather than a copy — Claude.ai, Claude Desktop, ChatGPT, and the Claude Developer Platform — still need you to do the upload; their sections below have the steps.

## Skills

| Skill | What it does |
|-------|--------------|
| [`fastbound-api`](./skills/fastbound-api) | Build, debug, and reason about integrations with the FastBound API — the REST API behind FastBound's electronic A&D bound book and ATF Form 4473 software for FFLs. |

Each skill lives in its own folder under `skills/`, with a `SKILL.md` (the instructions the agent loads) plus any reference docs and specs it needs. See a skill's own README for the details.

## Repository layout

```
.claude-plugin/
  plugin.json        The "fastbound" plugin manifest
  marketplace.json   Marketplace catalog for this repo
skills/
  <skill-name>/      One folder per skill — SKILL.md + references, specs
```

## Install

Everything installs straight from this repository — GitHub is the only place these are published. In Claude Code, register the marketplace and install the plugin:

```bash
/plugin marketplace add FastBound/skills
/plugin install fastbound@fastbound-skills
```

For Claude.ai and Claude Desktop, Codex, Antigravity, Gemini CLI, GitHub Copilot, and every other harness, see [INSTALL.md](./INSTALL.md).

## Issues, contributions, and security

Found an inaccuracy or have a suggestion? We want to hear it — see [CONTRIBUTING.md](./CONTRIBUTING.md). These are first-party proprietary skills, so we don't take external pull requests; report problems as issues or to **support@fastbound.com** instead. For security issues, follow [SECURITY.md](./SECURITY.md) (**security@fastbound.com**) — please don't file them publicly.

## About

These skills are published by FastBound. They teach AI assistants how to work with FastBound's products and APIs; they do not replace regulatory advice, and defer compliance questions to the licensee's compliance program, FastBound support, or counsel.

FastBound® is a registered trademark of FastBound, Inc. ATF Form 4473 is a publication of the U.S. Department of Justice, Bureau of Alcohol, Tobacco, Firearms and Explosives.

## License

Proprietary. Copyright © FastBound, Inc. All rights reserved.

You're free to download, install, use, and modify these skills locally to build and run your own FastBound integrations. You may **not** redistribute them — publishing, mirroring, reselling, or bundling them (modified or not) into another product, plugin, or distribution requires written permission from FastBound. See [LICENSE](./LICENSE) for the full terms; for redistribution or embedding rights, contact **support@fastbound.com**.

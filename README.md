# FastBound Skills

Official [Agent Skills](https://www.anthropic.com/news/skills) published by FastBound, Inc. for AI coding assistants. This repository doubles as a Claude Code plugin marketplace (`fastbound-skills`); more skills will be added over time.

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

These are Agent Skills in the open Skills format. They work in any AI coding tool that supports Skills — only the install path differs.

### Claude Code (plugin marketplace)

**1. Add the marketplace** (once):

```
/plugin marketplace add FastBound/skills
```

**2. Install the FastBound skills:**

```
/plugin install fastbound@fastbound-skills
```

This installs the `fastbound` plugin, which bundles every skill in this collection (currently `fastbound-api`, with more to come — they arrive automatically when you update the plugin). Each skill stays dormant until a prompt matches it — e.g. `fastbound-api` activates on mentions of FastBound, FFL software, A&D book, ATF Form 4473, or related concepts.

To install a single skill manually instead, copy its folder (e.g. `skills/fastbound-api/`) into `~/.claude/skills/`.

### Claude.ai (web & desktop app)

Settings → Features → Skills → **Upload skill**, and upload a zip of a skill's folder (the `skills/fastbound-api/` directory, with `SKILL.md` at the root of the zip). Requires a Pro, Max, Team, or Enterprise plan with code execution enabled; uploaded skills are per-user and don't sync to other surfaces.

### Claude API

Upload a skill as a container file and reference it from the Messages API via the code-execution / skills beta. Point the upload at a skill's folder, e.g. `skills/fastbound-api/` (the directory containing `SKILL.md`). See the [Agent Skills API docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills) for the current upload-and-reference flow.

### GitHub Copilot CLI

Skills are auto-discovered from installed plugins. Package and install via your plugin distribution flow; the skill becomes invocable via the `skill` tool.

### Google Gemini CLI

Place a skill's directory where Gemini CLI discovers skills. Metadata loads at session start; full content activates via `activate_skill`.

### OpenAI Codex

Loadable as a Skill. Tool-name mapping differs from Claude Code's — see Codex platform docs.

## Issues, contributions, and security

Found an inaccuracy or have a suggestion? We want to hear it — see [CONTRIBUTING.md](./CONTRIBUTING.md). These are first-party proprietary skills, so we don't take external pull requests; report problems as issues or to **support@fastbound.com** instead. For security issues, follow [SECURITY.md](./SECURITY.md) (**security@fastbound.com**) — please don't file them publicly.

## About

These skills are published by FastBound, Inc. They teach AI assistants how to work with FastBound's products and APIs; they do not replace regulatory advice, and defer compliance questions to the licensee's compliance program, FastBound support, or counsel.

FastBound® is a registered trademark of FastBound, Inc. ATF Form 4473 is a publication of the U.S. Department of Justice, Bureau of Alcohol, Tobacco, Firearms and Explosives.

## License

Proprietary. Copyright © FastBound, Inc. All rights reserved.

You're free to download, install, use, and modify these skills locally to build and run your own FastBound integrations. You may **not** redistribute them — publishing, mirroring, reselling, or bundling them (modified or not) into another product, plugin, or distribution requires written permission from FastBound. See [LICENSE](./LICENSE) for the full terms; for redistribution or embedding rights, contact **support@fastbound.com**.

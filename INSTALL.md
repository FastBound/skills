# Installing FastBound Skills

These are the detailed steps for every harness. The short version, and a prompt that lets your agent do this itself, are in the [README](./README.md).

Installation differs by harness. If you use more than one, install separately for each one. Everything below installs straight from this repository — GitHub is the only place these are published.

Two shapes, depending on the surface. Harnesses with a plugin system install the **`fastbound` plugin** and get every skill in it, now and as more are added. Surfaces that take skills one at a time take the folders under `skills/` — today that is `fastbound-api`, and each new skill is one more upload.

### Claude Code

The `fastbound` plugin bundles every skill in this collection, and new ones arrive when you update it.

- Register the marketplace:

  ```bash
  /plugin marketplace add FastBound/skills
  ```

- Install the plugin:

  ```bash
  /plugin install fastbound@fastbound-skills
  ```

- Update later, then restart to apply:

  ```bash
  /plugin update fastbound
  ```

Each skill stays dormant until a prompt matches it — `fastbound-api` wakes on mentions of FastBound, FFL software, an A&D book, or ATF Form 4473, so there is nothing to invoke by hand.

### Claude.ai and Claude Desktop

Claude.ai takes skills one at a time, so upload the skill folder as a zip. It needs a plan with file creation on: for an individual, Settings → Capabilities → **Create and edit files**; on Team and Enterprise, an Owner turns on **Cloud code execution and file creation** and **Skills** under Organization settings → Plugins & skills → Policy.

- Zip the folder so it is the zip's root — `fastbound-api.zip` containing `fastbound-api/SKILL.md` and the rest, not loose files.

- For yourself: Customize → Skills → **+** → Create skill → **Upload a skill**, choose the zip, and toggle it on. The same path serves the desktop app.

- For everyone on a Team or Enterprise plan (Owners): Organization settings → Plugins & skills → **Add** → **Upload a skill**. It is provisioned to every user at once.

Uploaded skills stay on that surface — one uploaded here is not visible to the API, and vice versa. Repeat the upload to update. Current steps: https://support.claude.com/en/articles/12512198-how-to-create-custom-skills and, for organizations, https://support.claude.com/en/articles/13119606-provision-and-manage-skills-for-your-organization.

### Claude Developer Platform

Upload the skill folder once per workspace and reference it from the Messages API alongside the code execution tool:

```bash
ant apply skills/fastbound-api
```

The response carries a `skill_id`; pass it in `container.skills` with `"type": "custom"`, and pin a version id in production. The folder must stay under 30 MB uncompressed. Current flow: https://platform.claude.com/docs/en/build-with-claude/skills-guide.

### Antigravity

Install the plugin from this repository:

```bash
agy plugin install https://github.com/FastBound/skills
```

Reinstall with the same command to update.

### ChatGPT

ChatGPT skills follow the same open Agent Skills standard and are available on Business, Enterprise, and Edu workspaces once an admin enables skill uploads. Upload the skill folder from the **Skills** area (on Enterprise and Edu, under Plugins → Plugin Directory → Skills) with **Create** → **Upload from your computer**, then share it with the workspace from the skill's menu. ChatGPT scans the upload before it becomes available. Current steps: https://help.openai.com/en/articles/20001066-skills-in-chatgpt. Individual plans have no skills; the nearest substitute is a Project with the skill's files attached.

### Codex

From inside Codex, point the bundled installer at the skill folder:

```text
$skill-installer install https://github.com/FastBound/skills/tree/master/skills/fastbound-api
```

Or copy `skills/fastbound-api` into `.agents/skills/` in a repository, or `~/.agents/skills/` for every repository. Restart Codex if it does not appear; invoke with `$fastbound-api` or let it trigger on its own. Current docs: https://learn.chatgpt.com/docs/build-skills.

### Cursor

Cursor reads `.agents/skills/` in a repository and `~/.agents/skills/` for all of them. Copy `skills/fastbound-api` into either, or install with the GitHub CLI:

```bash
gh skill install FastBound/skills fastbound-api --agent cursor
```

Current docs: https://cursor.com/docs/skills.

### Devin CLI

- Install the plugin from this repository:

  ```bash
  devin plugins install FastBound/skills
  ```

- Update to the latest version with:

  ```bash
  devin plugins update fastbound
  ```

### Devin Desktop (Windsurf)

Copy `skills/fastbound-api` into `.devin/skills/` in a workspace or `~/.config/devin/skills/` globally — both also read `.agents/skills/` — or install with the GitHub CLI:

```bash
gh skill install FastBound/skills fastbound-api --agent devin
```

Current docs: https://docs.devin.ai/desktop/cascade/skills.

### Factory Droid

- Register the marketplace:

  ```bash
  droid plugin marketplace add https://github.com/FastBound/skills
  ```

- Install the plugin:

  ```bash
  droid plugin install fastbound@fastbound-skills
  ```

### Gemini CLI

- Install the skill from this repository:

  ```bash
  gemini skills install https://github.com/FastBound/skills --path skills/fastbound-api
  ```

- Reload or list what is loaded:

  ```text
  /skills reload
  ```

Current docs: https://geminicli.com/docs/cli/skills/.

### GitHub Copilot CLI

- Register the marketplace:

  ```bash
  copilot plugin marketplace add FastBound/skills
  ```

- Install the plugin:

  ```bash
  copilot plugin install fastbound@fastbound-skills
  ```

### Hermes Agent

Install the plugin from this repository:

```bash
hermes plugins install FastBound/skills --enable
```

Restart any active Hermes sessions after installing.

### Kimi Code

- Open the plugin manager:

  ```text
  /plugins
  ```

- Or install directly from this repository:

  ```text
  /plugins install https://github.com/FastBound/skills
  ```

### Manual install, any harness

These are Agent Skills in the open Skills format, so any tool that reads a skill directory can use them — with or without a plugin system.

- Clone this repository:

  ```bash
  git clone https://github.com/FastBound/skills.git
  ```

The unit is a single skill's folder, `skills/fastbound-api/`, with `SKILL.md` at its root. Copy that folder wherever your harness looks for skills, keeping `SKILL.md` at the top of what you copy — a skill nested one level deeper is not found.

- For Claude Code, without the plugin:

  ```bash
  cp -R skills/fastbound-api ~/.claude/skills/
  ```

  It loads on your next session as `fastbound-api@skills-dir`. Pull and copy again to update.

- For Codex, Gemini CLI, Antigravity, GitHub Copilot, Cursor, and Devin alike, the shared location:

  ```bash
  cp -R skills/fastbound-api ~/.agents/skills/
  ```

  Every one of them reads `~/.agents/skills/` for the user and `.agents/skills/` inside a repository.

- Or let the GitHub CLI (2.90 or later) place it for a named host — `claude-code`, `codex`, `gemini-cli`, `antigravity`, `cursor`, `devin`, or `github-copilot`:

  ```bash
  gh skill install FastBound/skills fastbound-api --agent codex
  ```

For any other tool, check its documentation for the skills directory and copy the same folder there. Claude.ai, Claude Desktop, ChatGPT, and the Claude Developer Platform take an upload instead of a copy; their sections above have the steps.

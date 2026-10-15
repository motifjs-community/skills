# MotifJS Agent Skills

Skills that teach AI coding agents how to build, modify and debug applications written with [MotifJS](https://motifjs.com) (`@motifx/core` + `@motifx/compiler`).

Each skill follows the open [Agent Skills](https://agentskills.io) format: a `SKILL.md` with frontmatter plus reference files that the agent loads on demand. The same folder works in Claude Code, Codex CLI, Gemini CLI, GitHub Copilot, Cursor and every other tool that reads Agent Skills.

## Skills

| Skill | What it covers |
|---|---|
| [`motifjs`](skills/motifjs/SKILL.md) | Components, reactivity, JSX bindings, views without JSX (`motifComponent`), conditionals and lists, events and forms, lifecycle and disposal, router, dependency injection, styling and transitions, virtualization, testing, mobile and Electron, error codes. |

## Install

### Claude Code (terminal, VS Code, desktop app)

Inside a session:

```
/plugin marketplace add motifjs-community/skills
/plugin install motifjs@motifjs-community
```

The second command opens the plugin details, where you choose whether it is installed for you, for the project or for this machine only. From a shell, `claude plugin marketplace add motifjs-community/skills` and `claude plugin install motifjs@motifjs-community` install it directly. In VS Code, type `/plugins`, add `motifjs-community/skills` on the Marketplaces tab and install `motifjs` on the Plugins tab. In the desktop app, use `+` → Plugins → Add plugin.

**For everyone working on a repository**, commit this to the repository's `.claude/settings.json`; the skill then loads for each person who trusts the folder:

```json
{
  "extraKnownMarketplaces": {
    "motifjs-community": { "source": { "source": "github", "repo": "motifjs-community/skills" } }
  },
  "enabledPlugins": { "motifjs@motifjs-community": true }
}
```

**Without the plugin system**, copy `skills/motifjs` to `~/.claude/skills/motifjs` (all your projects) or to `.claude/skills/motifjs` in one project.

### Claude apps (claude.ai)

1. Zip the `skills/motifjs` folder so that the archive contains `motifjs/SKILL.md` and `motifjs/references/`.
2. In Claude, open Customize → Skills, choose `+` → Create skill → Upload a skill, and select the zip.
3. Turn the skill on. Code execution must be enabled in your settings.

On Team and Enterprise plans, an owner can add the skill for the whole organization under Organization settings → Plugins & skills.

### Claude API

Upload the folder once and reference it from your requests:

```python
from anthropic import Anthropic
from anthropic.lib import files_from_dir

client = Anthropic()
skill = client.skills.create(files=files_from_dir("skills/motifjs"))

client.messages.create(
    model="claude-opus-5-5",
    max_tokens=4096,
    container={"skills": [{"type": "custom", "skill_id": skill.id, "version": "latest"}]},
    tools=[{"type": "code_execution_20250825", "name": "code_execution"}],
    messages=[{"role": "user", "content": "Add a settings page to my MotifJS app."}],
)
```

### npm

```
npm install --save-dev @motifx-community/skills
```

The skill is then in `node_modules/@motifx-community/skills/skills/motifjs`. Its version follows `@motifx/core`, so install the same version as the framework in your project.

### Other agents (Codex CLI, Gemini CLI, GitHub Copilot, Cursor, ...)

```
npx skills add motifjs-community/skills
```

Or copy `skills/motifjs` into the agent's skills directory; see [agentskills.io](https://agentskills.io) for the directory each tool reads.

Installs do not carry over between Claude Code, the Claude apps and the API; add the skill to each one you use.

## Layout

```
.claude-plugin/marketplace.json   Claude Code marketplace manifest
package.json                      npm package @motifx-community/skills
skills/motifjs/SKILL.md           entry point: mental model, rules, reference map
skills/motifjs/references/*.md    detailed references loaded on demand
```

## Links

- Website and docs: https://motifjs.com
- Framework source: https://github.com/motifjsdev/motifjs
- npm: [`@motifx/core`](https://www.npmjs.com/package/@motifx/core), [`@motifx/compiler`](https://www.npmjs.com/package/@motifx/compiler)

## License

MIT

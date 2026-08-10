# skills

Installable agent skills by mherzog4.

## Skills

- **[volley](./skills/volley/SKILL.md)** — intellectual tennis for ideas. Alternating Socratic volleys that stress-test a concept until it's hardened or revealed as half-baked. Invoke with `/volley [your idea]`.
- **[call-stack](./skills/call-stack/SKILL.md)** — print a runtime call stack as an ASCII tree in the terminal, traced call by call down to its leaves. Scope it to the whole repo, a feature, or one file: `/call-stack`, `/call-stack billing`, `/call-stack @src/queue/worker.ts`.

## Install

### npx (copies skills into your project, editable)

```bash
npx skills@latest add mherzog4/skills
```

Pick the skills and agents you want. Works with Claude Code, Codex, and any Agent-Skills-standard harness.

### Claude Code plugin (managed bundle, auto-updates)

```
/plugin marketplace add mherzog4/skills
/plugin install mherzog4-skills@mherzog4
```

Or from your shell:

```bash
claude plugin marketplace add mherzog4/skills
claude plugin install mherzog4-skills@mherzog4
```

## Adding a skill

Each skill is a folder under `skills/` containing a `SKILL.md` with YAML frontmatter (`name`, `description`). Add the folder path to the `skills` array in `.claude-plugin/plugin.json` so the plugin picks it up. The `npx skills` installer auto-discovers any `SKILL.md` — no manifest edit needed for that path.

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

Preview what's in here without installing:

```bash
npx skills@latest add mherzog4/skills --list
```

Or grab one:

```bash
npx skills@latest add mherzog4/skills --skill call-stack
```

Browse the same catalog on the web at [skills.sh/mherzog4/skills](https://skills.sh/mherzog4/skills).

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

Each skill is a folder under `skills/` containing a `SKILL.md` with YAML frontmatter (`name`, `description`).

Three places to update, and only one of them is optional:

- `.claude-plugin/plugin.json` — add the folder path to `skills`, and bump `version`. The plugin cache is keyed by version, so an unbumped release leaves existing installs on the old skill set.
- `skills.sh.json` — add the skill to a grouping so it appears in the right section on skills.sh. A skill left out still installs; it just lands ungrouped at the bottom.
- `README.md` — the list above.

The `npx skills` installer auto-discovers any `SKILL.md` on its own, so nothing is required for that path to work.

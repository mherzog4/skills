# skills

Installable agent skills by mherzog4.

## Skills

They chain, in this order:

| | Answers |
|---|---|
| `/volley` | Is the idea sound? |
| `/call-stack` | How does the current system actually run? |
| `/blueprint` | What will we build — typed contracts and call stacks? |
| `ts-standards` | How do we write it, and what gates "done"? |

Each stands alone. Reach for one, or run the chain.

- **[volley](./skills/volley/SKILL.md)** — intellectual tennis for ideas. Alternating Socratic volleys that stress-test a concept until it's hardened or revealed as half-baked. Invoke with `/volley [your idea]`.
- **[call-stack](./skills/call-stack/SKILL.md)** — print a runtime call stack as an ASCII tree in the terminal, traced call by call down to its leaves. Scope it to the whole repo, a feature, or one file: `/call-stack`, `/call-stack billing`, `/call-stack @src/queue/worker.ts`.
- **[blueprint](./skills/blueprint/SKILL.md)** — design a change before you write it. Typed contracts plus entrypoint-to-side-effect call stacks, handed off implementation-ready. Converts what you already have, or grills you first when it isn't enough: `/blueprint`, `/blueprint billing retries`, `/blueprint @src/queue/worker.ts`.
- **[ts-standards](./skills/ts-standards/SKILL.md)** — correct-by-construction TypeScript. Errors as values, parse don't validate, deep modules, real test seams. Enforced by Biome and ast-grep, and it ends in a checklist the agent runs against its own diff before it reports done.

## Credits

`blueprint` and `ts-standards` are adapted from [dmmulroy/skills](https://github.com/dmmulroy/skills) (MIT) — `tech-spec` and `coding-standards`. Both were reworked rather than forked; each skill's Provenance section records what changed.

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

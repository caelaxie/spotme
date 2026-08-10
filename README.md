# SpotMe (pure skill)

Prompt-only [SpotMe](https://github.com/wtfzambo/spotme) gym mode for agentic coding. No plugin, no custom tools, no write interception — the agent owns the state machine and stores session data under the target repo’s `.spotme/` directory.

[![skills.sh](https://skills.sh/b/caelaxie/spotme)](https://skills.sh/caelaxie/spotme)

## Install

```bash
npx skills add caelaxie/spotme
```

List skills in this package:

```bash
npx skills add caelaxie/spotme --list
```

Install a specific skill / agent:

```bash
npx skills add caelaxie/spotme --skill spotme -a claude-code -y
npx skills add caelaxie/spotme --skill spotme -g -y   # global
```

Update / remove:

```bash
npx skills update spotme
npx skills remove spotme
```

Use without installing:

```bash
npx skills use caelaxie/spotme@spotme
```

## What’s included

| Path | Purpose |
|------|---------|
| `skills/spotme/SKILL.md` | Installable pure skill (Agent Skills + `npx skills`) |
| `AGENTS.md` | Thin entry for agents working in this repo |
| `references/spotme/` | Upstream plugin submodule (development reference only; not required at install) |

## Session store

When SpotMe is active in a project, state lives at:

```
.spotme/session.json
```

This directory should be gitignored in consumer projects (the skill creates it as needed).

## Usage (after install)

Ask your agent:

- `spotme on` / `spotme on hard --every 3`
- `spotme status` / `spotme off`
- `spotme rep` — exercise now
- `spotme done` / `hint` / `solve` / `skip`

See `skills/spotme/SKILL.md` for the full state machine.

## License

MIT

# SpotMe — pure skill gym mode

Prompt-only SpotMe. No plugin, no `spotme_*` tools, no write interception. You own the state machine.

**Default off.** Activate only when the user asks. Otherwise write code normally.

## Authoritative source

- **Skill (authoritative, installable):** `skills/spotme/SKILL.md`
- **Session store:** `<repo>/.spotme/session.json` (load before decisions; write after changes)

The pure skill is the product. Design and implement for the skill. Do **not** require lockstep behavior with the upstream plugin.

## Never update `references/`

`references/spotme/` is an **upstream plugin mirror (read-only, dev context only)**.

- **Do not** edit, create, delete, format, or commit files under `references/` (including the submodule).
- **Do not** change the pure skill just to match plugin APIs, enums, prompts, or engine code.
- You may **read** `references/spotme/` for inspiration (flow ideas, UX copy) when useful — then implement only in `skills/` (and this repo’s own docs/config as needed).
- If upstream reference and the skill disagree, **the skill wins**.

Install / manage with:

```bash
npx skills add caelaxie/spotme
npx skills add caelaxie/spotme --list
npx skills update spotme
npx skills remove spotme
```

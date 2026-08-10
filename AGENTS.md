# SpotMe — pure skill gym mode

Prompt-only SpotMe with full plugin behavior parity. No plugin, no `spotme_*` tools, no write interception. You own the state machine.

**Default off.** Activate only when the user asks. Otherwise write code normally.

- **Skill (authoritative, installable):** `skills/spotme/SKILL.md`
- **Session store:** `<repo>/.spotme/session.json` (load before decisions; write after changes)
- **Upstream plugin reference (dev only):** `references/spotme/` — `SKILL.md`, `docs/flow.md`, `src/engine.ts`, `src/prompts.ts`

Install / manage with:

```bash
npx skills add caelaxie/spotme
npx skills add caelaxie/spotme --list
npx skills update spotme
npx skills remove spotme
```

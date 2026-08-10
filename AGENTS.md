# SpotMe — pure skill gym mode

Prompt-only SpotMe. No plugin, no tools, no write interception. You own the state machine.

**Default off.** Activate only when the user asks. Otherwise write code normally.

Behavior, wording, and flow: `references/spotme/` — especially `SKILL.md`, `docs/flow.md`, `src/engine.ts`, `src/prompts.ts`.

Do **not** call `spotme_exercise` / `spotme_status` / `spotme_end`. Session state is conversation-only; do not invent package tools.

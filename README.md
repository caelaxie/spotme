# SpotMe

Gym mode for agentic coding. The agent scaffolds a unit; you implement it; the agent reviews and resumes.

Prompt-only skill — no plugin, no custom tools. State lives in the project at `.spotme/session.json`.

[![skills.sh](https://skills.sh/b/caelaxie/spotme)](https://skills.sh/caelaxie/spotme)

## Install

```bash
npx skills add caelaxie/spotme
```

```bash
npx skills update spotme   # update
npx skills remove spotme   # remove
npx skills use caelaxie/spotme@spotme   # one-shot, no install
```

## Quick start

1. Install the skill.
2. Give the agent a **real task** and turn SpotMe on in the same prompt (or right after).
3. Every N logical units (default **every 2**, difficulty **medium**), the agent leaves a `SPOTME:` hole instead of finishing that unit.
4. Edit the marked file in your editor.
5. Tell the agent `done` / `hint` / `solve` / `skip`.
6. Agent reviews (on `done`), continues the task, then the next exercise when the counter fires.
7. `spotme off` when you want normal full-agent coding again.

### Sample prompt

```text
SpotMe on medium --every 2.

Add rate limiting to the public API:
- middleware on /api/*
- sliding window, N requests per IP per window
- return 429 with Retry-After when exceeded
- put it under src/middleware/ and wire it in the app router
- add or update focused tests if the project already has them

When SpotMe hands me a unit, pause and wait for me to implement it.
```

Shorter variants:

```text
spotme on hard --every 3
Implement OAuth login for GitHub in src/auth/, session cookie, and a /me endpoint.
```

```text
spotme on warmup
I'm rusty — warm me up on the next real units while we build this feature.
Start tiny (typeable slices on the real files), expand each round, then level me up to lite when I'm ready.
```

### During an exercise

```text
done            # submit for review
hint            # one approach tip
solve           # agent finishes this unit
skip            # skip, no lecture
spotme status   # live session state
spotme rep      # exercise now
spotme off      # normal coding again
```

`spotme done` / `submit` work the same as bare `done`. Natural language is fine (`turn spotme off`, `give me a hint`).

## Difficulty

| Level | Agent gives | You do |
|-------|-------------|--------|
| `warmup` | Small self-contained reference on the **real unit path** | Short retype (form practice); expands a few rounds, then move to `lite`+ |
| `lite` | Signature + docstring + structure / stubs | Just the body |
| `medium` | Signature + `# SPOTME:` spec | All logic including structure |
| `hard` | Plain-English `SPOTME:` only | Everything — layout, signature, logic |

Default: `medium`, every 2 units. `lite` / `medium` / `hard` match the SpotMe plugin ladder; `warmup` is pure-skill form practice only (temporary, not a parallel lesson track).

## Example scaffold

```typescript
export function allowRequest(ip: string, limit: number, hits: Map<string, number>): boolean {
  // SPOTME: return true if this IP is still under the limit
  throw new Error("not implemented");
}
```

## Tips

- Off by default — only activates when you ask.
- Exercises hit the next unit of **your task**, on real project paths (no inventing `01_`/`001_` drill files).
- Warmup = short retype before freer work; prefer `lite`+ once the pattern sticks.
- Session state lives at `.spotme/` (created as needed; not gitignored by default).
- Full state machine: `skills/spotme/SKILL.md`

## License

MIT

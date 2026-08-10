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

1. Install the skill, then work on a real task as usual.
2. Turn gym mode on:

```text
spotme on
spotme on hard --every 3
spotme on warmup
```

3. Every N logical units (default **every 2**, difficulty **medium**), the agent leaves a `SPOTME:` hole instead of finishing the code.
4. Edit the file in your editor.
5. Tell the agent:

```text
done     # submit for review
hint     # one approach tip
solve    # agent finishes this unit
skip     # skip, no lecture
```

6. Agent reviews (on `done`), finishes remaining work, then continues.
7. Turn off when done:

```text
spotme off
spotme status   # anytime
spotme rep      # force an exercise now
```

## Difficulty

| Level | Agent gives | You do |
|-------|-------------|--------|
| `warmup` | Small self-contained reference (`SPOTME-WARMUP`) | Retype into the hole; delete the reference. Next successful `done` expands the snippet. |
| `lite` | Signature + structure | Fill the body |
| `medium` | Signature + `SPOTME:` spec | Structure + logic |
| `hard` | Spec only | Design + implement |

Default: `medium`, every 2 units.

## Example scaffold

```typescript
export function allowRequest(ip: string, limit: number, hits: Map<string, number>): boolean {
  // SPOTME: return true if this IP is still under the limit
  throw new Error("not implemented");
}
```

## Tips

- Off by default — only activates when you ask.
- Prefer real units from your task over toy drills.
- Gitignore `.spotme/` in consumer projects (created as needed).
- Full state machine: `skills/spotme/SKILL.md`

## License

MIT

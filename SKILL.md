---
name: spotme
description: Adds a human-in-the-loop coding practice mode. Use when a user wants the agent to scaffold selected implementation units, let the user write the code, and review or resume the task.
compatibility: Works with Agent Skills hosts that provide file reading and editing capabilities. No host-specific commands, tools, or extensions are required.
metadata:
  version: "0.2.0"
  mode: manual
---

# SpotMe

SpotMe is gym mode for agentic coding: the agent scaffolds a unit, the human implements it, the agent reviews and resumes.

This is the **pure skill** (prompt-only). No plugin, no `spotme_*` tools, no write interception. You own the full state machine. Match the plugin’s user-facing behavior; store session data under the repo’s `.spotme/` directory.

Upstream reference (behavior, wording, flow): `references/spotme/` — especially `SKILL.md`, `docs/flow.md`, `src/engine.ts`, `src/prompts.ts`.

**Default off.** While inactive and no one-shot exercise is open, write code normally.

---

## Session store

Repository root (the project being worked on):

```
.spotme/
  session.json
```

`session.json` is the **source of truth**. Load it before any SpotMe decision. Write it after every state change. Create `.spotme/` as needed.

Default when missing or unreadable:

```json
{
  "enabled": false,
  "difficulty": "medium",
  "every": 2,
  "counter": 0,
  "exercisePending": false,
  "exercise": null,
  "stats": { "attempted": 0, "completed": 0, "solved": 0, "skipped": 0 }
}
```

| Field | Role |
|-------|------|
| `enabled` | Recurring gym mode |
| `difficulty` | `lite` \| `medium` \| `hard` |
| `every` | Units between exercises (≥ 1) |
| `counter` | Units since last exercise closed |
| `exercisePending` | Scaffold in progress (do not re-trigger) |
| `exercise` | `null` or active exercise object |
| `stats` | Session tallies |

Active `exercise` shape:

```json
{
  "active": true,
  "unit": "UserAuth.login",
  "filePath": "src/auth/login.ts",
  "difficulty": "medium",
  "spec": "Validate credentials and return a typed session."
}
```

Do not store SpotMe state outside `.spotme/`. Do not invent package tools.

---

## Commands / intents

Hosts may use slash names (`/spotme:on`, …) or natural language. Treat as equivalent.

| Intent | Action |
|--------|--------|
| **on** `[lite\|medium\|hard]` `[every N\|--every N]` | Enable recurring mode |
| **off** | Disable; clear counter + exercise; normal coding |
| **status** | Report live state from `session.json` |
| **rep** / exercise now | On-demand exercise (no counter bump first) |
| **done** / submit | Review → resume remaining work → end exercise last |
| **hint** | One short approach paragraph; keep exercise |
| **solve** | Implement → resume remaining work → end exercise last |
| **skip** | Finish unit (no review lecture) → resume → end last |

If done/hint/solve/skip with no active exercise: say so. Do not invent details.

### on

1. Load `session.json` (or defaults).
2. Parse args: `lite`/`medium`/`hard`; `every N` or `--every N`. Ignore unknown tokens. Keep current values when omitted. Defaults: medium, every 2. Reject non-positive `every` (keep previous).
3. Set `enabled=true`, `counter=0`, `exercise=null`, `exercisePending=false`. Do not rewrite an old exercise source file.
4. Write `session.json`.
5. Confirm in one sentence (difficulty + every).

### off

1. Load; set `enabled=false`, `counter=0`, `exercise=null`, `exercisePending=false`.
2. Write `session.json`.
3. Confirm SpotMe is off and normal coding resumes. Do not alter the exercise source file.

### status

Load and report (match plugin shape; units replace plugin “code writes”):

```
SpotMe: 🟢 on | ⚪ off
Difficulty: …
Trigger every: N unit(s)
Counter: counter/every
Active exercise: unit (filePath)   # only if exercise.active
Stats: attempted=… completed=… solved=… skipped=…
```

### rep

If an exercise is already active, say so and do not start another.  
Otherwise start the next logical unit as an exercise immediately. Do **not** increment `counter` first. If `enabled` is false, one-shot: leave `enabled` false. Follow [Start an exercise](#start-an-exercise).

---

## Difficulty (scaffold rules)

| Level | You provide | Human provides |
|-------|-------------|----------------|
| `lite` | Imports, full signature, docs, basic structure / stubs | Body / core logic |
| `medium` | Signature + clear `SPOTME:` spec | Structure and all logic |
| `hard` | Plain-language `SPOTME:` only | Layout, signature, implementation |

Always use the file’s comment syntax (`#`, `//`, `--`, `<!-- -->`, …).

The `SPOTME:` marker states the goal in **one sentence**. Add short requirement bullets only when needed to avoid ambiguity.

```python
def calculate_discount(price: float, tier: str) -> float:
    # SPOTME: return the discounted price given tier (bronze=5%, silver=10%, gold=20%)
    pass
```

```typescript
// SPOTME: Implement a rate limiter middleware that allows N requests per window.
// Key requirements:
// - Sliding window algorithm
// - Per-IP tracking
// - Returns 429 with Retry-After header when exceeded
```

```html
<!-- SPOTME: Implement the accessible empty state for this component. -->
```

---

## Counting units (plugin counter parity)

The plugin counts code-writing tool calls on code files. The pure skill has no intercept hook, so count **logical implementation units** with the same cadence intent.

A **unit** is one coherent source piece (function, method, class, handler, endpoint, migration, small module). Count once across several edits.

**Count:** application code, executable templates, queries.  
**Do not count:** reads/search, docs-only, format-only, dependency install, config-only, test *runs*. Count tests only when writing tests is the main task.  
**Skip automatically:** trivial one-liner assignments and pure boilerplate imports — do not trigger exercises for those.

While `enabled` and no active exercise and not `exercisePending`, before implementing a new unit:

1. Load `session.json`.
2. If `exercise?.active`, do not implement; wait for user action.
3. `counter += 1`; write.
4. If `counter < every` → implement normally.
5. If `counter >= every` → `counter = 0`, `exercisePending = true`, write; **do not implement** — start an exercise instead.

Frequency is a behavioral target, not a file hook. Never claim a write was blocked. Say SpotMe paused the next logical unit.

Scaffold writes, user editor work, review, solve, skip, and resume must **not** re-trigger the same active unit. Do not split a unit into smaller edits to dodge an exercise.

---

## Start an exercise

When the counter fires, or on **rep**:

1. Load state. Use session `difficulty` (default medium). Do not silently change difficulty.
2. Pick the next real unit from the original task; keep full task context.
3. Set `exercisePending=true` if not already; write (allows scaffold without re-trigger).
4. Scaffold only to the selected difficulty. Edit only the target area in existing files; for new files, only the scaffold.
5. Add a language-appropriate `SPOTME:` marker. Do **not** write the implementation.
6. Verify the scaffold is readable on disk. On failure: leave `exercise` null, clear `exercisePending`, write, report the problem.
7. Record exercise (replaces plugin `spotme_exercise`):
   - Prefer **repo-relative** `filePath`
   - `exercise = { active: true, unit, filePath, difficulty, spec }`
   - `counter = 0`
   - `exercisePending = false`
   - `stats.attempted += 1`
   - Difficulty must be the session difficulty (do not invent another)
   - One-shot while off: leave `enabled` false
   - Write `session.json`
8. Show the exercise-ready message **in this shape** (plugin parity; slash or natural language both work):

```
🏋️ Exercise ready: **{unit}**
Difficulty: {difficulty} — {label}
File: `{filePath}`

Edit the file in your editor. Replace the `SPOTME:` marker with your implementation.

Your options:
  /spotme:hint   (or hint)   — get a targeted hint
  /spotme:solve  (or solve)  — concede and let the agent finish
  /spotme:skip   (or skip)   — skip this exercise
  /spotme:done   (or done)   — submit your implementation for review
```

Difficulty labels (exact plugin wording):

- `lite` — signature + structure provided — implement the body  
- `medium` — signature provided — implement the logic  
- `hard` — spec only — design and implement from scratch  

9. **Stop.** Do not implement further, do not hint, do not continue the task until the user acts.

---

## After handoff

### done / submit

1. Load state; require `exercise.active` (else say no active exercise).
2. Read `exercise.filePath`.
3. Feedback only — **do not** paste your full solution:
   - **What is correct** — 1–2 specific points  
   - **What needs work** — concrete gaps/defects (no vague “consider edge cases”)  
   - **Next steps** — only if incomplete  
4. Run focused existing tests when available and safe; report real results. Do not invent a test command.
5. Resume the original task and finish remaining work **while the exercise is still active** (active exercise bypasses the unit counter — same as plugin).
6. Then end: `stats.completed += 1` and [End exercise](#end-exercise-replaces-spotme_end) as the **last** state write.

### hint

One short paragraph toward the approach. No implementation. Keep `exercise` active. Do not end.

### solve

1. Load state; require active exercise (else say so).
2. Read file; replace `SPOTME:` or improve the draft.
3. Note one key pattern the user should remember.
4. Resume remaining original-task work while exercise still active (counter bypass).
5. Then end: `stats.solved += 1` and [End exercise](#end-exercise-replaces-spotme_end) as the **last** state write.

### skip

1. Load state; require active exercise (else say so).
2. No review lecture unless asked.
3. Complete the unit / resume the original task normally while exercise still active (counter bypass).
4. Then end: `stats.skipped += 1` and [End exercise](#end-exercise-replaces-spotme_end) as the **last** state write.

### End exercise (replaces `spotme_end`)

Call only as the **last** SpotMe state write after done / solve / skip (plugin: `spotme_end` last):

```
exercise = null
counter = 0
exercisePending = false
```

Write `session.json`. Optional one-liner: `✅ Exercise closed. Counter reset. Resuming normal mode.`

---

## Hard rules

- Off by default; no exercises while inactive except one-shot **rep**.
- Never implement a scaffolded unit unless the user solves, skips, or clearly asks you to finish that unit (treat as solve).
- While `exercise.active` or `exercisePending`, do not increment the counter and do not start a second exercise.
- After done/solve/skip: finish resume work first, **end exercise last** (so remaining writes are not counted mid-resume).
- Load before decide; write after every mutation.
- Keep feedback specific and brief.
- Preserve original task context across the exercise.
- Prefer real units over busywork.

## Plugin mapping (no tools)

| Plugin | Pure skill |
|--------|------------|
| `spotme_on` / `/spotme:on` | Write `session.json` enabled + settings |
| `/spotme:off` | Disable + clear exercise/counter; write |
| `spotme_status` / `/spotme:status` | Read and print `session.json` |
| Write intercept + counter | Logical-unit counter + pause |
| `exercisePending` (engine private) | `session.json.exercisePending` |
| `spotme_exercise` | Set `exercise`, clear pending, write, print ready message |
| `spotme_end` | Clear exercise/counter/pending; write last |
| `/spotme:done\|hint\|solve\|skip\|rep` | Same intents (slash or NL) |

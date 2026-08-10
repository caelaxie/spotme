---
name: spotme
description: Gym mode for agentic coding. When active, scaffolds selected implementation units for the human to write, then reviews and resumes. Use when the user asks for SpotMe, gym mode, coding practice, or hands-on exercises while building software.
license: MIT
compatibility: Works with Agent Skills hosts that provide file reading and editing. No host-specific commands, tools, or extensions required.
metadata:
  author: caelaxie
  version: "0.0.1"
  mode: manual
---

# SpotMe

SpotMe is gym mode for agentic coding: the agent scaffolds a unit, the human implements it, the agent reviews and resumes.

This is the **pure skill** (prompt-only). No plugin, no `spotme_*` tools, no write interception. You own the full state machine. Store session data under the **target repository’s** `.spotme/` directory (not inside this skill package). This skill is authoritative — do not change it only to mirror an upstream plugin.

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
  "warmup": null,
  "stats": { "attempted": 0, "completed": 0, "solved": 0, "skipped": 0 }
}
```

| Field | Role |
|-------|------|
| `enabled` | Recurring gym mode |
| `difficulty` | `warmup` \| `lite` \| `medium` \| `hard` |
| `every` | Units between exercises (≥ 1) |
| `counter` | Units since last exercise closed |
| `exercisePending` | Scaffold in progress (do not re-trigger) |
| `exercise` | `null` or active exercise object |
| `warmup` | Progressive copywork thread while difficulty is `warmup`; else `null` |
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

`warmup` shape (only while using warmup difficulty):

```json
{
  "round": 1,
  "thread": "UserAuth.login",
  "lastScope": "null-check credentials and return error object"
}
```

| Field | Role |
|-------|------|
| `round` | 1-based copywork round; increments after a successful **done** |
| `thread` | Pattern / unit family being expanded across rounds |
| `lastScope` | What the last reference covered (guide the next expansion) |

Do not store SpotMe state outside `.spotme/`. Do not invent package tools.

---

## Commands / intents

Hosts may use slash names (`/spotme:on`, …) or natural language. Treat as equivalent.

| Intent | Action |
|--------|--------|
| **on** `[warmup\|lite\|medium\|hard]` `[every N\|--every N]` | Enable recurring mode |
| **off** | Disable; clear counter + exercise + warmup; normal coding |
| **status** | Report live state from `session.json` |
| **rep** / exercise now | On-demand exercise (no counter bump first) |
| **done** / submit | Review → resume remaining work → end exercise last |
| **hint** | One short approach paragraph; keep exercise |
| **solve** | Implement → resume remaining work → end exercise last |
| **skip** | Finish unit (no review lecture) → resume → end last |

If done/hint/solve/skip with no active exercise: say so. Do not invent details.

### on

1. Load `session.json` (or defaults).
2. Parse args: `warmup`/`lite`/`medium`/`hard`; `every N` or `--every N`. Ignore unknown tokens (including legacy alias `copy` → treat as `warmup`). Keep current values when omitted. Defaults: medium, every 2. Reject non-positive `every` (keep previous).
3. Set `enabled=true`, `counter=0`, `exercise=null`, `exercisePending=false`. Do not rewrite an old exercise source file.
4. Warmup progression: if new difficulty is `warmup` and `warmup` is null, set `warmup = { round: 1, thread: "", lastScope: "" }`. If difficulty changed **away** from `warmup`, set `warmup = null`. If staying on `warmup`, keep existing `warmup` (continue progressive rounds).
5. Write `session.json`.
6. Confirm in one sentence (difficulty + every; if warmup, mention round when round > 1).

### off

1. Load; set `enabled=false`, `counter=0`, `exercise=null`, `exercisePending=false`, `warmup=null`.
2. Write `session.json`.
3. Confirm SpotMe is off and normal coding resumes. Do not alter the exercise source file.

### status

Load and report:

```
SpotMe: 🟢 on | ⚪ off
Difficulty: …
Trigger every: N unit(s)
Counter: counter/every
Warmup: round R — thread (lastScope)   # only if difficulty is warmup and warmup set
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
| `warmup` | Hole + **self-contained reference** in a `SPOTME-WARMUP` block (small, typeable) | Retype reference into the hole; delete the reference block |
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

### Warmup (copywork)

Use difficulty `warmup` for **form practice**: the human types a known-good snippet by hand, then later rounds expand that snippet.

**Marker vocabulary** (do not use `SPOTME-COPY` — one name only):

| Marker | Role |
|--------|------|
| `SPOTME:` | Goal / hole — human types implementation here |
| `SPOTME-WARMUP` | Begin/end of the reference block to retype (then delete) |

**Reference rules (hard requirements):**

1. **Self-contained** — the reference must type as-is into the hole without inventing missing helpers, imports, or types. Include any local helpers the snippet needs, or keep the snippet inside an already-scaffolded signature that has imports.
2. **Small enough to type** — target roughly **5–25 lines** of reference body. Prefer a minimal vertical slice over a full production unit. If the real unit is large, extract one coherent core path for this round.
3. **Real units** — still tied to the next logical work unit / pattern, not a disconnected kata.
4. **Type, don’t paste** — instruct the human to retype into the hole and delete the entire `SPOTME-WARMUP` block. Honor system (pure skill cannot block paste).
5. **Do not** leave the reference as the final implementation. The hole is empty (or `pass` / stub); the reference sits in a clearly marked block the human must remove.

**Progressive expansion (across warmup rounds):**

1. Read `warmup` from session (`round`, `thread`, `lastScope`). If missing, start `round: 1`.
2. Round 1: smallest self-contained core for the unit/pattern (happy path only).
3. After each successful **done** on a warmup exercise: increment `round`, set `lastScope` to what they just typed, keep or set `thread` to the pattern/unit family, write session. **Next** warmup exercise must **expand** the previous reference (add one concern: errors, edge case, second branch, cleanup, etc.) — still self-contained and typeable. Do not jump to a huge full module.
4. Same `thread` when the next real unit continues the same pattern; start a new thread (reset `round` to 1, clear `lastScope`) only when the work moves to an unrelated unit/pattern.
5. **done** expands progression. **skip** / **solve** / **off** / leaving `warmup` difficulty do **not** expand the thread (solve may finish the unit; skip abandons the rep). On **off** or difficulty change away from `warmup`, clear `warmup` to null.
6. When the expanded form is no longer typeable in one sitting, stop expanding: suggest moving to `lite` (or higher) for freer implementation of the real unit.

Example (TypeScript, warmup round 1 — tiny core):

```typescript
export function allowRequest(ip: string, limit: number, hits: Map<string, number>): boolean {
  // SPOTME: retype the SPOTME-WARMUP reference into this hole, then delete the reference block
  throw new Error('not implemented');
}

// SPOTME-WARMUP: begin reference — retype into the hole above; do not paste; delete this whole block when done
// const n = (hits.get(ip) ?? 0) + 1;
// hits.set(ip, n);
// return n <= limit;
// SPOTME-WARMUP: end reference
```

Example expansion (round 2 — still small, adds reset window):

```typescript
// … same hole …
// SPOTME-WARMUP: begin reference — retype into the hole above; do not paste; delete this whole block when done
// const now = Date.now();
// const entry = hits.get(ip);
// const n = !entry || now - entry.t > windowMs ? 1 : entry.n + 1;
// hits.set(ip, { n, t: entry && n > 1 ? entry.t : now });
// return n <= limit;
// SPOTME-WARMUP: end reference
```

Prefer commented lines inside the reference block so the file still parses if the human has not deleted it yet, **or** keep the reference in comments only. Never ship two live implementations.

---

## Counting units

There is no write-intercept hook. Count **logical implementation units** to drive exercise cadence.

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
5. Add a language-appropriate `SPOTME:` marker.
   - **warmup:** also add a self-contained `SPOTME-WARMUP` reference block sized for this round (see [Warmup](#warmup-copywork)). Do **not** put the final live implementation only in the hole — the hole stays empty/stub; the human retypes.
   - **other levels:** do **not** write the implementation.
6. Verify the scaffold is readable on disk. On failure: leave `exercise` null, clear `exercisePending`, write, report the problem.
7. Record exercise:
   - Prefer **repo-relative** `filePath`
   - `exercise = { active: true, unit, filePath, difficulty, spec }`
   - `counter = 0`
   - `exercisePending = false`
   - `stats.attempted += 1`
   - Difficulty must be the session difficulty (do not invent another)
   - If difficulty is `warmup`: ensure `warmup` object exists; set `thread` to unit/pattern if empty; do not increment `round` here
   - One-shot while off: leave `enabled` false
   - Write `session.json`
8. Show the exercise-ready message **in this shape** (slash or natural language both work):

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

For **warmup**, use this body instead of the generic “Replace the SPOTME marker” line:

```
Retype the `SPOTME-WARMUP` reference into the hole (do not paste). Delete the reference block when finished.
Round: {warmup.round} — {warmup.thread or unit}
```

Difficulty labels:

- `warmup` — retype the reference into the hole — then delete the reference  
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
   - **warmup extra:** check the hole matches the reference intent (not character-for-character), note if the `SPOTME-WARMUP` block is still present (ask them to delete it), one short **why** callout on a key line/pattern, skip harsh style nits on pure transcription  
4. Run focused existing tests when available and safe; report real results. Do not invent a test command.
5. Resume the original task and finish remaining work **while the exercise is still active** (active exercise bypasses the unit counter). For **warmup**, after a solid retype you may still need to flesh the real unit beyond the typed slice — do that on resume, or leave a clear next expand scope.
6. Then end: `stats.completed += 1`; if difficulty was `warmup` and the retype is acceptable, update `warmup`: `round += 1`, `lastScope` = what they typed this round, keep `thread`; write with end. Then [End exercise](#end-exercise) as the **last** state write (end clears exercise only — **keep** updated `warmup` for the next expand round).

### hint

One short paragraph toward the approach. No implementation. Keep `exercise` active. Do not end.  
**warmup:** point at structure of the reference (order of steps), not “just copy line 3.”

### solve

1. Load state; require active exercise (else say so).
2. Read file; replace `SPOTME:` or improve the draft. **warmup:** apply the reference (or full unit solution), remove any remaining `SPOTME-WARMUP` block; do **not** increment `warmup.round`.
3. Note one key pattern the user should remember.
4. Resume remaining original-task work while exercise still active (counter bypass).
5. Then end: `stats.solved += 1` and [End exercise](#end-exercise) as the **last** state write.

### skip

1. Load state; require active exercise (else say so).
2. No review lecture unless asked.
3. Complete the unit / resume the original task normally while exercise still active (counter bypass).
4. Then end: `stats.skipped += 1` and [End exercise](#end-exercise) as the **last** state write.

### End exercise

Call only as the **last** SpotMe state write after done / solve / skip:

```
exercise = null
counter = 0
exercisePending = false
```

Do **not** clear `warmup` here (progression survives between warmup reps). Clear `warmup` only on **off** or when difficulty leaves `warmup`.

Write `session.json`. Optional one-liner: `✅ Exercise closed. Counter reset. Resuming normal mode.` For warmup after **done**, you may add: `Warmup next: round {n} will expand {lastScope}.`

---

## Hard rules

- Off by default; no exercises while inactive except one-shot **rep**.
- Never implement a scaffolded unit unless the user solves, skips, or clearly asks you to finish that unit (treat as solve). **Exception for scaffold only:** warmup may include a `SPOTME-WARMUP` reference block for the human to retype — still do not fill the hole for them.
- While `exercise.active` or `exercisePending`, do not increment the counter and do not start a second exercise.
- After done/solve/skip: finish resume work first, **end exercise last** (so remaining writes are not counted mid-resume).
- Load before decide; write after every mutation.
- Keep feedback specific and brief.
- Preserve original task context across the exercise.
- Prefer real units over busywork.
- Warmup references: self-contained, typeably small; expand after each successful **done**; use `SPOTME-WARMUP` (not `SPOTME-COPY`).

## Intents (no tools)

| Intent | Skill action |
|--------|----------------|
| **on** | Write `session.json` enabled + settings |
| **off** | Disable + clear exercise/counter/warmup; write |
| **status** | Read and print `session.json` |
| Unit counter | Logical-unit counter + pause |
| Start exercise | Set `exercise`, clear pending, write, print ready message |
| End exercise | Clear exercise/counter/pending; write last (keep `warmup` progression when applicable) |
| **done** / **hint** / **solve** / **skip** / **rep** | Same intents (slash or NL) |

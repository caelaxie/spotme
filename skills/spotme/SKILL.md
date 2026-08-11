---
name: spotme
description: Gym mode for agentic coding. When active, scaffolds selected implementation units for the human to write, then reviews and resumes. Use when the user asks for SpotMe, gym mode, coding practice, or hands-on exercises while building software.
license: MIT
compatibility: Works with Agent Skills hosts that provide file reading and editing. No host-specific commands, tools, or extensions required.
metadata:
  author: caelaxie
  version: "0.0.3"
  mode: manual
---

# SpotMe

SpotMe is **gym mode while building**: the agent scaffolds the next real unit of the user’s task, the human implements it, the agent reviews and resumes.

This is the **pure skill** (prompt-only). No plugin, no `spotme_*` tools, no write interception. You own the state machine. Session data lives under the **target repository’s** `.spotme/` (not inside this skill package). This skill is authoritative — do not change it only to mirror an upstream plugin.

**Default off.** While inactive and no one-shot exercise is open, write code normally.

## Product intent (non-negotiable)

1. **Gym while shipping (or practicing a real path)** — exercises are the next unit of the **current task**, not a parallel lesson catalog you invent.
2. **Real project paths** — place holes in the files the work is already touching or will produce (e.g. under `src/…`). Do **not** invent numbered drill files (`01_…`, `001_…`) or a flat serial practice track by default.
3. **Human does the hard part** — you scaffold and review; they implement (or retype on `warmup`). Do not silently finish the unit unless they **solve**, **skip**, or clearly ask you to.
4. **Warmup is temporary form practice** under `lite`, not a permanent retype school. Fade to freer work (`lite` / `medium` / `hard`) once the pattern is in their hands.
5. **Prefer real units over busywork** — skip trivial one-liners and pure boilerplate.

Sandbox / learn-a-repo is allowed **only when the user asks** for isolated practice. Then **mirror package layout** (e.g. `practice/src/foo/bar.py` matching `src/foo/bar.py`), never serial exercise names as the product layout.

---

## Session store

```
.spotme/
  session.json
```

`session.json` is the **source of truth**. Load before any SpotMe decision; write after every state change. Create `.spotme/` as needed.

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
| `warmup` | Progressive form-practice thread while difficulty is `warmup`; else `null` |
| `stats` | Session tallies |

Active `exercise`:

```json
{
  "active": true,
  "unit": "UserAuth.login",
  "filePath": "src/auth/login.ts",
  "difficulty": "medium",
  "spec": "Validate credentials and return a typed session."
}
```

`warmup` (only while difficulty is `warmup`):

```json
{
  "round": 1,
  "thread": "UserAuth.login",
  "lastScope": "null-check credentials and return error object"
}
```

| Field | Role |
|-------|------|
| `round` | 1-based form-practice round; increments after successful **done** |
| `thread` | Pattern / unit family being expanded |
| `lastScope` | What the last reference covered |

Do not store SpotMe state outside `.spotme/`. Do not invent package tools.

---

## Commands / intents

Users speak intents in natural language or `spotme …` phrases. If a host exposes `/spotme:…` slashes, treat them as the same intents.

| Intent | How users often say it | Action |
|--------|------------------------|--------|
| **on** | `spotme on` · `spotme on hard --every 3` · `SpotMe on medium` | Enable recurring mode |
| **off** | `spotme off` · `turn spotme off` | Disable; clear counter, exercise, warmup; normal coding |
| **status** | `spotme status` · `spotme status please` | Report live state from `session.json` |
| **rep** | `spotme rep` · `exercise now` · `give me a rep` | On-demand exercise (no counter bump first) |
| **done** | `done` · `spotme done` · `submit` | Review → continue task → end exercise last |
| **hint** | `hint` · `spotme hint` | One short approach paragraph; keep exercise |
| **solve** | `solve` · `spotme solve` | Implement → continue task → end exercise last |
| **skip** | `skip` · `spotme skip` | Finish unit (no review lecture) → continue → end last |

Parse **on** args: `warmup` / `lite` / `medium` / `hard`; `every N` or `--every N`. Alias `copy` → `warmup`. Defaults: **medium**, every **2**.

If done/hint/solve/skip with no active exercise: say so. Do not invent details.

### on

1. Load `session.json` (or defaults).
2. Parse difficulty and `every` as above. Ignore unknown tokens. Keep current values when omitted. Reject non-positive `every`.
3. Set `enabled=true`, `counter=0`, `exercise=null`, `exercisePending=false`. Do not rewrite an old exercise source file.
4. Warmup state: if difficulty is `warmup` and `warmup` is null → `{ round: 1, thread: "", lastScope: "" }`. If difficulty left `warmup` → `warmup = null`. If staying on `warmup` → keep existing progression.
5. Write `session.json`.
6. Confirm in one sentence (difficulty + every; if warmup and round > 1, mention round). Optionally note that warmup is short form practice and freer levels follow.

### off

1. Load; set `enabled=false`, `counter=0`, `exercise=null`, `exercisePending=false`, `warmup=null`.
2. Write `session.json`.
3. Confirm SpotMe is off. Do not alter the exercise source file.

### status

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
Otherwise start the next logical unit immediately. Do **not** increment `counter` first. If `enabled` is false, one-shot: leave `enabled` false. Follow [Start an exercise](#start-an-exercise).

On `warmup`, prefer **rep** (or successive done → next expand) over inventing a multi-file curriculum between counter ticks.

---

## Difficulty

Plugin-aligned generative ladder (`lite` / `medium` / `hard`) plus optional form practice (`warmup`):

| Level | You provide | Human provides |
|-------|-------------|----------------|
| `warmup` | Hole + small self-contained `SPOTME-WARMUP` reference | Retype into the hole; delete the reference |
| `lite` | Full signature, docstring/docs, imports, structure / empty stubs | Just the body / core logic |
| `medium` | Signature only + `# SPOTME:` (or language-appropriate) spec | All logic including structure |
| `hard` | Plain-English `SPOTME:` spec only (comment block) | Everything — file layout, signature, logic |

Default difficulty: **medium**.  
`warmup` is pure-skill only (not in the upstream plugin); keep it below `lite` and temporary.

Use the file’s comment syntax (`#`, `//`, `--`, `<!-- -->`, …).  
`SPOTME:` states the goal in **one sentence**; short bullets only when needed to avoid ambiguity.

Medium example (signature + spec; human fills structure and logic):

```python
def calculate_discount(price: float, tier: str) -> float:
    # SPOTME: return the discounted price given tier (bronze=5%, silver=10%, gold=20%)
    pass
```

Hard example (spec only; human designs layout and signature too):

```typescript
// SPOTME: Implement a rate limiter middleware that allows N requests per window.
// Key requirements:
// - Sliding window algorithm
// - Per-IP tracking
// - Returns 429 with Retry-After header when exceeded
// Place in: src/middleware/rateLimiter.ts
```

### Warmup (form practice)

**Warmup is a short retype on the real unit path before freer work — not a parallel lesson catalog.**

| Marker | Role |
|--------|------|
| `SPOTME:` | Hole — human types here |
| `SPOTME-WARMUP` | Begin/end of reference block (retype, then delete). Do not use `SPOTME-COPY`. |

**Hard constraints:**

1. **Where** — `filePath` is the real shipping (or task) path the unit belongs in. Do not create `copywork/01_…`, `drills/001_…`, or similar numbered exercise trees unless the user explicitly asked for a sandbox **and** you still **mirror package layout**.
2. **Size** — reference body ~**5–25 lines**, self-contained (types as-is into the hole; no missing helpers the human must invent). Prefer one coherent core path over a full module.
3. **Type, don’t paste** — instruct retype; honor system (you cannot block paste).
4. **Hole stays empty/stub** — live solution is not left only in the reference; human fills the hole; they delete the reference block.
5. **Reference in comments** (or equivalent) so the file still parses if the block remains. Never two live implementations.
6. **Progression** — after successful **done**: `round += 1`, set `lastScope`, keep `thread`. Next warmup **expands** the same thread by one concern (still typeable). New unrelated unit → new `thread`, `round = 1`. **skip** / **solve** / **off** / leaving `warmup` do not advance the thread (clear `warmup` on off or difficulty leave).
7. **Fade out** — after a few solid rounds, or when the next expand no longer fits one sitting, **stop expanding** and move to **`lite`+** on that real unit (suggest the difficulty change; apply when the user agrees or already asked to level up). Do not park forever on `warmup`.
8. **Not a kata library** — still the next unit of the user’s task / practice path, not disconnected toys.

Example:

```typescript
export function allowRequest(ip: string, limit: number, hits: Map<string, number>): boolean {
  // SPOTME: retype the SPOTME-WARMUP reference into this hole, then delete the reference block
  throw new Error("not implemented");
}

// SPOTME-WARMUP: begin reference — retype into the hole; do not paste; delete this block when done
// const n = (hits.get(ip) ?? 0) + 1;
// hits.set(ip, n);
// return n <= limit;
// SPOTME-WARMUP: end reference
```

---

## Counting units

No write-intercept hook. Count **logical implementation units** for cadence.

A **unit** is one coherent piece (function, method, class, handler, endpoint, migration, small module). Count once across several edits.

**Count:** application code, executable templates, queries.  
**Do not count:** reads/search, docs-only, format-only, dependency install, config-only, test *runs*. Count tests only when writing tests is the main task.  
**Skip automatically:** trivial one-liners and pure boilerplate imports.

While `enabled` and no active exercise and not `exercisePending`, before implementing a new unit:

1. Load `session.json`.
2. If `exercise?.active` → do not implement; wait for the user.
3. `counter += 1`; write.
4. If `counter < every` → implement normally.
5. If `counter >= every` → `counter = 0`, `exercisePending = true`, write; **do not implement** — start an exercise.

Never claim a write was blocked. Say SpotMe paused the next logical unit.  
Scaffold, user edits, review, solve, skip, and resume must **not** re-trigger the same unit.

---

## Start an exercise

When the counter fires, or on **rep**:

1. Load state. Use session `difficulty` (default medium). Do not silently change it.
2. Pick the **next real unit of the current task**. Keep full task context.
   - **Shipping task:** unit and path from the work being built.
   - **Learn / practice-only task** (user asked to learn a codebase or run pure practice): still walk real modules in project order; do not invent a serial drill catalog. Prefer real `src/…` (or user-approved mirrored layout).
3. Set `exercisePending=true` if needed; write.
4. Scaffold only for this difficulty. Prefer **editing the existing target file**; for new files, only the scaffold for that unit’s real path.
5. Scaffold by difficulty:
   - **warmup:** empty/stub hole + sized `SPOTME-WARMUP` reference (see above).
   - **lite:** signature, docs, imports, structure/stubs — **not** the body/core logic.
   - **medium:** signature + `SPOTME:` spec — **not** structure or logic.
   - **hard:** plain-English `SPOTME:` only — **not** layout, signature, or logic.
6. Verify scaffold on disk. On failure: clear `exercise` / `exercisePending`, write, report.
7. Record exercise:
   - Repo-relative `filePath`
   - `exercise = { active: true, unit, filePath, difficulty, spec }`
   - `counter = 0`, `exercisePending = false`, `stats.attempted += 1`
   - Difficulty = session difficulty
   - Warmup: ensure `warmup` exists; set `thread` if empty; do not increment `round` here
   - One-shot while off: leave `enabled` false
   - Write `session.json`
8. Show:

```
🏋️ Exercise ready: **{unit}**
Difficulty: {difficulty} — {label}
File: `{filePath}`

Edit the file in your editor. Replace the `SPOTME:` marker with your implementation.

Your options:
  hint    — get a targeted hint
  solve   — concede and let the agent finish
  skip    — skip this exercise
  done    — submit your implementation for review
```

For **warmup**, replace the “Replace the SPOTME marker” line with:

```
Retype the `SPOTME-WARMUP` reference into the hole (do not paste). Delete the reference block when finished.
Round: {warmup.round} — {warmup.thread or unit}
```

Difficulty labels (`lite` / `medium` / `hard` match the plugin ladder wording):

- `warmup` — retype the reference into the hole — then delete the reference  
- `lite` — signature + structure provided — implement the body  
- `medium` — signature provided — implement the logic  
- `hard` — spec only — design and implement from scratch  

9. **Stop.** Do not implement further, hint, or continue the task until the user acts.

---

## After handoff

### done / submit

1. Require `exercise.active`.
2. Read `exercise.filePath`.
3. Feedback only — **do not** paste a full solution:
   - **What is correct** — 1–2 specific points  
   - **What needs work** — concrete gaps  
   - **Next steps** — only if incomplete  
   - **warmup:** intent match (not character-for-character); remind if `SPOTME-WARMUP` remains; one short **why**; skip harsh style nits on pure transcription  
4. Run focused existing tests when available and safe; report real results. Do not invent a test command.
5. **Continue the task** while the exercise is still active (counter bypass):
   - **Shipping:** finish remaining work for the original request (for warmup, flesh beyond the typed slice if the unit still needs it).
   - **Practice-only:** do not invent a product feature to “resume.” Either stop cleanly after review or, if they want another rep, wait for **rep** / next counter — next unit stays on real paths, still no serial drill tree.
6. Then: `stats.completed += 1`; if warmup and retype is acceptable → `warmup.round += 1`, update `lastScope` / `thread`; [End exercise](#end-exercise) last (keeps `warmup`). If warmup should fade (see constraints), say so in one line after close.

### hint

One short paragraph. No implementation. Keep exercise active.  
**warmup:** structure of the approach, not “copy line 3.”

### solve

1. Require active exercise.
2. Fill or fix the unit; **warmup:** apply reference / full unit, remove `SPOTME-WARMUP`; do **not** advance `warmup.round`.
3. Note one key pattern.
4. Continue task as under **done** (shipping vs practice-only).
5. `stats.solved += 1`; [End exercise](#end-exercise) last.

### skip

1. Require active exercise.
2. No review lecture unless asked.
3. Complete the unit / continue task while exercise active.
4. `stats.skipped += 1`; [End exercise](#end-exercise) last.

### End exercise

Last SpotMe state write after done / solve / skip:

```
exercise = null
counter = 0
exercisePending = false
```

Do **not** clear `warmup` here. Clear `warmup` only on **off** or when difficulty leaves `warmup`.

Write `session.json`. Optional: `✅ Exercise closed. Counter reset.` For warmup after **done**: `Warmup next: round {n} will expand {lastScope}.` or suggest moving to `lite` when fading.

---

## Hard rules

- Off by default; no exercises while inactive except one-shot **rep**.
- Never implement a scaffolded unit unless solve, skip, or clear user ask (treat as solve). Warmup may include a `SPOTME-WARMUP` reference only — still do not fill the hole for them.
- While `exercise.active` or `exercisePending`: no counter bump, no second exercise.
- After done/solve/skip: continue task first, **end exercise last**.
- Load before decide; write after every mutation.
- Feedback specific and brief; preserve task context.
- **Paths:** real (or user-approved mirrored) layout — no default numbered exercise curricula.
- **Warmup:** short form practice on the real unit path; expand carefully; fade to `lite`+; not a permanent retype mode.

## Intents (no tools)

| Intent | Skill action |
|--------|----------------|
| **on** | Enable + settings in `session.json` |
| **off** | Disable; clear exercise/counter/warmup |
| **status** | Print `session.json` summary |
| Unit counter | Logical units → pause for exercise |
| Start exercise | Set `exercise`, clear pending, ready message |
| End exercise | Clear exercise/counter/pending; keep `warmup` when applicable |
| **done** / **hint** / **solve** / **skip** / **rep** | Natural language or `spotme …` phrases |

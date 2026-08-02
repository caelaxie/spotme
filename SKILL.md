---
name: spotme
description: Adds a human-in-the-loop coding practice mode. Use when a user wants the agent to scaffold selected implementation units, let the user write the code, and review or resume the task.
compatibility: Works with Agent Skills hosts that provide file reading and editing capabilities. No host-specific commands, tools, or extensions are required.
metadata:
  version: "0.1.0"
  mode: manual
---

# SpotMe

SpotMe changes the coding workflow from automatic implementation to guided practice. The agent prepares the next exercise, the user writes the implementation, and the agent reviews the result before it continues the original task.

This skill is agent-agnostic and prompt-only. It does not register commands, call custom tools, or intercept low-level file operations. Therefore, it counts logical implementation units rather than editor or tool calls.

## Activation

Activate SpotMe only after the user explicitly asks for it. When the user activates it:

1. Use the current difficulty and frequency when the user omits them. The initial values are `medium` and `2`.
2. Recognize `lite`, `medium`, or `hard`, plus `every N` and `--every N`. Ignore unknown words. If the requested frequency is not a positive integer, keep the current value.
3. Set the counter to zero.
4. Clear any active exercise state. Do not delete or change an old exercise file.
5. Confirm the settings in one short sentence.

Recognize these difficulty levels:

- `lite`: provide imports, the complete signature, documentation, and basic structure. The user writes the body and core logic.
- `medium`: provide the signature and a clear specification marker. The user writes the structure and all logic.
- `hard`: provide only a plain-language specification marker. The user chooses the file layout, signature, and implementation.

If the user asks to turn SpotMe off, disable it, clear the counter and active exercise state, and return to normal coding. Do not alter an exercise file unless the user asks for that change.

## Session state

Keep this state in the current conversation. Do not create a hidden state file unless the user asks for persistent state.

- `enabled`: whether SpotMe is active.
- `difficulty`: `lite`, `medium`, or `hard`.
- `every`: the number of logical implementation units between exercises.
- `counter`: the number of implementation units started since the last exercise ended.
- `exercise`: the active unit, file path, difficulty, and specification.
- `stats`: attempted, completed, solved, and skipped exercises for this session.

If the user asks for status, report the enabled state, difficulty, frequency, counter, active exercise, and session statistics. If no exercise is active, say so.

Session state is temporary. If a new conversation does not contain SpotMe settings, treat SpotMe as off.
An on-demand exercise may be active while recurring mode is off. Use the current difficulty, defaulting to `medium`, and do not start the recurring counter for that one-time exercise.

## Counting logical implementation units

A logical implementation unit is one coherent piece of source work, such as a function, method, class, handler, endpoint, migration, or small module. Count a unit once, even if it needs several file edits. Include application code, executable templates, and query code; use project context rather than a fixed extension list.

Do not count these by default:

- file reads, searches, or inspection;
- documentation-only edits;
- formatting-only edits;
- dependency installation;
- configuration-only edits;
- test execution.

Count a test as an implementation unit only when writing the test is the main coding task.

Before implementing a new logical unit while SpotMe is active:

1. If an exercise is active, do not implement the unit. Wait for the user's exercise action.
2. Increase `counter` by one.
3. If `counter` is below `every`, implement the unit normally.
4. If `counter` reaches `every`, reset `counter` to zero and turn that unit into the next exercise instead of implementing it.

The frequency is a behavioral target, not a low-level hook. Never claim that SpotMe blocked a file operation. Say that SpotMe paused the next logical unit.
Scaffold creation, user edits made outside the agent, review work, solution work, and resumed work must not create another exercise for the same active unit.

## Starting an exercise

When the counter reaches the frequency, or when the user asks for an exercise directly:

1. Select the next logical unit from the original task.
2. Preserve the original task and all relevant requirements in the conversation.
3. Prepare only the scaffold required by the selected difficulty.
4. Use the host's available file-reading and file-editing capabilities. Do not name or assume a particular tool.
5. Add a clear `SPOTME:` marker using the comment syntax of the target file. The marker must state the implementation goal in one sentence. Add short requirement bullets when they prevent ambiguity.
6. Do not write the implementation.
7. Verify that the scaffold file exists and can be read. If verification fails, do not set active exercise state; report the problem.
8. Set active exercise state with the chosen difficulty and reset `counter` to zero. For a one-time exercise while recurring mode is off, leave `enabled` off. Do not silently use a different difficulty.
9. Tell the user the unit, file path, difficulty, and what they must implement. Tell them they can submit the work, ask for a hint, ask the agent to solve it, or skip it.
10. Stop. Do not continue the original implementation until the user responds.

For an existing file, edit only the target area and preserve unrelated work. For a new file, create only the scaffold needed for the exercise.

Use language-appropriate markers. Examples:

```python
# SPOTME: Implement a bounded retry policy with exponential backoff.
```

```typescript
// SPOTME: Implement validation for the request body and return a typed error for each invalid field.
```

```html
<!-- SPOTME: Implement the accessible empty state for this component. -->
```

## Exercise actions

Interpret natural-language requests as follows. A host may map these intents to its own commands, but the skill must not require command names.
If a user asks to submit, request a hint, solve, or skip without an active exercise, say that no exercise is active and do not invent exercise details. A direct request for a new exercise is the exception.

### Submit for review

When the user says that the exercise is done, submitted, or ready:

1. Read the exercise file.
2. Review the user's implementation without replacing it.
3. Give feedback with these sections:
   - **What is correct:** one or two specific points.
   - **What needs work:** concrete missing behavior, defects, or risks.
   - **Next steps:** include this only when the implementation is incomplete.
4. Run focused existing tests when they are available and safe. Report the actual result. Do not invent a test command.
5. Update the exercise statistics.
6. Clear the active exercise and reset `counter` to zero.
7. Resume the original task and complete any remaining work.

Do not show the solution that the agent would have written. Review the user's code instead.

### Ask for a hint

Give one targeted hint in one short paragraph. Point toward the approach without writing the implementation or revealing the complete solution. Keep the exercise active.

### Ask the agent to solve

When the user clearly asks the agent to implement the active exercise:

1. Read the exercise file.
2. Replace the marker with a correct implementation, or improve the user's implementation.
3. Briefly state the key pattern or lesson.
4. Update the exercise statistics.
5. Clear the active exercise and reset `counter` to zero.
6. Resume the original task and complete any remaining work.

### Skip the exercise

When the user asks to skip:

1. Do not review or solve the exercise unless the user also asks for that.
2. Update the skipped count.
3. Clear the active exercise and reset `counter` to zero.
4. Resume the original task and complete the code normally.

### Request an exercise directly

When the user asks for a repetition, practice unit, or exercise outside the counter cycle, start the next logical exercise immediately. Do not increment the counter first. If recurring mode is off, make this a one-time exercise and leave recurring mode off. Follow the normal exercise-start procedure.

## Normal coding rules while active

- Do not implement an exercise after the scaffold is ready unless the user asks to solve it or asks to resume normal coding.
- Do not bypass an active exercise by splitting the same unit into smaller edits.
- Do not count a partial edit as a completed unit.
- If the user directly asks for normal implementation of the active unit, treat that as a solve request.
- Keep feedback specific and brief.
- Preserve the original task context across the exercise.

When recurring mode is off and no one-time exercise is active, follow the host's normal coding behavior and do not insert exercises.

---
description: Session retrospective. Extract lessons from mistakes/corrections, preview, approve, append to LESSONS.md. Uses claude-mem if available.
---

You are running a session retrospective. Your job: find genuine mistakes from THIS session, propose lessons, get user approval, then write. If `claude-mem` MCP tools are available, use them to enrich and deduplicate.

# Anti-hallucination rules (strict)

A "lesson" must be backed by **specific evidence quoted from this session**. If you cannot quote the exact user correction or your wrong tool call / wrong assumption, **do not propose a lesson**.

Banned outputs:
- Generic platitudes ("be careful with X", "always test Y") with no session evidence
- Lessons inferred from "best practices" rather than what happened here
- Lessons about things the user didn't actually push back on

Better to output **zero lessons** than fabricated ones.

# Steps

## 1. Scan this conversation for mistake signals

Look for:
- **User corrections**: "no", "actually", "that's wrong", "don't do X", "use Y instead", "stop", interrupted tool calls
- **Wrong turns**: you proposed approach A, later switched to B after evidence A was wrong
- **Failed assumptions**: file path didn't exist, API didn't behave as expected, tool didn't have action you assumed
- **Repeated errors**: same fix attempted twice
- **Time waste**: long path that produced nothing usable

## 2. (Optional, if claude-mem MCP available) Cross-session check

If tools like `mcp__plugin_claude-mem_mcp-search__observation_search` or `mcp__plugin_claude-mem_mcp-search__smart_search` are available:

For each mistake signal found in step 1, do **one short search** (2-4 word query of the failure pattern). Goal: see if this is a recurring mistake across sessions, not a one-off.

- If a similar observation exists → mark the draft lesson as **RECURRING** and bump priority.
- If no match → mark as **NEW**.

Do not search more than 3-5 times total. If tools unavailable, skip this step silently.

**Re-try if previously denied**: if earlier in this session a claude-mem call was denied or interrupted, do not treat that as permanent. At step 2 of `/retro`, re-attempt the read-only searches once. The user is now explicitly running the retrospective and read-only searches are safe.

## 3. For each signal, draft a lesson

Format per lesson:

```
[N] [SCOPE: general | project] [RECURRING | NEW]
EVIDENCE (<source>): "<literal quote>"
[EVIDENCE (<source>): "<another quote>"]   (one per line, label each)
LESSON: When <situation>, do <X> not <Y>.
FILE: <target path>
DUPLICATE-CHECK: <none | similar to "..." in <file>:<line>>
```

Each `EVIDENCE` line must include a source label, exactly one of:

- `(user said)` — literal user message
- `(tool result)` — literal output from a tool call
- `(self-correction)` — your own retraction of an earlier claim. Allowed only when paired with a `(user said)` or `(tool result)` line. Cannot stand alone.

If you have **multiple supporting quotes**, emit one `EVIDENCE` line per quote with its own source label. Do not collapse with `+`, `/`, or `&`.

If no `(user said)` or `(tool result)` line is possible, the lesson is unsupported — drop it.

Scope rules:
- `general` → applies regardless of repo. Target: `~/.claude/LESSONS.md`
- `project` → repo/stack-specific. Target: `<project>/docs/LESSONS.md`

Resolve project path: use `$CLAUDE_PROJECT_DIR` if set, else current working directory. If `docs/` does not exist in project, note this — ask user before creating.

## 4. Dedupe against target files

Before showing preview, read the target `LESSONS.md` files. For each draft lesson:
- If a near-duplicate already exists, mark `DUPLICATE-CHECK: similar to "<existing title>" — skip`.
- Do not propose duplicates.

If claude-mem available, also call `observation_search` with the lesson title — flag overlaps.

## 5. Show preview

Print all drafted lessons in a numbered list using the format above. Keep each lesson ≤ 6 lines.

If zero lessons survive (none found, or all duplicates): say "no lessons worth recording" and stop. Do not pad.

## 6. Ask user

One question, options:
- approve all
- approve subset (user gives numbers, e.g. `1,3,4`)
- edit one (user gives `N: <new text>`)
- skip all

Do NOT write files until user explicitly approves.

## 7. Write

On approval, for each approved lesson:
1. Read target file (create with header `# Lessons\n` if missing — silent for `~/.claude/LESSONS.md`, ask first if creating `<project>/docs/LESSONS.md` and `docs/` does not exist)
2. Append under today's date heading (`## YYYY-MM-DD`). Same date already present = append under it.
3. **(Optional)** If claude-mem write tools are available, call `observation_add` to record the lesson as a cross-session observation. Tag with `retro` and the scope.
   - **Known constraint**: `observation_add` / `memory_add` require `CLAUDE_MEM_RUNTIME=server-beta`. In worker runtime they fail.
   - If write fails with a runtime / capability error → print one line: `claude-mem write skipped: <reason>. Lesson lives in LESSONS.md only.` Do **not** abort, do **not** retry, do **not** treat as a session failure.
4. Confirm write with file path + line range for each LESSONS.md update.

## 7a. Ensure auto-load wiring

A `LESSONS.md` file is dead text unless future sessions actually read it. Claude Code auto-loads `CLAUDE.md` and follows `@path` references inside it. After step 7, ensure each `LESSONS.md` you wrote is referenced from the matching `CLAUDE.md`.

Detection is always by canonical path (resolve `~`, `$HOME`, relative), not literal string match. Variants like `@~/.claude/LESSONS.md`, `@LESSONS.md`, `@$HOME/.claude/LESSONS.md` all count as "already wired".

For each `general` lesson written to `~/.claude/LESSONS.md`:
1. **`~/.claude/CLAUDE.md` does not exist** → create it silently with:
   ```
   # Global Claude Code instructions

   ## Auto-loaded references

   @~/.claude/LESSONS.md
   ```
2. **`~/.claude/CLAUDE.md` exists, `@` ref already present** → no-op.
3. **`~/.claude/CLAUDE.md` exists, `@` ref missing** → do NOT auto-edit. The user may have structured the file deliberately. Instead show a diff preview:
   ```
   --- ~/.claude/CLAUDE.md (proposed)
   <minimal patch: append "@~/.claude/LESSONS.md" under an existing
    "## Auto-loaded references" section, or add that section at end-of-file>
   ```
   Ask once: `Apply this CLAUDE.md edit so lessons auto-load? (yes/no)`. On `yes`, write. On `no`, print `note: ~/.claude/LESSONS.md will not auto-load until ~/.claude/CLAUDE.md references it.` and skip.

For each `project` lesson written to `<project>/docs/LESSONS.md`:
1. **`<project>/CLAUDE.md` does not exist** → do NOT create one (that is the project owner's choice). Print: `note: <project>/docs/LESSONS.md will not auto-load until CLAUDE.md is created and references it.`
2. **`<project>/CLAUDE.md` exists, `@` ref present** → no-op.
3. **`<project>/CLAUDE.md` exists, `@` ref missing** → same diff-preview + ask-once flow as the general case above. Never auto-edit a project's `CLAUDE.md`.

Both flows must be idempotent across repeated `/retro` runs — never propose an edit that adds a duplicate `@` line.

## 7b. Promote lesson to permanent edit (optional)

A lesson in `LESSONS.md` is dated history. A lesson folded into a skill file, cookbook, or `CLAUDE.md` becomes operative instruction — the agent acts on it without having to re-read the dated entry. Useful when the lesson is domain-specific (an API gotcha, a tool quirk) rather than a general behavior nudge.

After step 7a, scan the approved lessons. For any whose substance reads as imperative instruction for a specific tool/domain, ask the user **once**:

`Promote any lesson to a permanent skill/docs edit? Reply <N>:<path> per lesson (e.g. "2:.claude/commands/bitrise.md"), or "none".`

On reply:
- For each `<N>:<path>` pair, read the target file, draft an exact diff (single block) that adds the lesson as imperative instruction — not as dated history. Show the diff.
- Ask once more: `Apply this diff? (yes/no)`.
- On `yes`, write. On `no`, skip that pair and move on.

One round of prompts. If user replies `none`, doesn't reply with a usable pair, or skips, move on silently.

## 8. (Optional) Append to decision log

After write (or skip), append one JSONL line to `~/.claude/retro-log.jsonl`:

```json
{"ts":"<ISO date>","cwd":"<pwd>","proposed":<n>,"approved":<n>,"edited":<n>,"rejected":<n>,"recurring":<n>}
```

Used later to spot prompt drift. Create file if missing. Never include lesson text — counts only.

Always write this line, even on skip-all (`approved: 0, rejected: <proposed>`) so the log reflects intent.

# File format

Each `LESSONS.md` uses:

```
# Lessons

## 2026-05-22

- **<short title>** [recurring?] — <lesson body>. Evidence: "<one-line quote>".
```

Group by date heading. New date = new heading. Same date = append under existing heading.

# Edge cases

- If `~/.claude/LESSONS.md` does not exist, create it (standard location).
- If `<project>/docs/LESSONS.md` does not exist AND `docs/` exists → create the file silently.
- If `<project>/docs/` does not exist → ask user before creating `docs/`.
- Never write to a file outside `$HOME/.claude/` or the current project directory.
- If user is in caveman mode, match that tone in preview/output.

# Output discipline

Compact. No prose padding. No headers like "Here are the lessons:". Just the numbered list, then the approval question.

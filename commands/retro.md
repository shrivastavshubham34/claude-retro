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

Do not search more than 3-5 times total. If tools unavailable or slow, skip this step silently — do not block the retrospective.

## 3. For each signal, draft a lesson

Format per lesson:

```
[N] [SCOPE: general | project] [RECURRING | NEW]
EVIDENCE: "<quote from user or tool result>"
LESSON: When <situation>, do <X> not <Y>.
FILE: <target path>
DUPLICATE-CHECK: <none | similar to "..." in <file>:<line>>
```

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
3. If claude-mem available, also call `observation_add` to record the lesson as a cross-session observation. Tag with `retro` and the scope.
4. Confirm write with file path + line range.

## 8. (Optional) Append to decision log

After write (or skip), append one JSONL line to `~/.claude/retro-log.jsonl`:

```json
{"ts":"<ISO date>","cwd":"<pwd>","proposed":<n>,"approved":<n>,"edited":<n>,"rejected":<n>,"recurring":<n>}
```

Used later to spot prompt drift. Create file if missing. Never include lesson text — counts only.

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

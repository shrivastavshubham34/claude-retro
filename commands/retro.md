---
description: Session retrospective. Extract lessons from mistakes/corrections, preview, approve, append to LESSONS.md.
---

You are running a session retrospective. Your job: find genuine mistakes from THIS session, propose lessons, get user approval, then write.

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

## 2. For each signal, draft a lesson

Format per lesson:
```
[SCOPE] <general | project>
[EVIDENCE]
> "<quote from user or tool result>"
[LESSON] When <situation>, do <X> not <Y>.
[FILE] <target path>
```

Scope rules:
- `general` → applies regardless of repo. Target: `~/.claude/LESSONS.md`
- `project` → repo/stack-specific. Target: `<project>/docs/LESSONS.md`

Resolve project path: use `$CLAUDE_PROJECT_DIR` if set, else current working directory. If `docs/` does not exist in project, note this — ask user before creating.

## 3. Show preview

Print all drafted lessons in a numbered list. For each:
- Show the EVIDENCE quote
- Show the LESSON text
- Show the target FILE
- Show whether it duplicates an existing entry in that file (read the file first; skip if too similar)

If zero lessons: say "no lessons worth recording" and stop. Do not pad.

## 4. Ask user

Ask one question with options per lesson:
- approve all
- approve subset (user lists numbers)
- edit one (user gives new text)
- skip all

Do NOT write files until user explicitly approves.

## 5. Write

On approval, for each approved lesson:
1. Read target file (create with header `# Lessons\n` if missing — but ask first if creating new file)
2. Append under appropriate section (see below)
3. Confirm write with file path + line range

# File format

Each LESSONS.md uses this structure:

```
# Lessons

## <YYYY-MM-DD>

- **<short title>** — <lesson body>. Evidence: <one-line quote>.
```

Group by date heading. New date = new heading. Same date = append under existing heading.

# Edge cases

- If `~/.claude/LESSONS.md` does not exist, create it (no prompt needed, it's the standard location).
- If `<project>/docs/LESSONS.md` does not exist AND `docs/` exists → create the file silently.
- If `<project>/docs/` does not exist → ask user before creating `docs/`.
- Never write to a file outside `$HOME/.claude/` or the current project directory.

# Output discipline

Keep preview compact. One lesson = ~5 lines. No prose padding around them. If user is in caveman mode, match that tone.

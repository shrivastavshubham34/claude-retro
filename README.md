# claude-retro

Manual session retrospective for [Claude Code](https://claude.com/claude-code).

After a coding session, run `/retro` to make Claude scan the conversation, extract genuine mistakes (backed by quoted evidence), preview the lessons, and — only after you approve — append them to a growing `LESSONS.md` knowledge base.

## Why

LLM coding agents repeat the same wrong turns across sessions. A short note in `LESSONS.md` injected back via `CLAUDE.md` is the cheapest way to stop the bleeding. This plugin makes capturing those notes a one-command habit.

## What it captures

Only what actually happened in the session:

- **User corrections** ("no", "actually do X instead", interrupted tool calls)
- **Wrong turns** (approach A swapped for B once A was disproven)
- **Failed assumptions** (file not where expected, tool lacked the action assumed)
- **Repeated errors** (same fix attempted twice)

Anti-hallucination rule: each lesson must quote the specific user pushback or wrong tool result. No quote → no lesson. Zero lessons is a valid output.

## claude-mem integration (optional)

If [claude-mem](https://github.com/thedotmack/claude-mem) MCP tools are available in your environment, `/retro` will:

- **Detect recurring mistakes** — search past observations for each mistake signal. If a similar pattern existed across previous sessions, the lesson is flagged `RECURRING` so you can prioritize it.
- **Dedupe across sessions** — search observations for the lesson title and skip drafts already covered by stored memory.
- **Sync approved lessons** — write each approved lesson back as an observation tagged `retro` so future sessions auto-inject it.

If claude-mem is not installed, this step is skipped silently — the rest of `/retro` works the same.

**Known runtime constraint**: writing lessons back as observations (`observation_add` / `memory_add`) requires claude-mem in `server-beta` runtime. In the default worker runtime, write tools are gated and return a capability error. `/retro` handles this gracefully — search/dedupe still works, only the cross-session write is skipped with a one-line note. To enable writes, set:

```bash
export CLAUDE_MEM_RUNTIME=server-beta
```

before launching Claude Code (or in your shell rc).

## Where lessons go

| Scope | File |
| --- | --- |
| `general` (applies to any repo) | `~/.claude/LESSONS.md` |
| `project` (repo/stack-specific) | `$PROJECT/docs/LESSONS.md` |

Claude decides the scope per lesson at preview time. Each lesson is grouped under a `## YYYY-MM-DD` heading and shows a one-line evidence quote.

## Install

### Option A: as a Claude Code plugin

```
/plugin install claude-retro
```

(once the plugin is published to a marketplace you have added)

### Option B: drop the command in by hand

```bash
git clone <this-repo> ~/code/claude-retro
mkdir -p ~/.claude/commands
ln -s ~/code/claude-retro/commands/retro.md ~/.claude/commands/retro.md
```

Then in any Claude Code session, type `/retro`.

## Flow

1. You type `/retro`
2. Claude scans the current conversation for mistake signals
3. For each signal, it drafts a lesson with: scope, evidence quote, lesson text, target file, duplicate check
4. Claude prints the proposed lessons and asks for approval (all / subset / edit / skip)
5. Only on explicit approval does it append to the target files

Files outside `~/.claude/` or the current project directory are never touched.

## Lesson format

Each `LESSONS.md` follows:

```
# Lessons

## 2026-05-22

- **short title** — lesson body. Evidence: "one-line quote from session".
```

Group by date. Same date = append under existing heading.

## License

MIT. See [LICENSE](LICENSE).

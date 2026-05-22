# CLAUDE.md — claude-retro

Handoff notes for any future Claude Code session working on this repo. Read this first.

## What this repo is

`claude-retro` is a Claude Code plugin that adds a `/retro` slash command. After a coding session, you run `/retro` and it:

1. Scans the current conversation for genuine mistakes (user corrections, wrong turns, failed assumptions, repeated errors, time waste).
2. (Optional) Searches claude-mem observations to detect recurring patterns across past sessions.
3. Drafts lessons with literal evidence quotes (anti-hallucination rule: no quote → no lesson).
4. Previews lessons with scope (`general` → `~/.claude/LESSONS.md`, `project` → `$PROJECT/docs/LESSONS.md`), dedupe markers, and RECURRING/NEW flags.
5. Waits for explicit approval (`all` / `subset N,M` / `edit N: <text>` / `skip`).
6. Writes only what's approved, then logs counts to `~/.claude/retro-log.jsonl`.
7. If claude-mem write tools available, also stores each lesson as a cross-session observation.

The point: build a knowledge base that gets smarter every session, without polluting it with hallucinated "best practices."

## Current state

- **Version**: v0.3 (commit log shows the evolution)
- **Plugin manifest**: `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json`
- **Slash command body**: `commands/retro.md`
- **Live install**: `~/.claude/commands/retro.md` is a **symlink** to `commands/retro.md` in this repo. Editing the file in this repo = the command updates instantly. No reinstall needed.
- **Remote**: `git@github-personal:shrivastavshubham34/claude-retro.git` (uses `github-personal` SSH host alias pointing at `~/.ssh/github-personal`)
- **Local git identity**: per-repo override — `Shubham Shrivastava <shrivastavshubham34@gmail.com>`. Office identity stays in `--global`, this repo overrides via `git config user.email`.

## File layout

```
.
├── .claude-plugin/
│   ├── plugin.json          # Plugin manifest
│   └── marketplace.json     # Marketplace listing
├── commands/
│   └── retro.md             # Slash command body — this is the actual prompt
├── CLAUDE.md                # This file
├── README.md                # Public-facing docs
├── LICENSE                  # MIT
└── .gitignore
```

## First real-world run (2026-05-22, dev-platform-ci)

`/retro` was first dogfooded in a `dev-platform-ci` session. Findings folded back into v0.3:

- Anti-hallucination held: every lesson had a literal quote.
- Multi-quote evidence was joined with `+`. **Fixed in v0.3**: one `EVIDENCE` line per quote, each with a source label.
- One lesson had no direct user quote — only a tool result + Claude's self-correction. **Fixed in v0.3**: `EVIDENCE` lines now require an explicit source label: `(user said)`, `(tool result)`, or `(self-correction)`. The last cannot stand alone — must pair with a user/tool quote.
- claude-mem read tools worked. Cross-session search returned zero matches → all NEW.
- claude-mem write tools (`observation_add` / `memory_add`) **failed** with a runtime/capability error: they require `CLAUDE_MEM_RUNTIME=server-beta`, current was worker. **Fixed in v0.3**: `/retro` now prints a one-line skip note and continues — does not abort.
- Earlier in the session a claude-mem call had been denied by the user. `/retro` then silently skipped all further claude-mem use. **Fixed in v0.3**: at step 2 of `/retro`, read-only searches re-try once even if denied earlier — running `/retro` is a fresh explicit intent.
- `~/.claude/retro-log.jsonl` got its first line: `{proposed: 4, approved: 3, rejected: 1, recurring: 0}`. Working.

## TODOs / next steps

In rough priority order:

1. **Enable claude-mem writes.** Add `export CLAUDE_MEM_RUNTIME=server-beta` to `~/.zshrc`. Restart shell. Next `/retro` should successfully call `observation_add` for each approved lesson. Verify cross-session injection actually works on the session after that.
2. **Dogfood more.** Run `/retro` at the end of 3-5 real sessions across different repos. Watch for:
   - False positives (lessons that shouldn't have been proposed)
   - Missed real mistakes (under-detection)
   - Wrong scope tags (`project` lesson tagged `general` or vice versa)
   - Slow runs (5+ claude-mem searches eating time)
3. **Watch `retro-log.jsonl`.** After ~10 runs, eyeball it. If `rejected` rate climbs, prompt is drifting — investigate which lesson type gets rejected most.
4. **Add `/retro --dry-run`** (or equivalent). For debugging the prompt without ever writing. Currently the only way to dry-run is to invoke and pick `skip all`.
5. **Add a `/retro-show` command** that just `cat`s `~/.claude/LESSONS.md` + `$PROJECT/docs/LESSONS.md` so you can quickly review accumulated lessons.
6. **Periodic compaction.** When `LESSONS.md` gets long (say >50 entries), need a `/retro-compact` command that asks Claude to merge duplicates and drop one-offs. Not urgent until volume justifies.
7. **Publish to a marketplace.** Currently install is manual (`git clone` + symlink). For wider use, push the marketplace listing. Look at how `caveman` and `claude-mem` did it (see `~/.claude/plugins/marketplaces/`).

## Useful commands

```bash
# verify symlink is intact
ls -la ~/.claude/commands/retro.md
# expect: -> /Users/shubham-heady/personal/repos/claude-retro/commands/retro.md

# tail the decision log
tail -f ~/.claude/retro-log.jsonl

# inspect global lessons
cat ~/.claude/LESSONS.md

# inspect this repo's git config
git -C ~/personal/repos/claude-retro config --local --list | grep user

# claude-mem runtime check
echo "CLAUDE_MEM_RUNTIME=$CLAUDE_MEM_RUNTIME"
```

## Anti-hallucination invariant

This is the single most important rule of the whole tool. Every `EVIDENCE` line in a draft lesson must include either:

- `(user said): "<literal user message>"`, or
- `(tool result): "<literal tool output>"`.

`(self-correction)` lines may accompany them but cannot stand alone. If a draft lesson cannot satisfy this, drop it — outputting zero lessons is a valid run.

Any prompt change that weakens this rule should be rejected. If a future iteration wants to add inferred lessons, gate them behind an explicit user opt-in flag.

## Tone

Repo author works in caveman mode (`/caveman` plugin). When responding inside sessions, match that tone for chat/preview output. **Do not** use caveman style in committed files — code, commits, docs, CLAUDE.md, README all stay in normal English.

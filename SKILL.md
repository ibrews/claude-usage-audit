---
name: usage-audit
description: Reflect on past Claude Code sessions to find the highest-leverage improvements to your setup — mine daily logs/notes, local session transcripts, claude-mem observations (if installed), and your live skill/hook/automation inventory with cheap-model subagents, then cluster findings into new-skills / new-automations / small-config-fixes with per-item evidence. Use when asked to "audit our Claude usage", "run a usage audit", "find skill gaps", "what should we automate", "reflect on past sessions", or for a scheduled monthly run.
license: MIT
---

# Usage Audit

A skill for the "reflect on past sessions" pattern: instead of guessing what
skills or automations you need, mine the sessions you've already run for
repeated manual work, recurring friction, and workflows with a stable enough
shape to script.

The output is **proposals with evidence**, not implementations. Nothing gets
built during the audit run — a human reviews and approves first.

This audits *usage patterns* (how sessions run, what gets redone by hand,
what's missing) — not Claude's product feature set, and not your project's
content/documentation quality. Keep those as separate audits if you run them.

## Model routing — miners are cheap, clustering is not

Run the four miners below as **parallel subagents on cheap models** (Haiku or
Sonnet, or whatever your setup's cheapest capable tier is). Each miner returns
structured findings — counts, dates, session/observation IDs, one-line
examples — never raw dumps. Only the final clustering pass runs on the
session's frontier model. This is what keeps a full audit affordable, and is
worth doing even without a usage cap.

## Setup — define the window first

Pick the audit window (default: the previous 3–4 weeks) and state it at the
top of the report. Every miner filters to this window. Every frequency claim
gets date-bucketed inside it before ranking (see the clustering rules — this
is the single biggest lesson from real use, below).

## Miner 1 — logs and notes, if you keep them

If you or your team maintain any kind of written log (daily notes, a work
journal, a CHANGELOG, retro notes, a Slack digest exported to markdown),
read every entry in the window. Look for: the same task appearing on many
different days, manual runs of things that sound like scripts, re-diagnosis
of the same failure more than once, "finally fixed X" entries (they
date-bound earlier friction). Return per pattern: description, dates seen,
representative quotes (≤1 line each).

Skip this miner entirely if no such log exists — don't invent one just to
run the audit.

## Miner 2 — local session transcripts (token-efficiency rules)

Transcripts live under `~/.claude/projects/<project-dir>/*.jsonl` — **on the
machine running the audit only**. If you use Claude Code on more than one
machine, transcript mining only covers this one; say so in the scope notes.

Hard rules for this miner:
- **Never cat or Read a whole transcript file** — they run to tens of MB.
  Aggregate with `jq`/`grep`/`awk`; only read the aggregates.
- Counts before content. `head`-cap every extraction.
- Typed user prompts are `type=="user"` records whose `.message.content` is a
  **string**; tool-result records are arrays (jq's `strings` filter drops
  those for free).

Census, then interactive-vs-automation split (a session with zero typed
prompts is a scheduled/automated run, not human-driven work):

```bash
# How many sessions in the window, and where?
find ~/.claude/projects -name '*.jsonl' -mtime -30 | wc -l
find ~/.claude/projects -name '*.jsonl' -mtime -30 \
  | sed 's|.*/projects/||; s|/[^/]*$||' | sort | uniq -c | sort -rn | head -20

find ~/.claude/projects -name '*.jsonl' -mtime -30 -print0 \
  | while IFS= read -r -d '' f; do
      n=$(jq -r 'select(.type=="user") | .message.content | strings' "$f" \
            | grep -c -v -e '^<' -e '^Caveat:')
      echo "$n $f"
    done | awk '$1 == 0 {auto++} $1 > 0 {inter++}
                END {print inter " interactive / " auto " automation"}'
```

Mine the interactive slice for repeated prompt shapes, and the whole set for
repeated manual commands:

```bash
# Typed prompts, capped — cluster these by shape (what gets re-typed?)
find ~/.claude/projects -name '*.jsonl' -mtime -30 -print0 \
  | while IFS= read -r -d '' f; do
      jq -r 'select(.type=="user") | .message.content | strings' "$f" \
        | grep -v -e '^<' -e '^Caveat:' | head -5
    done > /tmp/usage-audit-prompts.txt
wc -l /tmp/usage-audit-prompts.txt

# Command census: what gets run by hand over and over?
find ~/.claude/projects -name '*.jsonl' -mtime -30 -print0 \
  | xargs -0 cat \
  | jq -r 'select(.type=="assistant") | .message.content[]?
           | select(.type=="tool_use" and .name=="Bash") | .input.command' \
  | awk '{print $1, $2}' | sort | uniq -c | sort -rn | head -30
```

High counts of service-restart commands, hand-written heredoc helpers, or
repeated build/restart sequences all mean the same thing: script-shaped work
running as interactive sessions.

## Miner 3 — claude-mem observation timeline (optional)

If [claude-mem](https://github.com/thedotmack/claude-mem) or an equivalent
persistent-memory plugin is installed, use its search/timeline tools over the
window. Cluster observations into repeated session shapes ("same job
re-diagnosed", "same doc found stale twice"). Cite observation IDs — they're
a citable evidence layer that raw logs lack.

Skip this miner if you don't have persistent cross-session memory installed.

## Miner 4 — live inventory

Snapshot what already exists, so the clustering pass doesn't propose
something that's already there:

```bash
ls ~/.claude/skills/
ls ~/.claude/scheduled-tasks/ 2>/dev/null
ls ~/.claude/commands/ 2>/dev/null || echo "no custom slash commands"
crontab -l 2>/dev/null
ls ~/Library/LaunchAgents/ 2>/dev/null   # macOS; adapt for your OS's scheduler
jq '.hooks | keys' ~/.claude/settings.json 2>/dev/null
```

Also check inventory *health*, not just presence: skills that never trigger,
cron jobs whose output is stale, scheduled jobs pointing at paths that no
longer exist. These are exactly the failures that go unnoticed for weeks
because nothing else is watching them.

## Clustering pass (frontier model, this session)

Merge the four miners' findings into signals, then bucket:

- **Batch 1 — new skills**: workflows re-improvised across many sessions
  that would benefit from being frozen into a skill.
- **Batch 2 — automations**: new scheduled jobs, or existing ones upgraded
  (cadence, escalation, freshness checks). General principle: prefer a
  deterministic script or scheduled job over a recurring AI session wherever
  the check is mechanical — reserve frontier-model sessions for judgment
  calls the script can't make.
- **Batch 3 — small fixes**: config-file or one-line fixes (a stale path, a
  hook that fires on the wrong trigger, a doc that contradicts the code).
- **Considered, no action proposed**: name what you rejected and why — it
  saves the next audit from re-deriving the same rejection.

Per-item requirements:

1. **Evidence** — session counts, observation IDs, dates. No evidence, no
   item.
2. **Leverage** = (sessions or minutes affected per month) × (confidence the
   fix removes the cost). Rank HIGH / MEDIUM / LOW within each batch.
3. **Check before calling anything "new"** — grep your own docs/scripts
   directories and the Miner-4 inventory (and a semantic search tool over
   your docs, if you have one) before declaring a skill, doc, or script
   missing. Record the check in the item ("checked: only scattered log
   traces exist, no script").
4. **Date-bucket frequency evidence before ranking.** Bucket every frequency
   claim by week inside the window. If occurrences collapse after a
   mid-window date, the problem was probably already fixed — check version
   control history around the cliff date before ranking it as live. This
   showed up in the very first real run of this skill: a finding was ranked
   #1 on total-count evidence, but nearly all of the flagged activity
   predated a fix that had already landed mid-window. Check it like this:

```bash
# Weekly histogram of commits touching a suspect path — a cliff means "fixed"
SUSPECT_PATH=path/to/suspect/file
git log --since="30 days ago" --format=%ad --date=format:'%Y-w%V' \
  -- "$SUSPECT_PATH" | sort | uniq -c
```

## Output contract

Write a dated report (e.g. `usage-audit-YYYY-MM.md`, wherever your project
keeps this kind of note — a `docs/` folder, a wiki, or the repo root).
Sections, in order: **Method and scope** (window, miner coverage, what was
out of reach) → **Headline findings** (≤5) → **Batch 1 / Batch 2 / Batch 3**
→ **Considered, no action proposed** → **Suggested review order**.

Follow-through:
- Commit per your repo's normal conventions.
- If you have somewhere you route items needing a human decision (an inbox
  file, a task tracker, a chat channel), put a pointer there.
- Do not implement findings in the audit session itself — wait for approval.

## Scope notes

- An always-on or automation-heavy machine's transcripts will skew almost
  entirely toward scheduled jobs, not human workflow — that's a legitimate
  signal about *that* machine, but it isn't a substitute for auditing the
  machine you actually work from interactively. If you use more than one
  machine, run this once per machine and compare.
- Transcript mining (Miner 2) only sees the local machine. Logs (Miner 1)
  and shared memory (Miner 3) are your only cross-machine signal.
- If you've run this skill before, read the most recent past report first
  and don't re-report items already marked done in it.

---
name: waypoint
description: "Write the current session's state to a uniquely-named waypoint doc in ~/.claude/waypoints/ (not the project directory) and produce a paste-in prompt for a fresh session, then recommend ending this one. Use when the user runs /waypoint, says the session feels slow/heavy/full of context, asks to save progress and continue later or in a new session, or when you notice this session has grown very long (many turns, many subagent round-trips, heavy tool output) in the middle of a multi-step task. This does not end the session itself — no tool can do that — it produces the doc and the handoff prompt and tells the user to open a new one."
---

# /waypoint

Formalizes the "checkpoint progress into a doc, then start a fresh session" pattern from the
user's global CLAUDE.md ("Long-running / open-ended tasks"). Exists because in a long session
every turn re-sends the full accumulated transcript as a cache read — cost grows with turn
count and transcript size, not with how long tool calls actually took to run. A multi-hour build
under `babysit-run` costs almost nothing while it runs; 200 turns of an orchestrating session
re-paying its own history each turn is what actually burns tokens. This skill does not fix that
by itself (nothing can end or replace the current session from inside it) — it makes the
handoff to a new session cheap and complete instead of an ad hoc summary written under time
pressure.

**This is a synchronous, foreground skill.** Do the steps below, print the result, then stop —
do not use `ScheduleWakeup`, `/loop`, or otherwise keep this session alive waiting on anything.
Staying alive is exactly the cost this skill exists to end.

**A waypoint doc is orchestration state, not a project artifact.** It documents *this session's*
handoff, not the project itself — it doesn't belong in the repo, shouldn't be committed, and
shouldn't compete for a filename like `PROGRESS.md` that a project might legitimately want to
use for its own purposes. Treat it the way you'd treat a shell history or a scratch buffer: live
outside the project tree, uniquely named, disposable.

## 1. Gather state (don't ask the user to restate what you already know)

- `git status` and `git diff --stat` (uncommitted changes, untracked files worth keeping vs.
  known scratch/pipeline dirs that don't need to move to the doc).
- The plan/task doc this session has been executing, if one exists (`*_PLAN.md`,
  `ROUNDN_PLAN.md`, an issue, or whatever the user originally handed you) — the waypoint
  doc points at it rather than re-deriving its content.
- What's actually done vs. still pending, per task/step, with enough per-task detail that a
  fresh session (or a fresh subagent it spawns) doesn't need to re-read this whole
  conversation to know *why* something was done a particular way — flag anything surprising,
  any deviation from the plan, any deferred/skipped item, any number or finding that would
  otherwise only exist in this transcript.
- Any local state that outlives the session: files on disk not in git (build artifacts, data
  dumps, fixtures), background pipelines under `babysit-run`/`babysit-attach` (name their
  rundirs — a fresh session's `SessionStart` hook will surface these automatically if they're
  under its cwd, so don't re-describe their contents, just name them), binaries that need a
  rebuild-before-trust note if edits landed after the last build.
- Constraints that must survive into the new session and are easy to silently drop: things the
  user said this session that aren't yet written down anywhere durable (not already in
  CLAUDE.md or memory), scope boundaries ("don't touch chain.c"), model assignments per
  remaining task, anything explicitly not-yet-approved (e.g. "report back before committing").
- The canonical absolute path of the project directory this session is working in (`pwd -P`,
  i.e. with symlinks resolved) and its path hash (see step 2) — needed for the doc's filename
  and identity header, and for `/waypoint-continue` to find it later. The basename alone is not
  enough: different projects can share a folder name.

## 2. Write the waypoint doc

**Location:** `~/.claude/waypoints/`, never the project directory. This is a hard rule, not a
convention to match against project files — waypoints are orchestration state that lives
alongside Claude Code's own config/hooks, not project output.

**Filename:** unique per waypoint, so nothing ever collides and multiple waypoints for the same
project coexist cleanly:

```
<project-slug>-<path-hash>_<YYYYMMDD-HHMMSS>.md
```

- `<project-slug>`: the project directory's basename, lowercased, non-alphanumerics replaced
  with `-` (e.g. `/home/ciccolella/checkpoint-skill` → `checkpoint-skill`). Human-readable
  only — never treat it as an identifier, since two projects in different places can share a
  basename (`~/work/pipeline` and `/data/proj/pipeline`).
- `<path-hash>`: first 8 hex chars of the SHA-256 of the project's canonical absolute path
  (symlinks resolved, no trailing slash). This is what disambiguates same-named directories.
  Compute it with exactly this command, run from the project directory, so
  `/waypoint-continue` reproduces the same value:

  ```bash
  printf '%s' "$(pwd -P)" | { sha256sum 2>/dev/null || shasum -a 256; } | cut -c1-8
  ```
- `<YYYYMMDD-HHMMSS>`: this session's local timestamp — doubles as a uniqueness guarantee (two
  waypoints in the same project in the same second are not a real scenario) and lets the user
  eyeball recency without opening the file.

Example: `~/.claude/waypoints/checkpoint-skill-3f9a1c2e_20260921-181600.md`.

**Content structure** — start with an identity header the `waypoint-continue` skill relies on
to match a waypoint back to the right project, then the same content shape as before:

```markdown
---
project_dir: <canonical absolute path — output of `pwd -P`, the same string that was hashed>
created: <ISO 8601 timestamp>
---

# <Title> — waypoint (<date>)

<One or two lines: what this is executing and where the handoff should resume reading.>

## Original instruction (paste this into the new session, or just point it at this file)

> <the kickoff instruction, verbatim, if there was one>

## Status: <one line — e.g. "Phase 2 of 4 done, nothing committed yet">

### What's done
<Per-task/step bullets, with the "why", the actual results/numbers, and any deviation from
plan — this is the part a fresh session cannot recover any other way.>

### Files modified (uncommitted)
<git status output, annotated — flag anything that's pre-existing/unrelated so a fresh session
doesn't touch or commit it by mistake.>

### Data artifacts / background pipelines
<Paths on disk, rundir names — brief, since these are independently discoverable.>

## Next steps (in order)
<Concrete, ordered, resumable without re-reading this conversation.>

## Reminders carried forward (not yet in CLAUDE.md/memory)
<Only what would otherwise be lost — don't restate CLAUDE.md's own rules.>
```

If a project's own plan/report doc convention exists (e.g. `DBSN_PLAN.md` / `DBSN_REPORT.md`),
that convention still governs *those* files if you're asked to write one — it does not change
where the waypoint doc itself goes. The waypoint is never the project's own progress-tracking
artifact; if the project wants one of those, that's a separate, project-tracked file the user
asks for explicitly.

## 3. Print the handoff

Give the user, directly in chat (not just in the file):

- Confirmation of what was written and where (`~/.claude/waypoints/<filename>`).
- The **exact prompt to paste into a new session**. If the new session will open in this same
  project directory (the normal case — a new terminal/`claude` invocation in the same repo),
  just `/waypoint-continue` with no argument is enough: it auto-detects the right waypoint by
  matching the new session's cwd against the doc's `project_dir:` header. Only give an explicit
  path — `/waypoint-continue <full-path-to-waypoint-doc>` — if the new session might open
  somewhere else (a different directory, or if several waypoints exist for this project and a
  specific one, not the most recent, is the one to resume).
- A one-line reminder that starting the new session is a manual step (open a new terminal /
  `claude` invocation) — nothing here can do that automatically.

## 4. Stop

End the turn after the summary. Don't keep working in this session past a waypoint unless the
user explicitly says to continue here instead of switching.

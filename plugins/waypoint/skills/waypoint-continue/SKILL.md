---
name: waypoint-continue
description: "Auto-detect and read the right waypoint doc from ~/.claude/waypoints/ (written by the /waypoint skill) for the current directory, plus its referenced plan, and resume the work directly in this fresh session — no path argument needed. Use when the user runs /waypoint-continue, says to pick up/continue/resume from a waypoint, or opens a new session pointing at a prior one's handoff doc."
---

# /waypoint-continue

Counterpart to the `waypoint` skill. `/waypoint` writes a handoff doc and a paste-in prompt
because nothing can open a new session automatically; `/waypoint-continue` is what removes the
*second* manual step — once a fresh session exists, don't make the user retype or re-explain
anything the waypoint doc already captured. Read it and continue from "Next steps" directly.

Waypoint docs live in `~/.claude/waypoints/`, not the project directory — they're orchestration
state, not project output. This skill finds them there, not in the current project's file tree.

## 1. Find the waypoint doc — auto-detect by default, no path needed

The default path is zero-argument: `/waypoint-continue` alone should just work.

- If the user gave an explicit path or filename (as `/waypoint-continue <path>` args, or
  named it in their message), use that instead of auto-detecting — an explicit pointer always
  wins.
- Otherwise, match on the **full canonical path**, never on the folder name. The filename's
  leading slug is just the basename and can be shared by unrelated projects in different
  places (`~/work/pipeline` vs `/data/proj/pipeline`) — a waypoint whose filename starts with
  this directory's name is *not* evidence it belongs here.
  1. Compute this directory's canonical path and path hash, with the same command `/waypoint`
     uses:

     ```bash
     pwd -P
     printf '%s' "$(pwd -P)" | { sha256sum 2>/dev/null || shasum -a 256; } | cut -c1-8
     ```
  2. Candidates are `~/.claude/waypoints/*-<hash>_*.md`, plus any legacy waypoints written
     before the hash was added (filename is `<slug>_<timestamp>.md`, no `-<8 hex>` before the
     underscore).
  3. For every candidate, read its `project_dir:` header and keep only those equal to the
     canonical path from (1) (ignore a trailing slash; also accept a header equal to the
     logical `$PWD`, since legacy waypoints may have recorded the unresolved path). The header
     is authoritative — the hash only narrows the search. Discard anything that doesn't match,
     even if the slug or hash looks right.
  - **Exactly one match**: use it, no confirmation needed — say which file was picked (filename
    is enough, e.g. "resuming from `checkpoint-skill-3f9a1c2e_20260921-181600.md`") as part of
    the normal step-3 summary, not as a question.
  - **Multiple matches**: pick the most recent one by the timestamp encoded in the filename and
    proceed the same way, but explicitly name it and mention the others exist (e.g. "resuming
    from the most recent waypoint for this directory (`..._181600.md`); N older ones for this
    project also exist") so the user can redirect if the latest isn't actually the one they
    meant — this is a heads-up, not a blocking question.
  - **No matches for this cwd**: only then fall back to asking — list the most recent few
    waypoints regardless of project (the user may be resuming into a moved/renamed directory)
    and ask which one, since there's no signal left to auto-resolve on. Show each one's
    `project_dir:` next to its filename — same-named projects are indistinguishable by
    filename alone. Never auto-pick one just because its slug equals this directory's name.

## 2. Read, in order

1. The waypoint doc itself, in full.
2. Whatever plan/task doc it points at (its "Original instruction" section and any referenced
   `*_PLAN.md`/issue/etc.), in full — the waypoint doc is a pointer plus delta, not a
   replacement for the plan.
3. `git status` / `git diff --stat` if the doc's "Files modified" section suggests uncommitted
   work, to confirm the working tree still matches what the doc describes (it may not — time may
   have passed, or someone else touched the repo). Flag any mismatch to the user before acting on
   stale assumptions rather than silently trusting the doc over what's actually on disk.

## 3. Resume

- Give a brief confirmation of what was read and the current status (one or two lines — the doc
  already has the detail, don't re-narrate it back in full).
- Carry forward anything under "Reminders carried forward" and "Constraints" as live constraints
  for this session, the same as if the user had just said them.
- Proceed directly into the first item under "Next steps" — don't ask the user to confirm
  "should I continue?" as a matter of course; resuming a named waypoint is the confirmation.
  Do still stop and ask if the next step is itself something that normally warrants a check-in
  (a destructive action, a decision the doc flags as not-yet-approved, genuine ambiguity about
  which of several next steps to take first).
- If background pipelines were named in the doc, check whether they're still running
  (`babysit-status`) rather than assuming their state.

## 4. Don't re-run /waypoint's own gathering steps

This skill reads state, it doesn't reconstruct it. If the doc is thin or clearly stale, say so
and ask the user how to proceed rather than trying to reverse-engineer missing context from the
repo.

## 5. Clean up once resumed — a waypoint is single-use

It's disposable orchestration state, like the rest of `~/.claude/waypoints/`. Don't copy its
content into the project or otherwise promote it to a tracked artifact.

- After step 3's summary is printed and you've started on the first "Next steps" item (i.e. the
  resume clearly succeeded — the doc was readable, the project matched, nothing was too stale to
  act on), delete the waypoint doc you just used: `rm ~/.claude/waypoints/<filename>`. Do this
  as a small aside, not something to ask permission for each time — it's the expected end state
  of a successful resume, the same way `/waypoint` doesn't ask before writing one.
  - Exception: if step 2 flagged a real mismatch (working tree doesn't match the doc, doc looked
    stale/thin) and you asked the user how to proceed instead of resuming cleanly, don't delete —
    leave it until the situation is actually resolved.
- At the same time, sweep any *other* waypoints whose `project_dir:` header matches this
  project's canonical path (the same header check as step 1 — never select by filename slug,
  or you'll delete a same-named project's waypoint) and that are older than the one you just
  used — they're superseded (a newer waypoint for the same
  project means the older one's "Next steps" are either done or subsumed). List what you're
  removing in the same aside rather than deleting silently. Leave waypoints for *other* projects
  untouched — this skill only ever cleans up after itself for the project it was just invoked in.

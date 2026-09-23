# waypoint

A Claude Code plugin for checkpointing a long session and resuming it cleanly, elsewhere.

Every turn in a long session re-sends the full accumulated transcript as a cache read — cost
grows with turn count and transcript size, not with how long tool calls actually took. Instead
of letting a session grow unbounded (or writing an ad hoc summary under time pressure when it
finally gets too slow), `/waypoint` writes a structured handoff doc and a paste-in prompt, and
`/waypoint-continue` picks it back up in a fresh session with no re-explaining required.

## Install

```
/plugin marketplace add sciccolella/claude-waypoint-skill
/plugin install waypoint@waypoint
```

## What's in it

Two skills, no scripts or hooks — pure prompt-driven workflow:

- **`waypoint`** (or `/wp`) — writes the current session's state (what's done, what's pending,
  files touched, background pipelines, constraints not yet written down anywhere durable) to a
  uniquely-named doc in `~/.claude/waypoints/` (named by folder plus a hash of its full path, so
  same-named folders in different places never collide), then prints the exact prompt to paste into a
  new session and recommends ending this one.
- **`waypoint-continue`** (or `/wpc`) — in a fresh session, auto-detects the right waypoint doc
  for the current directory (no path argument needed), reads it plus whatever plan it points at,
  resumes directly from "Next steps", and cleans up the doc once the resume has succeeded.

`/wp` and `/wpc` are plain aliases — they just forward to the skills above, nothing more.

Waypoint docs live in `~/.claude/waypoints/`, not the project directory — they're disposable
orchestration state, not project output, and are never committed to a project's own repo.

## License

MIT — see [LICENSE](LICENSE).

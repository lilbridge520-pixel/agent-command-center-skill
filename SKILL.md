---
name: agent-command-center
description: Build a live 3D office dashboard ("Agent Command Center") where the user watches their AI agents' real status and dispatches real work to them
---

# Agent Command Center

## Overview

An Agent Command Center is a 3D office you can open, look around in, and use to
run a small team of AI agents. Each agent has a workstation with a little figure
sitting at a desk. The board shows what each agent is really doing right now, and
the user can click an idle agent, type a task into an input box, and have that
agent actually do the work through a real channel — not a pretend status change.

This skill tells you how to build one. It is a product recipe, not a copy of any
one person's setup: you define your own agent roster, wire each agent to a real
execution channel, and point the status sync at your own task and schedule
sources.

**Three rules decide whether the result counts as a command center at all:**

1. **Status is real, never simulated.** Every status on the board must come from
   a real record (a dispatched task, a scheduled job, an in-progress item). If
   nothing is running, the board says so.
2. **Dispatch is real.** Submitting a task must start real work through a real
   channel, and the result must be written back to the board.
3. **Wording is honest.** Only a workstation that is actually executing a
   dispatched task may show "working". Anything else gets a different label.

## When to use

Use this skill when the user asks for:

- A 3D office / control room / mission-control view of their AI agents.
- A dashboard where they can see who is busy, who is idle, and assign work by
  clicking a person.
- A live version of an existing static agent dashboard.

Do not use it for a decorative 3D scene, a status page with made-up activity,
or a demo with sample data. If there is no real execution channel behind a
workstation, leave that workstation out.

## Architecture

Three parts. Keep them separate.

**1. 3D client (Three.js, runs in the browser)**

- Renders the office, the workstations, the figures, and the wall boards.
- Handles camera movement, workstation selection, and the dispatch input box.
- Polls the server for fresh board state (see "Status sync" below).
- Holds no business logic about what "working" means; it only renders what the
  server returns.

**2. Server actions + store**

- `getBoard` — returns the current board snapshot: agent states, in-progress
  items, scheduled jobs, dispatch records, activity log, last sync time.
- `refreshBoard` — triggers a background sync that reads the real task and
  schedule sources and writes the snapshot to the store.
- `dispatchTask` — accepts `{ assignee, taskText }`, creates a dispatch record
  in `executing` state, and hands the work to the executor.
- `saveDispatchResult` — called by the executor when work finishes; writes the
  result and flips the record to `done` or `failed`.
- Store tables (names are a suggestion, not a requirement): `agent_states`,
  `board_todos`, `board_jobs`, `dispatches`, `activity`, `sync_meta`.

**3. Background sync and executor**

- A sync job (triggered on open, on manual refresh, and on a daily schedule)
  reads the real sources and updates the snapshot.
- An executor job runs each dispatch through the assignee's real channel and
  calls `saveDispatchResult`. It must survive the web request that started it:
  dispatch is fire-and-forget from the page's point of view; the result arrives
  through the next poll.

## Agent roster

Define the roster with the user. Six workstations is a good size for one screen.
Each entry needs a role, a display name, a short description, and an execution
channel. Example roster — replace every row with the user's real agents:

| Workstation | Role | Example execution channel |
|---|---|---|
| Coordinator | Owns the overall plan, triage, and delegation | Direct execution by the main agent |
| Coder | Writes and edits code, runs commands | A coding CLI (non-interactive mode) |
| Writer / Director | Drafts scripts, outlines, and long-form text | A writing agent CLI or direct execution |
| Browser Runner | Logs in to sites, reads pages, fills forms, places orders | A browser-task worker |
| Researcher | Answers questions, checks facts, reads sources | A hosted chat session |
| Scheduler | Runs recurring work on a timetable | Creating real scheduled tasks |

Rules for the roster:

- Every workstation must map to a channel that really exists in the user's
  setup. A workstation with no channel is a decoration — remove it.
- Keep the roster small enough that all figures fit on screen at once.
- Give each agent a distinct colour and short label so the roster list, the 3D
  nameplates, and the dispatch records all refer to the same person.

## Status sync

### Sources

Point the sync at sources that already exist and already hold the truth:

- The user's tracked items / to-do list (in-progress items, due dates).
- The user's scheduled jobs / cron list (name, schedule, last run, next run).
- Dispatch records created by this board itself.

Do not invent a new source of truth for the board. The board is a view over
records that already exist.

### Cadence

- **On open:** run a sync so the first paint is real data.
- **On manual refresh:** run a sync.
- **Polling:** the page re-fetches board state about every 15 seconds.
- **Background:** a daily scheduled sync keeps the snapshot fresh even when
  nobody has the page open.

### Status states and honest wording

Use exactly these states. Do not add a "busy-looking" state.

| State | Meaning | Display |
|---|---|---|
| `idle` | No live dispatch and no in-progress items under this agent's name | Grey / "Idle" |
| `has-items` | Has in-progress items, but none of them is being executed right now | Yellow / "Has in-progress items (not executing right now)" |
| `executing` | A dispatch to this agent is currently running | Green / "Working" |
| `done` / `failed` | The last dispatch finished (shown on the dispatch record, not as a live status) | On the record only |

The wording matters. "Has in-progress items" must never be rendered as
"working", and a workstation must never show a progress bar for work that is
not running. In the reference build this was the single most-reported problem:
a simulated board showed an idle coding agent as "working", and the user
caught it immediately.

## Dispatch flow

The flow the user experiences:

1. Click a workstation (in the 3D scene or in the roster list).
2. A detail card opens with the agent's current real items and an input box.
3. The user types the task and submits.
4. The board shows the dispatch as `executing`, with elapsed time and step
   progress (see "Fail fast" below).
5. The executor runs the task through the agent's real channel.
6. The result is written back to the dispatch record; status becomes `done`
   or `failed`, and the record shows what actually happened.

### Execution channels

| Channel | How it works | Notes |
|---|---|---|
| Direct execution | The main agent (or a background subagent) performs the task | Best for research, writing, triage, anything the main agent can do with its own tools |
| Coding CLI | Invoke the agent's CLI in non-interactive mode with the task as the prompt, stdin closed | Pin the model if the agent supports it; pass credentials only through the platform's credential mechanism, never as visible text |
| Browser task | Start a browser-task worker with the target site and the steps | Use for logged-in sites, publishing, and ordering; the worker reports back what it actually did |
| Hosted chat session | Ask the question in the user's already-logged-in chat session | Subject to that session's rate limits; keep questions short and batched |
| Scheduled task | Create a real scheduled job (one-off or recurring) | Use for "remind me every morning" and other timetable work; a one-off job executes once, a recurring job repeats |

Rules:

- The executor must call the channel for real. Writing a plausible-looking
  result without running anything is fabrication — never do it.
- A dispatch that needs the user's own login, payment, or verification must
  stop at that step and say so; it must not be marked `done`.
- Never re-dispatch the same task in a tight loop to "see if it works". Each
  retry costs a full execution.

## Fail fast: probe the session before dispatching

This is the most important lesson from the reference build. A CLI-based
coding agent's login session had expired while its API token was still valid.
A dispatch ran the CLI twice, waited through token refresh and diagnosis, and
only failed after more than three minutes — with the reason hidden behind an
"expand" control. The user had to ask why a one-line question got no reply.

Build these three safeguards in from the start:

1. **Pre-dispatch probe.** Before running the real work, do one lightweight
   liveness check for the assignee's channel: for a CLI, a fast auth/session
   probe; for a browser worker, a session-alive check; for a hosted chat, a
   reachability check. If the session is dead, fail in seconds with the exact
   message "Not logged in — re-login required", and do not start the real work.
2. **Elapsed time and step progress.** While a dispatch runs, show how long it
   has been running and which step it is on (for example: "Dispatched /
   Attempt 1 / Running / Verifying"). The page polls, but a spinner with no
   timing tells the user nothing.
3. **Failure reason on the first line.** The dispatch record must show the
   failure reason and the required next step in its first lines, without
   requiring the user to expand anything. Long result text may be collapsed,
   but the conclusion must not be.

Also: a token stored in an environment variable is not the same as a CLI's
own login session. If the channel authenticates through its own saved session,
refreshing the token does not restore it — only a fresh interactive login
does. Record that distinction in the skill's channel notes so the next builder
does not "fix" a dead session by refreshing a token.

## The 3D office

Build it with Three.js (or an equivalent WebGL scene). Requirements:

- **Camera:** orbit (drag to rotate) and zoom (scroll / pinch). Provide four
  view buttons: Panorama, Boss view, Auto patrol, Rotate. Auto patrol moves
  the camera from workstation to workstation and highlights the matching
  roster entry and detail card.
- **Workstations:** one desk per agent, each with a monitor, a chair, and a
  figure. Clicking a workstation (or its roster entry) selects that agent and
  opens its detail card.
- **Figure orientation — get this right.** Every figure must sit facing its
  own screen, with its back to the default camera, like a person actually
  working. A figure facing the camera reads as a doll on display. If the
  workstation group is rotated, rotate the figure with it; verify the facing
  from the panorama view, the boss view, and the patrol camera — not from one
  angle only.
- **Wall dressings:** a task board, a completion-count screen, and a clock
  showing the real time. These are the only "live" decorations; everything
  else on the walls is static.
- **Nameplates:** each desk shows the agent's name and a status dot using the
  honest status colours.

## Mobile layout

The board must work as well on a phone as on a desktop. The reference build
failed this on the first pass: on a phone the 3D office was squeezed into a
thin strip and the user could not see what anything was doing.

- **Camera:** on portrait screens, widen the field of view and pull the camera
  back so all workstations fit in one frame. Do not just shrink the desktop
  layout.
- **Detail card:** hidden by default on narrow screens. Tapping a workstation
  or a roster entry opens it as a bottom sheet with a close button; closing it
  reveals the full office again.
- **Collapsible long sections:** long lists and long cards must be collapsible
  by default — truncate lists (for example first 6 items) with an "Expand all
  N items" button, give each long panel a collapse toggle in its header, and
  clamp long result text to a few lines with an "Expand full text" button.
  Apply this to every long region, including the workstation detail card.
- **No horizontal overflow:** the page itself must not scroll sideways; the
  roster may scroll horizontally inside its own container, but the page must
  not.
- **Polling and controls** work the same on mobile as on desktop.

## Adaptation guide

To use this skill for a different user, change exactly three things:

1. **Roster.** Ask the user who their agents are and what each one does.
   Replace the example table; keep the role/channel discipline.
2. **Channel mapping.** For each agent, name the real channel and how the
   executor invokes it (command, API, browser task, chat session, scheduler).
   If a channel does not exist yet, that workstation does not ship.
3. **Sync sources.** Point `refreshBoard` at the user's real tracked items and
   scheduled jobs. Keep the cadence (on open, on refresh, 15-second poll,
   daily background sync) unless the user asks for a different one.

Do not change the status states, the honest-wording rules, the fail-fast
probe, or the mobile requirements when adapting. Those are the skill, not the
decoration.

## Verification checklist

Run these before calling the board done. Check each one for real; a builder's
self-report is not verification.

- [ ] Open the board: the first paint shows real scheduled jobs and real
      in-progress items, not placeholders.
- [ ] An idle agent shows idle (or "has in-progress items"), never "working".
- [ ] Dispatch a trivial task to each channel type and confirm a real result
      is written back to the dispatch record (done or failed, with the actual
      output or error).
- [ ] Break one channel's login/session on purpose, dispatch to it, and
      confirm it fails in seconds with "Not logged in — re-login required"
      instead of waiting minutes.
- [ ] During a slow dispatch, the record shows elapsed time and step progress.
- [ ] A failed dispatch shows its reason on the first line, without expanding.
- [ ] On a phone viewport, the whole office fits on screen; the detail card
      is hidden until tapped and opens as a bottom sheet; every long section
      collapses.
- [ ] From panorama, boss view, and patrol, every figure faces its own
      screen.
- [ ] Refresh the page: dispatched history and statuses survive the reload.

## Anti-patterns

- **Simulated activity.** Randomly cycling agents through working / thinking /
  done states. The user will catch it, and it destroys trust in the whole
  board.
- **"Working" for backlog.** Showing a progress bar or a green dot for items
  that merely exist. Backlog is not execution.
- **Silent failure.** A dispatch that stays "executing" forever, or a failure
  whose reason is buried behind an expand control.
- **Retry loops.** Re-dispatching the same failing task repeatedly without
  fixing the cause. Each attempt costs a full execution.
- **Fake results.** Writing a plausible answer into the dispatch record
  without running the channel.
- **Secrets in code or logs.** Tokens, passwords, session keys, and account
  identifiers belong in the platform's credential store. Never embed them in
  the skill, the board's code, the dispatch records, or any log the user can
  open.

## Privacy note

This skill ships with generic example agents and tasks only. When building a
board for a real user, never copy their real task names, dates, order numbers,
account names, or personal details into the skill, the repo, or any shared
file. The board reads the user's live data at runtime; the skill itself stays
generic.

# Agent Command Center — Skill

A public, reusable skill for building a live **Agent Command Center**: a 3D
office dashboard where you watch your AI agents' real status and dispatch
real work to them.

## What it is

Open the board and you see a small 3D office. Each of your AI agents has a
workstation with a little figure sitting at a desk, facing its screen. The
board shows what each agent is really doing — and when someone is idle, you
click them, type a task into an input box, and they actually go and do it.

Three things make it a command center instead of a toy dashboard:

- **Real status, never simulated.** Statuses come from your real scheduled
  jobs and in-progress items. An idle agent shows idle; only a workstation
  that is actually executing a dispatched task shows "working".
- **Real dispatch.** Submitting a task runs it through a real channel — direct
  execution, a CLI, a browser task, a hosted chat session, or a new scheduled
  task — and the result is written back to the board.
- **Honest failure.** If an agent's login session is dead, the board says so
  in seconds ("Not logged in — re-login required") instead of making you wait
  minutes for a failure hidden behind an expand button.

It also works on a phone: the 3D office fits on one screen, detail cards open
as bottom sheets, and long sections collapse.

## How to use it

1. Copy `SKILL.md` into your skills directory (for example
   `~/workspace/custom-skills/agent-command-center/SKILL.md`).
2. Define your agent roster: who your agents are and what each one does.
   Replace the example roster — a coordinator, a coder, a writer/director, a
   browser runner, a researcher, and a scheduler.
3. Map each agent to a real execution channel (direct execution, a CLI, a
   browser task, a hosted chat session, or scheduled-task creation). A
   workstation with no real channel does not ship.
4. Point the status sync at your own task and schedule sources, then build the
   board as a web artifact (3D client + server actions + background sync).
5. Run the verification checklist in `SKILL.md` before calling it done —
   especially the fail-fast test (break a login session on purpose and confirm
   a dispatch fails in seconds, not minutes).

## Files

- `SKILL.md` — the full skill: architecture, roster, status sync, dispatch
  flow, fail-fast rules, 3D office and mobile requirements, adaptation guide,
  and verification checklist.
- `README.md` — this file.

## Notes

- The skill ships with generic example agents and tasks only. It contains no
  personal data, account names, tokens, or credentials.
- Secrets (tokens, passwords, session keys) belong in your platform's
  credential store, never in the skill, the board's code, or its logs.

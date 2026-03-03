# Tasks

## Autonomous Builder SOP

### Before starting any task
1. Read `CLAUDE.md` and `tasks.md` — know the current phase and what's in progress
2. Read `LEARNING.md` — check for gotchas and open bugs relevant to this task
3. Read `docs/DEBUG_CHECKLIST.md` — known issues that may affect your work
4. Run `npm run typecheck` — confirm TypeScript is clean before touching code

### After completing any task
1. Run `npm run typecheck && npm test` — both must pass before marking done
2. Mark task complete in `tasks.md` — add any discovered subtasks to the backlog
3. Update `LEARNING.md` — log anything surprising, any bug found, any decision made
4. If a bug was fixed: add an entry to the Decisions Log in `SPEC.md`

### If something breaks
1. Check `logs/nanoclaw.log` and `logs/nanoclaw.error.log`
2. Work through the relevant section of `docs/DEBUG_CHECKLIST.md`
3. Add a bug entry to `LEARNING.md` before attempting a fix — record what you observed, not just the fix

---

## Phase 1 — Autonomous Builder Foundation

_Goal: Claude Code can pick up any task in this repo, know where to start, what to verify, and where to log what it learns. Known bugs are tracked and either fixed or explicitly deferred._

- [ ] Create `LEARNING.md` — stub with sections: Bugs Found, Decisions, Gotchas, Open Questions
- [ ] Validate TypeScript build: `npm run typecheck` passes clean
- [ ] Validate test suite: `npm test` passes — note any failures
- [ ] Verify systemd service management works on this Linux host (launchd docs don't apply)
- [ ] Fix IDLE_TIMEOUT: set to 5min (300000ms) in `src/config.ts` so containers wind down gracefully between messages rather than always timing out via SIGKILL
- [ ] Fix cursor advancement: in `processGroupMessages` (`src/index.ts`), do not advance `lastAgentTimestamp` until agent confirms success — or roll back on container timeout, not just on agent error

_Phase 1 complete when: typecheck passes, tests pass, `LEARNING.md` exists, both known bugs are fixed or explicitly deferred with written rationale in `LEARNING.md`._

---

## Backlog

_(Add items here as they're discovered. Pull into a phase when prioritising.)_

- [ ] Verify container build works clean on this Linux host: `./container/build.sh`
- [ ] Confirm `.env` is present and auth tokens are valid
- [ ] Add systemd service file or document the manual start process for Linux

---

## Phase 2 — TBD

_(Define when Phase 1 is complete.)_

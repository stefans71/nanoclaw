# NanoClaw — Learning Log

_Append entries as you work. Never delete entries — mark them [FIXED] or [DEFERRED] if resolved._

---

## Bugs Found

### [FIXED] Resume branches from stale tree position
Session JSONL can have stale branch tips when subagents write to the same file. On subsequent `query()` resumes the CLI picks the wrong branch and the response lands on a branch the host never receives. **Fix:** pass `resumeSessionAt` with the last assistant message UUID to anchor each resume. See `docs/DEBUG_CHECKLIST.md` for diagnostic commands.

### [FIXED] IDLE_TIMEOUT == CONTAINER_TIMEOUT (both 30min)
Both timers fired at the same time, so containers always exited via hard SIGKILL (code 137) instead of graceful `_close` sentinel shutdown. Containers should wind down between messages (idle timeout short), while the hard limit stays long as a safety net. **Fix:** set `IDLE_TIMEOUT` to 5 min (300000 ms) in `src/config.ts`. `CONTAINER_TIMEOUT` stays at 30 min, so `Math.max(CONTAINER_TIMEOUT, IDLE_TIMEOUT + 30_000)` now resolves to 30 min (hard kill) with the graceful sentinel firing at 5 min of idle — well before the hard kill.

### [FIXED] WhatsApp connect() hung indefinitely on startup
If Baileys never emitted a `connection.update` event (e.g. bad auth state, network issue), `connect()` blocked forever — no timeout, no error, service appeared to start but never processed messages. **Fix:** wrap `connectInternal()` in a `Promise.race` with a 30-second timeout in `src/channels/whatsapp.ts`. If the timeout fires, the error message tells the user to check auth or delete `store/auth/` and re-authenticate.

### [FIXED] buildVolumeMounts() blocked the Node.js event loop
All `fs.*Sync` calls in `buildVolumeMounts()` in `src/container-runner.ts` ran synchronously on the main thread. On every container spawn, the entire event loop stalled while the host disk was checked and directories were created. **Fix:** converted the function to `async` using `fs.promises.*` throughout.

### [FIXED] cleanupOrphans() docker calls had no timeout
`docker ps` and `docker stop` in `cleanupOrphans()` in `src/container-runtime.ts` ran via `execSync` with no timeout. A hung Docker daemon or slow container could block service startup indefinitely. **Fix:** added `timeout: 5000` on `docker ps` and `timeout: 15000` on `docker stop`.

### [FIXED] No pre-flight environment check at startup
Neither an API credential nor the container image was verified at startup. Missing credentials caused silent failures deep inside container runs; a missing image caused confusing `spawn` errors with no actionable message. **Fix:** added `validateEnvironment()` in `src/index.ts` called before `ensureContainerSystemRunning()`. It checks for `CLAUDE_CODE_OAUTH_TOKEN` or `ANTHROPIC_API_KEY` and verifies the container image exists, with clear error messages for each failure.

### [FIXED] EACCES on .claude/debug/ — one-time host state issue
Container agent failed with `EACCES: permission denied` when trying to write inside `.claude/debug/`. Root cause: the `data/sessions/<group>/.claude/` directory was created by root with mode 755, so the container's `node` user (uid 1000) could not create subdirectories. **Fix:** `buildVolumeMounts()` now calls `fs.promises.chmod(groupSessionsDir, 0o777)` immediately after `mkdir` so the container user always has write access. The async rewrite preserves this chmod.

### [OPEN] Cursor advanced before agent completes
`processGroupMessages` in `src/index.ts` advances `lastAgentTimestamp` before the agent runs. If the container times out, retries find no messages — the cursor is already past them. Messages are permanently lost on timeout. **Fix:** only advance cursor on confirmed success, or roll back on timeout (not just on agent error). Tracked in Phase 1 tasks.

---

## Decisions

_(Record key decisions and the reasoning here as they happen.)_

---

## Gotchas

- **Container build cache**: `--no-cache` alone does NOT invalidate COPY steps. The buildkit builder volume retains stale files. Prune the builder first, then re-run `./container/build.sh`.
- **Mount path for sessions**: Container user is `node` with HOME=/home/node. Session mounts must go to `/home/node/.claude/`, not `/root/.claude/`.
- **Readonly mounts**: Use `--mount type=bind,...,readonly` not the `:ro` suffix — `:ro` may not work on all container runtimes.
- **Paths must be absolute**: `config.ts` resolves paths from `process.cwd()`. Container volume mounts require absolute paths or they silently break.
- **Linux vs macOS**: Service management on this install is systemd, not launchd. Commands throughout the docs assume macOS. Linux equivalents are in `CLAUDE.md`.
- **Trigger pattern is start-of-message**: `@Andy` must appear at the beginning of the message. `Hey @Andy` is ignored. Configured in `src/config.ts` as a regex.

---

## Notes on Container Timeout Behaviour

The `CONTAINER_TIMEOUT` (default 30min) is **not an absolute runtime cap** — it is a **rolling idle kill timer**. It resets every time the agent produces an output marker (`---NANOCLAW_OUTPUT_START---`). A task that produces periodic intermediate outputs can run for hours without being killed.

The container is only killed if there is **30 consecutive minutes of complete silence** — no output markers at all.

**This matters most for long-running skills** — specifically anything that triggers a FrontierBoard board review. Real-world timing from a 3-agent board (Claude + Qwen + Codex):

| Agent | Time to complete a single-file review |
|-------|---------------------------------------|
| Claude Code | 8–12 min |
| Qwen | 8–12 min |
| Codex (o4-series) | ~40 min |

A mixed board is bottlenecked by the slowest agent. If the NanoClaw container blocks on the bridge command and produces no output for 30+ minutes, the rolling kill timer fires and the review is lost. **This will happen reliably with any board that includes a Codex agent.**

The fix is in the `board-review` NanoClaw skill: run the bridge in the background and poll every few minutes, writing a status update to the user each time. Each write resets the rolling kill timer. See FrontierBoard's `board-review` skill template.

Practical breakdown:
- **Short tasks** (most WhatsApp queries): unaffected, complete in seconds to minutes.
- **Research/report tasks with intermediate outputs**: can run well beyond 30 minutes safely.
- **Board reviews — Claude/Qwen only**: 8–15 min total, no timeout risk.
- **Board reviews — includes Codex**: 40+ min, WILL be killed without the polling pattern.
- **`CONTAINER_TIMEOUT` is configurable**: users who want a blunt solution can raise it in `.env`, but the polling approach is better because it also keeps the user informed.

The `IDLE_TIMEOUT` (default 5min) is separate — it closes stdin (stops new WhatsApp messages being piped in) after 5 minutes of no output, but the container itself keeps running until the rolling kill timer fires.

The `IDLE_TIMEOUT` (default 5min) is separate — it closes stdin (stops new WhatsApp messages being piped in) after 5 minutes of no output, but the container itself keeps running until the rolling kill timer fires.

---

## Open Questions

_(Things not yet decided. Move to Decisions when resolved.)_

- Should the cursor rollback apply on all error paths, or only on container timeout specifically?
- Is there a need for a Linux-specific systemd unit file in the repo, or is `CLAUDE.md` enough?
- For long-running tasks (board reviews, research): should the agent emit periodic heartbeat output markers to keep the rolling kill timer alive, rather than relying on natural output frequency?

<img src="https://my-badges.github.io/my-badges/fix-5.png" alt="I did 5 sequential fixes." title="I did 5 sequential fixes." width="128">
<strong>I did 5 sequential fixes.</strong>
<br><br>

Commits:

- <a href="https://github.com/lweiss01/holistic/commit/3b7582747840b51848ab4be3e23f862c9b7c2c33">3b75827</a>: fix(andon): writer self-heals + correct agent identity (Phase 2 cont.)

Three issues surfaced verifying the daemon-run writer against a live session:

1. The writer only asserted lifecycle status on transitions. Heartbeats merely
   PRESERVE status, so when the API independently changed the stored status
   (e.g. a restart reconciliation sweep parked a running row) the writer never
   recovered. Now it re-asserts running/waiting on a periodic interval
   (ANDON_RUNTIME_WRITER_REASSERT_MS, default 60s) so it self-heals.

2. The writer reported sourceType "file_heartbeat" because normalizedSourceType
   never mapped agent "claude" -> "claude_code". Added an agent-agnostic
   agent->sourceType alias map (claude->claude_code, copilot->github_copilot,
   gsd2->gsd, etc.) so Mission Control shows the real agent identity.

3. maybeUpsertMirrorRuntimeFromLegacyEvent refused to update a row owned by a
   prior direct source (the retired per-agent hook), freezing the session in
   its last status and starving runtime_events (so the liveness sweep parked
   it). A runtime-writer event — Holistic's canonical agent-agnostic liveness
   source — may now take over such a row.

Verified end-to-end: daemon spawns writer, session projects live/running with
sourceType=claude_code, freshness fresh, ingestion active. 238 tests pass.

Refs holistic-m5c (epic holistic-wws)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
- <a href="https://github.com/lweiss01/holistic/commit/5c3f05af6ad9b0c14c6a76d5d4654da093796b9c">5c3f05a</a>: fix(andon): agent-agnostic liveness via daemon-run runtime-writer (Phase 1+2)

Root cause of "agent disconnected / shows running while waiting":
- The agent-agnostic forwarder (scripts/andon-runtime-writer.mjs) reads
  state.json and POSTs liveness for WHATEVER agent owns the session, but the
  daemon never started it — only the manual `npm run andon:dev` path did. So
  in normal use nothing heartbeats and Mission Control decays to disconnected.
- A turn-completion signal was being treated as session-end, marking active
  sessions completed/historical and dropping them off Mission Control.

Phase 1: daemon startAndonServices() now spawns the runtime-writer (like the
API), tracked in owned[] and killed via killTree on shutdown. Agent-agnostic
liveness for every agent, no per-agent hooks.

Phase 2: turn-completion != session end. The writer emits session.completed
ONLY when endedAt is set; a completion signal on an active session emits
session.needs_input (waiting), and work resuming emits work.started (running).
Freshness is purely state.json age (no completionSignal "fresh forever").
repository.ts: agent.summary_emitted+signal now maps to waiting_for_input,
not completed, so a `holistic checkpoint --reason task-complete` no longer
flips the live session to historical.

Tests: rewrote the writer completion test to the new contract; added
work-resumes and ended-session cases; allowed getTimeline in the dashboard API
client test (M007 timeline panel fetcher). 238 tests pass.

Refs holistic-35n holistic-m5c (epic holistic-wws)

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>
- <a href="https://github.com/lweiss01/holistic/commit/03439ac66787a9909f982774daa6a6bc9485e5dc">03439ac</a>: fix(M007): Stop hook sends session.needs_input so dashboard shows waiting state

When Claude finishes a turn and waits for user input, the hook was firing
agent.summary_emitted which falls through to the default "running" return in
legacyAgentEventToMirrorRuntimeStatus(). The correct event type is
session.needs_input → rawRuntimeStatus = "waiting_for_input" →
isRuntimeNeedsActionStatus = true → category: "needs_action" →
primaryStatus: "waiting_on_human_input".

When the next tool fires (first PostToolUse after user responds), the status
resets to "running" automatically via the default return path.

Also adds inputNeeded: true to the payload for clarity.

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
- <a href="https://github.com/lweiss01/holistic/commit/d92c440eb51b649f6ea73dda0e853e7ae07dba93">d92c440</a>: fix(M007): derive sourceType from state.json agent to surface session in Mission Control

Root cause: hook events posted without payload.sourceType caused
maybeUpsertMirrorRuntimeFromLegacyEvent to keep andonIngestMirror:true
on the runtime_sessions row. isAndonIngestMirrorSession() returning true
set sourceOfTruth:"mixed", and normalizeTopLevelOperationalItem forces
any non-"runtime" session to category:"historical" / belongsOnMissionControl:false.

Fix: read activeSession.agent from state.json (already loaded for sessionId)
and map it to the correct Andon sourceType value:
  claude   -> claude_code
  codex    -> codex
  cursor   -> cursor
  copilot  -> github_copilot
  gsd/gsd2 -> gsd
  others   -> http_event_source (non-empty = non-mirror)

Adding sourceType/sourceId/transport to every event payload sets
isDirectAgentSourceEvent=true, which flips andonIngestMirror to false,
sourceOfTruth to "runtime", and category to "live" in Mission Control.

Updated: renderAndonHookPs1(), renderAndonHookSh() in setup.ts.
Live scripts in .holistic-local/system/ patched directly.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
- <a href="https://github.com/lweiss01/holistic/commit/d062a3b9871cd6db66e00f425a6883ffb6910a50">d062a3b</a>: fix(M007): fix andon-hook.ps1 broken Invoke-RestMethod line continuation

The S01 executor wrote `` (double backtick) for PowerShell line
continuation instead of ` (single backtick). In PowerShell, `` is an
escaped literal backtick character, not line continuation — so
Invoke-RestMethod was called with only -Uri and no -Method/-Body,
causing all PostToolUse events to be silently dropped.

Fix: collapse the multi-line Invoke-RestMethod into a single line in
renderAndonHookPs1() in setup.ts so holistic repair regenerates the
correct script. Live script patched directly in .holistic-local/system/.

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>


Created by <a href="https://github.com/my-badges/my-badges">My Badges</a>
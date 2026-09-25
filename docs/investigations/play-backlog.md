---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "65810326-0542-45ae-8f47-25b9ca34cf61"
prompt: |
  @shape-backlog

  Use docs/investigations/play-evidence-map.md and
  docs/investigations/play-corroboration.md.

  Put the prioritized backlog in docs/investigations/play-backlog.md.
  Do not implement changes.
started: "2026-09-24"
ended: "2026-09-24"
task_durations:
  - task: "play workflow backlog shaping"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/65810326-0542-45ae-8f47-25b9ca34cf61/conversation.md"
source: "software-archaeology-prompts.md"
---

# Play Workflow Prioritized Backlog

## Scope and Change Policy

This backlog is derived from [play-evidence-map.md](play-evidence-map.md) and [play-corroboration.md](play-corroboration.md). It records evidence-backed investigation and modernization work for the `/play` workflow. No implementation changes are proposed or made in this step.

Priority reflects user impact, likelihood of regression, blast radius, and evidence confidence:

- **P0**: Confirmed user-facing failure or data/state risk requiring immediate ownership.
- **P1**: High-value contract or operational risk that can break multiple playback paths.
- **P2**: Important resilience, observability, or coverage improvement after higher-risk items are characterized.

## Prioritized Backlog

| Priority | Backlog item                                                    | Value                                                                                                               | Risk reduced                                                                                                                              | Evidence confidence                                                                 | First move                                                                                                                                                             | Ownership question                                                                                               |
| -------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| P0       | Resolve inactive non-empty playlist presentation failure        | Prevent a valid playlist from failing after tracks have already been downloaded and joined.                         | User-visible playback failure and possible partial voice/queue state.                                                                     | High: static control-flow contradiction confirmed in `streamtype="playlist"`.       | Reproduce with a controlled non-empty playlist in an inactive chat and capture queue, call, and final response state.                                                  | Who owns playlist playback and the user-facing summary contract?                                                 |
| P1       | Characterize and protect the queue-record protocol              | Preserve stream-end continuation and admin controls when `/play` or queue fields change.                            | Cross-module breakage after apparently successful playback, orphaned files, wrong next-track branch, and broken skip/seek/speed behavior. | High: writers and consumers corroborated; no schema or tests.                       | Build a record matrix for `vid_`, `live_`, `index_url`, Telegram, SoundCloud, and generic local records, then observe `change_stream` and controls.                    | Who owns the queue record contract across `stream.py`, `queue.py`, `call.py`, and admin plugins?                 |
| P1       | Characterize force-play stop/join ordering                      | Make replacement playback behavior predictable when an existing queue and voice call are active.                    | Mixed active-chat state, duplicate/failed joins, dropped queue heads, and replacement ordering regressions.                               | High for static ordering; runtime timing remains open.                              | Capture queue and active-call state before force-stop, after stop, after join, and after front insertion, including suppressed call errors.                            | Who owns voice-call lifecycle semantics and the expected preservation of remaining queued items?                 |
| P1       | Validate or redesign the direct index-link probe dependency     | Clarify whether the log-group probe is a required product behavior or an incidental gate.                           | Valid index/M3U8 requests rejected by unavailable log-group calls; unnecessary voice-call churn.                                          | High: direct call ordering and `LOG_GROUP_ID` dependency are explicit.              | Run a valid index request with log-group call availability on and off, recording whether target playback is reached and what users see.                                | Who owns the log-group playback probe and its availability SLO/configuration?                                    |
| P1       | Define failure outcomes for silent playback branches            | Ensure download-false, all-invalid playlist, and skipped provider results leave a deterministic user-visible state. | Stuck progress messages, silent failures, misleading success, and difficult incident diagnosis.                                           | High: broad catches and silent returns are corroborated; exact runtime UX is open.  | Capture Telegram messages and queue/call state for provider exception, download false, all-invalid playlist, and stream exception cases.                               | Who owns localized error text and the distinction between retryable, skipped, and terminal failures?             |
| P2       | Establish focused regression coverage for `/play` contracts     | Make future workflow changes falsifiable before deployment.                                                         | Regressions across wrapper gates, providers, queue shape, force-play, callbacks, and stream-end transitions.                              | High need, low current coverage confidence: no repository tests or workflows found. | Identify the supported test seam and create a minimal scenario matrix covering wrapper gates, one case per stream type, force-play, inactive playlist, and stream end. | Is there external CI or a hidden harness, and who can provide Telegram/PyTgCalls/Mongo test doubles or fixtures? |
| P2       | Add provenance and operational visibility for playback branches | Make future failures attributable without relying on broad exception names or silent returns.                       | Slow diagnosis of provider, assistant, queue, and voice-call failures; inability to distinguish branch-level regressions.                 | Medium: observability gaps are inferred from broad catches and sparse history.      | Inventory existing logs/metrics and correlate branch, chat, provider, stream type, queue position, and failure category without exposing user secrets.                 | Who owns production telemetry, retention, and privacy requirements for chat/playback metadata?                   |

## Detailed Items

### P0-1: Resolve Inactive Non-Empty Playlist Presentation Failure

- **Evidence**: `stream.py` assigns `position` in the active-chat playlist branch but not in the inactive-chat branch; the final non-empty summary formats `position`.
- **Impact**: A valid playlist can download tracks, join a call, and populate queue state before failing while building the final summary. This creates a partial-success state that is harder to recover than an early lookup failure.
- **Acceptance evidence**: A controlled non-empty inactive playlist produces either the intended summary or a clearly captured terminal failure with no unexplained partial state.
- **Unresolved**: Whether the summary should represent the first queued position, the last inserted position, or another playlist-level result.

### P1-1: Characterize and Protect the Queue-Record Protocol

- **Evidence**: `put_queue` and `put_queue_index` write process-local records consumed later by `Aviax.change_stream`, admin controls, cleanup, and presentation logic. Routing uses marker conventions such as `vid_`, `live_`, `index_url`, `telegram`, and `soundcloud`.
- **Impact**: A field or marker change can leave initial `/play` successful but break stream-end continuation or control commands.
- **Acceptance evidence**: Each record family has a documented field/marker matrix and a successful next-track/control transition in the supported test seam.
- **Unresolved**: Whether this process-local queue is authoritative, and whether any external consumers exist outside the repository.

### P1-2: Characterize Force-Play Stop/Join Ordering

- **Evidence**: `stream` force-stops before branch selection; `force_stop_stream` removes only the current head, suppresses several errors, clears active state, and leaves the call; force insertion then places the replacement at the front while preserving remaining records.
- **Impact**: Timing or partial cleanup differences can create mixed call state or unexpected queue order.
- **Acceptance evidence**: A force-play scenario records deterministic state transitions and confirms the intended treatment of remaining queued items.
- **Unresolved**: Whether preserving the remaining queue is an explicit requirement or an implementation side effect.

### P1-3: Validate or Redesign the Direct Index-Link Probe Dependency

- **Evidence**: `play_commnd` probes `LOG_GROUP_ID` through `Aviax.stream_call` before invoking target-chat `streamtype="index"`; the probe joins, waits, and leaves a call.
- **Impact**: A valid URL can fail before target playback because an unrelated log-group call is unavailable.
- **Acceptance evidence**: Behavior is documented for probe success, no active log-group call, assistant failure, and invalid stream URL.
- **Unresolved**: Whether the log group is intentionally a validation target, a legacy workaround, or an operational health check.

### P1-4: Define Failure Outcomes for Silent Playback Branches

- **Evidence**: Telegram download false returns silently; playlist detail failures are skipped; an all-invalid playlist can return without a summary; broad exception handlers collapse distinct causes.
- **Impact**: Users can be left with stale progress messages or no explanation, while operators lack a reliable failure category.
- **Acceptance evidence**: Each failure class has a deterministic user-visible outcome and captured queue/call cleanup state.
- **Unresolved**: Product policy for partial playlist success and whether failed items should be reported, retried, or omitted.

### P2-1: Establish Focused Regression Coverage for `/play` Contracts

- **Evidence**: No conventional test files or repository workflow configuration were found; queue and call contracts span multiple modules.
- **Impact**: Changes to `/play` can regress provider routing, wrapper gates, queue shape, force-play, callbacks, or stream-end behavior without a local signal.
- **Acceptance evidence**: A repeatable scenario matrix runs in the supported environment and covers the high-risk branches above.
- **Unresolved**: Test-environment ownership and availability of safe Telegram/PyTgCalls/Mongo doubles or fixtures.

### P2-2: Add Provenance and Operational Visibility for Playback Branches

- **Evidence**: Failure handling uses broad catches, exception names, silent returns, and limited history; runtime telemetry was unavailable during excavation.
- **Impact**: Incident response cannot reliably distinguish provider, assistant, queue, configuration, and voice-call failures.
- **Acceptance evidence**: Branch-level outcomes are observable with privacy-safe correlation and no credential or message-content leakage.
- **Unresolved**: Required telemetry platform, retention, access controls, and compliance constraints.

## Sequencing and Stop Conditions

1. **P0 first**: Reproduce and bound the inactive playlist failure before any broader playback change is considered.
2. **P1 contract work next**: Characterize queue records, force-play transitions, index probing, and failure outcomes because they can invalidate multiple provider paths.
3. **P2 enablement after characterization**: Establish repeatable regression coverage and operational visibility based on the confirmed contracts.

This backlog is ready for ownership assignment, but not for implementation approval until the P0 reproduction and the unresolved queue/call semantics are confirmed by the responsible maintainers.

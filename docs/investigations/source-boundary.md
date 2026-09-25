---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "65810326-0542-45ae-8f47-25b9ca34cf61"
prompt: |
  @locate-source-boundary
  Use the Investigation Framing in docs/investigations/play-workflow.md. Put the response in the file docs/investigations/source-boundary.md
started: "2026-09-24"
ended: "2026-09-24"
task_durations:
  - task: "source-boundary discovery"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/65810326-0542-45ae-8f47-25b9ca34cf61/conversation.md"
source: "software-archaeology-prompts.md"
---

# Play Workflow Source Boundary

## Scope

This record applies the [play workflow investigation framing](play-workflow.md). It identifies the smallest source boundary needed to begin execution tracing for `/play`; it does not map every collaborator implementation, inspect repository history, or recommend code changes.

## Initial Source Boundary

| Responsibility                              | Files and symbols                                                                                                                                                                                                                                            | Why it is in scope                                                                                                                                                                                                         |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Startup and plugin discovery                | `AviaxMusic/__main__.py::init`; `AviaxMusic/plugins/__init__.py::__list_all_modules`, `ALL_MODULES`                                                                                                                                                          | Startup imports the discovered plugin modules before command handlers can receive messages.                                                                                                                                |
| Command registration and handler            | `AviaxMusic/plugins/play/play.py::play_commnd`                                                                                                                                                                                                               | The module registers `/play`, `/vplay`, `/cplay`, `/cvplay`, and the force-play variants, then selects parsing, provider, response, and playback branches.                                                                 |
| Eligibility and request context             | `AviaxMusic/utils/decorators/play.py::PlayWrapper`                                                                                                                                                                                                           | The wrapper handles language, anonymous-admin rejection, maintenance, message deletion, media/URL extraction, channel routing, authorization, video mode, force-play, assistant membership, and dispatch to `play_commnd`. |
| Media lookup and download interfaces        | `AviaxMusic/platforms/Telegram.py`; `Youtube.py`; `Spotify.py`; `Apple.py`; `Resso.py`; `Soundcloud.py`                                                                                                                                                      | `play_commnd` directly calls these provider interfaces for reply media, URLs, search terms, playlists, albums, artists, and SoundCloud downloads.                                                                          |
| Playback handoff and queue mutation         | `AviaxMusic/utils/stream/stream.py::stream`; `AviaxMusic/utils/stream/queue.py::put_queue`, `put_queue_index`                                                                                                                                                | The handler delegates the main playback branches to `stream`, which chooses active-chat queueing versus new-call startup and invokes queue primitives.                                                                     |
| Voice-call control                          | `AviaxMusic/core/call.py::Aviax.join_call`, `force_stop_stream`, `stream_call`; `group_assistant` usage                                                                                                                                                      | Direct links, stream branches, and force-play behavior depend on assistant selection, joining/leaving calls, and active-chat state.                                                                                        |
| Playback state and configuration            | `AviaxMusic/utils/database.py::get_assistant`, `get_cmode`, `get_lang`, `get_playmode`, `get_playtype`, `is_active_chat`, `add_active_chat`, `add_active_video_chat`; `config.py`; `AviaxMusic/misc.py::SUDOERS`, `db`; `config.adminlist`, `config.lyrical` | These values and stores directly control reachability, routing, language, mode, limits, queue state, permissions, playlist handoff, and video-chat state.                                                                  |
| User-visible completion and failure signals | `AviaxMusic/utils/logger.py::play_logs`; `strings.get_string`; inline builders imported by `play.py`                                                                                                                                                         | Success, rejection, provider failure, queue status, and playback errors are surfaced through localized text, inline keyboards, media responses, and logs.                                                                  |

## Entry and Registration Path

1. **Fact**: `AviaxMusic/__main__.py::init` imports every path in `ALL_MODULES` after starting the bot client.
2. **Fact**: `AviaxMusic/plugins/__init__.py::__list_all_modules` discovers Python files one directory below the plugin package, so `plugins/play/play.py` is included.
3. **Fact**: Importing `plugins/play/play.py` evaluates the `app.on_message` decorator for eight command names and registers `play_commnd`.
4. **Fact**: `PlayWrapper` wraps `play_commnd`, so its early returns determine whether command-specific media handling runs.
5. **Fact**: The handler's principal playback branches call `stream`; the direct-link branch also probes the log group's voice call through `Aviax.stream_call` before handing off to `stream`.

## Direct Collaborators to Trace Next

- **Fact**: `PlayWrapper` directly reads language, maintenance, channel mode, play mode, play type, active-chat state, assistant identity, `SUDOERS`, `adminlist`, and `config` limits/settings; it may call Telegram membership, invite, approval, and assistant-join operations.
- **Fact**: `play_commnd` directly calls Telegram media helpers, provider interfaces, `stream`, `Aviax.stream_call`, `play_logs`, and inline markup builders.
- **Fact**: `stream` directly chooses playlist, YouTube, SoundCloud, Telegram, live, and index behavior; it mutates `db`, calls `put_queue` or `put_queue_index`, invokes `Aviax.join_call`, and sends or edits user-facing messages.
- **Inference**: `stream.py`, `queue.py`, and `call.py` are the highest-value next execution-tracing targets because they contain the queue mutation and voice-call side effects shared by multiple `/play` input forms. This should be confirmed by tracing each `streamtype` branch.

## Evidence Ledger

| Claim                                                                                                             | Classification | Confidence | Smallest disconfirming check                                                                                                                   |
| ----------------------------------------------------------------------------------------------------------------- | -------------- | ---------: | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `/play` reachability depends on plugin discovery and decorator registration before the wrapper runs.              | Fact           |       High | Confirm the runtime entry point invokes `init` in the deployed process.                                                                        |
| `PlayWrapper` can terminate a request before `play_commnd` receives it.                                           | Fact           |       High | Exercise anonymous-admin, maintenance, missing-media, channel-routing, authorization, inactive-chat, and assistant-membership return branches. |
| The handler has separate direct, playlist, search/inline, Telegram-media, SoundCloud, live, and index-link paths. | Fact           |       High | Enumerate every `streamtype`, provider branch, and callback handoff in `play.py`; identify any unobserved path.                                |
| Queue and voice-call behavior is shared across provider branches through `stream`.                                | Inference      |       High | Trace all `stream` callers and verify whether any branch performs those effects outside `stream`.                                              |
| Conventional automated coverage for this workflow is unknown.                                                     | Open question  |        Low | Search test directories and CI configuration after the boundary is recorded.                                                                   |

## Excluded from This Boundary Pass

- Callback handlers in `AviaxMusic/plugins/play/play.py` such as `play_music`, playlist callbacks, and slider callbacks: they are follow-up interaction paths, not direct command invocation, though their shared `stream` calls remain relevant later.
- Full implementations of provider downloads, Telegram file handling, queue consumers, stream-end handlers, database persistence, and assistant selection: they are direct collaborators but require execution tracing rather than boundary discovery.
- Unrelated commands and plugins: include only if later evidence shows they mutate state or registration used by `/play`.
- Repository history, deployment values, operational telemetry, and tests: required by the overall stop condition, but deferred until the first evidence map exists.

## Smallest Next Read

Read `AviaxMusic/utils/stream/stream.py::stream` branch by branch, then follow `put_queue`/`put_queue_index` and `Aviax.join_call` for the first evidence map. Record success, failure, retry, cancellation, state, external effects, and user-visible outcomes for each branch.

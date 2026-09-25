---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "86413381-aa2f-455d-8861-f99d23d14b14"
prompt: |
  @excavate

  Use docs/investigations/source-boundary.md to investigate the MusicBot `/play`
  workflow. Put the evidence map in docs/investigations/play-evidence-map.md.

  Do not propose or make code changes.
started: "2026-09-24T17:24:53.3345525-07:00"
ended: "2026-09-24T17:24:53.3345525-07:00"
task_durations:
  - task: "play workflow evidence mapping"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/86413381-aa2f-455d-8861-f99d23d14b14/conversation.md"
source: "johnmillerATcodemag-com"
---

# Play Workflow Evidence Map

## Scope and Stop Status

This map follows the [source boundary](source-boundary.md) for the `/play` workflow. It traces startup, registration, request gating, handler branches, queue mutation, voice-call effects, configuration, tests, and targeted history. It does not propose or make code changes.

The direct path is mapped. Test coverage and the rationale for the latest call-control change remain open questions and are recorded below.

## Execution Path

1. **Startup - Fact, impact 100**: `Dockerfile::CMD` and `Procfile` both run `start`; `start` executes `python3 -m AviaxMusic`, which invokes `AviaxMusic/__main__.py::init`.
2. **Plugin registration - Fact, impact 100**: `__main__.py::init` imports each `plugins.__init__::ALL_MODULES` item. Importing `plugins/play/play.py` evaluates its Pyrogram decorator and registers `play_commnd` for group `/play`, video, channel, and force-play variants, excluding `config::BANNED_USERS`.
3. **Request gating - Fact, impact 95**: `utils/decorators/play.py::PlayWrapper` resolves language, rejects sender-chat, maintenance, missing-input, channel-mode, authorization, inactive force-play, and assistant-access cases, then invokes `play_commnd` with the resolved context.
4. **Handler selection - Fact, impact 95**: `plugins/play/play.py::play_commnd` selects among replied Telegram audio/video, recognized provider URLs, an unrecognized direct index/M3U8-style URL, or a text search.
5. **Playback dispatch - Fact, impact 100**: the principal branches call `utils/stream/stream.py::stream`; this function decides whether to queue an item or start a voice call.
6. **Queue advancement - Fact, impact 90**: `core/call.py::Call.decorators` registers `stream_end_handler1`, which invokes `Call.change_stream` after an audio stream ends; it consumes the queue head, changes the voice stream, or leaves the call when no queued item remains.

## Branch Map

| Input or mode                                     | Direct behavior                                                                                                                                                                                                      | State and external effects                                                                                                                                                                                 | Failure behavior                                                                                                |
| ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Replied audio                                     | `play_commnd` checks a hard-coded 100 MB limit and `config::DURATION_LIMIT`, downloads through `Telegram`, then calls `stream(..., streamtype="telegram")`.                                                          | `stream` queues when active; otherwise joins the group call, records the queue item, and sends a stream message.                                                                                           | Size, duration, download, and stream failures become response edits.                                            |
| Replied video or document                         | Handler validates document format with `utils/formatters.py::formats`, checks `config::TG_VIDEO_FILESIZE_LIMIT`, downloads through `Telegram`, then calls the Telegram stream branch with `video=True`.              | Same queue/start split; initial video playback also calls `utils/database.py::add_active_video_chat`.                                                                                                      | Invalid extension, size, download, and stream failures return response edits.                                   |
| YouTube, Spotify, Apple, Resso, or SoundCloud URL | Handler validates with the corresponding platform interface, fetches track or playlist details, applies duration/provider checks, then streams directly or renders an inline selection flow based on `get_playmode`. | Provider and download calls are external effects; direct mode leads to `stream`, while non-direct mode stores playlist state in `config::lyrical` or presents an inline selector.                          | Provider exceptions are generally converted to a generic playback-error response.                               |
| Unrecognized URL                                  | Handler calls `Aviax.stream_call(url)` before calling `stream(..., streamtype="index")`.                                                                                                                             | `stream_call` joins then leaves `config::LOG_GROUP_ID`; the stream branch queues or joins the requested chat using the supplied URL.                                                                       | A missing active group call sends a log-group message; other failures become a generic response edit.           |
| Text query                                        | Handler resolves the query through `YouTube.track`. Direct mode streams it; non-direct mode renders an inline selector.                                                                                              | Direct mode reaches the shared stream/queue path; inline mode defers playback to callbacks outside this invocation path.                                                                                   | Missing query or lookup failure responds without queue or call mutation.                                        |
| Active chat                                       | `stream::stream` calls `put_queue` or `put_queue_index`.                                                                                                                                                             | `utils/stream/queue.py` appends a queue record to `AviaxMusic.misc::db` and sends a queue-position message.                                                                                                | Download/provider errors occur before the queue write for YouTube; call start is not attempted.                 |
| Inactive chat                                     | `stream::stream` initializes or preserves `db[chat_id]`, calls `Aviax.join_call`, then writes the first queue record.                                                                                                | `Call.join_call` invokes PyTgCalls, marks the chat active, enables music, optionally records active video, then `stream` sends a now-playing message and stores it on the first queue record.              | `NoActiveGroupCall`, already-joined, and server errors become `AssistantErr`; callers render a localized error. |
| Force play                                        | `stream` calls `Call.force_stop_stream`, then enters the inactive-chat path.                                                                                                                                         | `force_stop_stream` removes the current queue head, clears active flags, and leaves the group call; `put_queue(..., forceplay=True)` inserts the new item at queue index zero while retaining later items. | Leave failures are suppressed; a following join or queue operation may still fail normally.                     |

## State and Dependency Map

- **Fact, impact 100**: `AviaxMusic.misc::db` is the in-memory queue keyed by `chat_id`. `queue.py::put_queue` records metadata including media reference, requester, duration-derived seconds, and playback status.
- **Fact, impact 95**: `database.py::is_active_chat`, `add_active_chat`, and `remove_active_chat` manage in-memory `active` membership. `Call.join_call` adds membership after PyTgCalls joins; `force_stop_stream`, `stop_stream`, and queue exhaustion remove it.
- **Fact, impact 90**: `database.py::get_playmode` defaults a missing value to `"Direct"`; this default determines whether a search/provider result streams immediately or opens an inline selection workflow.
- **Fact, impact 90**: `database.py::get_assistant` can select or allocate an assistant client through persistent assistant data before the handler runs.
- **Fact, impact 90**: external effects include Telegram messages/files, provider metadata/download APIs, PyTgCalls voice-chat operations, Mongo-backed settings reads, thumbnail generation, and a paste service for playlists.

## Configuration and Operational Evidence

- **Fact, impact 85**: `config.py::DURATION_LIMIT_MIN` is read from `DURATION_LIMIT` and converted by `config.py::DURATION_LIMIT`; it gates Telegram audio, SoundCloud, and direct-track paths.
- **Fact, impact 80**: `config.py::PLAYLIST_FETCH_LIMIT`, Spotify credentials, media-size limits, and `LOG_GROUP_ID` directly alter provider, playlist, and direct-URL behavior.
- **Fact, impact 90**: `Dockerfile::CMD` and `Procfile` both converge on `start`; `start` runs the Python module entry point. `app.json` requires bot, Mongo, assistant-session, and log-group configuration for its Heroku deployment.
- **Open question, impact 75**: no workflow files were found under `.github/workflows`, and no test framework appears in `requirements.txt`. The only `*test*.py` filename match is `plugins/tools/speedtest.py`, which is an operational command rather than test evidence. This does not prove that no tests exist outside conventional paths.

## History and Provenance

- **Fact, impact 60**: focused history lists `e92667a Update call.py` and `3315863 v3.0 Stable` for the handler, stream, queue, and call-control files.
- **Open question, impact 70**: `e92667a` changes only `core/call.py`, but its commit message (`"Update call.py"` and `"hmm done"`) does not explain the intended behavioral contract. No linked issue or detailed rationale was established in this pass.

## Evidence Ledger

| Claim                                                                                                         | Classification | Impact | Evidence                                                                                                                             | Smallest disconfirming check                                                                                           |
| ------------------------------------------------------------------------------------------------------------- | -------------- | -----: | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `/play` reaches `play_commnd` through the runtime entry point and plugin import.                              | Fact           |    100 | `Dockerfile::CMD`; `Procfile`; `start`; `__main__.py::init`; `plugins/__init__.py::ALL_MODULES`; `plugins/play/play.py::play_commnd` | Run the configured process and verify the plugin-import log line appears.                                              |
| The wrapper can block or reshape a request before media handling.                                             | Fact           |     95 | `utils/decorators/play.py::PlayWrapper`                                                                                              | Send controlled messages for each early-return condition.                                                              |
| `stream` is the control point for active-queue versus inactive-call behavior.                                 | Fact           |    100 | `utils/stream/stream.py::stream`; `utils/database.py::is_active_chat`; `core/call.py::Call.join_call`                                | Trace a branch that bypasses `stream` but mutates the same queue/call state.                                           |
| Force play retains queued entries after the current item while placing the requested item first.              | Fact           |     95 | `core/call.py::Call.force_stop_stream`; `utils/stream/queue.py::put_queue`                                                           | Exercise force play with a queue of three items and inspect `db[chat_id]`.                                             |
| Changing `stream` can regress both immediate playback and queued playback.                                    | Inference      |    100 | Every primary handler media branch invokes `stream`; it owns queue writes and call joins.                                            | Identify a primary `/play` branch that does not call `stream` and independently performs equivalent state transitions. |
| The direct URL branch has an additional log-group voice-call side effect.                                     | Fact           |     80 | `plugins/play/play.py::play_commnd`; `core/call.py::Call.stream_call`                                                                | Invoke an unrecognized URL and observe whether `LOG_GROUP_ID` is used.                                                 |
| Automated regression coverage for `/play` is absent or not discoverable by conventional repository structure. | Open question  |     75 | No `.github/workflows` files; no test dependency; only `speedtest.py` matched the filename search.                                   | Inspect external CI, branch protection, and nonstandard test directories.                                              |

## Fragility and Unknowns

- **Inference, impact 95**: queue and active-chat state are process-local collections. A restart or multi-process topology could desynchronize perceived playback state from the active voice call. Disprove by locating shared-state synchronization outside the examined boundary.
- **Fact, impact 85**: multiple handler and stream branches use broad exception handling that collapse provider/download/call failures into generic messages, reducing branch-specific operational visibility.
- **Open question, impact 85**: retry behavior belongs to provider, download, Telegram, and PyTgCalls dependencies; no explicit retry policy was found in the traced `/play` path.
- **Open question, impact 70**: callback-based inline selection and playlist continuation are intentionally outside this direct-command map and require a separate callback-to-stream trace.

## Smallest Next Checks

1. Verify direct, queued, and force-play behavior against a controlled three-item queue, recording `db[chat_id]`, active membership, and PyTgCalls calls.
2. Trace `play_music`, `play_playlists_command`, and `slider_queries` in `plugins/play/play.py` to complete the deferred inline-selection paths.
3. Search deployment/CI systems and issue history for the missing `/play` regression tests and the rationale for `e92667a`.

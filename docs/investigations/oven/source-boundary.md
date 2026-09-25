---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "86413381-aa2f-455d-8861-f99d23d14b14"
prompt: |
  @locate-source-boundary
  Use the Investigation Framing in docs/investigations/play-workflow.md. Put the response in the file docs/investigations/source-boundary.md
started: "2026-09-24T17:19:05.4131472-07:00"
ended: "2026-09-24T17:19:05.4131472-07:00"
task_durations:
  - task: "source-boundary discovery"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/86413381-aa2f-455d-8861-f99d23d14b14/conversation.md"
source: "johnmillerATcodemag-com"
---

# Play Workflow Source Boundary

## Scope

This record applies the [play workflow investigation framing](play-workflow.md). It identifies the source files and symbols required to begin execution tracing. It does not trace collaborator implementations, inspect history, or recommend changes.

## Initial Source Boundary

| Responsibility                   | Files and symbols                                                                                       | Basis                                                                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Startup and plugin discovery     | `AviaxMusic/__main__.py::init`; `AviaxMusic/plugins/__init__.py::__list_all_modules`, `ALL_MODULES`     | `init` imports every discovered plugin module.                                                                                                                              |
| Command registration and handler | `AviaxMusic/plugins/play/play.py::play_commnd`                                                          | Pyrogram command registration and `PlayWrapper` decorate the handler.                                                                                                       |
| Eligibility and request context  | `AviaxMusic/utils/decorators/play.py::PlayWrapper`                                                      | The wrapper validates request state and calls the handler with resolved context.                                                                                            |
| Playback handoff                 | `AviaxMusic/utils/stream/stream.py::stream`                                                             | `play_commnd` directly calls `stream` for Telegram, provider, and search results.                                                                                           |
| Provider and media handling      | `AviaxMusic/platforms/Telegram.py`; `Youtube.py`; `Spotify.py`; `Apple.py`; `Resso.py`; `Soundcloud.py` | `play_commnd` directly invokes their imported provider interfaces.                                                                                                          |
| Voice-call control               | `AviaxMusic/core/call.py::Aviax`                                                                        | `play_commnd` directly invokes `Aviax.stream_call` for a direct stream branch.                                                                                              |
| State and configuration          | `AviaxMusic/utils/database.py`; `config.py`                                                             | `PlayWrapper` directly reads language, mode, authorization, maintenance, assistant, and active-chat state; handler and decorators read playback limits and access settings. |

## Entry and Registration Path

1. **Fact**: `AviaxMusic/__main__.py::init` iterates `ALL_MODULES` and imports each discovered plugin module.
2. **Fact**: `AviaxMusic/plugins/__init__.py::__list_all_modules` discovers non-`__init__` Python files one directory below the plugin package, which includes `plugins/play/play.py`.
3. **Fact**: Importing `AviaxMusic/plugins/play/play.py` evaluates the `app.on_message` decorator for `/play`, `/vplay`, `/cplay`, `/cvplay`, and force-play variants, registering `play_commnd`.
4. **Fact**: `PlayWrapper` wraps `play_commnd`, so it controls reachability before command-specific media handling begins.

## Direct Collaborators

- **Fact**: `PlayWrapper` directly calls `get_lang`, `is_maintenance`, `get_cmode`, `get_playmode`, `get_playtype`, `is_active_chat`, and `get_assistant` from `utils.database`, and may call Telegram client methods to validate or join the assistant client.
- **Fact**: `play_commnd` directly calls `Telegram` media helpers, provider interfaces (`YouTube`, `Spotify`, `Apple`, `Resso`, and `SoundCloud`), `stream`, `Aviax.stream_call`, channel-play helpers, and `play_logs`.
- **Fact**: `config.py` supplies duration, video-size, banned-user, administrator, and presentation settings directly consulted by the registration, wrapper, or handler.

## Evidence Ledger

| Claim                                                                                                          | Classification | Impact | Evidence                                                                                     | Smallest disconfirming check                                                                       |
| -------------------------------------------------------------------------------------------------------------- | -------------- | -----: | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `/play` reaches `play_commnd` only after plugin-module import and Pyrogram decorator evaluation.               | Fact           |     95 | `__main__.py::init`; `plugins/__init__.py::ALL_MODULES`; `plugins/play/play.py::play_commnd` | Confirm the running entry point invokes `init`.                                                    |
| `PlayWrapper` can terminate a `/play` request before `play_commnd` receives it.                                | Fact           |     90 | `utils/decorators/play.py::PlayWrapper`                                                      | Exercise each wrapper return branch with a controlled message.                                     |
| `stream::stream` is the highest-value next execution-tracing target for queue mutation and voice-call startup. | Inference      |     90 | `plugins/play/play.py::play_commnd` calls `stream` for the principal media branches.         | Read `utils/stream/stream.py::stream`; disprove by locating those effects entirely in the handler. |
| The project may lack conventional automated tests for this workflow.                                           | Open question  |     70 | No `test/` or `tests/` directory matched the discovery search.                               | Search nonstandard test locations and CI configuration.                                            |

## Excluded in This Pass

- Callback handlers in `plugins/play/play.py` (`play_music`, playlist callbacks, and slider callbacks): they are subsequent inline-interaction paths, not direct `/play` invocation.
- Stream, queue, database, provider, and voice-call implementations: they are direct collaborators but outside one-hop source-boundary discovery.
- History, deployment configuration, and operational evidence: required by the overall investigation stop condition, but deferred until the execution path is mapped.

## Smallest Next Read

Read `AviaxMusic/utils/stream/stream.py::stream` to determine the queue mutation and voice-call side effects that follow the handler's direct calls.

---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "86413381-aa2f-455d-8861-f99d23d14b14"
prompt: |
  @corroborate

  Use docs/investigations/play-evidence-map.md.
  Put the corroboration record in docs/investigations/play-corroboration.md.

  Do not propose or make code changes.
started: "2026-09-24T17:32:24.4319003-07:00"
ended: "2026-09-24T17:32:24.4319003-07:00"
task_durations:
  - task: "play workflow corroboration"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/86413381-aa2f-455d-8861-f99d23d14b14/conversation.md"
source: "johnmillerATcodemag-com"
---

# Play Workflow Corroboration

## Scope

This record corroborates high-impact claims and open questions from the [play evidence map](play-evidence-map.md). It compares execution code with configuration, deployment declarations, test-discovery artifacts, and focused history. It does not remap the workflow, propose changes, or modify application code.

## Corroboration Ledger

| Claim examined                                                                | Result        | Impact | Corroborating evidence                                                                                                                                                                                                                         | What would disprove it                                                      | Smallest next check                                                                   |
| ----------------------------------------------------------------------------- | ------------- | -----: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| The deployed worker reaches `AviaxMusic/__main__.py::init` through `start`.   | Fact          |    100 | `Dockerfile::CMD` runs `bash start`; `Procfile` and `heroku.yml::run.worker` also run `bash start`; `start` runs `python3 -m AviaxMusic`.                                                                                                      | A deployment command override that bypasses `start`.                        | Inspect the deployed process command or release manifest.                             |
| An unrecognized direct URL has a log-group voice-call side effect.            | Fact          |     80 | `plugins/play/play.py::play_commnd` calls `Aviax.stream_call(url)`; `core/call.py::Call.stream_call` resolves `config::LOG_GROUP_ID`; `app.json` and `sample.env` both declare `LOG_GROUP_ID`.                                                 | A different call implementation selected at runtime.                        | Invoke the branch in an isolated environment and trace PyTgCalls calls.               |
| `stream::stream` controls active-chat queueing versus inactive-chat joining.  | Fact          |    100 | `utils/stream/stream.py::stream` branches on `database.py::is_active_chat`; queue branches call `queue.py::put_queue` or `put_queue_index`; inactive branches call `core/call.py::Call.join_call`.                                             | A primary `/play` branch that mutates equivalent state without `stream`.    | Instrument all `play_commnd` calls in a controlled test run.                          |
| Force play preserves queued entries after replacing the current item.         | Fact          |     95 | `Call.force_stop_stream` removes only `db[chat_id][0]`; `stream::stream` skips queue reset when `forceplay`; `queue.py::put_queue` inserts the new item at index zero.                                                                         | `db[chat_id]` is cleared or replaced elsewhere during the same call.        | Execute force play against a three-item queue and capture the queue before and after. |
| Missing persisted play mode defaults to immediate playback.                   | Fact          |     90 | `database.py::get_playmode` stores and returns `"Direct"` when no setting exists; `play.py::play_commnd` streams directly when `playmode == "Direct"`.                                                                                         | A higher-level setting overwrites the value before the handler receives it. | Invoke a fresh chat with no play-mode document and log the wrapper arguments.         |
| Duration and provider behavior are environment-dependent.                     | Fact          |     85 | `config.py::DURATION_LIMIT_MIN`, Spotify credentials, and playlist limit are environment-backed; `sample.env` and `app.json` provide only the core required values, so these behavior-specific values use code defaults unless externally set. | A deployment layer always injects different values.                         | Inspect the deployed environment metadata without exposing secrets.                   |
| Local automated regression coverage for `/play` is available.                 | Open question |     75 | No `.github/workflows` files, conventional test configuration, or test-framework dependency was found; `rg --files` found only `plugins/tools/speedtest.py`, an operational command.                                                           | A local nonstandard test runner or external CI pipeline exists.             | Inspect external CI, branch protection, and hidden/ignored test paths.                |
| `e92667a` explains the queue or call-control contract.                        | Contradiction |     70 | `git show e92667a -- AviaxMusic/core/call.py` changes only `config.SUPPORT_CHAT` to `config.SUPPORT_GROUP` in Telegram and SoundCloud captions.                                                                                                | A linked issue, PR, or review discussion defines a wider intent.            | Search remote commit/PR metadata for `e92667a`.                                       |
| Docker and non-container runtime declarations select the same Python version. | Contradiction |     65 | `Dockerfile` uses `nikolaik/python-nodejs:python3.10-nodejs19`; `runtime.txt` declares `python-3.11.4`. Heroku uses Docker through `heroku.yml`, so the Docker declaration controls that path.                                                 | Deployment documentation showing `runtime.txt` overrides the Docker image.  | Build each deployment target and record `python --version`.                           |

## Configuration Comparison

- **Fact, impact 90**: `app.json` and `sample.env` agree on the core runtime inputs used by `config.py`: Telegram API credentials, bot token, Mongo URI, owner ID, one assistant session, and log-group ID.
- **Fact, impact 80**: `config.py` supports up to five assistant sessions, while `app.json` and `sample.env` declare only the first. The remaining sessions are optional in the inspected declarations; no inconsistency is established.
- **Open question, impact 80**: actual deployed values for duration, playlist, Spotify, and optional assistant settings are unavailable. Static defaults do not establish production behavior.

## Test and Operational Evidence

- **Fact, impact 75**: the workspace provides no conventional local test runner configuration or workflow under `.github/workflows`.
- **Open question, impact 85**: static inspection cannot establish the behavior of Telegram, provider APIs, PyTgCalls, MongoDB, or a deployed worker. The active queue and voice-call conclusions are code-level facts, not runtime observations.
- **Fact, impact 85**: no static contradiction was found between `play.py::play_commnd`, `stream.py::stream`, `queue.py::put_queue`, and `core/call.py::Call.join_call` for the active-chat versus inactive-chat transition.

## Contradictions Preserved

1. **Runtime version declarations differ**: Docker specifies Python 3.10; `runtime.txt` specifies Python 3.11.4. This is relevant to deployment reproducibility, but its effect on `/play` remains unverified.
2. **History does not support the presumed call-control rationale**: the latest focused `call.py` commit changes display-caption configuration only, not queue, stream, or voice-call behavior.

## Corroboration Status

The high-impact static control-path claims are corroborated. The remaining uncertainty is operational: runtime behavior, external dependencies, unavailable CI/test evidence, and deployed environment values require observations outside this workspace.

## Smallest Next Check

Run a controlled three-item queue scenario covering direct play, queued play, and force play; record `AviaxMusic.misc::db[chat_id]`, active-chat membership, and PyTgCalls join/change/leave calls. This single observation tests the highest-impact unexecuted state claims.

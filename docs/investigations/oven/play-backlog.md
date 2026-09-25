---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "86413381-aa2f-455d-8861-f99d23d14b14"
prompt: |
  @shape-backlog

  Use docs/investigations/play-evidence-map.md and
  docs/investigations/play-corroboration.md.

  Put the prioritized backlog in docs/investigations/play-backlog.md.
  Do not implement changes.
started: "2026-09-24T17:37:34.5931885-07:00"
ended: "2026-09-24T17:37:34.5931885-07:00"
task_durations:
  - task: "play workflow backlog shaping"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/86413381-aa2f-455d-8861-f99d23d14b14/conversation.md"
source: "johnmillerATcodemag-com"
---

# Play Workflow Backlog

## Basis

These items derive from the [play evidence map](play-evidence-map.md) and [play corroboration](play-corroboration.md). They are prioritized recommendations only; no implementation is included.

## 1. Characterize Queue and Force-Play State Transitions

- **Type**: Characterization test
- **Value**: Establishes an executable contract for the behavior most likely to regress when `/play`, `stream`, queue handling, or call control changes.
- **Risk reduced**: A change can otherwise alter whether a request starts playback, appends, replaces the current item, preserves queued items, or leaves/rejoins a voice call without detection.
- **Evidence confidence**: High. Static control flow is corroborated across `stream.py::stream`, `queue.py::put_queue`, `call.py::Call.force_stop_stream`, and `call.py::Call.join_call`; runtime behavior remains unobserved.
- **First move**: Define three controlled scenarios - direct play in an inactive chat, enqueue while active, and force play with three queued items - and record the expected queue order, active membership, and join/change/leave calls before choosing a test seam.
- **Owner question**: Which team owns the PyTgCalls integration boundary and can provide an isolated call client or test environment?
- **Validates**: The impact-100 claim that `stream` controls active-versus-inactive behavior and the impact-95 claim that force play retains later queue entries while inserting the requested item first.

## 2. Map and Characterize Inline Selection Continuations

- **Type**: Investigation followed by characterization test
- **Value**: Completes the currently excluded paths for non-direct search, playlist, and slider workflows, so future `/play` changes do not silently break the handoff from inline UI to playback.
- **Risk reduced**: Inline mode may produce a valid selection UI but fail to enqueue or start the selected item; the direct-command evidence map does not yet cover these callbacks.
- **Evidence confidence**: High that the gap exists; medium for the exact downstream behavior. `play.py::play_commnd` defers non-direct playback to callbacks, while `play_music`, `play_playlists_command`, and `slider_queries` were excluded from the direct-command trace.
- **First move**: Trace each callback from its callback-data contract to `stream`, then document one success and one error expectation for each before proposing test coverage.
- **Owner question**: Is inline interaction owned by the playback maintainer, the bot UX maintainer, or both?
- **Validates**: The impact-70 open question that callback-based inline selection and playlist continuation remain outside the current `/play` execution map.

## 3. Establish the Deployment Python Runtime Contract

- **Type**: Explicit deployment contract
- **Value**: Makes the interpreter selected by each supported deployment path reviewable and reproducible before dependency or playback changes are tested or released.
- **Risk reduced**: Docker and non-container deployments can execute the same `/play` workflow under different Python versions, making behavior and dependency failures difficult to reproduce.
- **Evidence confidence**: High. `Dockerfile` selects Python 3.10, while `runtime.txt` declares Python 3.11.4; `heroku.yml` uses the Docker worker path.
- **First move**: Document the supported deployment targets and their intended interpreter versions, then capture `python --version` from a Docker build and the non-container target before selecting a canonical contract.
- **Owner question**: Who owns the release/deployment matrix and approves supported runtime versions?
- **Validates**: The impact-65 contradiction in the corroboration record: Docker and non-container runtime declarations do not select the same Python version.

## Deferred Items

No backlog item is proposed for provider retry policy, external CI discovery, or historical rationale because available evidence does not yet identify an owner, observable contract, or bounded first validation beyond the three higher-priority items above.

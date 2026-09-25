---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "65810326-0542-45ae-8f47-25b9ca34cf61"
prompt: |
  @corroborate

  Use docs/investigations/play-evidence-map.md.
  Put the corroboration record in docs/investigations/play-corroboration.md.

  Do not propose or make code changes.
started: "2026-09-24"
ended: "2026-09-24"
task_durations:
  - task: "play workflow corroboration"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/65810326-0542-45ae-8f47-25b9ca34cf61/conversation.md"
source: "software-archaeology-prompts.md"
---

# Play Workflow Corroboration

## Site Overview

**Input**: [Play workflow evidence map](play-evidence-map.md).

**Purpose**: Test the map's highest-risk claims against the current queue consumers, call lifecycle, direct callers, failure handlers, and available Git history.

**Change policy**: No code changes are proposed or made. The conclusions below are evidence updates and focused runtime checks only.

## Claim Results

| Claim from evidence map                                               | Result                               | Classification    | Confidence | Corroborating evidence or contradiction                                                                                                                                                                                                                                                                      |
| --------------------------------------------------------------------- | ------------------------------------ | ----------------- | ---------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Queue records are a cross-module protocol.                            | Corroborated                         | Fact              |       High | `put_queue`/`put_queue_index` write fields consumed by `Aviax.change_stream`, admin callbacks, cleanup, and display code. The consumer branches on `file`, `vidid`, `streamtype`, `chat_id`, `dur`, `seconds`, and `played`.                                                                                 |
| Force-play stops the current stream before inserting the replacement. | Corroborated with ordering risk      | Fact              |       High | `stream` calls `Aviax.force_stop_stream` before selecting a branch. `force_stop_stream` pops the current queue head, removes active state, and leaves the call; subsequent no-active branches skip queue reset when `forceplay` is set and insert the replacement at the front.                              |
| Inactive playlist presentation was unverified.                        | Resolved contradiction               | Fact: defect path |       High | The inactive playlist branch downloads and joins each valid item but does not assign `position`; after the loop, the summary caption formats `position`. A non-empty successful inactive playlist can therefore fail at final presentation with an unbound local before the summary is sent.                 |
| Direct index links depend on a log-group voice probe.                 | Corroborated                         | Fact              |       High | `play_commnd` calls `Aviax.stream_call(url)` before `streamtype="index"`. `stream_call` selects an assistant for `config.LOG_GROUP_ID`, joins that group with the supplied URL, sleeps, and leaves it.                                                                                                       |
| Provider failures collapse into localized/general user errors.        | Corroborated, with silent branches   | Fact              |       High | Provider lookups are wrapped in broad catches that edit `play_3`; `stream` converts YouTube download failures to `AssistantErr(play_14)`; handler catches render `AssistantErr` text or `general_2`. Telegram download returning false and playlist items skipped by `stream` can produce no explicit error. |
| Stream-end behavior depends on queue record shape.                    | Corroborated                         | Fact              |       High | `change_stream` selects live, YouTube, index, Telegram, SoundCloud, or generic handling from queue values, then changes the call and writes new message metadata. Missing or mismatched fields route to failure/cleanup behavior.                                                                            |
| Conventional regression coverage is absent in this snapshot.          | Corroborated for repository contents | Fact              |       High | No conventional test files or `.github/workflows` were found, and the repository-wide test-pattern search found no relevant pytest/unittest setup. External CI or hidden tests remain unknown.                                                                                                               |

## Focused Corroboration

### 1. Queue-Record Consumption

**Observed write contract**:

- `put_queue` writes title, duration, stream type, requester, user ID, original chat, file, video ID, elapsed seconds, and played position; it also appends the file to `autoclean`.
- `put_queue_index` writes an analogous record with an index marker and URL/video ID.
- Force-play inserts at index zero when a queue exists; ordinary playback appends.

**Observed consumer contract**:

- `change_stream` pops or loops the first record, resets playback counters, and branches on `live_`, `vid_`, `index_`, or generic file values.
- Generic records use `vidid` markers `telegram` and `soundcloud` to choose presentation assets; other IDs use generated thumbnails.
- Admin skip/seek/speed paths read the same queue record fields.

**Corroboration**: Strong. The queue record is an implicit protocol, not an isolated implementation detail. A change to marker values, field names, or stream-type encoding can preserve initial `/play` success while breaking the next-track transition or control commands.

**Limit**: No live voice call was exercised, so event timing and PyTgCalls callback behavior remain unverified.

### 2. Force-Play Ordering

**Observed order**:

1. `stream` sees `forceplay` and calls `Aviax.force_stop_stream`.
2. `force_stop_stream` attempts to pop the current head, removes active video/chat state, and attempts to leave the call.
3. The selected branch starts a new call when the chat is no longer active.
4. `put_queue(..., forceplay=forceplay)` inserts the replacement at the front rather than appending.

**Corroboration**: Strong for intended ordering. The replacement is deliberately made the new head after the prior head is removed.

**Fragility**: If stopping the old call or clearing state is only partially successful, the subsequent `join_call` and queue insertion can observe a mixed call/state condition. `force_stop_stream` suppresses several errors, so the caller receives no detailed indication of partial cleanup.

### 3. Inactive Playlist Path

**Observed contradiction**:

- In the active-chat playlist branch, each accepted item assigns `position` after queue insertion.
- In the inactive-chat playlist branch, each accepted item joins the call and queues the item but does not assign `position`.
- After iteration, when `count > 0`, the function formats `play_21` with `position`.

**Corroboration**: This is a confirmed static failure path for a non-empty playlist played into an initially inactive chat, subject to the provider returning at least one valid item. It is stronger than the evidence map's former medium-confidence open inference.

**Limit**: A runtime reproduction was not performed because no live Telegram, Mongo, provider, or voice-call environment was available.

### 4. Direct Index Probe

**Observed order**:

- The handler classifies any URL not claimed by YouTube, Spotify, Apple, Resso, or SoundCloud as an index/M3U8 candidate.
- Before calling the target-chat `stream` branch, it probes the URL in `config.LOG_GROUP_ID` through `Aviax.stream_call`.
- `stream_call` joins the log-group call, waits briefly, and leaves it.
- Only after that probe succeeds does the handler call `streamtype="index"` for the requested chat.

**Corroboration**: Strong. The probe is an externally visible voice-call dependency and can reject or delay an otherwise valid target-chat request. `LOG_GROUP_ID`, assistant assignment, and log-group call availability are therefore part of the direct-link path.

### 5. Provider and Failure Messaging

**Observed layers**:

- Provider lookup exceptions in `play_commnd` generally become `play_3`.
- SoundCloud duration violations become `play_6`.
- YouTube download failures inside `stream` become `AssistantErr(play_14)` and are rendered by the caller.
- Non-`AssistantErr` exceptions are reduced to the exception type through `general_2`.
- Playlist detail failures are skipped inside the loop; an all-invalid playlist returns without a summary or explicit failure message.
- Telegram download returning false also returns without editing or deleting the progress message.

**Corroboration**: Strong for the error taxonomy and weaker for user-visible timing because Telegram/provider APIs were not exercised. The map should distinguish explicit localized errors from silent/no-op failure paths.

## Contradictions and Reconciliations

1. **Inactive playlist “open question” -> confirmed static defect path**: The variable-use path is visible in one function; only runtime provider conditions remain unknown.
2. **`live` as a stream branch -> callback-only reachability**: `play_commnd` does not assign `streamtype="live"`; `AviaxMusic/plugins/play/live.py::play_live_stream` does. The branch belongs to the broader `/play` interaction workflow, not the initial command path.
3. **“No retry” -> differentiated behavior**: Provider lookup/download paths generally do not retry, but the stream-end handler does attempt the next queued item after a prior track ends. This is continuation, not retry of the failed provider operation.
4. **“Force-play clears the queue” -> more precise behavior**: `force_stop_stream` removes only the current head, while force-play insertion preserves remaining queued items and places the replacement at the front. The map's phrase “removes the current queue head and leaves the call” is supported; “clears the queue” would be inaccurate for force-play.

## History Corroboration

- `git log --follow` keeps `play.py`, `decorators/play.py`, and `stream.py` at the `3315863` `v3.0 Stable` baseline in the available history.
- The latest bounded change is `e92667a` (`Update call.py`, 2023-11-19). It changes stream-end Telegram/SoundCloud presentation from `config.SUPPORT_CHAT` to `config.SUPPORT_GROUP` without documenting the reason beyond `hmm done`.
- History supports long-lived coupling in `call.py`, but it does not establish that current behavior is intentional or tested.

## Evidence Ledger

| Finding                                                                                                        | Classification | Confidence | What would still disconfirm or narrow it                                                                                                                            |
| -------------------------------------------------------------------------------------------------------------- | -------------- | ---------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Queue schema is load-bearing across playback and controls.                                                     | Fact           |       High | A complete search would need to find no additional consumers outside the bounded modules; current search found admin and cleanup consumers.                         |
| Force-play ordering is intentional but cleanup is error-suppressing.                                           | Fact           |       High | Instrument stop/join state transitions during a real force-play request.                                                                                            |
| Non-empty inactive playlists can fail at summary formatting because `position` is not assigned on that branch. | Fact           |       High | Execute the branch with a valid provider result and observe whether an alternate local binding exists at runtime; static control flow predicts `UnboundLocalError`. |
| Direct index URLs have a log-group probe dependency before target playback.                                    | Fact           |       High | Run with a valid index URL and unavailable log-group call; observe whether the handler reaches target `stream`.                                                     |
| Failure UX is inconsistent: some failures are localized and some are silent.                                   | Fact           |       High | Exercise provider lookup, download false, all-invalid playlist, and stream exception cases with captured Telegram responses.                                        |
| Runtime call/event timing remains uncorroborated.                                                              | Open question  |       High | Use a controlled test chat with PyTgCalls and inspect event order/state after join, force-stop, and stream end.                                                     |

## Deferred Checks

These are evidence checks only, not implementation proposals:

- Reproduce an inactive non-empty playlist with a controlled provider result and capture the final response/error.
- Observe force-play queue and call state before stop, after stop, after join, and after front insertion.
- Run one stream-end transition for each queue record family: `vid_`, `live_`, `index_url`, Telegram, SoundCloud, and generic local file.
- Exercise direct index playback with log-group voice availability both on and off.
- Capture user-visible output for provider exception, download-false, all-invalid playlist, and stream exception cases.

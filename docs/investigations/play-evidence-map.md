---
ai_generated: true
model: "anthropic/claude-3.5-sonnet@2024-10-22"
operator: "johnmillerATcodemag-com"
chat_id: "65810326-0542-45ae-8f47-25b9ca34cf61"
prompt: |
  @excavate

  Use docs/investigations/source-boundary.md to investigate the MusicBot `/play`
  workflow. Put the evidence map in docs/investigations/play-evidence-map.md.

  Do not propose or make code changes.
started: "2026-09-24"
ended: "2026-09-24"
task_durations:
  - task: "play workflow excavation"
    duration: "00:00:00"
total_duration: "00:00:00"
ai_log: "ai-logs/2026/09/24/65810326-0542-45ae-8f47-25b9ca34cf61/conversation.md"
source: "software-archaeology-prompts.md"
---

# Play Workflow Evidence Map

## Site Overview

**Question**: What user-facing, operational, and integration behavior could regress if the MusicBot `/play` workflow changes?

**Boundary examined**: Startup and plugin registration, `/play` wrapper gates, command parsing and provider selection, stream/queue/call effects, playback state, configuration, and user-visible success/failure signals. Callback-only continuations are mapped where they re-enter shared playback code, but are not treated as direct command invocation.

**Stop condition for this pass**: The bounded execution path and its direct side effects are mapped, each reachable input family has a success and failure path, shared state transitions are identified, and history/test evidence has been checked. Deeper corroboration of runtime behavior remains the next pass.

**Change policy**: No code changes are proposed or made by this investigation.

## Entry and Registration Chain

1. **Fact**: `AviaxMusic/__main__.py::init` starts the bot client and imports every module in `ALL_MODULES`.
2. **Fact**: `AviaxMusic/plugins/__init__.py::__list_all_modules` discovers Python files one directory below `AviaxMusic/plugins`, so `plugins/play/play.py` is imported.
3. **Fact**: Importing `AviaxMusic/plugins/play/play.py` registers one handler for `play`, `vplay`, `cplay`, `cvplay`, and four force-play variants, restricted to group messages and excluding `BANNED_USERS`.
4. **Fact**: `PlayWrapper` wraps `play_commnd`; all wrapper return branches prevent the handler from reaching provider lookup or stream setup.

## Request Gate Map: `PlayWrapper`

| Stage               | Success path                                                                                                                  | Early termination / external effect                                                                                                                | State or dependency                                                  |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Language/context    | Loads localized strings for the chat.                                                                                         | `sender_chat` requests receive an anonymous-admin explanation and callback button.                                                                 | `get_lang`, `strings.get_string`, Telegram reply.                    |
| Maintenance         | SUDOERS bypasses maintenance rejection.                                                                                       | Non-SUDO users receive a maintenance message.                                                                                                      | `is_maintenance`, `SUDOERS`, `SUPPORT_GROUP`.                        |
| Command cleanup     | Attempts to delete the command message, ignoring deletion errors.                                                             | No hard failure if deletion is denied.                                                                                                             | Telegram message deletion.                                           |
| Input extraction    | Detects replied audio/video/document or a URL.                                                                                | With no media, URL, or argument, returns a playlist/search prompt; `stream` command gets a distinct response.                                      | `YouTube.url`, message reply fields, `PLAYLIST_IMG_URL`.             |
| Channel routing     | Normal commands target the source group.                                                                                      | `c*` commands require configured channel mode and accessible channel; otherwise return settings, chat, or permission errors.                       | `get_cmode`, `app.get_chat`, `channelconnect` persistence.           |
| Authorization       | Everyone mode allows the request; configured admin mode checks `adminlist` or SUDOERS.                                        | Non-admin users receive an authorization response.                                                                                                 | `get_playtype`, `adminlist`, `SUDOERS`.                              |
| Playback mode       | Resolves Direct/queued mode and audio/video/force flags.                                                                      | Force-play requires an active chat; inactive chats receive an error.                                                                               | `get_playmode`, command spelling, `-v`, `is_active_chat`.            |
| Assistant readiness | Existing assistant membership is accepted.                                                                                    | Banned/restricted assistant, missing permissions, failed invite/export, join request, or peer resolution can terminate with localized call errors. | `get_assistant`, Pyrogram membership/invite APIs, in-memory `links`. |
| Handler dispatch    | Calls `play_commnd` with localized context, target chat, video flag, channel label, play mode, extracted URL, and force flag. | Any earlier return prevents the command-specific workflow.                                                                                         | Contract between `PlayWrapper` and `play_commnd`.                    |

## Command Handler Map: `play_commnd`

The handler creates a progress message first. It then chooses one mutually exclusive input family.

### Replied Telegram Audio

- **Lookup**: Reads reply audio or voice.
- **Guards**: Rejects files over 100 MB or longer than `DURATION_LIMIT`.
- **Effects**: Computes a local path, downloads through `Telegram.download`, obtains a message link/name/duration, and builds a details record.
- **Playback**: Calls `stream(..., streamtype="telegram")`; after success deletes the progress message.
- **Failure**: Download false returns silently; stream exceptions become `AssistantErr` text or a general exception label; duration/size failures edit the progress message.
- **Risk boundary**: Telegram file helpers, local playback files, and the shared `stream` contract are load-bearing.

### Replied Telegram Video or Document

- **Lookup**: Reads reply video or document.
- **Guards**: Documents must have an allowed extension from `formats`; size is checked against `TG_VIDEO_FILESIZE_LIMIT`.
- **Effects and failure**: Same download/details/stream pattern as audio, with `video=True` and `streamtype="telegram"`; stream failures edit the progress message.
- **Risk boundary**: Extension filtering, configured byte limit, video flag, active-video state, and Telegram presentation assets all affect behavior.

### Explicit URL

The wrapper supplies a URL, then the handler selects the first matching provider.

| Provider/input                | Lookup and preparation                                                                                                           | Direct stream handoff                                                                 | Failure behavior                                                                                                                         |
| ----------------------------- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| YouTube playlist              | `YouTube.playlist` with `PLAYLIST_FETCH_LIMIT`; sets `streamtype="playlist"`, `plist_type="yt"`, and playlist presentation data. | In Direct mode, `stream` receives playlist data.                                      | Provider errors edit `play_3`.                                                                                                           |
| YouTube track                 | `YouTube.track`; sets `streamtype="youtube"` and track presentation.                                                             | `stream` downloads/queues or starts playback.                                         | Provider errors edit `play_3`.                                                                                                           |
| Spotify track                 | Requires both Spotify credentials; `Spotify.track`; reuses `streamtype="youtube"`.                                               | Shared YouTube stream branch.                                                         | Unsupported credentials or provider errors produce user-facing rejection.                                                                |
| Spotify playlist/album/artist | Resolves provider data and stores playlist type/id for a later callback.                                                         | Direct mode sends playlist data to `stream`; non-Direct mode sends `playlist_markup`. | Provider errors edit `play_3`; unknown Spotify subtype edits `play_15`.                                                                  |
| Apple album                   | `Apple.track`; reuses `streamtype="youtube"`.                                                                                    | Shared YouTube stream branch.                                                         | Provider errors edit `play_3`.                                                                                                           |
| Apple playlist                | `Apple.playlist`; stores `plist_type="apple"` for playlist handling.                                                             | Direct mode can stream data; non-Direct mode presents playlist markup.                | Provider errors edit `play_3`.                                                                                                           |
| Resso track                   | `Resso.track`; reuses `streamtype="youtube"`.                                                                                    | Shared YouTube stream branch.                                                         | Provider errors edit `play_3`.                                                                                                           |
| SoundCloud track              | `SoundCloud.download`; checks duration against `DURATION_LIMIT`.                                                                 | Calls `stream(..., streamtype="soundcloud")`.                                         | Download/duration/stream errors edit the progress message.                                                                               |
| Other URL/index link          | Calls `Aviax.stream_call` against `LOG_GROUP_ID` as a probe, then calls `stream(..., streamtype="index")`.                       | `stream` either queues the URL or starts it directly.                                 | `NoActiveGroupCall` edits the progress message and sends a log-group warning; other probe/stream errors become localized/general errors. |

**Observed coupling**: A direct index URL must first succeed against the configured log-group call probe before the target chat's `stream` branch can run. This makes `LOG_GROUP_ID`, assistant assignment, and voice-call availability part of the apparent `/play` URL contract.

### Search Query Without URL

- If the message has no second argument, the handler returns the playlist/search prompt.
- Otherwise it strips `-v`, calls `YouTube.track(query)`, sets `streamtype="youtube"`, and continues into Direct or non-Direct mode.
- Direct mode validates duration, handles a missing duration by presenting `livestream_markup`, then calls `stream` and logs the play.
- Non-Direct mode presents `slider_markup` for a search result rather than mutating playback immediately.

### Direct Versus Non-Direct Mode

- **Direct**: Validates duration, rejects over-limit tracks, presents a live-stream choice when duration is absent, calls `stream`, deletes the progress message, and calls `play_logs`.
- **Non-Direct playlist**: Stores a random key in `config.lyrical`, deletes progress, presents playlist markup, and waits for a callback.
- **Non-Direct search/track**: Deletes progress, presents slider or track markup, and waits for a callback.
- **Callback continuation**: `play_music` and `play_playlists_command` re-resolve YouTube/provider data and call `stream`; `live.py::play_live_stream` is the only observed caller that supplies `streamtype="live"`.

## Shared Playback Map: `stream`

`stream` first returns on an empty result. If `forceplay` is set, it calls `Aviax.force_stop_stream(chat_id)` before selecting the branch.

| `streamtype` | Active chat                                                                                                                            | No active chat                                                                                                                                              | Shared state and presentation                                                                                                   |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `playlist`   | Iterates results, resolves YouTube details, skips unavailable/unknown/over-limit tracks, and appends each valid item with `put_queue`. | Downloads each valid track, joins the call, then appends its queue record.                                                                                  | Uses `db`, `PLAYLIST_FETCH_LIMIT`, `DURATION_LIMIT`, thumbnails, stream markup, Carbon/Pastebin summary, and per-item metadata. |
| `youtube`    | Downloads before queueing; appends a record and sends a queue-position message.                                                        | Clears `db` unless force-play, joins the call, appends the record, sends a now-playing photo, and stores the message/markup in `db[chat_id][0]`.            | Enforces a ten-item queue cap only in this branch; download failure raises `AssistantErr(play_14)`.                             |
| `soundcloud` | Appends a local file record and sends a queue-position message.                                                                        | Clears `db` unless force-play, joins the call, appends the record, sends SoundCloud now-playing presentation, and stores message metadata.                  | Uses `streamtype="soundcloud"` and audio-only playback.                                                                         |
| `telegram`   | Appends a local file record and sends a queue-position message.                                                                        | Clears `db` unless force-play, joins the call, appends the record, optionally marks active video, sends Telegram presentation, and stores message metadata. | Video state is updated separately with `add_active_video_chat`.                                                                 |
| `live`       | Appends a `live_<vidid>` record and sends queue status.                                                                                | Resolves a playable file through `YouTube.video`, joins the call, appends the live record, sends now-playing presentation, and stores metadata.             | Direct `/play` does not assign this type; `play/live.py` does. A zero result raises `AssistantErr(str_3)`.                      |
| `index`      | Appends an `index_url` record and edits the existing progress message to queue status.                                                 | Clears `db` unless force-play, joins the URL, appends the record, sends stream presentation, stores metadata, and deletes progress.                         | In the no-active branch it passes video mode based on the request; queue records use `put_queue_index`.                         |

**Queue record contract**: `put_queue` stores title, duration, stream type, requester, original chat, file/vidid, elapsed seconds, and played position in the process-global `db[chat_id]`, while adding the file to `autoclean`. `put_queue_index` stores analogous URL records and may probe a specific IP URL for duration.

**Call contract**: `Aviax.join_call` selects the assistant, constructs audio/video input, joins the group call, translates call errors into `AssistantErr`, then marks active chat/music/video state. `Aviax.force_stop_stream` removes the first queue item, clears active state, and leaves the call. Stream-end/left/kicked decorators in `Aviax.decorators` call `change_stream` or `stop_stream`, so queue metadata written by `/play` is consumed later by call events.

## State, External Effects, and Failure Paths

- **Process state**: `db`, `active`, `activevideo`, `autoclean`, `lyrical`, cached mode/language/channel dictionaries, and `links` are mutated or consulted across the path.
- **Persistent state**: language, channel mode, play mode, play type, assistant assignment, and feature flags are read from Mongo-backed helpers when not cached.
- **Telegram effects**: command deletion, progress replies/edits/deletes, queue messages, now-playing photos, inline keyboards, log messages, invite/export/join/approval operations, and assistant membership checks.
- **Voice effects**: log-group probe for index links, assistant join, force stop/leave, stream-end transition, and active-chat cleanup.
- **Retry behavior**: Most provider/download failures return a localized error without retrying. Queue continuation retries no provider download automatically; stream-end handling instead reports `call_6` on failed next-track resolution/change.
- **Cancellation/cleanup**: force-play removes the current queue head and leaves the call before adding the new item; stream-end and call-left handlers clear state when no queue remains. Downloaded files are registered with `autoclean`, but partial failure timing is shared-state dependent.

## History and Operational Evidence

- **Fact**: `git log --follow` shows `play.py`, `decorators/play.py`, and `stream.py` at the `3315863` `v3.0 Stable` baseline, with no later file-specific commits in the available repository history.
- **Fact**: The latest bounded change is commit `e92667a` (`Update call.py`, 2023-11-19), changing the stream-end presentation from `config.SUPPORT_CHAT` to `config.SUPPORT_GROUP` for Telegram and SoundCloud items. Its message is terse (`hmm done`), so the rationale is not recoverable from commit text.
- **Fact**: No conventional `tests/` or `test*` files were found; a repository-wide pattern search found no pytest/unittest configuration or assertions relevant to this workflow. No `.github/workflows` files were found.
- **Open question**: Runtime telemetry, deployed environment values, and real Pyrogram/PyTgCalls behavior were not available in this excavation.

## Evidence Ledger

| Claim                                                                                                 | Classification | Confidence | Evidence                                                                                                                         | Smallest disconfirming check                                                                                   |
| ----------------------------------------------------------------------------------------------------- | -------------- | ---------: | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| The wrapper is the primary reachability gate for `/play`.                                             | Fact           |       High | `PlayWrapper` early returns before `play_commnd`; handler is decorated by it.                                                    | Run controlled messages through each wrapper return branch.                                                    |
| Multiple input families converge on a small set of `streamtype` contracts.                            | Fact           |       High | `play_commnd` assigns telegram, playlist, youtube, soundcloud, and index; `live.py` assigns live.                                | Instrument all `stream` callers and compare runtime values.                                                    |
| Queue and voice-call side effects are coupled through `db` and active-chat state.                     | Fact           |       High | `stream`, `put_queue*`, `Aviax.join_call`, `change_stream`, and cleanup handlers share those stores.                             | Run one active-chat and one new-call case for each stream type, then inspect queue/state transitions.          |
| A change to queue record shape can break later stream transitions even if `/play` appears successful. | Inference      |       High | `change_stream` branches on `file`, `vidid`, `streamtype`, `chat_id`, and timing fields written by queue helpers.                | Characterize one stream-end transition for each queue record family.                                           |
| Direct index links have an extra log-group voice dependency.                                          | Fact           |       High | `play_commnd` calls `Aviax.stream_call(url)` before `streamtype="index"`; `stream_call` uses `LOG_GROUP_ID`.                     | Exercise a valid index URL with the log group call unavailable.                                                |
| Playlist handling has an unverified inactive-chat presentation path.                                  | Inference      |     Medium | Inactive playlist branch does not assign `position` before final `play_21` formatting, while the final response uses `position`. | Execute a playlist in a chat with no active call and observe whether the final summary raises before response. |
| Regression coverage is weak or absent in this repository snapshot.                                    | Fact           |       High | No test files/workflows and no relevant test configuration were found.                                                           | Check external CI, deployment repository, or hidden test harness.                                              |

## Artifacts and Fragility Found

- **Implicit contract**: Queue records are a cross-module protocol between `stream`, `queue.py`, `call.py`, admin controls, and cleanup. The protocol is not declared as a schema or type.
- **Implicit routing**: Provider-specific items are represented by generic values such as `vid_<id>`, `live_<id>`, `index_url`, and `vidid` markers (`telegram`, `soundcloud`).
- **Shared mutable state**: Process-local dictionaries/lists determine active chats, queue contents, cleanup, playlist callback IDs, cached settings, and assistant invite links.
- **Broad exception handling**: Many branches collapse provider, network, formatting, and state errors into one localized error or silent return, making failure cause difficult to observe.
- **History limitation**: The available Git history explains one presentation fix but does not document the original `/play` branch contracts.

## Deferred Corroboration

This map does not propose implementation changes. The next investigation should corroborate the highest-risk contracts with focused runtime or characterization evidence: queue-record consumption on stream end, force-play ordering, inactive playlist handling, direct index probe dependency, and provider failure messaging.

---
ai_generated: true
model: "anthropic/claude-sonnet-4-5@2025-02-19"
operator: "johnmillerATcodemag-com"
chat_id: "extract-software-archaeology-prompts-20260923"
prompt: |
  extract all of the prompts into a software-archaeology-prompts.md file. for each prompt
  include the slide number, the slide title and the prompt
started: "2026-09-23T01:00:00Z"
ended: "2026-09-23T01:15:00Z"
task_durations:
  - task: "prompt extraction and file assembly"
    duration: "00:15:00"
total_duration: "00:15:00"
ai_log: "ai-logs/2026/09/23/extract-software-archaeology-prompts-20260923/conversation.md"
source: "johnmillerATcodemag-com"
---

# Prompt Library: Software Archaeology with AI

Every `chat prompt` used in [`slides/marp/software-archaeology-with-ai.deck.md`](../slides/marp/software-archaeology-with-ai.deck.md), extracted in slide order. Slide numbers count each slide separator (`---`) in the merged deck, starting at 1 for the title slide.

## Index

| Slide | Title                                             |
| ----- | ------------------------------------------------- |
| 9     | Pass 1: Orient the Investigation                  |
| 14    | Prompts That Produce Reviewable Evidence          |
| 15    | Example Prompt: Map an Unfamiliar Repository      |
| 16    | Example Prompt: Trace a Behavior Safely           |
| 17    | Example Prompt: Recover Contracts and Rationale   |
| 27    | MusicBot Case Study: Ask for an Evidence Map      |
| 28    | MusicBot Case Study: Verify the Control Path      |
| 29    | MusicBot Case Study: Record the Ledger            |
| 30    | MusicBot Case Study: Shape the First Backlog Item |
| 32    | Demo Step 2: Build the First Evidence Map         |
| 33    | Demo Step 3: Trace Effects and Contradictions     |
| 34    | Demo Step 4: Produce the First Backlog Item       |

---

## Slide 9 — Pass 1: Orient the Investigation

```prompt
Map the repository areas relevant to: What could break if we change how the MusicBot handles /play?. Cite each file path and symbol. Separate direct evidence from inference. List unknowns and the next three files to inspect.
```

## Slide 14 — Prompts That Produce Reviewable Evidence

```prompt
Investigate what could break if the MusicBot changes how `/play` hands provider results to the stream layer.

Boundary: `c:/git/MusicBot/AviaxMusic/__main__.py`, `core/bot.py`, `plugins/play/play.py`, `utils/decorators/play.py`, `utils/stream/stream.py`, `utils/stream/queue.py`, and `core/call.py`.

For each claim:
1. Cite supporting files, symbols, tests, configuration, or history.
2. Mark it as fact, inference, or open question.
3. State the impact if the claim is wrong.
4. Propose the smallest next investigation step.

Do not propose code changes until the evidence summary is complete.
```

## Slide 15 — Example Prompt: Map an Unfamiliar Repository

```prompt
I need to understand the MusicBot `/play` workflow before changing it.

Start with `AviaxMusic/__main__.py` and identify how `ALL_MODULES` loads `plugins/play/play.py`. Then map `PlayWrapper`, `play_commnd`, the provider platforms, `stream`, queue mutation, and the voice-call handoff. Include `config.py`, deployment files, tests, and ownership documentation relevant to it.

Return an evidence map with file paths and symbols. Mark each statement as fact, inference, or open question. Do not recommend implementation changes yet.
```

## Slide 16 — Example Prompt: Trace a Behavior Safely

```prompt
Trace a group user's `/play Never Gonna Give You Up`, `/play https://www.youtube.com/watch?v=BaW_jenozKc`, replied Telegram audio, and `/playforce https://www.youtube.com/watch?v=BaW_jenozKc` requests from `PlayWrapper` and `play_commnd` to every state change and external side effect in the MusicBot. Follow the branches into `stream`, `put_queue`, `db`, and `Aviax.join_call`.

For each branch, cite the file and symbol, identify success/failure/retry behavior, and name the tests or configuration that corroborate it. Separate direct evidence from inference. End with the highest-risk unknown and the smallest next check.
```

## Slide 17 — Example Prompt: Recover Contracts and Rationale

```prompt
Investigate the contract and history around the MusicBot `/play` handoff: `PlayWrapper` -> `play_commnd` -> `stream` -> `put_queue`/`Aviax.join_call`.

Find callers and callback consumers for `MusicStream`, `AviaxPlaylists`, and `LiveStream`; inspect `config.py`, `database.py`, `core/call.py`, tests, recent
history, and operational references. Cite evidence for each conclusion. Identify undocumented assumptions, compatibility risks, and evidence gaps. Propose
investigation items, not a rewrite plan, for anything still uncertain.
```

## Slide 27 — MusicBot Case Study: Ask for an Evidence Map

```prompt
@excavate

Investigate the /play command in the MusicBot repository.

Boundary: AviaxMusic/__main__.py, AviaxMusic/core/bot.py, AviaxMusic/plugins/play/play.py, and the stream layer. Stop when you can map startup, plugin registration, command handling, state changes, external effects, tests, configuration, and unknowns.

For every claim, cite a file and symbol. Label it fact, inference, or open question. Do not propose code changes yet.
```

## Slide 28 — MusicBot Case Study: Verify the Control Path

```prompt
Corroborate the /play evidence map against utils/stream/stream.py, config.py, deployment configuration, and available tests. For each of the four claims above, cite the exact file and symbol, label it fact, inference, or open question, and state what observation would disprove it. Identify the smallest next check.
```

## Slide 29 — MusicBot Case Study: Record the Ledger

```prompt
Consolidate the verified /play findings into an evidence ledger with columns Claim, Evidence, Confidence, and Next Check. Label each claim fact, inference, or open question, matching your Evidence Ledger response format.
```

## Slide 30 — MusicBot Case Study: Shape the First Backlog Item

```prompt
@shape-backlog

Using the verified MusicBot evidence ledger, propose no more than three modernization or investigation items. For each, include value, risk reduced, evidence confidence, first move, owner question, and the claim it validates. Prefer a characterization test, telemetry, or explicit contract before a large refactor. Do not implement the item.
```

## Slide 32 — Demo Step 2: Build the First Evidence Map

```prompt
@excavate

Investigate the /play command in the MusicBot repository.

Boundary: AviaxMusic/__main__.py, AviaxMusic/core/bot.py, AviaxMusic/plugins/play/play.py, and the stream layer. Stop when you can map startup, plugin registration, command handling, state changes, external effects, tests, configuration, and unknowns.

For every claim, cite a file and symbol. Label it fact, inference, or open question. Do not propose code changes yet.
```

## Slide 33 — Demo Step 3: Trace Effects and Contradictions

```prompt
Corroborate the /play evidence map against utils/stream/stream.py, config.py, deployment configuration, and available tests. For each claim so far, cite the exact file and symbol, label it fact, inference, or open question, and state what observation would disprove it. Flag any contradiction between the code, tests, and configuration instead of resolving it by assumption. Identify the smallest next check.
```

## Slide 34 — Demo Step 4: Produce the First Backlog Item

```prompt
@shape-backlog

Using the verified /play evidence ledger, propose no more than three modernization or investigation items. For each, include value, risk reduced, evidence confidence, first move, owner question, and the claim it validates. Prefer a characterization test, telemetry, or explicit contract before a large refactor. Do not implement the item.
```

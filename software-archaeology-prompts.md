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

## Slide 31 — Step 1: Frame the Question

```prompt
Investigate what could break if we change the MusicBot /play workflow. Start by validating the question, boundary, and stop condition. Put the response in the file docs/investigations/play-workflow.md. Do not inspect code yet.
```

## Slide 32 — Step 2: Locate the Source Boundary

```prompt
@locate-source-boundary
Use the Investigation Framing in docs/investigations/play-workflow.md. Put the response in the file docs/investigations/source-boundary.md
```

## Slide 33 — Step 3: Build the First Evidence Map

```prompt
@excavate

Use docs/investigations/source-boundary.md to investigate the MusicBot `/play`
workflow. Put the evidence map in docs/investigations/play-evidence-map.md.

Do not propose or make code changes.
```

## Slide 34 — Demo Step 4: Trace Effects and Contradictions

```prompt
@corroborate

Use docs/investigations/play-evidence-map.md.
Put the corroboration record in docs/investigations/play-corroboration.md.

Do not propose or make code changes.
```

## Slide 35 — Demo Step 5: Produce the First Backlog Item

```prompt
@shape-backlog

Use docs/investigations/play-evidence-map.md and
docs/investigations/play-corroboration.md.

Put the prioritized backlog in docs/investigations/play-backlog.md.
Do not implement changes.

---
name: taskeval
description: Evaluate completed tasks with Grok and manage agent/job-context improvement proposals
user-invocable: true
---

# Purpose

Use this skill to review a meaningful completed task with Grok and store precise
improvement proposals for either:

- `agent_context`
- `job_context`

This skill uses Grok through `x_search`, stores proposals in workspace files,
and exposes a single command namespace:

- `/taskeval`
- `/taskeval run`
- `/taskeval list`
- `/taskeval list pending|approved|denied|all`
- `/taskeval show <id>`
- `/taskeval approve <id>`
- `/taskeval deny <id>`

## Core rules

- Evaluate only after a meaningful completed task.
- Do not evaluate read-only checks, status lookups, or trivial chat.
- Use `x_search` as the Grok backend.
- Read the agent bootstrap files separately. Never compact them into a single
  synthetic agent blob before asking Grok.
- Grok must reason about `job_goal` and `job_context`, but final proposals may
  only use the scopes `agent_context` or `job_context`.
- A single review may not contain:
  - more than one `agent_context` proposal for the same `target_file`
  - more than one `job_context` proposal
- `raw` must always be a full replacement:
  - full file content for `agent_context`
  - full replacement for `payload.message` for `job_context`
- Never keep a second tracking system. The only source of truth is the folder
  state in:
  - `<workspace>/task_evaluation_pending/`
  - `<workspace>/task_evaluation_approved/`
  - `<workspace>/task_evaluation_denied/`

## Command behavior

Treat the skill input as follows:

- empty input -> `run`
- `run` -> launch a new evaluation
- `list` -> same as `list pending`
- `list pending|approved|denied|all` -> list proposals
- `show <id>` -> display one proposal in full detail
- `approve <id>` -> move a pending proposal to approved, apply the approved
  change immediately via OpenClaw tools, then purge other pending proposals for
  the same logical subject
- `deny <id>` -> move a pending proposal to denied, nothing else

If the command is invalid, print the supported syntax and stop.

## Files and directories

Use these workspace-relative directories:

```text
task_evaluation_pending/
task_evaluation_approved/
task_evaluation_denied/
```

Create them if they do not exist.

Proposal filenames are numeric only:

```text
1.md
2.md
3.md
```

IDs are global and monotonic across all three directories. To allocate the next
ID, scan every `.md` file in the three directories, parse the numeric filename,
and use `max + 1`. Never reuse an old ID.

## Proposal file format

Each proposal is stored as:

```markdown
---
id: 12
createdAt: 2026-04-01T14:32:10Z
scope: agent_context
target_file: AGENTS.md
title: Clarify escalation boundary
effort: medium
jobId: weekly-status
---

## Description
<description>

## Why
<why>

## Job Goal
<analysis.job_goal>

## Job Context
<analysis.job_context>

## Raw
<raw full replacement content>
```

Rules:

- `target_file` must be one of `AGENTS.md`, `SOUL.md`, `TOOLS.md`,
  `IDENTITY.md`, `USER.md`, or `null`.
- For `job_context`, store `target_file: null`.
- Do not store `status` in the file. The containing folder is the status.

## Run flow

### 1. Guardrail on pending count

Count `.md` files in `task_evaluation_pending/`.

If the count is `>= 30`, stop immediately and reply with this exact sentence:

```text
Nombre maximal de propositions atteinte, tape /taskeval list pour voir la liste
```

Do not call Grok in that case.

### 2. Read agent context separately

Read these files separately when they exist:

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`

Do not merge or summarize them before the prompt. Preserve file boundaries in
the Grok request.

### 3. Resolve job context

If a `jobId` for the completed task is known, read the job definition from:

```text
~/.openclaw/cron/jobs.json
```

Use the matching entry's `payload` as the job source of truth.

When a job is found:

- include the raw job payload in the Grok prompt
- focus `job_context` proposals on `payload.message`

When `jobId` is unknown or the job cannot be found:

- still allow `agent_context` proposals
- do not allow Grok proposals with scope `job_context`

### 4. Gather task result

Build a concise task summary from the just-finished work:

- task description
- task result or failure summary
- changed files if known

This summary is evaluation input, not the thing being rewritten.

### 5. Ask Grok

Call `x_search` with a prompt that instructs Grok to return JSON only.

Use this structure and intent exactly:

```text
You are reviewing an OpenClaw agent after a completed task.

You must analyze:
1. agent context
2. job goal
3. job context
4. the task result

Definitions:
- job_goal = without technical details, what do we want to obtain?
- job_context = by what mechanism, steps, or payload instructions is the job trying to obtain it?

Important:
- job_goal is analysis only and must never appear as a final proposal scope.
- Final proposal scopes are only: agent_context or job_context.
- agent_context means improving exactly one of these files:
  AGENTS.md, SOUL.md, TOOLS.md, IDENTITY.md, USER.md
- job_context means improving only the full payload.message of the current job.
- For agent_context, target_file is required.
- For job_context, target_file must be null.
- raw must contain the full replacement content, never a partial patch.
- Be selective and relevant. Do not produce filler ideas.
- In one review:
  - do not output more than one agent_context proposal for the same target_file
  - do not output more than one job_context proposal
- If job data is missing, do not output any job_context proposal.
- Return JSON only. No markdown fences. No commentary.

Required JSON:
{
  "analysis": {
    "job_goal": "what we want, without technical details",
    "job_context": "how the payload tries to achieve it"
  },
  "improvements": [
    {
      "scope": "agent_context | job_context",
      "target_file": "AGENTS.md | SOUL.md | TOOLS.md | IDENTITY.md | USER.md | null",
      "title": "short title",
      "description": "precise change needed",
      "why": "why it matters",
      "effort": "low | medium | high",
      "raw": "full replacement content"
    }
  ]
}

Context files are provided below as separate sections.
```

Then append:

- one section per agent file
- one section for the raw job payload, if available
- one section for the recent task summary

### 6. Validate and normalize Grok output

Parse the reply as JSON.

If the reply is not valid JSON, stop and report that Grok returned an invalid
response. Do not create any files.

Validate:

- `analysis.job_goal` is a string
- `analysis.job_context` is a string
- `improvements` is an array
- each item has:
  - `scope`
  - `title`
  - `description`
  - `why`
  - `effort`
  - `raw`
- `scope` must be `agent_context` or `job_context`
- `effort` must be `low`, `medium`, or `high`
- for `agent_context`, `target_file` must be one of the allowed files
- for `job_context`, `target_file` must be `null`

Then deduplicate:

- keep only the first proposal per `target_file` for `agent_context`
- keep only the first `job_context` proposal

Then enforce the storage cap:

- available slots = `30 - pending_count`
- store only up to the number of available slots
- ignore the rest

### 7. Write pending proposals

For each retained proposal:

- allocate the next numeric ID
- write `<workspace>/task_evaluation_pending/<id>.md`
- include:
  - frontmatter
  - `Description`
  - `Why`
  - `Job Goal`
  - `Job Context`
  - `Raw`

### 8. Report the result

If no valid proposal remains after validation and deduplication, reply clearly
that Grok did not return any storable improvement proposal.

If proposals were stored, print a short summary and tell the user to use:

- `/taskeval list`
- `/taskeval show <id>`
- `/taskeval approve <id>`
- `/taskeval deny <id>`

## List flow

Supported filters:

- `pending`
- `approved`
- `denied`
- `all`

Default filter is `pending`.

Read proposal files from the relevant directories, parse their frontmatter, and
sort them from newest to oldest using:

1. `createdAt` descending
2. numeric `id` descending as fallback

Display a compact table-like list with one line per proposal:

```text
ID | date | scope | target | effort | title
```

For `all`, include the folder status at the start of each line.

If no proposal matches, say so clearly.

## Show flow

`show <id>` must search the three directories.

If found, display:

- status derived from the folder name
- id
- createdAt
- scope
- target_file
- effort
- jobId
- title
- Description
- Why
- Job Goal
- Job Context
- Raw

If the ID does not exist, say that no proposal with that ID was found.

## Approve flow

`approve <id>` only works on pending proposals.

Steps:

1. Find `task_evaluation_pending/<id>.md`
2. Move it to `task_evaluation_approved/<id>.md`
3. Apply the approved change immediately using direct OpenClaw tool calls
4. Delete the other files still present in `task_evaluation_pending/` that
   refer to the same logical subject
5. Report whether the real application succeeded or failed
6. Do not rewrite the proposal file contents during approval

Application rules:

- For `job_context`, call `cron.update` directly with this exact payload shape:

```json
{
  "jobId": "<jobId>",
  "patch": {
    "payload": {
      "message": "<raw>"
    }
  }
}
```

- For `agent_context`, call the `write` tool directly:
  - the target workspace is the current workspace, meaning the workspace where
    `task_evaluation_pending/<id>.md` was found
  - path: `<current_workspace>/<target_file>`
  - content: exact `raw` value as the full replacement content

- Do not use `exec` to emulate `cron.update` or `write`.
- Do not present command-line text as the primary `/approve` behavior.
- `approved/` means the human approval decision is final even if the technical
  application fails.
- If the application fails, keep the file in `approved/` anyway and still apply
  the pending purge rules.

Purge rules:

- For `agent_context`, the logical subject is `target_file`
- For `job_context`, the logical subject is `jobId`
- Only purge files from `task_evaluation_pending/`
- Never delete the file that was just moved to `task_evaluation_approved/`
- Never delete anything from `task_evaluation_approved/` or
  `task_evaluation_denied/`
- For `job_context`, if `jobId` is missing, do not purge any sibling pending
  file because the logical subject cannot be matched safely

Failure handling:

- If `job_context` is missing `jobId`, archive to `approved/`, skip real
  application, keep sibling pending proposals untouched, and report the failure
- If `agent_context` cannot resolve the target path or the `write` call fails,
  archive to `approved/`, still purge same-subject pending proposals, and
  report the failure reason
- If `cron.update` fails, archive to `approved/`, still purge same-subject
  pending proposals, and report the failure reason

## Deny flow

`deny <id>` only works on pending proposals.

Steps:

1. Find `task_evaluation_pending/<id>.md`
2. Move it to `task_evaluation_denied/<id>.md`
3. Confirm the archive action

Do not do anything else.

## Safety and boundaries

- Only `approve` may auto-apply the approved proposal.
- Only `approve` may purge sibling proposals, and only from
  `task_evaluation_pending/`.
- During `approve`, use direct tool calls for `cron.update` and `write`; do not
  emulate them through `exec`.
- Never propose `job_context` when the job source cannot be resolved.
- Never generate partial replacements in `raw`.
- Never create more than 30 pending proposals total.
- Keep proposals sharp and non-redundant.

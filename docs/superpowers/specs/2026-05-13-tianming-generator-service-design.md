# Tianming Generator Service Design

## Goal

Turn `tianming-novel-ai-writer` into a Linux-deployable novel generation service that `novel-center` can call asynchronously, without changing `novel-center` business logic.

The first release only supports public chapter generation. Opening-story generation, personal branches, cover/image generation, and direct `novel-center` integration are out of scope for this spec.

## Constraints

- The service must run on Linux.
- The current WPF app remains unchanged and must continue to build as a desktop app.
- The Linux service cannot depend on `net8.0-windows`, WPF, `System.Windows`, `Window`, `Dispatcher`, `GlobalToast`, or UI-only services.
- `novel-center` must not know Tianming internal guide, snapshot, workspace, or CHANGES structures.
- Postgres is the durable job store. Redis is the execution queue and progress channel.

## Architecture

Add two projects under `ServicesHost/`:

```text
ServicesHost/
  Tianming.GeneratorCore/
    Tianming.GeneratorCore.csproj
  Tianming.GeneratorHost/
    Tianming.GeneratorHost.csproj
```

`Tianming.GeneratorCore` is a pure `net8.0` library. It contains Linux-safe generation code, service-facing request/response models, workspace management, CHANGES conversion, and abstraction interfaces for logging, progress, storage paths, and AI calls.

`Tianming.GeneratorHost` is a pure `net8.0` ASP.NET Core service. It exposes HTTP APIs, stores jobs in Postgres, queues work in Redis, and runs a background worker that calls `GeneratorCore`.

The existing `Core/App/天命.csproj` WPF app remains the desktop host for the current application. Shared code should move gradually into `GeneratorCore` only when it is needed by the service and can be made UI-free.

## Service API

### POST `/v1/chapters/generate`

Creates an asynchronous public chapter generation job.

Request:

```json
{
  "external_request_id": "novel-center-job-id",
  "story_id": "story_xxx",
  "story_line_id": "line_xxx",
  "volume_id": "vol_xxx",
  "chapter_id": "vol1_ch12",
  "chapter_no": 12,
  "volume_no": 1,
  "volume_chapter_no": 12,
  "channel": "xiuxian",
  "story": {},
  "volume": {},
  "chapter_plan": {},
  "recent_chapters": [],
  "previous_chapter_tail": "",
  "volume_archives": [],
  "vector_recall_fragments": [],
  "ledger_entity_ids": [],
  "hard_constraints": []
}
```

Response:

```json
{
  "job_id": "tmgen_01HZX7J3K5P8B2Q9R4C6",
  "status": "queued"
}
```

`external_request_id` is idempotent per caller. If the same value is submitted again, the service returns the existing job instead of creating duplicate work.

### GET `/v1/jobs/{job_id}`

Returns durable job state.

Queued or running:

```json
{
  "job_id": "tmgen_01HZX7J3K5P8B2Q9R4C6",
  "status": "running",
  "progress": {
    "stage": "generation",
    "attempt": 1,
    "message": "generating chapter"
  }
}
```

Succeeded:

```json
{
  "job_id": "tmgen_01HZX7J3K5P8B2Q9R4C6",
  "status": "succeeded",
  "result": {
    "title": "第十二章 雨夜旧案",
    "body": "章节正文内容",
    "changes": {},
    "raw_output": "模型原始输出",
    "attempts": 2,
    "warnings": []
  }
}
```

Failed:

```json
{
  "job_id": "tmgen_01HZX7J3K5P8B2Q9R4C6",
  "status": "failed",
  "error": {
    "code": "generation_gate_failed",
    "message": "chapter output did not pass generation gate",
    "retryable": true
  }
}
```

## Job Storage

Postgres table `generation_jobs`:

```text
id text primary key
external_request_id text unique not null
story_id text not null
chapter_id text not null
status text not null
request_json jsonb not null
result_json jsonb null
error_json jsonb null
attempts int not null default 0
created_at timestamptz not null
started_at timestamptz null
finished_at timestamptz null
updated_at timestamptz not null
```

Redis keys:

```text
tmgen:queue:chapter
tmgen:job:{job_id}:progress
tmgen:job:{job_id}:lock
tmgen:worker:{worker_id}:heartbeat
```

Postgres is the source of truth. Redis only controls execution, progress, locks, and worker liveness.

## Generation Flow

1. HTTP API validates the request and inserts a `queued` job into Postgres.
2. API pushes the job id to Redis queue.
3. Worker claims the job with a Redis lock.
4. Worker marks the Postgres job `running`.
5. Worker prepares a per-story Tianming workspace.
6. Worker builds a service-side generation context from the request.
7. Worker calls `GeneratorCore.GeneratePublicChapterAsync`.
8. Core generates the draft, validates it, retries or rewrites as configured.
9. Core converts Tianming CHANGES into `novel-center` compatible `GenerationChanges`.
10. Worker writes `succeeded` result or `failed` error to Postgres.

## Workspace Strategy

Each `story_id` maps to one isolated workspace:

```text
{GENERATOR_WORKSPACE_ROOT}/stories/{story_id}/
  chapters/
  guides/
  indexes/
  archives/
  runtime/
```

The first release may bootstrap missing guides from request data and generated CHANGES. It should not require `novel-center` to provide Tianming-native guide files.

Workspace writes must be per-story serialized with a Redis lock to avoid concurrent chapters corrupting guide, index, or archive state.

## Core Extraction Strategy

Do not directly reference the WPF project from the Linux service. Move only Linux-safe code into `Tianming.GeneratorCore` as needed.

Initial candidates:

- CHANGES models and parsing/canonicalization.
- Fact snapshot models.
- `LayeredPromptBuilder` after replacing UI dependencies.
- `GenerationGate` after replacing `TM.App.Log`.
- `AutoRewriteEngine` after replacing `ServiceLocator`, `GenerationProgressHub`, and UI callbacks.
- Non-UI portions of `ContentGenerationCallback`.
- Multi-layer index builder and short-form policy utilities.

Required abstractions:

```text
IGeneratorLogger
IJobProgressReporter
IWorkspacePathService
IAITextGenerationClient
IChapterArtifactStore
IGenerationClock
```

Desktop-specific calls map to these abstractions:

```text
TM.App.Log             -> IGeneratorLogger
GlobalToast            -> no-op/log in service
GenerationProgressHub  -> IJobProgressReporter
StoragePathHelper      -> IWorkspacePathService
ServiceLocator         -> ASP.NET Core DI
```

## CHANGES Conversion

The service result must return the shape expected by `novel-center`, not Tianming's internal CHANGES schema.

Conversion rules for the first release:

- `summary`: derive from Tianming structured summary or chapter body summary.
- `key_events`: derive from `NewPlotPoints`, conflict progress, and important state changes.
- `entity_events`: derive from character, location, faction, item, and plot events.
- `character_changes`: derive from `CharacterStateChanges` and relationship deltas.
- `world_changes`: derive from location, faction, timeline, item, and world-rule observations.
- `foreshadowing_updates`: derive from `ForeshadowingActions`.
- `plot_updates`: derive from `NewPlotPoints` and conflict progress.
- `public_state_delta`: compact state delta suitable for `novel-center`.
- `choice_node`: optional. If Tianming output has no explicit choice, the converter may synthesize a minimal two-option node from the chapter hook.
- `branch_anchor_candidate`: optional, generated from high-pressure plot points when available.
- `new_entities` and `ledger_events`: preserve valid ShortIds where possible; otherwise emit names with deterministic service-side ids.

The converter must never leak Tianming-only fields that `novel-center` cannot parse.

## Error Handling

Errors use stable codes:

```text
invalid_request
duplicate_request
workspace_lock_timeout
workspace_prepare_failed
ai_provider_failed
changes_parse_failed
generation_gate_failed
changes_conversion_failed
internal_error
```

Retryable errors are limited to transient AI, Redis, lock timeout, and worker interruption cases. Gate failures are retryable only when attempts remain.

If a worker crashes after claiming a job, a recovery loop finds stale `running` jobs in Postgres and requeues them when the heartbeat has expired.

## Testing

Unit tests:

- Request validation.
- CHANGES conversion from sample Tianming output to `novel-center` format.
- Job status transitions.
- Idempotency by `external_request_id`.
- Workspace path isolation.

Integration tests:

- Postgres-backed job create/query.
- Redis queue claim and completion.
- Worker recovery for stale `running` job.
- Fake AI client returning valid output.
- Fake AI client returning invalid CHANGES and triggering failed job.

Manual smoke test:

1. Start Postgres and Redis.
2. Start `Tianming.GeneratorHost`.
3. Submit one `POST /v1/chapters/generate`.
4. Poll `GET /v1/jobs/{job_id}` until `succeeded`.
5. Confirm the result contains `title`, `body`, `changes`, `attempts`, and `warnings`.

## Rollout Plan

Phase 1: Build Linux service skeleton, job store, Redis queue, worker, and fake generator.

Phase 2: Extract minimum `GeneratorCore` pieces and run real public chapter generation against a per-story workspace.

Phase 3: Add CHANGES conversion and compatibility tests against `novel-center` `GenerationChanges`.

Phase 4: Add operational recovery, retries, locks, and smoke tests.

Phase 5: Only after the service is stable, add `novel-center` remote generator client in a separate spec and implementation plan.

## Acceptance Criteria

- `Tianming.GeneratorHost` builds and runs on Linux as `net8.0`.
- The WPF app remains buildable separately.
- A public chapter generation request can be submitted asynchronously.
- Job state is durable in Postgres.
- Redis is used for queue/progress/locking.
- A completed job returns `novel-center` compatible result JSON.
- The first implementation does not require changes to `novel-center` business logic.

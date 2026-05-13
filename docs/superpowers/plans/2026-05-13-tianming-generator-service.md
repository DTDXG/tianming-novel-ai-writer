# Tianming Generator Service Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Linux-deployable asynchronous Tianming generator service that exposes public chapter generation jobs and returns `novel-center` compatible results.

**Architecture:** Add a pure `net8.0` `GeneratorCore` library plus a pure `net8.0` ASP.NET Core `GeneratorHost`. The host owns HTTP APIs, Postgres job persistence, Redis queue/progress/locks, and background execution; the core owns request/result contracts, workspace paths, generation facade, and CHANGES conversion. Start with a fake generator to make the service testable end-to-end, then wire the real Tianming extraction path behind the same interface.

**Tech Stack:** .NET 8, ASP.NET Core Minimal API, xUnit, Npgsql, StackExchange.Redis, Microsoft.Extensions.Hosting, System.Text.Json.

---

## Scope Check

This plan implements only the confirmed service-side spec:

- Linux service project.
- Async public chapter job API.
- Postgres durable job table.
- Redis queue/progress/lock channel.
- Per-story workspace paths.
- `novel-center` compatible result contracts.
- Fake generator first, real Tianming core extraction behind an interface.

It does not modify `novel-center`. It does not implement opening-story generation, personal branches, images, or a remote client in `novel-center`.

## File Structure

Create these project files:

- `Tianming.GeneratorService.sln`: solution for host, core, and tests.
- `ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj`: pure `net8.0` library.
- `ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj`: pure `net8.0` ASP.NET Core host.
- `Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj`: core unit tests.
- `Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj`: host unit tests.

Create core files:

- `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationRequest.cs`: public API request model.
- `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationResult.cs`: service result and `novel-center` result models.
- `ServicesHost/Tianming.GeneratorCore/Contracts/JobContracts.cs`: job status, progress, error contracts.
- `ServicesHost/Tianming.GeneratorCore/Changes/TianmingChanges.cs`: minimal Tianming CHANGES model for conversion.
- `ServicesHost/Tianming.GeneratorCore/Changes/NovelCenterChanges.cs`: `novel-center` compatible changes model.
- `ServicesHost/Tianming.GeneratorCore/Changes/ChangesConverter.cs`: maps Tianming CHANGES to `NovelCenterChanges`.
- `ServicesHost/Tianming.GeneratorCore/Generation/IPublicChapterGenerator.cs`: generator interface.
- `ServicesHost/Tianming.GeneratorCore/Generation/FakePublicChapterGenerator.cs`: deterministic fake generator for service tests.
- `ServicesHost/Tianming.GeneratorCore/Workspace/WorkspaceOptions.cs`: workspace root options.
- `ServicesHost/Tianming.GeneratorCore/Workspace/StoryWorkspaceService.cs`: per-story workspace path creation.
- `ServicesHost/Tianming.GeneratorCore/Progress/IJobProgressReporter.cs`: progress abstraction.

Create host files:

- `ServicesHost/Tianming.GeneratorHost/Program.cs`: Minimal API entry point and DI wiring.
- `ServicesHost/Tianming.GeneratorHost/appsettings.json`: local defaults.
- `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobRecord.cs`: Postgres row model.
- `ServicesHost/Tianming.GeneratorHost/Data/IGenerationJobStore.cs`: durable job store interface.
- `ServicesHost/Tianming.GeneratorHost/Data/PostgresGenerationJobStore.cs`: Npgsql implementation.
- `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobSchema.cs`: schema creation SQL.
- `ServicesHost/Tianming.GeneratorHost/Queue/IChapterJobQueue.cs`: queue abstraction.
- `ServicesHost/Tianming.GeneratorHost/Queue/RedisChapterJobQueue.cs`: Redis queue/progress/lock implementation.
- `ServicesHost/Tianming.GeneratorHost/Workers/ChapterGenerationWorker.cs`: background worker.
- `ServicesHost/Tianming.GeneratorHost/Api/ChapterGenerationEndpoints.cs`: API route mapping.
- `ServicesHost/Tianming.GeneratorHost/Api/JobStatusEndpoints.cs`: job status route mapping.

Modify existing files:

- `docs/superpowers/specs/2026-05-13-tianming-generator-service-design.md`: no change expected.
- Existing WPF files: no change in Phase 1. Any real core extraction in later tasks must be additive or move Linux-safe code into `GeneratorCore` without changing WPF behavior.

---

### Task 1: Create Solution And Project Skeleton

**Files:**
- Create: `Tianming.GeneratorService.sln`
- Create: `ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj`
- Create: `ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj`
- Create: `Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj`
- Create: `Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj`

- [ ] **Step 1: Create the projects**

Run:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
mkdir -p ServicesHost Tests
dotnet new sln -n Tianming.GeneratorService
dotnet new classlib -n Tianming.GeneratorCore -o ServicesHost/Tianming.GeneratorCore --framework net8.0
dotnet new web -n Tianming.GeneratorHost -o ServicesHost/Tianming.GeneratorHost --framework net8.0
dotnet new xunit -n Tianming.GeneratorCore.Tests -o Tests/Tianming.GeneratorCore.Tests --framework net8.0
dotnet new xunit -n Tianming.GeneratorHost.Tests -o Tests/Tianming.GeneratorHost.Tests --framework net8.0
dotnet sln Tianming.GeneratorService.sln add ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj
dotnet sln Tianming.GeneratorService.sln add ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj
dotnet sln Tianming.GeneratorService.sln add Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj
dotnet sln Tianming.GeneratorService.sln add Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj
dotnet add ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj reference ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj
dotnet add Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj reference ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj
dotnet add Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj reference ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj
dotnet add Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj reference ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj
```

Expected: solution and four projects are created.

- [ ] **Step 2: Add NuGet packages**

Run:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
dotnet add ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj package Npgsql --version 8.0.6
dotnet add ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj package StackExchange.Redis --version 2.8.16
dotnet add Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj package Microsoft.AspNetCore.Mvc.Testing --version 8.0.11
```

Expected: package references are added.

- [ ] **Step 3: Remove template classes**

Delete template files if present:

```bash
rm -f ServicesHost/Tianming.GeneratorCore/Class1.cs
rm -f Tests/Tianming.GeneratorCore.Tests/UnitTest1.cs
rm -f Tests/Tianming.GeneratorHost.Tests/UnitTest1.cs
```

Expected: no template test or class remains.

- [ ] **Step 4: Verify solution builds**

Run:

```bash
dotnet build Tianming.GeneratorService.sln
```

Expected: build succeeds with no errors.

- [ ] **Step 5: Commit**

```bash
git add Tianming.GeneratorService.sln ServicesHost Tests/Tianming.GeneratorCore.Tests Tests/Tianming.GeneratorHost.Tests
git commit -m "build: add generator service projects"
```

Expected: one commit containing only the solution and project skeleton.

---

### Task 2: Add Core Contracts

**Files:**
- Create: `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationRequest.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationResult.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Contracts/JobContracts.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Contracts/ContractSerializationTests.cs`

- [ ] **Step 1: Write serialization tests**

Create `Tests/Tianming.GeneratorCore.Tests/Contracts/ContractSerializationTests.cs`:

```csharp
using System.Text.Json;
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorCore.Tests.Contracts;

public sealed class ContractSerializationTests
{
    [Fact]
    public void ChapterGenerationRequest_UsesSnakeCaseJsonNames()
    {
        var json = """
        {
          "external_request_id": "nc-job-1",
          "story_id": "story-1",
          "story_line_id": "line-1",
          "volume_id": "volume-1",
          "chapter_id": "vol1_ch1",
          "chapter_no": 1,
          "volume_no": 1,
          "volume_chapter_no": 1,
          "channel": "xiuxian",
          "story": { "title": "外门旧案" },
          "volume": { "title": "第一卷" },
          "chapter_plan": { "title": "雨夜" },
          "recent_chapters": ["前文"],
          "previous_chapter_tail": "尾声",
          "volume_archives": [],
          "vector_recall_fragments": [],
          "ledger_entity_ids": ["C123456789ABC"],
          "hard_constraints": ["不得推翻已发生事件"]
        }
        """;

        var request = JsonSerializer.Deserialize<ChapterGenerationRequest>(
            json,
            JsonOptions.Default);

        Assert.NotNull(request);
        Assert.Equal("nc-job-1", request.ExternalRequestId);
        Assert.Equal("story-1", request.StoryId);
        Assert.Equal("vol1_ch1", request.ChapterId);
        Assert.Equal(1, request.ChapterNo);
        Assert.Single(request.RecentChapters);
        Assert.Single(request.LedgerEntityIds);
    }

    [Fact]
    public void JobStatusResponse_SerializesSucceededResult()
    {
        var response = JobStatusResponse.Succeeded(
            "tmgen_1",
            new ChapterGenerationResult(
                "第一章 雨夜",
                "正文",
                new NovelCenterChanges { Summary = "摘要" },
                "原始输出",
                1,
                Array.Empty<string>()));

        var json = JsonSerializer.Serialize(response, JsonOptions.Default);

        Assert.Contains("\"status\":\"succeeded\"", json);
        Assert.Contains("\"title\":\"第一章 雨夜\"", json);
        Assert.Contains("\"summary\":\"摘要\"", json);
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter ContractSerializationTests
```

Expected: fails because `Tianming.GeneratorCore.Contracts` types do not exist.

- [ ] **Step 3: Add JSON options**

Create `ServicesHost/Tianming.GeneratorCore/Contracts/JsonOptions.cs`:

```csharp
using System.Text.Json;

namespace Tianming.GeneratorCore.Contracts;

public static class JsonOptions
{
    public static readonly JsonSerializerOptions Default = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,
        DictionaryKeyPolicy = JsonNamingPolicy.SnakeCaseLower,
        PropertyNameCaseInsensitive = true,
        WriteIndented = false
    };
}
```

- [ ] **Step 4: Add request contract**

Create `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationRequest.cs`:

```csharp
using System.Text.Json.Nodes;

namespace Tianming.GeneratorCore.Contracts;

public sealed record ChapterGenerationRequest(
    string ExternalRequestId,
    string StoryId,
    string StoryLineId,
    string VolumeId,
    string ChapterId,
    int ChapterNo,
    int VolumeNo,
    int VolumeChapterNo,
    string Channel,
    JsonObject Story,
    JsonObject Volume,
    JsonObject ChapterPlan,
    IReadOnlyList<string> RecentChapters,
    string PreviousChapterTail,
    IReadOnlyList<JsonObject> VolumeArchives,
    IReadOnlyList<JsonObject> VectorRecallFragments,
    IReadOnlyList<string> LedgerEntityIds,
    IReadOnlyList<string> HardConstraints);
```

- [ ] **Step 5: Add result contract**

Create `ServicesHost/Tianming.GeneratorCore/Contracts/ChapterGenerationResult.cs`:

```csharp
namespace Tianming.GeneratorCore.Contracts;

public sealed record ChapterGenerationResult(
    string Title,
    string Body,
    NovelCenterChanges Changes,
    string RawOutput,
    int Attempts,
    IReadOnlyList<string> Warnings);

public sealed class NovelCenterChanges
{
    public string Summary { get; set; } = string.Empty;
    public List<string> KeyEvents { get; set; } = new();
    public List<Dictionary<string, object?>> EntityEvents { get; set; } = new();
    public List<Dictionary<string, object?>> CharacterChanges { get; set; } = new();
    public List<Dictionary<string, object?>> WorldChanges { get; set; } = new();
    public List<Dictionary<string, object?>> ForeshadowingUpdates { get; set; } = new();
    public List<Dictionary<string, object?>> PlotUpdates { get; set; } = new();
    public Dictionary<string, object?> StateDelta { get; set; } = new();
    public List<string> RuleObservations { get; set; } = new();
    public List<string> QualityFlags { get; set; } = new();
    public Dictionary<string, object?>? PublicStateDelta { get; set; }
    public Dictionary<string, object?>? ChoiceNode { get; set; }
    public Dictionary<string, object?>? BranchAnchor { get; set; }
    public Dictionary<string, object?>? BranchAnchorCandidate { get; set; }
    public List<Dictionary<string, object?>> VoteDirectivesUsed { get; set; } = new();
    public List<Dictionary<string, object?>> NewEntities { get; set; } = new();
    public List<Dictionary<string, object?>> LedgerEvents { get; set; } = new();
}
```

- [ ] **Step 6: Add job contracts**

Create `ServicesHost/Tianming.GeneratorCore/Contracts/JobContracts.cs`:

```csharp
namespace Tianming.GeneratorCore.Contracts;

public static class JobStatuses
{
    public const string Queued = "queued";
    public const string Running = "running";
    public const string Succeeded = "succeeded";
    public const string Failed = "failed";
    public const string Cancelled = "cancelled";
}

public sealed record CreateJobResponse(string JobId, string Status);

public sealed record JobProgress(string Stage, int Attempt, string Message);

public sealed record JobError(string Code, string Message, bool Retryable);

public sealed record JobStatusResponse(
    string JobId,
    string Status,
    JobProgress? Progress,
    ChapterGenerationResult? Result,
    JobError? Error)
{
    public static JobStatusResponse Queued(string jobId) =>
        new(jobId, JobStatuses.Queued, null, null, null);

    public static JobStatusResponse Running(string jobId, JobProgress progress) =>
        new(jobId, JobStatuses.Running, progress, null, null);

    public static JobStatusResponse Succeeded(string jobId, ChapterGenerationResult result) =>
        new(jobId, JobStatuses.Succeeded, null, result, null);

    public static JobStatusResponse Failed(string jobId, JobError error) =>
        new(jobId, JobStatuses.Failed, null, null, error);
}
```

- [ ] **Step 7: Run tests and verify they pass**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter ContractSerializationTests
```

Expected: all `ContractSerializationTests` pass.

- [ ] **Step 8: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Contracts Tests/Tianming.GeneratorCore.Tests/Contracts
git commit -m "feat: add generator service contracts"
```

Expected: one commit with contracts and tests.

---

### Task 3: Add CHANGES Conversion

**Files:**
- Create: `ServicesHost/Tianming.GeneratorCore/Changes/TianmingChanges.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Changes/ChangesConverter.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Changes/ChangesConverterTests.cs`

- [ ] **Step 1: Write converter tests**

Create `Tests/Tianming.GeneratorCore.Tests/Changes/ChangesConverterTests.cs`:

```csharp
using Tianming.GeneratorCore.Changes;

namespace Tianming.GeneratorCore.Tests.Changes;

public sealed class ChangesConverterTests
{
    [Fact]
    public void Convert_MapsCoreTianmingFieldsToNovelCenterChanges()
    {
        var changes = new TianmingChanges
        {
            CharacterStateChanges =
            {
                new CharacterStateChange
                {
                    CharacterId = "C123456789ABC",
                    KeyEvent = "陆青衣发现禁库旧案",
                    NewMentalState = "警觉",
                    Importance = "critical"
                }
            },
            ConflictProgress =
            {
                new ConflictProgressChange
                {
                    ConflictId = "K123456789ABC",
                    NewStatus = "旧案重新浮出水面",
                    KeyEvent = "戒律堂封锁藏经楼",
                    Importance = "important"
                }
            },
            NewPlotPoints =
            {
                new PlotPointChange
                {
                    Context = "残剑说出被宗门抹去的名字",
                    Keywords = new List<string> { "残剑", "旧案" },
                    Importance = "critical",
                    Storyline = "main"
                }
            },
            ForeshadowingActions =
            {
                new ForeshadowingAction
                {
                    ForeshadowId = "F123456789ABC",
                    Action = "setup"
                }
            },
            TimeProgression = new TimeProgressionChange
            {
                TimePeriod = "雨夜",
                ElapsedTime = "一刻钟",
                KeyTimeEvent = "禁库门开",
                Importance = "important"
            }
        };

        var result = ChangesConverter.ToNovelCenter(changes, "本章摘要", "下一步如何推进？");

        Assert.Equal("本章摘要", result.Summary);
        Assert.Contains("陆青衣发现禁库旧案", result.KeyEvents);
        Assert.Contains(result.ForeshadowingUpdates, x => x["foreshadowing_id"]!.Equals("F123456789ABC"));
        Assert.Contains(result.PlotUpdates, x => x["context"]!.Equals("残剑说出被宗门抹去的名字"));
        Assert.NotNull(result.PublicStateDelta);
        Assert.NotNull(result.ChoiceNode);
    }

    [Fact]
    public void Convert_SynthesizesTwoChoiceOptions()
    {
        var result = ChangesConverter.ToNovelCenter(new TianmingChanges(), "摘要", "是否追查禁库？");

        Assert.NotNull(result.ChoiceNode);
        var options = Assert.IsAssignableFrom<List<Dictionary<string, object?>>>(result.ChoiceNode!["options"]);
        Assert.Equal(2, options.Count);
        Assert.Equal("option_a", options[0]["id"]);
        Assert.Equal("option_b", options[1]["id"]);
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter ChangesConverterTests
```

Expected: fails because `Tianming.GeneratorCore.Changes` does not exist.

- [ ] **Step 3: Add Tianming CHANGES model**

Create `ServicesHost/Tianming.GeneratorCore/Changes/TianmingChanges.cs`:

```csharp
using System.Text.Json.Serialization;

namespace Tianming.GeneratorCore.Changes;

public sealed class TianmingChanges
{
    [JsonPropertyName("CharacterStateChanges")]
    public List<CharacterStateChange> CharacterStateChanges { get; set; } = new();

    [JsonPropertyName("ConflictProgress")]
    public List<ConflictProgressChange> ConflictProgress { get; set; } = new();

    [JsonPropertyName("ForeshadowingActions")]
    public List<ForeshadowingAction> ForeshadowingActions { get; set; } = new();

    [JsonPropertyName("NewPlotPoints")]
    public List<PlotPointChange> NewPlotPoints { get; set; } = new();

    [JsonPropertyName("LocationStateChanges")]
    public List<LocationStateChange> LocationStateChanges { get; set; } = new();

    [JsonPropertyName("FactionStateChanges")]
    public List<FactionStateChange> FactionStateChanges { get; set; } = new();

    [JsonPropertyName("TimeProgression")]
    public TimeProgressionChange? TimeProgression { get; set; }

    [JsonPropertyName("CharacterMovements")]
    public List<CharacterMovementChange> CharacterMovements { get; set; } = new();

    [JsonPropertyName("ItemTransfers")]
    public List<ItemTransferChange> ItemTransfers { get; set; } = new();
}

public sealed class CharacterStateChange
{
    public string CharacterId { get; set; } = string.Empty;
    public string NewLevel { get; set; } = string.Empty;
    public string NewAbilities { get; set; } = string.Empty;
    public string LostAbilities { get; set; } = string.Empty;
    public Dictionary<string, int> RelationshipChanges { get; set; } = new();
    public string NewMentalState { get; set; } = string.Empty;
    public string KeyEvent { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class ConflictProgressChange
{
    public string ConflictId { get; set; } = string.Empty;
    public string NewStatus { get; set; } = string.Empty;
    public string KeyEvent { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class ForeshadowingAction
{
    public string ForeshadowId { get; set; } = string.Empty;
    public string Action { get; set; } = string.Empty;
}

public sealed class PlotPointChange
{
    public List<string> Keywords { get; set; } = new();
    public string Context { get; set; } = string.Empty;
    public List<string> InvolvedCharacters { get; set; } = new();
    public string Importance { get; set; } = "normal";
    public string Storyline { get; set; } = "main";
}

public sealed class LocationStateChange
{
    public string LocationId { get; set; } = string.Empty;
    public string NewStatus { get; set; } = string.Empty;
    public string Event { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class FactionStateChange
{
    public string FactionId { get; set; } = string.Empty;
    public string NewStatus { get; set; } = string.Empty;
    public string Event { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class TimeProgressionChange
{
    public string TimePeriod { get; set; } = string.Empty;
    public string ElapsedTime { get; set; } = string.Empty;
    public string KeyTimeEvent { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class CharacterMovementChange
{
    public string CharacterId { get; set; } = string.Empty;
    public string FromLocation { get; set; } = string.Empty;
    public string ToLocation { get; set; } = string.Empty;
    public string Event { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}

public sealed class ItemTransferChange
{
    public string ItemId { get; set; } = string.Empty;
    public string ItemName { get; set; } = string.Empty;
    public string FromHolder { get; set; } = string.Empty;
    public string ToHolder { get; set; } = string.Empty;
    public string NewStatus { get; set; } = string.Empty;
    public string Event { get; set; } = string.Empty;
    public string Importance { get; set; } = "normal";
}
```

- [ ] **Step 4: Add converter**

Create `ServicesHost/Tianming.GeneratorCore/Changes/ChangesConverter.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorCore.Changes;

public static class ChangesConverter
{
    public static NovelCenterChanges ToNovelCenter(
        TianmingChanges changes,
        string summary,
        string choicePrompt)
    {
        var result = new NovelCenterChanges
        {
            Summary = summary,
            PublicStateDelta = new Dictionary<string, object?>
            {
                ["summary"] = summary,
                ["has_tianming_changes"] = true
            },
            ChoiceNode = BuildChoiceNode(choicePrompt)
        };

        foreach (var change in changes.CharacterStateChanges)
        {
            AddKeyEvent(result, change.KeyEvent);
            result.CharacterChanges.Add(new Dictionary<string, object?>
            {
                ["character_id"] = change.CharacterId,
                ["new_level"] = change.NewLevel,
                ["new_abilities"] = change.NewAbilities,
                ["lost_abilities"] = change.LostAbilities,
                ["relationship_changes"] = change.RelationshipChanges,
                ["new_mental_state"] = change.NewMentalState,
                ["key_event"] = change.KeyEvent,
                ["importance"] = change.Importance
            });
        }

        foreach (var conflict in changes.ConflictProgress)
        {
            AddKeyEvent(result, conflict.KeyEvent);
            result.PlotUpdates.Add(new Dictionary<string, object?>
            {
                ["kind"] = "conflict_progress",
                ["conflict_id"] = conflict.ConflictId,
                ["new_status"] = conflict.NewStatus,
                ["key_event"] = conflict.KeyEvent,
                ["importance"] = conflict.Importance
            });
        }

        foreach (var action in changes.ForeshadowingActions)
        {
            result.ForeshadowingUpdates.Add(new Dictionary<string, object?>
            {
                ["foreshadowing_id"] = action.ForeshadowId,
                ["action"] = action.Action
            });
        }

        foreach (var plot in changes.NewPlotPoints)
        {
            AddKeyEvent(result, plot.Context);
            result.PlotUpdates.Add(new Dictionary<string, object?>
            {
                ["kind"] = "plot_point",
                ["context"] = plot.Context,
                ["keywords"] = plot.Keywords,
                ["involved_characters"] = plot.InvolvedCharacters,
                ["importance"] = plot.Importance,
                ["storyline"] = plot.Storyline
            });
        }

        foreach (var location in changes.LocationStateChanges)
        {
            AddWorldChange(result, "location_state", location.LocationId, location.NewStatus, location.Event, location.Importance);
        }

        foreach (var faction in changes.FactionStateChanges)
        {
            AddWorldChange(result, "faction_state", faction.FactionId, faction.NewStatus, faction.Event, faction.Importance);
        }

        if (changes.TimeProgression is not null)
        {
            result.WorldChanges.Add(new Dictionary<string, object?>
            {
                ["kind"] = "time_progression",
                ["time_period"] = changes.TimeProgression.TimePeriod,
                ["elapsed_time"] = changes.TimeProgression.ElapsedTime,
                ["key_time_event"] = changes.TimeProgression.KeyTimeEvent,
                ["importance"] = changes.TimeProgression.Importance
            });
        }

        foreach (var movement in changes.CharacterMovements)
        {
            result.EntityEvents.Add(new Dictionary<string, object?>
            {
                ["kind"] = "character_movement",
                ["character_id"] = movement.CharacterId,
                ["from_location"] = movement.FromLocation,
                ["to_location"] = movement.ToLocation,
                ["event"] = movement.Event,
                ["importance"] = movement.Importance
            });
        }

        foreach (var transfer in changes.ItemTransfers)
        {
            result.EntityEvents.Add(new Dictionary<string, object?>
            {
                ["kind"] = "item_transfer",
                ["item_id"] = transfer.ItemId,
                ["item_name"] = transfer.ItemName,
                ["from_holder"] = transfer.FromHolder,
                ["to_holder"] = transfer.ToHolder,
                ["new_status"] = transfer.NewStatus,
                ["event"] = transfer.Event,
                ["importance"] = transfer.Importance
            });
        }

        result.StateDelta["summary"] = summary;
        return result;
    }

    private static void AddKeyEvent(NovelCenterChanges result, string value)
    {
        if (!string.IsNullOrWhiteSpace(value))
            result.KeyEvents.Add(value);
    }

    private static void AddWorldChange(
        NovelCenterChanges result,
        string kind,
        string id,
        string status,
        string eventText,
        string importance)
    {
        result.WorldChanges.Add(new Dictionary<string, object?>
        {
            ["kind"] = kind,
            ["entity_id"] = id,
            ["new_status"] = status,
            ["event"] = eventText,
            ["importance"] = importance
        });
    }

    private static Dictionary<string, object?> BuildChoiceNode(string prompt)
    {
        var normalizedPrompt = string.IsNullOrWhiteSpace(prompt) ? "下一步如何推进？" : prompt;
        return new Dictionary<string, object?>
        {
            ["prompt"] = normalizedPrompt,
            ["options"] = new List<Dictionary<string, object?>>
            {
                new()
                {
                    ["id"] = "option_a",
                    ["label"] = "主动推进"
                },
                new()
                {
                    ["id"] = "option_b",
                    ["label"] = "谨慎观察"
                }
            }
        };
    }
}
```

- [ ] **Step 5: Run converter tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter ChangesConverterTests
```

Expected: all `ChangesConverterTests` pass.

- [ ] **Step 6: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Changes Tests/Tianming.GeneratorCore.Tests/Changes
git commit -m "feat: convert tianming changes to novel-center changes"
```

Expected: one commit with converter and tests.

---

### Task 4: Add Workspace Service

**Files:**
- Create: `ServicesHost/Tianming.GeneratorCore/Workspace/WorkspaceOptions.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Workspace/StoryWorkspace.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Workspace/StoryWorkspaceService.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Workspace/StoryWorkspaceServiceTests.cs`

- [ ] **Step 1: Write workspace tests**

Create `Tests/Tianming.GeneratorCore.Tests/Workspace/StoryWorkspaceServiceTests.cs`:

```csharp
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Tests.Workspace;

public sealed class StoryWorkspaceServiceTests
{
    [Fact]
    public void Prepare_CreatesExpectedStoryDirectories()
    {
        var root = Path.Combine(Path.GetTempPath(), "tmgen-tests", Guid.NewGuid().ToString("N"));
        var service = new StoryWorkspaceService(new WorkspaceOptions { Root = root });

        var workspace = service.Prepare("story:one");

        Assert.EndsWith(Path.Combine("stories", "story_one"), workspace.Root);
        Assert.True(Directory.Exists(workspace.ChaptersPath));
        Assert.True(Directory.Exists(workspace.GuidesPath));
        Assert.True(Directory.Exists(workspace.IndexesPath));
        Assert.True(Directory.Exists(workspace.ArchivesPath));
        Assert.True(Directory.Exists(workspace.RuntimePath));
    }

    [Fact]
    public void Prepare_RejectsEmptyStoryId()
    {
        var service = new StoryWorkspaceService(new WorkspaceOptions { Root = Path.GetTempPath() });

        var ex = Assert.Throws<ArgumentException>(() => service.Prepare(""));

        Assert.Contains("storyId", ex.Message);
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter StoryWorkspaceServiceTests
```

Expected: fails because workspace types do not exist.

- [ ] **Step 3: Add workspace options and model**

Create `ServicesHost/Tianming.GeneratorCore/Workspace/WorkspaceOptions.cs`:

```csharp
namespace Tianming.GeneratorCore.Workspace;

public sealed class WorkspaceOptions
{
    public string Root { get; set; } = Path.Combine(Path.GetTempPath(), "tianming-generator");
}
```

Create `ServicesHost/Tianming.GeneratorCore/Workspace/StoryWorkspace.cs`:

```csharp
namespace Tianming.GeneratorCore.Workspace;

public sealed record StoryWorkspace(
    string StoryId,
    string Root,
    string ChaptersPath,
    string GuidesPath,
    string IndexesPath,
    string ArchivesPath,
    string RuntimePath);
```

- [ ] **Step 4: Add workspace service**

Create `ServicesHost/Tianming.GeneratorCore/Workspace/StoryWorkspaceService.cs`:

```csharp
using System.Text.RegularExpressions;

namespace Tianming.GeneratorCore.Workspace;

public sealed class StoryWorkspaceService
{
    private static readonly Regex UnsafeChars = new("[^a-zA-Z0-9_-]+", RegexOptions.Compiled);
    private readonly WorkspaceOptions _options;

    public StoryWorkspaceService(WorkspaceOptions options)
    {
        _options = options;
    }

    public StoryWorkspace Prepare(string storyId)
    {
        if (string.IsNullOrWhiteSpace(storyId))
            throw new ArgumentException("storyId must not be empty", nameof(storyId));

        var safeStoryId = UnsafeChars.Replace(storyId.Trim(), "_").Trim('_');
        if (safeStoryId.Length == 0)
            throw new ArgumentException("storyId must contain at least one safe character", nameof(storyId));

        var root = Path.Combine(_options.Root, "stories", safeStoryId);
        var workspace = new StoryWorkspace(
            storyId,
            root,
            Path.Combine(root, "chapters"),
            Path.Combine(root, "guides"),
            Path.Combine(root, "indexes"),
            Path.Combine(root, "archives"),
            Path.Combine(root, "runtime"));

        Directory.CreateDirectory(workspace.ChaptersPath);
        Directory.CreateDirectory(workspace.GuidesPath);
        Directory.CreateDirectory(workspace.IndexesPath);
        Directory.CreateDirectory(workspace.ArchivesPath);
        Directory.CreateDirectory(workspace.RuntimePath);

        return workspace;
    }
}
```

- [ ] **Step 5: Run workspace tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter StoryWorkspaceServiceTests
```

Expected: all `StoryWorkspaceServiceTests` pass.

- [ ] **Step 6: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Workspace Tests/Tianming.GeneratorCore.Tests/Workspace
git commit -m "feat: add per-story workspace service"
```

Expected: one commit with workspace service and tests.

---

### Task 5: Add Public Chapter Generator Interface And Fake Generator

**Files:**
- Create: `ServicesHost/Tianming.GeneratorCore/Generation/IPublicChapterGenerator.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Generation/FakePublicChapterGenerator.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Progress/IJobProgressReporter.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Generation/FakePublicChapterGeneratorTests.cs`

- [ ] **Step 1: Write fake generator test**

Create `Tests/Tianming.GeneratorCore.Tests/Generation/FakePublicChapterGeneratorTests.cs`:

```csharp
using System.Text.Json.Nodes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Tests.Generation;

public sealed class FakePublicChapterGeneratorTests
{
    [Fact]
    public async Task GenerateAsync_ReturnsNovelCenterCompatibleResult()
    {
        var generator = new FakePublicChapterGenerator();
        var request = new ChapterGenerationRequest(
            "req-1",
            "story-1",
            "line-1",
            "volume-1",
            "vol1_ch1",
            1,
            1,
            1,
            "xiuxian",
            new JsonObject(),
            new JsonObject(),
            new JsonObject { ["title"] = "雨夜旧案" },
            Array.Empty<string>(),
            "",
            Array.Empty<JsonObject>(),
            Array.Empty<JsonObject>(),
            Array.Empty<string>(),
            Array.Empty<string>());
        var workspace = new StoryWorkspace("story-1", "/tmp/story-1", "/tmp/story-1/chapters", "/tmp/story-1/guides", "/tmp/story-1/indexes", "/tmp/story-1/archives", "/tmp/story-1/runtime");

        var result = await generator.GenerateAsync(request, workspace, NullJobProgressReporter.Instance, CancellationToken.None);

        Assert.Equal("雨夜旧案", result.Title);
        Assert.Contains("vol1_ch1", result.Body);
        Assert.Equal("fake generator output", result.RawOutput);
        Assert.NotNull(result.Changes.ChoiceNode);
        Assert.True(result.Attempts >= 1);
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter FakePublicChapterGeneratorTests
```

Expected: fails because generation and progress types do not exist.

- [ ] **Step 3: Add progress abstraction**

Create `ServicesHost/Tianming.GeneratorCore/Progress/IJobProgressReporter.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorCore.Progress;

public interface IJobProgressReporter
{
    Task ReportAsync(string jobId, JobProgress progress, CancellationToken cancellationToken);
}

public sealed class NullJobProgressReporter : IJobProgressReporter
{
    public static readonly NullJobProgressReporter Instance = new();

    private NullJobProgressReporter()
    {
    }

    public Task ReportAsync(string jobId, JobProgress progress, CancellationToken cancellationToken)
        => Task.CompletedTask;
}
```

- [ ] **Step 4: Add generator interface**

Create `ServicesHost/Tianming.GeneratorCore/Generation/IPublicChapterGenerator.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Generation;

public interface IPublicChapterGenerator
{
    Task<ChapterGenerationResult> GenerateAsync(
        ChapterGenerationRequest request,
        StoryWorkspace workspace,
        IJobProgressReporter progressReporter,
        CancellationToken cancellationToken);
}
```

- [ ] **Step 5: Add fake generator**

Create `ServicesHost/Tianming.GeneratorCore/Generation/FakePublicChapterGenerator.cs`:

```csharp
using Tianming.GeneratorCore.Changes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Generation;

public sealed class FakePublicChapterGenerator : IPublicChapterGenerator
{
    public async Task<ChapterGenerationResult> GenerateAsync(
        ChapterGenerationRequest request,
        StoryWorkspace workspace,
        IJobProgressReporter progressReporter,
        CancellationToken cancellationToken)
    {
        await progressReporter.ReportAsync(
            request.ExternalRequestId,
            new JobProgress("fake_generation", 1, "generating deterministic fake chapter"),
            cancellationToken);

        var title = ReadString(request.ChapterPlan, "title") ?? $"Chapter {request.ChapterNo}";
        var body = $"[{request.ChapterId}] {title}\n\n这是用于服务链路验证的确定性章节正文。";
        var tianmingChanges = new TianmingChanges
        {
            NewPlotPoints =
            {
                new PlotPointChange
                {
                    Context = $"{title} 形成新的公共主线压力点",
                    Keywords = new List<string> { "公共主线", "压力点" },
                    Importance = "important",
                    Storyline = "main"
                }
            }
        };
        var changes = ChangesConverter.ToNovelCenter(
            tianmingChanges,
            $"{title} 的服务化生成摘要",
            "下一步如何推进？");

        return new ChapterGenerationResult(
            title,
            body,
            changes,
            "fake generator output",
            1,
            Array.Empty<string>());
    }

    private static string? ReadString(System.Text.Json.Nodes.JsonObject value, string key)
    {
        return value.TryGetPropertyValue(key, out var node) ? node?.GetValue<string>() : null;
    }
}
```

- [ ] **Step 6: Run fake generator tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter FakePublicChapterGeneratorTests
```

Expected: all fake generator tests pass.

- [ ] **Step 7: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Generation ServicesHost/Tianming.GeneratorCore/Progress Tests/Tianming.GeneratorCore.Tests/Generation
git commit -m "feat: add public chapter generator abstraction"
```

Expected: one commit with generator abstraction, fake implementation, and tests.

---

### Task 6: Add Postgres Job Store

**Files:**
- Create: `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobRecord.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobSchema.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Data/IGenerationJobStore.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Data/PostgresGenerationJobStore.cs`
- Test: `Tests/Tianming.GeneratorHost.Tests/Data/GenerationJobSchemaTests.cs`

- [ ] **Step 1: Write schema tests**

Create `Tests/Tianming.GeneratorHost.Tests/Data/GenerationJobSchemaTests.cs`:

```csharp
using Tianming.GeneratorHost.Data;

namespace Tianming.GeneratorHost.Tests.Data;

public sealed class GenerationJobSchemaTests
{
    [Fact]
    public void CreateTableSql_ContainsRequiredColumns()
    {
        var sql = GenerationJobSchema.CreateTableSql;

        Assert.Contains("generation_jobs", sql);
        Assert.Contains("external_request_id text unique not null", sql);
        Assert.Contains("request_json jsonb not null", sql);
        Assert.Contains("result_json jsonb null", sql);
        Assert.Contains("error_json jsonb null", sql);
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter GenerationJobSchemaTests
```

Expected: fails because host data types do not exist.

- [ ] **Step 3: Add job record and schema**

Create `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobRecord.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorHost.Data;

public sealed record GenerationJobRecord(
    string Id,
    string ExternalRequestId,
    string StoryId,
    string ChapterId,
    string Status,
    ChapterGenerationRequest Request,
    ChapterGenerationResult? Result,
    JobError? Error,
    int Attempts,
    DateTimeOffset CreatedAt,
    DateTimeOffset? StartedAt,
    DateTimeOffset? FinishedAt,
    DateTimeOffset UpdatedAt);
```

Create `ServicesHost/Tianming.GeneratorHost/Data/GenerationJobSchema.cs`:

```csharp
namespace Tianming.GeneratorHost.Data;

public static class GenerationJobSchema
{
    public const string CreateTableSql = """
    create table if not exists generation_jobs (
        id text primary key,
        external_request_id text unique not null,
        story_id text not null,
        chapter_id text not null,
        status text not null,
        request_json jsonb not null,
        result_json jsonb null,
        error_json jsonb null,
        attempts int not null default 0,
        created_at timestamptz not null,
        started_at timestamptz null,
        finished_at timestamptz null,
        updated_at timestamptz not null
    );
    """;
}
```

- [ ] **Step 4: Add job store interface**

Create `ServicesHost/Tianming.GeneratorHost/Data/IGenerationJobStore.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorHost.Data;

public interface IGenerationJobStore
{
    Task EnsureSchemaAsync(CancellationToken cancellationToken);

    Task<GenerationJobRecord> CreateOrGetQueuedAsync(
        string jobId,
        ChapterGenerationRequest request,
        CancellationToken cancellationToken);

    Task<GenerationJobRecord?> GetAsync(string jobId, CancellationToken cancellationToken);

    Task<GenerationJobRecord?> GetByExternalRequestIdAsync(
        string externalRequestId,
        CancellationToken cancellationToken);

    Task MarkRunningAsync(string jobId, DateTimeOffset startedAt, CancellationToken cancellationToken);

    Task MarkSucceededAsync(
        string jobId,
        ChapterGenerationResult result,
        int attempts,
        DateTimeOffset finishedAt,
        CancellationToken cancellationToken);

    Task MarkFailedAsync(
        string jobId,
        JobError error,
        int attempts,
        DateTimeOffset finishedAt,
        CancellationToken cancellationToken);
}
```

- [ ] **Step 5: Add Postgres implementation**

Create `ServicesHost/Tianming.GeneratorHost/Data/PostgresGenerationJobStore.cs`:

```csharp
using System.Text.Json;
using Npgsql;
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorHost.Data;

public sealed class PostgresGenerationJobStore : IGenerationJobStore
{
    private readonly string _connectionString;

    public PostgresGenerationJobStore(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("Postgres")
            ?? throw new InvalidOperationException("ConnectionStrings:Postgres is required");
    }

    public async Task EnsureSchemaAsync(CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand(GenerationJobSchema.CreateTableSql, connection);
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    public async Task<GenerationJobRecord> CreateOrGetQueuedAsync(
        string jobId,
        ChapterGenerationRequest request,
        CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        var now = DateTimeOffset.UtcNow;
        var sql = """
        insert into generation_jobs
            (id, external_request_id, story_id, chapter_id, status, request_json, attempts, created_at, updated_at)
        values
            (@id, @external_request_id, @story_id, @chapter_id, @status, @request_json::jsonb, 0, @created_at, @updated_at)
        on conflict (external_request_id) do nothing;
        """;
        await using (var command = new NpgsqlCommand(sql, connection))
        {
            command.Parameters.AddWithValue("id", jobId);
            command.Parameters.AddWithValue("external_request_id", request.ExternalRequestId);
            command.Parameters.AddWithValue("story_id", request.StoryId);
            command.Parameters.AddWithValue("chapter_id", request.ChapterId);
            command.Parameters.AddWithValue("status", JobStatuses.Queued);
            command.Parameters.AddWithValue("request_json", JsonSerializer.Serialize(request, JsonOptions.Default));
            command.Parameters.AddWithValue("created_at", now);
            command.Parameters.AddWithValue("updated_at", now);
            await command.ExecuteNonQueryAsync(cancellationToken);
        }

        var existing = await GetByExternalRequestIdAsync(request.ExternalRequestId, cancellationToken);
        return existing ?? throw new InvalidOperationException("queued job was not created");
    }

    public async Task<GenerationJobRecord?> GetAsync(string jobId, CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand("select * from generation_jobs where id = @id", connection);
        command.Parameters.AddWithValue("id", jobId);
        await using var reader = await command.ExecuteReaderAsync(cancellationToken);
        return await reader.ReadAsync(cancellationToken) ? Read(reader) : null;
    }

    public async Task<GenerationJobRecord?> GetByExternalRequestIdAsync(
        string externalRequestId,
        CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand("select * from generation_jobs where external_request_id = @external_request_id", connection);
        command.Parameters.AddWithValue("external_request_id", externalRequestId);
        await using var reader = await command.ExecuteReaderAsync(cancellationToken);
        return await reader.ReadAsync(cancellationToken) ? Read(reader) : null;
    }

    public async Task MarkRunningAsync(string jobId, DateTimeOffset startedAt, CancellationToken cancellationToken)
    {
        await ExecuteStatusUpdateAsync(
            "update generation_jobs set status = @status, started_at = @started_at, updated_at = @updated_at where id = @id",
            jobId,
            JobStatuses.Running,
            command => command.Parameters.AddWithValue("started_at", startedAt),
            cancellationToken);
    }

    public async Task MarkSucceededAsync(
        string jobId,
        ChapterGenerationResult result,
        int attempts,
        DateTimeOffset finishedAt,
        CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand("""
            update generation_jobs
            set status = @status,
                result_json = @result_json::jsonb,
                attempts = @attempts,
                finished_at = @finished_at,
                updated_at = @updated_at
            where id = @id
            """, connection);
        command.Parameters.AddWithValue("id", jobId);
        command.Parameters.AddWithValue("status", JobStatuses.Succeeded);
        command.Parameters.AddWithValue("result_json", JsonSerializer.Serialize(result, JsonOptions.Default));
        command.Parameters.AddWithValue("attempts", attempts);
        command.Parameters.AddWithValue("finished_at", finishedAt);
        command.Parameters.AddWithValue("updated_at", DateTimeOffset.UtcNow);
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    public async Task MarkFailedAsync(
        string jobId,
        JobError error,
        int attempts,
        DateTimeOffset finishedAt,
        CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand("""
            update generation_jobs
            set status = @status,
                error_json = @error_json::jsonb,
                attempts = @attempts,
                finished_at = @finished_at,
                updated_at = @updated_at
            where id = @id
            """, connection);
        command.Parameters.AddWithValue("id", jobId);
        command.Parameters.AddWithValue("status", JobStatuses.Failed);
        command.Parameters.AddWithValue("error_json", JsonSerializer.Serialize(error, JsonOptions.Default));
        command.Parameters.AddWithValue("attempts", attempts);
        command.Parameters.AddWithValue("finished_at", finishedAt);
        command.Parameters.AddWithValue("updated_at", DateTimeOffset.UtcNow);
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    private async Task<NpgsqlConnection> OpenAsync(CancellationToken cancellationToken)
    {
        var connection = new NpgsqlConnection(_connectionString);
        await connection.OpenAsync(cancellationToken);
        return connection;
    }

    private async Task ExecuteStatusUpdateAsync(
        string sql,
        string jobId,
        string status,
        Action<NpgsqlCommand> addParameters,
        CancellationToken cancellationToken)
    {
        await using var connection = await OpenAsync(cancellationToken);
        await using var command = new NpgsqlCommand(sql, connection);
        command.Parameters.AddWithValue("id", jobId);
        command.Parameters.AddWithValue("status", status);
        command.Parameters.AddWithValue("updated_at", DateTimeOffset.UtcNow);
        addParameters(command);
        await command.ExecuteNonQueryAsync(cancellationToken);
    }

    private static GenerationJobRecord Read(NpgsqlDataReader reader)
    {
        var request = JsonSerializer.Deserialize<ChapterGenerationRequest>(
            reader.GetString(reader.GetOrdinal("request_json")),
            JsonOptions.Default) ?? throw new InvalidOperationException("invalid request_json");
        var resultOrdinal = reader.GetOrdinal("result_json");
        var errorOrdinal = reader.GetOrdinal("error_json");
        var result = reader.IsDBNull(resultOrdinal)
            ? null
            : JsonSerializer.Deserialize<ChapterGenerationResult>(reader.GetString(resultOrdinal), JsonOptions.Default);
        var error = reader.IsDBNull(errorOrdinal)
            ? null
            : JsonSerializer.Deserialize<JobError>(reader.GetString(errorOrdinal), JsonOptions.Default);

        return new GenerationJobRecord(
            reader.GetString(reader.GetOrdinal("id")),
            reader.GetString(reader.GetOrdinal("external_request_id")),
            reader.GetString(reader.GetOrdinal("story_id")),
            reader.GetString(reader.GetOrdinal("chapter_id")),
            reader.GetString(reader.GetOrdinal("status")),
            request,
            result,
            error,
            reader.GetInt32(reader.GetOrdinal("attempts")),
            reader.GetFieldValue<DateTimeOffset>(reader.GetOrdinal("created_at")),
            reader.IsDBNull(reader.GetOrdinal("started_at")) ? null : reader.GetFieldValue<DateTimeOffset>(reader.GetOrdinal("started_at")),
            reader.IsDBNull(reader.GetOrdinal("finished_at")) ? null : reader.GetFieldValue<DateTimeOffset>(reader.GetOrdinal("finished_at")),
            reader.GetFieldValue<DateTimeOffset>(reader.GetOrdinal("updated_at")));
    }
}
```

- [ ] **Step 6: Run schema tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter GenerationJobSchemaTests
```

Expected: all schema tests pass.

- [ ] **Step 7: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/Data Tests/Tianming.GeneratorHost.Tests/Data
git commit -m "feat: add postgres generation job store"
```

Expected: one commit with job store and schema test.

---

### Task 7: Add Redis Queue, Progress, And Locks

**Files:**
- Create: `ServicesHost/Tianming.GeneratorHost/Queue/IChapterJobQueue.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Queue/RedisChapterJobQueue.cs`
- Test: `Tests/Tianming.GeneratorHost.Tests/Queue/RedisKeyTests.cs`

- [ ] **Step 1: Write Redis key tests**

Create `Tests/Tianming.GeneratorHost.Tests/Queue/RedisKeyTests.cs`:

```csharp
using Tianming.GeneratorHost.Queue;

namespace Tianming.GeneratorHost.Tests.Queue;

public sealed class RedisKeyTests
{
    [Fact]
    public void Keys_UseExpectedPrefix()
    {
        Assert.Equal("tmgen:queue:chapter", RedisKeys.ChapterQueue);
        Assert.Equal("tmgen:job:job-1:progress", RedisKeys.Progress("job-1"));
        Assert.Equal("tmgen:job:story-1:lock", RedisKeys.Lock("story-1"));
        Assert.Equal("tmgen:worker:worker-1:heartbeat", RedisKeys.WorkerHeartbeat("worker-1"));
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter RedisKeyTests
```

Expected: fails because queue types do not exist.

- [ ] **Step 3: Add queue interface and key helper**

Create `ServicesHost/Tianming.GeneratorHost/Queue/IChapterJobQueue.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorHost.Queue;

public interface IChapterJobQueue
{
    Task EnqueueAsync(string jobId, CancellationToken cancellationToken);
    Task<string?> DequeueAsync(TimeSpan timeout, CancellationToken cancellationToken);
    Task ReportProgressAsync(string jobId, JobProgress progress, CancellationToken cancellationToken);
    Task<JobProgress?> GetProgressAsync(string jobId, CancellationToken cancellationToken);
    Task<bool> TryAcquireStoryLockAsync(string storyId, string owner, TimeSpan ttl, CancellationToken cancellationToken);
    Task ReleaseStoryLockAsync(string storyId, string owner, CancellationToken cancellationToken);
}

public static class RedisKeys
{
    public const string ChapterQueue = "tmgen:queue:chapter";

    public static string Progress(string jobId) => $"tmgen:job:{jobId}:progress";

    public static string Lock(string storyId) => $"tmgen:job:{storyId}:lock";

    public static string WorkerHeartbeat(string workerId) => $"tmgen:worker:{workerId}:heartbeat";
}
```

- [ ] **Step 4: Add Redis implementation**

Create `ServicesHost/Tianming.GeneratorHost/Queue/RedisChapterJobQueue.cs`:

```csharp
using System.Text.Json;
using StackExchange.Redis;
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorHost.Queue;

public sealed class RedisChapterJobQueue : IChapterJobQueue
{
    private readonly IDatabase _db;

    public RedisChapterJobQueue(IConnectionMultiplexer redis)
    {
        _db = redis.GetDatabase();
    }

    public async Task EnqueueAsync(string jobId, CancellationToken cancellationToken)
    {
        await _db.ListRightPushAsync(RedisKeys.ChapterQueue, jobId);
    }

    public async Task<string?> DequeueAsync(TimeSpan timeout, CancellationToken cancellationToken)
    {
        var deadline = DateTimeOffset.UtcNow.Add(timeout);
        while (DateTimeOffset.UtcNow < deadline && !cancellationToken.IsCancellationRequested)
        {
            var value = await _db.ListLeftPopAsync(RedisKeys.ChapterQueue);
            if (value.HasValue)
                return value.ToString();

            await Task.Delay(TimeSpan.FromMilliseconds(250), cancellationToken);
        }

        return null;
    }

    public async Task ReportProgressAsync(string jobId, JobProgress progress, CancellationToken cancellationToken)
    {
        await _db.StringSetAsync(
            RedisKeys.Progress(jobId),
            JsonSerializer.Serialize(progress, JsonOptions.Default),
            TimeSpan.FromHours(24));
    }

    public async Task<JobProgress?> GetProgressAsync(string jobId, CancellationToken cancellationToken)
    {
        var value = await _db.StringGetAsync(RedisKeys.Progress(jobId));
        if (!value.HasValue)
            return null;

        return JsonSerializer.Deserialize<JobProgress>(value.ToString(), JsonOptions.Default);
    }

    public async Task<bool> TryAcquireStoryLockAsync(
        string storyId,
        string owner,
        TimeSpan ttl,
        CancellationToken cancellationToken)
    {
        return await _db.StringSetAsync(RedisKeys.Lock(storyId), owner, ttl, When.NotExists);
    }

    public async Task ReleaseStoryLockAsync(string storyId, string owner, CancellationToken cancellationToken)
    {
        var key = RedisKeys.Lock(storyId);
        var value = await _db.StringGetAsync(key);
        if (value.HasValue && value.ToString() == owner)
            await _db.KeyDeleteAsync(key);
    }
}
```

- [ ] **Step 5: Run Redis key tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter RedisKeyTests
```

Expected: all Redis key tests pass.

- [ ] **Step 6: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/Queue Tests/Tianming.GeneratorHost.Tests/Queue
git commit -m "feat: add redis chapter job queue"
```

Expected: one commit with Redis queue implementation.

---

### Task 8: Add Worker

**Files:**
- Create: `ServicesHost/Tianming.GeneratorHost/Workers/ChapterGenerationWorker.cs`
- Test: `Tests/Tianming.GeneratorHost.Tests/Workers/ChapterGenerationWorkerTests.cs`

- [ ] **Step 1: Write worker test with fakes**

Create `Tests/Tianming.GeneratorHost.Tests/Workers/ChapterGenerationWorkerTests.cs`:

```csharp
using System.Text.Json.Nodes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;
using Tianming.GeneratorHost.Workers;

namespace Tianming.GeneratorHost.Tests.Workers;

public sealed class ChapterGenerationWorkerTests
{
    [Fact]
    public async Task ProcessOneAsync_GeneratesAndMarksJobSucceeded()
    {
        var request = new ChapterGenerationRequest(
            "external-1",
            "story-1",
            "line-1",
            "volume-1",
            "vol1_ch1",
            1,
            1,
            1,
            "xiuxian",
            new JsonObject(),
            new JsonObject(),
            new JsonObject { ["title"] = "雨夜" },
            Array.Empty<string>(),
            "",
            Array.Empty<JsonObject>(),
            Array.Empty<JsonObject>(),
            Array.Empty<string>(),
            Array.Empty<string>());
        var store = new InMemoryJobStore(request);
        var queue = new InMemoryQueue("job-1");
        var workspace = new StoryWorkspaceService(new WorkspaceOptions { Root = Path.Combine(Path.GetTempPath(), Guid.NewGuid().ToString("N")) });
        var worker = new ChapterGenerationWorker(store, queue, workspace, new FakePublicChapterGenerator());

        var processed = await worker.ProcessOneAsync(CancellationToken.None);

        Assert.True(processed);
        Assert.Equal(JobStatuses.Succeeded, store.Status);
        Assert.NotNull(store.Result);
    }

    private sealed class InMemoryJobStore : IGenerationJobStore
    {
        private readonly ChapterGenerationRequest _request;
        public string Status { get; private set; } = JobStatuses.Queued;
        public ChapterGenerationResult? Result { get; private set; }

        public InMemoryJobStore(ChapterGenerationRequest request) => _request = request;

        public Task EnsureSchemaAsync(CancellationToken cancellationToken) => Task.CompletedTask;

        public Task<GenerationJobRecord> CreateOrGetQueuedAsync(string jobId, ChapterGenerationRequest request, CancellationToken cancellationToken)
            => throw new NotSupportedException();

        public Task<GenerationJobRecord?> GetAsync(string jobId, CancellationToken cancellationToken)
            => Task.FromResult<GenerationJobRecord?>(new GenerationJobRecord(
                jobId,
                _request.ExternalRequestId,
                _request.StoryId,
                _request.ChapterId,
                Status,
                _request,
                Result,
                null,
                Result?.Attempts ?? 0,
                DateTimeOffset.UtcNow,
                null,
                null,
                DateTimeOffset.UtcNow));

        public Task<GenerationJobRecord?> GetByExternalRequestIdAsync(string externalRequestId, CancellationToken cancellationToken)
            => Task.FromResult<GenerationJobRecord?>(null);

        public Task MarkRunningAsync(string jobId, DateTimeOffset startedAt, CancellationToken cancellationToken)
        {
            Status = JobStatuses.Running;
            return Task.CompletedTask;
        }

        public Task MarkSucceededAsync(string jobId, ChapterGenerationResult result, int attempts, DateTimeOffset finishedAt, CancellationToken cancellationToken)
        {
            Status = JobStatuses.Succeeded;
            Result = result;
            return Task.CompletedTask;
        }

        public Task MarkFailedAsync(string jobId, JobError error, int attempts, DateTimeOffset finishedAt, CancellationToken cancellationToken)
        {
            Status = JobStatuses.Failed;
            return Task.CompletedTask;
        }
    }

    private sealed class InMemoryQueue : IChapterJobQueue
    {
        private string? _jobId;

        public InMemoryQueue(string jobId) => _jobId = jobId;

        public Task EnqueueAsync(string jobId, CancellationToken cancellationToken)
            => Task.CompletedTask;

        public Task<string?> DequeueAsync(TimeSpan timeout, CancellationToken cancellationToken)
        {
            var jobId = _jobId;
            _jobId = null;
            return Task.FromResult(jobId);
        }

        public Task ReportProgressAsync(string jobId, JobProgress progress, CancellationToken cancellationToken)
            => Task.CompletedTask;

        public Task<JobProgress?> GetProgressAsync(string jobId, CancellationToken cancellationToken)
            => Task.FromResult<JobProgress?>(null);

        public Task<bool> TryAcquireStoryLockAsync(string storyId, string owner, TimeSpan ttl, CancellationToken cancellationToken)
            => Task.FromResult(true);

        public Task ReleaseStoryLockAsync(string storyId, string owner, CancellationToken cancellationToken)
            => Task.CompletedTask;
    }
}
```

- [ ] **Step 2: Run tests and verify they fail**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter ChapterGenerationWorkerTests
```

Expected: fails because `ChapterGenerationWorker` does not exist.

- [ ] **Step 3: Add worker**

Create `ServicesHost/Tianming.GeneratorHost/Workers/ChapterGenerationWorker.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;

namespace Tianming.GeneratorHost.Workers;

public sealed class ChapterGenerationWorker : BackgroundService, IJobProgressReporter
{
    private readonly IGenerationJobStore _store;
    private readonly IChapterJobQueue _queue;
    private readonly StoryWorkspaceService _workspaceService;
    private readonly IPublicChapterGenerator _generator;

    public ChapterGenerationWorker(
        IGenerationJobStore store,
        IChapterJobQueue queue,
        StoryWorkspaceService workspaceService,
        IPublicChapterGenerator generator)
    {
        _store = store;
        _queue = queue;
        _workspaceService = workspaceService;
        _generator = generator;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await _store.EnsureSchemaAsync(stoppingToken);

        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessOneAsync(stoppingToken);
        }
    }

    public async Task<bool> ProcessOneAsync(CancellationToken cancellationToken)
    {
        var jobId = await _queue.DequeueAsync(TimeSpan.FromSeconds(1), cancellationToken);
        if (jobId is null)
            return false;

        var job = await _store.GetAsync(jobId, cancellationToken);
        if (job is null)
            return false;

        var lockOwner = $"worker-{Environment.MachineName}-{Guid.NewGuid():N}";
        var locked = await _queue.TryAcquireStoryLockAsync(job.StoryId, lockOwner, TimeSpan.FromMinutes(30), cancellationToken);
        if (!locked)
        {
            await _queue.EnqueueAsync(jobId, cancellationToken);
            return false;
        }

        try
        {
            await _store.MarkRunningAsync(jobId, DateTimeOffset.UtcNow, cancellationToken);
            await ReportAsync(jobId, new JobProgress("workspace", 1, "preparing story workspace"), cancellationToken);
            var workspace = _workspaceService.Prepare(job.StoryId);
            await ReportAsync(jobId, new JobProgress("generation", 1, "generating chapter"), cancellationToken);
            var result = await _generator.GenerateAsync(job.Request, workspace, this, cancellationToken);
            await _store.MarkSucceededAsync(jobId, result, result.Attempts, DateTimeOffset.UtcNow, cancellationToken);
            return true;
        }
        catch (Exception ex)
        {
            var error = new JobError("internal_error", ex.Message, true);
            await _store.MarkFailedAsync(jobId, error, 1, DateTimeOffset.UtcNow, cancellationToken);
            return true;
        }
        finally
        {
            await _queue.ReleaseStoryLockAsync(job.StoryId, lockOwner, cancellationToken);
        }
    }

    public Task ReportAsync(string jobId, JobProgress progress, CancellationToken cancellationToken)
        => _queue.ReportProgressAsync(jobId, progress, cancellationToken);
}
```

- [ ] **Step 4: Run worker tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter ChapterGenerationWorkerTests
```

Expected: all worker tests pass.

- [ ] **Step 5: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/Workers Tests/Tianming.GeneratorHost.Tests/Workers
git commit -m "feat: add chapter generation worker"
```

Expected: one commit with worker and tests.

---

### Task 9: Add HTTP Endpoints And Host Wiring

**Files:**
- Modify: `ServicesHost/Tianming.GeneratorHost/Program.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Api/ChapterGenerationEndpoints.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Api/JobStatusEndpoints.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/appsettings.json`
- Test: `Tests/Tianming.GeneratorHost.Tests/Api/EndpointMappingTests.cs`

- [ ] **Step 1: Write endpoint mapping test**

Create `Tests/Tianming.GeneratorHost.Tests/Api/EndpointMappingTests.cs`:

```csharp
using System.Net;
using Microsoft.AspNetCore.Mvc.Testing;

namespace Tianming.GeneratorHost.Tests.Api;

public sealed class EndpointMappingTests
{
    [Fact]
    public async Task HealthEndpoint_ReturnsOk()
    {
        await using var factory = new WebApplicationFactory<Program>();
        var client = factory.CreateClient();

        var response = await client.GetAsync("/health");

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);
    }
}
```

- [ ] **Step 2: Run test and verify it fails**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter EndpointMappingTests
```

Expected: fails because `Program` is not public or endpoint wiring is incomplete.

- [ ] **Step 3: Add endpoint classes**

Create `ServicesHost/Tianming.GeneratorHost/Api/ChapterGenerationEndpoints.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;

namespace Tianming.GeneratorHost.Api;

public static class ChapterGenerationEndpoints
{
    public static IEndpointRouteBuilder MapChapterGenerationEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapPost("/v1/chapters/generate", async (
            ChapterGenerationRequest request,
            IGenerationJobStore store,
            IChapterJobQueue queue,
            CancellationToken cancellationToken) =>
        {
            if (string.IsNullOrWhiteSpace(request.ExternalRequestId)
                || string.IsNullOrWhiteSpace(request.StoryId)
                || string.IsNullOrWhiteSpace(request.ChapterId))
            {
                return Results.BadRequest(new JobError("invalid_request", "external_request_id, story_id, and chapter_id are required", false));
            }

            var existing = await store.GetByExternalRequestIdAsync(request.ExternalRequestId, cancellationToken);
            if (existing is not null)
            {
                return Results.Ok(new CreateJobResponse(existing.Id, existing.Status));
            }

            var jobId = $"tmgen_{Guid.NewGuid():N}";
            var job = await store.CreateOrGetQueuedAsync(jobId, request, cancellationToken);
            if (job.Status == JobStatuses.Queued)
                await queue.EnqueueAsync(job.Id, cancellationToken);

            return Results.Accepted($"/v1/jobs/{job.Id}", new CreateJobResponse(job.Id, job.Status));
        });

        return app;
    }
}
```

Create `ServicesHost/Tianming.GeneratorHost/Api/JobStatusEndpoints.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;

namespace Tianming.GeneratorHost.Api;

public static class JobStatusEndpoints
{
    public static IEndpointRouteBuilder MapJobStatusEndpoints(this IEndpointRouteBuilder app)
    {
        app.MapGet("/v1/jobs/{jobId}", async (
            string jobId,
            IGenerationJobStore store,
            IChapterJobQueue queue,
            CancellationToken cancellationToken) =>
        {
            var job = await store.GetAsync(jobId, cancellationToken);
            if (job is null)
                return Results.NotFound(new JobError("job_not_found", $"job {jobId} was not found", false));

            if (job.Status == JobStatuses.Succeeded && job.Result is not null)
                return Results.Ok(JobStatusResponse.Succeeded(job.Id, job.Result));

            if (job.Status == JobStatuses.Failed && job.Error is not null)
                return Results.Ok(JobStatusResponse.Failed(job.Id, job.Error));

            var progress = await queue.GetProgressAsync(job.Id, cancellationToken);
            if (job.Status == JobStatuses.Running)
                return Results.Ok(JobStatusResponse.Running(job.Id, progress ?? new JobProgress("running", job.Attempts, "running")));

            return Results.Ok(JobStatusResponse.Queued(job.Id));
        });

        return app;
    }
}
```

- [ ] **Step 4: Wire Program**

Replace `ServicesHost/Tianming.GeneratorHost/Program.cs` with:

```csharp
using StackExchange.Redis;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Workspace;
using Tianming.GeneratorHost.Api;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;
using Tianming.GeneratorHost.Workers;

var builder = WebApplication.CreateBuilder(args);

builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.PropertyNamingPolicy = System.Text.Json.JsonNamingPolicy.SnakeCaseLower;
    options.SerializerOptions.DictionaryKeyPolicy = System.Text.Json.JsonNamingPolicy.SnakeCaseLower;
    options.SerializerOptions.PropertyNameCaseInsensitive = true;
});

builder.Services.AddSingleton(new WorkspaceOptions
{
    Root = builder.Configuration.GetValue<string>("WorkspaceRoot")
        ?? Path.Combine(Path.GetTempPath(), "tianming-generator")
});
builder.Services.AddSingleton<StoryWorkspaceService>();
builder.Services.AddSingleton<IGenerationJobStore, PostgresGenerationJobStore>();
builder.Services.AddSingleton<IConnectionMultiplexer>(_ =>
{
    var redis = builder.Configuration.GetConnectionString("Redis")
        ?? throw new InvalidOperationException("ConnectionStrings:Redis is required");
    return ConnectionMultiplexer.Connect(redis);
});
builder.Services.AddSingleton<IChapterJobQueue, RedisChapterJobQueue>();
builder.Services.AddSingleton<IPublicChapterGenerator, FakePublicChapterGenerator>();
builder.Services.AddHostedService<ChapterGenerationWorker>();

var app = builder.Build();

app.MapGet("/health", () => Results.Ok(new { status = "ok" }));
app.MapChapterGenerationEndpoints();
app.MapJobStatusEndpoints();

app.Run();

public partial class Program
{
}
```

- [ ] **Step 5: Add appsettings**

Create `ServicesHost/Tianming.GeneratorHost/appsettings.json`:

```json
{
  "WorkspaceRoot": "/tmp/tianming-generator",
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Port=5432;Database=tianming_generator;Username=postgres;Password=postgres",
    "Redis": "localhost:6379"
  }
}
```

- [ ] **Step 6: Run endpoint test**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter EndpointMappingTests
```

Expected: `HealthEndpoint_ReturnsOk` passes.

- [ ] **Step 7: Build solution**

Run:

```bash
dotnet build Tianming.GeneratorService.sln
```

Expected: build succeeds.

- [ ] **Step 8: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/Program.cs ServicesHost/Tianming.GeneratorHost/Api ServicesHost/Tianming.GeneratorHost/appsettings.json Tests/Tianming.GeneratorHost.Tests/Api
git commit -m "feat: expose generator service endpoints"
```

Expected: one commit with API wiring.

---

### Task 10: Add Local Docker Compose And Smoke Test Script

**Files:**
- Create: `ServicesHost/Tianming.GeneratorHost/docker-compose.yml`
- Create: `ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh`
- Create: `ServicesHost/Tianming.GeneratorHost/README.md`

- [ ] **Step 1: Add Docker Compose**

Create `ServicesHost/Tianming.GeneratorHost/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: tianming_generator
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - tmgen-postgres:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

volumes:
  tmgen-postgres:
```

- [ ] **Step 2: Add smoke script**

Create `ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL="${BASE_URL:-http://localhost:5000}"

response="$(
  curl -sS -X POST "$BASE_URL/v1/chapters/generate" \
    -H 'content-type: application/json' \
    -d '{
      "external_request_id": "smoke-req-1",
      "story_id": "smoke-story",
      "story_line_id": "smoke-line",
      "volume_id": "smoke-volume",
      "chapter_id": "vol1_ch1",
      "chapter_no": 1,
      "volume_no": 1,
      "volume_chapter_no": 1,
      "channel": "xiuxian",
      "story": {"title": "烟测故事"},
      "volume": {"title": "第一卷"},
      "chapter_plan": {"title": "雨夜旧案"},
      "recent_chapters": [],
      "previous_chapter_tail": "",
      "volume_archives": [],
      "vector_recall_fragments": [],
      "ledger_entity_ids": [],
      "hard_constraints": []
    }'
)"

job_id="$(python -c 'import json,sys; print(json.load(sys.stdin)["job_id"])' <<< "$response")"
echo "job_id=$job_id"

for _ in $(seq 1 40); do
  status="$(
    curl -sS "$BASE_URL/v1/jobs/$job_id"
  )"
  echo "$status"
  state="$(python -c 'import json,sys; print(json.load(sys.stdin)["status"])' <<< "$status")"
  if [[ "$state" == "succeeded" ]]; then
    exit 0
  fi
  if [[ "$state" == "failed" ]]; then
    exit 1
  fi
  sleep 1
done

echo "timed out waiting for job"
exit 1
```

Run:

```bash
chmod +x ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh
```

- [ ] **Step 3: Add README**

Create `ServicesHost/Tianming.GeneratorHost/README.md`:

```markdown
# Tianming Generator Host

Linux-safe asynchronous generator service for `novel-center`.

## Local Run

Start dependencies:

```bash
cd ServicesHost/Tianming.GeneratorHost
docker compose up -d
```

Start service:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
ASPNETCORE_URLS=http://localhost:5000 dotnet run --project ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj
```

Run smoke test:

```bash
ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh
```

The first release uses `FakePublicChapterGenerator` until real Tianming core extraction is wired.
```

- [ ] **Step 4: Run smoke test manually**

Terminal 1:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer/ServicesHost/Tianming.GeneratorHost
docker compose up -d
```

Terminal 2:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
ASPNETCORE_URLS=http://localhost:5000 dotnet run --project ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj
```

Terminal 3:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh
```

Expected: script exits `0` and prints a `succeeded` job status with `title`, `body`, `changes`, `attempts`, and `warnings`.

- [ ] **Step 5: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/docker-compose.yml ServicesHost/Tianming.GeneratorHost/scripts ServicesHost/Tianming.GeneratorHost/README.md
git commit -m "docs: add generator host smoke workflow"
```

Expected: one commit with local run docs and smoke script.

---

### Task 11: Add Real Generator Facade Placeholder With Explicit Unsupported Mode

**Files:**
- Create: `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPublicChapterGenerator.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorTests.cs`
- Modify: `ServicesHost/Tianming.GeneratorHost/Program.cs`

- [ ] **Step 1: Write unsupported-mode test**

Create `Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorTests.cs`:

```csharp
using System.Text.Json.Nodes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Tests.Generation;

public sealed class TianmingPublicChapterGeneratorTests
{
    [Fact]
    public async Task GenerateAsync_ReportsUnsupportedUntilCoreExtractionIsWired()
    {
        var generator = new TianmingPublicChapterGenerator();
        var request = new ChapterGenerationRequest(
            "req-1",
            "story-1",
            "line-1",
            "volume-1",
            "vol1_ch1",
            1,
            1,
            1,
            "xiuxian",
            new JsonObject(),
            new JsonObject(),
            new JsonObject(),
            Array.Empty<string>(),
            "",
            Array.Empty<JsonObject>(),
            Array.Empty<JsonObject>(),
            Array.Empty<string>(),
            Array.Empty<string>());
        var workspace = new StoryWorkspace("story-1", "/tmp/story-1", "/tmp/story-1/chapters", "/tmp/story-1/guides", "/tmp/story-1/indexes", "/tmp/story-1/archives", "/tmp/story-1/runtime");

        var ex = await Assert.ThrowsAsync<NotSupportedException>(() =>
            generator.GenerateAsync(request, workspace, NullJobProgressReporter.Instance, CancellationToken.None));

        Assert.Contains("real Tianming core extraction is not wired", ex.Message);
    }
}
```

- [ ] **Step 2: Run test and verify it fails**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter TianmingPublicChapterGeneratorTests
```

Expected: fails because `TianmingPublicChapterGenerator` does not exist.

- [ ] **Step 3: Add real generator facade**

Create `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPublicChapterGenerator.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Generation;

public sealed class TianmingPublicChapterGenerator : IPublicChapterGenerator
{
    public Task<ChapterGenerationResult> GenerateAsync(
        ChapterGenerationRequest request,
        StoryWorkspace workspace,
        IJobProgressReporter progressReporter,
        CancellationToken cancellationToken)
    {
        throw new NotSupportedException(
            "real Tianming core extraction is not wired; set Generator:Mode=Fake until extraction tasks are complete");
    }
}
```

- [ ] **Step 4: Add mode switch in Program**

Modify the generator registration in `ServicesHost/Tianming.GeneratorHost/Program.cs`:

```csharp
var generatorMode = builder.Configuration.GetValue<string>("Generator:Mode") ?? "Fake";
if (string.Equals(generatorMode, "Real", StringComparison.OrdinalIgnoreCase))
{
    builder.Services.AddSingleton<IPublicChapterGenerator, TianmingPublicChapterGenerator>();
}
else
{
    builder.Services.AddSingleton<IPublicChapterGenerator, FakePublicChapterGenerator>();
}
```

Replace the previous single-line registration:

```csharp
builder.Services.AddSingleton<IPublicChapterGenerator, FakePublicChapterGenerator>();
```

- [ ] **Step 5: Add appsettings mode**

Modify `ServicesHost/Tianming.GeneratorHost/appsettings.json`:

```json
{
  "WorkspaceRoot": "/tmp/tianming-generator",
  "Generator": {
    "Mode": "Fake"
  },
  "ConnectionStrings": {
    "Postgres": "Host=localhost;Port=5432;Database=tianming_generator;Username=postgres;Password=postgres",
    "Redis": "localhost:6379"
  }
}
```

- [ ] **Step 6: Run tests**

Run:

```bash
dotnet test Tianming.GeneratorService.sln
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Generation/TianmingPublicChapterGenerator.cs ServicesHost/Tianming.GeneratorHost/Program.cs ServicesHost/Tianming.GeneratorHost/appsettings.json Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorTests.cs
git commit -m "feat: add real generator mode switch"
```

Expected: one commit with explicit fake/real generator selection.

---

### Task 12: Extract Linux-Safe Tianming Models Into GeneratorCore

**Files:**
- Create or modify: `ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj`
- Create linked or copied source files under `ServicesHost/Tianming.GeneratorCore/Tianming/`
- Test: `Tests/Tianming.GeneratorCore.Tests/Tianming/TianmingModelCompileTests.cs`

- [ ] **Step 1: Write compile-level test**

Create `Tests/Tianming.GeneratorCore.Tests/Tianming/TianmingModelCompileTests.cs`:

```csharp
using Tianming.GeneratorCore.Changes;

namespace Tianming.GeneratorCore.Tests.Tianming;

public sealed class TianmingModelCompileTests
{
    [Fact]
    public void TianmingChanges_ModelCanRepresentNineTopLevelFields()
    {
        var changes = new TianmingChanges
        {
            TimeProgression = new TimeProgressionChange()
        };

        Assert.Empty(changes.CharacterStateChanges);
        Assert.Empty(changes.ConflictProgress);
        Assert.Empty(changes.ForeshadowingActions);
        Assert.Empty(changes.NewPlotPoints);
        Assert.Empty(changes.LocationStateChanges);
        Assert.Empty(changes.FactionStateChanges);
        Assert.NotNull(changes.TimeProgression);
        Assert.Empty(changes.CharacterMovements);
        Assert.Empty(changes.ItemTransfers);
    }
}
```

- [ ] **Step 2: Run compile-level test**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter TianmingModelCompileTests
```

Expected: passes using the service-side minimal model from Task 3.

- [ ] **Step 3: Decide source extraction set**

Start with only Linux-safe files that do not reference WPF, `TM.App`, `ServiceLocator`, `GlobalToast`, or `System.Windows`:

```text
Services/Modules/ProjectData/Models/Tracking/TrackingChangeModels.cs
Services/Modules/ProjectData/Models/Tracking/GenerationResult.cs
Services/Modules/ProjectData/Models/Tracking/GateModels.cs
Services/Framework/AI/SemanticKernel/MultiLayerIndexDocumentBuilder.cs
Services/Framework/AI/SemanticKernel/Plugins/ShortFormBlueprintPolicy.cs
Services/Framework/AI/SemanticKernel/Plugins/FactSnapshotConstraintFormatter.cs
```

If a file references UI-only code, do not link it. Extract the needed model shape into `GeneratorCore` instead.

- [ ] **Step 4: Add linked compile items incrementally**

Modify `ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj` with one safe file at a time. Example for one file:

```xml
<ItemGroup>
  <Compile Include="..\..\Services\Modules\ProjectData\Models\Tracking\TrackingChangeModels.cs" Link="Tianming\TrackingChangeModels.cs" />
</ItemGroup>
```

After each linked file:

```bash
dotnet build ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj
```

Expected: build succeeds after each link. If a link fails because of UI or desktop dependency, remove that link and keep the local service-side model.

- [ ] **Step 5: Run tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj
```

Expected: all core tests pass.

- [ ] **Step 6: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Tianming.GeneratorCore.csproj Tests/Tianming.GeneratorCore.Tests/Tianming
git commit -m "refactor: link linux-safe tianming generation models"
```

Expected: one commit with only Linux-safe model extraction.

---

### Task 13: Wire Real Generator Behind Facade

**Files:**
- Modify: `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPublicChapterGenerator.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPromptBuilder.cs`
- Create: `ServicesHost/Tianming.GeneratorCore/Generation/IAITextGenerationClient.cs`
- Test: `Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorRealModeTests.cs`

- [ ] **Step 1: Write real-mode test with fake AI client**

Create `Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorRealModeTests.cs`:

```csharp
using System.Text.Json.Nodes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Tests.Generation;

public sealed class TianmingPublicChapterGeneratorRealModeTests
{
    [Fact]
    public async Task GenerateAsync_ParsesBodyAndChangesFromAiOutput()
    {
        var ai = new StaticAITextGenerationClient("""
        第一章 雨夜

        陆青衣在雨夜推开禁库旧门。

        ---CHANGES---
        {
          "CharacterStateChanges": [],
          "ConflictProgress": [],
          "ForeshadowingActions": [],
          "NewPlotPoints": [
            {
              "Keywords": ["禁库"],
              "Context": "陆青衣发现禁库旧门",
              "InvolvedCharacters": [],
              "Importance": "important",
              "Storyline": "main"
            }
          ],
          "LocationStateChanges": [],
          "FactionStateChanges": [],
          "TimeProgression": {
            "TimePeriod": "雨夜",
            "ElapsedTime": "一刻钟",
            "KeyTimeEvent": "禁库旧门开启",
            "Importance": "important"
          },
          "CharacterMovements": [],
          "ItemTransfers": []
        }
        """);
        var generator = new TianmingPublicChapterGenerator(ai, new TianmingPromptBuilder());
        var request = new ChapterGenerationRequest(
            "req-1",
            "story-1",
            "line-1",
            "volume-1",
            "vol1_ch1",
            1,
            1,
            1,
            "xiuxian",
            new JsonObject(),
            new JsonObject(),
            new JsonObject { ["title"] = "雨夜" },
            Array.Empty<string>(),
            "",
            Array.Empty<JsonObject>(),
            Array.Empty<JsonObject>(),
            Array.Empty<string>(),
            Array.Empty<string>());
        var workspace = new StoryWorkspace("story-1", "/tmp/story-1", "/tmp/story-1/chapters", "/tmp/story-1/guides", "/tmp/story-1/indexes", "/tmp/story-1/archives", "/tmp/story-1/runtime");

        var result = await generator.GenerateAsync(request, workspace, NullJobProgressReporter.Instance, CancellationToken.None);

        Assert.Equal("雨夜", result.Title);
        Assert.Contains("陆青衣", result.Body);
        Assert.Contains(result.Changes.PlotUpdates, x => x["context"]!.Equals("陆青衣发现禁库旧门"));
    }

    private sealed class StaticAITextGenerationClient : IAITextGenerationClient
    {
        private readonly string _output;

        public StaticAITextGenerationClient(string output) => _output = output;

        public Task<string> GenerateAsync(string prompt, CancellationToken cancellationToken)
            => Task.FromResult(_output);
    }
}
```

- [ ] **Step 2: Run test and verify it fails**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter TianmingPublicChapterGeneratorRealModeTests
```

Expected: fails because real constructor, prompt builder, and AI interface do not exist.

- [ ] **Step 3: Add AI interface**

Create `ServicesHost/Tianming.GeneratorCore/Generation/IAITextGenerationClient.cs`:

```csharp
namespace Tianming.GeneratorCore.Generation;

public interface IAITextGenerationClient
{
    Task<string> GenerateAsync(string prompt, CancellationToken cancellationToken);
}
```

- [ ] **Step 4: Add prompt builder**

Create `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPromptBuilder.cs`:

```csharp
using System.Text;
using System.Text.Json;
using Tianming.GeneratorCore.Contracts;

namespace Tianming.GeneratorCore.Generation;

public sealed class TianmingPromptBuilder
{
    public string Build(ChapterGenerationRequest request)
    {
        var sb = new StringBuilder();
        sb.AppendLine("<chapter_generation_task>");
        sb.AppendLine("请生成一章公共主线小说正文。正文后必须输出 ---CHANGES--- 和 JSON。");
        sb.AppendLine("JSON 必须包含 CharacterStateChanges、ConflictProgress、ForeshadowingActions、NewPlotPoints、LocationStateChanges、FactionStateChanges、TimeProgression、CharacterMovements、ItemTransfers。");
        sb.AppendLine("<request>");
        sb.AppendLine(JsonSerializer.Serialize(request, JsonOptions.Default));
        sb.AppendLine("</request>");
        sb.AppendLine("</chapter_generation_task>");
        return sb.ToString();
    }
}
```

- [ ] **Step 5: Replace real generator facade with parser implementation**

Replace `ServicesHost/Tianming.GeneratorCore/Generation/TianmingPublicChapterGenerator.cs` with:

```csharp
using System.Text.Json;
using Tianming.GeneratorCore.Changes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Generation;

public sealed class TianmingPublicChapterGenerator : IPublicChapterGenerator
{
    private const string ChangesMarker = "---CHANGES---";
    private readonly IAITextGenerationClient _ai;
    private readonly TianmingPromptBuilder _promptBuilder;

    public TianmingPublicChapterGenerator(IAITextGenerationClient ai, TianmingPromptBuilder promptBuilder)
    {
        _ai = ai;
        _promptBuilder = promptBuilder;
    }

    public async Task<ChapterGenerationResult> GenerateAsync(
        ChapterGenerationRequest request,
        StoryWorkspace workspace,
        IJobProgressReporter progressReporter,
        CancellationToken cancellationToken)
    {
        await progressReporter.ReportAsync(request.ExternalRequestId, new JobProgress("prompt", 1, "building tianming prompt"), cancellationToken);
        var prompt = _promptBuilder.Build(request);
        await progressReporter.ReportAsync(request.ExternalRequestId, new JobProgress("ai_call", 1, "calling text generation model"), cancellationToken);
        var raw = await _ai.GenerateAsync(prompt, cancellationToken);
        var markerIndex = raw.IndexOf(ChangesMarker, StringComparison.Ordinal);
        if (markerIndex < 0)
            throw new InvalidOperationException("changes_parse_failed: output missing ---CHANGES--- marker");

        var body = raw[..markerIndex].Trim();
        var changesJson = raw[(markerIndex + ChangesMarker.Length)..].Trim();
        var changes = JsonSerializer.Deserialize<TianmingChanges>(changesJson)
            ?? throw new InvalidOperationException("changes_parse_failed: output changes JSON is empty");
        var title = ReadString(request.ChapterPlan, "title") ?? $"Chapter {request.ChapterNo}";
        var summary = changes.NewPlotPoints.FirstOrDefault()?.Context ?? $"{title} 已生成";
        var novelCenterChanges = ChangesConverter.ToNovelCenter(changes, summary, "下一步如何推进？");

        return new ChapterGenerationResult(
            title,
            body,
            novelCenterChanges,
            raw,
            1,
            Array.Empty<string>());
    }

    private static string? ReadString(System.Text.Json.Nodes.JsonObject value, string key)
    {
        return value.TryGetPropertyValue(key, out var node) ? node?.GetValue<string>() : null;
    }
}
```

- [ ] **Step 6: Update unsupported-mode test**

Modify `Tests/Tianming.GeneratorCore.Tests/Generation/TianmingPublicChapterGeneratorTests.cs` so it validates parser failures instead of unsupported mode:

```csharp
using System.Text.Json.Nodes;
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorCore.Generation;
using Tianming.GeneratorCore.Progress;
using Tianming.GeneratorCore.Workspace;

namespace Tianming.GeneratorCore.Tests.Generation;

public sealed class TianmingPublicChapterGeneratorTests
{
    [Fact]
    public async Task GenerateAsync_FailsWhenChangesMarkerIsMissing()
    {
        var generator = new TianmingPublicChapterGenerator(
            new StaticAITextGenerationClient("正文但没有变更标记"),
            new TianmingPromptBuilder());
        var request = new ChapterGenerationRequest(
            "req-1",
            "story-1",
            "line-1",
            "volume-1",
            "vol1_ch1",
            1,
            1,
            1,
            "xiuxian",
            new JsonObject(),
            new JsonObject(),
            new JsonObject(),
            Array.Empty<string>(),
            "",
            Array.Empty<JsonObject>(),
            Array.Empty<JsonObject>(),
            Array.Empty<string>(),
            Array.Empty<string>());
        var workspace = new StoryWorkspace("story-1", "/tmp/story-1", "/tmp/story-1/chapters", "/tmp/story-1/guides", "/tmp/story-1/indexes", "/tmp/story-1/archives", "/tmp/story-1/runtime");

        var ex = await Assert.ThrowsAsync<InvalidOperationException>(() =>
            generator.GenerateAsync(request, workspace, NullJobProgressReporter.Instance, CancellationToken.None));

        Assert.Contains("changes_parse_failed", ex.Message);
    }

    private sealed class StaticAITextGenerationClient : IAITextGenerationClient
    {
        private readonly string _output;

        public StaticAITextGenerationClient(string output) => _output = output;

        public Task<string> GenerateAsync(string prompt, CancellationToken cancellationToken)
            => Task.FromResult(_output);
    }
}
```

- [ ] **Step 7: Run real-mode tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorCore.Tests/Tianming.GeneratorCore.Tests.csproj --filter "TianmingPublicChapterGenerator"
```

Expected: all `TianmingPublicChapterGenerator` tests pass.

- [ ] **Step 8: Commit**

```bash
git add ServicesHost/Tianming.GeneratorCore/Generation Tests/Tianming.GeneratorCore.Tests/Generation
git commit -m "feat: parse real tianming generator output"
```

Expected: one commit with real facade parser and tests.

---

### Task 14: Add Worker Recovery For Stale Running Jobs

**Files:**
- Modify: `ServicesHost/Tianming.GeneratorHost/Data/IGenerationJobStore.cs`
- Modify: `ServicesHost/Tianming.GeneratorHost/Data/PostgresGenerationJobStore.cs`
- Create: `ServicesHost/Tianming.GeneratorHost/Workers/StaleJobRecoveryService.cs`
- Test: `Tests/Tianming.GeneratorHost.Tests/Workers/StaleJobRecoveryServiceTests.cs`

- [ ] **Step 1: Write stale recovery test**

Create `Tests/Tianming.GeneratorHost.Tests/Workers/StaleJobRecoveryServiceTests.cs`:

```csharp
using Tianming.GeneratorCore.Contracts;
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;
using Tianming.GeneratorHost.Workers;

namespace Tianming.GeneratorHost.Tests.Workers;

public sealed class StaleJobRecoveryServiceTests
{
    [Fact]
    public async Task RecoverOnceAsync_RequeuesStaleRunningJobs()
    {
        var store = new FakeRecoveryStore(new[] { "job-1", "job-2" });
        var queue = new FakeQueue();
        var service = new StaleJobRecoveryService(store, queue, TimeSpan.FromMinutes(10));

        var count = await service.RecoverOnceAsync(CancellationToken.None);

        Assert.Equal(2, count);
        Assert.Equal(new[] { "job-1", "job-2" }, queue.Enqueued);
    }

    private sealed class FakeRecoveryStore : IGenerationJobStore
    {
        private readonly IReadOnlyList<string> _stale;

        public FakeRecoveryStore(IReadOnlyList<string> stale) => _stale = stale;

        public Task<IReadOnlyList<string>> FindStaleRunningJobIdsAsync(DateTimeOffset staleBefore, CancellationToken cancellationToken)
            => Task.FromResult(_stale);

        public Task ResetToQueuedAsync(string jobId, CancellationToken cancellationToken)
            => Task.CompletedTask;

        public Task EnsureSchemaAsync(CancellationToken cancellationToken) => Task.CompletedTask;
        public Task<GenerationJobRecord> CreateOrGetQueuedAsync(string jobId, ChapterGenerationRequest request, CancellationToken cancellationToken) => throw new NotSupportedException();
        public Task<GenerationJobRecord?> GetAsync(string jobId, CancellationToken cancellationToken) => throw new NotSupportedException();
        public Task<GenerationJobRecord?> GetByExternalRequestIdAsync(string externalRequestId, CancellationToken cancellationToken) => throw new NotSupportedException();
        public Task MarkRunningAsync(string jobId, DateTimeOffset startedAt, CancellationToken cancellationToken) => throw new NotSupportedException();
        public Task MarkSucceededAsync(string jobId, ChapterGenerationResult result, int attempts, DateTimeOffset finishedAt, CancellationToken cancellationToken) => throw new NotSupportedException();
        public Task MarkFailedAsync(string jobId, JobError error, int attempts, DateTimeOffset finishedAt, CancellationToken cancellationToken) => throw new NotSupportedException();
    }

    private sealed class FakeQueue : IChapterJobQueue
    {
        public List<string> Enqueued { get; } = new();
        public Task EnqueueAsync(string jobId, CancellationToken cancellationToken) { Enqueued.Add(jobId); return Task.CompletedTask; }
        public Task<string?> DequeueAsync(TimeSpan timeout, CancellationToken cancellationToken) => Task.FromResult<string?>(null);
        public Task ReportProgressAsync(string jobId, JobProgress progress, CancellationToken cancellationToken) => Task.CompletedTask;
        public Task<JobProgress?> GetProgressAsync(string jobId, CancellationToken cancellationToken) => Task.FromResult<JobProgress?>(null);
        public Task<bool> TryAcquireStoryLockAsync(string storyId, string owner, TimeSpan ttl, CancellationToken cancellationToken) => Task.FromResult(true);
        public Task ReleaseStoryLockAsync(string storyId, string owner, CancellationToken cancellationToken) => Task.CompletedTask;
    }
}
```

- [ ] **Step 2: Run test and verify it fails**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter StaleJobRecoveryServiceTests
```

Expected: fails because recovery service and store methods do not exist.

- [ ] **Step 3: Extend job store interface**

Add to `ServicesHost/Tianming.GeneratorHost/Data/IGenerationJobStore.cs`:

```csharp
Task<IReadOnlyList<string>> FindStaleRunningJobIdsAsync(
    DateTimeOffset staleBefore,
    CancellationToken cancellationToken);

Task ResetToQueuedAsync(string jobId, CancellationToken cancellationToken);
```

- [ ] **Step 4: Implement Postgres stale methods**

Add to `ServicesHost/Tianming.GeneratorHost/Data/PostgresGenerationJobStore.cs`:

```csharp
public async Task<IReadOnlyList<string>> FindStaleRunningJobIdsAsync(
    DateTimeOffset staleBefore,
    CancellationToken cancellationToken)
{
    await using var connection = await OpenAsync(cancellationToken);
    await using var command = new NpgsqlCommand("""
        select id from generation_jobs
        where status = @status and updated_at < @stale_before
        order by updated_at asc
        """, connection);
    command.Parameters.AddWithValue("status", JobStatuses.Running);
    command.Parameters.AddWithValue("stale_before", staleBefore);
    await using var reader = await command.ExecuteReaderAsync(cancellationToken);
    var ids = new List<string>();
    while (await reader.ReadAsync(cancellationToken))
        ids.Add(reader.GetString(0));
    return ids;
}

public async Task ResetToQueuedAsync(string jobId, CancellationToken cancellationToken)
{
    await using var connection = await OpenAsync(cancellationToken);
    await using var command = new NpgsqlCommand("""
        update generation_jobs
        set status = @status,
            started_at = null,
            updated_at = @updated_at
        where id = @id
        """, connection);
    command.Parameters.AddWithValue("id", jobId);
    command.Parameters.AddWithValue("status", JobStatuses.Queued);
    command.Parameters.AddWithValue("updated_at", DateTimeOffset.UtcNow);
    await command.ExecuteNonQueryAsync(cancellationToken);
}
```

- [ ] **Step 5: Add recovery service**

Create `ServicesHost/Tianming.GeneratorHost/Workers/StaleJobRecoveryService.cs`:

```csharp
using Tianming.GeneratorHost.Data;
using Tianming.GeneratorHost.Queue;

namespace Tianming.GeneratorHost.Workers;

public sealed class StaleJobRecoveryService : BackgroundService
{
    private readonly IGenerationJobStore _store;
    private readonly IChapterJobQueue _queue;
    private readonly TimeSpan _staleAfter;

    public StaleJobRecoveryService(
        IGenerationJobStore store,
        IChapterJobQueue queue,
        TimeSpan? staleAfter = null)
    {
        _store = store;
        _queue = queue;
        _staleAfter = staleAfter ?? TimeSpan.FromMinutes(30);
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await RecoverOnceAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }

    public async Task<int> RecoverOnceAsync(CancellationToken cancellationToken)
    {
        var staleBefore = DateTimeOffset.UtcNow.Subtract(_staleAfter);
        var stale = await _store.FindStaleRunningJobIdsAsync(staleBefore, cancellationToken);
        foreach (var jobId in stale)
        {
            await _store.ResetToQueuedAsync(jobId, cancellationToken);
            await _queue.EnqueueAsync(jobId, cancellationToken);
        }

        return stale.Count;
    }
}
```

- [ ] **Step 6: Register recovery service**

Modify `ServicesHost/Tianming.GeneratorHost/Program.cs`:

```csharp
builder.Services.AddHostedService<StaleJobRecoveryService>();
```

Place it next to:

```csharp
builder.Services.AddHostedService<ChapterGenerationWorker>();
```

- [ ] **Step 7: Run recovery tests**

Run:

```bash
dotnet test Tests/Tianming.GeneratorHost.Tests/Tianming.GeneratorHost.Tests.csproj --filter StaleJobRecoveryServiceTests
```

Expected: all recovery tests pass.

- [ ] **Step 8: Commit**

```bash
git add ServicesHost/Tianming.GeneratorHost/Data ServicesHost/Tianming.GeneratorHost/Workers Tests/Tianming.GeneratorHost.Tests/Workers ServicesHost/Tianming.GeneratorHost/Program.cs
git commit -m "feat: recover stale generator jobs"
```

Expected: one commit with stale job recovery.

---

### Task 15: Full Verification

**Files:**
- No new files expected.

- [ ] **Step 1: Run all tests**

Run:

```bash
dotnet test Tianming.GeneratorService.sln
```

Expected: all tests pass.

- [ ] **Step 2: Build Linux-safe service**

Run:

```bash
dotnet publish ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj -c Release -o /tmp/tianming-generator-host
```

Expected: publish succeeds. The output should not require `net8.0-windows`.

- [ ] **Step 3: Confirm WPF project still builds separately on a Windows-capable SDK environment**

Run on a Windows-capable environment:

```bash
dotnet build Core/App/天命.csproj
```

Expected: WPF project builds. If this Linux machine cannot build Windows-targeted WPF, record that limitation in the final implementation notes instead of treating it as a service failure.

- [ ] **Step 4: Run smoke test**

Run dependencies:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer/ServicesHost/Tianming.GeneratorHost
docker compose up -d
```

Run service:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
ASPNETCORE_URLS=http://localhost:5000 dotnet run --project ServicesHost/Tianming.GeneratorHost/Tianming.GeneratorHost.csproj
```

Run smoke script:

```bash
cd /home/wangcheng/novel-center-2/tianming-novel-ai-writer
ServicesHost/Tianming.GeneratorHost/scripts/smoke-generate.sh
```

Expected: script exits `0` with a succeeded job.

- [ ] **Step 5: Inspect git status**

Run:

```bash
git status --short
```

Expected: only intentional implementation files are modified. Existing unrelated dirty files from before this plan may still appear; do not revert them.

- [ ] **Step 6: Commit verification fixes if needed**

If verification required small fixes:

```bash
git add <fixed-files>
git commit -m "test: verify generator service"
```

Expected: verification commit contains only test or smoke-fix changes.

---

## Implementation Notes

- Keep `origin` unchanged. Push this branch to `dtdxg` when the implementation is ready:

```bash
git push dtdxg upgrade/novel-generation-2.8.7
```

- Do not commit the existing unrelated dirty changes unless they are required by the service implementation and reviewed as part of a task.
- Keep `Generator:Mode` set to `Fake` until real Tianming extraction is verified.
- The first working service is valuable even with the fake generator because it proves API shape, Postgres, Redis, worker execution, idempotency, and polling before the complex extraction work begins.

## Self-Review

Spec coverage:

- Linux-safe `net8.0` projects: Tasks 1, 9, 15.
- Async public chapter API: Tasks 8 and 9.
- Postgres durable jobs: Task 6.
- Redis queue/progress/locks: Task 7.
- Per-story workspace: Task 4.
- `novel-center` compatible result: Tasks 2 and 3.
- Fake first, real extraction behind interface: Tasks 5, 11, 12, 13.
- Recovery and retries foundation: Task 14.
- Smoke workflow: Task 10.

Placeholder scan: no unresolved requirement markers remain in this plan.

Type consistency: `ChapterGenerationRequest`, `ChapterGenerationResult`, `NovelCenterChanges`, `JobProgress`, `JobError`, `IGenerationJobStore`, `IChapterJobQueue`, and `IPublicChapterGenerator` are introduced before use in later tasks.

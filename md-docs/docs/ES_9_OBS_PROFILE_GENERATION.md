# Task 9: OBS Profile Generation — Implementation Plan

---

## Overview

Refactor the existing OBS profile generation to produce a **real OBS Studio importable profile** delivered as a downloadable **ZIP archive**. The ZIP contains the actual file structure OBS expects when using _Profile → Import_, including `service.json` (stream server/key), `streamEncoder.json` (encoding settings), `recordEncoder.json`, and `basic.ini` (video/audio/output mode).

The current implementation returns a flat custom DTO serialized as JSON — this does not match OBS's import format. This task replaces that with protocol-aware (RTMP + SRT), configurable, ZIP-based profile generation at the application layer.

---

## Current State (What Exists)

| Artifact                         | Location                                         | Status                                  |
| -------------------------------- | ------------------------------------------------ | --------------------------------------- |
| `GetObsProfileQuery`             | `Application/Queries/GetObsProfile/`             | ✅ Exists — no changes needed           |
| `GetObsProfileQueryHandler`      | `Application/Queries/GetObsProfile/`             | ⚠️ Exists — needs full refactor         |
| `ObsProfileDto` (+ sub-DTOs)     | `Application/DTOs/ObsProfileDto.cs`              | ⚠️ Exists — needs replacement           |
| `EventsController.GetObsProfile` | `Api/Controllers/EventsController.cs`            | ⚠️ Exists — needs refactor (return ZIP) |
| `StreamingSettings`              | `Application/StreamingSettings.cs`               | ⚠️ Exists — needs OBS defaults added    |
| Integration test                 | `IntegrationTests/EventsControllerQueryTests.cs` | ⚠️ Exists — needs update for ZIP        |
| Unit tests for handler           | —                                                | ❌ Missing                              |

---

## OBS Studio Profile Format Reference

An OBS profile is a **directory** that OBS imports via _Profile → Import_. It contains:

### `service.json` — Streaming Service Configuration

**RTMP mode:**

```json
{
  "settings": {
    "bwtest": false,
    "key": "evt_a1b2c3d4_sk_e5f6a7b8c9d0",
    "server": "rtmp://ingest.terrastream.de/live"
  },
  "type": "rtmp_custom"
}
```

**SRT mode:**

```json
{
  "settings": {
    "bwtest": false,
    "key": "",
    "server": "srt://ingest.terrastream.de:9000?streamid=evt_a1b2c3d4_sk_e5f6a7b8c9d0&latency=2000000&mode=caller"
  },
  "type": "rtmp_custom"
}
```

> Note: OBS uses `type: "rtmp_custom"` even for SRT. The SRT URL goes in the `server` field with the stream key embedded as a `streamid` query parameter.

### `streamEncoder.json` — Stream Encoder Settings

```json
{
  "bitrate": 5000,
  "keyint_sec": 2,
  "preset2": "veryfast",
  "profile": "main",
  "rate_control": "CBR"
}
```

### `recordEncoder.json` — Recording Encoder Settings

```json
{
  "bitrate": 5000,
  "keyint_sec": 2,
  "preset2": "veryfast",
  "profile": "main",
  "rate_control": "CBR"
}
```

### `basic.ini` — Video, Audio, and Output Mode Settings

```ini
[General]
Name=TerraStream - My Event Name

[Video]
BaseCX=1920
BaseCY=1080
OutputCX=1920
OutputCY=1080
FPSType=0
FPSCommon=30

[Audio]
SampleRate=48000
ChannelSetup=Stereo

[Output]
Mode=Advanced

[AdvOut]
TrackIndex=1
Encoder=obs_x264
Track1Bitrate=128
```

---

## Design Decisions

| Decision              | Choice                                              | Rationale                                                                                                        |
| --------------------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| **Delivery format**   | ZIP archive                                         | OBS _Profile → Import_ expects a directory; ZIP is the standard way to deliver a directory as a single download  |
| **Protocol support**  | RTMP + SRT                                          | Domain already models `StreamProtocol`; build protocol-aware `service.json` now so SRT works when enabled        |
| **State restriction** | No restriction                                      | Users may download profiles for reference/re-use in any event state                                              |
| **Encoding defaults** | Externalized to `StreamingSettings`                 | Adjustable per deployment via `appsettings.json` without code changes                                            |
| **Architecture**      | `IObsProfileGenerator` service in Application layer | Separates profile file generation logic from the MediatR handler for testability                                 |
| **ZIP assembly**      | Controller layer (or thin helper)                   | `System.IO.Compression.ZipArchive` is .NET BCL — the handler returns structured data, the controller packages it |

---

## Architecture

```
GetObsProfileQuery
    │
    ▼
GetObsProfileQueryHandler
    │   - loads Event from repository
    │   - delegates to IObsProfileGenerator
    │
    ▼
IObsProfileGenerator  (Application/Abstractions/)
    │   - builds ObsProfileResult: a collection of (fileName, content) entries
    │   - uses StreamingSettings (IOptions) for encoding defaults
    │   - handles RTMP vs SRT protocol branching for service.json
    │
    ▼
ObsProfileGenerator  (Application/Services/)
    │   - implements IObsProfileGenerator
    │   - generates: service.json, streamEncoder.json, recordEncoder.json, basic.ini
    │
    ▼
ObsProfileResult  (Application/DTOs/)
    │   - ProfileName: string
    │   - Files: IReadOnlyList<ObsProfileFileEntry>
    │       - FileName: string
    │       - Content: byte[]
    │
    ▼
EventsController.GetObsProfile
    │   - receives ObsProfileResult
    │   - packages entries into a ZipArchive in memory
    │   - returns File(zipBytes, "application/zip", "obs-profile-{eventId}.zip")
```

---

## Implementation Tasks

### Task 9.1: Extend `StreamingSettings` with OBS Defaults

**File:** `Application/StreamingSettings.cs`

Add an `ObsDefaults` nested class with configurable properties:

```csharp
public sealed class ObsDefaultSettings
{
    public string Encoder { get; init; } = "obs_x264";
    public string RateControl { get; init; } = "CBR";
    public string Preset { get; init; } = "veryfast";
    public string EncoderProfile { get; init; } = "main";
    public int KeyframeIntervalSeconds { get; init; } = 2;
    public int Fps { get; init; } = 30;
    public int AudioSampleRate { get; init; } = 48000;
    public string AudioChannelLayout { get; init; } = "Stereo";
    public int AudioBitrateKbps { get; init; } = 128;
    public string SrtIngestEndpoint { get; init; } = "srt://ingest.terrastream.de:9000";
    public int SrtLatencyMicroseconds { get; init; } = 2000000;
}
```

**Update `appsettings.json`:**

```json
"Streaming": {
    "DefaultIngestEndpoint": "rtmp://ingest.terrastream.de/live",
    "ObsDefaults": {
        "Encoder": "obs_x264",
        "RateControl": "CBR",
        "Preset": "veryfast",
        "EncoderProfile": "main",
        "KeyframeIntervalSeconds": 2,
        "Fps": 30,
        "AudioSampleRate": 48000,
        "AudioChannelLayout": "Stereo",
        "AudioBitrateKbps": 128,
        "SrtIngestEndpoint": "srt://ingest.terrastream.de:9000",
        "SrtLatencyMicroseconds": 2000000
    }
}
```

---

### Task 9.2: Create `ObsProfileResult` and `ObsProfileFileEntry` DTOs

**File:** `Application/DTOs/ObsProfileResult.cs` (new)

```csharp
public sealed record ObsProfileResult(
    string ProfileName,
    IReadOnlyList<ObsProfileFileEntry> Files);

public sealed record ObsProfileFileEntry(
    string FileName,
    byte[] Content);
```

This replaces the return type of the query. The old `ObsProfileDto` and its sub-DTOs will be removed.

---

### Task 9.3: Create `IObsProfileGenerator` Interface

**File:** `Application/Abstractions/IObsProfileGenerator.cs` (new)

```csharp
public interface IObsProfileGenerator
{
    ObsProfileResult Generate(Event @event);
}
```

Takes the full Event aggregate (has stream config, name, etc.) and returns the profile.

---

### Task 9.4: Implement `ObsProfileGenerator`

**File:** `Application/Services/ObsProfileGenerator.cs` (new)

This is the core logic. It:

1. **Generates `service.json`:**
   - RTMP: `{ "type": "rtmp_custom", "settings": { "server": ingestEndpoint, "key": streamKey, "bwtest": false } }`
   - SRT: `{ "type": "rtmp_custom", "settings": { "server": "srt://...?streamid={key}&latency=...", "key": "", "bwtest": false } }`

2. **Generates `streamEncoder.json`:**
   - Uses event's `MaxBitrateKbps` + defaults from `StreamingSettings.ObsDefaults` (encoder, rate_control, preset, profile, keyint_sec)

3. **Generates `recordEncoder.json`:**
   - Same encoder settings as stream (users can change independently in OBS later)

4. **Generates `basic.ini`:**
   - `[General]` section with profile name = `"TerraStream - {EventName}"`
   - `[Video]` section with `BaseCX/CY` and `OutputCX/CY` mapped from `MaxResolution`, plus `FPSCommon`
   - `[Audio]` section with `SampleRate` and `ChannelSetup`
   - `[Output]` section with `Mode=Advanced`
   - `[AdvOut]` section with `TrackIndex=1`, `Encoder`, `Track1Bitrate`

5. **Maps resolution:** `1080 → 1920x1080`, `720 → 1280x720`, `480 → 854x480`, etc. (existing `MapResolution` logic moves here)

**Dependencies:** `IOptions<StreamingSettings>` (injected)

---

### Task 9.5: Refactor `GetObsProfileQueryHandler`

**File:** `Application/Queries/GetObsProfile/GetObsProfileQueryHandler.cs`

- Change return type from `ObsProfileDto` to `ObsProfileResult`
- Inject `IObsProfileGenerator` instead of building the DTO inline
- Handler becomes a thin orchestrator: load event → call generator → return result
- Remove the `MapResolution` method (moved to generator)

**Also update:** `GetObsProfileQuery.cs` to return `IRequest<ObsProfileResult>`

---

### Task 9.6: Refactor `EventsController.GetObsProfile` Endpoint

**File:** `Api/Controllers/EventsController.cs`

- Change response from JSON file to ZIP file
- Use `System.IO.Compression.ZipArchive` to package the `ObsProfileResult.Files` into a ZIP in memory
- Return `File(zipBytes, "application/zip", $"obs-profile-{id}.zip")`
- Update Swagger attributes (`[Produces("application/zip")]`, response type descriptions)

---

### Task 9.7: Remove Old `ObsProfileDto` and Sub-DTOs

**File:** `Application/DTOs/ObsProfileDto.cs`

Delete the file entirely. It contained:

- `ObsProfileDto`
- `ObsVideoSettingsDto`
- `ObsAudioSettingsDto`
- `ObsOutputSettingsDto`

These are replaced by `ObsProfileResult` + `ObsProfileFileEntry`.

---

### Task 9.8: Register `IObsProfileGenerator` in DI

**File:** `Application/DependencyInjection.cs`

Add: `services.AddSingleton<IObsProfileGenerator, ObsProfileGenerator>();`

(Singleton is fine — the generator is stateless, it reads settings from `IOptions` which is also singleton.)

Wait — `IOptions<StreamingSettings>` is typically singleton anyway, but the generator itself should be registered as **singleton** because it has no per-request state. However, if using `IOptionsMonitor` for hot-reload support, it's still fine as singleton.

Alternatively, register as **scoped** or **transient** — no material difference given it's lightweight. Use **singleton** for efficiency.

---

### Task 9.9: Update `appsettings.json` and `appsettings.Development.json`

Add the `ObsDefaults` section under `Streaming` in both files (see Task 9.1 for the JSON structure).

---

### Task 9.10: Unit Tests — `ObsProfileGenerator`

**File:** `UnitTests/Application/ObsProfileGeneratorTests.cs` (new)

Test cases:

1. **RTMP profile produces correct `service.json`** — verify `type: "rtmp_custom"`, server URL, stream key
2. **SRT profile produces correct `service.json`** — verify SRT URL with `streamid` query param, empty key field
3. **`streamEncoder.json` uses event's MaxBitrateKbps** — verify bitrate from event, other settings from defaults
4. **`recordEncoder.json` matches stream encoder** — verify same settings
5. **`basic.ini` maps resolution correctly** — 1080→1920x1080, 720→1280x720, 480→854x480
6. **`basic.ini` uses correct audio settings** — from `StreamingSettings.ObsDefaults`
7. **Profile name includes event name** — `"TerraStream - {EventName}"`
8. **All 4 files are present** — `service.json`, `streamEncoder.json`, `recordEncoder.json`, `basic.ini`
9. **Custom settings override defaults** — supply non-default encoder/preset, verify they appear in output

---

### Task 9.11: Unit Tests — `GetObsProfileQueryHandler`

**File:** `UnitTests/Application/GetObsProfileQueryHandlerTests.cs` (new)

Test cases:

1. **Valid event returns profile result** — mock repo and generator, verify handler calls generator
2. **Non-existent event throws `EventNotFoundException`** — mock repo returns null
3. **Handler passes correct event to generator** — verify event loaded by ID and tenant

---

### Task 9.12: Update Integration Tests

**File:** `IntegrationTests/EventsControllerQueryTests.cs`

Update `GetObsProfile_ShouldReturnFileDownload`:

1. Assert `Content-Type: application/zip`
2. Assert filename is `obs-profile-{id}.zip`
3. Extract ZIP in memory, verify it contains exactly 4 files: `service.json`, `streamEncoder.json`, `recordEncoder.json`, `basic.ini`
4. Deserialize `service.json`, verify it contains the event's stream key and the configured ingest endpoint
5. Deserialize `streamEncoder.json`, verify bitrate matches event's `MaxBitrateKbps`
6. Parse `basic.ini`, verify resolution matches `MaxResolution` (1920x1080 for default 1080)

Add new test:

- **`GetObsProfile_WithSrtProtocol_ShouldReturnSrtServiceJson`** — create event with SRT protocol, download ZIP, verify `service.json` has SRT URL format

---

## Execution Order

```
9.1  StreamingSettings extension          (no dependencies)
9.2  ObsProfileResult DTOs               (no dependencies)
9.3  IObsProfileGenerator interface       (depends on 9.2)
9.4  ObsProfileGenerator implementation   (depends on 9.1, 9.2, 9.3)
9.5  Refactor handler + query             (depends on 9.2, 9.3)
9.6  Refactor controller endpoint         (depends on 9.2, 9.5)
9.7  Remove old ObsProfileDto             (depends on 9.5, 9.6)
9.8  DI registration                      (depends on 9.3, 9.4)
9.9  appsettings.json updates             (depends on 9.1)
9.10 Unit tests: ObsProfileGenerator      (depends on 9.4)
9.11 Unit tests: Handler                  (depends on 9.5)
9.12 Integration tests update             (depends on 9.6)
```

**Parallelizable:** 9.1 + 9.2 can run in parallel. 9.10 + 9.11 can run in parallel. 9.9 can run alongside 9.4.

---

## Files Changed Summary

| Action     | File                                                             |
| ---------- | ---------------------------------------------------------------- |
| **Modify** | `Application/StreamingSettings.cs`                               |
| **Create** | `Application/DTOs/ObsProfileResult.cs`                           |
| **Create** | `Application/Abstractions/IObsProfileGenerator.cs`               |
| **Create** | `Application/Services/ObsProfileGenerator.cs`                    |
| **Modify** | `Application/Queries/GetObsProfile/GetObsProfileQuery.cs`        |
| **Modify** | `Application/Queries/GetObsProfile/GetObsProfileQueryHandler.cs` |
| **Modify** | `Api/Controllers/EventsController.cs`                            |
| **Delete** | `Application/DTOs/ObsProfileDto.cs`                              |
| **Modify** | `Application/DependencyInjection.cs`                             |
| **Modify** | `Api/appsettings.json`                                           |
| **Modify** | `Api/appsettings.Development.json`                               |
| **Create** | `UnitTests/Application/ObsProfileGeneratorTests.cs`              |
| **Create** | `UnitTests/Application/GetObsProfileQueryHandlerTests.cs`        |
| **Modify** | `IntegrationTests/EventsControllerQueryTests.cs`                 |

---

## Risk / Notes

- **OBS version compatibility:** The profile format is based on OBS 28+. Older OBS versions may use `preset` instead of `preset2` in `streamEncoder.json`. We target OBS 28+ since SRT support was added in OBS 28.
- **SRT ingest endpoint:** The SRT URL includes `latency` and `mode` parameters. These are configurable via `StreamingSettings.ObsDefaults` so operators can tune them.
- **ZIP size:** Minimal — the 4 text files total ~1-2 KB uncompressed. No performance concern.
- **basic.ini generation:** .NET doesn't have a built-in INI writer. Use simple string building (`StringBuilder`) since the format is trivial and predictable.
- **No `obs://` deep link:** The plan mentioned this as optional. Not implemented in this task — can be added later if OBS supports it on the user's platform.

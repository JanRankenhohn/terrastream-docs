# Application Layer Implementation Plan

---

## Overview

This plan covers **Tasks 3 and 4** from the EventService Implementation Plan — the full Application Layer (commands, queries, handlers, validators, DTOs, mapping) and the Contracts project (integration events, gRPC proto).

**Scope:**

- `TerraStream.EventService.Application` — commands, queries, handlers, validators, DTOs, abstractions, behaviors
- `TerraStream.EventService.Contracts` — integration event DTOs, gRPC proto file
- Minor domain adjustment — add cross-tenant stream key lookup to `IEventRepository`

**Architectural decisions (confirmed):**
| Decision | Choice |
|---|---|
| CreateEventCommand style | All-at-once (full sub-entity data required; matches existing `Event.Create()` signature) |
| Stream key validation | Add cross-tenant `GetByStreamKeyAsync(StreamKey)` to `IEventRepository` (no tenantId, since SRS has no tenant context) |
| Event publishing pattern | **IUnitOfWork + transactional outbox** — `IUnitOfWork.SaveChangesAsync()` persists aggregate changes AND writes integration events to an `OutboxMessages` table in the **same database transaction**. A separate background process (`OutboxDeliveryService`) polls the table and publishes to RabbitMQ. This eliminates the dual-write problem. Handlers do NOT manually publish. Powered by MassTransit's EF Core transactional outbox. |
| gRPC/Contracts scope | Included — proto file and integration event DTOs created in this plan |

---

## Project Structure

### Application Project

```
TerraStream.EventService.Application/
├── Abstractions/
│   ├── ITenantContext.cs
│   ├── IUnitOfWork.cs
│   └── IClock.cs
├── Behaviors/
│   └── ValidationBehavior.cs
├── Commands/
│   ├── CreateEvent/
│   │   ├── CreateEventCommand.cs
│   │   ├── CreateEventCommandHandler.cs
│   │   └── CreateEventCommandValidator.cs
│   ├── UpdateEventDetails/
│   │   ├── UpdateEventDetailsCommand.cs
│   │   ├── UpdateEventDetailsCommandHandler.cs
│   │   └── UpdateEventDetailsCommandValidator.cs
│   ├── SetEventTheme/
│   │   ├── SetEventThemeCommand.cs
│   │   ├── SetEventThemeCommandHandler.cs
│   │   └── SetEventThemeCommandValidator.cs
│   ├── SetEventSchedule/
│   │   ├── SetEventScheduleCommand.cs
│   │   ├── SetEventScheduleCommandHandler.cs
│   │   └── SetEventScheduleCommandValidator.cs
│   ├── ConfigureEventFeatures/
│   │   ├── ConfigureEventFeaturesCommand.cs
│   │   ├── ConfigureEventFeaturesCommandHandler.cs
│   │   └── ConfigureEventFeaturesCommandValidator.cs
│   ├── SetEventAccess/
│   │   ├── SetEventAccessCommand.cs
│   │   ├── SetEventAccessCommandHandler.cs
│   │   └── SetEventAccessCommandValidator.cs
│   ├── InviteUsers/
│   │   ├── InviteUsersCommand.cs
│   │   ├── InviteUsersCommandHandler.cs
│   │   └── InviteUsersCommandValidator.cs
│   ├── PublishEvent/
│   │   ├── PublishEventCommand.cs
│   │   └── PublishEventCommandHandler.cs
│   ├── StartTestStream/
│   │   ├── StartTestStreamCommand.cs
│   │   └── StartTestStreamCommandHandler.cs
│   ├── StopTestStream/
│   │   ├── StopTestStreamCommand.cs
│   │   └── StopTestStreamCommandHandler.cs
│   ├── GoLive/
│   │   ├── GoLiveCommand.cs
│   │   └── GoLiveCommandHandler.cs
│   ├── EndStream/
│   │   ├── EndStreamCommand.cs
│   │   └── EndStreamCommandHandler.cs
│   ├── ArchiveEvent/
│   │   ├── ArchiveEventCommand.cs
│   │   └── ArchiveEventCommandHandler.cs
│   ├── CancelEvent/
│   │   ├── CancelEventCommand.cs
│   │   └── CancelEventCommandHandler.cs
│   └── ValidateStreamKey/
│       ├── ValidateStreamKeyCommand.cs
│       └── ValidateStreamKeyCommandHandler.cs
├── Queries/
│   ├── GetEventById/
│   │   ├── GetEventByIdQuery.cs
│   │   └── GetEventByIdQueryHandler.cs
│   ├── ListEvents/
│   │   ├── ListEventsQuery.cs
│   │   └── ListEventsQueryHandler.cs
│   ├── GetEventStreamConfig/
│   │   ├── GetEventStreamConfigQuery.cs
│   │   └── GetEventStreamConfigQueryHandler.cs
│   ├── GetObsProfile/
│   │   ├── GetObsProfileQuery.cs
│   │   └── GetObsProfileQueryHandler.cs
│   └── GetEventsByStatus/
│       ├── GetEventsByStatusQuery.cs
│       └── GetEventsByStatusQueryHandler.cs
├── DTOs/
│   ├── EventDto.cs
│   ├── EventSummaryDto.cs
│   ├── EventThemeDto.cs
│   ├── EventScheduleDto.cs
│   ├── EventStreamConfigDto.cs
│   ├── EventAccessConfigDto.cs
│   ├── EventFeatureDto.cs
│   ├── EventInvitationDto.cs
│   ├── PagedResultDto.cs
│   ├── ObsProfileDto.cs
│   └── StreamKeyValidationResultDto.cs
├── Mapping/
│   └── EventMappingExtensions.cs
└── Exceptions/
    ├── EventNotFoundException.cs
    └── ValidationException.cs
```

### Contracts Project

```
TerraStream.EventService.Contracts/
├── IntegrationEvents/
│   ├── EventScheduledIntegrationEvent.cs
│   ├── EventWentLiveIntegrationEvent.cs
│   ├── EventEndedIntegrationEvent.cs
│   └── EventCancelledIntegrationEvent.cs
└── Protos/
    └── event_service.proto
```

---

## NuGet Packages

### Application

| Package                                          | Version | Purpose                                                                       |
| ------------------------------------------------ | ------- | ----------------------------------------------------------------------------- |
| `Mediator.Abstractions`                          | 2.x     | CQRS interfaces (IRequest, IRequestHandler, IPipelineBehavior) — MIT licensed |
| `Mediator.SourceGenerator`                       | 2.x     | Source-generated mediator dispatch (zero reflection) — MIT licensed           |
| `FluentValidation`                               | 11.x    | Command input validation                                                      |
| `FluentValidation.DependencyInjectionExtensions` | 11.x    | Auto-registration of validators                                               |

> **Note:** We use [Mediator](https://github.com/martinothamar/Mediator) by martinothamar instead of MediatR.
> MediatR 12+ uses a restrictive license. Mediator is MIT-licensed, source-generated (zero reflection, faster startup),
> and exposes nearly identical abstractions (`IRequest<T>`, `IRequestHandler<T,R>`, `IPipelineBehavior<T,R>`).
> Registration: `services.AddMediator(opts => opts.ServiceLifetime = ServiceLifetime.Scoped);` —
> the source generator discovers all handlers and validators at compile time (no assembly scanning).

### Contracts

| Package           | Version | Purpose                                      |
| ----------------- | ------- | -------------------------------------------- |
| `Google.Protobuf` | 3.x     | Protobuf runtime                             |
| `Grpc.Tools`      | 2.x     | Proto code generation (build-time)           |
| `Grpc.Net.Client` | 2.x     | gRPC client support (for consuming services) |

---

## Prerequisite: Domain Adjustment

Before the Application layer work begins, one change to the Domain layer is needed:

### Add cross-tenant stream key lookup to IEventRepository

The SRS `on_publish` callback sends only a stream key — no tenant context. The current `GetByStreamKeyAsync(StreamKey, Guid tenantId)` requires a tenantId. Add a second overload:

```csharp
// In IEventRepository.cs — NEW method
Task<Event?> GetByStreamKeyAsync(StreamKey streamKey, CancellationToken cancellationToken = default);
```

This is safe because stream keys are globally unique by design (`evt_{eventIdHex}_sk_{randomHex}`).

---

## Detailed Design

### 1. Abstractions

#### 1.1 ITenantContext

```csharp
namespace TerraStream.EventService.Application.Abstractions;

/// <summary>
/// Provides the current tenant context for multi-tenant operations.
/// Resolved from HTTP X-Tenant-Id header or gRPC call metadata.
/// Registered as a scoped service.
/// </summary>
public interface ITenantContext
{
    Guid TenantId { get; }
}
```

#### 1.2 IUnitOfWork

```csharp
namespace TerraStream.EventService.Application.Abstractions;

/// <summary>
/// Unit of Work abstraction for coordinating persistence and domain event dispatch
/// using the transactional outbox pattern.
///
/// When SaveChangesAsync is called, the implementation must — within a SINGLE
/// database transaction:
/// 1. Intercept tracked aggregates and collect their domain events
/// 2. Convert domain events to integration event records
/// 3. Write integration events to the OutboxMessages table
/// 4. Persist all tracked entity changes to the database
/// 5. Clear domain events from the aggregates
/// 6. Commit the transaction
///
/// A separate background process (MassTransit OutboxDeliveryService) polls the
/// OutboxMessages table and publishes messages to RabbitMQ, then marks them
/// as delivered. This guarantees at-least-once delivery without dual writes.
/// </summary>
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

**Infrastructure implementation contract** (for later):

The implementation uses **MassTransit's EF Core transactional outbox** (`UseEntityFrameworkCoreOutbox()`), which adds three tables to the `EventDbContext`:

| Table           | Purpose                                               |
| --------------- | ----------------------------------------------------- |
| `OutboxMessage` | Stores serialized integration events pending delivery |
| `OutboxState`   | Tracks delivery state per outbox message              |
| `InboxState`    | Idempotency for consumers (de-duplication)            |

**SaveChangesAsync flow:**

1. Override `EventDbContext.SaveChangesAsync()` (or wrap in a UoW class)
2. Before calling `base.SaveChangesAsync()`, iterate `ChangeTracker.Entries<Event>()` and collect `DomainEvents`
3. Map each `IDomainEvent` to the corresponding integration event record
4. Call `AddRange()` on the MassTransit outbox (writes to `OutboxMessage` in the same `DbContext`)
5. Call `ClearDomainEvents()` on each aggregate
6. Call `base.SaveChangesAsync()` — entity changes + outbox rows committed atomically

**Background delivery:**

- `MassTransit.EntityFrameworkCoreIntegration` registers a hosted service (`BusOutboxDeliveryService`) that polls `OutboxMessage` and publishes to RabbitMQ
- Delivery is retried automatically on transient failures
- Messages are published in order per outbox instance
- Successfully delivered messages are cleaned up after a configurable retention period

**Guarantees:**

- **No lost events:** If the DB transaction commits, outbox rows exist and will eventually be delivered
- **No phantom events:** If the DB transaction rolls back, no outbox rows are written
- **At-least-once delivery:** Consumers must be idempotent (use `InboxState` table or application-level de-duplication)
- **Works when RabbitMQ is down:** Messages accumulate in the outbox table and are delivered when the broker recovers

#### 1.3 IClock

```csharp
namespace TerraStream.EventService.Application.Abstractions;

/// <summary>
/// Abstraction over system clock for testability.
/// Allows unit tests to control time-dependent behavior.
/// </summary>
public interface IClock
{
    DateTimeOffset UtcNow { get; }
}
```

Production implementation returns `DateTimeOffset.UtcNow`. Tests can inject a fixed or controllable clock.

---

### 2. Exceptions

#### 2.1 EventNotFoundException

```csharp
namespace TerraStream.EventService.Application.Exceptions;

public sealed class EventNotFoundException : Exception
{
    public Guid EventId { get; }

    public EventNotFoundException(Guid eventId)
        : base($"Event with ID '{eventId}' was not found.")
    {
        EventId = eventId;
    }
}
```

The API layer maps this to **404 Not Found**.

#### 2.2 ValidationException

Custom `ValidationException` wrapping FluentValidation failures:

```csharp
namespace TerraStream.EventService.Application.Exceptions;

public sealed class ValidationException : Exception
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IDictionary<string, string[]> errors)
        : base("One or more validation errors occurred.")
    {
        Errors = errors;
    }
}
```

The API layer maps this to **400 Bad Request** with the errors dictionary as the response body.

---

### 3. DTOs

All DTOs are **immutable record types**. They represent the read model returned to API consumers.

#### 3.1 EventDto (full detail)

```csharp
public sealed record EventDto(
    Guid Id,
    Guid TenantId,
    string Name,
    string Description,
    string Status,
    DateTimeOffset CreatedAt,
    DateTimeOffset UpdatedAt,
    int Version,
    EventThemeDto Theme,
    EventScheduleDto Schedule,
    EventStreamConfigDto StreamConfig,
    EventAccessConfigDto AccessConfig,
    IReadOnlyList<EventFeatureDto> Features,
    IReadOnlyList<EventInvitationDto> Invitations);
```

#### 3.2 EventSummaryDto (for list views)

```csharp
public sealed record EventSummaryDto(
    Guid Id,
    string Name,
    string Status,
    DateTimeOffset? ScheduledStart,
    DateTimeOffset? ScheduledEnd,
    int InvitationCount,
    DateTimeOffset CreatedAt,
    DateTimeOffset UpdatedAt);
```

#### 3.3 Sub-entity DTOs

```csharp
public sealed record EventThemeDto(
    string PrimaryColor,
    string? SecondaryColor,
    string? AccentColor,
    Guid? LogoAssetId,
    Guid? WatermarkAssetId,
    Guid? PosterAssetId,
    bool HidePlatformBranding,
    string? CustomCss);

public sealed record EventScheduleDto(
    DateTimeOffset ScheduledStart,
    DateTimeOffset ScheduledEnd,
    string Timezone,
    int LobbyMinutesBefore,
    Guid? LobbyContentAssetId,
    Guid? PostEventContentAssetId);

public sealed record EventStreamConfigDto(
    string StreamKey,
    string IngestEndpoint,
    string Protocol,
    int MaxResolution,
    int MaxBitrateKbps,
    bool RecordingEnabled);

public sealed record EventAccessConfigDto(
    string AccessMode,
    int? MaxConcurrentViewers,
    bool RequireAuthentication);

public sealed record EventFeatureDto(
    Guid Id,
    string FeatureType,
    bool Enabled);

public sealed record EventInvitationDto(
    Guid Id,
    string Email,
    string Status,
    DateTimeOffset? SentAt);
```

#### 3.4 PagedResultDto

```csharp
public sealed record PagedResultDto<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int PageNumber,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => PageNumber < TotalPages;
    public bool HasPreviousPage => PageNumber > 1;
}
```

#### 3.5 ObsProfileDto

```csharp
public sealed record ObsProfileDto(
    string ProfileName,
    string ServerUrl,
    string StreamKey,
    ObsVideoSettingsDto Video,
    ObsAudioSettingsDto Audio,
    ObsOutputSettingsDto Output);

public sealed record ObsVideoSettingsDto(
    string BaseResolution,
    string OutputResolution,
    int Fps);

public sealed record ObsAudioSettingsDto(
    int SampleRate,
    string ChannelLayout,
    int BitrateKbps);

public sealed record ObsOutputSettingsDto(
    string Encoder,
    string RateControl,
    int BitrateKbps,
    string Preset,
    string Profile,
    int KeyframeIntervalSeconds);
```

#### 3.6 StreamKeyValidationResultDto

```csharp
public sealed record StreamKeyValidationResultDto(
    bool IsValid,
    Guid? EventId,
    Guid? TenantId,
    string? RejectReason);
```

---

### 4. Mapping

#### EventMappingExtensions

Static extension methods to map domain entities to DTOs. Located in `Mapping/EventMappingExtensions.cs`.

```csharp
namespace TerraStream.EventService.Application.Mapping;

public static class EventMappingExtensions
{
    public static EventDto ToDto(this Event @event) { ... }
    public static EventSummaryDto ToSummaryDto(this Event @event) { ... }
    public static EventThemeDto ToDto(this EventTheme theme) { ... }
    public static EventScheduleDto ToDto(this EventSchedule schedule) { ... }
    public static EventStreamConfigDto ToDto(this EventStreamConfig config) { ... }
    public static EventAccessConfigDto ToDto(this EventAccessConfig config) { ... }
    public static EventFeatureDto ToDto(this EventFeature feature) { ... }
    public static EventInvitationDto ToDto(this EventInvitation invitation) { ... }
}
```

**Enum-to-string mapping:** All enum values are returned as their `string` name (e.g., `"Draft"`, `"RTMP"`, `"Public"`) using `ToString()`. This makes the API self-documenting and decoupled from enum integer values.

---

### 5. Commands

Each command is a `record` implementing `Mediator.IRequest<TResult>`. Handlers implement `IRequestHandler<TCommand, TResult>`. (From the Mediator library by martinothamar — MIT licensed, source-generated.)

#### 5.1 CreateEventCommand

```csharp
public sealed record CreateEventCommand(
    // Core
    string Name,
    string Description,
    // Theme
    string PrimaryColor,
    string? SecondaryColor,
    string? AccentColor,
    Guid? LogoAssetId,
    Guid? WatermarkAssetId,
    Guid? PosterAssetId,
    bool HidePlatformBranding,
    string? CustomCss,
    // Schedule
    DateTimeOffset ScheduledStart,
    DateTimeOffset ScheduledEnd,
    string Timezone,
    int LobbyMinutesBefore,
    // StreamConfig
    string Protocol,            // "RTMP" or "SRT"
    int MaxResolution,          // 1080, 720, 480
    int MaxBitrateKbps,         // 5000, 2500, etc.
    bool RecordingEnabled,
    // AccessConfig
    string AccessMode,          // "Public", "InviteOnly", "IdpRestricted"
    int? MaxConcurrentViewers,
    bool RequireAuthentication
) : IRequest<EventDto>;
```

**Handler logic:**

1. Resolve `tenantId` from `ITenantContext`
2. Parse enum strings (`Protocol`, `AccessMode`) — throw validation error if invalid
3. Construct owned entities: `EventTheme`, `EventSchedule`, `EventStreamConfig` (with placeholder ingest endpoint from config), `EventAccessConfig`
4. Call `Event.Create(tenantId, name, description, theme, schedule, streamConfig, accessConfig)`
5. Call `event.GenerateStreamKey()`
6. Call `repository.AddAsync(event)`
7. Call `unitOfWork.SaveChangesAsync()` (persists; no domain events raised at Draft creation)
8. Return `event.ToDto()`

**Validator rules:**

- Name: required, max 200 characters
- Description: required, max 2000 characters
- PrimaryColor: required, valid hex color format (`^#[0-9A-Fa-f]{6}$`)
- ScheduledStart: must be in the future
- ScheduledEnd: must be after ScheduledStart
- Timezone: must be a valid IANA/system timezone
- LobbyMinutesBefore: 0–120
- Protocol: must be "RTMP" or "SRT"
- MaxResolution: must be one of 480, 720, 1080, 1440, 2160
- MaxBitrateKbps: 500–20000
- AccessMode: must be "Public", "InviteOnly", or "IdpRestricted"
- MaxConcurrentViewers: if provided, >= 1

**Configuration dependency:** The ingest endpoint URL (e.g., `rtmp://ingest.terrastream.de/live`) comes from application configuration, not from the command. The handler reads this from an injected `IOptions<StreamingSettings>` or similar.

```csharp
// Settings class for injection
public sealed class StreamingSettings
{
    public string DefaultIngestEndpoint { get; init; } = "rtmp://ingest.terrastream.de/live";
}
```

#### 5.2 UpdateEventDetailsCommand

```csharp
public sealed record UpdateEventDetailsCommand(
    Guid EventId,
    string Name,
    string Description
) : IRequest<EventDto>;
```

**Handler:** Load event → `event.UpdateDetails(name, description)` → `unitOfWork.SaveChangesAsync()` → return DTO.

**Validator:** Same name/description rules as CreateEvent.

#### 5.3 SetEventThemeCommand

```csharp
public sealed record SetEventThemeCommand(
    Guid EventId,
    string PrimaryColor,
    string? SecondaryColor,
    string? AccentColor,
    Guid? LogoAssetId,
    Guid? WatermarkAssetId,
    Guid? PosterAssetId,
    bool HidePlatformBranding,
    string? CustomCss
) : IRequest<EventDto>;
```

**Handler:** Load event → construct new `EventTheme` → `event.SetTheme(theme)` → save → return DTO.

**Validator:** PrimaryColor required, valid hex color.

#### 5.4 SetEventScheduleCommand

```csharp
public sealed record SetEventScheduleCommand(
    Guid EventId,
    DateTimeOffset ScheduledStart,
    DateTimeOffset ScheduledEnd,
    string Timezone,
    int LobbyMinutesBefore
) : IRequest<EventDto>;
```

**Handler:** Load event → construct new `EventSchedule` → `event.SetSchedule(schedule)` → save → return DTO.

**Validator:** Same schedule rules as CreateEvent.

#### 5.5 ConfigureEventFeaturesCommand

```csharp
public sealed record ConfigureEventFeaturesCommand(
    Guid EventId,
    IReadOnlyList<FeatureConfigItem> Features
) : IRequest<EventDto>;

public sealed record FeatureConfigItem(
    string FeatureType,
    bool Enabled);
```

**Handler:** Load event → for each feature in the list, call `event.ConfigureFeature(type, enabled)` → save → return DTO.

**Validator:** Features list must not be empty. Each FeatureType must be non-empty. Valid feature types: `"QnA"`, `"Chat"`, `"Reactions"`, `"Analytics"`.

#### 5.6 SetEventAccessCommand

```csharp
public sealed record SetEventAccessCommand(
    Guid EventId,
    string AccessMode,
    int? MaxConcurrentViewers,
    bool RequireAuthentication
) : IRequest<EventDto>;
```

**Handler:** Load event → parse AccessMode enum → construct `EventAccessConfig` → `event.SetAccessConfig(config)` → save → return DTO.

**Validator:** AccessMode must be valid. MaxConcurrentViewers >= 1 if provided.

#### 5.7 InviteUsersCommand

```csharp
public sealed record InviteUsersCommand(
    Guid EventId,
    IReadOnlyList<string> Emails
) : IRequest<EventDto>;
```

**Handler:** Load event → for each email, call `event.AddInvitation(email)` → save → return DTO.

**Validator:** Emails list must not be empty. Each email must be a valid email format.

#### 5.8 State Transition Commands

All follow the same pattern — thin commands with just `EventId`:

```csharp
public sealed record PublishEventCommand(Guid EventId) : IRequest<EventDto>;
public sealed record StartTestStreamCommand(Guid EventId) : IRequest<EventDto>;
public sealed record StopTestStreamCommand(Guid EventId) : IRequest<EventDto>;
public sealed record GoLiveCommand(Guid EventId) : IRequest<EventDto>;
public sealed record EndStreamCommand(Guid EventId) : IRequest<EventDto>;
public sealed record ArchiveEventCommand(Guid EventId) : IRequest<EventDto>;
public sealed record CancelEventCommand(Guid EventId) : IRequest<EventDto>;
```

**Handler pattern (same for all 7):**

1. Resolve `tenantId` from `ITenantContext`
2. Load event via repository (throw `EventNotFoundException` if null)
3. Call `event.TransitionTo(targetStatus)` — aggregate raises domain event internally
4. Call `unitOfWork.SaveChangesAsync()` — domain events are converted to integration events and written to the outbox table atomically with the entity changes; background delivery publishes to RabbitMQ
5. Return `event.ToDto()`

The domain's `TransitionTo()` already validates the state machine and raises domain events. The handler is deliberately thin. Handlers never interact with messaging infrastructure directly.

**No separate validators** for these commands — the only input is `EventId` (a Guid), and the domain enforces transition rules.

#### 5.9 ValidateStreamKeyCommand

```csharp
public sealed record ValidateStreamKeyCommand(
    string StreamKey
) : IRequest<StreamKeyValidationResultDto>;
```

**Handler logic:**

1. Parse stream key string to `StreamKey` value object (reject if invalid format)
2. Call repository's **cross-tenant** `GetByStreamKeyAsync(streamKey)` (no tenantId)
3. If event not found → return `IsValid = false, RejectReason = "Unknown stream key"`
4. If event found, check `Status` is one of `Scheduled`, `Testing`, `Live` → return `IsValid = true`
5. Otherwise → return `IsValid = false, RejectReason = "Event is not in a streamable state"`

**Note:** This handler does NOT use `ITenantContext`. It is an unauthenticated internal endpoint called by SRS.

---

### 6. Queries

#### 6.1 GetEventByIdQuery

```csharp
public sealed record GetEventByIdQuery(Guid EventId) : IRequest<EventDto>;
```

**Handler:** Resolve tenantId → load event → throw `EventNotFoundException` if null → return `event.ToDto()`.

#### 6.2 ListEventsQuery

```csharp
public sealed record ListEventsQuery(
    string? Status,
    string? SearchText,
    DateTimeOffset? ScheduledStartFrom,
    DateTimeOffset? ScheduledStartTo,
    int PageNumber = 1,
    int PageSize = 20,
    string? SortBy = "createdAt",
    string? SortDirection = "desc"
) : IRequest<PagedResultDto<EventSummaryDto>>;
```

**Handler:**

1. Resolve tenantId → build `EventQueryFilter(tenantId)` with all optional filters
2. Call `repository.ListAsync(filter)`
3. Map result to `PagedResultDto<EventSummaryDto>`

#### 6.3 GetEventStreamConfigQuery

```csharp
public sealed record GetEventStreamConfigQuery(Guid EventId) : IRequest<EventStreamConfigDto>;
```

**Handler:** Load event → return `event.StreamConfig.ToDto()`.

#### 6.4 GetObsProfileQuery

```csharp
public sealed record GetObsProfileQuery(Guid EventId) : IRequest<ObsProfileDto>;
```

**Handler:**

1. Load event
2. Build OBS profile from stream config (stream key, ingest endpoint, max resolution, max bitrate)
3. Map resolution → OBS resolution string (1080 → `"1920x1080"`, 720 → `"1280x720"`, etc.)
4. Map bitrate → OBS output bitrate
5. Set sensible defaults for encoder (`obs_x264`), rate control (`CBR`), preset (`veryfast`), profile (`main`), keyframe interval (2s), audio (48kHz, stereo, 128kbps)
6. Return `ObsProfileDto`

See **Section 9** for the OBS profile JSON structure.

#### 6.5 GetEventsByStatusQuery

```csharp
public sealed record GetEventsByStatusQuery(
    string Status,
    int PageNumber = 1,
    int PageSize = 50
) : IRequest<PagedResultDto<EventSummaryDto>>;
```

**Handler:** This is an internal query (used by ResourcePlanner via gRPC). It resolves tenantId from context, builds a filter with the status enum, and returns paged summaries.

---

### 7. Validation Behavior (Mediator Pipeline)

A generic pipeline behavior that runs FluentValidation validators before the handler.
Mediator supports `IPipelineBehavior<TRequest, TResponse>` identically to MediatR:

```csharp
namespace TerraStream.EventService.Application.Behaviors;

public sealed class ValidationBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators)
    {
        _validators = validators;
    }

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!_validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);

        var validationResults = await Task.WhenAll(
            _validators.Select(v => v.ValidateAsync(context, cancellationToken)));

        var failures = validationResults
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
        {
            var errors = failures
                .GroupBy(f => f.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(f => f.ErrorMessage).ToArray());

            throw new Exceptions.ValidationException(errors);
        }

        return await next();
    }
}
```

**Registration:** Added as an open generic in DI:

```csharp
// Mediator source generator auto-discovers IPipelineBehavior implementations.
// No manual open-generic registration needed — just define the class.
// If explicit registration is preferred:
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

---

### 8. Integration Events (Contracts Project)

Located in `TerraStream.EventService.Contracts/IntegrationEvents/`.

These are the integration event records written to the transactional outbox table during `SaveChangesAsync()`. They are delivered to RabbitMQ asynchronously by MassTransit's `BusOutboxDeliveryService`.

```csharp
public sealed record EventScheduledIntegrationEvent(
    Guid EventId,
    Guid TenantId,
    string EventName,
    DateTimeOffset ScheduledStart,
    int ExpectedViewers,
    DateTimeOffset OccurredAt);

public sealed record EventWentLiveIntegrationEvent(
    Guid EventId,
    Guid TenantId,
    string EventName,
    string StreamKey,
    string IngestEndpoint,
    DateTimeOffset OccurredAt);

public sealed record EventEndedIntegrationEvent(
    Guid EventId,
    Guid TenantId,
    string EventName,
    TimeSpan Duration,
    DateTimeOffset OccurredAt);

public sealed record EventCancelledIntegrationEvent(
    Guid EventId,
    Guid TenantId,
    string EventName,
    string Reason,
    DateTimeOffset OccurredAt);
```

**Mapping from domain events** (implemented in Infrastructure):

| Domain Event                | Integration Event                | Extra Fields                                                    |
| --------------------------- | -------------------------------- | --------------------------------------------------------------- |
| `EventScheduledDomainEvent` | `EventScheduledIntegrationEvent` | `ExpectedViewers` from `AccessConfig.MaxConcurrentViewers ?? 0` |
| `EventWentLiveDomainEvent`  | `EventWentLiveIntegrationEvent`  | `IngestEndpoint` from `StreamConfig.IngestEndpoint`             |
| `EventEndedDomainEvent`     | `EventEndedIntegrationEvent`     | Direct mapping                                                  |
| `EventCancelledDomainEvent` | `EventCancelledIntegrationEvent` | Direct mapping                                                  |

---

### 9. OBS Profile Generation

The `GetObsProfileQueryHandler` generates a JSON-serializable OBS profile. The structure follows OBS Studio's importable format:

```json
{
  "settings": {
    "service": {
      "type": "rtmp_custom",
      "server": "rtmp://ingest.terrastream.de/live",
      "key": "evt_abcdef12_sk_1234567890ab"
    },
    "output": {
      "mode": "Simple",
      "streaming": {
        "encoder": "obs_x264",
        "rate_control": "CBR",
        "bitrate": 5000,
        "preset": "veryfast",
        "profile": "main",
        "tune": "zerolatency",
        "keyframe_interval": 2
      },
      "recording": {
        "enabled": true,
        "format": "mp4"
      }
    },
    "video": {
      "base_resolution": "1920x1080",
      "output_resolution": "1920x1080",
      "fps": 30
    },
    "audio": {
      "sample_rate": 48000,
      "channels": "stereo",
      "bitrate": 128
    }
  }
}
```

Resolution mapping table:

| MaxResolution | base_resolution | output_resolution |
| ------------- | --------------- | ----------------- |
| 2160          | 3840x2160       | 3840x2160         |
| 1440          | 2560x1440       | 2560x1440         |
| 1080          | 1920x1080       | 1920x1080         |
| 720           | 1280x720        | 1280x720          |
| 480           | 854x480         | 854x480           |

---

### 10. gRPC Proto File (Contracts)

Located at `TerraStream.EventService.Contracts/Protos/event_service.proto`:

```protobuf
syntax = "proto3";

option csharp_namespace = "TerraStream.EventService.Contracts.Grpc";

package terrastream.eventservice.v1;

service EventGrpcService {
  rpc GetEvent (GetEventRequest) returns (EventResponse);
  rpc GetEventsByStatus (GetEventsByStatusRequest) returns (EventListResponse);
  rpc GetEventStreamConfig (GetStreamConfigRequest) returns (StreamConfigResponse);
  rpc ValidateStreamKey (ValidateStreamKeyRequest) returns (ValidateStreamKeyResponse);
}

message GetEventRequest {
  string event_id = 1;
  string tenant_id = 2;
}

message EventResponse {
  string id = 1;
  string tenant_id = 2;
  string name = 3;
  string description = 4;
  string status = 5;
  string created_at = 6;          // ISO 8601
  string updated_at = 7;          // ISO 8601
  ScheduleInfo schedule = 8;
  StreamConfigInfo stream_config = 9;
}

message ScheduleInfo {
  string scheduled_start = 1;     // ISO 8601
  string scheduled_end = 2;       // ISO 8601
  string timezone = 3;
}

message StreamConfigInfo {
  string stream_key = 1;
  string ingest_endpoint = 2;
  string protocol = 3;
  int32 max_resolution = 4;
  int32 max_bitrate_kbps = 5;
}

message GetEventsByStatusRequest {
  string status = 1;
  string tenant_id = 2;
  int32 page_number = 3;
  int32 page_size = 4;
}

message EventListResponse {
  repeated EventResponse events = 1;
  int32 total_count = 2;
}

message GetStreamConfigRequest {
  string event_id = 1;
  string tenant_id = 2;
}

message StreamConfigResponse {
  string stream_key = 1;
  string ingest_endpoint = 2;
  string protocol = 3;
  int32 max_resolution = 4;
  int32 max_bitrate_kbps = 5;
  bool recording_enabled = 6;
}

message ValidateStreamKeyRequest {
  string stream_key = 1;
}

message ValidateStreamKeyResponse {
  bool is_valid = 1;
  string event_id = 2;
  string tenant_id = 3;
  string reject_reason = 4;
}
```

---

## Implementation Tasks (Ordered)

### Task 3.0: Prerequisite — Domain Adjustment

**Scope:** `TerraStream.EventService.Domain`

1. Add `GetByStreamKeyAsync(StreamKey, CancellationToken)` overload to `IEventRepository`

**Estimated files:** 1 modified

---

### Task 3.1: Application Project Setup

**Scope:** `TerraStream.EventService.Application`

1. Add NuGet packages: `Mediator.Abstractions`, `Mediator.SourceGenerator`, `FluentValidation`, `FluentValidation.DependencyInjectionExtensions`, `Microsoft.Extensions.Options`
2. Create folder structure: `Abstractions/`, `Behaviors/`, `Commands/`, `Queries/`, `DTOs/`, `Mapping/`, `Exceptions/`

**Estimated files:** 0 new code files (structure + csproj)

---

### Task 3.2: Abstractions and Exceptions

**Scope:** `TerraStream.EventService.Application`

1. Create `Abstractions/ITenantContext.cs`
2. Create `Abstractions/IUnitOfWork.cs`
3. Create `Abstractions/IClock.cs`
4. Create `Exceptions/EventNotFoundException.cs`
5. Create `Exceptions/ValidationException.cs`
6. Create settings class `StreamingSettings.cs` (in root or `Abstractions/`)

**Estimated files:** 6 new

---

### Task 3.3: DTOs

**Scope:** `TerraStream.EventService.Application`

1. Create all DTO record types in `DTOs/`:
   - `EventDto.cs`
   - `EventSummaryDto.cs`
   - `EventThemeDto.cs`
   - `EventScheduleDto.cs`
   - `EventStreamConfigDto.cs`
   - `EventAccessConfigDto.cs`
   - `EventFeatureDto.cs`
   - `EventInvitationDto.cs`
   - `PagedResultDto.cs`
   - `ObsProfileDto.cs` (includes `ObsVideoSettingsDto`, `ObsAudioSettingsDto`, `ObsOutputSettingsDto`)
   - `StreamKeyValidationResultDto.cs`

**Estimated files:** 11 new

---

### Task 3.4: Mapping Extensions

**Scope:** `TerraStream.EventService.Application`

1. Create `Mapping/EventMappingExtensions.cs` with all `ToDto()` and `ToSummaryDto()` extension methods

**Estimated files:** 1 new

---

### Task 3.5: Commands — Records and Validators

**Scope:** `TerraStream.EventService.Application`

1. Create all 15 command record files in their respective folders
2. Create validators for commands that need them:
   - `CreateEventCommandValidator.cs`
   - `UpdateEventDetailsCommandValidator.cs`
   - `SetEventThemeCommandValidator.cs`
   - `SetEventScheduleCommandValidator.cs`
   - `ConfigureEventFeaturesCommandValidator.cs`
   - `SetEventAccessCommandValidator.cs`
   - `InviteUsersCommandValidator.cs`
3. Create shared helper: `FeatureConfigItem.cs` in `Commands/ConfigureEventFeatures/`

**Estimated files:** 22 new (15 commands + 7 validators)

---

### Task 3.6: Queries — Records

**Scope:** `TerraStream.EventService.Application`

1. Create all 5 query record files in their respective folders

**Estimated files:** 5 new

---

### Task 3.7: Validation Behavior

**Scope:** `TerraStream.EventService.Application`

1. Create `Behaviors/ValidationBehavior.cs`

**Estimated files:** 1 new

---

### Task 3.8: Contracts — Integration Events

**Scope:** `TerraStream.EventService.Contracts`

1. Add NuGet packages to Contracts project (if needed for proto compilation)
2. Create integration event records:
   - `IntegrationEvents/EventScheduledIntegrationEvent.cs`
   - `IntegrationEvents/EventWentLiveIntegrationEvent.cs`
   - `IntegrationEvents/EventEndedIntegrationEvent.cs`
   - `IntegrationEvents/EventCancelledIntegrationEvent.cs`

**Estimated files:** 4 new

---

### Task 3.9: Contracts — gRPC Proto

**Scope:** `TerraStream.EventService.Contracts`

1. Add gRPC NuGet packages: `Google.Protobuf`, `Grpc.Tools`, `Grpc.Net.Client`
2. Create `Protos/event_service.proto` with service definition and all message types
3. Configure `.csproj` with `<Protobuf Include="..." GrpcServices="Both" />` item group

**Estimated files:** 1 new proto file + csproj modification

---

### Task 4.1: Command Handlers — CRUD

**Scope:** `TerraStream.EventService.Application`

1. `CreateEventCommandHandler` — creates event, generates stream key, persists
2. `UpdateEventDetailsCommandHandler` — loads + updates name/description
3. `SetEventThemeCommandHandler` — loads + sets theme
4. `SetEventScheduleCommandHandler` — loads + sets schedule
5. `ConfigureEventFeaturesCommandHandler` — loads + configures features
6. `SetEventAccessCommandHandler` — loads + sets access config
7. `InviteUsersCommandHandler` — loads + adds invitations

**Estimated files:** 7 new

---

### Task 4.2: Command Handlers — State Transitions

**Scope:** `TerraStream.EventService.Application`

1. `PublishEventCommandHandler` — Draft → Scheduled
2. `StartTestStreamCommandHandler` — Scheduled → Testing
3. `StopTestStreamCommandHandler` — Testing → Scheduled
4. `GoLiveCommandHandler` — Scheduled/Testing → Live
5. `EndStreamCommandHandler` — Live → Ended
6. `ArchiveEventCommandHandler` — Ended → Archived
7. `CancelEventCommandHandler` — Draft/Scheduled/Testing → Cancelled

All follow the same thin pattern: load → `TransitionTo()` → save.

**Estimated files:** 7 new

---

### Task 4.3: Command Handlers — Stream Key Validation

**Scope:** `TerraStream.EventService.Application`

1. `ValidateStreamKeyCommandHandler` — cross-tenant lookup, status check

**Estimated files:** 1 new

---

### Task 4.4: Query Handlers

**Scope:** `TerraStream.EventService.Application`

1. `GetEventByIdQueryHandler`
2. `ListEventsQueryHandler` — paginated with filters
3. `GetEventStreamConfigQueryHandler`
4. `GetObsProfileQueryHandler` — OBS profile JSON generation
5. `GetEventsByStatusQueryHandler`

**Estimated files:** 5 new

---

### Task 4.5: Build Verification

**Scope:** Full solution

1. Verify `dotnet build` succeeds with 0 errors, 0 warnings
2. Verify all existing 152 domain unit tests still pass
3. Fix any breaking changes from the domain adjustment (Task 3.0)

---

## Task Summary

| Task | Description                                        | Files      | Depends On    |
| ---- | -------------------------------------------------- | ---------- | ------------- |
| 3.0  | Domain adjustment (cross-tenant stream key lookup) | ~1         | —             |
| 3.1  | Application project setup (packages, folders)      | csproj     | —             |
| 3.2  | Abstractions and exceptions                        | 6          | 3.1           |
| 3.3  | DTOs                                               | 11         | 3.1           |
| 3.4  | Mapping extensions                                 | 1          | 3.3           |
| 3.5  | Commands — records and validators                  | 22         | 3.2           |
| 3.6  | Queries — records                                  | 5          | 3.2, 3.3      |
| 3.7  | Validation behavior                                | 1          | 3.5           |
| 3.8  | Contracts — integration events                     | 4          | —             |
| 3.9  | Contracts — gRPC proto                             | 1 + csproj | 3.8           |
| 4.1  | Command handlers — CRUD                            | 7          | 3.0, 3.4, 3.5 |
| 4.2  | Command handlers — state transitions               | 7          | 3.0, 3.4, 3.5 |
| 4.3  | Command handler — stream key validation            | 1          | 3.0, 3.5      |
| 4.4  | Query handlers                                     | 5          | 3.4, 3.6      |
| 4.5  | Build verification                                 | 0          | all           |

**Total new files:** ~79  
**Total modified files:** ~3 (IEventRepository, Application .csproj, Contracts .csproj)

---

## Design Principles

1. **Thin handlers:** Handlers orchestrate; the domain enforces business rules. No business logic in handlers.
2. **No Infrastructure dependencies in Application:** Handlers depend on `IEventRepository`, `IUnitOfWork`, `ITenantContext` — all defined in Application/Domain. No MassTransit, no EF Core, no HTTP references.
3. **Testability:** `IClock` + `ITenantContext` + `IUnitOfWork` + `IEventRepository` are all injectable/mockable. Every handler can be unit tested in isolation.
4. **Transactional outbox:** Domain event → integration event mapping and outbox writes happen inside `IUnitOfWork.SaveChangesAsync()` in Infrastructure. Handlers never touch messaging. The outbox guarantees at-least-once delivery without dual writes. Consumers must be idempotent.
5. **Enum strings over integers:** DTOs use string representations for enums. Validators validate the string → enum parsing upfront.

---

Generated: 2026-02-15  
Depends on: Domain Layer (COMPLETE)  
Next: Infrastructure Layer (EF Core DbContext, Repository, MassTransit transactional outbox, TenantContext implementation)

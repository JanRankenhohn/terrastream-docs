# Task 2: Domain Layer Implementation Plan

## Overview

Implement the Domain layer for EventService using Domain-Driven Design (DDD) principles. The domain layer contains:
- Enums for event state and configuration
- Value objects for stream configuration
- Owned entities for event configuration (Theme, Schedule, StreamConfig, AccessConfig)
- Collection entities (EventFeature, EventInvitation)
- Event aggregate root with state machine and business logic
- Domain event marker classes
- Repository interface

---

## TL;DR

The Domain layer is the foundation of the service. It defines the Event aggregate with owned entities, enforces a strict state machine, generates stream keys, and raises domain events. Each subtask is designed to be small, focused, testable, and builds on the previous. Implementations are tested with unit tests before moving to the Application layer.

---

## Key Decisions (Locked In)

1. **Stream Key Generation:** Explicit call via `SetStreamConfig()` method (not automatic in factory)
2. **Delete Strategy:** Soft delete using `IsDeleted` flag on Event aggregate
3. **JSON Storage:** Use `JsonDocument` from `System.Text.Json` for schema flexibility and performance
4. **Value Objects:** Use `EventId` value object for type safety; `StreamKey` as value object for format validation
5. **Concurrency:** Optimistic concurrency via `Version` property (for EF Core RowVersion)

---

## Subtasks (in order of implementation)

### 2.1: Enums and Constants

**Objective:** Create foundational enums that define valid states and configurations.

**Files to create:**
- `src/TerraStream.EventService.Domain/Enums/EventStatus.cs`
- `src/TerraStream.EventService.Domain/Enums/StreamProtocol.cs`
- `src/TerraStream.EventService.Domain/Enums/AccessMode.cs`
- `src/TerraStream.EventService.Domain/Enums/InvitationStatus.cs`

**EventStatus enum values:**
- Draft
- Scheduled
- Testing
- Live
- Ended
- Archived
- Cancelled

**StreamProtocol enum values:**
- RTMP
- SRT

**AccessMode enum values:**
- Public
- InviteOnly
- IdpRestricted

**InvitationStatus enum values:**
- Pending
- Sent
- Failed

**Dependencies:** None

**Validation:**
- All files compile
- Each enum has XML doc comments
- Code follows .editorconfig (2-space indent)

---

### 2.2: Value Objects

**Objective:** Create immutable, reusable types for strongly-typed domain concepts.

**Files to create:**
- `src/TerraStream.EventService.Domain/ValueObjects/EventId.cs`
- `src/TerraStream.EventService.Domain/ValueObjects/StreamKey.cs`

**EventId.cs:**
- Wraps `Guid` identifier
- Constructor validates non-empty Guid
- Implements `IEquatable<EventId>`
- Private constructor for parameter validation
- Static factory method `public static EventId Create(Guid value)` or implicit operators for convenience

**StreamKey.cs:**
- Wraps `string` for stream key
- Private constructor
- Static factory method `public static StreamKey Generate(EventId eventId)` that creates format: `evt_{eventId.Value.ToString("N").Substring(0, 8)}_sk_{12 random hex chars}`
- Method `public static StreamKey FromString(string value)` for hydration (validates format)
- Implements `IEquatable<StreamKey>`
- Does not contain business logic beyond formatting/validation

**Dependencies:** None (uses Guid, string, System.Text.Json)

**Validation:**
- Both classes compile
- Factory methods tested manually (will have unit tests in 2.14-2.15)
- Equality comparison works correctly

---

### 2.3: Owned Entities

**Objective:** Create aggregate-scoped entities that cannot exist independently.

**Files to create:**
- `src/TerraStream.EventService.Domain/Entities/EventTheme.cs`
- `src/TerraStream.EventService.Domain/Entities/EventSchedule.cs`
- `src/TerraStream.EventService.Domain/Entities/EventStreamConfig.cs`
- `src/TerraStream.EventService.Domain/Entities/EventAccessConfig.cs`

**EventTheme.cs:**
- Private constructor (owned entity pattern)
- Properties: `PrimaryColor` (string), `SecondaryColor` (string?), `AccentColor` (string?), `LogoAssetId` (Guid?), `WatermarkAssetId` (Guid?), `PosterAssetId` (Guid?), `HidePlatformBranding` (bool), `CustomCss` (string?)
- Static factory method `public static EventTheme Create(...)` for aggregate to call

**EventSchedule.cs:**
- Private constructor
- Properties: `ScheduledStart` (DateTimeOffset), `ScheduledEnd` (DateTimeOffset), `Timezone` (string, e.g., "Europe/Berlin"), `LobbyMinutesBefore` (int, default 15), `LobbyContentAssetId` (Guid?), `PostEventContentAssetId` (Guid?)
- Static factory method validates: StartDate < EndDate, timezone is valid (test against TimeZoneInfo)
- Both properties immutable (no setters after creation)

**EventStreamConfig.cs:**
- Private constructor
- Properties: `StreamKey` (StreamKey value object), `IngestEndpoint` (string, e.g., "rtmp://ingest.terrastream.de/live"), `Protocol` (StreamProtocol enum), `MaxResolution` (int, e.g., 1080), `MaxBitrateKbps` (int, e.g., 5000), `RecordingEnabled` (bool), `AbsLadderConfig` (JsonDocument or string?—decide per 3. above)
- Static factory method validates: StreamKey not null, MaxResolution > 0, MaxBitrateKbps > 0
- AbsLadderConfig defaults to standard ABR ladder if not provided

**EventAccessConfig.cs:**
- Private constructor
- Properties: `AccessMode` (AccessMode enum), `MaxConcurrentViewers` (int?), `RequireAuthentication` (bool)
- Static factory method validates: MaxConcurrentViewers > 0 if set

**Dependencies:** Enums (2.1), StreamKey (2.2)

**Pattern Note:** All owned entities have private constructors. The Event aggregate (created in 2.7-2.10) will provide public methods like `SetTheme()`, `SetSchedule()`, etc., that create instances via static factories.

**Validation:**
- All files compile
- Value types are immutable (no public setters)
- Factories perform basic validation

---

### 2.4: Collection Entities

**Objective:** Create entities that belong to the aggregate but can be reasoned about independently.

**Files to create:**
- `src/TerraStream.EventService.Domain/Entities/EventFeature.cs`
- `src/TerraStream.EventService.Domain/Entities/EventInvitation.cs`

**EventFeature.cs:**
- Public constructor: `public EventFeature(Guid id, string featureType, bool enabled, JsonDocument? config = null)`
- Or factory method if validation needed
- Properties: `Id` (Guid), `FeatureType` (string, e.g., "QnA", "Chat", "Reactions", "Analytics"), `Enabled` (bool), `Config` (JsonDocument?)
- Can be constructed freely (added to collection by aggregate)
- Implements `IEquatable<EventFeature>` by Id

**EventInvitation.cs:**
- Public constructor or factory
- Properties: `Id` (Guid), `Email` (string), `Status` (InvitationStatus enum), `SentAt` (DateTime?)
- Email validated as basic format (x@y pattern, or use `System.Net.Mail.MailAddress`)
- `SentAt` is null when Status is Pending, set when Status transitions to Sent/Failed

**Dependencies:** Enums (2.1)

**Validation:**
- Both files compile
- Email validation does not throw on valid formats
- Constructors accept reasonable defaults

---

### 2.5: Domain Event Base Interface

**Objective:** Define the contract for domain events raised by the aggregate.

**Files to create:**
- `src/TerraStream.EventService.Domain/Events/IDomainEvent.cs`

**IDomainEvent interface:**
```csharp
public interface IDomainEvent
{
  Guid EventId { get; }
  Guid TenantId { get; }
  DateTime OccurredAt { get; }
}
```

**Dependencies:** None

**Validation:**
- Interface compiles
- Clear and minimal contract
- Will be implemented by event classes in 2.6

---

### 2.6: Domain Event Classes

**Objective:** Create immutable event classes raised when aggregate transitions state.

**Files to create:**
- `src/TerraStream.EventService.Domain/Events/EventScheduledDomainEvent.cs`
- `src/TerraStream.EventService.Domain/Events/EventWentLiveDomainEvent.cs`
- `src/TerraStream.EventService.Domain/Events/EventEndedDomainEvent.cs`
- `src/TerraStream.EventService.Domain/Events/EventCancelledDomainEvent.cs`

**EventScheduledDomainEvent:**
- Implements IDomainEvent
- Properties: `EventId`, `TenantId`, `OccurredAt`, `ScheduledStart` (DateTimeOffset), `ExpectedViewerCount` (int?)
- Constructor sets all from parameters; OccurredAt defaults to DateTime.UtcNow if not provided

**EventWentLiveDomainEvent:**
- Properties: `EventId`, `TenantId`, `OccurredAt`, `StreamKey` (string), `IngestEndpoint` (string)

**EventEndedDomainEvent:**
- Properties: `EventId`, `TenantId`, `OccurredAt`, `Duration` (TimeSpan?)

**EventCancelledDomainEvent:**
- Properties: `EventId`, `TenantId`, `OccurredAt`, `Reason` (string?)

**Dependencies:** IDomainEvent (2.5)

**Validation:**
- All classes compile
- Implement IDomainEvent correctly
- Properties are immutable (init-only or read-only)

---

### 2.7: Event Aggregate Root — Core Structure

**Objective:** Create the Event aggregate root with essential properties and initialization logic.

**Files to create:**
- `src/TerraStream.EventService.Domain/Aggregates/Event.cs` (beginning)

**Event.cs — Part 1 (Core properties and factory):**

```csharp
public class Event
{
  public Guid Id { get; private set; }
  public Guid TenantId { get; private set; }
  public string Name { get; private set; }
  public string? Description { get; private set; }
  public EventStatus Status { get; private set; }
  public DateTime CreatedAt { get; private set; }
  public DateTime UpdatedAt { get; private set; }
  public int Version { get; set; }  // For optimistic concurrency / RowVersion
  public bool IsDeleted { get; private set; } // For soft delete

  // Owned entities (initialized separately)
  public EventTheme? Theme { get; private set; }
  public EventSchedule? Schedule { get; private set; }
  public EventStreamConfig? StreamConfig { get; private set; }
  public EventAccessConfig? AccessConfig { get; private set; }

  // Collections
  private readonly List<EventFeature> _features = new();
  private readonly List<EventInvitation> _invitations = new();

  public IReadOnlyCollection<EventFeature> Features => _features.AsReadOnly();
  public IReadOnlyCollection<EventInvitation> Invitations => _invitations.AsReadOnly();

  // Domain events collection (added in 2.11)
  private readonly List<IDomainEvent> _domainEvents = new();
  public IReadOnlyList<IDomainEvent> DomainEvents => _domainEvents.AsReadOnly();

  // Private constructor
  private Event(Guid id, Guid tenantId, string name, string? description)
  {
    Id = id;
    TenantId = tenantId;
    Name = name;
    Description = description;
    Status = EventStatus.Draft;
    CreatedAt = DateTime.UtcNow;
    UpdatedAt = CreatedAt;
    IsDeleted = false;
    Version = 1;
  }

  // Factory method
  public static Event Create(Guid tenantId, string name, string? description = null)
  {
    if (string.IsNullOrWhiteSpace(name)) throw new ArgumentException("Name cannot be empty", nameof(name));
    if (name.Length > 200) throw new ArgumentException("Name cannot exceed 200 characters", nameof(name));
    
    return new Event(Guid.NewGuid(), tenantId, name, description);
  }
}
```

**Key features:**
- Factory method validates name (required, max 200 chars)
- Constructor is private; only factory creates instances
- Status initialized to Draft
- Timestamps are UTC
- IsDeleted for soft delete pattern
- Version for optimistic concurrency

**Dependencies:** Enums (2.1), Owned Entities (2.3), Collection Entities (2.4)

**Validation:**
- File compiles
- Factory creates Draft events
- Properties accessible

---

### 2.8: Event State Machine

**Objective:** Implement state transition logic.

**Files to edit:**
- `src/TerraStream.EventService.Domain/Aggregates/Event.cs` — Add state machine method

**Add method to Event:**

```csharp
public void TransitionTo(EventStatus targetStatus)
{
  // Define valid transitions
  var validTransitions = new Dictionary<EventStatus, HashSet<EventStatus>>
  {
    { EventStatus.Draft, new() { EventStatus.Scheduled, EventStatus.Cancelled } },
    { EventStatus.Scheduled, new() { EventStatus.Testing, EventStatus.Live, EventStatus.Cancelled } },
    { EventStatus.Testing, new() { EventStatus.Scheduled, EventStatus.Live } },
    { EventStatus.Live, new() { EventStatus.Ended } },
    { EventStatus.Ended, new() { EventStatus.Archived } },
    { EventStatus.Archived, new() },  // Terminal state
    { EventStatus.Cancelled, new() }   // Terminal state
  };

  if (!validTransitions.TryGetValue(Status, out var allowed) || !allowed.Contains(targetStatus))
  {
    throw new InvalidOperationException(
      $"Cannot transition from '{Status}' to '{targetStatus}'. " +
      $"Valid transitions from '{Status}' are: {string.Join(", ", allowed)}");
  }

  Status = targetStatus;
  UpdatedAt = DateTime.UtcNow;

  // Domain events raised (implemented in 2.11)
}
```

**Dependencies:** Event (2.7), Enums (2.1)

**State diagram (from architecture plan):**
```
Draft → Scheduled → Testing → Live → Ended → Archived
  ↓                   ↑        ↓
  └─────── Cancelled ←─┴────────┘ (Cancelled is input from Draft, Scheduled, Testing)
```

**Validation:**
- Method compiles
- Exception message is clear
- Invalid transitions are caught

---

### 2.9: Event Business Logic — Owned Entity Management and Stream Key

**Objective:** Add methods to initialize and manage owned entities.

**Files to edit:**
- `src/TerraStream.EventService.Domain/Aggregates/Event.cs` — Add setter methods

**Add methods to Event:**

```csharp
public void GenerateStreamKey(string ingestEndpoint = "rtmp://ingest.terrastream.de/live")
{
  // Idempotent: if already set, don't regenerate
  if (StreamConfig != null) return;

  var streamKey = StreamKey.Generate(new EventId(Id));
  StreamConfig = EventStreamConfig.Create(
    streamKey: streamKey,
    ingestEndpoint: ingestEndpoint,
    protocol: StreamProtocol.RTMP,
    maxResolution: 1080,
    maxBitrateKbps: 5000,
    recordingEnabled: true,
    absLadderConfig: GetDefaultAbsLadderConfig()
  );

  UpdatedAt = DateTime.UtcNow;
}

public void SetTheme(
  string primaryColor,
  string? secondaryColor = null,
  string? accentColor = null,
  Guid? logoAssetId = null,
  Guid? watermarkAssetId = null,
  Guid? posterAssetId = null,
  bool hidePlatformBranding = false,
  string? customCss = null)
{
  Theme = EventTheme.Create(primaryColor, secondaryColor, accentColor, logoAssetId, watermarkAssetId, posterAssetId, hidePlatformBranding, customCss);
  UpdatedAt = DateTime.UtcNow;
}

public void SetSchedule(DateTimeOffset scheduledStart, DateTimeOffset scheduledEnd, string timezone, int lobbyMinutesBefore = 15)
{
  if (scheduledStart >= scheduledEnd) throw new ArgumentException("Start time must be before end time");
  if (scheduledEnd < DateTimeOffset.UtcNow) throw new ArgumentException("End time cannot be in the past");

  Schedule = EventSchedule.Create(scheduledStart, scheduledEnd, timezone, lobbyMinutesBefore);
  UpdatedAt = DateTime.UtcNow;
}

public void SetStreamConfig(
  StreamProtocol protocol = StreamProtocol.RTMP,
  int maxResolution = 1080,
  int maxBitrateKbps = 5000,
  bool recordingEnabled = true)
{
  // GenerateStreamKey must have been called
  if (StreamConfig == null)
  {
    GenerateStreamKey();
  }

  // Update config (or keep existing stream key)
  var existingKey = StreamConfig.StreamKey;
  StreamConfig = EventStreamConfig.Create(
    streamKey: existingKey,
    ingestEndpoint: StreamConfig.IngestEndpoint,
    protocol: protocol,
    maxResolution: maxResolution,
    maxBitrateKbps: maxBitrateKbps,
    recordingEnabled: recordingEnabled,
    absLadderConfig: StreamConfig.AbsLadderConfig
  );

  UpdatedAt = DateTime.UtcNow;
}

public void SetAccessConfig(AccessMode accessMode, int? maxConcurrentViewers = null, bool requireAuthentication = false)
{
  AccessConfig = EventAccessConfig.Create(accessMode, maxConcurrentViewers, requireAuthentication);
  UpdatedAt = DateTime.UtcNow;
}

private static JsonDocument GetDefaultAbsLadderConfig()
{
  // Returns default ABR ladder: Source, 1080p 5000kbps, 720p 2500kbps, 480p 1000kbps, audio-only 128kbps
  var json = @"{
    ""ladder"": [
      { ""quality"": ""source"", ""resolution"": null, ""bitrate"": null, ""fps"": null },
      { ""quality"": ""high"", ""resolution"": ""1920x1080"", ""bitrate"": 5000, ""fps"": 30 },
      { ""quality"": ""medium"", ""resolution"": ""1280x720"", ""bitrate"": 2500, ""fps"": 30 },
      { ""quality"": ""low"", ""resolution"": ""854x480"", ""bitrate"": 1000, ""fps"": 30 },
      { ""quality"": ""audio-only"", ""resolution"": null, ""bitrate"": 128, ""fps"": null }
    ]
  }";
  return JsonDocument.Parse(json);
}
```

**Dependencies:** Event (2.7), Owned Entities (2.3), StreamKey (2.2), Enums (2.1)

**Key decisions:**
- `GenerateStreamKey()` is explicit, called on-demand (per decision 1)
- `SetSchedule()` validates start < end and not in past
- `SetStreamConfig()` calls `GenerateStreamKey()` if not yet set, preserving the stream key
- ABR ladder is provided as default JSON via helper method

**Validation:**
- Methods compile
- Setting owned entities updates `UpdatedAt`
- Validation throws appropriate exceptions

---

### 2.10: Event Feature & Invitation Management

**Objective:** Add methods to manage dynamic collections.

**Files to edit:**
- `src/TerraStream.EventService.Domain/Aggregates/Event.cs` — Add collection methods

**Add methods to Event:**

```csharp
public void ConfigureFeature(string featureType, bool enabled, JsonDocument? config = null)
{
  if (string.IsNullOrWhiteSpace(featureType)) throw new ArgumentException("Feature type cannot be empty", nameof(featureType));

  // Upsert pattern
  var existingFeature = _features.FirstOrDefault(f => f.FeatureType == featureType);
  if (existingFeature != null)
  {
    _features.Remove(existingFeature);
  }

  var feature = new EventFeature(Guid.NewGuid(), featureType, enabled, config);
  _features.Add(feature);

  UpdatedAt = DateTime.UtcNow;
}

public void AddInvitation(string email)
{
  if (string.IsNullOrWhiteSpace(email)) throw new ArgumentException("Email cannot be empty", nameof(email));

  // Basic email validation
  try
  {
    var addr = new System.Net.Mail.MailAddress(email);
  }
  catch
  {
    throw new ArgumentException($"Invalid email format: {email}", nameof(email));
  }

  var invitation = new EventInvitation(Guid.NewGuid(), email, InvitationStatus.Pending, sentAt: null);
  _invitations.Add(invitation);

  UpdatedAt = DateTime.UtcNow;
}

public IReadOnlyCollection<EventInvitation> GetPendingInvitations()
  => _invitations.Where(i => i.Status == InvitationStatus.Pending).ToList().AsReadOnly();

public void MarkInvitationAsSent(Guid invitationId, DateTime sentAt)
{
  var invitation = _invitations.FirstOrDefault(i => i.Id == invitationId);
  if (invitation == null) throw new KeyNotFoundException($"Invitation {invitationId} not found");

  invitation.Status = InvitationStatus.Sent;
  invitation.SentAt = sentAt;

  UpdatedAt = DateTime.UtcNow;
}
```

**Dependencies:** Event (2.7), Collection Entities (2.4), Enums (2.1)

**Key decisions:**
- `ConfigureFeature()` uses upsert pattern: if feature type exists, replace it
- `AddInvitation()` validates email format using `MailAddress`
- `GetPendingInvitations()` is a filter helper
- `MarkInvitationAsSent()` allows infrastructure to update invitation status (called by NotificationService)

**Validation:**
- Methods compile
- Collections grow correctly
- Upsert logic works as expected

---

### 2.11: Domain Event Raising

**Objective:** Wire domain events to state transitions.

**Files to edit:**
- `src/TerraStream.EventService.Domain/Aggregates/Event.cs` — Implement domain event methods

**Add methods to Event:**

```csharp
protected void RaiseDomainEvent(IDomainEvent domainEvent)
{
  _domainEvents.Add(domainEvent);
}

public void ClearDomainEvents()
{
  _domainEvents.Clear();
}
```

**Update `TransitionTo()` method to raise events:**

```csharp
public void TransitionTo(EventStatus targetStatus)
{
  var validTransitions = new Dictionary<EventStatus, HashSet<EventStatus>>
  {
    { EventStatus.Draft, new() { EventStatus.Scheduled, EventStatus.Cancelled } },
    { EventStatus.Scheduled, new() { EventStatus.Testing, EventStatus.Live, EventStatus.Cancelled } },
    { EventStatus.Testing, new() { EventStatus.Scheduled, EventStatus.Live } },
    { EventStatus.Live, new() { EventStatus.Ended } },
    { EventStatus.Ended, new() { EventStatus.Archived } },
    { EventStatus.Archived, new() },
    { EventStatus.Cancelled, new() }
  };

  if (!validTransitions.TryGetValue(Status, out var allowed) || !allowed.Contains(targetStatus))
  {
    throw new InvalidOperationException(
      $"Cannot transition from '{Status}' to '{targetStatus}'.");
  }

  Status = targetStatus;
  UpdatedAt = DateTime.UtcNow;

  // Raise domain events based on transition
  switch (targetStatus)
  {
    case EventStatus.Scheduled:
      RaiseDomainEvent(new EventScheduledDomainEvent(
        EventId: Id,
        TenantId: TenantId,
        OccurredAt: DateTime.UtcNow,
        ScheduledStart: Schedule?.ScheduledStart,
        ExpectedViewerCount: AccessConfig?.MaxConcurrentViewers
      ));
      break;

    case EventStatus.Live:
      RaiseDomainEvent(new EventWentLiveDomainEvent(
        EventId: Id,
        TenantId: TenantId,
        OccurredAt: DateTime.UtcNow,
        StreamKey: StreamConfig?.StreamKey.Value ?? string.Empty,
        IngestEndpoint: StreamConfig?.IngestEndpoint ?? string.Empty
      ));
      break;

    case EventStatus.Ended:
      RaiseDomainEvent(new EventEndedDomainEvent(
        EventId: Id,
        TenantId: TenantId,
        OccurredAt: DateTime.UtcNow,
        Duration: Schedule != null ? Schedule.ScheduledEnd - Schedule.ScheduledStart : null
      ));
      break;

    case EventStatus.Cancelled:
      RaiseDomainEvent(new EventCancelledDomainEvent(
        EventId: Id,
        TenantId: TenantId,
        OccurredAt: DateTime.UtcNow,
        Reason: null
      ));
      break;
  }
}
```

**Dependencies:** Event (2.8), Domain Events (2.6)

**Key decisions:**
- Domain events are raised in `TransitionTo()` method
- Events carry relevant data from the aggregate (stream key, ingest endpoint, duration, etc.)
- `ClearDomainEvents()` is called by infrastructure after publishing
- Events are **not** persisted in the domain layer; infrastructure publishes them

**Validation:**
- Methods compile
- Domain events are added to `_domainEvents` collection
- Events are properly structured with required fields

---

### 2.12: Repository Interface

**Objective:** Define the contract for data persistence.

**Files to create:**
- `src/TerraStream.EventService.Domain/Repositories/IEventRepository.cs`
- `src/TerraStream.EventService.Domain/Repositories/EventQueryFilter.cs`

**IEventRepository.cs:**

```csharp
public interface IEventRepository
{
  Task<Event?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);

  Task<Event?> GetByStreamKeyAsync(string streamKey, CancellationToken cancellationToken = default);

  Task<(IEnumerable<Event> Items, int TotalCount)> ListAsync(
    EventQueryFilter filter,
    CancellationToken cancellationToken = default);

  Task AddAsync(Event @event, CancellationToken cancellationToken = default);

  Task UpdateAsync(Event @event, CancellationToken cancellationToken = default);

  Task DeleteAsync(Guid id, CancellationToken cancellationToken = default); // Soft delete
}
```

**EventQueryFilter.cs:**

```csharp
public class EventQueryFilter
{
  public EventStatus? Status { get; set; }
  public DateRange? DateRange { get; set; }
  public Guid? TenantId { get; set; } // Implicit from ITenantContext, but included for clarity
  public int PageNumber { get; set; } = 1;
  public int PageSize { get; set; } = 20;
  public string? SearchText { get; set; } // Searches Name and Description
  public bool IncludeDeleted { get; set; } = false;

  public class DateRange
  {
    public DateTimeOffset Start { get; set; }
    public DateTimeOffset End { get; set; }
  }
}
```

**Dependencies:** Event (2.7), Enums (2.1)

**Key decisions:**
- Repository filters by TenantId (for multi-tenancy)
- `DeleteAsync()` performs soft delete (sets IsDeleted = true)
- `ListAsync()` returns tuple with items and total count (for pagination)
- `GetByStreamKeyAsync()` used by SRS ingest validation

**Validation:**
- Interface and classes compile
- Method signatures are clear
- Will be implemented by Infrastructure layer

---

### 2.13: Unit Tests — State Machine

**Objective:** Test state transition logic exhaustively.

**Files to create:**
- `tests/TerraStream.EventService.UnitTests/Domain/EventStateTransitionTests.cs`

**Test scenarios:**
- Valid transition: Draft → Scheduled ✓
- Valid transition: Draft → Cancelled ✓
- Valid transition: Scheduled → Testing ✓
- Valid transition: Scheduled → Live ✓
- Valid transition: Scheduled → Cancelled ✓
- Valid transition: Testing → Scheduled ✓
- Valid transition: Testing → Live ✓
- Valid transition: Live → Ended ✓
- Valid transition: Ended → Archived ✓
- Invalid transition: Draft → Live (should throw) ✗
- Invalid transition: Draft → Testing (should throw) ✗
- Invalid transition: Live → Scheduled (should throw) ✗
- Invalid transition: Ended → Live (should throw) ✗
- Invalid transition: Archived → anything (should throw) ✗
- Invalid transition: Cancelled → anything (should throw) ✗

**Use xUnit `Theory` with `InlineData` for parameterized testing:**

```csharp
[Theory]
[MemberData(nameof(ValidTransitions))]
public void TransitionTo_WithValidTransition_ChangesStatus(EventStatus from, EventStatus to)
{
  // Arrange
  var evt = Event.Create(Guid.NewGuid(), "Test Event");
  SetEventStatus(evt, from); // Helper to set status without validation

  // Act
  evt.TransitionTo(to);

  // Assert
  Assert.Equal(to, evt.Status);
  Assert.True(evt.UpdatedAt > evt.CreatedAt);
}

[Theory]
[MemberData(nameof(InvalidTransitions))]
public void TransitionTo_WithInvalidTransition_Throws(EventStatus from, EventStatus to)
{
  // Arrange
  var evt = Event.Create(Guid.NewGuid(), "Test Event");
  SetEventStatus(evt, from);

  // Act & Assert
  Assert.Throws<InvalidOperationException>(() => evt.TransitionTo(to));
}

public static TheoryData<EventStatus, EventStatus> ValidTransitions() => new()
{
  { EventStatus.Draft, EventStatus.Scheduled },
  { EventStatus.Draft, EventStatus.Cancelled },
  // ... continue for all 9 valid transitions
};

public static TheoryData<EventStatus, EventStatus> InvalidTransitions() => new()
{
  { EventStatus.Draft, EventStatus.Live },
  { EventStatus.Draft, EventStatus.Testing },
  // ... continue for all invalid transitions
};
```

**Dependencies:** Event (2.7), EventStatus enum (2.1)

**Validation:**
- All tests pass
- Invalid transitions throw `InvalidOperationException`
- Valid transitions update `Status` and `UpdatedAt`

---

### 2.14: Unit Tests — Stream Key Generation

**Objective:** Test stream key generation logic.

**Files to create:**
- `tests/TerraStream.EventService.UnitTests/Domain/EventStreamKeyGenerationTests.cs`

**Test scenarios:**
- `GenerateStreamKey()` creates key matching format `evt_{id}_sk_{random}`
- `GenerateStreamKey()` is idempotent (calling twice returns same key)
- `GenerateStreamKey()` updates `UpdatedAt`
- `GenerateStreamKey()` sets `StreamConfig` property
- Two events generate different stream keys
- Stream key includes first 8 chars of event ID

**Example tests:**

```csharp
[Fact]
public void GenerateStreamKey_CreatesValidFormat()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  evt.GenerateStreamKey();

  Assert.NotNull(evt.StreamConfig);
  var key = evt.StreamConfig.StreamKey.Value;
  Assert.Matches(@"^evt_[a-f0-9]{8}_sk_[a-f0-9]{12}$", key);
}

[Fact]
public void GenerateStreamKey_IsIdempotent()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  evt.GenerateStreamKey();
  var firstKey = evt.StreamConfig!.StreamKey;

  evt.GenerateStreamKey();
  var secondKey = evt.StreamConfig!.StreamKey;

  Assert.Equal(firstKey, secondKey);
}

[Fact]
public void GenerateStreamKey_UniquePerEvent()
{
  var evt1 = Event.Create(Guid.NewGuid(), "Test 1");
  var evt2 = Event.Create(Guid.NewGuid(), "Test 2");

  evt1.GenerateStreamKey();
  evt2.GenerateStreamKey();

  Assert.NotEqual(evt1.StreamConfig!.StreamKey, evt2.StreamConfig!.StreamKey);
}
```

**Dependencies:** Event (2.9), StreamKey (2.2)

**Validation:**
- All tests pass
- Stream key format is correct
- Idempotency works

---

### 2.15: Unit Tests — Aggregate Creation & Initialization

**Objective:** Test owned entity management and validation.

**Files to create:**
- `tests/TerraStream.EventService.UnitTests/Domain/EventAggregateTests.cs`

**Test scenarios:**
- `Event.Create()` creates Draft event
- Created event has generated Id and TenantId
- Owned entities are null after creation
- `SetTheme()` initializes Theme
- `SetSchedule()` initializes Schedule with validation
- `SetSchedule()` with StartDate >= EndDate throws `ArgumentException`
- `SetSchedule()` with future EndDate accepts it
- `SetStreamConfig()` initializes StreamConfig
- `SetAccessConfig()` initializes AccessConfig
- `UpdatedAt` is bumped on every modification
- `ConfigureFeature()` upserts feature by type
- Calling `ConfigureFeature()` twice with same type replaces it
- Features collection grows correctly
- `AddInvitation()` creates Pending invitation
- `AddInvitation()` with invalid email throws `ArgumentException`
- `GetPendingInvitations()` returns only Pending invitations

**Example tests:**

```csharp
[Fact]
public void Create_InitializesDraftEvent()
{
  var tenantId = Guid.NewGuid();
  var evt = Event.Create(tenantId, "My Event", "Description");

  Assert.Equal(EventStatus.Draft, evt.Status);
  Assert.Equal(tenantId, evt.TenantId);
  Assert.Equal("My Event", evt.Name);
  Assert.Null(evt.Theme);
  Assert.Null(evt.Schedule);
  Assert.Null(evt.StreamConfig);
  Assert.Null(evt.AccessConfig);
}

[Fact]
public void SetSchedule_WithStartBeforeEnd_Succeeds()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  var start = DateTimeOffset.UtcNow.AddHours(1);
  var end = start.AddHours(2);

  evt.SetSchedule(start, end, "Europe/Berlin");

  Assert.NotNull(evt.Schedule);
  Assert.Equal(start, evt.Schedule.ScheduledStart);
}

[Fact]
public void SetSchedule_WithStartAfterEnd_Throws()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  var start = DateTimeOffset.UtcNow.AddHours(2);
  var end = start.AddHours(1);

  Assert.Throws<ArgumentException>(() => evt.SetSchedule(start, end, "Europe/Berlin"));
}

[Fact]
public void ConfigureFeature_Upserts()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  
  evt.ConfigureFeature("QnA", true);
  Assert.Single(evt.Features);
  
  evt.ConfigureFeature("QnA", false); // Update same feature
  Assert.Single(evt.Features); // Still 1, not 2
  Assert.False(evt.Features.First().Enabled);
}

[Fact]
public void AddInvitation_WithInvalidEmail_Throws()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  Assert.Throws<ArgumentException>(() => evt.AddInvitation("notanemail"));
}
```

**Dependencies:** Event (2.9, 2.10), Owned Entities (2.3), Collection Entities (2.4)

**Validation:**
- All tests pass
- Validation rules enforced
- Collections behave correctly

---

### 2.16: Unit Tests — Domain Events

**Objective:** Test that domain events are raised when aggregate transitions state.

**Files to create:**
- `tests/TerraStream.EventService.UnitTests/Domain/EventDomainEventTests.cs`

**Test scenarios:**
- `DomainEvents` is empty on creation
- `TransitionTo(Scheduled)` raises `EventScheduledDomainEvent`
- `TransitionTo(Live)` raises `EventWentLiveDomainEvent`
- `TransitionTo(Ended)` raises `EventEndedDomainEvent`
- `TransitionTo(Cancelled)` raises `EventCancelledDomainEvent`
- Domain event contains correct EventId, TenantId, OccurredAt
- Domain event includes relevant data (stream key, ingest endpoint, etc.)
- `ClearDomainEvents()` empties the collection

**Example tests:**

```csharp
[Fact]
public void DomainEvents_EmptyOnCreation()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  Assert.Empty(evt.DomainEvents);
}

[Fact]
public void TransitionTo_Scheduled_RaisesEventScheduledDomainEvent()
{
  var tenantId = Guid.NewGuid();
  var evt = Event.Create(tenantId, "Test");

  evt.TransitionTo(EventStatus.Scheduled);

  Assert.Single(evt.DomainEvents);
  var domainEvent = evt.DomainEvents.First() as EventScheduledDomainEvent;
  Assert.NotNull(domainEvent);
  Assert.Equal(evt.Id, domainEvent.EventId);
  Assert.Equal(tenantId, domainEvent.TenantId);
}

[Fact]
public void TransitionTo_Live_RaisesEventWentLiveDomainEvent()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  
  // Setup: create stream config with key
  evt.GenerateStreamKey();
  
  // Move to Scheduled first
  evt.TransitionTo(EventStatus.Scheduled);
  evt.ClearDomainEvents();
  
  // Now go Live
  evt.TransitionTo(EventStatus.Live);

  var domainEvent = evt.DomainEvents.First() as EventWentLiveDomainEvent;
  Assert.NotNull(domainEvent);
  Assert.NotNull(domainEvent.StreamKey);
  Assert.NotNull(domainEvent.IngestEndpoint);
}

[Fact]
public void ClearDomainEvents_RemovesAllEvents()
{
  var evt = Event.Create(Guid.NewGuid(), "Test");
  evt.TransitionTo(EventStatus.Scheduled);
  
  Assert.NotEmpty(evt.DomainEvents);
  
  evt.ClearDomainEvents();
  
  Assert.Empty(evt.DomainEvents);
}
```

**Dependencies:** Event (2.11), Domain Events (2.6)

**Validation:**
- All tests pass
- Events are raised correctly
- Event properties contain expected data

---

### 2.17: Code Organization and Final Verification

**Objective:** Organize domain classes into appropriate folders and verify the complete Domain layer.

**Folder structure:**

```
src/TerraStream.EventService.Domain/
├── Enums/
│   ├── EventStatus.cs
│   ├── StreamProtocol.cs
│   ├── AccessMode.cs
│   └── InvitationStatus.cs
├── ValueObjects/
│   ├── EventId.cs
│   └── StreamKey.cs
├── Entities/
│   ├── EventTheme.cs
│   ├── EventSchedule.cs
│   ├── EventStreamConfig.cs
│   ├── EventAccessConfig.cs
│   ├── EventFeature.cs
│   └── EventInvitation.cs
├── Aggregates/
│   └── Event.cs
├── Events/
│   ├── IDomainEvent.cs
│   ├── EventScheduledDomainEvent.cs
│   ├── EventWentLiveDomainEvent.cs
│   ├── EventEndedDomainEvent.cs
│   └── EventCancelledDomainEvent.cs
└── Repositories/
    ├── IEventRepository.cs
    └── EventQueryFilter.cs

tests/TerraStream.EventService.UnitTests/
└── Domain/
    ├── EventStateTransitionTests.cs
    ├── EventStreamKeyGenerationTests.cs
    ├── EventAggregateTests.cs
    ├── EventDomainEventTests.cs
    └── [Helpers or Fixtures if needed]
```

**Verification checklist:**
- ✓ All domain classes compile without errors
- ✓ All unit tests pass (xUnit: `dotnet test tests/TerraStream.EventService.UnitTests`)
- ✓ Code follows .editorconfig (2-space indent, 140-char line length)
- ✓ XML doc comments on all public types and members
- ✓ No circular dependencies or layer violations
- ✓ State machine logic is exhaustively tested (20+ test cases)
- ✓ Full solution builds: `dotnet build`
- ✓ No warnings or errors

**Build command:**
```bash
dotnet build --configuration Release
```

**Test command:**
```bash
dotnet test tests/TerraStream.EventService.UnitTests --configuration Release
```

---

## Summary

**Task 2: Domain Layer** is broken into 17 focused subtasks:
- **2.1-2.6:** Foundational types (enums, value objects, entities, events)
- **2.7-2.11:** Event aggregate root with state machine and business logic
- **2.12:** Repository interface contract
- **2.13-2.16:** Comprehensive unit tests
- **2.17:** Final organization and verification

Each subtask is designed to:
1. Be implemented independently without blocking others
2. Have clear, testable acceptance criteria
3. Build on previous subtasks in a logical order
4. Produce code that compiles and works correctly

**Total implementation effort:** Approximately 8-12 hours for experienced .NET developer (including testing).

**Next task after Task 2 completion:** Task 3 — Application Layer (Commands, Queries, Handlers, Validators)

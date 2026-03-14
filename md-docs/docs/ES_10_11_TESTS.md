# EventService Tasks 10 & 11 — Test Implementation Plan

## Status Overview

Tasks 1–9 are fully implemented. Domain tests, OBS profile tests, and the majority of integration tests already exist. This plan covers the **remaining gaps** in Task 10 (unit tests) and Task 11 (integration tests).

### Existing Coverage

| Area                                                                                               | Status              |
| -------------------------------------------------------------------------------------------------- | ------------------- |
| Domain-layer unit tests (state transitions, stream key, aggregates, value objects, domain events)  | ✅ Complete         |
| OBS profile generator + handler unit tests                                                         | ✅ Complete         |
| Integration test infrastructure (fixture, base classes, Testcontainers, Respawn)                   | ✅ Complete         |
| Integration lifecycle, CRUD, query, tenant isolation, stream key validation, repository tests      | ✅ Largely complete |
| Command handler unit tests (CreateEvent, PublishEvent, GoLive, CancelEvent, ValidateStreamKey)     | ✅ Complete         |
| Validator unit tests (CreateEventCommandValidator, SetEventScheduleCommandValidator)               | ✅ Complete         |
| Some integration test scenarios (full 7-step lifecycle, Live/Ended stream key, integration events) | ✅ Complete         |

---

## Task 10 — Unit Tests

### 10.1 — `CreateEventCommandHandlerTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/CreateEventCommandHandlerTests.cs`

**Dependencies to mock:** `IEventRepository`, `ITenantContext`, `IOptions<StreamingSettings>`

**Pattern:** Follow `GetObsProfileQueryHandlerTests.cs` — private mock fields, `CreateHandler()` factory, arrange/act/assert with Shouldly.

| Test method                                       | Asserts                                                                                        |
| ------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `Handle_ShouldCreateEventInDraftStatus`           | Event passed to `AddAsync` has `Status == EventStatus.Draft`                                   |
| `Handle_ShouldGenerateStreamKey`                  | Event passed to `AddAsync` has a non-null `StreamConfig.StreamKey`                             |
| `Handle_ShouldReturnEventDtoWithCorrectFields`    | Returned `EventDto` has correct `Name`, `TenantId`, `Status == "Draft"`                        |
| `Handle_ShouldUseCorrectIngestEndpoint_ForRTMP`   | Event's `StreamConfig.IngestEndpoint` equals `StreamingSettings.DefaultIngestEndpoint`         |
| `Handle_ShouldUseCorrectIngestEndpoint_ForSRT`    | Event's `StreamConfig.IngestEndpoint` equals `StreamingSettings.ObsDefaults.SrtIngestEndpoint` |
| `Handle_ShouldCallRepositoryAddAsync_ExactlyOnce` | `_repositoryMock.Verify(r => r.AddAsync(...), Times.Once)`                                     |

---

### 10.2 — `PublishEventCommandHandlerTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/PublishEventCommandHandlerTests.cs`

**Dependencies to mock:** `IEventRepository`, `ITenantContext`

**Setup helper:** private `CreateDraftEvent()` method using `Event.Create(...)` factory (same pattern as `GetObsProfileQueryHandlerTests`).

| Test method                                                  | Asserts                                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `Handle_ShouldTransitionToScheduled`                         | After handle, event status captured in `UpdateAsync` call is `EventStatus.Scheduled` |
| `Handle_ShouldRaiseEventScheduledDomainEvent`                | Event's `DomainEvents` contains exactly one `EventScheduledDomainEvent`              |
| `Handle_ShouldCallRepositoryUpdateAsync_ExactlyOnce`         | `UpdateAsync` called once                                                            |
| `Handle_ShouldThrowEventNotFoundException_WhenEventNotFound` | Repo returns `null` → `EventNotFoundException` thrown                                |

---

### 10.3 — `GoLiveCommandHandlerTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/GoLiveCommandHandlerTests.cs`

**Dependencies to mock:** `IEventRepository`, `ITenantContext`

**Setup helpers:** `CreateScheduledEvent()`, `CreateTestingEvent()` — create event and call `TransitionTo(...)` to set correct state before test.

| Test method                                                  | Asserts                                                 |
| ------------------------------------------------------------ | ------------------------------------------------------- |
| `Handle_ShouldTransitionToLive_FromScheduled`                | Event status after handle is `EventStatus.Live`         |
| `Handle_ShouldTransitionToLive_FromTesting`                  | Event status after handle is `EventStatus.Live`         |
| `Handle_ShouldRaiseEventWentLiveDomainEvent`                 | `DomainEvents` contains `EventWentLiveDomainEvent`      |
| `Handle_ShouldThrow_WhenEventInDraftStatus`                  | Draft → Live triggers `InvalidStateTransitionException` |
| `Handle_ShouldThrowEventNotFoundException_WhenEventNotFound` | Repo returns `null` → `EventNotFoundException`          |

---

### 10.4 — `CancelEventCommandHandlerTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/CancelEventCommandHandlerTests.cs`

**Dependencies to mock:** `IEventRepository`, `ITenantContext`

| Test method                                                  | Asserts                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `Handle_ShouldTransitionToCancelled_FromDraft`               | Status becomes `EventStatus.Cancelled`                       |
| `Handle_ShouldTransitionToCancelled_FromScheduled`           | Status becomes `EventStatus.Cancelled`                       |
| `Handle_ShouldRaiseEventCancelledDomainEvent`                | `DomainEvents` contains `EventCancelledDomainEvent`          |
| `Handle_ShouldThrow_WhenEventIsLive`                         | Live → Cancelled triggers `InvalidStateTransitionException`  |
| `Handle_ShouldThrow_WhenEventIsEnded`                        | Ended → Cancelled triggers `InvalidStateTransitionException` |
| `Handle_ShouldThrowEventNotFoundException_WhenEventNotFound` | Repo returns `null` → `EventNotFoundException`               |

---

### 10.5 — `ValidateStreamKeyQueryHandlerTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/ValidateStreamKeyQueryHandlerTests.cs`

**Dependencies to mock:** `IEventRepository` only — this handler is cross-tenant and has no `ITenantContext` dependency.

**Note:** Use the cross-tenant `GetByStreamKeyAsync(StreamKey, CancellationToken)` overload in mocks.

| Test method                                              | Asserts                                                                             |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `Handle_ShouldReturnValid_WhenKeyMatchesScheduledEvent`  | `IsValid == true`, `EventId` and `TenantId` populated                               |
| `Handle_ShouldReturnValid_WhenKeyMatchesTestingEvent`    | `IsValid == true`                                                                   |
| `Handle_ShouldReturnValid_WhenKeyMatchesLiveEvent`       | `IsValid == true`                                                                   |
| `Handle_ShouldReturnInvalid_WhenKeyFormatIsInvalid`      | Garbage string fails `StreamKey.TryFromString` → `IsValid == false` without DB call |
| `Handle_ShouldReturnInvalid_WhenKeyNotFoundInRepository` | Repo returns `null` → `IsValid == false`                                            |
| `Handle_ShouldReturnInvalid_WhenEventInDraftStatus`      | Draft event → `IsValid == false`                                                    |
| `Handle_ShouldReturnInvalid_WhenEventInEndedStatus`      | Ended event → `IsValid == false`                                                    |
| `Handle_ShouldReturnInvalid_WhenEventIsArchived`         | Archived event → `IsValid == false`                                                 |

---

### 10.6 — `CreateEventCommandValidatorTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/Validators/CreateEventCommandValidatorTests.cs`

**Dependencies:** Mock `IClock` returning a fixed future time.  
**Requires NuGet:** `FluentValidation.TestHelper` (see 10.8).

**Setup helper:** `BuildValidCommand()` — returns a fully valid `CreateEventCommand` to use as a baseline, mutated per test.

| Test method                                     | Asserts                                            |
| ----------------------------------------------- | -------------------------------------------------- |
| `ShouldHaveError_WhenNameIsEmpty`               | `result.ShouldHaveValidationErrorFor(x => x.Name)` |
| `ShouldHaveError_WhenNameExceedsMaxLength`      | Name > 200 chars fails                             |
| `ShouldHaveError_WhenScheduledStartIsInThePast` | Start before `clock.UtcNow` fails                  |
| `ShouldHaveError_WhenScheduledEndIsBeforeStart` | End before start fails                             |
| `ShouldHaveError_WhenProtocolIsInvalid`         | `"FTP"` → validation error                         |
| `ShouldHaveError_WhenMaxResolutionIsInvalid`    | `999` (not in allowed set) fails                   |
| `ShouldHaveError_WhenPrimaryColorIsInvalidHex`  | `"red"` or `"#GGG"` fails                          |
| `ShouldHaveError_WhenAccessModeIsInvalid`       | `"Unknown"` fails                                  |
| `ShouldNotHaveErrors_WhenAllFieldsAreValid`     | `result.ShouldNotHaveAnyValidationErrors()`        |

---

### 10.7 — `SetEventScheduleCommandValidatorTests.cs`

**File:** `tests/TerraStream.EventService.UnitTests/Application/Validators/SetEventScheduleCommandValidatorTests.cs`

**Dependencies:** Mock `IClock`.

| Test method                                     | Asserts                |
| ----------------------------------------------- | ---------------------- |
| `ShouldHaveError_WhenScheduledStartIsInThePast` | Past start fails       |
| `ShouldHaveError_WhenScheduledEndIsBeforeStart` | End before start fails |
| `ShouldHaveError_WhenTimezoneIsEmpty`           | Empty string fails     |
| `ShouldHaveError_WhenTimezoneIsInvalid`         | `"NotARealZone"` fails |
| `ShouldHaveError_WhenLobbyMinutesIsNegative`    | `-1` fails             |
| `ShouldHaveError_WhenLobbyMinutesExceedsMax`    | `121` fails            |
| `ShouldNotHaveErrors_WhenAllFieldsAreValid`     | Valid command passes   |

---

### 10.8 — Add `FluentValidation.TestHelper` NuGet package

**File:** `tests/TerraStream.EventService.UnitTests/TerraStream.EventService.UnitTests.csproj`

Add:

```xml
<PackageReference Include="FluentValidation.TestHelper" Version="11.*" />
```

This enables the `TestValidate()` extension and `ShouldHaveValidationErrorFor` / `ShouldNotHaveAnyValidationErrors()` Shouldly-compatible assertions.

---

## Task 11 — Integration Tests

### 11.1 — Full 7-step lifecycle test

**File:** `tests/TerraStream.EventService.IntegrationTests/EventsControllerLifecycleTests.cs` (add test)

The existing `FullLifecycle_Create_Publish_GoLive_End_Archive` skips intermediate PUT steps. Add:

**`FullLifecycle_WithIntermediateUpdates_AllStepsSucceed`**

Steps:

1. `POST /api/v1/events` → 201, capture `id`
2. `PUT /api/v1/events/{id}/details` → 200
3. `PUT /api/v1/events/{id}/schedule` → 200
4. `PUT /api/v1/events/{id}/theme` → 200
5. `GET /api/v1/events/{id}` → assert `Status == "Draft"`, updated `Name` and `Description`
6. `POST /api/v1/events/{id}/publish` → 200, assert `Status == "Scheduled"`
7. `POST /api/v1/events/{id}/go-live` → 200, assert `Status == "Live"`
8. `POST /api/v1/events/{id}/end` → 200, assert `Status == "Ended"`
9. `POST /api/v1/events/{id}/archive` → 200, assert `Status == "Archived"`

---

### 11.2 — Schedule validation on update

**File:** `tests/TerraStream.EventService.IntegrationTests/EventsControllerValidationTests.cs` (new file, extends `ApiTestBase`)

Since `CreateEventCommand` requires a schedule at creation, the "publish without schedule" scenario from the plan is adapted:

| Test method                                      | Asserts                                               |
| ------------------------------------------------ | ----------------------------------------------------- |
| `PutSchedule_WithStartInPast_ShouldReturn400`    | `PUT /{id}/schedule` with past `ScheduledStart` → 400 |
| `PutSchedule_WithEndBeforeStart_ShouldReturn400` | End before start → 400                                |
| `Create_WithInvalidProtocol_ShouldReturn400`     | `POST /` with `Protocol = "FTP"` → 400                |
| `Create_WithDescriptionTooLong_ShouldReturn400`  | Description > 2000 chars → 400                        |

---

### 11.3 — Stream key validation for Live and Ended states

**File:** `tests/TerraStream.EventService.IntegrationTests/StreamKeyValidationTests.cs` (add 2 tests)

The existing tests cover Scheduled→valid and Draft→invalid. Add:

| Test method                                           | Setup                           | Asserts                               |
| ----------------------------------------------------- | ------------------------------- | ------------------------------------- |
| `ValidateStreamKey_ForLiveEvent_ShouldReturnValid`    | Create → Publish → GoLive       | `IsValid == true`                     |
| `ValidateStreamKey_ForEndedEvent_ShouldReturnInvalid` | Create → Publish → GoLive → End | `IsValid == false` with reject reason |

---

### 11.4 — Integration event publishing tests

**File:** `tests/TerraStream.EventService.IntegrationTests/IntegrationEventTests.cs` (new file, extends `ApiTestBase`)

Uses `Fixture.TestHarness.Published.Select<TMessage>()` provided by the in-memory MassTransit test harness.

| Test method                                                 | Setup                                | Asserts                                                                                 |
| ----------------------------------------------------------- | ------------------------------------ | --------------------------------------------------------------------------------------- |
| `PublishEvent_ShouldPublish_EventScheduledIntegrationEvent` | Create → POST publish                | Harness has one `EventScheduledIntegrationEvent` with matching `EventId` and `TenantId` |
| `GoLive_ShouldPublish_EventWentLiveIntegrationEvent`        | Create → Publish → POST go-live      | Harness has `EventWentLiveIntegrationEvent`                                             |
| `EndStream_ShouldPublish_EventEndedIntegrationEvent`        | Create → Publish → GoLive → POST end | Harness has `EventEndedIntegrationEvent`                                                |
| `CancelEvent_ShouldPublish_EventCancelledIntegrationEvent`  | Create → POST cancel                 | Harness has `EventCancelledIntegrationEvent`                                            |

---

## New and Modified Files

| #   | File                                                                                  | Action               |
| --- | ------------------------------------------------------------------------------------- | -------------------- |
| 1   | `tests/.../UnitTests/Application/CreateEventCommandHandlerTests.cs`                   | Create               |
| 2   | `tests/.../UnitTests/Application/PublishEventCommandHandlerTests.cs`                  | Create               |
| 3   | `tests/.../UnitTests/Application/GoLiveCommandHandlerTests.cs`                        | Create               |
| 4   | `tests/.../UnitTests/Application/CancelEventCommandHandlerTests.cs`                   | Create               |
| 5   | `tests/.../UnitTests/Application/ValidateStreamKeyQueryHandlerTests.cs`               | Create               |
| 6   | `tests/.../UnitTests/Application/Validators/CreateEventCommandValidatorTests.cs`      | Create               |
| 7   | `tests/.../UnitTests/Application/Validators/SetEventScheduleCommandValidatorTests.cs` | Create               |
| 8   | `tests/.../UnitTests/TerraStream.EventService.UnitTests.csproj`                       | Modify (add NuGet)   |
| 9   | `tests/.../IntegrationTests/EventsControllerLifecycleTests.cs`                        | Modify (add 1 test)  |
| 10  | `tests/.../IntegrationTests/EventsControllerValidationTests.cs`                       | Create               |
| 11  | `tests/.../IntegrationTests/StreamKeyValidationTests.cs`                              | Modify (add 2 tests) |
| 12  | `tests/.../IntegrationTests/IntegrationEventTests.cs`                                 | Create               |

---

## Conventions

- **Assertions:** Use `Shouldly` consistently (`ShouldBe`, `ShouldThrow`, `ShouldNotBeNull`, etc.) — matches the majority of existing test files
- **Validator assertions:** Use `FluentValidation.TestHelper` (`TestValidate()`, `ShouldHaveValidationErrorFor`, `ShouldNotHaveAnyValidationErrors`)
- **Naming:** `{Method}_{Scenario}_{ExpectedResult}` (matches existing pattern)
- **Test traits:** Apply `[Trait("Category", "Integration")]` on integration test classes (already set via `ApiTestBase`)
- **Stream key validation returns 200 with `IsValid: false`** — this is the implemented behavior, not HTTP 403 as the original plan suggested; tests match the implementation
- **No fake JWT auth handler needed** — the `Testing` environment resolves tenant from the `X-Tenant-Id` header directly; no JWT enforcement in tests

## Verification

```
dotnet test tests/TerraStream.EventService.UnitTests/ --verbosity normal
dotnet test tests/TerraStream.EventService.IntegrationTests/ --verbosity normal
```

All new tests should pass. Existing tests must have no regressions.

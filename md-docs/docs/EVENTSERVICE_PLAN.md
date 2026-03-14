---
name: EventService Implementation Plan
overview: Implementation plan for the TerraStream EventService -- the core domain microservice owning event lifecycle, stream key generation, and OBS profile generation. REST for external, gRPC for internal, dedicated PostgreSQL, CQRS via MediatR, Clean Architecture.
todos:
  - id: scaffold-solution
    content: Scaffold .sln, all 7 projects, and docker-compose.yml
    status: pending
  - id: domain-layer
    content: "Implement Domain layer: Event aggregate, owned entities, enums, state machine, domain events"
    status: pending
  - id: application-contracts
    content: "Implement Application layer: commands, queries, validators, handler interfaces"
    status: pending
  - id: application-handlers
    content: Implement all command and query handlers with MediatR
    status: pending
  - id: infrastructure-dbcontext
    content: "Implement Infrastructure: EventDbContext, EF Core config, migrations, repository"
    status: pending
  - id: infrastructure-messaging
    content: Implement MassTransit integration event publishing
    status: pending
  - id: infrastructure-tenant
    content: Implement ITenantContext with HTTP header and gRPC metadata resolution
    status: pending
  - id: api-rest
    content: Implement REST controllers for all endpoints
    status: pending
  - id: api-grpc
    content: Implement gRPC proto file and service with MediatR dispatch
    status: pending
  - id: api-startup
    content: "Wire up Program.cs: DI, EF Core, MassTransit, gRPC, Swagger, health checks"
    status: pending
  - id: obs-profile
    content: Implement OBS profile JSON generation logic
    status: pending
  - id: unit-tests
    content: "Write unit tests: domain state machine, all command handlers, validators"
    status: pending
  - id: integration-tests
    content: "Write integration tests: Testcontainers Postgres, WebApplicationFactory, full API roundtrips"
    status: pending
  - id: docker-ci
    content: Create Dockerfile and verify docker-compose local dev flow
    status: pending
---

# EventService Implementation Plan

---

## Overview

The EventService is the core domain service of TerraStream. It owns the event lifecycle (CRUD, state machine), stream key generation, and OBS profile generation. It exposes a **REST API** for the frontend/gateway and a **gRPC API** for internal service-to-service calls (e.g., AI Assistant, NotificationService querying event state). It uses a **dedicated PostgreSQL** instance, **CQRS via MediatR**, and follows Clean Architecture.

---

## Solution Structure

```
terrastream-event-service/
  src/
    TerraStream.EventService.Domain/          # Entities, value objects, enums, domain events
    TerraStream.EventService.Application/     # Commands, queries, handlers, validators, interfaces
    TerraStream.EventService.Infrastructure/  # EF Core DbContext, repos, message bus publishing
    TerraStream.EventService.Api/             # REST controllers, gRPC services, Program.cs
    TerraStream.EventService.Contracts/       # Shared Protobuf files and integration event DTOs
  tests/
    TerraStream.EventService.UnitTests/       # Moq-based unit tests for handlers and domain
    TerraStream.EventService.IntegrationTests/ # Testcontainers-based tests against real Postgres + API
  docker-compose.yml                          # Local dev: Postgres + RabbitMQ
  Dockerfile
  terrastream-event-service.sln
```

---

## Domain Model

### Event Aggregate (Domain layer)

```
Event (Aggregate Root)
  - Id: Guid
  - TenantId: Guid
  - Name: string
  - Description: string?
  - Status: EventStatus (enum: Draft, Scheduled, Testing, Live, Ended, Archived, Cancelled)
  - CreatedAt: DateTime
  - UpdatedAt: DateTime

EventTheme (owned entity)
  - PrimaryColor: string
  - SecondaryColor: string?
  - AccentColor: string?
  - LogoAssetId: Guid?
  - WatermarkAssetId: Guid?
  - PosterAssetId: Guid?
  - HidePlatformBranding: bool

EventSchedule (owned entity)
  - ScheduledStart: DateTimeOffset
  - ScheduledEnd: DateTimeOffset
  - Timezone: string
  - LobbyMinutesBefore: int (default: 15)

EventStreamConfig (owned entity)
  - StreamKey: string (auto-generated, unique)
  - IngestEndpoint: string (assigned by system)
  - Protocol: StreamProtocol (enum: RTMP, SRT)
  - MaxResolution: int (1080)
  - MaxBitrateKbps: int (5000)
  - RecordingEnabled: bool (default: true)

EventAccessConfig (owned entity)
  - AccessMode: AccessMode (enum: Public, InviteOnly, IdpRestricted)
  - MaxConcurrentViewers: int
  - RequireAuthentication: bool

EventFeature (collection)
  - Id: Guid
  - FeatureType: string (QnA, Chat, Reactions, Analytics)
  - Enabled: bool

EventInvitation (collection)
  - Id: Guid
  - Email: string
  - Status: InvitationStatus (enum: Pending, Sent, Failed)
  - SentAt: DateTime?
```

The `Event` entity encapsulates state transitions. The `TransitionTo(EventStatus)` method enforces the state machine rules and raises domain events (`EventScheduled`, `EventWentLive`, `EventEnded`, etc.).

---

## CQRS: Commands and Queries

### Commands (write side, via MediatR `IRequest<T>`)

| Command | Description |

|---|---|

| `CreateEventCommand` | Creates event in Draft status. Auto-generates StreamKey. |

| `UpdateEventDetailsCommand` | Updates name, description. |

| `SetEventThemeCommand` | Sets branding/theme on event. |

| `SetEventScheduleCommand` | Sets start/end time, timezone, lobby. |

| `ConfigureEventFeaturesCommand` | Enables/disables Q&A, Chat, Reactions, Analytics. |

| `SetEventAccessCommand` | Sets access mode, max viewers, auth requirement. |

| `InviteUsersCommand` | Adds invitations to event. |

| `PublishEventCommand` | Transitions Draft -> Scheduled. Publishes `EventScheduled` integration event. |

| `StartTestStreamCommand` | Transitions Scheduled -> Testing. |

| `StopTestStreamCommand` | Transitions Testing -> Scheduled. |

| `GoLiveCommand` | Transitions Scheduled/Testing -> Live. Publishes `EventWentLive`. |

| `EndStreamCommand` | Transitions Live -> Ended. Publishes `EventEnded`. |

| `ArchiveEventCommand` | Transitions Ended -> Archived. |

| `CancelEventCommand` | Transitions Draft/Scheduled -> Cancelled. Publishes `EventCancelled`. |

| `ValidateStreamKeyCommand` | Called by SRS on_publish. Returns accept/reject. |

### Queries (read side, via MediatR `IRequest<T>`)

| Query | Description |

|---|---|

| `GetEventByIdQuery` | Full event details including all owned entities. |

| `ListEventsQuery` | Paginated list for tenant. Filters: status, date range, search. |

| `GetEventStreamConfigQuery` | Returns stream key + ingest endpoint (for OBS profile). |

| `GetObsProfileQuery` | Returns generated OBS profile JSON for download. |

| `GetEventsByStatusQuery` | Internal: events by status (used by ResourcePlanner via gRPC). |

---

## APIs

### REST API (external -- consumed by frontend via API Gateway)

Base path: `/api/v1/events`

| Method | Endpoint | Handler |

|---|---|---|

| POST | `/` | CreateEventCommand |

| GET | `/{id}` | GetEventByIdQuery |

| GET | `/` | ListEventsQuery (query params: page, size, status, search) |

| PUT | `/{id}/details` | UpdateEventDetailsCommand |

| PUT | `/{id}/theme` | SetEventThemeCommand |

| PUT | `/{id}/schedule` | SetEventScheduleCommand |

| PUT | `/{id}/features` | ConfigureEventFeaturesCommand |

| PUT | `/{id}/access` | SetEventAccessCommand |

| POST | `/{id}/invitations` | InviteUsersCommand |

| POST | `/{id}/publish` | PublishEventCommand |

| POST | `/{id}/test-stream/start` | StartTestStreamCommand |

| POST | `/{id}/test-stream/stop` | StopTestStreamCommand |

| POST | `/{id}/go-live` | GoLiveCommand |

| POST | `/{id}/end` | EndStreamCommand |

| POST | `/{id}/archive` | ArchiveEventCommand |

| POST | `/{id}/cancel` | CancelEventCommand |

| GET | `/{id}/obs-profile` | GetObsProfileQuery (returns .json file) |

| POST | `/stream-key/validate` | ValidateStreamKeyCommand (called by SRS) |

### gRPC API (internal -- consumed by other services)

Proto file: `Contracts/Protos/event_service.proto`

```protobuf
service EventGrpcService {
  rpc GetEvent (GetEventRequest) returns (EventResponse);
  rpc GetEventsByStatus (GetEventsByStatusRequest) returns (EventListResponse);
  rpc GetEventStreamConfig (GetStreamConfigRequest) returns (StreamConfigResponse);
  rpc ValidateStreamKey (ValidateStreamKeyRequest) returns (ValidateStreamKeyResponse);
}
```

The gRPC service implementations are thin wrappers that dispatch to the same MediatR handlers as the REST controllers.

---

## Infrastructure

### Database (EF Core + PostgreSQL)

- Dedicated PostgreSQL instance for EventService.
- `EventDbContext` with `DbSet<Event>` as the aggregate root.
- Owned entity types for Theme, Schedule, StreamConfig, AccessConfig (mapped as owned types in EF Core, stored in the same table or split tables).
- `EventFeature` and `EventInvitation` as separate tables with FK to Event.
- `TenantId` column on Event table. **Global query filter** in EF Core: `.HasQueryFilter(e => e.TenantId == _tenantContext.TenantId)` for automatic tenant scoping.
- Migrations via `dotnet ef migrations`.

### Message Bus (MassTransit + RabbitMQ)

- Integration events published after successful command handling:
        - `EventScheduledIntegrationEvent { EventId, TenantId, ScheduledStart, ExpectedViewers }`
        - `EventWentLiveIntegrationEvent { EventId, TenantId, StreamKey, IngestEndpoint }`
        - `EventEndedIntegrationEvent { EventId, TenantId, Duration }`
        - `EventCancelledIntegrationEvent { EventId, TenantId }`
- Published via `IPublishEndpoint` from MassTransit, injected into command handlers.

### Tenant Context

- `ITenantContext` interface with `TenantId` property.
- Resolved from the `X-Tenant-Id` HTTP header (set by API Gateway after JWT validation) or from gRPC metadata.
- Registered as a scoped service.

---

## Testing Strategy

### Unit Tests (`TerraStream.EventService.UnitTests`)

- **Framework:** xUnit + Moq + FluentAssertions
- **What to test:**
        - Each command handler: mock `IEventRepository`, `IPublishEndpoint`, `ITenantContext`. Verify correct state transitions, validation failures, and published events.
        - Domain entity methods: `Event.TransitionTo()` state machine rules. Verify allowed and disallowed transitions.
        - `ValidateStreamKeyCommand` handler: verify accept/reject logic.
        - FluentValidation validators for each command.
- **Naming convention:** `{MethodUnderTest}_Should_{ExpectedResult}_When_{Condition}`

### Integration Tests (`TerraStream.EventService.IntegrationTests`)

- **Framework:** xUnit + Testcontainers (PostgreSQL container) + `WebApplicationFactory<Program>` + Respawn (DB reset between tests)
- **What to test:**
        - Full REST API roundtrips: create event -> update details -> set theme -> publish -> go live -> end -> archive. Assert HTTP status codes and response bodies at each step.
        - Invalid state transitions return 400/409 (e.g., go-live on a Draft event).
        - Tenant isolation: create events under tenant A, assert they are invisible to tenant B.
        - `ValidateStreamKeyCommand` endpoint: valid key returns 200, unknown key returns 403.
        - OBS profile download returns valid JSON with correct stream key and server URL.
        - Pagination and filtering on `ListEventsQuery`.
        - gRPC endpoints: `GetEvent`, `GetEventsByStatus`, `ValidateStreamKey`.
- **Test base class:** `EventServiceIntegrationTestBase` sets up `WebApplicationFactory` with Testcontainers PostgreSQL, applies migrations on startup, and runs Respawn between tests.
- **Auth simulation:** Replace Keycloak JWT validation with a test authentication handler that accepts a fake JWT with configurable `tenant_id` and `roles` claims.

---

## Key NuGet Packages

| Package | Project | Purpose |

|---|---|---|

| `MediatR` | Application | CQRS command/query dispatch |

| `FluentValidation` | Application | Command input validation |

| `MassTransit` + `MassTransit.RabbitMQ` | Infrastructure | Integration event publishing |

| `Npgsql.EntityFrameworkCore.PostgreSQL` | Infrastructure | EF Core PostgreSQL provider |

| `Grpc.AspNetCore` | Api | gRPC server hosting |

| `Google.Protobuf` + `Grpc.Tools` | Contracts | Protobuf code generation |

| `Swashbuckle.AspNetCore` | Api | Swagger/OpenAPI docs |

| `AspNetCore.HealthChecks.NpgSql` | Api | PostgreSQL health check |

| `xUnit` | Tests | Test framework |

| `Moq` | UnitTests | Mocking |

| `FluentAssertions` | Tests | Assertion helpers |

| `Testcontainers.PostgreSql` | IntegrationTests | Disposable Postgres container |

| `Respawn` | IntegrationTests | DB reset between tests |

| `Microsoft.AspNetCore.Mvc.Testing` | IntegrationTests | `WebApplicationFactory` |

---

## Implementation Tasks (ordered for a dev agent)

These tasks should be executed in order. Each task builds on the previous.

### Task 1: Scaffold Solution and Projects

Create the solution file and all 7 projects with correct project references:

```
Domain        -> (no dependencies)
Contracts     -> (no dependencies)
Application   -> Domain
Infrastructure -> Application, Domain, Contracts
Api           -> Application, Infrastructure, Contracts
UnitTests     -> Application, Domain
IntegrationTests -> Api
```

Create `docker-compose.yml` with PostgreSQL 16 (port 5432, db: `eventservice_db`) and RabbitMQ 3 (port 5672/15672). Add a `.editorconfig` and `Directory.Build.props` with nullable enabled, implicit usings, and `net8.0` target.

### Task 2: Domain Layer

In `TerraStream.EventService.Domain`:

1. Create enums: `EventStatus`, `StreamProtocol`, `AccessMode`, `InvitationStatus`.
2. Create the `Event` aggregate root class with all properties from the domain model above. Constructor should set `Status = Draft`, generate `Id = Guid.NewGuid()`, and set `CreatedAt`.
3. Create owned entity classes: `EventTheme`, `EventSchedule`, `EventStreamConfig`, `EventAccessConfig`.
4. Create entity classes: `EventFeature`, `EventInvitation`.
5. Implement `Event.TransitionTo(EventStatus target)` method that enforces the state machine:

        - Draft -> Scheduled, Cancelled
        - Scheduled -> Testing, Live, Cancelled
        - Testing -> Scheduled, Live
        - Live -> Ended
        - Ended -> Archived
        - Throws `InvalidOperationException` for invalid transitions.

6. Implement `Event.GenerateStreamKey()` that creates a unique key in format `evt_{Id.First8Chars}_sk_{RandomHex16}`.
7. Create domain event marker classes: `EventScheduledDomainEvent`, `EventWentLiveDomainEvent`, `EventEndedDomainEvent`, `EventCancelledDomainEvent` -- each carrying the `Event` reference.
8. Create `IEventRepository` interface: `GetByIdAsync`, `ListAsync(filters)`, `AddAsync`, `UpdateAsync`, `GetByStreamKeyAsync`.

### Task 3: Application Layer -- Contracts and Commands

In `TerraStream.EventService.Application`:

1. Create `ITenantContext` interface with `Guid TenantId { get; }`.
2. Create all command record classes (see CQRS table above). Each implements `IRequest<T>` with appropriate result types.
3. Create all query record classes implementing `IRequest<T>`.
4. Create FluentValidation validators for each command (e.g., `CreateEventCommandValidator` -- Name required, max 200 chars; `SetEventScheduleCommandValidator` -- ScheduledStart must be in the future).
5. Create response/result DTOs: `EventDto`, `EventListDto`, `ObsProfileDto`, `StreamKeyValidationResult`.

In `TerraStream.EventService.Contracts`:

1. Create `Protos/event_service.proto` with the gRPC service definition and message types.
2. Create integration event record classes: `EventScheduledIntegrationEvent`, `EventWentLiveIntegrationEvent`, `EventEndedIntegrationEvent`, `EventCancelledIntegrationEvent`.

### Task 4: Application Layer -- Handlers

In `TerraStream.EventService.Application`:

1. Implement `CreateEventCommandHandler`: create Event via factory, generate stream key, persist via repo, return EventDto.
2. Implement `UpdateEventDetailsCommandHandler`: load event, update name/description, persist.
3. Implement `SetEventThemeCommandHandler`, `SetEventScheduleCommandHandler`, `ConfigureEventFeaturesCommandHandler`, `SetEventAccessCommandHandler`: load event, update respective owned entity, persist.
4. Implement `InviteUsersCommandHandler`: load event, add EventInvitation entries with status Pending, persist.
5. Implement state transition handlers (`PublishEventCommandHandler`, `StartTestStreamCommandHandler`, `StopTestStreamCommandHandler`, `GoLiveCommandHandler`, `EndStreamCommandHandler`, `ArchiveEventCommandHandler`, `CancelEventCommandHandler`): load event, call `TransitionTo()`, persist, publish integration event via `IPublishEndpoint`.
6. Implement `ValidateStreamKeyCommandHandler`: query repo by stream key, check event status is Scheduled/Testing/Live, return accept/reject.
7. Implement query handlers: `GetEventByIdQueryHandler`, `ListEventsQueryHandler` (with pagination), `GetEventStreamConfigQueryHandler`, `GetObsProfileQueryHandler` (generates OBS JSON), `GetEventsByStatusQueryHandler`.
8. Register a MediatR pipeline behavior for FluentValidation: `ValidationBehavior<TRequest, TResponse>` that runs validators before the handler.

### Task 5: Infrastructure Layer

In `TerraStream.EventService.Infrastructure`:

1. Create `EventDbContext` extending `DbContext`. Configure `Event` as aggregate root with owned types (Theme, Schedule, StreamConfig, AccessConfig). Configure `EventFeature` and `EventInvitation` as separate tables. Add global query filter for `TenantId`.
2. Create `EventRepository` implementing `IEventRepository` using `EventDbContext`.
3. Create `TenantContext` implementing `ITenantContext`, reading from `IHttpContextAccessor` header `X-Tenant-Id`.
4. Register MassTransit with RabbitMQ in a service extension method. Configure integration event publishing.
5. Generate initial EF Core migration.

### Task 6: API Layer -- REST

In `TerraStream.EventService.Api`:

1. Create `EventsController` inheriting `ControllerBase` with `[ApiController]` and `[Route("api/v1/events")]`. Inject `IMediator`. Each action method maps an HTTP endpoint to a MediatR send/request call (see REST API table above).
2. The OBS profile endpoint should return `Content-Type: application/json` with `Content-Disposition: attachment; filename=obs-profile-{eventId}.json`.
3. The stream-key validate endpoint should be unauthenticated (called by SRS internally).
4. Add global exception handling middleware: map `InvalidOperationException` (bad state transition) to 409 Conflict, `ValidationException` to 400 Bad Request, `KeyNotFoundException` to 404 Not Found.

### Task 7: API Layer -- gRPC

In `TerraStream.EventService.Api`:

1. Create `EventGrpcServiceImpl` extending the generated gRPC base class. Each method extracts `tenant_id` from gRPC call metadata, sets `ITenantContext`, and dispatches to MediatR.
2. Register gRPC in `Program.cs` with `app.MapGrpcService<EventGrpcServiceImpl>()`.

### Task 8: Program.cs Wiring

In `TerraStream.EventService.Api/Program.cs`:

1. Register all DI: DbContext (with Npgsql connection string from config), repositories, MediatR (scan Application assembly), FluentValidation validators, MassTransit, TenantContext, gRPC.
2. Add Swagger via Swashbuckle.
3. Add health checks: `/health/live` (basic) and `/health/ready` (Postgres connectivity).
4. Add CORS policy for development.
5. Map REST controllers and gRPC services.
6. Apply EF Core migrations on startup in development mode.

### Task 9: OBS Profile Generation

In the `GetObsProfileQueryHandler`:

1. Load event stream config (stream key, ingest endpoint, max resolution, max bitrate).
2. Build a JSON object matching OBS Studio's importable profile format (see Section 12 of the architecture plan for the exact structure).
3. Map max resolution to `output_res` string (e.g., 1080 -> "1920x1080"), max bitrate to `bitrate` integer.
4. Return as a downloadable JSON response.

### Task 10: Unit Tests

In `TerraStream.EventService.UnitTests`:

1. **Domain tests:** Test `Event.TransitionTo()` for every valid transition (7 cases) and every invalid transition (e.g., Draft -> Live, Ended -> Live). Test `GenerateStreamKey()` produces correct format.
2. **Handler tests:** For each command handler, mock `IEventRepository` and `IPublishEndpoint`. Test:

        - `CreateEventCommandHandler`: creates event in Draft status with a generated stream key.
        - `PublishEventCommandHandler`: transitions to Scheduled, publishes `EventScheduledIntegrationEvent`.
        - `GoLiveCommandHandler`: transitions to Live from both Scheduled and Testing states.
        - `CancelEventCommandHandler`: works from Draft and Scheduled, fails from Live.
        - `ValidateStreamKeyCommandHandler`: returns true for valid key on a Live event, false for unknown key, false for an Ended event.

3. **Validator tests:** Test that `CreateEventCommandValidator` rejects empty name, that `SetEventScheduleCommandValidator` rejects past start times.

### Task 11: Integration Tests

In `TerraStream.EventService.IntegrationTests`:

1. Create `EventServiceWebAppFactory` extending `WebApplicationFactory<Program>`. Override `ConfigureWebHost` to:

        - Replace the PostgreSQL connection string with the Testcontainers-provided one.
        - Replace MassTransit with the in-memory test harness (`AddMassTransitTestHarness`).
        - Add a fake authentication handler that accepts a test JWT.

2. Create `IntegrationTestBase` class implementing `IAsyncLifetime`. On `InitializeAsync`: start Testcontainers PostgreSQL, create the factory, apply migrations, create an `HttpClient`. On `DisposeAsync`: dispose factory and container. Use Respawn to reset DB between tests.
3. Write test classes:

        - `EventLifecycleTests`: POST create -> PUT details -> PUT schedule -> POST publish -> POST go-live -> POST end -> POST archive. Assert 200/201 at each step and correct status in GET.
        - `EventValidationTests`: POST create with empty name -> assert 400. POST publish on event without schedule -> assert 400/409.
        - `TenantIsolationTests`: Create event as tenant A, GET as tenant B -> assert 404.
        - `StreamKeyValidationTests`: POST validate with correct key -> 200. POST with unknown key -> 403.
        - `ObsProfileTests`: GET obs-profile -> assert JSON contains correct stream key and server URL.

### Task 12: Dockerfile and Docker Compose

1. Create a multi-stage `Dockerfile`: build stage with `mcr.microsoft.com/dotnet/sdk:8.0`, publish stage with `mcr.microsoft.com/dotnet/aspnet:8.0`. Expose ports 8080 (HTTP/REST) and 8081 (gRPC).
2. Update `docker-compose.yml` to include the EventService container alongside Postgres and RabbitMQ, with correct environment variables for connection strings.
3. Verify `docker-compose up` starts all containers and the service responds to health checks.
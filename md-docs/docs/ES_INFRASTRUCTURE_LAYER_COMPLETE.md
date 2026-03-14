# TerraStream EventService — Infrastructure Layer Implementation

| Field                 | Value                                                  |
| --------------------- | ------------------------------------------------------ |
| **Status**            | ✅ Complete                                            |
| **Date**              | 2026-02-20                                             |
| **Build**             | 0 warnings, 0 errors (`TreatWarningsAsErrors` enabled) |
| **Unit Tests**        | 152 passing (no regressions)                           |
| **Integration Tests** | 14 passing (Testcontainers PostgreSQL)                 |

---

## Overview

Sub-task 5 of the EventService plan — the Infrastructure Layer — provides EF Core persistence against PostgreSQL, MassTransit messaging with a transactional outbox, multi-tenant resolution, and a full integration test scaffold. All 14 planned steps have been implemented.

---

## Architecture

```
Infrastructure/
├── DependencyInjection.cs                       # AddInfrastructure extension
├── Messaging/
│   └── DomainEventMapper.cs                     # Domain → Integration event mapping
├── Persistence/
│   ├── EventDbContext.cs                        # DbContext + MassTransit outbox entities
│   ├── EventDbContextFactory.cs                 # Design-time factory for dotnet-ef
│   ├── UnitOfWork.cs                            # Transactional outbox pattern
│   ├── Configurations/
│   │   ├── EventConfiguration.cs                # Event aggregate mapping
│   │   ├── EventFeatureConfiguration.cs         # EventFeature table mapping
│   │   └── EventInvitationConfiguration.cs      # EventInvitation table mapping
│   ├── Migrations/
│   │   └── 20260220141845_InitialCreate.cs      # Initial schema + outbox tables
│   └── Repositories/
│       └── EventRepository.cs                   # IEventRepository implementation
└── Services/
    ├── SystemClock.cs                           # IClock → DateTimeOffset.UtcNow
    └── TenantContext.cs                         # ITenantContext → X-Tenant-Id header
```

**Cross-layer additions:**

| File                                                      | Layer       | Change                                                             |
| --------------------------------------------------------- | ----------- | ------------------------------------------------------------------ |
| `Application/DependencyInjection.cs`                      | Application | `AddApplication()` — Mediator (scoped) + FluentValidation          |
| `Application/Behaviors/UnitOfWorkBehavior.cs`             | Application | Pipeline behavior: SaveChanges after commands                      |
| `Application/TerraStream.EventService.Application.csproj` | Application | `Mediator_ServiceLifetime=Scoped` MSBuild property                 |
| `Api/Program.cs`                                          | API         | Calls `AddApplication()` + `AddInfrastructure()` + options binding |
| `Api/appsettings.json`                                    | API         | Connection string placeholders + Streaming section                 |
| `Api/appsettings.Development.json`                        | API         | Dev connection strings (localhost PostgreSQL / RabbitMQ)           |
| `Api/TerraStream.EventService.Api.csproj`                 | API         | Added `Microsoft.EntityFrameworkCore.Design`                       |
| `.config/dotnet-tools.json`                               | Tooling     | `dotnet-ef` 8.0.11 local tool                                      |

---

## Summary Statistics

| Metric                   | Value                                    |
| ------------------------ | ---------------------------------------- |
| Infrastructure .cs files | 11 (excl. migrations)                    |
| Entity configurations    | 3 (Event, EventFeature, EventInvitation) |
| Database tables          | 6 (3 domain + 3 MassTransit outbox)      |
| Indexes                  | 10 (4 domain + 6 outbox)                 |
| Integration test classes | 3                                        |
| Integration test methods | 14                                       |
| NuGet packages added     | 9 (infra: 6, test: 5, api: 1)            |

---

## Implementation Breakdown

### 1. EF Core Persistence

**EventDbContext** — Configures `DbSet<Event>` with `ApplyConfigurationsFromAssembly` and MassTransit outbox entities (`InboxState`, `OutboxMessage`, `OutboxState`). Uses a `using Event = Domain.Aggregates.Event` alias to resolve the name collision with `MassTransit.Event`.

**Entity configurations:**

- **EventConfiguration** — Maps the aggregate root to `event."Events"`. Value conversions for `EventId` (↔ Guid) and `StreamKey` (↔ string). All enums stored as strings. Four owned entities (Theme, Schedule, StreamConfig, AccessConfig) with column prefixes. Collection navigations (Features, Invitations) via backing fields. Global query filter on `IsDeleted`. Concurrency token on `Version`. Unique index on `StreamConfig_StreamKey`.
- **EventFeatureConfiguration** — Table `event."EventFeatures"`, `Config` as `jsonb`, FK cascade to Events.
- **EventInvitationConfiguration** — Table `event."EventInvitations"`, `Status` as string, index on Email, FK cascade to Events.

**EventDbContextFactory** — `IDesignTimeDbContextFactory<EventDbContext>` for `dotnet ef` migrations tooling.

### 2. Repository

**EventRepository** — Implements `IEventRepository` with eager loading of Features and Invitations. Supports:

- Tenant-scoped CRUD (`GetByIdAsync`, `AddAsync`, `UpdateAsync`, `DeleteAsync`, `ExistsAsync`)
- Cross-tenant `GetByStreamKeyAsync` using `IgnoreQueryFilters()` + manual `!IsDeleted` check
- `ListAsync` with `EF.Functions.ILike` text search, configurable sorting (name/scheduledstart/updatedat/status/createdat), Skip/Take pagination
- `DeleteAsync` calls `SoftDelete()` on the aggregate

### 3. Transactional Outbox

**UnitOfWork** — Implements `IUnitOfWork`. On `SaveChangesAsync`:

1. Collects domain events from all tracked `Event` aggregates via `ChangeTracker`
2. Maps each to an integration event via `DomainEventMapper`
3. Publishes via the outbox-aware `IPublishEndpoint` (writes to `OutboxMessage` table)
4. Calls `DbContext.SaveChangesAsync()` — atomically persists the aggregate state + outbox messages
5. Clears domain events from the aggregates

**UnitOfWorkBehavior** (Application layer) — `IPipelineBehavior<TMessage, TResponse>` that calls `IUnitOfWork.SaveChangesAsync()` after the handler completes, but only for command messages (namespace convention: `typeof(TMessage).Namespace.Contains("Commands")`).

### 4. Domain Event Mapping

**DomainEventMapper** — Maps domain events to integration events with aggregate enrichment:

| Domain Event                | Integration Event                | Enrichment                                                                                     |
| --------------------------- | -------------------------------- | ---------------------------------------------------------------------------------------------- |
| `EventScheduledDomainEvent` | `EventScheduledIntegrationEvent` | `ExpectedViewers` ← `AccessConfig.MaxConcurrentViewers ?? 0`                                   |
| `EventWentLiveDomainEvent`  | `EventWentLiveIntegrationEvent`  | `IngestEndpoint` ← `StreamConfig.IngestEndpoint`, `StreamKey` ← `StreamConfig.StreamKey.Value` |
| `EventEndedDomainEvent`     | `EventEndedIntegrationEvent`     | Direct mapping                                                                                 |
| `EventCancelledDomainEvent` | `EventCancelledIntegrationEvent` | Direct mapping                                                                                 |

### 5. Cross-Cutting Services

- **TenantContext** — Resolves `TenantId` from `X-Tenant-Id` HTTP header via `IHttpContextAccessor`. Cached with `Lazy<Guid>`. Throws `InvalidOperationException` on missing/invalid header. Registered scoped.
- **SystemClock** — Returns `DateTimeOffset.UtcNow`. Registered singleton.

### 6. DI Wiring

**Infrastructure `AddInfrastructure(IConfiguration)`** registers:

- `EventDbContext` with Npgsql + `SplitQuery`
- `IEventRepository → EventRepository` (scoped)
- `IUnitOfWork → UnitOfWork` (scoped)
- `ITenantContext → TenantContext` (scoped) + `HttpContextAccessor`
- `IClock → SystemClock` (singleton)
- `DomainEventMapper` (scoped)
- MassTransit with RabbitMQ transport + EF Core outbox (`UsePostgres`, `UseBusOutbox`)

**Application `AddApplication()`** registers:

- Mediator with `ServiceLifetime.Scoped`
- FluentValidation validators from assembly

### 7. Database Migration

**InitialCreate** migration creates:

| Schema   | Table              | Notes                                              |
| -------- | ------------------ | -------------------------------------------------- |
| `event`  | `Events`           | 31 columns (scalar + 4 owned entities), PK on `Id` |
| `event`  | `EventFeatures`    | `Config` as jsonb, FK cascade                      |
| `event`  | `EventInvitations` | FK cascade, index on Email                         |
| `public` | `InboxState`       | MassTransit inbox deduplication                    |
| `public` | `OutboxMessage`    | MassTransit transactional outbox                   |
| `public` | `OutboxState`      | MassTransit outbox delivery tracking               |

### 8. Integration Test Scaffold

**IntegrationTestFixture** (`IAsyncLifetime`) — Starts PostgreSQL 16 via Testcontainers, creates `WebApplicationFactory<Program>` with overridden `EventDbContext` (Testcontainer connection) and `AddMassTransitTestHarness()`, applies migrations, sets up Respawn for database resets.

**IntegrationTestBase** — Abstract base class with `[Trait("Category", "Integration")]`, resets database between tests, provides `CreateValidEvent()` factory and scope helpers.

**Test classes:**

| Class                  | Tests | Coverage                                                                                      |
| ---------------------- | ----- | --------------------------------------------------------------------------------------------- |
| `EventRepositoryTests` | 7     | Add, GetById, GetById (not found), GetByStreamKey, SoftDelete, List pagination, Exists        |
| `TenantIsolationTests` | 4     | Cross-tenant GetById blocked, List isolation, cross-tenant StreamKey lookup, Delete isolation |
| `UnitOfWorkTests`      | 3     | Outbox publish via test harness, domain events cleared, no-op safe                            |

---

## NuGet Packages

### Infrastructure Project

| Package                                 | Version | Purpose                           |
| --------------------------------------- | ------- | --------------------------------- |
| `Microsoft.EntityFrameworkCore`         | 8.0.11  | ORM                               |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | 8.0.11  | PostgreSQL provider               |
| `Microsoft.EntityFrameworkCore.Design`  | 8.0.11  | Migration tooling (PrivateAssets) |
| `MassTransit`                           | 8.2.5   | Messaging abstractions            |
| `MassTransit.RabbitMQ`                  | 8.2.5   | RabbitMQ transport                |
| `MassTransit.EntityFrameworkCore`       | 8.2.5   | EF Core transactional outbox      |

### Integration Tests Project

| Package                            | Version | Purpose                          |
| ---------------------------------- | ------- | -------------------------------- |
| `Testcontainers.PostgreSql`        | 4.2.0   | PostgreSQL Testcontainer         |
| `Respawn`                          | 6.2.1   | Database reset between tests     |
| `Microsoft.AspNetCore.Mvc.Testing` | 8.0.11  | `WebApplicationFactory<Program>` |
| `MassTransit`                      | 8.2.5   | `AddMassTransitTestHarness`      |
| `Shouldly`                         | 4.3.0   | Assertions (project convention)  |

---

## Key Decisions

| Decision                                                | Rationale                                                                                                                                                         |
| ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Global query filter on `IsDeleted` only, not `TenantId` | Cross-tenant `GetByStreamKeyAsync` (SRS callbacks) needs to bypass tenant filtering without removing the soft-delete filter                                       |
| `UnitOfWorkBehavior` in Application layer               | Depends only on `IUnitOfWork` (application abstraction), consistent with `ValidationBehavior` placement                                                           |
| Dedicated `DomainEventMapper` class                     | Separated from `UnitOfWork` for testability — enrichment logic is unit-testable in isolation                                                                      |
| Mediator `ServiceLifetime.Scoped`                       | Source-generated Mediator needs compile-time + runtime agreement on lifetime; scoped is required because handlers consume scoped services (repository, DbContext) |
| `AddApplication()` in Application project               | Source generator must see `AddMediator()` call during its own compilation to generate correct lifetime code                                                       |
| Schema prefix `"event"`                                 | Isolates Event Service tables in a shared PostgreSQL instance                                                                                                     |
| MassTransit scoped `IPublishEndpoint`                   | With `AddEntityFrameworkOutbox`, calls to `Publish()` write to `OutboxMessage` table within the same DbContext transaction                                        |

---

## Verification Checklist

- [x] `dotnet build` — 0 warnings, 0 errors
- [x] `dotnet test` (unit) — 152 passing, 0 regressions
- [x] `dotnet test` (integration) — 14 passing against Testcontainers PostgreSQL
- [x] EF Core migration generates correct schema (6 tables, 10 indexes)
- [x] Transactional outbox publishes integration events atomically with persistence
- [x] Multi-tenant isolation verified (4 dedicated tests)
- [x] Cross-tenant StreamKey lookup works with query filter bypass

---

_Document version: 1.0 — 2026-02-20_

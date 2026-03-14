# Task 5: Infrastructure Layer — Implementation Plan

## Summary

Implement the complete Infrastructure layer for the Event Service: EF Core persistence with PostgreSQL (`EventDbContext`, entity configurations, repository), MassTransit transactional outbox with RabbitMQ, tenant resolution from HTTP headers, and a `SystemClock`. A `UnitOfWorkBehavior` pipeline behavior goes in the Application layer (consistent with `ValidationBehavior`), while a dedicated `DomainEventMapper` in Infrastructure handles domain→integration event conversion with aggregate enrichment. Includes initial EF Core migration and integration test scaffolding using Testcontainers + WebApplicationFactory. All wired via an `AddInfrastructure(IConfiguration)` extension method.

---

## Steps

### Step 1 — Add NuGet Packages to Infrastructure Project

Update `src/TerraStream.EventService.Infrastructure/TerraStream.EventService.Infrastructure.csproj`:

| Package | Version | Purpose |
|---|---|---|
| `Microsoft.EntityFrameworkCore` | 8.0.x | ORM framework |
| `Npgsql.EntityFrameworkCore.PostgreSQL` | 8.0.x | PostgreSQL provider |
| `Microsoft.EntityFrameworkCore.Design` | 8.0.x | Migrations tooling (PrivateAssets=All) |
| `MassTransit` | 8.x | Message bus abstraction |
| `MassTransit.RabbitMQ` | 8.x | RabbitMQ transport |
| `MassTransit.EntityFrameworkCoreIntegration` | 8.x | Transactional outbox with EF Core |
| `Microsoft.AspNetCore.Http.Abstractions` | — | `IHttpContextAccessor` for TenantContext |

---

### Step 2 — Create `EventDbContext`

**File:** `src/TerraStream.EventService.Infrastructure/Persistence/EventDbContext.cs`

- Extend `DbContext`
- Single `DbSet<Event> Events`
- Override `OnModelCreating`:
  - Apply all `IEntityTypeConfiguration` from the assembly via `modelBuilder.ApplyConfigurationsFromAssembly(...)`
  - Add MassTransit outbox entities: `modelBuilder.AddInboxStateEntity()`, `modelBuilder.AddOutboxMessageEntity()`, `modelBuilder.AddOutboxStateEntity()`
- Accept `DbContextOptions<EventDbContext>` in constructor

---

### Step 3 — Create Entity Configurations

**Directory:** `src/TerraStream.EventService.Infrastructure/Persistence/Configurations/`

#### 3a. `EventConfiguration.cs` — `IEntityTypeConfiguration<Event>`

- Table: `"Events"`, schema: `"event"`
- Primary key: `Id` with value conversion `EventId ↔ Guid` via `HasConversion(id => (Guid)id, g => (EventId)g)`
- Concurrency token on `Version`
- Global query filter: `.HasQueryFilter(e => !e.IsDeleted)` (IsDeleted only — see Decisions)
- **Owned entities** (same table, column prefix):
  - `.OwnsOne(e => e.Theme, ...)` — all `EventTheme` properties mapped to columns with `Theme_` prefix
  - `.OwnsOne(e => e.Schedule, ...)` — all `EventSchedule` properties
  - `.OwnsOne(e => e.StreamConfig, ...)` — value conversion for `StreamKey` property (`StreamKey ↔ string`), `AbsLadderConfig` as `jsonb` column type
  - `.OwnsOne(e => e.AccessConfig, ...)` — `AccessMode` stored as string enum conversion
- **Collection navigations** via backing fields:
  - `.HasMany(e => e.Features)` with `.UsePropertyAccessMode(PropertyAccessMode.Field)` and metadata `"_features"`
  - `.HasMany(e => e.Invitations)` with field metadata `"_invitations"`
- **Ignore** `DomainEvents` property (transient, not persisted): `.Ignore(e => e.DomainEvents)`
- **Indexes**: `TenantId`, `Status`, composite `(TenantId, Status)`, unique on `StreamConfig_StreamKey`
- **Enum conversions**: store `EventStatus`, `StreamProtocol`, `AccessMode` as strings via `HasConversion<string>()`

#### 3b. `EventFeatureConfiguration.cs` — `IEntityTypeConfiguration<EventFeature>`

- Table: `"EventFeatures"`, schema: `"event"`
- PK: `Id` (Guid)
- FK to `Event` (shadow property `EventId`)
- `Config` as `jsonb` column type
- `FeatureType` stored as string

#### 3c. `EventInvitationConfiguration.cs` — `IEntityTypeConfiguration<EventInvitation>`

- Table: `"EventInvitations"`, schema: `"event"`
- PK: `Id` (Guid)
- FK to `Event` (shadow property `EventId`)
- `InvitationStatus` stored as string
- Index on `Email`

---

### Step 4 — Create `EventRepository`

**File:** `src/TerraStream.EventService.Infrastructure/Persistence/Repositories/EventRepository.cs`

Implements `IEventRepository`. Constructor injects `EventDbContext`.

| Method | Implementation |
|---|---|
| `GetByIdAsync` | Query with `.Include(Features).Include(Invitations)`, filter by `Id` + `TenantId` |
| `GetByStreamKeyAsync` (tenant-scoped) | Filter by `StreamConfig.StreamKey` + `TenantId`, include navigations |
| `GetByStreamKeyAsync` (cross-tenant) | `.IgnoreQueryFilters()` to bypass IsDeleted filter, filter only by `StreamKey`, include navigations |
| `ListAsync` | Build `IQueryable` from `EventQueryFilter`: filter on `TenantId`, optional `Status`, `SearchText` (ILIKE on `Name`/`Description`), date range on `Schedule.ScheduledStart`. If `IncludeDeleted` → `.IgnoreQueryFilters()` then re-apply tenant filter. Sort by `SortBy` (`createdAt`, `name`, `scheduledStart`) + direction. Paginate via `.Skip`/`.Take`. Separate `.CountAsync()` for total. Return `(events, totalCount)`. |
| `AddAsync` | `dbContext.Events.AddAsync(event)`, return entity |
| `UpdateAsync` | `dbContext.Events.Update(event)` (or attach if detached) |
| `DeleteAsync` | Load entity by `Id`+`TenantId`, call `event.SoftDelete()`, mark as modified |
| `ExistsAsync` | `.AnyAsync(e => e.Id == eventId && e.TenantId == tenantId)` |

---

### Step 5 — Create `DomainEventMapper`

**File:** `src/TerraStream.EventService.Infrastructure/Messaging/DomainEventMapper.cs`

Internal class with method:
```
IEnumerable<object> MapToIntegrationEvents(Event aggregate, IReadOnlyList<IDomainEvent> domainEvents)
```

Pattern-match on each domain event type:

| Domain Event | Integration Event | Enrichment |
|---|---|---|
| `EventScheduledDomainEvent` | `EventScheduledIntegrationEvent` | `ExpectedViewers` ← `aggregate.AccessConfig.MaxConcurrentViewers ?? 0` |
| `EventWentLiveDomainEvent` | `EventWentLiveIntegrationEvent` | `IngestEndpoint` ← `aggregate.StreamConfig.IngestEndpoint`, `StreamKey` → `.Value` (string) |
| `EventEndedDomainEvent` | `EventEndedIntegrationEvent` | Direct mapping |
| `EventCancelledDomainEvent` | `EventCancelledIntegrationEvent` | Direct mapping |

All integration events: `EventId` = `domain.EventId.Value` (Guid), `TenantId` = `domain.TenantId`, `OccurredAt` = `domain.OccurredAt`.

---

### Step 6 — Create `UnitOfWork`

**File:** `src/TerraStream.EventService.Infrastructure/Persistence/UnitOfWork.cs`

Implements `IUnitOfWork` with transactional outbox pattern. Constructor injects `EventDbContext`, MassTransit's scoped `IPublishEndpoint`, and `DomainEventMapper`.

**`SaveChangesAsync` flow:**

1. Get all tracked `Event` entities from `dbContext.ChangeTracker.Entries<Event>()`
2. Collect domain events from each: `aggregate.DomainEvents.ToList()`
3. For each aggregate with events, call `DomainEventMapper.MapToIntegrationEvents(aggregate, events)`
4. Publish each integration event via `publishEndpoint.Publish(integrationEvent, ct)` — MassTransit outbox intercepts and writes to `OutboxMessage` table within the same DbContext transaction
5. Call `dbContext.SaveChangesAsync(ct)` — atomically persists entity changes + outbox messages
6. Clear domain events from all aggregates: `aggregate.ClearDomainEvents()`
7. Return the result from `SaveChangesAsync`

---

### Step 7 — Create `UnitOfWorkBehavior`

**File:** `src/TerraStream.EventService.Application/Behaviors/UnitOfWorkBehavior.cs`

- In the **Application** layer (alongside `ValidationBehavior`) — depends only on the `IUnitOfWork` abstraction
- Implements `IPipelineBehavior<TMessage, TResponse>` (Mediator source-generated)
- Only wrap **command** messages (not queries): constrain `TMessage` or check by convention (e.g., messages in `Commands` namespace)
- Handler flow: `var response = await next();` → `await unitOfWork.SaveChangesAsync(ct);` → `return response;`
- Register in pipeline **after** `ValidationBehavior` (validation runs first, then handler, then SaveChanges)

---

### Step 8 — Create `TenantContext`

**File:** `src/TerraStream.EventService.Infrastructure/Services/TenantContext.cs`

Implements `ITenantContext`:
- Constructor injects `IHttpContextAccessor`
- `TenantId` getter: read `X-Tenant-Id` header from `httpContextAccessor.HttpContext.Request.Headers`
- Parse as `Guid`; throw `InvalidOperationException` if missing, null, or not a valid Guid
- Cache the parsed value via `Lazy<Guid>` to avoid repeated parsing per request

---

### Step 9 — Create `SystemClock`

**File:** `src/TerraStream.EventService.Infrastructure/Services/SystemClock.cs`

Implements `IClock`:
- `DateTimeOffset UtcNow => DateTimeOffset.UtcNow;`
- Register as singleton

---

### Step 10 — Create `DependencyInjection.cs`

**File:** `src/TerraStream.EventService.Infrastructure/DependencyInjection.cs`

Static class with extension method `AddInfrastructure(this IServiceCollection services, IConfiguration configuration)`:

- **EF Core**: `services.AddDbContext<EventDbContext>(o => o.UseNpgsql(configuration.GetConnectionString("EventServiceDb")))` with split-query behavior
- **Repository**: `services.AddScoped<IEventRepository, EventRepository>()`
- **UnitOfWork**: `services.AddScoped<IUnitOfWork, UnitOfWork>()`
- **TenantContext**: `services.AddScoped<ITenantContext, TenantContext>()`, `services.AddHttpContextAccessor()`
- **Clock**: `services.AddSingleton<IClock, SystemClock>()`
- **DomainEventMapper**: `services.AddScoped<DomainEventMapper>()` (internal, no interface needed)
- **MassTransit**:
  ```csharp
  services.AddMassTransit(x =>
  {
    x.AddEntityFrameworkOutbox<EventDbContext>(o =>
    {
      o.UsePostgres();
      o.UseBusOutbox(); // enables BusOutboxDeliveryService background service
    });
    x.UsingRabbitMq((ctx, cfg) =>
    {
      cfg.Host(configuration.GetConnectionString("RabbitMq") ?? "amqp://guest:guest@localhost:5672");
      cfg.ConfigureEndpoints(ctx);
    });
  });
  ```

---

### Step 11 — Update Configuration Files

**`src/TerraStream.EventService.Api/appsettings.json`** — add placeholder structure:
```json
{
  "ConnectionStrings": {
    "EventServiceDb": "",
    "RabbitMq": ""
  },
  "Streaming": {
    "DefaultIngestEndpoint": "rtmp://ingest.terrastream.de/live"
  }
}
```

**`src/TerraStream.EventService.Api/appsettings.Development.json`** — add development values:
```json
{
  "ConnectionStrings": {
    "EventServiceDb": "Host=localhost;Port=5432;Database=eventservice_db;Username=eventservice_user;Password=eventservice_password",
    "RabbitMq": "amqp://guest:guest@localhost:5672"
  },
  "Streaming": {
    "DefaultIngestEndpoint": "rtmp://ingest.terrastream.de/live"
  }
}
```

---

### Step 12 — Update `Program.cs`

In `src/TerraStream.EventService.Api/Program.cs`:

- Call `builder.Services.AddInfrastructure(builder.Configuration)`
- Wire up Application layer registration (AddApplication extension if not already done):
  - Register Mediator
  - Register FluentValidation validators
  - Register pipeline behaviors (`ValidationBehavior`, `UnitOfWorkBehavior`)
  - Bind `StreamingSettings` options

---

### Step 13 — Generate Initial EF Core Migration

1. Ensure `dotnet-ef` tool is installed
2. Run:
   ```
   dotnet ef migrations add InitialCreate \
     --project src/TerraStream.EventService.Infrastructure \
     --startup-project src/TerraStream.EventService.Api \
     --output-dir Persistence/Migrations
   ```
3. Verify the generated migration covers:
   - `event."Events"` table with all owned-type columns
   - `event."EventFeatures"` table
   - `event."EventInvitations"` table
   - MassTransit outbox tables: `InboxState`, `OutboxMessage`, `OutboxState`
   - All indexes

---

### Step 14 — Integration Test Scaffolding

#### 14a. Add NuGet Packages to IntegrationTests Project

Update `tests/TerraStream.EventService.IntegrationTests/TerraStream.EventService.IntegrationTests.csproj`:

| Package | Purpose |
|---|---|
| `Testcontainers.PostgreSql` | PostgreSQL Testcontainer |
| `Respawn` | Database reset between tests |
| `Microsoft.AspNetCore.Mvc.Testing` | `WebApplicationFactory<Program>` |
| `MassTransit` | `AddMassTransitTestHarness` |
| `Shouldly` | Assertions (project convention) |

#### 14b. Create `IntegrationTestFixture.cs`

Shared xUnit `IAsyncLifetime` fixture:
- Spin up PostgreSQL Testcontainer on startup
- Create `WebApplicationFactory<Program>` with overridden services:
  - Replace connection string with Testcontainer connection string
  - Replace MassTransit with test harness: `services.AddMassTransitTestHarness()`
- Apply EF Core migrations automatically on startup: `dbContext.Database.MigrateAsync()`
- Create `Respawn` checkpoint for database reset between tests
- Provide `HttpClient` factory, `IServiceScope` factory, and `ITestHarness` accessor

#### 14c. Create `IntegrationTestBase.cs`

Base class for integration test classes:
- Accept `IntegrationTestFixture` via constructor injection (`IClassFixture<>`)
- `InitializeAsync`: reset database via Respawn
- Helper methods: `CreateScope()`, `GetService<T>()`, `GetDbContext()`, `GetHttpClient(tenantId)` (pre-configures `X-Tenant-Id` header)

#### 14d. Create Basic Smoke Tests

| Test Class | Scope |
|---|---|
| `EventRepositoryTests.cs` | Verify CRUD operations against real PostgreSQL |
| `TenantIsolationTests.cs` | Verify events from different tenants are isolated |
| `UnitOfWorkTests.cs` | Verify domain events are mapped to outbox messages after SaveChanges |

#### 14e. Remove Placeholder Test

Delete `tests/TerraStream.EventService.IntegrationTests/IntegrationTestPlaceholder.cs`

---

## Verification Checklist

1. `dotnet build` — all projects compile without warnings (`TreatWarningsAsErrors` enabled)
2. `dotnet test --filter "Category=Unit"` — all 152 existing domain unit tests pass (no regressions)
3. `docker-compose up -d` — PostgreSQL and RabbitMQ containers start
4. `dotnet ef database update` — migration applies cleanly, verify tables exist:
   - `event."Events"`, `event."EventFeatures"`, `event."EventInvitations"`
   - `"InboxState"`, `"OutboxMessage"`, `"OutboxState"`
5. `dotnet test --filter "Category=Integration"` — repository CRUD, tenant isolation, and outbox tests pass against Testcontainers PostgreSQL
6. `dotnet run --project src/TerraStream.EventService.Api` — starts without errors and connects to PostgreSQL/RabbitMQ

---

## Files Created / Modified

### New Files

| File | Layer | Purpose |
|---|---|---|
| `Infrastructure/Persistence/EventDbContext.cs` | Infrastructure | EF Core DbContext with outbox tables |
| `Infrastructure/Persistence/Configurations/EventConfiguration.cs` | Infrastructure | Event aggregate mapping |
| `Infrastructure/Persistence/Configurations/EventFeatureConfiguration.cs` | Infrastructure | EventFeature table mapping |
| `Infrastructure/Persistence/Configurations/EventInvitationConfiguration.cs` | Infrastructure | EventInvitation table mapping |
| `Infrastructure/Persistence/Repositories/EventRepository.cs` | Infrastructure | `IEventRepository` implementation |
| `Infrastructure/Persistence/UnitOfWork.cs` | Infrastructure | `IUnitOfWork` with transactional outbox |
| `Infrastructure/Messaging/DomainEventMapper.cs` | Infrastructure | Domain→Integration event mapping |
| `Infrastructure/Services/TenantContext.cs` | Infrastructure | `ITenantContext` from HTTP header |
| `Infrastructure/Services/SystemClock.cs` | Infrastructure | `IClock` implementation |
| `Infrastructure/DependencyInjection.cs` | Infrastructure | `AddInfrastructure` extension |
| `Infrastructure/Persistence/Migrations/` | Infrastructure | Auto-generated EF Core migration |
| `Application/Behaviors/UnitOfWorkBehavior.cs` | Application | Pipeline behavior for SaveChanges |
| `IntegrationTests/IntegrationTestFixture.cs` | Tests | Shared Testcontainers + WAF fixture |
| `IntegrationTests/IntegrationTestBase.cs` | Tests | Base class with helpers |
| `IntegrationTests/EventRepositoryTests.cs` | Tests | Repository CRUD tests |
| `IntegrationTests/TenantIsolationTests.cs` | Tests | Multi-tenant isolation tests |
| `IntegrationTests/UnitOfWorkTests.cs` | Tests | Outbox integration tests |

### Modified Files

| File | Change |
|---|---|
| `Infrastructure/TerraStream.EventService.Infrastructure.csproj` | Add NuGet packages |
| `IntegrationTests/TerraStream.EventService.IntegrationTests.csproj` | Add NuGet packages |
| `Api/appsettings.json` | Add connection strings + streaming config |
| `Api/appsettings.Development.json` | Add development connection strings |
| `Api/Program.cs` | Wire up `AddInfrastructure` + `AddApplication` |

### Deleted Files

| File | Reason |
|---|---|
| `IntegrationTests/IntegrationTestPlaceholder.cs` | Replaced by real tests |

---

## Decisions

| Decision | Rationale |
|---|---|
| **`UnitOfWorkBehavior` in Application layer** | Consistent with `ValidationBehavior` — depends only on `IUnitOfWork` (an application abstraction), keeps infrastructure concerns out of the behavior definition |
| **Dedicated `DomainEventMapper`** | Separate from `UnitOfWork` for testability — enrichment logic (pulling `ExpectedViewers` from `AccessConfig`, `IngestEndpoint` from `StreamConfig`) is unit-testable in isolation |
| **Global query filter for `IsDeleted` only, not `TenantId`** | Tenant filtering is enforced at the repository level because the cross-tenant `GetByStreamKeyAsync` overload (used by SRS callbacks) needs to bypass tenant filtering without `IgnoreQueryFilters()` (which would also remove the `IsDeleted` filter) |
| **Schema prefix `"event"`** | Isolates Event Service tables in the shared PostgreSQL instance, avoids naming conflicts with other services |
| **MassTransit scoped `IPublishEndpoint`** | With `AddEntityFrameworkOutbox`, MassTransit replaces the publish endpoint with an outbox-aware version — calls to `Publish()` write to `OutboxMessage` table within the same DbContext transaction, achieving atomicity without manual outbox management |

Plan: API Layer (Tasks 6 + 7 + 8)
TL;DR: Build the full API surface for EventService — an EventsController with 18 REST endpoints, global exception-handling middleware, a gRPC service with 4 RPCs, and Program.cs wiring (health checks, CORS, dev auto-migration). Plus HTTP-level integration tests for the complete event lifecycle. The project uses the source-generated Mediator library (not MediatR), so all dispatches use ISender.Send() returning ValueTask<T>. Commands/queries already exist; the controller maps HTTP requests to them via thin request models.

Steps

1. Add NuGet packages to Api.csproj
   Grpc.AspNetCore — gRPC server hosting
   AspNetCore.HealthChecks.NpgSql — PostgreSQL readiness probe
2. Create API request models in Api/Requests/
   Thin record types that decouple the public HTTP contract from internal commands. Needed when route params ({id}) and body params must be combined into a command:

Request Model Fields Maps To
CreateEventRequest All 20+ creation fields (Name, Description, PrimaryColor, ScheduledStart, Protocol, AccessMode, …) CreateEventCommand
UpdateEventDetailsRequest Name, Description UpdateEventDetailsCommand(id, …)
SetEventThemeRequest PrimaryColor, SecondaryColor?, AccentColor?, LogoAssetId?, WatermarkAssetId?, PosterAssetId?, HidePlatformBranding, CustomCss? SetEventThemeCommand(id, …)
SetEventScheduleRequest ScheduledStart, ScheduledEnd, Timezone, LobbyMinutesBefore, LobbyContentAssetId?, PostEventContentAssetId? SetEventScheduleCommand(id, …)
ConfigureEventFeaturesRequest Features (list of { FeatureType, Enabled }) ConfigureEventFeaturesCommand(id, …)
SetEventAccessRequest AccessMode, MaxConcurrentViewers?, RequireAuthentication SetEventAccessCommand(id, …)
InviteUsersRequest Emails (list of string) InviteUsersCommand(id, …)
ValidateStreamKeyRequest StreamKey ValidateStreamKeyQuery(…)
State-transition endpoints (publish, go-live, end, archive, cancel, test-stream/\*) take only {id} from the route — no request body, no request model.

3. Create global exception handler in Api/Middleware/GlobalExceptionHandler.cs
   Implement IExceptionHandler (ASP.NET Core 8 pattern), registered via builder.Services.AddExceptionHandler<GlobalExceptionHandler>() and activated with app.UseExceptionHandler().

Exception HTTP Status Response Body
Application.Exceptions.ValidationException 400 Bad Request ProblemDetails with Extensions["errors"] containing the IDictionary<string, string[]>
Application.Exceptions.EventNotFoundException 404 Not Found ProblemDetails with event ID in detail
InvalidOperationException (from domain state transitions) 409 Conflict ProblemDetails with transition error message
Unhandled exceptions 500 Internal Server Error Generic ProblemDetails, log full exception
Use TypedResults / ProblemDetails factory for RFC 7807 compliance. Log at appropriate levels (Warning for 4xx, Error for 5xx).

4. Create EventsController in Api/Controllers/EventsController.cs
   [ApiController], [Route("api/v1/events")], inject ISender (from Mediator namespace). 18 action methods:

Action Attribute Route Behavior
Create [HttpPost] / Bind CreateEventRequest body → construct CreateEventCommand → Send → 201 Created with Location header via CreatedAtAction
GetById [HttpGet("{id:guid}")] /{id} GetEventByIdQuery(id) → 200 OK
List [HttpGet] / Bind query params (status, search, scheduledStartFrom, scheduledStartTo, page, size, sortBy, sortDirection) → ListEventsQuery(…) → 200 OK with PagedResultDto
UpdateDetails [HttpPut("{id:guid}/details")] /{id}/details Bind body + route → UpdateEventDetailsCommand(id, …) → 200 OK
SetTheme [HttpPut("{id:guid}/theme")] /{id}/theme → SetEventThemeCommand(id, …) → 200 OK
SetSchedule [HttpPut("{id:guid}/schedule")] /{id}/schedule → SetEventScheduleCommand(id, …) → 200 OK
ConfigureFeatures [HttpPut("{id:guid}/features")] /{id}/features → ConfigureEventFeaturesCommand(id, …) → 200 OK
SetAccess [HttpPut("{id:guid}/access")] /{id}/access → SetEventAccessCommand(id, …) → 200 OK
InviteUsers [HttpPost("{id:guid}/invitations")] /{id}/invitations → InviteUsersCommand(id, …) → 200 OK
Publish [HttpPost("{id:guid}/publish")] /{id}/publish → PublishEventCommand(id) → 200 OK
StartTestStream [HttpPost("{id:guid}/test-stream/start")] /{id}/test-stream/start → StartTestStreamCommand(id) → 200 OK
StopTestStream [HttpPost("{id:guid}/test-stream/stop")] /{id}/test-stream/stop → StopTestStreamCommand(id) → 200 OK
GoLive [HttpPost("{id:guid}/go-live")] /{id}/go-live → GoLiveCommand(id) → 200 OK
EndStream [HttpPost("{id:guid}/end")] /{id}/end → EndStreamCommand(id) → 200 OK
Archive [HttpPost("{id:guid}/archive")] /{id}/archive → ArchiveEventCommand(id) → 200 OK
Cancel [HttpPost("{id:guid}/cancel")] /{id}/cancel → CancelEventCommand(id) → 200 OK
GetObsProfile [HttpGet("{id:guid}/obs-profile")] /{id}/obs-profile → GetObsProfileQuery(id) → serialize ObsProfileDto → return FileContentResult with Content-Type: application/json and Content-Disposition: attachment; filename=obs-profile-{id}.json
ValidateStreamKey [HttpPost("stream-key/validate")], [AllowAnonymous] /stream-key/validate → ValidateStreamKeyQuery(request.StreamKey) → 200 OK
Key details:

All actions use await \_sender.Send(…) (returns ValueTask<T>)
Route constraints use {id:guid} for type safety
ValidateStreamKey is the only [AllowAnonymous] endpoint (called by SRS ingest server internally)
Create returns 201 with CreatedAtAction(nameof(GetById), new { id = result.Id }, result)
GetObsProfile serializes the DTO to JSON bytes, wraps in File(bytes, "application/json", $"obs-profile-{id}.json") to force download

5. Create EventGrpcServiceImpl in Api/GrpcServices/EventGrpcServiceImpl.cs
   Extends the generated EventGrpcService.EventGrpcServiceBase from event_service.proto. Inject ISender. 4 RPC implementations:

RPC Dispatches To Tenant Handling
GetEvent GetEventByIdQuery Extract tenant_id from request message → set X-Tenant-Id header on ServerCallContext.GetHttpContext()
GetEventsByStatus GetEventsByStatusQuery Same tenant header injection
GetEventStreamConfig GetEventStreamConfigQuery Same tenant header injection
ValidateStreamKey ValidateStreamKeyQuery No tenant needed (cross-tenant)
Tenant flow: For RPCs that include tenant_id in the request message, set context.GetHttpContext().Request.Headers["X-Tenant-Id"] = request.TenantId before dispatching. This reuses the existing TenantContext infrastructure without refactoring.

Map application DTOs → protobuf response messages using explicit mapping in each method. Handle EventNotFoundException → RpcException(StatusCode.NotFound).

6. Update Program.cs — Full wiring
   Add to service registration (before Build()):

builder.Services.AddExceptionHandler<GlobalExceptionHandler>() + builder.Services.AddProblemDetails()
builder.Services.AddGrpc()
Health checks: builder.Services.AddHealthChecks().AddNpgSql(connectionString, name: "postgresql") — use the same connection string from configuration
CORS: builder.Services.AddCors(options => options.AddPolicy("Development", policy => policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader())) — restrict in production
Add to middleware pipeline (after Build()):

app.UseExceptionHandler() — early in pipeline, catches all exceptions
app.UseCors("Development") — only in development (wrap in if (app.Environment.IsDevelopment()))
app.UseRouting() (if not already)
app.UseAuthentication() (placeholder for Keycloak)
app.UseAuthorization()
app.MapControllers()
app.MapGrpcService<EventGrpcServiceImpl>()
app.MapHealthChecks("/health/live", new HealthCheckOptions { Predicate = \_ => false }) — liveness (no checks, always healthy if process runs)
app.MapHealthChecks("/health/ready", new HealthCheckOptions { Predicate = check => check.Name == "postgresql" }) — readiness (PostgreSQL connectivity)
Add dev-only auto-migration block:

Keep existing Swagger setup (already in place, dev-only gated).

7. Add API integration tests in TerraStream.EventService.IntegrationTests
   Leverage the existing IntegrationTestFixture (Testcontainers PostgreSQL + WebApplicationFactory<Program> + Respawn) and IntegrationTestBase. The fixture already provides CreateHttpClient(tenantId) which sets X-Tenant-Id header.

Create test classes:

Test Class Tests Coverage
EventsControllerCreateTests 3-4 POST create → 201 + Location header, validation errors → 400, missing required fields → 400
EventsControllerCrudTests 5-6 GET by id → 200, GET by id not found → 404, PUT details/theme/schedule/features/access → 200, POST invitations → 200
EventsControllerLifecycleTests 5-6 Full lifecycle roundtrip: create → publish (→ Scheduled) → go-live (→ Live) → end (→ Ended) → archive (→ Archived). Also: invalid transition → 409, cancel from Draft → 200
EventsControllerQueryTests 3-4 GET list with pagination, status filter, search filter. GET obs-profile returns attachment disposition.
StreamKeyValidationTests 3 POST validate with valid key → IsValid: true, invalid key → IsValid: false, endpoint is unauthenticated (no X-Tenant-Id header needed)
EventsControllerTenantIsolationTests 2-3 Tenant A creates event → Tenant B GET by id → 404. Tenant A list doesn't show Tenant B events.
Test pattern: Use HttpClient from fixture → send HTTP requests → assert status codes, response bodies, and headers. Deserialize responses with System.Text.Json.

Estimated: ~20-25 new integration tests.

8. Swagger enhancements (optional, low-effort)
   Add XML documentation comments on EventsController actions for auto-generated OpenAPI docs. Add [ProducesResponseType] attributes for each endpoint to document possible status codes (200, 201, 400, 404, 409). The existing Swashbuckle.AspNetCore package + GenerateDocumentationFile=true in Directory.Build.props handles the rest.

Verification

dotnet build — 0 errors, 0 warnings (TreatWarningsAsErrors enabled)
dotnet test — all 152 existing unit tests pass + ~20-25 new integration tests pass
docker-compose up → service starts → GET /health/live returns 200 → GET /health/ready returns 200 (PostgreSQL connected)
Swagger UI at /swagger shows all 18 endpoints with correct routes and response types
Manual smoke test: POST /api/v1/events with valid body → 201, GET /api/v1/events/{id} → 200 with full DTO
POST /api/v1/events/stream-key/validate works without X-Tenant-Id header
Decisions

Separate request models over direct command binding: Decouples public API contract from internals; needed anyway for endpoints combining route {id} with body fields
IExceptionHandler over custom middleware: ASP.NET Core 8 built-in pattern, integrates with ProblemDetails factory
gRPC tenant via header injection on HttpContext: Reuses existing TenantContext without refactoring; the proto messages already carry tenant_id
ISender over IMediator: Narrower interface — only needs Send(), not Publish() (publishing is handled by UnitOfWork + MassTransit outbox)
Health check split (live/ready): Kubernetes convention — liveness = process alive, readiness = dependencies reachable

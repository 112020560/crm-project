# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

All commands run from the solution root (`/Users/home/Documents/GitHub/crm-project`).

**Build:**
```bash
dotnet build CrmProject.sln
```

**Run the API:**
```bash
dotnet run --project Backend/Src/Crm.WebApi/Crm.WebApi.csproj
```

**EF Core migrations** (run from the `Crm.WebApi` or `Crm.Infrastructure` directory):
```bash
dotnet ef migrations add <MigrationName> --project ../Crm.Infrastructure --startup-project .
dotnet ef database update --project ../Crm.Infrastructure --startup-project .
```

## Architecture

This is a .NET 9 Web API following **Clean Architecture** with four layers:

```
Crm.Domain         → Entities, domain abstractions (ICustomersRepository, IUnitOfWork)
Crm.Application    → Use cases via MediatR (Commands/Queries), DTOs, FluentValidation
Crm.Infrastructure → EF Core (PostgreSQL), MassTransit/RabbitMQ implementations
Crm.WebApi         → Minimal API endpoints, DI wiring, middleware
```

### Request flow

HTTP request → `IEndpoint` (Minimal API) → `IMediator.Send()` → MediatR pipeline → `ICommandHandler` / `IQueryHandler` → Repository via `IUnitOfWork` → PostgreSQL

MediatR pipeline behaviors (applied in order):
1. `RequestLoggingPipelineBehavior` — logs every request
2. `ValidationPipelineBehavior` — runs FluentValidation; returns `Result.ValidationFailure` instead of throwing

### Key patterns

- **Result type**: All handlers return `Result<T>` or `Result` from the `SharedKernel` NuGet package. Endpoints call `.Match()` to produce `IResult`.
- **Endpoints**: Each endpoint class implements `IEndpoint` and is auto-discovered via reflection at startup (`AddEndpoints` / `MapEndpoints` in `EndpointExtensions.cs`). All routes are versioned under `api/v{version}`.
- **Commands vs Queries**: Commands implement `ICommand<TResponse>` (wraps `IRequest<Result<TResponse>>`); queries implement `IQuery<TResponse>`.
- **Messaging**: After mutating state, commands publish to RabbitMQ via `IMqProducerService`. Two patterns are used: `SendCommand` (point-to-point to a named queue) and `PublishEvent` (broadcast). MassTransit wraps RabbitMQ.
- **Domain entities**: `Customer` is a partial class split across `Customer.cs` (properties + `ConvertToModel()`) and EF configuration in `CrmDbContext.cs`. Domain entities are mapped directly to DB tables (no separate EF models).
- **Telemetry**: OpenTelemetry via the internal `SmartCore.Telemetry` NuGet package, exporting to OTLP. Serilog is used for structured logging (Console + Seq in development).

### Infrastructure dependencies

- **Database**: PostgreSQL via Npgsql EF Core. Connection string key: `ConnectionStrings:DefaultConnection`.
- **Message broker**: RabbitMQ. Connection URI key: `RabbitMqSettings:Uri`.
- **Logging**: Seq at `http://localhost:5341` in development.
- **Tracing**: OTLP endpoint at `Telemetry:OtlpEndpoint` (default `http://localhost:4317`).

### Configuration

Secrets and environment-specific values go in `appsettings.Development.json` (not committed in production). Required config keys:
- `ConnectionStrings:DefaultConnection`
- `RabbitMqSettings:Uri`
- `Jwt:Secret`, `Jwt:Issuer`, `Jwt:Audience`

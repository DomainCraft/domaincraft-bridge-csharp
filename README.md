# DomainCraft C# Bridge

A bridge template for [DomainCraft](https://github.com/Gitlawb/domaincraft) that generates a production-ready **ASP.NET Core REST API** with clean architecture, EF Core, PostgreSQL, JWT authentication, and more.

## What It Generates

Given a `domain.yaml` file, this bridge produces a complete, runnable C# solution:

```
Generated/
├── src/
│   ├── Domain/           # Entities, enums, DTOs
│   ├── Application/      # Service interfaces, services (custom + generated), repositories
│   ├── Infrastructure/   # EF Core, repositories, seeders, Redis cache
│   └── WebApi/           # Controllers, auth, health checks, Docker
├── tests/
│   └── ApiTests.cs       # Integration tests with InMemory DB
├── EcommercePlatform.sln
├── Dockerfile
└── docker-compose.yml
```

## Day-2 Safety (Generation Gap)

The hardest problem for a code generator is the second day: you regenerate and your hand-written
code is wiped. This bridge solves it with the **Generation Gap** pattern backed by DomainCraft's
`overwrite: false` support.

Every entity gets a `partial class` service split across two files:

| File | Behavior |
|------|----------|
| `src/Application/Generated/<Entity>Service.g.cs` | **Regenerated on every run.** CRUD logic, soft-delete handling, and `partial void OnBeforeCreate / OnBeforeUpdate / OnBeforeDelete` hooks (no body). |
| `src/Application/Services/<Entity>Service.cs` | **Generated once** (`overwrite: false`), owned by you. Implement the hooks to add custom business logic. |

Controllers talk to the `I<Entity>Service` interface, and DI resolves it to the *merged* partial
class — so your hook implementations always take effect:

```csharp
// src/Application/Services/ProductService.cs — never overwritten
public partial class ProductService
{
    partial void OnBeforeCreate(Product entity)
    {
        entity.SKU = "CUSTOM-" + Guid.NewGuid().ToString("N")[..8];
    }
}
```

Add a field to `domain.yaml` and regenerate: only the `.g.cs` part is rewritten, your custom hook
survives. Rename or delete the entity and the migration engine renames/deletes your custom file
alongside it.

### Hand-editable files

The only scaffold-once (`overwrite: false`) files in this bridge are the custom service partials:

- `src/Application/Services/<Entity>Service.cs` — custom service partials (hooks)

Everything else — including entities, repositories, controllers, `Program.cs`, `appsettings.json`,
and the `.g.cs` services — is regenerated on every run. Put business logic in the custom partials,
never in `src/Application/Generated/`.

## Implemented Features

### Core (from domain.yaml spec)

| Feature | Status | Notes |
|---------|--------|-------|
| **Entities & Fields** | Done | All field types: string, text, int, float, decimal, bool, uuid, datetime, json/jsonb |
| **Enums** | Done | Generated as C# enums with string serialization |
| **Relations** | Done | One-to-many, many-to-many with navigation properties |
| **Indexes** | Done | B-tree and unique indexes via EF Core configuration |
| **Seed Data** | Done | JSON-based seed data with `DomainSeeder` |
| **Permissions** | Done | Role-based policies: `@Owner`, `Admin`, `User`, `*` (public) |
| **Features** | Done | `audit`, `soft_delete`, `optimistic_lock`, `audit_log` |
| **Validations** | Done | `required`, `unique`, `email`, `url`, `min`, `max`, `gte`, `lte`, `gt`, `lt`, `regex` |
| **Hidden Fields** | Done | Fields marked `hidden` (e.g. password) excluded from API responses |
| **Multi-tenancy** | Done | Column-based tenancy with `TenantId` filter |

### Architecture & Patterns

| Feature | Status | Notes |
|---------|--------|-------|
| **Clean Architecture** | Done | Domain → Application → Infrastructure → WebApi layers |
| **Generation Gap (Day-2)** | Done | `partial` services: regenerated core + `overwrite: false` custom hooks |
| **Service Layer** | Done | Controllers talk to `I<Entity>Service`; services orchestrate repositories + hooks |
| **Repository Pattern** | Done | Generic `IRepository<T>` + per-entity interfaces |
| **Separate Repositories** | Done | Each entity gets its own repository interface and implementation |
| **EF Core Configurations** | Done | `IEntityTypeConfiguration<T>` per entity with proper column mapping |
| **Value Converters** | Done | `JsonDocument?` ↔ string for json/jsonb fields |
| **Owner Resolution** | Done | `IOwnerResolver` / `OwnerResolver` for `@Owner` permission checks |
| **Permission Service** | Done | `IPermissionService` with policy-based authorization |

### Infrastructure & DevOps

| Feature | Status | Notes |
|---------|--------|-------|
| **JWT Authentication** | Done | Full JWT bearer with configurable secret, issuer, audience |
| **Authorization Policies** | Done | Auto-generated per-entity policies (e.g. `ProductRead`, `OrderCreate`) |
| **Wildcard Permissions** | Done | `*` maps to `[AllowAnonymous]` on controllers |
| **Redis Caching** | Done | `ICacheService` with Redis implementation + graceful fallback |
| **Health Checks** | Done | PostgreSQL + Redis health endpoints with JSON response |
| **Docker** | Done | Multi-stage `Dockerfile` + `docker-compose.yml` with PostgreSQL, Redis, API |
| **Swagger/OpenAPI** | Done | Enabled in development mode |
| **CORS** | Done | Configurable policy |
| **Integration Tests** | Done | xUnit + `WebApplicationFactory` + EF Core InMemory provider |
| **Test Data Generation** | Done | Smart defaults: email fields → valid email, enum fields → default value |

## Quick Start

### 1. Define your domain

```yaml
# domain.yaml
project:
  name: My App

entities:
  Product:
    fields:
      id: uuid [primary]
      title: string [required, min:3, max:200]
      price: decimal [required, gte:0]
      status: enum(Status) [default:DRAFT]
    permissions:
      read: ["*"]
      create: [Admin]
      delete: [Admin]
    seed:
      - { id: "...", title: "Widget", price: "9.99", status: DRAFT }
```

### 2. Generate code

```bash
domaincraft generate --domain domain.yaml --bridge ../DomainCraftCsharp --output ./generated
```

### 3. Run

```bash
cd generated
dotnet run --project src/WebApi
```

### 4. Test

```bash
dotnet test
```

### 5. Docker

```bash
docker-compose up --build
```

## Permission System

Permissions map directly from `domain.yaml` to ASP.NET Core authorization:

| YAML Permission | Controller Attribute | Behavior |
|----------------|---------------------|----------|
| `read: ["*"]` | `[AllowAnonymous]` | Public read access |
| `read: [Admin, Editor]` | `[Authorize(Policy = "EntityRead")]` | Role-restricted |
| `create: ["*"]` | `[AllowAnonymous]` | Public create |
| `update: ["@Owner", Admin]` | `[Authorize]` + ownership check | Owner or admin |
| *(no permissions defined)* | `[AllowAnonymous]` | Public by default |

## Template Files

| Template | Generates |
|----------|-----------|
| `solution.sln.tmpl` | Solution file |
| `*.csproj.tmpl` | Project files (Domain, Application, Infrastructure, WebApi, Tests) |
| `entity.cs.tmpl` | Entity classes with data annotations |
| `enums.cs.tmpl` | Enum definitions |
| `entity-configuration.cs.tmpl` | EF Core `IEntityTypeConfiguration<T>` |
| `controller.cs.tmpl` | REST API controllers with auth (talk to `I<Entity>Service`) |
| `dbcontext.cs.tmpl` | `DbContext` with entity registration |
| `repository*.tmpl` | Repository interfaces and implementations |
| `iservice.cs.tmpl` | Per-entity `I<Entity>Service` interfaces |
| `generated-service.cs.tmpl` | Regenerated `partial` services with `OnBeforeCreate/Update/Delete` hooks |
| `custom-service.cs.tmpl` | Developer-owned `overwrite: false` partial services |
| `service-registration.cs.tmpl` | Application DI (`I<Entity>Service → <Entity>Service`) |
| `permissions.cs.tmpl` | Permission policy definitions |
| `IPermissionService.cs.tmpl` | Permission service interface |
| `PermissionService.cs.tmpl` | Permission service implementation |
| `IOwnerResolver.cs.tmpl` | Owner resolver interface |
| `OwnerResolver.cs.tmpl` | Owner resolver implementation |
| `seed-seeder.cs.tmpl` | Database seeder |
| `redis-cache.cs.tmpl` | Redis cache service |
| `icache-service.cs.tmpl` | Cache service interface |
| `health-checks.cs.tmpl` | Health check endpoints |
| `Program.cs.tmpl` | Application entry point |
| `Dockerfile.tmpl` | Multi-stage Docker build |
| `docker-compose.yml.tmpl` | Docker Compose with PostgreSQL + Redis |
| `tests.cs.tmpl` | Integration tests |
| `appsettings.json.tmpl` | Configuration |

## Planned Features

These features are defined in the domain.yaml spec but not yet fully implemented:

| Feature | Spec Status | Notes |
|---------|-------------|-------|
| **GraphQL API** | In spec (`api_style: graphql`) | Not yet implemented |
| **gRPC API** | In spec (`api_style: grpc`) | Not yet implemented |
| **MySQL support** | In spec (`database: mysql`) | EF Core provider swap needed |
| **SQLite support** | In spec (`database: sqlite`) | EF Core provider swap needed |
| **MSSQL support** | In spec (`database: mssql`) | EF Core provider swap needed |
| **MongoDB support** | In spec (`database: mongodb`) | Requires different repository pattern |
| **File uploads** | Not in spec | Common requirement for avatars, documents |
| **Pagination** | Not in spec | Offset/cursor pagination for list endpoints |
| **Sorting/Filtering** | Not in spec | Query parameter-based filtering |
| **Rate Limiting** | Not in spec | API throttling |
| **SignalR / WebSockets** | Not in spec | Real-time updates |
| **Event Sourcing** | Not in spec | Audit log could evolve into event store |
| **CQRS** | Not in spec | Command/Query separation |

## Requirements

- .NET 9.0 SDK
- PostgreSQL (for production)
- Redis (optional, for caching)

## License

Part of the [DomainCraft](https://github.com/Gitlawb/domaincraft) project.

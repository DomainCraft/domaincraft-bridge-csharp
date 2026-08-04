# DomainCraft C# Bridge

A bridge template for [DomainCraft](https://github.com/Gitlawb/domaincraft) that generates a production-ready **ASP.NET Core REST API** with clean architecture, EF Core, PostgreSQL, JWT authentication, and more.

## What It Generates

Given a `domain.yaml` file, this bridge produces a complete, runnable C# solution:

```
Generated/
├── src/
│   ├── Domain/           # Entities, enums, DTOs
│   ├── Application/      # Service interfaces, services (custom + generated), repositories
│   ├── Infrastructure/   # EF Core, repositories, seeders, caching (Dapr / in-memory)
│   └── WebApi/           # Controllers, auth, health checks, Docker
├── tests/
│   └── ApiTests.cs       # Integration tests: WebApplicationFactory + Testcontainers (real PostgreSQL)
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
| `src/Application/Generated/<Entity>Service.g.cs` | **Regenerated on every run.** CRUD logic, soft-delete handling, and `partial void OnBeforeCreate/OnBeforeUpdate/OnBeforeDelete` hooks (pre-commit) + `partial void OnAfterCreate/OnAfterUpdate/OnAfterDelete` hooks (post-commit, for side effects). No body. |
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
| **Relations** | Done | One-to-many/one-to-one via FKs; many-to-many via an EF Core join table (navigations on both sides) |
| **Indexes** | Done | B-tree and unique indexes via EF Core configuration |
| **Seed Data** | Done | JSON-based seed data with `DomainSeeder` |
| **Permissions** | Done | Role-based policies: `@Owner`, `Admin`, `User`, `*` (public) |
| **Features** | Done | `audit`, `soft_delete`, `optimistic_lock`, `audit_log` |
| **Event Published** | Done | `event_sourced` entities publish `X Created/Updated/Deleted` via `IEventPublisher` |
| **Validations** | Done | `required`, `unique`, `email`, `url`, `min`, `max`, `gte`, `lte`, `gt`, `lt`, `regex`. String min/max → `[StringLength]`; numeric bounds → `[Range]`; email/url/regex → data annotations; `unique` → index |
| **Hidden Fields** | Done | Fields marked `hidden` excluded from API responses; the auth entity's `password` field is always hidden from DTOs/requests/patch — it is only ever set through the auth controller (BCrypt) |
| **Readonly Fields** | Done | Fields marked `readonly` (`balance: decimal [required, readonly, gte:0]`) stay in API responses but are excluded from Create/Update/PATCH requests and request→entity mapping — server-owned values the client can read but never set |

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
| **JWT Authentication** | Done | Full JWT bearer with configurable secret, issuer, audience; auth controller goes through an `IAuthService` port (Generation Gap partials: `Generated/AuthService.g.cs` + custom `Services/AuthService.cs` with `OnBeforeRegister` hook) — no `DbContext` in the controller |
| **Pagination / Sort / Search** | Done | `PagedResult<T>` on every list endpoint (`?page=&pageSize=&sort=-field&q=`); DB-side paging, typed sort keys, string search |
| **Authorization Policies** | Done | Auto-generated per-entity policies (e.g. `ProductRead`, `OrderCreate`) |
| **Wildcard Permissions** | Done | `*` maps to `[AllowAnonymous]` on controllers |
| **Caching (Dapr / in-memory)** | Done | `ICacheService` → `DaprCacheService` (state store, `--addons dapr`) or `InMemoryCacheService`. No direct Redis. |
| **Domain Events / Dapr** | Addon | `IEventPublisher` port → `InMemoryEventPublisher` (default) or `DaprEventPublisher` (`--addons dapr`) |
| **Email** | Addon | `IEmailService` port → `InMemoryEmailService` (default) or `DaprEmailService` (SMTP binding, `--addons dapr`) |
| **Object Storage** | Done | `IStorageService` port → `LocalFileStorageService` (default, local folder) or `DaprStorageService` (output binding: local/S3/Blob/GCS, `--addons dapr`) |
| **Background Jobs** | Addon | Hangfire (PostgreSQL storage) with a daily recurring seed job, gate to the dapr addon |
| **API Versioning** | Done | `Asp.Versioning` + `[ApiVersion("1.0")]` on controllers (default v1, version header for v2+) |
| **Partial Updates (PATCH)** | Done | Merge-patch on every resource controller, implemented entirely with `System.Text.Json` (no Newtonsoft.Json) |
| **DB Migrations** | Done | `domaincraft generate --migrate` runs `dotnet ef migrations add` + `database update` (declared in `bridge.yaml`) |
| **Password Hashing** | Done | `IPasswordHasher` → `BcryptPasswordHasher` (BCrypt.Net-Next, work factor 12); wired into register/login. No plain-text passwords |
| **Rate Limiting** | Done | `AddRateLimiter` with fixed + sliding-window policies, `429` rejection |
| **Security Headers** | Done | HSTS/CSP/X-Content-Type-Options/X-Frame-Options/Referrer-Policy middleware |
| **Response Compression** | Done | Brotli + Gzip |
| **Output Caching** | Not enabled | `AddOutputCache` wires middleware, but the base policy is `NoCache` (paginated/sorted/user-aware endpoints are intentionally not cached). Add an explicit policy + `[OutputCache]` where a route is proven cache-safe. |
| **EF Connection Pooling** | Done | `AddDbContextPool` (poolSize 128) + transient-failure retry + command timeout |
| **Eager Loading** | Done | `GetAllAsync` uses `AsNoTracking()` + generated `Include(...)` for relations |
| **Health Checks** | Done | PostgreSQL health endpoint with JSON response |
| **Docker** | Done | Multi-stage `Dockerfile` + `docker-compose.yml` with PostgreSQL, API |
| **Swagger/OpenAPI** | Done | Enabled in development mode |
| **CORS** | Done | Configurable policy |
| **Integration Tests** | Done | xUnit + `WebApplicationFactory` + Testcontainers PostgreSQL (real FK/cascade/SQL) |
| **Test Data Generation** | Done | Smart defaults: email fields → valid email, enum fields → default value |

## Dapr Addon (`--addons dapr`)

Dapr is an **infrastructure addon**, not a separate domain feature. The domain
model only declares *what* events matter (`event_sourced`), never *which* broker:

```bash
# Default monolith — in-process event publishing (no sidecar, deploy anywhere)
domaincraft generate

# Enterprise — Dapr sidecar wired for the declared infrastructure
domaincraft generate --addons dapr
```

When the addon is enabled, this bridge additionally generates:

- `src/Application/Events/IEventPublisher.cs` — the outbound event port (always generated).
- `src/Infrastructure/Events/InMemoryEventPublisher.cs` — default; logs + dispatches in-process.
- `src/Infrastructure/Events/DaprEventPublisher.cs` — publishes via the Dapr sidecar (`--addons dapr`).
- `src/Infrastructure/Caching/DaprCacheService.cs` — cache via Dapr **state store** (Redis is not used directly).
- `src/Infrastructure/Email/DaprEmailService.cs` — email via Dapr's SMTP **output binding**.
- `src/Infrastructure/Storage/DaprStorageService.cs` — objects/files via Dapr's storage **output binding** (local/S3/Blob/GCS).
- `dapr/components/pubsub.yaml`, `dapr/components/statestore.yaml`, `dapr/components/email.yaml`, `dapr/components/storage.yaml` —
  Dapr component manifests, driven by `project.infrastructure.queue` / `.cache` / `.secrets` / `.storage`.
- Dapr wiring in `Program.cs` (`AddDaprClient`) and a Dapr sidecar service in `docker-compose.yml`.
- Hangfire background processing (`Program.cs` + `Hangfire.PostgreSql`) with a daily recurring seed/health job.

`event_sourced` entities publish `X Created / Updated / Deleted` events after a successful
`SaveChangesAsync`. Swap the broker / cache store / mailer / object store by editing the manifests under
`dapr/components/*` — your code never changes (no vendor lock-in, no direct Redis SDK).

```yaml
project:
  infrastructure:
    queue: "redis"     # broker: pubsub, rabbitmq, kafka, redis, nats
    cache: "redis"     # store: redis, memcached, in-memory, ...
    secrets: "local"   # local, kubernetes, azure-keyvault, aws-secrets
    storage: "s3"      # object store: local, s3, azure-blob, gcs

entities:
  Order:
    features: [audit, event_sourced, cacheable]
```

### Database migrations (`--migrate`)

Schema changes reach the database without hand-written SQL. The bridge declares the
migration commands in `bridge.yaml`:

```yaml
migrations:
  enabled: true
  commands:
    - "dotnet ef migrations add InitialCreate --project src/Infrastructure --startup-project src/WebApi --output-dir Persistence/Migrations"
    - "dotnet ef database update --project src/Infrastructure --startup-project src/WebApi"
```

Run them (opt-in, only when you ask):

```bash
domaincraft generate --addons dapr --migrate
```

The core CLI executes the commands in order from the generated output directory.

### API versioning & PATCH

- Every controller is marked `[ApiVersion("1.0")]`; `AddApiVersioning` + `AddApiExplorer`
  are wired in `Program.cs`. Future routes add `[ApiVersion("2.0")]` without touching v1.
- Every resource endpoint ships a `PATCH /api/.../{id}` **merge-patch** action: it reads the
  body as a `JsonElement` and assigns only the scalar fields present in the JSON. It is built
  on `System.Text.Json` — **no Newtonsoft.Json.** The project intentionally avoids the
  Newtonsoft-dependent JSON Patch SDK; partial updates are plain object deserialization.

### Security (1.3)
- **BCrypt passwords**: `IPasswordHasher` / `BcryptPasswordHasher` hash on register and verify
  on login — credentials are never stored or compared in plain text.
- **Rate limiting**: `AddRateLimiter` (fixed + sliding window) with `429` + `Retry-After`.
- **Security headers**: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`,
  `Content-Security-Policy`, HSTS (production).
- **CORS from configuration**: single `CorsPolicy` whose origins come from
  `appsettings.json` → `Cors:AllowedOrigins`, falling back to `project.cors`, then allow-all.

### Performance (1.4)
- **Response compression** (Brotli + Gzip), **output caching** (opt-in via `[OutputCache]`; default policy `NoCache` so paginated/sorted/user-aware endpoints aren't cached).
- **EF Core connection pooling** via `AddDbContextPool` plus retry-on-failure and command timeout.
- **Eager loading** in `GetAllAsync` and `GetPagedAsync` (`AsNoTracking()` + generated `Include(...)` for relations).

### API list endpoints (1.2)
Every `GET /api/<resource>` is paginated and searchable:
```
GET /api/orders?page=1&pageSize=20&sort=-total&q=laptop
```
Returns `PagedResult<OrderDto>` with `items`, `totalCount`, `page`, `pageSize`, `totalPages`,
`hasNextPage`. Sorting keys (`sort`) and the searched string fields are generated per entity.

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
| *(no permissions defined)* | `[AllowAnonymous]` | Public by default |

> **How roles work end-to-end.** To use role-based policies, give the auth entity a `role`
> field (any enum/string). On login the JWT is issued with a `ClaimTypes.Role` claim taken
> from that field, so `[Authorize(Policy = "EntityRead")]` / `RequireRole("...")` honour real
> roles. Registration does **not** let a caller self-assign a role — new users get the role
> field's schema default (e.g. `Customer`); create privileged users via the entity `seed`
> data (the sample `admin@example.com` is `role: Admin`). The runtime bypass list for
> `@Owner` checks is also data-driven from the operation's `@Owner, <roles>` list — nothing is
> hardcoded.

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
| `generated-service.cs.tmpl` | Regenerated `partial` services with `OnBeforeCreate/Update/Delete` + `OnAfterCreate/Update/Delete` hooks |
| `custom-service.cs.tmpl` | Developer-owned `overwrite: false` partial services |
| `service-registration.cs.tmpl` | Application DI (`I<Entity>Service → <Entity>Service`) |
| `permissions.cs.tmpl` | Permission policy definitions |
| `IPermissionService.cs.tmpl` | Permission service interface |
| `PermissionService.cs.tmpl` | Permission service implementation |
| `IOwnerResolver.cs.tmpl` | Owner resolver interface |
| `OwnerResolver.cs.tmpl` | Owner resolver implementation |
| `seed-seeder.cs.tmpl` | Database seeder |
| `redis-cache.cs.tmpl` | *(removed — no direct Redis)* |
| `DaprCacheService.cs.tmpl` | Distributed cache via Dapr state store (`--addons dapr`) |
| `InMemoryCacheService.cs.tmpl` | In-process cache (default) |
| `IEmailService.cs.tmpl` | Email port |
| `DaprEmailService.cs.tmpl` | Email via Dapr SMTP binding (`--addons dapr`) |
| `InMemoryEmailService.cs.tmpl` | Email logging (default) |
| `icache-service.cs.tmpl` | Cache service interface |
| `health-checks.cs.tmpl` | Health check endpoints |
| `Program.cs.tmpl` | Application entry point |
| `Dockerfile.tmpl` | Multi-stage Docker build |
| `docker-compose.yml.tmpl` | Docker Compose with PostgreSQL + API |
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
| **SignalR / WebSockets** | Not in spec | Real-time updates |
| **CQRS** | Not in spec | Command/Query separation |

## Requirements

- .NET 10.0 SDK (target framework `net10.0`)
- PostgreSQL (for production)
- Dapr runtime + sidecar (only for `--addons dapr`; not needed for the default monolith)

## License

Part of the [DomainCraft](https://github.com/Gitlawb/domaincraft) project.

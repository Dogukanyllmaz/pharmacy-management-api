# Pharmacy Management API

ASP.NET Core Web API for managing a pharmacy's drug catalogue, built on a **layered architecture**
with cross-cutting concerns implemented as **aspects** (Autofac interceptors) instead of being repeated in every method.

## Architecture

```
WebAPI  ──►  Business  ──►  DataAccess  ──►  SQL Server
              │                 │
              └──── Core ◄──────┘   (aspects, caching, logging, security, results)
Entities: Drug, User, OperationClaim
```

| Layer | Responsibility |
|---|---|
| `Entities` | Domain objects (`Drug`) and DTO marker interfaces |
| `DataAccess` | Generic EF Core repository (`EfEntityRepositoryBase`) and `PharmacyContext` |
| `Business` | Services (`DrugManager`), FluentValidation rules, user messages, Autofac module |
| `Core` | Reusable infrastructure: aspects, caching, logging, security, result types, middleware |
| `WebAPI` | Controllers, middleware pipeline, CORS for the Angular client |

## Cross-cutting concerns as attributes

```csharp
[LoggingAspect(typeof(SerilogLogger))]
[ValidationAspect(typeof(DrugValidator))]
[CacheRemoveAspect("IDrugService.Get")]
public IResult Add(Drug drug) { ... }

[CacheAspect]
public IDataResult<List<Drug>> GetAll() { ... }
```

| Concern | Implementation |
|---|---|
| Validation | `ValidationAspect` + FluentValidation (`DrugValidator`: name ≥ 2 chars, description ≤ 400 chars, price > 0) |
| Caching | `CacheAspect` / `CacheRemoveAspect` backed by **Redis** (`RedisCacheManager`) |
| Logging | `LoggingAspect` with **Serilog** (console, file, **Seq**) and request/response logging middleware |
| Errors | Global `ExceptionMiddleware` returning a consistent `ErrorDetails` payload |
| Security | JWT helper, salted password hashing (`HashingHelper`), claim-based operation roles |
| Results | `IResult` / `IDataResult<T>` with success and error variants |
| Search | Elasticsearch manager in `Core` (ready for indexing drugs) |

## Endpoints

| Method | Route | Description |
|---|---|---|
| GET | `/api/drugs/getall` | List all drugs (cached) |
| GET | `/api/drugs/getbyid?id={id}` | Get one drug |

`Add` and `Delete` are implemented in the business layer with validation, logging and cache invalidation.
Exposing them in the controller is next on the list.

## Tech

.NET 8 · ASP.NET Core Web API · Entity Framework Core · SQL Server (LocalDB) · Autofac + Castle DynamicProxy ·
FluentValidation · Redis · Serilog / Seq · Elasticsearch (NEST) · JWT

## Run

```bash
dotnet ef migrations add Initial --project DataAccess --startup-project WebAPI
dotnet ef database update --project DataAccess --startup-project WebAPI
dotnet run --project WebAPI
```

Local services used in development: Redis for caching, Seq at `http://localhost:5341` for logs and,
optionally, Elasticsearch at `http://localhost:9200`. Set the JWT signing key with user secrets instead of
`appsettings.json`:

```bash
dotnet user-secrets init --project WebAPI
dotnet user-secrets set "TokenOptions:SecurityKey" "<your-64+-character-key>" --project WebAPI
```

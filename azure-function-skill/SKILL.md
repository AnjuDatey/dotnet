---
name: new-azure-function
description: Scaffold production-ready Azure Function Apps (.NET 10, C# 13) from scratch using Clean Architecture (4 layers), EF Core 10 Code-First with PostgreSQL, JWT auth, RBAC, unified ApiResponse envelope, FluentValidation, AutoMapper, OpenTelemetry, AES-256 PII encryption, mandatory OpenAPI 3.0 + Swagger UI, and controlled EF Core migrations. For NEW Function Apps only - use add-function-dotnet10 for extending existing apps.
---

# Azure Function App - Complete .NET 10 Guideline

> SELF-CONTAINED.
> Every pattern required to generate a complete, compiling app is inlined below.

## When to Use
USE when a NEW module Function App is required and
`src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions/{ProjectName}.{Module}.Functions.csproj` does NOT exist.
Do NOT use for existing modules (use add-function-dotnet10).

## CRITICAL: Canonical Folder Structure (MANDATORY)
```
<root>/
  {ProjectName}.sln
  src/{ProjectName}.{Module}/
    {ProjectName}.{Module}.Domain/          {ProjectName}.{Module}.Domain.csproj
    {ProjectName}.{Module}.Application/      {ProjectName}.{Module}.Application.csproj
    {ProjectName}.{Module}.Infrastructure/   {ProjectName}.{Module}.Infrastructure.csproj
    {ProjectName}.{Module}.Functions/       {ProjectName}.{Module}.Functions.csproj
  tests/{ProjectName}.{Module}.Tests/        {ProjectName}.{Module}.Tests.csproj
```
RULES:
- Projects live under src/{ProjectName}.{Module}/. Tests under tests/.
- Module name ALWAYS {ProjectName}.{Module} (e.g. ProjectName.Module).
- Namespaces EQUAL project names.
- Correct:  src/{ProjectName}.{Module}/{ProjectName}.{Module}.Domain/Entities/{Entity}.cs
- WRONG:    {Module}FunctionApp/Domain/... (flat / single combined project)
- WRONG:    src/{Module}.Domain/... (missing {ProjectName}. module folder)
- NEVER create a single combined {FunctionAppName} project. ALWAYS 4 layer projects.

## Tech Stack
.NET 10 (Isolated), C# 13, EF Core 10, Azure Functions v4, Npgsql 10.x,
FluentValidation 11.x, AutoMapper 13.x, OpenTelemetry 2.x,
Microsoft.Azure.Functions.Worker.Extensions.OpenApi 2.x, IdentityModel.Tokens 8.x.

## NuGet - Infrastructure
```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.*" />
<PackageReference Include="EFCore.NamingConventions" Version="10.*" />
<PackageReference Include="Microsoft.IdentityModel.Tokens" Version="8.*" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="8.*" />
```
## NuGet - Functions
```xml
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="4.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.OpenApi" Version="2.*" />
<PackageReference Include="OpenTelemetry" Version="2.*" />
```
## Common csproj
```xml
<TargetFramework>net10.0</TargetFramework>
<LangVersion>13.0</LangVersion>
<Nullable>enable</Nullable>
<ImplicitUsings>enable</ImplicitUsings>
```
References: Application->Domain; Infrastructure->Domain; Functions->Application,Infrastructure.
Infrastructure NEVER referenced by Application/Domain.

## Layer Responsibilities
Domain: BaseEntity (CreatedAt/By, UpdatedAt/By, IsActive, IsDeleted, Version),
entities with private setters + Create/Update/Deactivate, Enums, Exceptions,
IUnitOfWork, IRepository + per-entity repo interfaces.
Application: Common/Responses (ApiResponse, ApiError, ValidationErrorDetail, PagedResponse),
Common/Constants (ErrorCodes, ErrorCategories, Roles), DTOs/{Entity}, Validators,
Services + Interfaces, Mappings/MappingProfile.
Infrastructure: ApplicationDbContext (+ design-time factory,
MigrationsAssembly({ProjectName}.{Module}.Infrastructure), UseSnakeCaseNamingConvention),
Configurations, Migrations (CLI only), Repositories, UnitOfWork, DependencyInjection.
Functions: {Entity}Function (HTTP triggers, zero business logic), Middleware order
Correlation -> JwtAuthentication -> ExceptionHandling, AuthorizationHelper,
Extensions, OpenApi/OpenApiConfigurationOptions, Program.cs, host.json, local.settings.json.

## MANDATORY: OpenAPI 3.0 + Swagger UI
Every app MUST:
1. Reference the OpenApi worker extension.
2. Provide OpenApiConfigurationOptions (V3, title, version, Bearer/JWT scheme).
3. Expose /api/swagger/ui and /api/openapi/v3.json.
4. Decorate EVERY endpoint with: [OpenApiOperation], [OpenApiSecurity Bearer JWT],
   [OpenApiParameter] (path/query), [OpenApiRequestBody] (POST/PUT),
   [OpenApiResponseWithBody] for EVERY status code (200/201,400,401,403,404,409)
   using ApiResponse<T>.
An app without reachable Swagger UI or missing [OpenApi*] on any endpoint is INCOMPLETE.
```csharp
public sealed class OpenApiConfigurationOptions : DefaultOpenApiConfigurationOptions {
  public override OpenApiInfo Info { get; set; } = new() {
    Version="1.0.0", Title="{ProjectName} {Module} API",
    Description="{ProjectName} {Module} module - Azure Functions (.NET 10)." };
  public override OpenApiVersionType OpenApiVersion { get; set; } = OpenApiVersionType.V3;
}
```

## API Response Contract
Success: { success:true, data, correlationId, timestamp }
Error:   { success:false, error:{ code, category, message, details[] }, correlationId, timestamp }

## System-Generated / Immutable / Unique / Audit (story-driven)
Honour design.json field flags:
- isSystemGenerated: service sets on create; NOT in Create DTO / request body.
- isImmutable: no update path; reject changes.
- defaultValue: service sets on create (e.g. Status=Draft, Version=1).
- isUnique + uniquenessScope: enforce uniqueness in service within scope; field-level error.
- allowedValues: model as enum or validate against configured list.
- audit requirement: generate AuditRecord (UserId, Timestamp, Action, EntityId)
  written in the SAME transaction; append-only, never overwritten.
```csharp
public static {Entity} Create(string name, string? description, Guid businessUnitId, string owner) {
  ArgumentException.ThrowIfNullOrWhiteSpace(name);
  ArgumentException.ThrowIfNullOrWhiteSpace(owner);
  return new {Entity} {
    {Entity}Id = GenerateId(),  // immutable system id
    Name = name, Description = description, BusinessUnitId = businessUnitId,
    Status = {Entity}Status.Draft, Owner = owner, Version = 1,
    CreatedBy = owner, CreatedAt = DateTime.UtcNow, IsActive = true };
}
```

## EF Core Migrations
NEVER call Database.Migrate() in app code; never hand-edit migrations.
```bash
dotnet ef migrations add Initial{Module}Schema \
  --project src/{ProjectName}.{Module}/{ProjectName}.{Module}.Infrastructure \
  --startup-project src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions \
  --output-dir Persistence/Migrations
```
All FKs DeleteBehavior.Restrict; global soft-delete filter.

## Build, Solution, Test Wiring
```bash
dotnet sln {ProjectName}.sln add \
  src/{ProjectName}.{Module}/{ProjectName}.{Module}.Domain/{ProjectName}.{Module}.Domain.csproj \
  src/{ProjectName}.{Module}/{ProjectName}.{Module}.Application/{ProjectName}.{Module}.Application.csproj \
  src/{ProjectName}.{Module}/{ProjectName}.{Module}.Infrastructure/{ProjectName}.{Module}.Infrastructure.csproj \
  src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions/{ProjectName}.{Module}.Functions.csproj \
  tests/{ProjectName}.{Module}.Tests/{ProjectName}.{Module}.Tests.csproj
dotnet restore src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions/{ProjectName}.{Module}.Functions.csproj
dotnet build   src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions/{ProjectName}.{Module}.Functions.csproj -c Release --no-restore
```
Tests use xUnit + FluentAssertions + Moq + EFCore.InMemory.

## Final LLM Instructions
1. .NET 10 / C# 13 / EF Core 10.
2. Canonical src/{ProjectName}.{Module}/{ProjectName}.{Module}.{Layer}/ structure - never flat, never combined.
3. Swagger UI + OpenAPI 3.0 MANDATORY, verified before completion.
4. Honour system-generated/immutable/unique/default/audit flags from design.json.
5. ApiResponse<T> everywhere; FluentValidationException on validation failures.
6. Migrations via CLI; never Database.Migrate() in app code.
7. Add all projects to {ProjectName}.sln; test project under tests/.
8. Output summary: File Path | Lines | Responsibility.

Version: 2.0 (.NET 10, self-contained)

---
name: add-function-dotnet10
description: Add a new entity/operation to an EXISTING {ProjectName} module Azure Function App (.NET 10). Minimal changes only, respects existing Clean Architecture, mandatory OpenAPI + Swagger UI, controlled EF Core migrations. Structure src/{ProjectName}.{Module}/{ProjectName}.{Module}.{Layer}.
---

# Add a New Function to an EXISTING {ProjectName} Module (.NET 10)

> Use when the module already exists. Touch only what is strictly necessary (YAGNI, DRY).

## Module Detection (deterministic)
EXISTING if present: src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions/{ProjectName}.{Module}.Functions.csproj
If absent -> STOP and use new-azure-function instead.

## Placeholders
{ProjectName}=project name (CMS); {Module}=module (QuestionBank) -> projects {ProjectName}.{Module}.{Layer}; {Entity}=new entity;
{entity-route}=kebab plural; {ParentEntity}=existing parent; {ExistingEntity}/{OperationName}=new op.
Inspect the existing module first; mirror conventions exactly.

## Pre-Flight Checklist
- [ ] src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions builds.
- [ ] ApplicationDbContext, UnitOfWork, Program.cs, middleware pipeline registered.
- [ ] ApiResponse<T>, ApiError, ValidationErrorDetail, ErrorCodes, ErrorCategories, Roles exist.
- [ ] IAuthorizationHelper, HttpRequestDataExtensions, OpenApiConfigurationOptions exist;
      Swagger UI wired (/api/swagger/ui, /api/openapi/v3.json).
- [ ] Known parent/FK and required roles per operation.

## MUST NOT Touch
Middleware (Correlation/JwtAuth/ExceptionHandling); OpenTelemetry/KeyVault/AppInsights wiring;
shared ApiResponse/ApiError contracts; existing Roles; host.json/local.settings.json structure;
unrelated entities.

## Where New Files Go
```
src/{ProjectName}.{Module}/
  {ProjectName}.{Module}.Domain/Entities/{Entity}.cs                            NEW
  {ProjectName}.{Module}.Domain/Interfaces/Repositories/I{Entity}Repository.cs  NEW
  {ProjectName}.{Module}.Application/DTOs/{Entity}/{Entity}Dto|Create|Update.cs NEW
  {ProjectName}.{Module}.Application/Validators/Create|Update{Entity}Validator.cs NEW
  {ProjectName}.{Module}.Application/Services/Interfaces/I{Entity}Service.cs     NEW
  {ProjectName}.{Module}.Application/Services/{Entity}Service.cs                 NEW
  {ProjectName}.{Module}.Application/Mappings/MappingProfile.cs                 MODIFIED (+2)
  {ProjectName}.{Module}.Infrastructure/Persistence/Configurations/{Entity}Configuration.cs NEW
  {ProjectName}.{Module}.Infrastructure/Persistence/ApplicationDbContext.cs     MODIFIED (DbSet+filter)
  {ProjectName}.{Module}.Infrastructure/Repositories/{Entity}Repository.cs      NEW
  {ProjectName}.{Module}.Infrastructure/UnitOfWork/UnitOfWork.cs               MODIFIED (+property)
  {ProjectName}.{Module}.Functions/Functions/{Entity}Function.cs              NEW (full [OpenApi*])
  {ProjectName}.{Module}.Functions/Program.cs                                  MODIFIED (+DI lines)
```

## Steps
1. Domain entity: sealed, BaseEntity, private setters, Create/Update/Deactivate; honour
   system-generated/immutable/default/unique flags; add nav collection on parent if related.
2. EF config + DbContext: {Entity}Configuration : IEntityTypeConfiguration; reuse schema;
   FK Restrict; add DbSet + HasQueryFilter(!IsDeleted).
2A. Migration via CLI:
```bash
dotnet ef migrations add Add{Entity} \
  --project src/{ProjectName}.{Module}/{ProjectName}.{Module}.Infrastructure \
  --startup-project src/{ProjectName}.{Module}/{ProjectName}.{Module}.Functions \
  --output-dir Persistence/Migrations
```
   Never hand-edit; never Database.Migrate() in app code; expand/contract if breaking;
   commit .cs + .Designer.cs + ModelSnapshot; attach idempotent SQL to PR.
3-7. I{Entity}Repository : IRepository; {Entity}Repository(ApplicationDbContext);
   add property+param to IUnitOfWork/UnitOfWork; DTOs (records); FluentValidation validators;
   {Entity}Service (IUnitOfWork, IMapper, ILogger, IValidator) returns ApiResponse<T>,
   throws FluentValidationException/NotFoundException; +2 CreateMap lines in MappingProfile.
8. {Entity}Function (new file): authHelper.AuthorizeAsync -> service -> CreateApiResponseAsync.
9. OpenAPI + Swagger UI (MANDATORY): every endpoint gets [OpenApiOperation],
   [OpenApiSecurity Bearer JWT], [OpenApiParameter], [OpenApiRequestBody] (POST/PUT),
   [OpenApiResponseWithBody] per status using ApiResponse<T>; endpoints MUST appear in Swagger UI.
10-12. Reuse existing Roles; add ErrorCodes ({ENTITY}_NOT_FOUND); Program.cs add only:
```csharp
services.AddScoped<I{Entity}Repository, {Entity}Repository>();
services.AddScoped<I{Entity}Service, {Entity}Service>();
```

## New Operation on Existing Entity
1 domain method + 1 service method + 1 [Function("{ExistingEntity}_{OperationName}")] on the
existing function class + [OpenApi*] + optional error code + RBAC. No new files, no DI changes.

## Best Practices
No raw SQL; no async void; no .Result/.Wait(); nullable + guard clauses; CancellationToken last;
private setters; soft delete only; no magic strings; SRP; DRY; YAGNI.

## Final LLM Instruction
Confirm module exists & mirror conventions; resolve placeholders from story; new entity=full
checklist, new op=minimal; never touch shared middleware/contracts; never Database.Migrate();
ApiResponse<T> everywhere; full [OpenApi*] + Swagger UI; reuse IEncryptionService for PII;
most-restrictive RBAC default if unspecified; output summary File Path | New/Modified | Responsibility.

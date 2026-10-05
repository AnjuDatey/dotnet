---
name: new-azure-function
description:  Scaffold production-ready Azure Function Apps (.NET 10, C# 13) from scratch using Clean Architecture (4 layers), Entity Framework Core 10 Code-First with PostgreSQL, JWT authentication, RBAC authorization, unified ApiResponse
 envelope, FluentValidation, AutoMapper, OpenTelemetry observability, AES-256 PII encryption, and enterprise-grade EF Core migration strategy (versioned, PR-reviewable, pipeline-controlled). For NEW Function Apps onlyΓÇöuse
 add-function-dotnet10 for extending existing apps.
---
# Azure Function App - Complete .NET 10 Implementation Guideline
## When to Use This Skill

✅ **Use this skill when:**
- User story requires a NEW Azure Function App
- Building from scratch (not adding to existing app)
- Need complete end-to-end scaffolding
- Code-First EF Core with PostgreSQL
- Clean Architecture (4 layers)

❌ **Do NOT use when:**
- Adding function to existing app
- Modifying existing entities
- Non-Azure Functions project

---

## .NET 10 Technology Stack

**CRITICAL: This guideline uses .NET 10

| Component | Version |
|-----------|---------|
| Runtime | **.NET 10** (Isolated Worker Model) |
| Language | **C# 13** |
| EF Core | **Entity Framework Core 10** |
| Azure Functions | v4 (Isolated) |
| PostgreSQL Provider | Npgsql.EntityFrameworkCore.PostgreSQL **10.x** |
| FluentValidation | 11.x |
| AutoMapper | 13.x |
| OpenTelemetry | 2.x |
| JWT Tokens | Microsoft.IdentityModel.Tokens **8.x** |

---

## Complete Reference Document

**Primary Source:** Your comprehensive `NewFunctionApp-LLM-Guideline.md` document

**Apply these .NET 10 overrides** to every section:

### 1. Technology Stack Overrides

```
WHERE the reference doc says:     USE instead:
----------------------------------  ---------------------------------
.NET 8                           →  .NET 10
C# 12                            →  C# 13
Entity Framework Core 8          →  Entity Framework Core 10
EF Core 8.x                      →  EF Core 10.x
```

### 2. NuGet Package Version Overrides

**Infrastructure Project:**
```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.*" />
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.*" />
<PackageReference Include="EFCore.NamingConventions" Version="10.*" />
<PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="10.*" />
<PackageReference Include="Microsoft.IdentityModel.Tokens" Version="8.*" />
<PackageReference Include="System.IdentityModel.Tokens.Jwt" Version="8.*" />
```

**Functions Project:**
```xml
<PackageReference Include="Microsoft.Azure.Functions.Worker" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http" Version="4.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.Http.AspNetCore" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Sdk" Version="2.*" />
<PackageReference Include="Microsoft.Azure.Functions.Worker.Extensions.OpenApi" Version="2.*" />
<PackageReference Include="Azure.Monitor.OpenTelemetry.AspNetCore" Version="2.*" />
<PackageReference Include="OpenTelemetry" Version="2.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.Http" Version="2.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.EntityFrameworkCore" Version="1.*" />
```

### 3. Project File (.csproj) Overrides

```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <LangVersion>13.0</LangVersion>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

### 4. Migration CLI Command Overrides

```bash
# Install/Update EF Core tools for .NET 10
dotnet tool install --global dotnet-ef --version 10.*
dotnet tool update --global dotnet-ef --version 10.*
```

### 5. C# 13 Language Features to Use

**Primary Constructor Enhancement:**
```csharp
// Preferred pattern in .NET 10 / C# 13
public class AssessmentService(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    ILogger<AssessmentService> logger) : IAssessmentService
{
    // No need to declare private fields manually - parameters become fields automatically
}
```

**Collection Expressions:**
```csharp
// C# 13 style
string[] allowedRoles = ["Manager", "Administrator"];
List<ValidationError> errors = [];
```

---

## Key Architecture Principles

(All principles from reference document apply - key highlights:)

### Clean Architecture Dependency Rules

```
Functions (Presentation Layer)
    ↓ depends on
Application Layer
    ↓ depends on
Domain Layer
    ↑
Infrastructure (also depends on Domain, implements interfaces)
```

**CRITICAL:** Infrastructure NEVER referenced by Application or Domain

### Code-First Migration Strategy

**NEVER:**
- ❌ Call `Database.Migrate()` or `MigrateAsync()` from `Program.cs`
- ❌ Hand-edit generated migration files
- ❌ Apply migrations from application startup

**ALWAYS:**
- ✅ Generate via `dotnet ef migrations add {MigrationName}`
- ✅ Apply via separate, controlled pipeline step
- ✅ Include idempotent SQL script in PR for review
- ✅ Use expand/contract pattern for breaking changes

### API Response Contract

**ALL endpoints return `ApiResponse<T>` envelope:**

✅ Success:
```json
{
  "success": true,
  "data": { "...": "..." },
  "correlationId": "...",
  "timestamp": "2025-01-02T..."
}
```

❌ Error:
```json
{
  "success": false,
  "error": {
    "code": "ENTITY_NOT_FOUND",
    "category": "NOT_FOUND",
    "message": "...",
    "details": []
  },
  "correlationId": "...",
  "timestamp": "2025-01-02T..."
}
```

---

## Design Workflow

When using this skill in a Design Agent:

1. **Read user story** from `story.json`
2. **Extract entities, fields, relationships** from acceptance criteria
3. **Infer business area** from feature area/tags
4. **Apply all patterns** from reference document
5. **Override versions** to .NET 10 per this skill
6. **Output** complete design as `complete-design.json`

---

## Code Generation Checklist

For each `{Entity}` from user story:

### Domain Layer ✅
- [ ] `{Entity}.cs` (inherit BaseEntity, private setters, factory method)
- [ ] Related enums (if any)
- [ ] `I{Entity}Repository.cs`

### Application Layer ✅
- [ ] `{Entity}Dto.cs`, `Create{Entity}Dto.cs`, `Update{Entity}Dto.cs`
- [ ] `Create{Entity}Validator.cs`, `Update{Entity}Validator.cs` (FluentValidation)
- [ ] `I{Entity}Service.cs` + `{Entity}Service.cs`
- [ ] Add error codes to `ErrorCodes.cs`

### Infrastructure Layer ✅
- [ ] `{Entity}Configuration.cs` (EF Core fluent config)
- [ ] `{Entity}Repository.cs`
- [ ] Update `ApplicationDbContext.cs` (add DbSet)
- [ ] Update `UnitOfWork.cs`
- [ ] **Generate migration** (via CLI, not hand-authored)

### Functions Layer ✅
- [ ] `{Entity}Function.cs` with full OpenAPI attributes
- [ ] RBAC role assignments per endpoint

---

## Critical Safety Rules

### 🔒 Security
- ✅ PII fields encrypted at application layer (AES-256-GCM)
- ✅ Secrets from Azure Key Vault (never hardcoded)
- ✅ PostgreSQL SSL Mode=Require
- ✅ JWT validation via middleware

### 🗄️ Database
- ✅ Code-First with EF Core 10
- ✅ Migrations via separate pipeline (not app startup)
- ✅ Soft delete (never `DbSet.Remove()`)
- ✅ All FK relationships: `DeleteBehavior.Restrict`

### 🎯 Coding Standards
- ✅ Async all the way (propagate `CancellationToken ct`)
- ✅ Private setters on entities
- ✅ Factory methods for entity creation
- ✅ No raw SQL (EF Core LINQ only)
- ✅ Structured logging (never string interpolation)

---

## Example: Complete Entity Pattern (.NET 10)

```csharp
// Domain/Entities/AssessmentSession.cs
namespace Assessment.Domain.Entities;

public sealed class AssessmentSession : BaseEntity
{
    public Guid AssessmentSessionId { get; private set; } = Guid.NewGuid();
    public string Title { get; private set; } = string.Empty;
    public string? Description { get; private set; }
    public AssessmentStatus Status { get; private set; }

    private AssessmentSession() { } // EF Core constructor

    public static AssessmentSession Create(string title, string? description, string createdBy)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(title);
        ArgumentException.ThrowIfNullOrWhiteSpace(createdBy);

        return new AssessmentSession
        {
            AssessmentSessionId = Guid.NewGuid(),
            Title = title,
            Description = description,
            Status = AssessmentStatus.Draft,
            CreatedBy = createdBy,
            CreatedAt = DateTime.UtcNow,
            IsActive = true
        };
    }

    public void Update(string title, string? description, string updatedBy)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(title);
        Title = title;
        Description = description;
        UpdatedBy = updatedBy;
        UpdatedAt = DateTime.UtcNow;
    }

    public void Deactivate(string updatedBy)
    {
        IsActive = false;
        IsDeleted = true;
        UpdatedBy = updatedBy;
        UpdatedAt = DateTime.UtcNow;
    }
}
```

---

## Migration Generation Example (.NET 10)

```bash
# Generate migration
dotnet ef migrations add AddAssessmentSessionTable \
  --project src/Assessment.Infrastructure \
  --startup-project src/Assessment.Functions \
  --output-dir Persistence/Migrations

# Generate idempotent SQL for PR review
dotnet ef migrations script --idempotent \
  --project src/Assessment.Infrastructure \
  --startup-project src/Assessment.Functions \
  --output migration.sql
```

---

## Final LLM Instructions

When generating code using this skill:

1. ✅ Use **.NET 10** runtime (not .NET 8)
2. ✅ Use **C# 13** language features
3. ✅ Use **EF Core 10** package versions
4. ✅ Follow **all patterns** from reference document
5. ✅ Apply **version overrides** from this skill
6. ✅ Generate **migration CLI commands** (don't hand-author)
7. ✅ Include **complete OpenAPI attributes** on every endpoint
8. ✅ Use **ApiResponse<T>** envelope for all responses
9. ✅ Never call `Database.Migrate()` from application code
10. ✅ Output summary table: File Path | Lines | Responsibility

---

**Version:** 1.0 (.NET 10 Edition)  
**Last Updated:** January 2025  
**Reference Document:** NewFunctionApp-LLM-Guideline.md (with .NET 10 overrides applied)

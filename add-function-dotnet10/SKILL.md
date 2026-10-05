---
name: add-function-dotnet10
description: Add a new entity/operation to an existing Azure Function App (.NET 10) - minimal changes only,
 respects existing architecture, Clean Architecture layers, Code-First EF Core migrations.
---
# 📘 LLM Code-Generation Guideline — Add a New Azure Function to an Existing Function App (Generic)

 **Purpose:** This is a **generic, domain-agnostic** specification for an LLM to add a **new entity + Azure Function** (or a new operation on an existing entity) **inside an already-scaffolded Azure Function App**. It assumes the target Function App already has its Clean Architecture skeleton, middleware pipeline, authentication, exception handling, logging, and DI bootstrap in place (built using the companion `NewFunctionApp-LLM-Guideline.md`).

**Scope:** Use this document when a user story requires:
 - A **new entity/sub-resource** added to an **existing** business-area Function App, OR
 - A **new operation/endpoint** added to an **existing** entity's Function class.

**Do NOT** use this document to scaffold a brand-new Function App, re-create middleware, re-register OpenTelemetry/Key Vault/JWT pipeline, or duplicate the DbContext/Program.cs bootstrap — those already exist. Touch only what is strictly necessary to add the new capability (YAGNI, DRY, minimal blast radius).

**Companion documents:**
 - If no Function App exists yet for the target business area, use `NewFunctionApp-LLM-Guideline.md` instead.
 - `AzureFunctionTesting-LLM-Guideline.md` — test strategy per layer.
>
**Note:** The PostgreSQL schema migration strategy (versioned EF Core migrations, PR review rules, and environment-specific apply rules) required for the new entity/field below is fully merged into this document as **§6A** — this is now a single, self-contained file.

---

## 📑 Table of Contents
1. [How To Use This Document](#1-how-to-use-this-document)
2. [Pre-Flight Checklist — Confirm Before Coding](#2-pre-flight-checklist--confirm-before-coding)
3. [What You MUST NOT Touch](#3-what-you-must-not-touch)
4. [Where New Files Go](#4-where-new-files-go)
5. [Step 1 — Domain Entity](#5-step-1--domain-entity)
6. [Step 2 — EF Core Configuration & DbContext Update](#6-step-2--ef-core-configuration--dbcontext-update)
6A. [PostgreSQL Schema Migration Strategy via EF Core](#6a-postgresql-schema-migration-strategy-via-ef-core)
7. [Step 3 — Repository](#7-step-3--repository)
8. [Step 4 — Unit of Work Update](#8-step-4--unit-of-work-update)
9. [Step 5 — DTOs & Validators](#9-step-5--dtos--validators)
10. [Step 6 — Service Layer](#10-step-6--service-layer)
11. [Step 7 — AutoMapper Profile Update](#11-step-7--automapper-profile-update)
12. [Step 8 — Azure Function Class (HTTP Triggers)](#12-step-8--azure-function-class-http-triggers)
13. [Step 9 — OpenAPI / Swagger Decoration](#13-step-9--openapi--swagger-decoration)
14. [Step 10 — RBAC Role Mapping](#14-step-10--rbac-role-mapping)
15. [Step 11 — Error Codes Update](#15-step-11--error-codes-update)
16. [Step 12 — Dependency Injection Registration](#16-step-12--dependency-injection-registration)
17. [Adding a Single New Operation to an Existing Entity](#17-adding-a-single-new-operation-to-an-existing-entity)
18. [PII & Encryption Considerations](#18-pii--encryption-considerations)
19. [Logging Considerations](#19-logging-considerations)
20. [Code Contracts & Best Practices (Reminder)](#20-code-contracts--best-practices-reminder)
21. [Complete Add-Function Checklist](#21-complete-add-function-checklist)
22. [Final LLM Instruction](#22-final-llm-instruction)

---

## 1. How To Use This Document

This guideline uses **placeholders**. Resolve them from the user story before generating code:

| Placeholder | Meaning | Example |
|---|---|---|
| `{BusinessArea}` | The existing business-area Function App/solution this entity belongs to | `Assessment`, `Notifications` |
| `{Entity}` | The new domain entity name, PascalCase singular | `AssessmentFeedback`, `NotificationChannel` |
| `{entity-route}` | kebab-case plural route segment | `assessment-feedbacks`, `notification-channels` |
| `{ParentEntity}` | Existing parent entity this new entity relates to, if any | `Assessment`, `Notification` |
| `{ExistingEntity}` | An existing entity already in the app, when adding a new operation to it | `Assessment` |
| `{OperationName}` | A new operation/endpoint name beyond standard CRUD | `Approve`, `Resend`, `Archive` |

> **Instruction to LLM:** Before writing any code, inspect the existing solution structure (`src/{BusinessArea}.Domain`, `.Application`, `.Infrastructure`, `.Functions`) to confirm naming conventions, schema name, Roles already defined, and the exact shape of existing entities/services/functions. Mirror those conventions exactly — consistency with existing code takes priority over this document's exact wording.

---

## 2. Pre-Flight Checklist — Confirm Before Coding

Before adding a new entity or operation, verify:

- [ ] The target Function App (`{BusinessArea}.Functions`) already exists and builds successfully.
- [ ] `ApplicationDbContext`, `UnitOfWork`, `Program.cs`, middleware pipeline (`CorrelationMiddleware` → `JwtAuthenticationMiddleware` → `ExceptionHandlingMiddleware`) are already registered and functioning.
- [ ] `ApiResponse<T>`, `ApiError`, `ValidationErrorDetail`, `ErrorCodes`, `ErrorCategories`, `Roles` already exist in `{BusinessArea}.Application.Common`.
- [ ] `IAuthorizationHelper`, `HttpRequestDataExtensions`, `OpenApiConfigurationOptions` already exist in `{BusinessArea}.Functions`.
- [ ] You know which existing entity (if any) the new entity has a parent/FK relationship to.
- [ ] You know the exact roles/permissions required per operation from the user story.

If any of the above is missing, **stop** — the Function App is not fully scaffolded; use `NewFunctionApp-LLM-Guideline.md` to complete the bootstrap first, or ask the user for clarification.

---

## 3. What You MUST NOT Touch

When only adding a function/entity to an existing app, do **not**:

- Re-create or modify `CorrelationMiddleware.cs`, `JwtAuthenticationMiddleware.cs`, `ExceptionHandlingMiddleware.cs` unless the user story explicitly changes cross-cutting behavior.
- Re-register OpenTelemetry, Key Vault, or Application Insights wiring in `Program.cs` — only add the new service/repository registrations.
- Duplicate `ApiResponse<T>`, `ApiError`, `ValidationErrorDetail`, or any shared response/error contract class.
- Re-define `Roles` constants that already exist — only add new role constants if the story introduces a genuinely new role.
- Change the `host.json` / `local.settings.json` structure beyond adding a new config key if the new entity genuinely requires one (e.g., a new external integration secret).
- Touch unrelated entities' files.

---

## 4. Where New Files Go

Add new files into the **existing** solution's folder structure, following the pattern already established:

```
src/
├── {BusinessArea}.Domain/
│   ├── Entities/{Entity}.cs                         ← NEW
│   └── Interfaces/Repositories/I{Entity}Repository.cs ← NEW
│
├── {BusinessArea}.Application/
│   ├── DTOs/{Entity}/{Entity}Dto.cs                  ← NEW
│   ├── DTOs/{Entity}/Create{Entity}Dto.cs             ← NEW
│   ├── DTOs/{Entity}/Update{Entity}Dto.cs             ← NEW
│   ├── Validators/Create{Entity}Validator.cs          ← NEW
│   ├── Validators/Update{Entity}Validator.cs          ← NEW
│   ├── Services/Interfaces/I{Entity}Service.cs        ← NEW
│   ├── Services/{Entity}Service.cs                    ← NEW
│   └── Mappings/MappingProfile.cs                     ← MODIFIED (add mapping lines)
│
├── {BusinessArea}.Infrastructure/
│   ├── Persistence/Configurations/{Entity}Configuration.cs ← NEW
│   ├── Persistence/ApplicationDbContext.cs            ← MODIFIED (add DbSet + query filter)
│   ├── Repositories/{Entity}Repository.cs             ← NEW
│   └── UnitOfWork/UnitOfWork.cs                       ← MODIFIED (add property + ctor param)
│
└── {BusinessArea}.Functions/
    ├── Functions/{Entity}Function.cs                  ← NEW
    └── Program.cs                                     ← MODIFIED (add DI registration lines only)
```

---

## 5. Step 1 — Domain Entity

```csharp
// {BusinessArea}.Domain/Entities/{Entity}.cs
namespace {BusinessArea}.Domain.Entities;

public sealed class {Entity} : BaseEntity
{
    public Guid {Entity}Id { get; private set; } = Guid.NewGuid();

    // FK to existing parent entity, if applicable:
    public Guid {ParentEntity}Id { get; private set; }
    public {ParentEntity} {ParentEntity} { get; private set; } = null!;

    // Fields per user story:
    public string {Field1} { get; private set; } = string.Empty;
    public string? {Field2} { get; private set; }

    private {Entity}() { }  // EF Core ctor

    public static {Entity} Create(/* required params */ string createdBy)
    {
        // Guard clauses for required fields
        return new {Entity}
        {
            {Entity}Id = Guid.NewGuid(),
            CreatedBy = createdBy,
            CreatedAt = DateTime.UtcNow,
            IsActive = true
        };
    }

    public void Update(/* updatable params */ string updatedBy)
    {
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

> If `{Entity}` relates to an existing parent entity, also add a navigation collection on the **existing** parent entity class (e.g., `IReadOnlyCollection<{Entity}> {Entity}s` + backing private list) — this is the one acceptable, minimal edit to an existing entity file.

---

## 6. Step 2 — EF Core Configuration & DbContext Update

```csharp
// {BusinessArea}.Infrastructure/Persistence/Configurations/{Entity}Configuration.cs
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {BusinessArea}.Domain.Entities;

namespace {BusinessArea}.Infrastructure.Persistence.Configurations;

public class {Entity}Configuration : IEntityTypeConfiguration<{Entity}>
{
    public void Configure(EntityTypeBuilder<{Entity}> builder)
    {
        builder.ToTable("{Entity}s", "{existing-schema}");  // reuse the SAME schema already used by the app
        builder.HasKey(e => e.{Entity}Id);
        builder.Property(e => e.{Entity}Id).ValueGeneratedNever();
        builder.Property(e => e.CreatedBy).IsRequired().HasMaxLength(256);
        builder.Property(e => e.UpdatedBy).HasMaxLength(256);

        // Field constraints per user story

        // FK relationship, if any — always Restrict:
        // builder.HasOne(e => e.{ParentEntity})
        //        .WithMany(p => p.{Entity}s)
        //        .HasForeignKey(e => e.{ParentEntity}Id)
        //        .OnDelete(DeleteBehavior.Restrict);
    }
}
```

**Modify `ApplicationDbContext.cs`** — add only these two lines, do not touch anything else:

```csharp
public DbSet<{Entity}> {Entity}s => Set<{Entity}>();   // add to existing DbSet list

// inside OnModelCreating, alongside existing filters:
modelBuilder.Entity<{Entity}>().HasQueryFilter(e => !e.IsDeleted);
```

`ApplyConfigurationsFromAssembly` already picks up the new `{Entity}Configuration` automatically — no change needed there.

> Generate an EF Core migration (`dotnet ef migrations add Add{Entity}`) after this step — do not hand-edit migration files. Follow §6A below for naming, PR review requirements including the generated idempotent SQL script, and environment-specific apply rules. If the new field is a required (`NOT NULL`) column being added to an already-populated table, or involves a rename, use the expand/contract pattern in §6A.9 instead of a single breaking migration.

---

## 6A. PostgreSQL Schema Migration Strategy via EF Core

> **Purpose:** Enterprise best practice for managing **PostgreSQL schema changes** using **versioned, PR-reviewable EF Core Code-First migrations**. It enforces **decoupling "deploy code" from "apply schema change"**, so schema changes go live through a controlled, auditable gate — never as a side effect of a Function App cold start. The target app's `ApplicationDbContextFactory` design-time factory and `.MigrationsAssembly("{BusinessArea}.Infrastructure")` configuration already exist (scaffolded via `NewFunctionApp-LLM-Guideline.md` §7A) — this section only concerns generating, reviewing, and applying the **new** migration for `{Entity}`.

**Placeholders used in this section:**

| Placeholder | Meaning | Example |
|---|---|---|
| `{MigrationName}` | PascalCase description of the schema change | `Add{Entity}Table`, `AddStatusColumnTo{Entity}` |

> **Instruction to LLM:** Never generate a migration for a change not explicitly requested in the user story. Never hand-write or hand-edit the generated migration `.cs`/`.Designer.cs` files — always produce them via the `dotnet ef migrations add` command workflow described in §6A.3, then review/adjust only the `Up()`/`Down()` method bodies if a data-fix step is required.

### 6A.1 Core Principle — Decouple Deploy from Migrate

**Deploying application code** (publishing the Function App) and **applying a database schema change** (running a migration) are **two independent, separately-triggered operations**, each with its own approval gate:

```
Code Deploy Pipeline                 Schema Migration Pipeline
────────────────────────                 ──────────────────────────
1. Build                              1. Build (same artifact, different step)
2. Run unit/integration tests         2. Generate idempotent SQL script
3. Publish Function App binaries      3. Manual/automated approval gate
4. Swap/deploy to slot                4. Apply script against target DB
                                       5. Verify + smoke test
```

Function Apps can scale out to **multiple concurrent instances** on cold start — if `Database.MigrateAsync()` were called from `Program.cs`, multiple instances could race to apply the same migration simultaneously, causing lock contention or partial schema states. Decoupling allows a reviewer to approve the exact SQL that will run in production **before** it runs, the same discipline applied to code via PR review.

### 6A.2 Environment Rules — How Migrations Get Applied

| Environment | How migrations get applied | Trigger |
|---|---|---|
| **Local dev** | Developer runs `dotnet ef database update` manually after pulling or generating new migrations | Manual, on-demand |
| **CI (build/test)** | Pipeline applies migrations against an **ephemeral/test DB** (Testcontainers or a disposable CI DB instance), then runs integration tests against that schema | Automatic, every PR/build |
| **Staging** | Applied via a **separate, controlled pipeline stage** — after build succeeds, before/alongside code deploy to staging | Automatic, gated on successful build + tests |
| **Production** | Applied via a **separate, controlled pipeline stage** with a **mandatory manual approval gate** — never bundled into the Function App's deployment or cold-start path | Manual approval required |

> **Instruction to LLM:** Never emit the migration-apply step inside the Function App's build-and-deploy job, and never as code inside `Program.cs` that runs on every cold start.

### 6A.3 Generating the Migration — Local Dev Workflow

```bash
# 1. Entity/configuration changes already made in {BusinessArea}.Infrastructure / .Domain (§5–§6 above)

# 2. Generate the migration
dotnet ef migrations add Add{Entity} \
  --project src/{BusinessArea}.Infrastructure \
  --startup-project src/{BusinessArea}.Functions \
  --output-dir Persistence/Migrations

# 3. Review the generated files (see §6A.5 PR checklist) BEFORE applying locally

# 4. Apply to your LOCAL dev database only
dotnet ef database update \
  --project src/{BusinessArea}.Infrastructure \
  --startup-project src/{BusinessArea}.Functions

# 5. Commit the migration files as part of the same PR as the entity/config change
```

> **Instruction to LLM:** Emit the exact `dotnet ef migrations add Add{Entity}` command for the user/CI to run — do NOT hand-author the migration `.cs` file content yourself.

### 6A.4 Migration Naming Convention

```
PascalCase, verb-first, describes the schema intent:

Add{Entity}Table                         e.g., AddAssessmentFeedbackTable
Add{Field}ColumnTo{Entity}                e.g., AddStatusColumnToAssessmentFeedback
AddIndexOn{Entity}{Field}                 e.g., AddIndexOnNotificationChannelEmail
AddForeignKey{Entity}To{ParentEntity}     e.g., AddForeignKeyFeedbackToAssessment
Rename{OldField}To{NewField}On{Entity}    e.g., RenameEmailAddressToEmailOnAssessor
Drop{Field}From{Entity}                   e.g., DropLegacyStatusFromAssessment (only after expand/contract, §6A.9)
```

### 6A.5 Migration File Review Rules — PR Checklist

Every PR containing the new migration **must** include:

- [ ] The migration `.cs` file, `.Designer.cs` file, and the updated `ApplicationDbContextModelSnapshot.cs` — all three, always committed together.
- [ ] The output of `dotnet ef migrations script --idempotent` for the new migration, attached to the PR description so reviewers see the **actual SQL**, not just the C# `migrationBuilder` calls.
- [ ] Confirmation the migration was tested locally via `dotnet ef database update` against a real (or Testcontainers) Postgres instance — not just InMemory.
- [ ] For destructive changes — explicit reviewer sign-off and a documented rollback/backfill plan (§6A.8, §6A.9).
- [ ] No unrelated schema drift in the diff (only the `{Entity}` change from this PR's user story).
- [ ] Migration name follows §6A.4 convention.
- [ ] If adding a new required (`NOT NULL`) column to an existing populated table, a safe default value or the expand/contract approach (§6A.9) is used — never a bare `NOT NULL` with no default against a non-empty table in production.

### 6A.6 CI Pipeline — Build/Test Stage

```
CI Build/Test Stage:
1. dotnet build
2. Start ephemeral Postgres (Testcontainers, or a disposable CI service container)
3. dotnet ef database update --project src/{BusinessArea}.Infrastructure \
     --startup-project src/{BusinessArea}.Functions \
     --connection "<ephemeral-ci-db-connection-string>"
4. dotnet test   (unit + integration tests, including repository integration tests)
5. Tear down ephemeral DB container
```

This validates the migration applies cleanly but does **not** touch staging/production.

### 6A.7 Staging / Production — Controlled Migration Pipeline Step

```
Schema Migration Job (Staging):
  trigger: automatic, after Build/Test stage succeeds
  steps:
    1. dotnet ef migrations script --idempotent -o migration-staging.sql
         --project src/{BusinessArea}.Infrastructure --startup-project src/{BusinessArea}.Functions
    2. Apply migration-staging.sql against the Staging Postgres instance (via psql or a dedicated
         DB-migration task — NOT via the Function App)
    3. Run smoke tests against Staging

Schema Migration Job (Production):
  trigger: MANUAL APPROVAL required
  steps:
    1. dotnet ef migrations script --idempotent --from <last-known-deployed-migration> \
         -o migration-production.sql --project src/{BusinessArea}.Infrastructure \
         --startup-project src/{BusinessArea}.Functions
    2. Reviewer/approver inspects migration-production.sql as part of the approval gate
    3. Backup the production database
    4. Apply migration-production.sql against the Production Postgres instance
    5. Run post-migration smoke tests / health checks
    6. Only THEN proceed with (or confirm) the Function App code deployment
```

The Function App's deployment pipeline (publish binaries, swap slots) has **no knowledge of or dependency on** the migration job succeeding at the SQL level beyond "schema must already be compatible." Use backward-compatible migrations (§6A.8/§6A.9) so the OLD code version can still run correctly during the brief window between schema migration and code deployment completion.

### 6A.8 Breaking vs Non-Breaking Schema Changes

| Change Type | Breaking? | Safe to apply directly? |
|---|---|---|
| Add new nullable column | No | ✅ Yes |
| Add new table | No | ✅ Yes |
| Add new index (non-unique) | No | ✅ Yes (consider `CONCURRENTLY` for large tables) |
| Add new required (`NOT NULL`) column to populated table | **Yes** | ❌ No — use expand/contract (§6A.9) |
| Rename a column | **Yes** | ❌ No — old code breaks immediately; use expand/contract |
| Drop a column | **Yes** | ❌ No — only after confirming no code references it |
| Change a column's data type (narrowing) | **Yes** | ❌ No — risk of data truncation/loss |
| Add a unique constraint to existing populated column | **Yes**, if duplicates exist | ⚠️ Validate data first |
| Add FK with `OnDelete(Restrict)` | Usually No | ✅ Yes, but validate orphan rows first |

Since `{Entity}` is typically a **brand-new table** being added to an existing app, most additions here are non-breaking. Apply the breaking-change treatment only when adding a column/FK to an entity that already has production data (e.g., the §17 "new operation on existing entity" scenario touching an existing table).

### 6A.9 Expand/Contract Pattern for Breaking Changes

For any breaking change (rename, new required column, drop column) on an **existing, populated** entity, split into multiple migrations across multiple deploys:

```
Step 1 (Expand)   — Migration: Add{NewField}ColumnTo{Entity} (nullable, additive)
                     Deploy code that WRITES to both {OldField} and {NewField} (dual-write),
                     but still READS from {OldField}.

Step 2 (Backfill) — One-off data migration script (NOT an EF migration) to populate
                     {NewField} for existing rows from {OldField}.

Step 3 (Switch)   — Deploy code that READS from {NewField} instead of {OldField}.
                     Continue dual-write for one more release cycle as a safety net.

Step 4 (Contract) — Migration: Drop{OldField}From{Entity}, once confident no code
                     path reads/writes the old column and enough time has passed
                     for rollback safety.
```

> **Instruction to LLM:** When a user story requests a rename or a new required field on an existing populated entity, always propose the expand/contract sequence above (as separate PRs/migrations) rather than a single breaking migration.

### 6A.10 Rollback & Secrets Handling

- Generate a down-script when needed: `dotnet ef migrations script {TargetMigration} {CurrentMigration} --output rollback.sql`.
- **Prefer forward-fix over rollback** once data has been written against the new schema.
- Flag any non-reversible migration (`DropColumn`, `TRUNCATE`) explicitly in the PR with a documented recovery plan.
- Always take a point-in-time backup immediately before applying a production migration script.
- The migration pipeline job uses a **dedicated, scoped DB credential** (DDL privileges) distinct from the Function App's own runtime connection string (which should only need DML privileges) — principle of least privilege.

### 6A.11 Migration Checklist for This Entity

- [ ] Migration generated via `dotnet ef migrations add Add{Entity}` (never hand-authored).
- [ ] Migration name follows §6A.4 convention.
- [ ] Migration classified as breaking or non-breaking (§6A.8); expand/contract plan documented if breaking (§6A.9).
- [ ] Migration `.cs`, `.Designer.cs`, and updated `ModelSnapshot.cs` committed together in the PR.
- [ ] `dotnet ef migrations script --idempotent` output attached to PR for reviewer visibility.
- [ ] Migration applied and tested locally against a real/Testcontainers Postgres instance.
- [ ] CI build/test stage applies migration to ephemeral DB and passes integration tests.
- [ ] No `Database.Migrate()`/`MigrateAsync()` call added anywhere in `Program.cs` or Function trigger code.
- [ ] Staging/production migration jobs apply the idempotent script via separate, scoped-credential pipeline steps (§6A.7), never via the Function App.

---

## 7. Step 3 — Repository

```csharp
// {BusinessArea}.Domain/Interfaces/Repositories/I{Entity}Repository.cs
namespace {BusinessArea}.Domain.Interfaces.Repositories;

public interface I{Entity}Repository : IRepository<{Entity}, Guid>
{
    // Add entity-specific query methods only if required by the user story, e.g.:
    // Task<IReadOnlyList<{Entity}>> GetBy{ParentEntity}IdAsync(Guid {parentEntity}Id, CancellationToken ct = default);
}
```

```csharp
// {BusinessArea}.Infrastructure/Repositories/{Entity}Repository.cs
using {BusinessArea}.Domain.Entities;
using {BusinessArea}.Domain.Interfaces.Repositories;
using {BusinessArea}.Infrastructure.Persistence;

namespace {BusinessArea}.Infrastructure.Repositories;

public class {Entity}Repository(ApplicationDbContext context)
    : Repository<{Entity}, Guid>(context), I{Entity}Repository
{
    // Implement any entity-specific query methods declared on the interface
}
```

---

## 8. Step 4 — Unit of Work Update

Modify the **existing** `IUnitOfWork` and `UnitOfWork` — add one property and constructor parameter only:

```csharp
// IUnitOfWork.cs — add this line to the existing interface:
I{Entity}Repository {Entity}s { get; }
```

```csharp
// UnitOfWork.cs — add this parameter to the existing primary constructor and expose it:
public class UnitOfWork(ApplicationDbContext context, /* existing repos, */ I{Entity}Repository {entity}s) : IUnitOfWork
{
    public I{Entity}Repository {Entity}s { get; } = {entity}s;
    // existing members unchanged
}
```

---

## 9. Step 5 — DTOs & Validators

```csharp
// {BusinessArea}.Application/DTOs/{Entity}/{Entity}Dto.cs
namespace {BusinessArea}.Application.DTOs.{Entity};
public sealed record {Entity}Dto(Guid {Entity}Id, /* fields */ bool IsActive, DateTime CreatedAt);

// {BusinessArea}.Application/DTOs/{Entity}/Create{Entity}Dto.cs
public sealed record Create{Entity}Dto(/* required fields per user story */);

// {BusinessArea}.Application/DTOs/{Entity}/Update{Entity}Dto.cs
public sealed record Update{Entity}Dto(/* updatable fields per user story */);
```

```csharp
// {BusinessArea}.Application/Validators/Create{Entity}Validator.cs
using FluentValidation;
using {BusinessArea}.Application.DTOs.{Entity};

namespace {BusinessArea}.Application.Validators;

public class Create{Entity}Validator : AbstractValidator<Create{Entity}Dto>
{
    public Create{Entity}Validator()
    {
        // RuleFor(x => x.{Field}).NotEmpty().MaximumLength(200);
    }
}

// Update{Entity}Validator.cs follows the same pattern for Update{Entity}Dto
```

---

## 10. Step 6 — Service Layer

```csharp
// {BusinessArea}.Application/Services/Interfaces/I{Entity}Service.cs
using {BusinessArea}.Application.DTOs.{Entity};
using {BusinessArea}.Application.Common.Responses;

namespace {BusinessArea}.Application.Services.Interfaces;

public interface I{Entity}Service
{
    Task<ApiResponse<IReadOnlyList<{Entity}Dto>>> GetAllAsync(CancellationToken ct = default);
    Task<ApiResponse<{Entity}Dto>> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<ApiResponse<{Entity}Dto>> CreateAsync(Create{Entity}Dto dto, string createdBy, CancellationToken ct = default);
    Task<ApiResponse<{Entity}Dto>> UpdateAsync(Guid id, Update{Entity}Dto dto, string updatedBy, CancellationToken ct = default);
    Task<ApiResponse<bool>> DeleteAsync(Guid id, string deletedBy, CancellationToken ct = default);
}
```

```csharp
// {BusinessArea}.Application/Services/{Entity}Service.cs
using AutoMapper;
using FluentValidation;
using Microsoft.Extensions.Logging;
using {BusinessArea}.Application.Common.Constants;
using {BusinessArea}.Application.Common.Responses;
using {BusinessArea}.Application.DTOs.{Entity};
using {BusinessArea}.Application.Services.Interfaces;
using {BusinessArea}.Domain.Entities;
using {BusinessArea}.Domain.Exceptions;
using {BusinessArea}.Domain.Interfaces;

namespace {BusinessArea}.Application.Services;

public class {Entity}Service(
    IUnitOfWork unitOfWork,
    IMapper mapper,
    ILogger<{Entity}Service> logger,
    IValidator<Create{Entity}Dto> createValidator,
    IValidator<Update{Entity}Dto> updateValidator)
    : I{Entity}Service
{
    public async Task<ApiResponse<IReadOnlyList<{Entity}Dto>>> GetAllAsync(CancellationToken ct = default)
    {
        logger.LogDebug("Fetching all {Entity} records");
        var entities = await unitOfWork.{Entity}s.GetAllAsync(ct);
        var dtos = mapper.Map<IReadOnlyList<{Entity}Dto>>(entities);
        return ApiResponse<IReadOnlyList<{Entity}Dto>>.Ok(dtos);
    }

    public async Task<ApiResponse<{Entity}Dto>> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogDebug("Fetching {Entity} {Id}", id);
        var entity = await unitOfWork.{Entity}s.GetByIdAsync(id, ct)
                     ?? throw new NotFoundException(nameof({Entity}), id, ErrorCodes.{Entity}NotFound);
        return ApiResponse<{Entity}Dto>.Ok(mapper.Map<{Entity}Dto>(entity));
    }

    public async Task<ApiResponse<{Entity}Dto>> CreateAsync(Create{Entity}Dto dto, string createdBy, CancellationToken ct = default)
    {
        var result = await createValidator.ValidateAsync(dto, ct);
        if (!result.IsValid) throw new FluentValidationException(result.Errors);

        var entity = {Entity}.Create(/* map dto fields */ createdBy);
        await unitOfWork.{Entity}s.AddAsync(entity, ct);
        await unitOfWork.SaveChangesAsync(ct);

        logger.LogInformation("{Entity} created. Id: {EntityId}, By: {CreatedBy}", entity.{Entity}Id, createdBy);
        return ApiResponse<{Entity}Dto>.Ok(mapper.Map<{Entity}Dto>(entity), "{Entity} created successfully.");
    }

    public async Task<ApiResponse<{Entity}Dto>> UpdateAsync(Guid id, Update{Entity}Dto dto, string updatedBy, CancellationToken ct = default)
    {
        var result = await updateValidator.ValidateAsync(dto, ct);
        if (!result.IsValid) throw new FluentValidationException(result.Errors);

        var entity = await unitOfWork.{Entity}s.GetByIdAsync(id, ct)
                     ?? throw new NotFoundException(nameof({Entity}), id, ErrorCodes.{Entity}NotFound);

        entity.Update(/* map dto fields */ updatedBy);
        await unitOfWork.{Entity}s.UpdateAsync(entity, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return ApiResponse<{Entity}Dto>.Ok(mapper.Map<{Entity}Dto>(entity), "{Entity} updated successfully.");
    }

    public async Task<ApiResponse<bool>> DeleteAsync(Guid id, string deletedBy, CancellationToken ct = default)
    {
        var exists = await unitOfWork.{Entity}s.ExistsAsync(id, ct);
        if (!exists) throw new NotFoundException(nameof({Entity}), id, ErrorCodes.{Entity}NotFound);

        await unitOfWork.{Entity}s.DeleteAsync(id, ct);
        await unitOfWork.SaveChangesAsync(ct);

        return ApiResponse<bool>.Ok(true, "{Entity} deleted successfully.");
    }
}
```

---

## 11. Step 7 — AutoMapper Profile Update

Modify the **existing** `MappingProfile.cs` — add two lines only, inside the existing constructor:

```csharp
CreateMap<{Entity}, {Entity}Dto>();
CreateMap<Create{Entity}Dto, {Entity}>();
```

---

## 12. Step 8 — Azure Function Class (HTTP Triggers)

```csharp
// {BusinessArea}.Functions/Functions/{Entity}Function.cs
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using System.Net;
using {BusinessArea}.Application.Services.Interfaces;
using {BusinessArea}.Application.DTOs.{Entity};
using {BusinessArea}.Functions.Authorization;
using {BusinessArea}.Functions.Extensions;
using {BusinessArea}.Application.Common.Constants;

namespace {BusinessArea}.Functions.Functions;

public class {Entity}Function(
    I{Entity}Service service,
    ILogger<{Entity}Function> logger,
    IAuthorizationHelper authHelper)
{
    [Function("{Entity}_GetAll")]
    public async Task<HttpResponseData> GetAllAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "{entity-route}")] HttpRequestData req,
        FunctionContext context, CancellationToken ct)
    {
        await authHelper.AuthorizeAsync(req, context, [/* roles */]);
        var result = await service.GetAllAsync(ct);
        return await req.CreateApiResponseAsync(result, HttpStatusCode.OK, context);
    }

    [Function("{Entity}_GetById")]
    public async Task<HttpResponseData> GetByIdAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "get", Route = "{entity-route}/{id:guid}")] HttpRequestData req,
        Guid id, FunctionContext context, CancellationToken ct)
    {
        await authHelper.AuthorizeAsync(req, context, [/* roles */]);
        var result = await service.GetByIdAsync(id, ct);
        return await req.CreateApiResponseAsync(result, HttpStatusCode.OK, context);
    }

    [Function("{Entity}_Create")]
    public async Task<HttpResponseData> CreateAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post", Route = "{entity-route}")] HttpRequestData req,
        FunctionContext context, CancellationToken ct)
    {
        await authHelper.AuthorizeAsync(req, context, [/* roles */]);
        var dto = await req.ReadFromJsonAsync<Create{Entity}Dto>()
                  ?? throw new Domain.Exceptions.ValidationException("Request body is required");
        var createdBy = context.GetUserIdentity();
        var result = await service.CreateAsync(dto, createdBy, ct);
        return await req.CreateApiResponseAsync(result, HttpStatusCode.Created, context);
    }

    [Function("{Entity}_Update")]
    public async Task<HttpResponseData> UpdateAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "put", Route = "{entity-route}/{id:guid}")] HttpRequestData req,
        Guid id, FunctionContext context, CancellationToken ct)
    {
        await authHelper.AuthorizeAsync(req, context, [/* roles */]);
        var dto = await req.ReadFromJsonAsync<Update{Entity}Dto>()
                  ?? throw new Domain.Exceptions.ValidationException("Request body is required");
        var updatedBy = context.GetUserIdentity();
        var result = await service.UpdateAsync(id, dto, updatedBy, ct);
        return await req.CreateApiResponseAsync(result, HttpStatusCode.OK, context);
    }

    [Function("{Entity}_Delete")]
    public async Task<HttpResponseData> DeleteAsync(
        [HttpTrigger(AuthorizationLevel.Anonymous, "delete", Route = "{entity-route}/{id:guid}")] HttpRequestData req,
        Guid id, FunctionContext context, CancellationToken ct)
    {
        await authHelper.AuthorizeAsync(req, context, [/* roles, usually Administrator-only */]);
        var deletedBy = context.GetUserIdentity();
        var result = await service.DeleteAsync(id, deletedBy, ct);
        return await req.CreateApiResponseAsync(result, HttpStatusCode.OK, context);
    }
}
```

> This is a brand-new file — it does not require touching any other Function class in the app.

---

## 13. Step 9 — OpenAPI / Swagger Decoration

Decorate every endpoint on the new `{Entity}Function` with the same `[OpenApi*]` attribute pattern already used elsewhere in the app:

```csharp
using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Attributes;
using Microsoft.Azure.WebJobs.Extensions.OpenApi.Core.Enums;
using Microsoft.OpenApi.Models;

[Function("{Entity}_GetAll")]
[OpenApiOperation(operationId: "{Entity}_GetAll", tags: ["{Entity}"], Summary = "Get all {Entity}")]
[OpenApiSecurity("Bearer", SecuritySchemeType.Http, Scheme = OpenApiSecuritySchemeType.Bearer, BearerFormat = "JWT")]
[OpenApiResponseWithBody(HttpStatusCode.OK, "application/json", typeof(ApiResponse<IReadOnlyList<{Entity}Dto>>))]
[OpenApiResponseWithBody(HttpStatusCode.Unauthorized, "application/json", typeof(ApiResponse<object>))]
[OpenApiResponseWithBody(HttpStatusCode.Forbidden, "application/json", typeof(ApiResponse<object>))]
public async Task<HttpResponseData> GetAllAsync(...) { /* ... */ }
```

> Apply the equivalent decoration to GetById (`[OpenApiParameter("id", ...)]`), Create/Update (`[OpenApiRequestBody(...)]`), and Delete, mirroring the pattern used for existing entities in this app. No changes to `OpenApiConfigurationOptions.cs` are needed unless the app-level title/description must change.

---

## 14. Step 10 — RBAC Role Mapping

Determine the roles for the new entity's endpoints from the user story and document them, reusing **existing** `Roles` constants wherever possible:

| Operation | Allowed Roles |
|---|---|
| GetAll | *(per story — reuse existing roles)* |
| GetById | *(per story)* |
| Create | *(per story)* |
| Update | *(per story)* |
| Delete | *(per story — usually most restrictive role)* |

Only add a **new** role constant to `Roles.cs` if the user story introduces a genuinely new role that doesn't already exist in the app.

---

## 15. Step 11 — Error Codes Update

Add one new constant to the **existing** `ErrorCodes.cs` (do not restructure the file):

```csharp
public const string {Entity}NotFound = "{ENTITY}_NOT_FOUND";
```

Add any additional business-rule error codes the new entity's operations require, following the existing naming convention (`SCREAMING_SNAKE_CASE`).

---

## 16. Step 12 — Dependency Injection Registration

Modify the **existing** `Program.cs` — add only these lines inside the existing `ConfigureServices` block, next to the other repository/service registrations:

```csharp
services.AddScoped<I{Entity}Repository, {Entity}Repository>();
services.AddScoped<I{Entity}Service, {Entity}Service>();
```

Do not touch the middleware registration, OpenTelemetry pipeline, Key Vault configuration, or any other existing registration.

---

## 17. Adding a Single New Operation to an Existing Entity

If the user story only requires a **new operation** (not a new entity) on an entity that already has a Function class (e.g., `Approve`, `Archive`, `Resend`), do the following minimal, targeted changes instead of the full flow above:

1. **Domain:** Add a new method to the existing `{ExistingEntity}` entity class (e.g., `public void {OperationName}(string actorBy) { ... }`), following the same immutability pattern (private setters, guard clauses, `UpdatedAt`/`UpdatedBy` touch).
2. **Service interface & implementation:** Add one new method to `I{ExistingEntity}Service` / `{ExistingEntity}Service`:
   ```csharp
   Task<ApiResponse<{ExistingEntity}Dto>> {OperationName}Async(Guid id, string actorBy, CancellationToken ct = default);
   ```
   Implementation fetches the entity, calls the new domain method, saves via `UnitOfWork.SaveChangesAsync`, maps to DTO, returns `ApiResponse<T>.Ok(...)`.
3. **Function class:** Add one new `[Function("{ExistingEntity}_{OperationName}")]` HTTP-triggered method to the **existing** `{ExistingEntity}Function.cs`, following the same pattern as other endpoints in that class (auth check → call service → `CreateApiResponseAsync`).
4. **OpenAPI:** Decorate the new method with the same `[OpenApi*]` attribute set used by sibling methods in that class.
5. **Error codes:** Add a new business-rule error code only if the operation can fail with a domain-specific rule (e.g., `ErrorCodes.{ExistingEntity}AlreadyApproved`).
6. **RBAC:** Confirm/add the allowed roles for the new operation.
7. **No DI changes needed** — the service/repository are already registered.

This keeps the change surface minimal: one domain method, one service method, one Function method, zero new files for simple cases.

---

## 18. PII & Encryption Considerations

- If any new field is PII (per the user story), reuse the **existing** `IEncryptionService` already registered in the app — do not create a new encryption service.
- Encrypt before `AddAsync`/`UpdateAsync` in the service layer; decrypt after fetch, before mapping to DTO.
- Never log decrypted PII values; log entity ID and field name only.

---

## 19. Logging Considerations

- Reuse the existing `ILogger<T>` and OpenTelemetry pipeline — no new registration needed.
- Follow existing logging level conventions (`Debug` for entry/exit, `Information` for business milestones, `Warning` for business exceptions, `Error` for infra failures).
- Use structured (named) placeholders, never string interpolation:
  ```csharp
  logger.LogInformation("{Entity} created. Id: {EntityId}, By: {CreatedBy}", entity.{Entity}Id, createdBy);
  ```

---

## 20. Code Contracts & Best Practices (Reminder)

All mandatory patterns from the app-wide standard still apply to new code:

1. No raw SQL — EF Core LINQ only.
2. No `async void` — always `async Task`/`async Task<T>`.
3. No `.Result`/`.Wait()` — always `await`.
4. Nullable reference types enabled; guard clauses at method boundaries.
5. Propagate `CancellationToken ct = default` as the last parameter on all async methods.
6. Domain entity setters remain `private`; use factory/update/deactivate methods.
7. Soft delete only — never `DbSet.Remove()`.
8. No magic strings — use `nameof()`, `ErrorCodes`, `Roles` constants.
9. Single Responsibility — one new file per concern; do not merge unrelated logic into existing files.
10. DRY — reuse `Repository<TEntity, TKey>`, `ApiResponse<T>`, `IAuthorizationHelper`, `IEncryptionService` rather than re-implementing.
11. YAGNI — implement only what the user story requires; do not add speculative endpoints or fields.

---

## 21. Complete Add-Function Checklist

### New Entity Scenario
- [ ] `{Entity}.cs` (Domain)
- [ ] `I{Entity}Repository.cs` (Domain)
- [ ] `{Entity}Configuration.cs` (Infrastructure)
- [ ] `ApplicationDbContext.cs` — modified (new `DbSet` + query filter)
- [ ] `{Entity}Repository.cs` (Infrastructure)
- [ ] `IUnitOfWork.cs` / `UnitOfWork.cs` — modified (new property)
- [ ] `{Entity}Dto.cs`, `Create{Entity}Dto.cs`, `Update{Entity}Dto.cs` (Application)
- [ ] `Create{Entity}Validator.cs`, `Update{Entity}Validator.cs` (Application)
- [ ] `I{Entity}Service.cs`, `{Entity}Service.cs` (Application)
- [ ] `MappingProfile.cs` — modified (2 new map lines)
- [ ] `{Entity}Function.cs` (Functions — full `[OpenApi*]` decoration)
- [ ] `ErrorCodes.cs` — modified (new NotFound/business codes)
- [ ] `Program.cs` — modified (2 new DI registration lines)
- [ ] EF Core migration generated (`Add{Entity}`) and reviewed per §6A.5–§6A.11 (naming, PR checklist, breaking-change classification, environment apply rules)
- [ ] RBAC roles confirmed/documented per endpoint

### New Operation on Existing Entity Scenario
- [ ] New method on existing `{ExistingEntity}` domain entity
- [ ] New method on `I{ExistingEntity}Service` / `{ExistingEntity}Service`
- [ ] New `[Function("{ExistingEntity}_{OperationName}")]` method on existing `{ExistingEntity}Function.cs`
- [ ] `[OpenApi*]` decoration on the new method
- [ ] New error code(s) added to `ErrorCodes.cs`, only if needed
- [ ] RBAC roles confirmed for the new operation
- [ ] No new files, no DI changes (service/repository already registered)

---

## 22. Final LLM Instruction

> You are a senior .NET 10 Azure Functions engineer extending an **existing** `{BusinessArea}` Function App. Using this document as your guide:
>
> **Rules:**
> 1. First inspect the existing solution to confirm naming conventions, schema, existing roles, and existing error codes — mirror them exactly.
> 2. Resolve `{BusinessArea}`, `{Entity}` (or `{ExistingEntity}`/`{OperationName}`), and field placeholders from the user story before generating code.
> 3. For a **new entity**, generate every file in the §21 "New Entity Scenario" checklist — no more, no less.
> 4. For a **new operation on an existing entity**, make only the minimal, targeted changes in §17 — do not re-scaffold the entity or duplicate existing files.
> 5. Never modify `CorrelationMiddleware`, `JwtAuthenticationMiddleware`, `ExceptionHandlingMiddleware`, OpenTelemetry/Key Vault wiring, or shared `ApiResponse`/`ApiError` contracts.
> 5a. Never add `Database.Migrate()`/`MigrateAsync()` to `Program.cs` or any Function trigger to "auto-apply" the new entity's migration — schema changes are applied exclusively via the controlled pipeline steps defined in §6A.1–§6A.7.
> 5b. For the new migration, classify it as breaking or non-breaking (§6A.8). If breaking (e.g., a required column added to an existing populated table, or a rename), propose the expand/contract plan (§6A.9) instead of a single unsafe migration.
> 6. Every HTTP response must use the existing `ApiResponse<T>` envelope — never `ProblemDetails` or raw strings.
> 7. Always throw `FluentValidationException` on FluentValidation failures and entity-specific `NotFoundException` with a dedicated `ErrorCodes` constant.
> 8. Every new Function method must have complete `[OpenApiOperation]`, `[OpenApiSecurity]`, `[OpenApiParameter]` (path params), `[OpenApiRequestBody]` (POST/PUT), and `[OpenApiResponseWithBody]` (every status code) attributes, matching the style of sibling methods in the app.
> 9. Reuse the existing `IEncryptionService` for any new PII field — never create a second encryption service.
> 10. If the user story doesn't specify RBAC roles, apply the most restrictive sensible default and flag the assumption for user confirmation.
> 11. After generating all files, output a summary table: `File Path | New/Modified | Key Responsibility`.

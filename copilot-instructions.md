
# AI Development Instructions
## .NET Web API – Service Layer → Clean Architecture

This repository contains an **ASP.NET Core Web API** currently implemented using a **Service Layer architecture** and gradually transitioning toward **Clean Architecture**.

All AI coding agents (GitHub Copilot Agents, Claude Code, GPT agents, etc.) must strictly follow the rules defined in this document when generating or modifying code.

---

# 1. Technology Stack Requirements

## 1.1 Framework
- ASP.NET Core Web API
- Use the **.NET LTS version** defined by the project.

## 1.2 Database
- Database must be **Microsoft SQL Server (MSSQL) compatible**.
- Avoid SQL syntax specific to MySQL/PostgreSQL.

## 1.3 ORM
Use **Entity Framework Core**.
- All EF Core packages must use the **same version**.
- EF Core version must match the project target and each other (SqlServer/Tools/Design/etc.).

Example (conceptual):

```csharp
Microsoft.EntityFrameworkCore
Microsoft.EntityFrameworkCore.SqlServer
Microsoft.EntityFrameworkCore.Tools
Microsoft.EntityFrameworkCore.Design
```

---

# 2. Architecture Direction

## Current architecture (legacy modules)

Controller
↓
Service Layer
↓
Repository
↓
DbContext

## Target architecture (for new modules)

API
↓
Application (Commands / Queries / Handlers)
↓
Domain
↓
Infrastructure

### Rules
- Existing modules remain **Service Layer**.
- New modules should follow **Clean Architecture**.
- Avoid large refactors of legacy modules unless requested.
- Controllers must not access `DbContext` directly:
  - Old modules: `Controller → Service → Repo`
  - New modules: `Controller → Command/Query Handler → Repo Interface → Infrastructure`

---

# 3. Logging and Observability

## 3.1 Logging Framework
- Use **Serilog** for all logging.
- Logs must be compatible with **Grafana/Loki pipelines** (sink and shipping must be configuration-driven).

## 3.2 Log Retention
- Logs must be **rolling/cyclic retention of 7 days**.
- Do not keep unlimited log files.

Conceptual example:

```json
{
  "RollingInterval": "Day",
  "RetainedFileCountLimit": 7
}
```

## 3.3 Enrichment
Logs should include:
- Request ID
- Correlation ID
- Username
- Module name
- Timestamp

## 3.4 Sensitive Data
Never log:
- passwords
- tokens
- secrets
- sensitive personal data

---

# 4. Identity Framework Rules

Authentication must use **ASP.NET Core Identity**.

## 4.1 Identity Isolation Rule
Identity users must **NOT** be directly related to business entities via foreign keys.

Forbidden:
- `Order.UserId → FK → AspNetUsers`

Allowed:
- Store username metadata in audit fields (no FK):
  - `CreatedByUsername`
  - `UpdatedByUsername`
  - `DeletedByUsername`

Business user concepts (employee/client-contact/vendor-user) must be separate domain entities (if needed).

---

# 5. Audit Fields (Username Based)

Entities must track auditing information using **username strings** (not Identity user id).

Required fields:
- `CreatedAt`
- `CreatedByUsername`
- `UpdatedAt`
- `UpdatedByUsername`
- `DeletedAt`
- `DeletedByUsername`
- `IsDeleted`

Rules:
- Store **username** string
- No FK relationship to Identity tables
- Username comes from authenticated user context

---

# 6. Business Entity ID Strategy

All business entities must use **integer identity primary keys**.

Example:
- `Id INT IDENTITY(1,1) PRIMARY KEY`

Forbidden:
- GUID primary keys
- string primary keys

Allowed:
- GUID may exist as a **public identifier only** (e.g., `PublicId UNIQUEIDENTIFIER`), but **primary key remains INT**.

---

# 7. Foreign Key Rules
All relationships between business entities must use **INT foreign keys**.

Example:

Order
- Id INT

OrderItem
- Id INT
- OrderId INT

---

# 8. Soft Delete Policy

All tables must support **soft delete**.

Required fields:
- `IsDeleted`
- `DeletedAt`
- `DeletedByUsername`

Delete operations must perform soft delete (update), not hard delete.
Hard delete is forbidden unless explicitly required.

---

# 9. DTO Mapping Rules

Do **NOT** use AutoMapper.
- DTO mapping must be **manual/custom mapping**:
  - `ToDto()` / `FromDto()` methods
  - static mappers
  - explicit constructors

Avoid:
- AutoMapper
- reflection based mapping
- dynamic mapping

Parent/child DTO rule:
- When returning master-child DTOs, include major parent fields (not just parentId) in the response shape suitable for form screens.

---

# 10. Controller Rules

Controllers must remain **thin**.
Controllers should:
- validate input (shape only)
- call service/handler
- return typed DTOs

Controllers must NOT:
- access DbContext directly
- contain business logic
- return `dynamic`, anonymous objects, or `object` responses

Controllers must define proper response metadata:
- `[Produces]`, `[Consumes]`, `[ProducesResponseType]`

---

# 11. Validation Rules

All request DTOs must use **FluentValidation**.
- Validators should be defined per request model.
- Nested child collection validation must be included for master-child forms.

---

# 12. EF Core Relationship Definition

Relationships must use **EF Core Fluent API**:
- Implement `IEntityTypeConfiguration<T>` per entity.
- Avoid relying only on annotations for relations.
- Use MSSQL-friendly indexing (including soft-delete-aware uniqueness where appropriate).

---

# 13. File Upload System

## 13.1 Temporary Storage
Files are initially saved locally using:
- `private/{module}/{year}/{month}/{filename}`

Example:
- `private/invoices/2026/03/invoice123.pdf`

## 13.2 Processing to Cloudflare R2
Background worker moves files to **Cloudflare R2**.
Before processing, worker must verify:
- Cloudflare settings exist
- Cloudflare feature enabled

## 13.3 File Access Security
- Never expose the physical `private/` folder root in API responses.
- If not yet uploaded to R2, file must be accessed via a secured controller endpoint.
- Endpoint accepts a path **without** `private/` prefix:
  - Example input: `invoices/2026/03/file.pdf`
- Controller maps internally to: `private/{path}`
- Reject traversal: `..`, absolute paths, invalid characters

---

# 14. Background Workers

Background workers handle:
- File upload processing to R2
- Email outbox sending

Workers must be:
- idempotent
- retry-safe
- concurrency-safe

---

# 15. Email System (Outbox Pattern)

Emails must follow **Outbox pattern**.

Workflow:
API Request
↓
Store Email in Outbox Table (transactional with business change)
↓
Background Worker Sends Email

Outbox fields must include (or equivalent):
- Status
- AttemptCount
- LastAttempt
- ErrorMessage

Also support:
- enable/disable email sending feature flag
- retry policy via config

---

# 16. Admin Monitoring Endpoints

System must expose monitoring endpoints (secured by roles/policies):

Email queue:
- `GET /emails/unprocessed`
- `GET /emails/failed`
- `POST /emails/retry`

File upload queue:
- `GET /files/unprocessed`
- `GET /files/failed`
- `POST /files/retry`

Settings management endpoints:
- Email server settings
- Organization metadata
- Feature toggles (enable/disable email, enable/disable R2 upload)
- Cloudflare R2 configuration (view/validate/update)

---

# 17. Centralized Settings Module

All system settings must be stored centrally and documented.
Examples:
- Email server configuration
- Organization metadata
- Feature toggles
- Cloudflare R2 settings
- Logging settings

Settings should be accessed using:
- SettingsService
- Options pattern

Settings changes should be auditable when auditing is available.

---

# 18. Database Migration Safety Rules (Must Follow)

## 18.1 Adding NOT NULL Columns
Never directly add NOT NULL columns to existing populated tables.

Safe sequence:
1. Add column as **nullable**
2. Backfill existing rows (update nulls)
3. Alter column to **NOT NULL**

## 18.2 Relation-dependent columns
If adding a new relation-dependent (FK-like) column and existing rows need a value:
- Backfill with a deterministic rule:
  - use the **first record** from the parent table (ORDER BY PK) as fallback, unless business rules specify otherwise.
- If parent table is empty:
  - seed required default parent record first, or keep nullable until parent exists.

Migrations must include required data updates whenever schema changes imply existing data must be corrected.

---

# 19. Database Auto Migration on Startup

Application may automatically apply migrations if enabled by configuration:

- `Database:AutoMigrate=true`

Startup must:
- detect pending migrations
- apply migrations safely
- log migration results

---

# 20. Master–Child Form Operations (Module Standard)

Besides basic CRUD, modules representing master-child screens must support:
- create master with children in one transaction
- update master + reconcile children (insert/update + soft-delete removed)
- get master including children (form-friendly DTO)
- list/search/paging
- soft delete

Child reconciliation algorithm:
1. Load existing children
2. Update existing
3. Insert new
4. Soft delete missing
5. Save in transaction

---

# 21. Enum Serialization Rule (Important)

API enums must be serialized/deserialized as **strings**, not integers.

Rules:
- Configure JSON serialization to use string enums globally.
- Swagger/OpenAPI should display enums as strings (and show allowed values).

Do not expose numeric enum values in API contracts unless explicitly requested.

---

# 22. Documentation Requirements (Must Keep Updated)

AI agents must update documentation **whenever implementing or changing features**.
Documentation must be maintained in two developer-friendly views:

## 22.1 Frontend Developer Friendly Docs
Maintain/update: `docs/frontend-api.md` (or equivalent). It must include:
- API integration notes per module (purpose + workflows)
- Auth/headers conventions
- Request/response examples (typed)
- **Constants and enums** needed for integration:
  - enum string values and meanings
  - status flows and allowed transitions
- Module notes explaining:
  - features and usage
  - workflow steps
  - assumptions used for default data entry
  - rule engines used for auto-calculations (what inputs, what outputs, when applied)

## 22.2 Backend Developer Friendly Docs
Maintain/update: `docs/backend-architecture.md` (or equivalent). It must include:
- Architecture diagram (high level) and module introductions
- Entity notes with detailed field lists per entity
- Relationship schema diagram (ERD-style)
- Config settings explanation:
  - what each setting does
  - defaults and environment overrides
- Instructions for adding new modules/features manually:
  - folders/projects to create
  - DI registration conventions
  - migration + entity configuration rules
- Rule engines used for auto-calculations:
  - assumptions and defaulting logic
  - where rules live (service/handler)
  - how to test/extend

### Documentation update rule of thumb
When a task changes any of the below, docs MUST be updated in the same change:
- endpoints, DTOs, request/response fields
- enums/constants/status flows
- workflows or assumptions
- settings/config keys
- background worker behavior (outbox/uploads)
- entity fields/relations

---

# 23. Default Roles & Seeding

System must seed roles:
- SuperAdmin
- GeneralAdmin
- GeneralUser
- ClientAdmin
- ClientUser

Seed a default SuperAdmin user from configuration:
- `Seed:SuperAdminEmail`
- `Seed:SuperAdminPassword`

Seeding must be idempotent and must never log passwords.

---

# 24. Code Generation Expectations for AI Agents

When implementing features, generate:
1. DTOs (manual mapping)
2. FluentValidation validators
3. Services (legacy) or Command/Query handlers (new)
4. Repository interfaces + implementations
5. EF Fluent API configurations
6. Controllers (thin + response attributes)
7. Serilog logging
8. Background worker integration (if uploads/outbox)
9. Safe migrations (if schema changes)
10. Documentation updates (frontend + backend docs)

---

# 25. When AI Must Stop and Propose a Plan

Agents must propose a plan before code changes if the request involves:
- major schema redesign
- authentication changes
- wide architecture restructuring
- new infrastructure components/sinks

---

# End of AI Instructions
These rules must be followed by **GitHub Copilot Agents, Claude Code, and GPT coding assistants** working in this repository.

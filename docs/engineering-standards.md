# Engineering Standards — Code, Reviews and Tests

Part of the Phase 0 design set: [plan](cashflow-app-plan.md) · [architecture](architecture.md) · [database](database-design.md) · [API](api-design.md) · [UI](ui-design.md) · **engineering standards**

---

## 1. Repository and workflow

| Topic | Standard |
|---|---|
| Branching | Trunk-based: short-lived branches off `main` (`feat/…`, `fix/…`, `chore/…`), squash-merge via PR |
| Commits | [Conventional Commits](https://www.conventionalcommits.org/) (`feat(planner): fill-right for rows`) |
| PRs | Small (< 400 changed lines where possible), description with *what / why / how tested*, linked issue, screenshots for UI |
| Reviews | 1 approval (self-review checklist when solo); CI must be green; no force-push to `main` |
| Versioning | SemVer tags `vX.Y.Z`; `CHANGELOG.md` generated from commits |
| Decisions | ADRs in `docs/adr/` (MADR template) for anything hard to reverse |
| Definition of done | Code + tests + docs updated; analyzers clean; migration script reviewed; feature flag or complete |

---

## 2. C# and .NET coding standards

### 2.1 Build settings (`Directory.Build.props`)

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <InvariantGlobalization>false</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

- **Central Package Management** (`Directory.Packages.props`), `.slnx` solution, `global.json` pinning the .NET 10 SDK band.
- `.editorconfig` committed; `dotnet format --verify-no-changes` in CI.
- Analyzers: built-in .NET analyzers, `Meziantou.Analyzer`, `Roslynator.Analyzers`; banned APIs via `BannedSymbols.txt` (`DateTime.Now`, `DateTime.UtcNow` → `TimeProvider`; `Math.Round(decimal)` without `MidpointRounding` → `Money.Round`).

### 2.2 Style

| Rule | Example |
|---|---|
| File-scoped namespaces, one public type per file (feature slice files may hold request + validator + handler) | `namespace Cashflow.Application.Features.TaxTables;` |
| Naming: `PascalCase` types/members, `camelCase` locals/params, `_camelCase` private fields, `I` prefix for interfaces, `Async` suffix for async methods | `Task<Result<TaxTableDto>> HandleAsync(…)` |
| Prefer `sealed` classes, `record` for DTOs/commands, `readonly record struct` for small value objects | `public readonly record struct Money(decimal Amount, string Currency);` |
| Primary constructors for DI | `public sealed class CloneTaxTableHandler(AppDbContext db, TimeProvider clock)` |
| Collection expressions, pattern matching, `switch` expressions | `var rate = method switch { … };` |
| `var` when the type is obvious; explicit otherwise | |
| No `#region`; methods ≤ ~40 lines; nesting ≤ 3 | |
| Comments explain *why*, not *what*; XML docs on public engine APIs | |

### 2.3 Correctness rules for money and dates

- Money is `decimal` only. Never `double`/`float` for amounts or rates.
- Round only through `Money.Round(value)` (2 decimals, `MidpointRounding.AwayFromZero` unless the workspace chose banker's rounding) at defined points: stored allocation, payslip line, transfer, interest.
- Business dates are `DateOnly`; instants are `DateTimeOffset` in UTC; "today" comes from `TimeProvider` converted to the workspace time zone.
- Percentages are stored as fractions (`0.25`) and displayed as `25%`.

### 2.4 Architecture rules

- Dependencies point inward (Api → Application → Domain); Domain has no package references except BCL. Enforced by NetArchTest.
- Endpoints are thin: bind → validate → call handler → map `Result` to `TypedResults`.
- Handlers own one use case; no handler calls another handler — share logic through domain services or the engine.
- Use `AppDbContext` directly in handlers; projections with `Select` to DTOs for queries (`AsNoTracking`); no lazy loading.
- Expected failures return `Result.Failure(Error)`; exceptions only for bugs and infrastructure faults.
- Async all the way; pass `CancellationToken` through every async call; no `.Result`/`.Wait()`.
- Configuration via strongly-typed options with `ValidateDataAnnotations().ValidateOnStart()`.
- Logging with `ILogger<T>` and source-generated `[LoggerMessage]`; never log amounts, notes, names or emails.
- Every tenant query goes through the filtered `AppDbContext`; `IgnoreQueryFilters()` is banned (BannedSymbols) except in seed migrations; there are no admin features.

### 2.5 Example slice

```csharp
namespace Cashflow.Application.Features.TaxTables;

public sealed record CloneTaxTable(Guid SourceId, string Name);

public sealed class CloneTaxTableValidator : AbstractValidator<CloneTaxTable>
{
    public CloneTaxTableValidator() => RuleFor(x => x.Name).NotEmpty().MaximumLength(120);
}

public sealed class CloneTaxTableHandler(AppDbContext db, ICurrentWorkspace workspace)
{
    public async Task<Result<TaxTableDto>> HandleAsync(CloneTaxTable cmd, CancellationToken ct)
    {
        var source = await db.TaxTables
            .Include(t => t.Versions).ThenInclude(v => v.Brackets)
            .SingleOrDefaultAsync(t => t.Id == cmd.SourceId, ct);   // filter: own or template

        if (source is null)
            return Errors.NotFound("tax-table", cmd.SourceId);

        var copy = source.CloneInto(workspace.Id, cmd.Name);
        db.TaxTables.Add(copy);
        await db.SaveChangesAsync(ct);
        return copy.ToDto();
    }
}
```

---

## 3. Front-end (Blazor) standards

- Components: `PascalCase.razor` + code-behind `PascalCase.razor.cs` when logic exceeds ~30 lines; parameters are `[Parameter, EditorRequired]` where required.
- No business calculations in components — call the API (or the shared engine for instant what-if previews).
- Use `MoneyText`/`MoneyInput` for every amount; never format money with string interpolation.
- Styling is Tailwind utility classes only: use the theme tokens (`bg-surface`, `text-needs`) — never raw hex values or inline `style`; build dynamic class names from complete literal strings (Tailwind can't detect `"text-" + color`); reuse UI-library components instead of copying long class lists; keep class order consistent (layout → spacing → typography → color → state).
- Dispose subscriptions (`IAsyncDisposable`); avoid unnecessary re-renders (`ShouldRender`, `@key` in lists).
- Accessibility checklist per screen (labels, focus order, contrast) is part of the PR template.

---

## 4. Testing standards

### 4.1 Test pyramid and tools

| Layer | Project | Tools | Runs |
|---|---|---|---|
| Unit — engine & domain | `Cashflow.Domain.Tests` | xUnit v3, Shouldly, CsCheck (property), Verify (snapshots) | Every build |
| Unit — application handlers | `Cashflow.Application.Tests` | xUnit v3, real `AppDbContext` on Testcontainers PostgreSQL (no EF in-memory provider) | Every PR |
| Integration — API | `Cashflow.Api.Tests` | `WebApplicationFactory`, Testcontainers PostgreSQL, Respawn, fake Google auth handler | Every PR |
| Architecture | `Cashflow.Architecture.Tests` | NetArchTest.Rules | Every PR |
| Component | `Cashflow.Web.Tests` | bUnit | Every PR |
| End-to-end | `Cashflow.E2E` | Playwright for .NET against the Compose stack | On `main` and before release |
| Mutation | engine only | Stryker.NET | Weekly |

### 4.2 Conventions

- Name: `Method_Scenario_ExpectedResult` or a sentence (`Tax_is_zero_below_250k`); one behavior per test.
- Arrange-Act-Assert with blank lines; builders/`ObjectMother` for test data (`SalaryProfileBuilder.Default().WithGross(80_000m)`).
- No `Thread.Sleep`; use `FakeTimeProvider` for time.
- Tests are independent and parallel-safe; each integration test class gets a fresh database via Respawn.
- External calls (Google, Web Push) are faked at the boundary; never hit real services in tests.

### 4.3 Required test suites

| Suite | Must cover |
|---|---|
| Golden workbook | All values in plan Appendix C; January/February period totals; known workbook defects asserted as documented differences |
| Tax engine | Each bracket boundary (exactly at, ±0.01), flat and none methods, custom user tables, monthly vs annual basis, benefit exemption cap, every split method |
| Contributions | Percentage with floor/ceiling, bracket-table lookup, fixed, overrides, disabled scheme, placement P1/P2/split |
| Schedules | Even/P1/P2/custom/spread/once, active months, effective ranges, overlap rejection, override survives regeneration |
| Allocation & transfers | Σ buckets + unallocated = income (property), nested routes, unrouted items flagged |
| Interest, goals, debt | ADB with mid-month rate change, final tax, phase targets, payoff with zero and non-zero rates |
| Tenancy & auth | Cross-workspace read/write attempts return 404/403; RLS blocks raw queries without `app.workspace_id`; viewer cannot write; unknown Google account rejected; invite single-use |
| API contract | OpenAPI document snapshot; Problem Details shapes; `If-Match` → 412 |
| Import | Fixture workbook (anonymized) → expected counts, rules, overrides and comparison report |

### 4.4 Quality gates in CI

| Gate | Threshold |
|---|---|
| Build | Zero warnings (warnings as errors) |
| Format | `dotnet format --verify-no-changes` |
| Tests | 100% pass |
| Coverage (Coverlet) | Domain ≥ 90% lines, Application ≥ 80%, overall ≥ 75% |
| Mutation score (weekly) | Engine ≥ 80% |
| Security | No high/critical vulnerable packages; secret scanning clean; container scan clean |
| OpenAPI | No unreviewed breaking change (snapshot diff) |

---

## 5. Database standards

- One migration per PR, named `YYYYMMDDHHmm_DescriptiveName`; generated SQL reviewed in the PR.
- No data-destroying migration without an expand/contract plan and a backup step.
- Every FK indexed; every tenant table has `workspace_id`, an RLS policy and a query filter (checked by a test that scans the model).
- No raw SQL in handlers; reviewed exceptions live in `Infrastructure/Persistence/Sql`.

---

## 6. Security checklist (per PR touching endpoints)

- [ ] Endpoint has an authorization policy (no anonymous endpoints except `/bff/*` sign-in and invite preview, health, static files)
- [ ] Input validated; sizes limited (strings, file uploads ≤ 10 MB)
- [ ] Tenant isolation covered by a test
- [ ] No sensitive data in logs or error messages
- [ ] Antiforgery enforced on state-changing methods

# Phase 0 — Toolchain and Security Hardening Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** get the backend building and tested locally, then fix the six confirmed security defects from the [roadmap audit](./2026-10-01-aegis-build-roadmap.md#1-where-we-are-audit-2026-10-01).

**Architecture:** small, independent changes in `aegis/src/Aegis.Api`, `Aegis.Shared`, and `Aegis.Modules.Audit`. No schema changes. Every fix ships with a regression test that fails before the change.

**Tech Stack:** .NET 9, ASP.NET Core built-in rate limiting and CORS, System.Text.Json, EF Core 9 + Npgsql, xUnit 2.9, `WebApplicationFactory`.

## Global Constraints

- Repo: `/Users/david/projects/money-laundry-detector/aegis`, branch per task off `main`.
- No new NuGet packages. Everything here is in the ASP.NET Core 9 shared framework.
- Integration tests need Postgres: `docker-compose up -d postgres` and database `aegis_test` (override with `AEGIS_TEST_CONNECTION`).
- Every audit `BeforeState`/`AfterState` must be produced by `AuditPayload.Json(...)` after Task 2.
- Keep controller style as-is (explicit `if (!_tenant.IsAuthenticated) return Unauthorized();` then authorization check).

---

### Task 1: Toolchain and baseline

**Files:** none changed. Output recorded in the PR description.

- [ ] **Step 1: Install the .NET 9 SDK** (needs the owner's approval; installs to `~/.dotnet`)

```bash
curl -sSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh
bash /tmp/dotnet-install.sh --channel 9.0 --install-dir "$HOME/.dotnet"
echo 'export DOTNET_ROOT="$HOME/.dotnet"; export PATH="$DOTNET_ROOT:$DOTNET_ROOT/tools:$PATH"' >> ~/.zshrc
source ~/.zshrc && dotnet --version
```

Expected: `9.0.x`.

- [ ] **Step 2: Start Postgres and create the test database**

```bash
cd aegis && docker-compose up -d postgres
docker-compose exec -T postgres psql -U aegis -c "CREATE DATABASE aegis_test;" || true
```

- [ ] **Step 3: Baseline build and test**

```bash
dotnet restore && dotnet build --no-restore
dotnet test --no-build
```

Expected: build succeeds. Record pass/fail counts per test project. If anything fails on a clean `main`, stop and fix that first in its own PR.

---

### Task 2: Safe audit payloads

Interpolated JSON goes into a `jsonb` column. A `"` in `assignedTo` makes Postgres reject the insert (HTTP 500), and crafted values can inject keys into audit rows.

**Files:**
- Create: `src/Aegis.Modules.Audit/Application/AuditPayload.cs`
- Modify: `src/Aegis.Api/Controllers/AlertsController.cs:138,223`, `src/Aegis.Api/Controllers/CasesController.cs:112,183`, `src/Aegis.Api/Controllers/RulesController.cs:121,158`, `src/Aegis.Api/Controllers/DevTenantsController.cs:112,124`, `src/Aegis.Application/Transactions/IngestAndEvaluateStructuring.cs:224`
- Modify: `tests/Unit/Aegis.Tests.Unit.csproj` (add Audit project reference)
- Test: `tests/Unit/Audit/AuditPayloadTests.cs`, `tests/Integration/Alerts/AlertAuditPayloadTests.cs`

**Interfaces:**
- Produces: `public static string AuditPayload.Json(object value)` in namespace `Aegis.Modules.Audit.Application`. Serializes with camelCase property names.

- [ ] **Step 1: Write the failing unit test**

Add to `tests/Unit/Aegis.Tests.Unit.csproj` inside the existing `ItemGroup` of project references:

```xml
<ProjectReference Include="..\..\src\Aegis.Modules.Audit\Aegis.Modules.Audit.csproj" />
```

Create `tests/Unit/Audit/AuditPayloadTests.cs`:

```csharp
namespace Aegis.Tests.Unit.Audit;

using System.Text.Json;
using Aegis.Modules.Audit.Application;

public sealed class AuditPayloadTests
{
    [Fact]
    public void Escapes_quotes_and_does_not_allow_key_injection()
    {
        var hostile = "bob\",\"isAdmin\":true,\"x\":\"";

        var json = AuditPayload.Json(new { AssignedTo = hostile });

        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;
        Assert.Equal(hostile, root.GetProperty("assignedTo").GetString());
        Assert.False(root.TryGetProperty("isAdmin", out _));
    }

    [Fact]
    public void Uses_camel_case_and_keeps_numbers_numeric()
    {
        var json = AuditPayload.Json(new { RuleId = Guid.Empty, Version = 3 });

        using var doc = JsonDocument.Parse(json);
        Assert.Equal(3, doc.RootElement.GetProperty("version").GetInt32());
        Assert.Equal(Guid.Empty, doc.RootElement.GetProperty("ruleId").GetGuid());
    }
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `dotnet test tests/Unit --filter AuditPayloadTests`
Expected: build error, `The name 'AuditPayload' does not exist`.

- [ ] **Step 3: Implement**

Create `src/Aegis.Modules.Audit/Application/AuditPayload.cs`:

```csharp
namespace Aegis.Modules.Audit.Application;

using System.Text.Json;

public static class AuditPayload
{
    private static readonly JsonSerializerOptions Options = new(JsonSerializerDefaults.Web);

    public static string Json(object value) => JsonSerializer.Serialize(value, Options);
}
```

- [ ] **Step 4: Run the unit test and confirm it passes**

Run: `dotnet test tests/Unit --filter AuditPayloadTests`
Expected: 2 passed.

- [ ] **Step 5: Write the failing integration test**

Create `tests/Integration/Alerts/AlertAuditPayloadTests.cs`. It reuses the bootstrap/login pattern from `RuleRbacTests` and the `SliceHelpers`-style ingest to produce an alert. Use the existing helper in `tests/Integration/Alerts/AlertDedupeTests.cs` that produces a structuring alert; if it is private, copy its setup into this class.

```csharp
namespace Aegis.Tests.Integration.Alerts;

using System.Net;
using System.Net.Http.Json;
using System.Text.Json;
using Aegis.Modules.Audit.Application;
using Aegis.Modules.Audit.Domain;
using Microsoft.Extensions.DependencyInjection;

public sealed class AlertAuditPayloadTests : IAsyncLifetime
{
    private readonly AlertTestFixture _fx = new();

    public Task InitializeAsync() => _fx.InitializeAsync();
    public Task DisposeAsync() => _fx.DisposeAsync();

    [Fact]
    public async Task Assigning_with_quotes_in_name_succeeds_and_audits_exact_value()
    {
        var alertId = await _fx.CreateStructuringAlertAsync();
        var hostile = "bob\",\"isAdmin\":true,\"x\":\"";

        var response = await _fx.AdminClient.PostAsJsonAsync(
            $"/api/v1/alerts/{alertId}/assign", new { assignedTo = hostile });

        Assert.Equal(HttpStatusCode.OK, response.StatusCode);

        await using var scope = _fx.Factory.Services.CreateAsyncScope();
        var audit = scope.ServiceProvider.GetRequiredService<IAuditEventRepository>();
        var events = await audit.ListByTenantAsync(_fx.TenantId, take: 50);
        var assigned = events.First(e => e.EventType == AuditEventTypes.ALERT_ASSIGNED);
        using var doc = JsonDocument.Parse(assigned.AfterState!);
        Assert.Equal(hostile, doc.RootElement.GetProperty("assignedTo").GetString());
        Assert.False(doc.RootElement.TryGetProperty("isAdmin", out _));
    }
}
```

Create `tests/Integration/Alerts/AlertTestFixture.cs` by moving the tenant bootstrap, admin login, customer/account creation, and structuring ingest loop out of `AlertDedupeTests.cs` into public members: `AegisApiFactory Factory`, `HttpClient AdminClient`, `Guid TenantId`, `string Slug`, `Task<Guid> CreateStructuringAlertAsync()`, `Task<HttpClient> CreateUserClientAsync(string email, string[] roles)` (the last one copies the analyst-creation block from `RuleRbacTests.InitializeAsync` and logs in). Update `AlertDedupeTests` to use the fixture so the setup isn't duplicated.

- [ ] **Step 6: Run it and confirm it fails**

Run: `dotnet test tests/Integration --filter AlertAuditPayloadTests`
Expected: FAIL, status `InternalServerError` (Postgres rejects invalid `jsonb`).

- [ ] **Step 7: Replace every interpolated payload**

Add `using Aegis.Modules.Audit.Application;` where missing, then replace each call argument:

| File:line | Replace with |
|---|---|
| `AlertsController.cs:138` | `AuditPayload.Json(new { assignedTo = assignee })` |
| `AlertsController.cs:223` | `AuditPayload.Json(new { alertId = alert.Id })` |
| `CasesController.cs:112` | `AuditPayload.Json(new { assignedTo = assignee })` |
| `CasesController.cs:183` | `AuditPayload.Json(new { disposition = disposition.ToString() })` |
| `RulesController.cs:121` | `AuditPayload.Json(new { ruleId, version = next })` |
| `RulesController.cs:158` | `AuditPayload.Json(new { ruleId = version.RuleId, version = version.VersionNumber })` |
| `DevTenantsController.cs:112` | `AuditPayload.Json(new { slug = tenant.Slug })` |
| `DevTenantsController.cs:124` | `AuditPayload.Json(new { email = admin.Email })` |
| `IngestAndEvaluateStructuring.cs:224` | `AuditPayload.Json(new { ruleCode = result.RuleCode, ruleVersionId = result.RuleVersionId })` |

Confirm none remain:

Run: `rg -n '\$"\{\{' src -g '*.cs'`
Expected: no output.

- [ ] **Step 8: Run all tests**

Run: `dotnet test`
Expected: all pass, including the two new test classes.

- [ ] **Step 9: Commit**

```bash
git add src tests
git commit -m "fix(audit): serialize audit payloads instead of interpolating user input"
```

---

### Task 3: Authorize alert and case work actions

Alert assign/dismiss/resolve/create-case and case assign/notes accept any authenticated user, including one with no roles.

**Files:**
- Modify: `src/Aegis.Shared/Security/TenantAuthorization.cs`
- Modify: `src/Aegis.Api/Controllers/AlertsController.cs` (assign, dismiss, resolve, create-case), `src/Aegis.Api/Controllers/CasesController.cs` (assign, notes)
- Test: `tests/Integration/Alerts/AlertWorkRbacTests.cs`

**Interfaces:**
- Consumes: `AlertTestFixture.CreateUserClientAsync(string email, string[] roles)` from Task 2.
- Produces: `public static bool TenantAuthorization.CanWorkAlerts(ITenantContext tenant)` — true for Admin or Analyst. Phase 1 PR 1.2 replaces it with a permission check.

- [ ] **Step 1: Write the failing test**

```csharp
namespace Aegis.Tests.Integration.Alerts;

using System.Net;
using System.Net.Http.Json;
using System.Text.Json;
using Aegis.Shared.Security;

public sealed class AlertWorkRbacTests : IAsyncLifetime
{
    private readonly AlertTestFixture _fx = new();

    public Task InitializeAsync() => _fx.InitializeAsync();
    public Task DisposeAsync() => _fx.DisposeAsync();

    [Theory]
    [InlineData("assign")]
    [InlineData("dismiss")]
    [InlineData("resolve")]
    [InlineData("create-case")]
    public async Task User_without_work_role_is_forbidden(string action)
    {
        var alertId = await _fx.CreateStructuringAlertAsync();
        var noRole = await _fx.CreateUserClientAsync($"viewer@{_fx.Slug}.test", Array.Empty<string>());

        var response = await noRole.PostAsJsonAsync($"/api/v1/alerts/{alertId}/{action}", new { });

        Assert.Equal(HttpStatusCode.Forbidden, response.StatusCode);
    }

    [Fact]
    public async Task Analyst_can_work_alerts_and_cases()
    {
        var alertId = await _fx.CreateStructuringAlertAsync();
        var analyst = await _fx.CreateUserClientAsync($"analyst@{_fx.Slug}.test", new[] { RoleNames.Analyst });

        var created = await analyst.PostAsync($"/api/v1/alerts/{alertId}/create-case", null);
        Assert.Equal(HttpStatusCode.Created, created.StatusCode);
        var caseId = (await created.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("id").GetGuid();

        var note = await analyst.PostAsJsonAsync($"/api/v1/cases/{caseId}/notes", new { text = "Reviewed." });
        Assert.Equal(HttpStatusCode.OK, note.StatusCode);
    }

    [Fact]
    public async Task User_without_work_role_cannot_assign_or_annotate_cases()
    {
        var alertId = await _fx.CreateStructuringAlertAsync();
        var created = await _fx.AdminClient.PostAsync($"/api/v1/alerts/{alertId}/create-case", null);
        var caseId = (await created.Content.ReadFromJsonAsync<JsonElement>()).GetProperty("id").GetGuid();
        var noRole = await _fx.CreateUserClientAsync($"viewer2@{_fx.Slug}.test", Array.Empty<string>());

        var assign = await noRole.PostAsJsonAsync($"/api/v1/cases/{caseId}/assign", new { });
        var note = await noRole.PostAsJsonAsync($"/api/v1/cases/{caseId}/notes", new { text = "x" });

        Assert.Equal(HttpStatusCode.Forbidden, assign.StatusCode);
        Assert.Equal(HttpStatusCode.Forbidden, note.StatusCode);
    }
}
```

If `CaseResponse` serializes its id under a different name than `id`, adjust the property name to match `CasesController.ToResponse`.

- [ ] **Step 2: Run it and confirm it fails**

Run: `dotnet test tests/Integration --filter AlertWorkRbacTests`
Expected: the forbidden cases FAIL with `OK`/`Created` instead of `Forbidden`.

- [ ] **Step 3: Implement**

In `TenantAuthorization.cs`, add and reuse it for case closing:

```csharp
    public static bool CanWorkAlerts(ITenantContext tenant)
        => tenant.IsAuthenticated
           && (IsAdmin(tenant)
               || tenant.Roles.Any(r => string.Equals(r, RoleNames.Analyst, StringComparison.OrdinalIgnoreCase)));

    public static bool CanCloseCases(ITenantContext tenant) => CanWorkAlerts(tenant);
```

(Replace the existing `CanCloseCases` body.) In each of the six actions, directly after the `Unauthorized()` line, add:

```csharp
        if (!TenantAuthorization.CanWorkAlerts(_tenant)) return Forbid();
```

Add `using Aegis.Shared.Security;` to `AlertsController.cs` if it isn't there.

- [ ] **Step 4: Run all tests**

Run: `dotnet test`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add src tests
git commit -m "fix(authz): require Admin or Analyst for alert and case work actions"
```

---

### Task 4: Reject login for non-active tenants

**Files:**
- Modify: `src/Aegis.Modules.Identity/Domain/Tenant.cs` (add `Suspend()`)
- Modify: `src/Aegis.Api/Controllers/AuthController.cs:47-51`
- Test: `tests/Integration/Identity/TenantStatusLoginTests.cs`

**Interfaces:**
- Produces: `public void Tenant.Suspend()`; also needs a way to persist it in the test. Add `Task UpdateAsync(Tenant tenant, CancellationToken cancellationToken = default);` to `ITenantRepository` and implement it in `Repositories.cs` the same way `UserRepository.UpdateAsync` is implemented.

- [ ] **Step 1: Write the failing test**

```csharp
namespace Aegis.Tests.Integration.Identity;

using System.Net;
using System.Net.Http.Json;
using Aegis.Modules.Identity.Application;
using Aegis.Shared.Persistence;
using Microsoft.Extensions.DependencyInjection;

public sealed class TenantStatusLoginTests : IAsyncLifetime
{
    private static readonly string ConnectionString =
        Environment.GetEnvironmentVariable("AEGIS_TEST_CONNECTION")
        ?? "Host=localhost;Port=5432;Database=aegis_test;Username=aegis;Password=aegis_dev_password";

    private AegisApiFactory _factory = null!;
    private HttpClient _client = null!;
    private string _slug = null!;

    public async Task InitializeAsync()
    {
        _factory = new AegisApiFactory(ConnectionString);
        _client = _factory.CreateClient();
        _slug = $"susp-{Guid.NewGuid():N}"[..16];
        var bootstrap = await _client.PostAsJsonAsync("/api/v1/tenants", new
        {
            name = "Suspended Tenant",
            slug = _slug,
            adminEmail = $"admin@{_slug}.test",
            adminName = "Admin",
            adminPassword = "Passw0rd!"
        });
        bootstrap.EnsureSuccessStatusCode();
    }

    public Task DisposeAsync()
    {
        _client.Dispose();
        _factory.Dispose();
        return Task.CompletedTask;
    }

    [Fact]
    public async Task Suspended_tenant_users_cannot_log_in()
    {
        await using (var scope = _factory.Services.CreateAsyncScope())
        {
            var tenants = scope.ServiceProvider.GetRequiredService<ITenantRepository>();
            var uow = scope.ServiceProvider.GetRequiredService<IUnitOfWork>();
            var tenant = (await tenants.GetBySlugAsync(_slug))!;
            tenant.Suspend();
            await tenants.UpdateAsync(tenant);
            await uow.SaveChangesAsync();
        }

        var login = await _client.PostAsJsonAsync("/api/v1/auth/login", new
        {
            email = $"admin@{_slug}.test",
            password = "Passw0rd!",
            tenantSlug = _slug
        });

        Assert.Equal(HttpStatusCode.Unauthorized, login.StatusCode);
    }
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `dotnet test tests/Integration --filter TenantStatusLoginTests`
Expected: build error on `Suspend` / `UpdateAsync`. After adding those two members (Step 3a), it fails with `OK`.

- [ ] **Step 3a: Add the domain method and repository update**

In `Tenant.cs`:

```csharp
    public void Suspend()
    {
        Status = TenantStatus.SUSPENDED;
        UpdatedAt = DateTimeOffset.UtcNow;
    }
```

Add `UpdateAsync` to `ITenantRepository` and its implementation in `Repositories.cs`, mirroring `UserRepository.UpdateAsync`.

- [ ] **Step 3b: Enforce in login**

In `AuthController.Login`, replace the tenant null check with:

```csharp
        if (tenant is null || tenant.Status != TenantStatus.ACTIVE)
        {
            return Unauthorized();
        }
```

- [ ] **Step 4: Run all tests**

Run: `dotnet test`
Expected: all pass. Bootstrap activates tenants, so existing tests are unaffected.

- [ ] **Step 5: Commit**

```bash
git add src tests
git commit -m "fix(auth): reject login for suspended, onboarding, or terminated tenants"
```

---

### Task 5: CORS allowlist from configuration

**Files:**
- Modify: `src/Aegis.Api/Program.cs:43-44`, `src/Aegis.Api/appsettings.json`, create `src/Aegis.Api/appsettings.Development.json` if missing (otherwise add the key to it)
- Modify: `tests/Integration/Identity/AuthAndAuditTests.cs` (factory accepts overrides)
- Test: `tests/Integration/Identity/CorsTests.cs`

**Interfaces:**
- Produces: config key `Cors:AllowedOrigins` (string array). `AegisApiFactory(string connectionString, IDictionary<string, string?>? overrides = null)` — overrides are added after the defaults.

- [ ] **Step 1: Let the test factory take overrides**

In `tests/Integration/Identity/AuthAndAuditTests.cs`, change `AegisApiFactory`:

```csharp
public sealed class AegisApiFactory : WebApplicationFactory<Program>
{
    private readonly string _connectionString;
    private readonly IDictionary<string, string?> _overrides;

    public AegisApiFactory(string connectionString, IDictionary<string, string?>? overrides = null)
    {
        _connectionString = connectionString;
        _overrides = overrides ?? new Dictionary<string, string?>();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Development");
        builder.ConfigureAppConfiguration((_, config) =>
        {
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ConnectionStrings:Aegis"] = _connectionString,
                ["Jwt:SigningKey"] = "dev-only-signing-key-change-me-32chars-min!!",
                ["Jwt:Issuer"] = "aegis",
                ["Jwt:Audience"] = "aegis-api",
                ["Aegis:AllowDevBootstrap"] = "true"
            });
        });
        // Program.cs reads CORS and rate-limit settings before Build(); only UseSetting is visible that early.
        foreach (var (key, value) in _overrides)
        {
            builder.UseSetting(key, value);
        }
    }
}
```

- [ ] **Step 2: Write the failing test**

```csharp
namespace Aegis.Tests.Integration.Identity;

public sealed class CorsTests : IDisposable
{
    private static readonly string ConnectionString =
        Environment.GetEnvironmentVariable("AEGIS_TEST_CONNECTION")
        ?? "Host=localhost;Port=5432;Database=aegis_test;Username=aegis;Password=aegis_dev_password";

    private readonly AegisApiFactory _factory = new(ConnectionString, new Dictionary<string, string?>
    {
        ["Cors:AllowedOrigins:0"] = "http://localhost:3000"
    });

    public void Dispose() => _factory.Dispose();

    private async Task<HttpResponseMessage> PreflightAsync(string origin)
    {
        using var client = _factory.CreateClient();
        var request = new HttpRequestMessage(HttpMethod.Options, "/api/v1/auth/login");
        request.Headers.Add("Origin", origin);
        request.Headers.Add("Access-Control-Request-Method", "POST");
        return await client.SendAsync(request);
    }

    [Fact]
    public async Task Allowed_origin_gets_cors_header()
    {
        var response = await PreflightAsync("http://localhost:3000");
        Assert.Equal("http://localhost:3000",
            response.Headers.GetValues("Access-Control-Allow-Origin").Single());
    }

    [Fact]
    public async Task Unknown_origin_gets_no_cors_header()
    {
        var response = await PreflightAsync("https://evil.example");
        Assert.False(response.Headers.Contains("Access-Control-Allow-Origin"));
    }
}
```

- [ ] **Step 3: Run it and confirm it fails**

Run: `dotnet test tests/Integration --filter CorsTests`
Expected: `Unknown_origin_gets_no_cors_header` FAILS (header is `*`), and `Allowed_origin_gets_cors_header` FAILS (header is `*`, not the origin).

- [ ] **Step 4: Implement**

Replace `Program.cs` lines 43–44:

```csharp
    var allowedOrigins = builder.Configuration.GetSection("Cors:AllowedOrigins").Get<string[]>() ?? [];
    builder.Services.AddCors(o => o.AddPolicy("Default", p =>
        p.WithOrigins(allowedOrigins).AllowAnyMethod().AllowAnyHeader()));
```

In `appsettings.json` add `"Cors": { "AllowedOrigins": [] }`. In `appsettings.Development.json` add `"Cors": { "AllowedOrigins": [ "http://localhost:3000" ] }` (the console's dev port).

- [ ] **Step 5: Run all tests, then commit**

Run: `dotnet test` — expected all pass.

```bash
git add src tests
git commit -m "fix(api): restrict CORS to configured origins"
```

---

### Task 6: Login rate limiting

**Files:**
- Modify: `src/Aegis.Api/Program.cs`, `src/Aegis.Api/Controllers/AuthController.cs`, `src/Aegis.Api/appsettings.json`
- Test: `tests/Integration/Identity/LoginRateLimitTests.cs`

**Interfaces:**
- Consumes: `AegisApiFactory` overrides from Task 5.
- Produces: rate limiter policy name `"login"`; config keys `RateLimiting:Login:PermitLimit` (default 10) and `RateLimiting:Login:WindowSeconds` (default 60). Partitioned by client IP.

- [ ] **Step 1: Write the failing test**

```csharp
namespace Aegis.Tests.Integration.Identity;

using System.Net;
using System.Net.Http.Json;

public sealed class LoginRateLimitTests : IDisposable
{
    private static readonly string ConnectionString =
        Environment.GetEnvironmentVariable("AEGIS_TEST_CONNECTION")
        ?? "Host=localhost;Port=5432;Database=aegis_test;Username=aegis;Password=aegis_dev_password";

    private readonly AegisApiFactory _factory = new(ConnectionString, new Dictionary<string, string?>
    {
        ["RateLimiting:Login:PermitLimit"] = "3",
        ["RateLimiting:Login:WindowSeconds"] = "60"
    });

    public void Dispose() => _factory.Dispose();

    [Fact]
    public async Task Fourth_login_attempt_in_window_is_rejected_with_429()
    {
        using var client = _factory.CreateClient();
        var body = new { email = "nobody@x.test", password = "wrong", tenantSlug = "no-such-tenant" };

        for (var i = 0; i < 3; i++)
        {
            var r = await client.PostAsJsonAsync("/api/v1/auth/login", body);
            Assert.Equal(HttpStatusCode.Unauthorized, r.StatusCode);
        }

        var limited = await client.PostAsJsonAsync("/api/v1/auth/login", body);
        Assert.Equal(HttpStatusCode.TooManyRequests, limited.StatusCode);
    }
}
```

- [ ] **Step 2: Run it and confirm it fails**

Run: `dotnet test tests/Integration --filter LoginRateLimitTests`
Expected: FAIL, fourth response is `Unauthorized`.

- [ ] **Step 3: Implement**

In `Program.cs`, add `using System.Threading.RateLimiting;` and `using Microsoft.AspNetCore.RateLimiting;`, then after `AddCors`:

```csharp
    var loginPermitLimit = builder.Configuration.GetValue("RateLimiting:Login:PermitLimit", 10);
    var loginWindow = TimeSpan.FromSeconds(builder.Configuration.GetValue("RateLimiting:Login:WindowSeconds", 60));
    builder.Services.AddRateLimiter(o =>
    {
        o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
        o.AddPolicy("login", http => RateLimitPartition.GetFixedWindowLimiter(
            http.Connection.RemoteIpAddress?.ToString() ?? "unknown",
            _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = loginPermitLimit,
                Window = loginWindow,
                QueueLimit = 0
            }));
    });
```

After `app.UseCors("Default");` add `app.UseRateLimiter();`. On `AuthController.Login` add `[EnableRateLimiting("login")]` (with `using Microsoft.AspNetCore.RateLimiting;`). Add to `appsettings.json`:

```json
  "RateLimiting": { "Login": { "PermitLimit": 10, "WindowSeconds": 60 } },
```

Other tests log in a handful of times per factory instance, well under 10, so they're unaffected. If a suite trips the limit, raise `PermitLimit` in that suite's factory overrides rather than in the defaults.

- [ ] **Step 4: Run all tests, then commit**

Run: `dotnet test` — expected all pass.

```bash
git add src tests
git commit -m "feat(api): rate limit login attempts per client IP"
```

---

### Task 7: Startup migrations behind config, Dockerfile restore fix

**Files:**
- Modify: `src/Aegis.Api/Program.cs:48-52`, `src/Aegis.Api/appsettings.json`, `src/Aegis.Api/appsettings.Development.json`, `Dockerfile`, `README.md` (run instructions)

**Interfaces:**
- Produces: config key `Database:MigrateOnStartup` (bool). `false` in `appsettings.json`, `true` in `appsettings.Development.json`. Test factories run as `Development`, so they keep migrating.

- [ ] **Step 1: Gate the migration**

Replace the migration block in `Program.cs`:

```csharp
    if (app.Configuration.GetValue("Database:MigrateOnStartup", false))
    {
        using var scope = app.Services.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AegisDbContext>();
        db.Database.Migrate();
    }
```

Add `"Database": { "MigrateOnStartup": false }` to `appsettings.json` and `"Database": { "MigrateOnStartup": true }` to `appsettings.Development.json`.

- [ ] **Step 2: Fix the Dockerfile restore layer**

Add these lines with the other `COPY ... .csproj` lines, before `RUN dotnet restore`:

```dockerfile
COPY src/Aegis.Application/Aegis.Application.csproj src/Aegis.Application/
COPY src/Aegis.Modules.Identity/Aegis.Modules.Identity.csproj src/Aegis.Modules.Identity/
COPY src/Aegis.Modules.Customers/Aegis.Modules.Customers.csproj src/Aegis.Modules.Customers/
```

- [ ] **Step 3: Document production migrations**

In `README.md`, under the run instructions, add that non-Development environments must run migrations explicitly before deploying:

```bash
dotnet ef database update --project src/Aegis.Infrastructure --startup-project src/Aegis.Api
```

- [ ] **Step 4: Verify**

Run: `dotnet test` — expected all pass.
Run: `docker build -t aegis-api:phase0 .` — expected success, and the restore step is cached on a second build after touching only a `.cs` file.

- [ ] **Step 5: Commit**

```bash
git add src Dockerfile README.md
git commit -m "chore(api): opt-in startup migrations and complete Dockerfile restore layer"
```

---

## Phase 0 exit checklist

Completed 2026-10-01 on `aegis` branch `phase-0-hardening` (7 commits, 86 tests green, up from 72 passing + 1 failing).

- [x] `dotnet build` and `dotnet test` green locally (CI runs on push).
- [x] `rg -n '\$"\{\{' src -g '*.cs'` returns nothing.
- [x] A user with no roles gets 403 on every alert and case work action.
- [x] Suspended-tenant login returns 401.
- [x] Unknown origins receive no CORS header (verified in tests and against the running API).
- [x] 11th login attempt within a minute from one IP returns 429 with default settings (verified against the running API).
- [x] `docker build` succeeds.
- [ ] Phase 1 PR 1.1 plan written against the post-Phase-0 code.

### What changed versus this plan

- **Baseline failure fixed first.** `Tenant_bootstrap_is_blocked_when_not_allowed` failed on clean `main`: settings added through `ConfigureAppConfiguration` aren't visible to `Program.cs` before `Build()`, so the Production host had no signing key. Tests now pass early-read settings with `UseSetting`, and the factory overrides in Task 5 do the same.
- **Audit injection was worse than predicted.** Postgres accepted the injected JSON rather than rejecting it, so forged keys (for example `isAdmin`) were being stored in audit rows. Rows written before this fix may contain injected fields.
- **Dev settings live in `launchSettings.json`, not `appsettings.Development.json`.** The latter is gitignored. The console origin is set there as `Cors__AllowedOrigins__0`.
- **`Database:MigrateOnStartup` defaults to the environment** (`true` in Development, `false` elsewhere) instead of separate appsettings files.

# Keycloak Bulk User Creation — C# Minimal API Guide

This guide shows how to bulk-create Keycloak users from a mock “database” in **C# (.NET 8)**, set their initial passwords to each person’s **nationalId** (as a **temporary** password), and force a **password change at first login**.

You’ll get three files:

- `PersonDatabase.cs` — mock data source
- `KeycloakUserServices.cs` — service that talks to Keycloak Admin REST API
- `UserEndpoints.cs` — minimal API endpoints to trigger the bulk import

At the end, there’s a quick scaffold for `Program.cs` and `appsettings.json`.

---

## Prerequisites

- **.NET 8 SDK**
- A running **Keycloak** instance (URL, admin username/password)
- Keycloak realm (e.g., `my-realm`), and the default admin client `admin-cli` in the admin realm (`master`)

> For security, use HTTPS in real environments and protect any files containing admin credentials.

---

## 1) Data Source — `PersonDatabase.cs`

A simple mock list of people we plan to import. Replace this with your real DB access later.

```csharp
// PersonDatabase.cs
// Mock data source for people to import into Keycloak.
using System.Collections.Immutable;

namespace MyApp.Data;

public static class PersonDatabase
{
    // Domain model used by the importer
    public sealed record Person(string Name, string LastName, string Email, string NationalId);

    // In a real app you would hit a DB; here we hardcode a few rows.
    public static readonly IReadOnlyList<Person> People = new List<Person>
    {
        new("Ana", "Gómez", "ana@example.com", "12345678"),
        new("Juan", "Pérez", "juan@example.com", "87654321"),
        new("Lucía", "Martínez", "lucia@example.com", "44556677"),
        new("Carlos", "Rojas", "carlos@example.com", "99887766")
    }.ToImmutableList();
}
```

**What this does**
- Defines a `Person` record with `Name`, `LastName`, `Email`, `NationalId`.
- Exposes an in-memory list `People`. You can swap this for a database query later.

---

## 2) Keycloak Admin Service — `KeycloakUserServices.cs`

This service authenticates to the **Keycloak Admin API**, creates users, sets a **temporary** password equal to `nationalId`, and adds the required action **UPDATE_PASSWORD**.

```csharp
// KeycloakUserServices.cs
// A small service that calls Keycloak Admin REST API to create users,
// set a temporary password (nationalId), and force a password reset.
using System.Net.Http.Headers;
using System.Net.Http.Json;
using System.Text.Json;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Options;
using MyApp.Data;

namespace MyApp.Keycloak;

public sealed class KeycloakOptions
{
    public string BaseUrl { get; set; } = "http://localhost:8080";
    public string AdminRealm { get; set; } = "master";
    public string Realm { get; set; } = "my-realm";
    public string AdminClientId { get; set; } = "admin-cli";
    public string AdminUsername { get; set; } = "admin";
    public string AdminPassword { get; set; } = "admin";
    /// <summary>"email" or "nationalId" for username.</summary>
    public string UsernameField { get; set; } = "email";
    /// <summary>Required actions to add after creation.</summary>
    public string[] RequiredActions { get; set; } = new[] { "UPDATE_PASSWORD" };
}

public interface IKeycloakUserService
{
    Task BulkCreateAsync(IEnumerable<PersonDatabase.Person> people, CancellationToken ct = default);
    Task<string> CreateOrGetUserIdAsync(string username, string first, string last, string email, CancellationToken ct = default);
    Task SetTemporaryPasswordAsync(string userId, string tempPassword, CancellationToken ct = default);
    Task RequireActionsAsync(string userId, IEnumerable<string> actions, CancellationToken ct = default);
}

public sealed class KeycloakUserService : IKeycloakUserService
{
    private readonly HttpClient _http;
    private readonly KeycloakOptions _opt;

    private string? _token;
    private DateTimeOffset _tokenExpiry;

    public KeycloakUserService(HttpClient http, IOptions<KeycloakOptions> options)
    {
        _http = http;
        _opt = options.Value;
    }

    public async Task BulkCreateAsync(IEnumerable<PersonDatabase.Person> people, CancellationToken ct = default)
    {
        await EnsureTokenAsync(ct);
        foreach (var p in people)
        {
            var username = _opt.UsernameField.Equals("nationalid", StringComparison.OrdinalIgnoreCase)
                ? p.NationalId : p.Email;

            if (string.IsNullOrWhiteSpace(username))
                continue;

            var id = await CreateOrGetUserIdAsync(username, p.Name, p.LastName, p.Email, ct);
            await SetTemporaryPasswordAsync(id, p.NationalId, ct);
            await RequireActionsAsync(id, _opt.RequiredActions, ct);
        }
    }

    public async Task<string> CreateOrGetUserIdAsync(string username, string first, string last, string email, CancellationToken ct = default)
    {
        await EnsureTokenAsync(ct);

        // Check existing
        var existing = await _http.GetFromJsonAsync<List<JsonElement>>(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users?username={Uri.EscapeDataString(username)}", ct);

        if (existing is { Count: > 0 })
            return existing[0].GetProperty("id").GetString()!;

        // Create
        var payload = new
        {
            username,
            firstName = first,
            lastName = last,
            email,
            enabled = true,
            emailVerified = false
        };

        using var res = await _http.PostAsJsonAsync(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users", payload, ct);

        if (res.StatusCode != System.Net.HttpStatusCode.Conflict)
            res.EnsureSuccessStatusCode();

        // Fetch ID
        existing = await _http.GetFromJsonAsync<List<JsonElement>>(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users?username={Uri.EscapeDataString(username)}", ct);

        return existing![0].GetProperty("id").GetString()!;
    }

    public async Task SetTemporaryPasswordAsync(string userId, string tempPassword, CancellationToken ct = default)
    {
        await EnsureTokenAsync(ct);
        var payload = new { type = "password", value = tempPassword, temporary = true };
        using var res = await _http.PutAsJsonAsync(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users/{Uri.EscapeDataString(userId)}/reset-password", payload, ct);
        res.EnsureSuccessStatusCode();
    }

    public async Task RequireActionsAsync(string userId, IEnumerable<string> actions, CancellationToken ct = default)
    {
        await EnsureTokenAsync(ct);
        var payload = new { requiredActions = actions.ToArray() };
        using var res = await _http.PutAsJsonAsync(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users/{Uri.EscapeDataString(userId)}", payload, ct);
        res.EnsureSuccessStatusCode();
    }

    private async Task EnsureTokenAsync(CancellationToken ct)
    {
        if (_token is not null && DateTimeOffset.UtcNow < _tokenExpiry) return;

        using var req = new HttpRequestMessage(HttpMethod.Post,
            $"/realms/{Uri.EscapeDataString(_opt.AdminRealm)}/protocol/openid-connect/token")
        {
            Content = new FormUrlEncodedContent(new Dictionary<string, string>
            {
                ["grant_type"] = "password",
                ["client_id"] = _opt.AdminClientId,
                ["username"] = _opt.AdminUsername,
                ["password"] = _opt.AdminPassword
            })
        };

        using var res = await _http.SendAsync(req, ct);
        res.EnsureSuccessStatusCode();

        var json = await res.Content.ReadFromJsonAsync<JsonElement>(cancellationToken: ct);
        _token = json.GetProperty("access_token").GetString();
        var expiresIn = json.TryGetProperty("expires_in", out var exp) ? exp.GetInt32() : 60;
        _tokenExpiry = DateTimeOffset.UtcNow.AddSeconds(expiresIn - 10);

        _http.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", _token);
        if (!_http.DefaultRequestHeaders.Accept.Any(h => h.MediaType == "application/json"))
            _http.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
    }
}

// DI helper
public static class KeycloakServiceCollectionExtensions
{
    public static IServiceCollection AddKeycloakUserService(this IServiceCollection services, IConfiguration config)
    {
        services.Configure<KeycloakOptions>(config.GetSection("Keycloak"));
        services.AddHttpClient<IKeycloakUserService, KeycloakUserService>((sp, http) =>
        {
            var opt = sp.GetRequiredService<IOptions<KeycloakOptions>>().Value;
            http.BaseAddress = new Uri(opt.BaseUrl.TrimEnd('/'));
        });
        return services;
    }
}
```

**How it works**

1. **Options** (`KeycloakOptions`) carry Keycloak URLs, admin credentials, target realm, and behavior (username field and required actions).
2. `EnsureTokenAsync` obtains an **admin access token** and sets the `Authorization: Bearer` header.
3. `CreateOrGetUserIdAsync` idempotently creates a user and returns the Keycloak `id`.
4. `SetTemporaryPasswordAsync` sets `temporary=true`, which forces a password prompt.
5. `RequireActionsAsync` adds **UPDATE_PASSWORD** (and any others you include).

> You can add more required actions like `"VERIFY_EMAIL"` if your flow needs email verification.

---

## 3) Minimal API — `UserEndpoints.cs`

A single endpoint to bulk-create from our mock database.

```csharp
// UserEndpoints.cs
// Minimal API endpoints that trigger the bulk creation using PersonDatabase.
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using MyApp.Data;
using MyApp.Keycloak;

namespace MyApp.Api;

public static class UserEndpoints
{
    /// <summary>
    /// Map endpoints under /users.
    /// </summary>
    public static IEndpointRouteBuilder MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        // POST /users/bulk-create -> uses mock PersonDatabase
        app.MapPost("/users/bulk-create", async (IKeycloakUserService service, CancellationToken ct) =>
        {
            await service.BulkCreateAsync(PersonDatabase.People, ct);
            return Results.Ok(new { imported = PersonDatabase.People.Count });
        })
        .WithName("BulkCreateUsers")
        .WithSummary("Bulk-create Keycloak users from the mock PersonDatabase.")
        .WithDescription("Creates users, sets nationalId as a TEMP password, and requires UPDATE_PASSWORD on next login.");

        return app;
    }
}

/*
Usage in Program.cs:

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddKeycloakUserService(builder.Configuration);
var app = builder.Build();
app.MapUserEndpoints();
app.Run();

appsettings.json example:
{
  "Keycloak": {
    "BaseUrl": "http://localhost:8080",
    "AdminRealm": "master",
    "Realm": "my-realm",
    "AdminClientId": "admin-cli",
    "AdminUsername": "admin",
    "AdminPassword": "admin",
    "UsernameField": "email",
    "RequiredActions": [ "UPDATE_PASSWORD" ]
  }
}
*/
```

**What this does**
- Exposes `POST /users/bulk-create` which calls the service with `PersonDatabase.People`.
- Returns `{ imported: N }` for a quick confirmation.

---

## 4) Wire-up — `Program.cs` (scaffold)

Create a minimal host, add the service, and map the endpoint.

```csharp
using MyApp.Api;
using MyApp.Keycloak;

var builder = WebApplication.CreateBuilder(args);

// Bind "Keycloak" section from appsettings.json (or env vars) to KeycloakOptions
builder.Services.AddKeycloakUserService(builder.Configuration);

var app = builder.Build();
app.MapUserEndpoints();
app.Run();
```

**NuGet references**  
Add to your project file (e.g., `MyApp.csproj`) if they’re not already present:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Http" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="8.0.0" />
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.AspNetCore.App" Version="8.0.0" />
</ItemGroup>
```

---

## 5) Configuration — `appsettings.json` (scaffold)

```json
{
  "Keycloak": {
    "BaseUrl": "http://localhost:8080",
    "AdminRealm": "master",
    "Realm": "my-realm",
    "AdminClientId": "admin-cli",
    "AdminUsername": "admin",
    "AdminPassword": "admin",
    "UsernameField": "email",
    "RequiredActions": [ "UPDATE_PASSWORD" ]
  }
}
```

> You can override any setting via environment variables (e.g., `Keycloak__BaseUrl`, `Keycloak__AdminPassword`).

---

## 6) Running the API

1. Create a new web project and add the three files above plus the `Program.cs`.
2. Put the `appsettings.json` in the same project root or use environment variables.
3. Run:
   ```bash
   dotnet run
   ```
4. Trigger the import:
   ```bash
   curl -X POST http://localhost:5000/users/bulk-create
   ```
   (Adjust port to your Kestrel binding.)

---

## Security & Policy Notes

- **Temporary passwords**: Using `temporary=true` ensures users must set a new password on first login.
- **Password policy**: If national IDs are too short or numeric-only, Keycloak may reject them. Temporarily relax the realm policy or transform the temp password (e.g., `nid + "!Temp2025"`).
- **Transport security**: Prefer **HTTPS** when calling Keycloak and hosting this API.
- **Credentials**: Store admin credentials securely (secrets manager, env vars, or KeyVault).

---

## Troubleshooting Tips

- **401/403**: Confirm the admin token was acquired (`admin-cli` enabled, correct realm/credentials).
- **409 Conflict** on user creation: User already exists — the service is idempotent and fetches the ID afterwards.
- **Password rejected**: Check **Authentication → Password Policy** in your Keycloak realm.
- **Endpoint base path**: Ensure `BaseUrl` is correct and includes the port (e.g., `http://localhost:8080`).

---

## Next steps

- Replace `PersonDatabase.People` with your real DB query or an external file (CSV/JSON).
- Add role mappings or group assignments after creation using the relevant Admin endpoints.
- Add logging, retries, and observability as needed.

# Creación masiva de usuarios en Keycloak — Guía con C# y Minimal API

Esta guía muestra cómo **crear usuarios en Keycloak en lote** desde una “base de datos” simulada en **C# (.NET 8)**, establecer su **contraseña inicial** con el **número de documento (nationalId)** como **temporal**, y **forzar el cambio de contraseña** en el primer inicio de sesión.

Obtendrás tres archivos:

- `PersonDatabase.cs` — fuente de datos simulada
- `KeycloakUserServices.cs` — servicio que llama al **Admin REST API** de Keycloak
- `UserEndpoints.cs` — Minimal API con un endpoint para disparar la importación masiva

Al final, hay un andamiaje rápido para `Program.cs` y `appsettings.json`.

---

## Requisitos previos

- **.NET 8 SDK**
- Una instancia de **Keycloak** en ejecución (URL, usuario y contraseña de administrador)
- Un **realm** (p. ej., `my-realm`) y el cliente admin por defecto `admin-cli` en el realm de administración (`master`).

> Por seguridad, usa **HTTPS** en ambientes reales y protege cualquier archivo que contenga credenciales de administrador.

---

## 1) Fuente de datos — `PersonDatabase.cs`

Una lista simple de personas que importaremos. Sustitúyela por tu acceso real a base de datos más adelante.

```csharp
// PersonDatabase.cs
// Fuente de datos simulada para importar personas a Keycloak.
using System.Collections.Immutable;

namespace MyApp.Data;

public static class PersonDatabase
{
    // Modelo de dominio usado por el importador
    public sealed record Person(string Name, string LastName, string Email, string NationalId);

    // En una app real consultarías una BD; aquí dejamos algunos registros fijos.
    public static readonly IReadOnlyList<Person> People = new List<Person>
    {
        new("Ana", "Gómez", "ana@example.com", "12345678"),
        new("Juan", "Pérez", "juan@example.com", "87654321"),
        new("Lucía", "Martínez", "lucia@example.com", "44556677"),
        new("Carlos", "Rojas", "carlos@example.com", "99887766")
    }.ToImmutableList();
}
```

**Qué hace**
- Define el record `Person` con `Name`, `LastName`, `Email`, `NationalId`.
- Expone una lista en memoria `People`. Puedes reemplazarla por una consulta real a BD.

---

## 2) Servicio Admin de Keycloak — `KeycloakUserServices.cs`

Este servicio se autentica contra el **Admin API** de Keycloak, crea usuarios, establece una contraseña **temporal** igual al `nationalId`, y agrega la acción requerida **UPDATE_PASSWORD**.

```csharp
// KeycloakUserServices.cs
// Servicio que llama al Admin REST API de Keycloak para crear usuarios,
// fijar una contraseña temporal (nationalId) y forzar su actualización.
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
    /// <summary>"email" o "nationalId" para el nombre de usuario.</summary>
    public string UsernameField { get; set; } = "email";
    /// <summary>Acciones requeridas a agregar tras la creación.</summary>
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

        // Verificar existente
        var existing = await _http.GetFromJsonAsync<List<JsonElement>>(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users?username={Uri.EscapeDataString(username)}", ct);

        if (existing is { Count: > 0 })
            return existing[0].GetProperty("id").GetString()!;

        // Crear
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

        // Obtener ID
        existing = await _http.GetFromJsonAsync<List<JsonElement>>(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users?username={Uri.EscapeDataString(username)}", ct);

        return existing![0].GetProperty("id").GetString()!;
    }

    public async Task SetTemporaryPasswordAsync(string userId, string tempPassword, CancellationToken ct = default)
    {
        await EnsureTokenAsync(ct);
        var payload = new { type = "password", value = tempPassword, temporary = true };
        using var res = await _http.PutAsJsonAsync(
            $"/admin/realms/{Uri.EscapeDataString(_opt.Realm)}/users/{Uri.EcapeDataString(userId)}/reset-password", payload, ct);
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

// Extensión para DI
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

**Cómo funciona**

1. **Opciones** (`KeycloakOptions`) guardan URL de Keycloak, credenciales admin, realm destino y preferencias (campo para username y acciones requeridas).
2. `EnsureTokenAsync` obtiene un **token de administrador** y configura el header `Authorization: Bearer`.
3. `CreateOrGetUserIdAsync` crea el usuario de forma idempotente y devuelve el `id`.
4. `SetTemporaryPasswordAsync` establece `temporary=true`, lo que obliga al cambio de contraseña.
5. `RequireActionsAsync` agrega **UPDATE_PASSWORD** (y otras que definas).

> Puedes sumar acciones como `"VERIFY_EMAIL"` si tu flujo lo requiere.

---

## 3) Minimal API — `UserEndpoints.cs`

Un endpoint para crear en lote usando nuestra base simulada.

```csharp
// UserEndpoints.cs
// Endpoints Minimal API que disparan la creación masiva usando PersonDatabase.
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Routing;
using MyApp.Data;
using MyApp.Keycloak;

namespace MyApp.Api;

public static class UserEndpoints
{
    /// <summary>
    /// Mapea endpoints bajo /users.
    /// </summary>
    public static IEndpointRouteBuilder MapUserEndpoints(this IEndpointRouteBuilder app)
    {
        // POST /users/bulk-create -> usa la base simulada PersonDatabase
        app.MapPost("/users/bulk-create", async (IKeycloakUserService service, CancellationToken ct) =>
        {
            await service.BulkCreateAsync(PersonDatabase.People, ct);
            return Results.Ok(new { imported = PersonDatabase.People.Count });
        })
        .WithName("BulkCreateUsers")
        .WithSummary("Crea usuarios en Keycloak a partir de PersonDatabase (simulada).")
        .WithDescription("Crea usuarios, fija nationalId como contraseña TEMPORAL y exige UPDATE_PASSWORD al iniciar sesión.");

        return app;
    }
}

/*
Uso en Program.cs:

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddKeycloakUserService(builder.Configuration);
var app = builder.Build();
app.MapUserEndpoints();
app.Run();

Ejemplo de appsettings.json:
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

**Qué hace**
- Expone `POST /users/bulk-create` que llama al servicio con `PersonDatabase.People`.
- Devuelve `{ imported: N }` para confirmar rápidamente.

---

## 4) Conexión — `Program.cs` (andamiaje)

Crea el host mínimo, registra el servicio y mapea el endpoint.

```csharp
using MyApp.Api;
using MyApp.Keycloak;

var builder = WebApplication.CreateBuilder(args);

// Vincula la sección "Keycloak" de appsettings.json (o variables de entorno)
builder.Services.AddKeycloakUserService(builder.Configuration);

var app = builder.Build();
app.MapUserEndpoints();
app.Run();
```

**Referencias NuGet**  
Agrega a tu `.csproj` si aún no están presentes:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.Http" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Options.ConfigurationExtensions" Version="8.0.0" />
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.AspNetCore.App" Version="8.0.0" />
</ItemGroup>
```

---

## 5) Configuración — `appsettings.json` (andamiaje)

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

> Puedes sobrescribir cualquier ajuste vía variables de entorno (por ejemplo, `Keycloak__BaseUrl`, `Keycloak__AdminPassword`).

---

## 6) Ejecutar la API

1. Crea un proyecto web y agrega los tres archivos anteriores más `Program.cs`.
2. Coloca el `appsettings.json` en la raíz del proyecto o usa variables de entorno.
3. Ejecuta:
   ```bash
   dotnet run
   ```
4. Dispara la importación:
   ```bash
   curl -X POST http://localhost:5000/users/bulk-create
   ```
   (Ajusta el puerto al binding de Kestrel).

---

## Notas de seguridad y políticas

- **Contraseñas temporales**: `temporary=true` garantiza que el usuario deba cambiar su contraseña al primer inicio.
- **Política de contraseñas**: Si los DNI son muy cortos o solo numéricos, Keycloak podría rechazarlos. Relaja temporalmente la política del realm o transforma la contraseña temporal (p. ej., `dni + "!Temp2025"`).
- **Transporte**: Prefiere **HTTPS** tanto para llamar a Keycloak como para hospedar esta API.
- **Credenciales**: Guarda credenciales en un gestor de secretos, variables de entorno o un servicio seguro.

---

## Siguientes pasos

- Reemplaza `PersonDatabase.People` por tu consulta real a BD o un archivo externo (CSV/JSON).
- Agrega asignación de **roles** o **grupos** tras la creación con los endpoints de administración correspondientes.
- Añade logging, reintentos y observabilidad según lo necesites.

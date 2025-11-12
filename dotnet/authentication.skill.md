# Skill: Authentication & Authorization en .NET

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET 6+ / ASP.NET Core  
**Tipo:** Skill Específico  
**Autor:** SWO Team

## Propósito

Este skill se enfoca exclusivamente en **autenticación y autorización** en aplicaciones .NET. Cubre implementación de JWT, ASP.NET Core Identity, políticas de autorización, y protección de endpoints.

## Lo que CUBRE este Skill

- ✅ Autenticación con JWT (JSON Web Tokens)
- ✅ ASP.NET Core Identity (usuarios, roles, claims)
- ✅ Autorización basada en roles y políticas
- ✅ Protección de endpoints (Controllers y Minimal APIs)
- ✅ Refresh tokens y renovación de sesiones
- ✅ Configuración de CORS para autenticación
- ✅ Claims personalizados
- ✅ Middleware de autenticación

## Lo que NO CUBRE este Skill

- ❌ Validación de datos de entrada → Ver `validation.skill.md`
- ❌ Configuración de base de datos → Ver `ef-core.skill.md`
- ❌ Estructura de endpoints REST → Ver `aspnet-webapi.skill.md`
- ❌ Dependency injection detallada → Ver `di-configuration.skill.md`
- ❌ Testing de autenticación → Ver `testing.skill.md`

## Autenticación con JWT

### Configuración Básica de JWT

**NuGet Packages requeridos:**
```
Microsoft.AspNetCore.Authentication.JwtBearer
System.IdentityModel.Tokens.Jwt
```

**appsettings.json:**
```json
{
  "JwtSettings": {
    "Secret": "your-256-bit-secret-key-here-minimum-32-characters",
    "Issuer": "MyApp",
    "Audience": "MyApp-Users",
    "ExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 7
  }
}
```

**Clase de configuración:**
```csharp
public class JwtSettings
{
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; }
    public int RefreshTokenExpirationDays { get; set; }
}
```

**Configuración en Program.cs:**
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Bind JWT settings
var jwtSettings = new JwtSettings();
builder.Configuration.GetSection("JwtSettings").Bind(jwtSettings);
builder.Services.Configure<JwtSettings>(builder.Configuration.GetSection("JwtSettings"));

// Configurar autenticación JWT
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.SaveToken = true;
    options.RequireHttpsMetadata = false; // true en producción
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings.Issuer,
        ValidAudience = jwtSettings.Audience,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtSettings.Secret)),
        ClockSkew = TimeSpan.Zero // Eliminar delay de 5 minutos por defecto
    };
    
    // Eventos opcionales para debugging
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception.GetType() == typeof(SecurityTokenExpiredException))
            {
                context.Response.Headers.Add("Token-Expired", "true");
            }
            return Task.CompletedTask;
        },
        OnTokenValidated = context =>
        {
            Console.WriteLine("Token validated successfully");
            return Task.CompletedTask;
        }
    };
});

builder.Services.AddAuthorization();

var app = builder.Build();

// Middleware en orden correcto
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

### Servicio de Generación de JWT

```csharp
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;

public interface ITokenService
{
    string GenerateAccessToken(Guid userId, string email, IEnumerable<string> roles);
    string GenerateRefreshToken();
    ClaimsPrincipal? GetPrincipalFromExpiredToken(string token);
}

public class TokenService : ITokenService
{
    private readonly JwtSettings _jwtSettings;

    public TokenService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }

    public string GenerateAccessToken(Guid userId, string email, IEnumerable<string> roles)
    {
        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
            new Claim(ClaimTypes.Email, email),
            new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString())
        };

        // Agregar roles como claims
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.Secret));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var token = new JwtSecurityToken(
            issuer: _jwtSettings.Issuer,
            audience: _jwtSettings.Audience,
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(_jwtSettings.ExpirationMinutes),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public string GenerateRefreshToken()
    {
        var randomNumber = new byte[32];
        using var rng = RandomNumberGenerator.Create();
        rng.GetBytes(randomNumber);
        return Convert.ToBase64String(randomNumber);
    }

    public ClaimsPrincipal? GetPrincipalFromExpiredToken(string token)
    {
        var tokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false,
            ValidateIssuer = false,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_jwtSettings.Secret)),
            ValidateLifetime = false // No validamos expiración aquí
        };

        var tokenHandler = new JwtSecurityTokenHandler();
        var principal = tokenHandler.ValidateToken(token, tokenValidationParameters, out var securityToken);

        if (securityToken is not JwtSecurityToken jwtSecurityToken ||
            !jwtSecurityToken.Header.Alg.Equals(SecurityAlgorithms.HmacSha256,
                StringComparison.InvariantCultureIgnoreCase))
        {
            throw new SecurityTokenException("Invalid token");
        }

        return principal;
    }
}
```

### Controller de Autenticación

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace MyApp.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly ITokenService _tokenService;
    private readonly IUserService _userService;
    private readonly ILogger<AuthController> _logger;

    public AuthController(
        ITokenService tokenService,
        IUserService userService,
        ILogger<AuthController> logger)
    {
        _tokenService = tokenService;
        _userService = userService;
        _logger = logger;
    }

    [HttpPost("login")]
    [AllowAnonymous]
    public async Task<ActionResult<AuthResponse>> Login(
        [FromBody] LoginRequest request,
        CancellationToken cancellationToken)
    {
        // Validar credenciales (normalmente con hash de password)
        var user = await _userService.ValidateCredentialsAsync(
            request.Email, 
            request.Password,
            cancellationToken);

        if (user is null)
        {
            _logger.LogWarning("Failed login attempt for {Email}", request.Email);
            return Unauthorized(new { message = "Invalid credentials" });
        }

        // Obtener roles del usuario
        var roles = await _userService.GetUserRolesAsync(user.Id, cancellationToken);

        // Generar tokens
        var accessToken = _tokenService.GenerateAccessToken(user.Id, user.Email, roles);
        var refreshToken = _tokenService.GenerateRefreshToken();

        // Guardar refresh token en BD
        await _userService.SaveRefreshTokenAsync(
            user.Id, 
            refreshToken, 
            DateTime.UtcNow.AddDays(7),
            cancellationToken);

        _logger.LogInformation("User {UserId} logged in successfully", user.Id);

        return Ok(new AuthResponse
        {
            AccessToken = accessToken,
            RefreshToken = refreshToken,
            ExpiresIn = 3600, // segundos
            TokenType = "Bearer"
        });
    }

    [HttpPost("refresh")]
    [AllowAnonymous]
    public async Task<ActionResult<AuthResponse>> RefreshToken(
        [FromBody] RefreshTokenRequest request,
        CancellationToken cancellationToken)
    {
        var principal = _tokenService.GetPrincipalFromExpiredToken(request.AccessToken);
        
        if (principal is null)
        {
            return Unauthorized(new { message = "Invalid access token" });
        }

        var userIdClaim = principal.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        if (userIdClaim is null || !Guid.TryParse(userIdClaim, out var userId))
        {
            return Unauthorized(new { message = "Invalid token claims" });
        }

        // Validar refresh token en BD
        var isValid = await _userService.ValidateRefreshTokenAsync(
            userId, 
            request.RefreshToken,
            cancellationToken);

        if (!isValid)
        {
            _logger.LogWarning("Invalid refresh token for user {UserId}", userId);
            return Unauthorized(new { message = "Invalid refresh token" });
        }

        // Obtener datos actualizados del usuario
        var user = await _userService.GetByIdAsync(userId, cancellationToken);
        if (user is null)
        {
            return Unauthorized(new { message = "User not found" });
        }

        var roles = await _userService.GetUserRolesAsync(userId, cancellationToken);

        // Generar nuevos tokens
        var newAccessToken = _tokenService.GenerateAccessToken(user.Id, user.Email, roles);
        var newRefreshToken = _tokenService.GenerateRefreshToken();

        // Actualizar refresh token en BD
        await _userService.UpdateRefreshTokenAsync(
            userId, 
            newRefreshToken, 
            DateTime.UtcNow.AddDays(7),
            cancellationToken);

        return Ok(new AuthResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken,
            ExpiresIn = 3600,
            TokenType = "Bearer"
        });
    }

    [HttpPost("logout")]
    [Authorize]
    public async Task<IActionResult> Logout(CancellationToken cancellationToken)
    {
        var userIdClaim = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        
        if (userIdClaim is not null && Guid.TryParse(userIdClaim, out var userId))
        {
            await _userService.RevokeRefreshTokenAsync(userId, cancellationToken);
            _logger.LogInformation("User {UserId} logged out", userId);
        }

        return NoContent();
    }
}

// DTOs
public record LoginRequest(string Email, string Password);

public record RefreshTokenRequest(string AccessToken, string RefreshToken);

public record AuthResponse
{
    public string AccessToken { get; init; } = string.Empty;
    public string RefreshToken { get; init; } = string.Empty;
    public int ExpiresIn { get; init; }
    public string TokenType { get; init; } = string.Empty;
}
```

## Protección de Endpoints

### Con Controllers

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
[Authorize] // Todos los endpoints requieren autenticación
public class ProductsController : ControllerBase
{
    [HttpGet]
    [AllowAnonymous] // Excepto este
    public async Task<IActionResult> GetAll() { /* ... */ }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(Guid id) { /* Requiere autenticación */ }

    [HttpPost]
    [Authorize(Roles = "Admin,Manager")] // Solo estos roles
    public async Task<IActionResult> Create([FromBody] CreateProductRequest request) { /* ... */ }

    [HttpPut("{id}")]
    [Authorize(Roles = "Admin")]
    public async Task<IActionResult> Update(Guid id, [FromBody] UpdateProductRequest request) { /* ... */ }

    [HttpDelete("{id}")]
    [Authorize(Roles = "Admin")]
    [Authorize(Policy = "CanDeleteProducts")] // Política adicional
    public async Task<IActionResult> Delete(Guid id) { /* ... */ }

    // Acceder a información del usuario autenticado
    [HttpGet("my-purchases")]
    public async Task<IActionResult> GetMyPurchases()
    {
        var userIdClaim = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var userId = Guid.Parse(userIdClaim!);
        
        var email = User.FindFirst(ClaimTypes.Email)?.Value;
        var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value);

        // Usar userId para obtener datos...
        return Ok(/* ... */);
    }
}
```

### Con Minimal APIs

```csharp
var app = builder.Build();

// Endpoint público
app.MapGet("/api/products", async (IProductRepository repo) =>
{
    var products = await repo.GetAllAsync();
    return Results.Ok(products);
});

// Endpoint que requiere autenticación
app.MapGet("/api/products/{id}", async (Guid id, IProductRepository repo) =>
{
    var product = await repo.GetByIdAsync(id);
    return product is not null ? Results.Ok(product) : Results.NotFound();
})
.RequireAuthorization(); // Requiere estar autenticado

// Endpoint con roles específicos
app.MapPost("/api/products", async (CreateProductRequest request, IProductRepository repo) =>
{
    var product = new Product { /* ... */ };
    await repo.AddAsync(product);
    return Results.Created($"/api/products/{product.Id}", product);
})
.RequireAuthorization(policy => policy.RequireRole("Admin", "Manager"));

// Endpoint con política personalizada
app.MapDelete("/api/products/{id}", async (Guid id, IProductRepository repo) =>
{
    await repo.DeleteAsync(id);
    return Results.NoContent();
})
.RequireAuthorization("CanDeleteProducts");

// Acceder a ClaimsPrincipal en Minimal API
app.MapGet("/api/me", (ClaimsPrincipal user) =>
{
    var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var email = user.FindFirst(ClaimTypes.Email)?.Value;
    var roles = user.FindAll(ClaimTypes.Role).Select(c => c.Value);

    return Results.Ok(new 
    { 
        UserId = userId, 
        Email = email, 
        Roles = roles 
    });
})
.RequireAuthorization();
```

## Authorization Policies (Políticas)

### Configuración de Políticas

```csharp
builder.Services.AddAuthorization(options =>
{
    // Política basada en rol
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    // Política basada en múltiples roles
    options.AddPolicy("ManagerOrAdmin", policy =>
        policy.RequireRole("Admin", "Manager"));

    // Política basada en claim específico
    options.AddPolicy("MustBeVerified", policy =>
        policy.RequireClaim("EmailVerified", "true"));

    // Política con múltiples requisitos
    options.AddPolicy("CanDeleteProducts", policy =>
    {
        policy.RequireRole("Admin");
        policy.RequireClaim("Department", "Sales", "Management");
        policy.Requirements.Add(new MinimumAgeRequirement(18));
    });

    // Política personalizada
    options.AddPolicy("CanAccessReports", policy =>
        policy.Requirements.Add(new ReportAccessRequirement()));
});
```

### Authorization Requirement Personalizado

```csharp
using Microsoft.AspNetCore.Authorization;

// Requirement
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

// Handler
public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirthClaim = context.User.FindFirst(c => c.Type == "DateOfBirth");

        if (dateOfBirthClaim is null)
        {
            return Task.CompletedTask;
        }

        if (DateTime.TryParse(dateOfBirthClaim.Value, out var dateOfBirth))
        {
            var age = DateTime.Today.Year - dateOfBirth.Year;
            if (dateOfBirth.Date > DateTime.Today.AddYears(-age))
            {
                age--;
            }

            if (age >= requirement.MinimumAge)
            {
                context.Succeed(requirement);
            }
        }

        return Task.CompletedTask;
    }
}

// Registrar el handler
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### Resource-Based Authorization

```csharp
// Requirement basado en recurso
public class DocumentAuthorizationHandler 
    : AuthorizationHandler<SameAuthorRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        SameAuthorRequirement requirement,
        Document resource)
    {
        var userIdClaim = context.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;

        if (userIdClaim is not null && 
            Guid.TryParse(userIdClaim, out var userId) &&
            resource.AuthorId == userId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

public class SameAuthorRequirement : IAuthorizationRequirement { }

// Uso en controller
[HttpPut("documents/{id}")]
public async Task<IActionResult> UpdateDocument(
    Guid id,
    [FromBody] UpdateDocumentRequest request,
    [FromServices] IAuthorizationService authorizationService)
{
    var document = await _documentRepository.GetByIdAsync(id);
    
    if (document is null)
    {
        return NotFound();
    }

    // Verificar autorización basada en el recurso
    var authResult = await authorizationService.AuthorizeAsync(
        User, 
        document, 
        new SameAuthorRequirement());

    if (!authResult.Succeeded)
    {
        return Forbid();
    }

    // Proceder con actualización...
    return NoContent();
}
```

## Claims Personalizados

### Agregar Claims al Token

```csharp
public string GenerateAccessToken(
    Guid userId, 
    string email, 
    IEnumerable<string> roles,
    Dictionary<string, string>? customClaims = null)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
        new Claim(ClaimTypes.Email, email),
        new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
    };

    // Agregar roles
    foreach (var role in roles)
    {
        claims.Add(new Claim(ClaimTypes.Role, role));
    }

    // Agregar claims personalizados
    if (customClaims is not null)
    {
        foreach (var (key, value) in customClaims)
        {
            claims.Add(new Claim(key, value));
        }
    }

    // ... resto del código de generación de token
}

// Uso
var customClaims = new Dictionary<string, string>
{
    { "Department", user.Department },
    { "EmployeeId", user.EmployeeId },
    { "EmailVerified", user.EmailVerified.ToString() }
};

var token = _tokenService.GenerateAccessToken(
    user.Id, 
    user.Email, 
    roles, 
    customClaims);
```

### Leer Claims en Endpoints

```csharp
[HttpGet("profile")]
[Authorize]
public IActionResult GetProfile()
{
    // Claims estándar
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var email = User.FindFirst(ClaimTypes.Email)?.Value;
    var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();

    // Claims personalizados
    var department = User.FindFirst("Department")?.Value;
    var employeeId = User.FindFirst("EmployeeId")?.Value;
    var isVerified = User.FindFirst("EmailVerified")?.Value == "true";

    return Ok(new
    {
        UserId = userId,
        Email = email,
        Roles = roles,
        Department = department,
        EmployeeId = employeeId,
        IsEmailVerified = isVerified
    });
}
```

## ASP.NET Core Identity (Opcional)

Si necesitas gestión completa de usuarios con Entity Framework:

**NuGet Package:**
```
Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

**Entidad de usuario:**
```csharp
using Microsoft.AspNetCore.Identity;

public class ApplicationUser : IdentityUser<Guid>
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
    public DateTime? LastLoginAt { get; set; }
}
```

**DbContext con Identity:**
```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;

public class ApplicationDbContext : IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder); // Importante para Identity

        // Configuraciones personalizadas...
    }
}
```

**Configuración en Program.cs:**
```csharp
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

builder.Services.AddIdentity<ApplicationUser, IdentityRole<Guid>>(options =>
{
    // Password settings
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;

    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;

    // User settings
    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();
```

## Mejores Prácticas

1. **Usar HTTPS en producción** - JWT debe transmitirse de forma segura
2. **Nunca exponer el Secret** en código o repositorios
3. **Usar Refresh Tokens** para sesiones largas
4. **Implementar token expiration apropiado** (15-60 minutos para access token)
5. **Revocar refresh tokens en logout**
6. **Validar tokens en cada request** (automático con middleware)
7. **Usar claims para información que no cambia frecuentemente**
8. **Implementar rate limiting** en endpoints de autenticación
9. **Loggear intentos fallidos** de autenticación
10. **Usar políticas de autorización** para reglas de negocio complejas

---

## Skills Relacionados

- **Para validar requests de login** → `validation.skill.md`
- **Para guardar refresh tokens en BD** → `ef-core.skill.md`
- **Para endpoints de auth** → `aspnet-webapi.skill.md`
- **Para testing de autenticación** → `testing.skill.md`
- **Para configurar servicios** → `di-configuration.skill.md`

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

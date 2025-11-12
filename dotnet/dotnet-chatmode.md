# Guía General de .NET - Chat Mode

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET & C#  
**Tipo:** Chat Mode / Guía General  
**Autor:** SWO Team

## Propósito

Esta guía proporciona conocimiento general sobre el ecosistema .NET, características modernas de C#, y arquitectura de aplicaciones. **Para implementaciones específicas, referencia los skills correspondientes.**

## Características Modernas de C# (.NET 6+)

### File-Scoped Namespaces

```csharp
namespace MyApp.Services; // Una sola línea - aplica a todo el archivo

public class UserService
{
    // Implementación
}
```

### Record Types

```csharp
// Record inmutable para DTOs
public record UserDto(Guid Id, string Email, string Name);

// Record con miembros adicionales
public record ProductDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    
    // Computed property
    public string DisplayPrice => $"${Price:F2}";
}

// Record con herencia
public record Person(string FirstName, string LastName);
public record Employee(string FirstName, string LastName, string Department) 
    : Person(FirstName, LastName);
```

### Init-Only Properties

```csharp
public class Configuration
{
    // Solo se puede asignar en inicialización
    public string ConnectionString { get; init; } = string.Empty;
    public int MaxConnections { get; init; } = 100;
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
}

// Uso
var config = new Configuration 
{ 
    ConnectionString = "Server=...",
    MaxConnections = 50
};
```

### Pattern Matching

```csharp
// Switch expressions
public string GetDiscount(Customer customer) => customer switch
{
    { IsPremium: true, YearsActive: > 5 } => "20% descuento",
    { IsPremium: true } => "15% descuento",
    { YearsActive: > 3 } => "10% descuento",
    _ => "Sin descuento"
};

// Property pattern
public decimal CalculateShipping(Order order) => order switch
{
    { TotalAmount: > 100 } => 0m,
    { Items.Count: > 10 } => 5m,
    { Destination: "Local" } => 3m,
    _ => 10m
};

// Type pattern
public string DescribeShape(object shape) => shape switch
{
    Circle { Radius: var r } => $"Circle with radius {r}",
    Rectangle { Width: var w, Height: var h } => $"Rectangle {w}x{h}",
    _ => "Unknown shape"
};
```

### Nullable Reference Types

```csharp
#nullable enable

public class User
{
    public string Email { get; set; }  // Non-nullable - requerido
    public string? MiddleName { get; set; }  // Nullable - opcional
    public Address? BillingAddress { get; set; }  // Nullable
}

// Null-forgiving operator cuando sabes que no es null
public void ProcessUser(User? user)
{
    var email = user!.Email; // ! indica que confías que no es null
}
```

### Global Usings

```csharp
// GlobalUsings.cs - aplica a todo el proyecto
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;
global using Microsoft.Extensions.Logging;
global using MyApp.Domain.Entities;
```

### Primary Constructors (C# 12+)

```csharp
// Constructor primario en clase
public class UserService(IUserRepository repository, ILogger<UserService> logger)
{
    // Los parámetros están disponibles en toda la clase
    public async Task<User?> GetUserAsync(Guid id)
    {
        logger.LogInformation("Getting user {Id}", id);
        return await repository.GetByIdAsync(id);
    }
}
```

## Estructura de Proyecto Recomendada

```
MyApp/
├── src/
│   ├── MyApp.Api/                    # Web API / Presentation Layer
│   │   ├── Controllers/
│   │   ├── Middleware/
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   └── appsettings.Development.json
│   │
│   ├── MyApp.Application/            # Application/Business Logic Layer
│   │   ├── Services/
│   │   │   ├── UserService.cs
│   │   │   └── ProductService.cs
│   │   ├── Interfaces/
│   │   │   ├── IUserService.cs
│   │   │   └── IProductService.cs
│   │   ├── DTOs/
│   │   │   ├── UserDto.cs
│   │   │   └── ProductDto.cs
│   │   └── Validators/
│   │       └── CreateUserValidator.cs
│   │
│   ├── MyApp.Domain/                 # Domain Layer
│   │   ├── Entities/
│   │   │   ├── User.cs
│   │   │   ├── Product.cs
│   │   │   └── Order.cs
│   │   ├── Enums/
│   │   │   └── OrderStatus.cs
│   │   ├── ValueObjects/
│   │   │   └── Email.cs
│   │   └── Exceptions/
│   │       └── DomainException.cs
│   │
│   └── MyApp.Infrastructure/         # Infrastructure Layer
│       ├── Data/
│       │   ├── ApplicationDbContext.cs
│       │   └── Configurations/
│       │       ├── UserConfiguration.cs
│       │       └── ProductConfiguration.cs
│       ├── Repositories/
│       │   ├── Repository.cs
│       │   └── ProductRepository.cs
│       └── Services/
│           └── EmailService.cs
│
└── tests/
    ├── MyApp.UnitTests/
    │   ├── Services/
    │   └── Validators/
    ├── MyApp.IntegrationTests/
    │   └── Api/
    └── MyApp.FunctionalTests/
```

## Principios SOLID

### Single Responsibility (Responsabilidad Única)

```csharp
// ❌ Mal: Múltiples responsabilidades
public class UserService
{
    public void CreateUser() { }
    public void SendEmail() { }
    public void LogToDatabase() { }
}

// ✅ Bien: Una responsabilidad por clase
public class UserService { public void CreateUser() { } }
public class EmailService { public void SendEmail() { } }
public class LoggingService { public void LogToDatabase() { } }
```

### Open/Closed (Abierto/Cerrado)

```csharp
// ✅ Abierto para extensión, cerrado para modificación
public interface INotificationService
{
    Task SendAsync(string message);
}

public class EmailNotificationService : INotificationService
{
    public async Task SendAsync(string message) { /* Email logic */ }
}

public class SmsNotificationService : INotificationService
{
    public async Task SendAsync(string message) { /* SMS logic */ }
}
```

### Liskov Substitution (Sustitución de Liskov)

```csharp
// ✅ Las clases derivadas deben ser sustituibles por su clase base
public abstract class Shape
{
    public abstract double CalculateArea();
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override double CalculateArea() => Math.PI * Radius * Radius;
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public override double CalculateArea() => Width * Height;
}
```

### Interface Segregation (Segregación de Interfaces)

```csharp
// ❌ Mal: Interface muy grande
public interface IUserOperations
{
    void Create();
    void Update();
    void Delete();
    void SendEmail();
    void GenerateReport();
}

// ✅ Bien: Interfaces pequeñas y específicas
public interface IUserCrudOperations
{
    void Create();
    void Update();
    void Delete();
}

public interface IUserNotificationOperations
{
    void SendEmail();
}

public interface IUserReportOperations
{
    void GenerateReport();
}
```

### Dependency Inversion (Inversión de Dependencias)

```csharp
// ✅ Depender de abstracciones, no de implementaciones concretas
public interface IUserRepository
{
    Task<User?> GetByIdAsync(Guid id);
}

public class UserService
{
    private readonly IUserRepository _repository; // Interfaz, no clase concreta
    
    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

## Dependency Injection

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Registrar servicios con diferentes lifetimes

// Transient: Nueva instancia cada vez que se solicita
builder.Services.AddTransient<IEmailService, EmailService>();

// Scoped: Una instancia por request HTTP
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();

// Singleton: Una única instancia para toda la aplicación
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);

var app = builder.Build();
```

**Para configuración detallada de DI** → Ver `di-configuration.skill.md`

## Async/Await Patterns

```csharp
// ✅ Usar async/await para operaciones I/O
public async Task<User?> GetUserAsync(Guid id)
{
    return await _repository.GetByIdAsync(id);
}

// ✅ ConfigureAwait(false) en libraries
public async Task<User?> GetUserLibraryAsync(Guid id)
{
    var user = await _repository.GetByIdAsync(id).ConfigureAwait(false);
    return user;
}

// ✅ ValueTask para hot paths (cuando result puede ser sync o async)
public async ValueTask<bool> ExistsAsync(Guid id)
{
    return await _cache.ContainsKeyAsync(id);
}

// ✅ Operaciones paralelas con Task.WhenAll
public async Task<(User user, List<Order> orders)> GetUserWithOrdersAsync(Guid userId)
{
    var userTask = _userRepository.GetByIdAsync(userId);
    var ordersTask = _orderRepository.GetByUserIdAsync(userId);
    
    await Task.WhenAll(userTask, ordersTask);
    
    return (await userTask, await ordersTask);
}

// ✅ Siempre usar CancellationToken
public async Task<List<Product>> GetProductsAsync(CancellationToken cancellationToken)
{
    return await _context.Products.ToListAsync(cancellationToken);
}
```

## Logging

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }

    public async Task<User?> GetUserAsync(Guid id)
    {
        _logger.LogInformation("Fetching user {UserId}", id);
        
        try
        {
            var user = await _repository.GetByIdAsync(id);
            
            if (user is null)
            {
                _logger.LogWarning("User {UserId} not found", id);
                return null;
            }

            _logger.LogDebug("User {UserId} retrieved successfully", id);
            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error fetching user {UserId}", id);
            throw;
        }
    }
}
```

## Configuration Management

```csharp
// appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;"
  },
  "JwtSettings": {
    "Secret": "your-secret-key",
    "ExpirationMinutes": 60,
    "Issuer": "MyApp",
    "Audience": "MyApp-Users"
  },
  "EmailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "From": "noreply@myapp.com"
  }
}

// Clase para configuración tipada
public class JwtSettings
{
    public string Secret { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; }
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
}

// Registrar en Program.cs
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("JwtSettings"));

// Usar en servicio
public class AuthService
{
    private readonly JwtSettings _jwtSettings;
    
    public AuthService(IOptions<JwtSettings> jwtSettings)
    {
        _jwtSettings = jwtSettings.Value;
    }
}
```

## Escenarios Comunes y Skills a Usar

### Crear una REST API Completa

**Necesitas:**
1. Controllers y endpoints → `aspnet-webapi.skill.md`
2. Acceso a base de datos → `ef-core.skill.md`
3. Validación de entrada → `validation.skill.md`
4. Configuración de servicios → Este archivo (DI básico)

**Ejemplo básico de servicio:**

```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository repository,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<ProductDto?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Getting product {ProductId}", id);
        
        var product = await _repository.GetByIdAsync(id, cancellationToken);
        
        if (product is null)
        {
            _logger.LogWarning("Product {ProductId} not found", id);
            return null;
        }

        // Mapear entidad a DTO
        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Stock = product.Stock
        };
    }

    public async Task<ProductDto> CreateAsync(
        CreateProductRequest request,
        CancellationToken cancellationToken = default)
    {
        var product = new Product
        {
            Id = Guid.NewGuid(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            Stock = request.Stock,
            CreatedAt = DateTime.UtcNow
        };

        await _repository.AddAsync(product, cancellationToken);
        
        _logger.LogInformation("Product {ProductId} created", product.Id);

        return new ProductDto
        {
            Id = product.Id,
            Name = product.Name,
            Price = product.Price,
            Stock = product.Stock
        };
    }
}
```

**Para implementación completa** → Combinar skills de `aspnet-webapi`, `ef-core`, y `validation`

### Agregar Autenticación

**Necesitas:** `authentication.skill.md`

Este skill cubre JWT, Identity, roles, y protección de endpoints.

### Validar Datos de Entrada

**Necesitas:** `validation.skill.md`

Este skill cubre FluentValidation, Data Annotations, y manejo de errores.

### Escribir Tests

**Necesitas:** `testing.skill.md`

Este skill cubre unit tests, integration tests, y mocking.

### Optimizar Performance

**Necesitas:** `performance.skill.md`

Este skill cubre caching, async patterns, y optimizaciones.

## Guía de Decisión Rápida

| Quieres... | Usa este skill |
|---|---|
| Crear endpoints REST | `aspnet-webapi.skill.md` |
| Trabajar con base de datos | `ef-core.skill.md` |
| Agregar autenticación | `authentication.skill.md` |
| Validar entrada de datos | `validation.skill.md` |
| Escribir tests | `testing.skill.md` |
| Configurar DI y settings | `di-configuration.skill.md` |
| Optimizar performance | `performance.skill.md` |
| Arquitectura y patrones | Este archivo |

## Mejores Prácticas Generales

1. **Usar características modernas de C#** (records, pattern matching, file-scoped namespaces)
2. **Habilitar nullable reference types** (`#nullable enable`)
3. **Seguir patrones async/await** para operaciones I/O
4. **Usar dependency injection** para loose coupling
5. **Implementar logging apropiado** con ILogger
6. **Manejar excepciones correctamente**
7. **Usar cancellation tokens** en métodos async
8. **Escribir código auto-documentado** y agregar XML docs cuando sea necesario
9. **Seguir principios SOLID**
10. **Referenciar skills específicos** para implementaciones detalladas

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

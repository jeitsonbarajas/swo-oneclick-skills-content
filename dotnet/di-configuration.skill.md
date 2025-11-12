# Skill: Dependency Injection & Configuration en .NET

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET 6+ / ASP.NET Core  
**Tipo:** Skill Específico  
**Autor:** SWO Team

## Propósito

Este skill se enfoca exclusivamente en **Dependency Injection (DI)** y **Configuration Management** en aplicaciones .NET. Cubre registro de servicios, lifetimes, options pattern, configuración tipada, y mejores prácticas para IoC.

## Lo que CUBRE este Skill

- ✅ Dependency Injection container built-in
- ✅ Service lifetimes (Transient, Scoped, Singleton)
- ✅ Options Pattern para configuración
- ✅ Configuración tipada desde appsettings.json
- ✅ Environment-specific configuration
- ✅ Secrets management
- ✅ Service registration patterns
- ✅ Factory patterns y DI avanzado

## Lo que NO CUBRE este Skill

- ❌ Implementación de servicios → Ver `aspnet-webapi.skill.md`
- ❌ Implementación de repositories → Ver `ef-core.skill.md`
- ❌ Implementación de autenticación → Ver `authentication.skill.md`
- ❌ Testing de servicios → Ver `testing.skill.md`

## Service Lifetimes

### Transient

**Nueva instancia cada vez que se solicita.**

```csharp
// Registro
builder.Services.AddTransient<IEmailService, EmailService>();
builder.Services.AddTransient<INotificationService, NotificationService>();

// Uso: Cada vez que se inyecta IEmailService, se crea una nueva instancia
public class UserController : ControllerBase
{
    private readonly IEmailService _emailService1;
    private readonly IEmailService _emailService2;

    // _emailService1 y _emailService2 son instancias DIFERENTES
    public UserController(IEmailService emailService1, IEmailService emailService2)
    {
        _emailService1 = emailService1;
        _emailService2 = emailService2;
    }
}
```

**Cuándo usar:**
- Servicios ligeros sin estado
- Servicios que no deben compartirse
- Utilidades y helpers

### Scoped

**Una instancia por request HTTP (o scope).**

```csharp
// Registro
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<ApplicationDbContext>();

// Durante un request HTTP, todas las inyecciones reciben la MISMA instancia
// En el siguiente request, se crea una nueva instancia
```

**Cuándo usar:**
- Servicios que trabajan con DbContext (Entity Framework)
- Servicios que mantienen estado durante un request
- La mayoría de servicios de aplicación

### Singleton

**Una única instancia para toda la vida de la aplicación.**

```csharp
// Registro
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddSingleton<IConfiguration>(builder.Configuration);
builder.Services.AddSingleton<AppSettings>();

// Toda la aplicación usa la MISMA instancia
```

**Cuándo usar:**
- Servicios sin estado que son costosos de crear
- Caches y configuraciones
- Servicios thread-safe que se pueden compartir

### Tabla Comparativa

| Lifetime | Creación | Destrucción | Uso Típico |
|----------|----------|-------------|------------|
| **Transient** | Cada solicitud | Inmediato (GC) | Utilities, helpers |
| **Scoped** | Por HTTP request | Fin del request | Services, Repositories, DbContext |
| **Singleton** | Una vez | Fin de la app | Caches, Configuration, Logging |

## Registro de Servicios

### Básico

```csharp
var builder = WebApplication.CreateBuilder(args);

// Interface → Implementation
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IProductService, ProductService>();

// Clase concreta (sin interface)
builder.Services.AddScoped<EmailSender>();

// Múltiples implementaciones
builder.Services.AddScoped<INotificationService, EmailNotificationService>();
builder.Services.AddScoped<INotificationService, SmsNotificationService>();

var app = builder.Build();
```

### Con Factory Method

```csharp
// Factory simple
builder.Services.AddSingleton<ILogger>(serviceProvider =>
{
    var config = serviceProvider.GetRequiredService<IConfiguration>();
    return new CustomLogger(config["LogPath"]);
});

// Factory con lógica condicional
builder.Services.AddScoped<IPaymentService>(serviceProvider =>
{
    var config = serviceProvider.GetRequiredService<IConfiguration>();
    var paymentProvider = config["PaymentProvider"];

    return paymentProvider switch
    {
        "Stripe" => new StripePaymentService(),
        "PayPal" => new PayPalPaymentService(),
        _ => throw new InvalidOperationException($"Unknown payment provider: {paymentProvider}")
    };
});

// Factory con dependencias
builder.Services.AddScoped<IOrderService>(serviceProvider =>
{
    var repository = serviceProvider.GetRequiredService<IOrderRepository>();
    var logger = serviceProvider.GetRequiredService<ILogger<OrderService>>();
    var config = serviceProvider.GetRequiredService<IOptions<OrderSettings>>();

    return new OrderService(repository, logger, config);
});
```

### TryAdd Methods

```csharp
// Solo agrega si no existe una registration previa
builder.Services.TryAddScoped<IUserService, UserService>();

// Si ya existe IUserService registrado, NO se sobrescribe
builder.Services.TryAddScoped<IUserService, AnotherUserService>(); // Ignorado

// TryAddEnumerable: Agrega solo si la combinación service/implementation no existe
builder.Services.TryAddEnumerable(
    ServiceDescriptor.Scoped<INotificationService, EmailNotificationService>());
```

### Registrar Múltiples Implementaciones

```csharp
// Registrar múltiples implementaciones
builder.Services.AddScoped<INotificationService, EmailNotificationService>();
builder.Services.AddScoped<INotificationService, SmsNotificationService>();
builder.Services.AddScoped<INotificationService, PushNotificationService>();

// Consumir todas las implementaciones
public class NotificationCoordinator
{
    private readonly IEnumerable<INotificationService> _notificationServices;

    public NotificationCoordinator(IEnumerable<INotificationService> notificationServices)
    {
        _notificationServices = notificationServices;
    }

    public async Task SendToAllChannelsAsync(string message)
    {
        foreach (var service in _notificationServices)
        {
            await service.SendAsync(message);
        }
    }
}
```

### Registro por Convención

```csharp
// Extension method para registrar automáticamente
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        var assembly = typeof(ServiceCollectionExtensions).Assembly;

        // Registrar todos los servicios que implementan IService
        var serviceTypes = assembly.GetTypes()
            .Where(t => t.IsClass && !t.IsAbstract && t.Name.EndsWith("Service"))
            .ToList();

        foreach (var serviceType in serviceTypes)
        {
            var interfaceType = serviceType.GetInterface($"I{serviceType.Name}");
            if (interfaceType != null)
            {
                services.AddScoped(interfaceType, serviceType);
            }
        }

        return services;
    }
}

// Uso
builder.Services.AddApplicationServices();
```

## Options Pattern

### Configuración Básica

**appsettings.json:**
```json
{
  "EmailSettings": {
    "SmtpHost": "smtp.gmail.com",
    "SmtpPort": 587,
    "Username": "noreply@myapp.com",
    "Password": "secret",
    "FromEmail": "noreply@myapp.com",
    "EnableSsl": true
  },
  "JwtSettings": {
    "Secret": "your-256-bit-secret",
    "Issuer": "MyApp",
    "Audience": "MyApp-Users",
    "ExpirationMinutes": 60
  },
  "CacheSettings": {
    "DefaultExpirationMinutes": 30,
    "SlidingExpirationMinutes": 10,
    "AbsoluteExpirationHours": 24
  }
}
```

**Clases de configuración:**
```csharp
public class EmailSettings
{
    public const string SectionName = "EmailSettings";

    public string SmtpHost { get; set; } = string.Empty;
    public int SmtpPort { get; set; }
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string FromEmail { get; set; } = string.Empty;
    public bool EnableSsl { get; set; }
}

public class JwtSettings
{
    public const string SectionName = "JwtSettings";

    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpirationMinutes { get; set; }
}

public class CacheSettings
{
    public const string SectionName = "CacheSettings";

    public int DefaultExpirationMinutes { get; set; }
    public int SlidingExpirationMinutes { get; set; }
    public int AbsoluteExpirationHours { get; set; }
}
```

**Registro en Program.cs:**
```csharp
// Opción 1: Configure
builder.Services.Configure<EmailSettings>(
    builder.Configuration.GetSection(EmailSettings.SectionName));

builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection(JwtSettings.SectionName));

// Opción 2: Bind (más explícito)
var emailSettings = new EmailSettings();
builder.Configuration.GetSection(EmailSettings.SectionName).Bind(emailSettings);
builder.Services.AddSingleton(emailSettings);
```

### Consumir Options

**Con IOptions<T>:**
```csharp
using Microsoft.Extensions.Options;

public class EmailService : IEmailService
{
    private readonly EmailSettings _settings;
    private readonly ILogger<EmailService> _logger;

    // IOptions<T> - Se resuelve una vez al crear el servicio
    public EmailService(
        IOptions<EmailSettings> options,
        ILogger<EmailService> logger)
    {
        _settings = options.Value;
        _logger = logger;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        using var smtpClient = new SmtpClient(_settings.SmtpHost, _settings.SmtpPort)
        {
            Credentials = new NetworkCredential(_settings.Username, _settings.Password),
            EnableSsl = _settings.EnableSsl
        };

        var message = new MailMessage(_settings.FromEmail, to, subject, body);
        await smtpClient.SendMailAsync(message);

        _logger.LogInformation("Email sent to {To}", to);
    }
}
```

**Con IOptionsSnapshot<T> (Scoped - recarga en cada request):**
```csharp
public class CacheService : ICacheService
{
    private readonly CacheSettings _settings;

    // IOptionsSnapshot<T> - Se recarga en cada request (Scoped)
    // Útil cuando la configuración puede cambiar
    public CacheService(IOptionsSnapshot<CacheSettings> options)
    {
        _settings = options.Value;
    }

    public void Set<T>(string key, T value)
    {
        var expiration = TimeSpan.FromMinutes(_settings.DefaultExpirationMinutes);
        // ... usar expiration
    }
}
```

**Con IOptionsMonitor<T> (Singleton - detecta cambios en runtime):**
```csharp
public class FeatureFlagService : IFeatureFlagService
{
    private readonly IOptionsMonitor<FeatureSettings> _optionsMonitor;

    // IOptionsMonitor<T> - Detecta cambios en appsettings en runtime
    // Puede usarse en Singleton services
    public FeatureFlagService(IOptionsMonitor<FeatureSettings> optionsMonitor)
    {
        _optionsMonitor = optionsMonitor;
    }

    public bool IsEnabled(string featureName)
    {
        // CurrentValue siempre tiene la configuración más reciente
        var settings = _optionsMonitor.CurrentValue;
        return settings.EnabledFeatures.Contains(featureName);
    }
}
```

### Validación de Options

```csharp
public class EmailSettings
{
    public string SmtpHost { get; set; } = string.Empty;
    public int SmtpPort { get; set; }
    public string FromEmail { get; set; } = string.Empty;
}

// Validación con Data Annotations
public class EmailSettingsWithValidation
{
    [Required]
    [MinLength(5)]
    public string SmtpHost { get; set; } = string.Empty;

    [Range(1, 65535)]
    public int SmtpPort { get; set; }

    [Required]
    [EmailAddress]
    public string FromEmail { get; set; } = string.Empty;
}

// Registro con validación
builder.Services.AddOptions<EmailSettingsWithValidation>()
    .Bind(builder.Configuration.GetSection("EmailSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart(); // Valida al inicio de la aplicación

// Validación personalizada
builder.Services.AddOptions<EmailSettings>()
    .Bind(builder.Configuration.GetSection("EmailSettings"))
    .Validate(settings =>
    {
        if (settings.SmtpPort <= 0 || settings.SmtpPort > 65535)
            return false;

        if (string.IsNullOrWhiteSpace(settings.SmtpHost))
            return false;

        return true;
    }, "Invalid email configuration")
    .ValidateOnStart();
```

## Configuración por Ambiente

### appsettings por Environment

```
appsettings.json                 // Base configuration
appsettings.Development.json     // Development overrides
appsettings.Staging.json         // Staging overrides
appsettings.Production.json      // Production overrides
```

**Precedencia (último gana):**
1. appsettings.json
2. appsettings.{Environment}.json
3. User Secrets (Development)
4. Environment Variables
5. Command-line arguments

**Ejemplo:**

**appsettings.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

**appsettings.Development.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApp_Dev;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Warning"
    }
  },
  "DetailedErrors": true
}
```

**appsettings.Production.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=prod-server;Database=MyApp_Prod;User=sa;Password=..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft": "Error"
    }
  },
  "DetailedErrors": false
}
```

### User Secrets (Development)

**Nunca commitear secrets en appsettings.json!**

```powershell
# Inicializar user secrets
dotnet user-secrets init

# Agregar secrets
dotnet user-secrets set "JwtSettings:Secret" "my-super-secret-key-12345"
dotnet user-secrets set "EmailSettings:Password" "email-password"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=...;Password=dev123"

# Listar secrets
dotnet user-secrets list

# Remover secret
dotnet user-secrets remove "JwtSettings:Secret"

# Limpiar todos
dotnet user-secrets clear
```

Los secrets se almacenan en:
- Windows: `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`
- Linux/Mac: `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`

### Environment Variables

```csharp
// Leer environment variable
var connectionString = builder.Configuration["ConnectionStrings:DefaultConnection"];
var jwtSecret = builder.Configuration["JwtSettings:Secret"];

// En producción, las variables de entorno sobrescriben appsettings
// Formato: SectionName__PropertyName (doble underscore)
```

**Configurar en launchSettings.json (Development):**
```json
{
  "profiles": {
    "MyApp": {
      "commandName": "Project",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "ConnectionStrings__DefaultConnection": "Server=localhost;...",
        "JwtSettings__Secret": "dev-secret-key"
      }
    }
  }
}
```

**En Docker/Kubernetes:**
```yaml
env:
  - name: ConnectionStrings__DefaultConnection
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: connection-string
  - name: JwtSettings__Secret
    valueFrom:
      secretKeyRef:
        name: jwt-secret
        key: secret-key
```

## Patterns Avanzados

### Service Locator Pattern (Anti-pattern - evitar)

```csharp
// ❌ Anti-pattern: No hagas esto
public class BadService
{
    private readonly IServiceProvider _serviceProvider;

    public BadService(IServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    public void DoWork()
    {
        // Service Locator - dificulta testing y oculta dependencias
        var repository = _serviceProvider.GetRequiredService<IProductRepository>();
        repository.GetAll();
    }
}

// ✅ Mejor: Inyectar dependencia directamente
public class GoodService
{
    private readonly IProductRepository _repository;

    public GoodService(IProductRepository repository)
    {
        _repository = repository;
    }

    public void DoWork()
    {
        _repository.GetAll();
    }
}
```

### Named Services

```csharp
// Registrar con key
public interface IStorageService { }
public class AzureStorageService : IStorageService { }
public class LocalStorageService : IStorageService { }

// Registrar con factory que usa Named Options
builder.Services.AddScoped<IStorageService>(serviceProvider =>
{
    var config = serviceProvider.GetRequiredService<IConfiguration>();
    var storageType = config["StorageType"];

    return storageType switch
    {
        "Azure" => new AzureStorageService(),
        "Local" => new LocalStorageService(),
        _ => throw new InvalidOperationException("Invalid storage type")
    };
});
```

### Decorador Pattern con DI

```csharp
public interface IProductService
{
    Task<Product> GetByIdAsync(Guid id);
}

public class ProductService : IProductService
{
    public async Task<Product> GetByIdAsync(Guid id)
    {
        // Implementación base
    }
}

// Decorador que agrega caching
public class CachedProductService : IProductService
{
    private readonly IProductService _inner;
    private readonly ICacheService _cache;

    public CachedProductService(IProductService inner, ICacheService cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Product> GetByIdAsync(Guid id)
    {
        var cacheKey = $"product_{id}";
        var cached = await _cache.GetAsync<Product>(cacheKey);
        
        if (cached != null)
            return cached;

        var product = await _inner.GetByIdAsync(id);
        await _cache.SetAsync(cacheKey, product, TimeSpan.FromMinutes(10));
        
        return product;
    }
}

// Registro
builder.Services.AddScoped<ProductService>();
builder.Services.AddScoped<IProductService>(serviceProvider =>
{
    var productService = serviceProvider.GetRequiredService<ProductService>();
    var cacheService = serviceProvider.GetRequiredService<ICacheService>();
    
    return new CachedProductService(productService, cacheService);
});
```

## Mejores Prácticas

1. **Preferir Constructor Injection** sobre Property Injection
2. **Usar interfaces** para abstracciones, no clases concretas
3. **Elegir el lifetime correcto** (Scoped para la mayoría de servicios)
4. **Evitar Service Locator pattern**
5. **Usar Options Pattern** para configuración tipada
6. **Nunca commitear secrets** - usar User Secrets o Azure Key Vault
7. **Validar options al startup** con ValidateOnStart()
8. **Organizar registrations** en extension methods por área
9. **No inyectar Singleton en Scoped/Transient**
10. **Usar IOptionsSnapshot** cuando la config puede cambiar

### Organización de Registrations

```csharp
// Extensions/ServiceCollectionExtensions.cs
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IProductService, ProductService>();
        services.AddScoped<IOrderService, OrderService>();
        return services;
    }

    public static IServiceCollection AddInfrastructureServices(this IServiceCollection services)
    {
        services.AddScoped<IUserRepository, UserRepository>();
        services.AddScoped<IProductRepository, ProductRepository>();
        services.AddScoped<IOrderRepository, OrderRepository>();
        return services;
    }

    public static IServiceCollection AddExternalServices(this IServiceCollection services)
    {
        services.AddScoped<IEmailService, EmailService>();
        services.AddScoped<ISmsService, TwilioSmsService>();
        services.AddSingleton<ICacheService, RedisCacheService>();
        return services;
    }
}

// Program.cs - limpio y organizado
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices();
builder.Services.AddExternalServices();

var app = builder.Build();
```

---

## Skills Relacionados

- **Para servicios a registrar** → `aspnet-webapi.skill.md`, `ef-core.skill.md`
- **Para configurar autenticación** → `authentication.skill.md`
- **Para testing con DI** → `testing.skill.md`
- **Para validar configuración** → `validation.skill.md`

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

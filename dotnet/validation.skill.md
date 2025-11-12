# Skill: Validation & Error Handling en .NET

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET 6+ / ASP.NET Core  
**Tipo:** Skill Específico  
**Autor:** SWO Team

## Propósito

Este skill se enfoca exclusivamente en **validación de datos** y **manejo de errores** en aplicaciones .NET. Cubre FluentValidation, Data Annotations, validación personalizada, manejo global de excepciones, y respuestas de error consistentes.

## Lo que CUBRE este Skill

- ✅ FluentValidation para validación robusta
- ✅ Data Annotations para validación simple
- ✅ Validadores personalizados
- ✅ Manejo global de excepciones
- ✅ Middleware de error handling
- ✅ Respuestas de error consistentes (Problem Details)
- ✅ Validación de modelos en controllers y Minimal APIs
- ✅ Mensajes de error localizados

## Lo que NO CUBRE este Skill

- ❌ Autenticación y autorización → Ver `authentication.skill.md`
- ❌ Estructura de controllers → Ver `aspnet-webapi.skill.md`
- ❌ Configuración de Entity Framework → Ver `ef-core.skill.md`
- ❌ Testing de validadores → Ver `testing.skill.md`
- ❌ Dependency injection → Ver `di-configuration.skill.md`

## FluentValidation

### Instalación

**NuGet Packages:**
```
FluentValidation
FluentValidation.DependencyInjectionExtensions
FluentValidation.AspNetCore (opcional para integración automática)
```

### Configuración en Program.cs

```csharp
using FluentValidation;
using FluentValidation.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Opción 1: Registro automático de validators desde assembly
builder.Services.AddValidatorsFromAssemblyContaining<Program>();

// Opción 2: Integración automática con Model Binding (opcional)
builder.Services.AddFluentValidationAutoValidation()
                .AddFluentValidationClientsideAdapters();

// Configurar respuestas de validación personalizadas
builder.Services.Configure<ApiBehaviorOptions>(options =>
{
    options.InvalidModelStateResponseFactory = context =>
    {
        var errors = context.ModelState
            .Where(e => e.Value?.Errors.Count > 0)
            .Select(e => new
            {
                Field = e.Key,
                Errors = e.Value!.Errors.Select(x => x.ErrorMessage).ToArray()
            })
            .ToList();

        return new BadRequestObjectResult(new
        {
            Type = "ValidationError",
            Title = "One or more validation errors occurred",
            Status = 400,
            Errors = errors,
            TraceId = Activity.Current?.Id ?? context.HttpContext.TraceIdentifier
        });
    };
});

var app = builder.Build();
app.Run();
```

### Validator Básico

```csharp
using FluentValidation;

// Request DTO
public record CreateUserRequest
{
    public string Email { get; init; } = string.Empty;
    public string Password { get; init; } = string.Empty;
    public string FirstName { get; init; } = string.Empty;
    public string LastName { get; init; } = string.Empty;
    public DateTime? DateOfBirth { get; init; }
    public string? PhoneNumber { get; init; }
}

// Validator
public class CreateUserRequestValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserRequestValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format")
            .MaximumLength(256).WithMessage("Email must not exceed 256 characters");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches(@"[A-Z]").WithMessage("Password must contain at least one uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain at least one lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain at least one number")
            .Matches(@"[\W_]").WithMessage("Password must contain at least one special character");

        RuleFor(x => x.FirstName)
            .NotEmpty().WithMessage("First name is required")
            .MaximumLength(100).WithMessage("First name must not exceed 100 characters")
            .Matches(@"^[a-zA-Z\s'-]+$").WithMessage("First name contains invalid characters");

        RuleFor(x => x.LastName)
            .NotEmpty().WithMessage("Last name is required")
            .MaximumLength(100).WithMessage("Last name must not exceed 100 characters")
            .Matches(@"^[a-zA-Z\s'-]+$").WithMessage("Last name contains invalid characters");

        RuleFor(x => x.DateOfBirth)
            .NotNull().WithMessage("Date of birth is required")
            .Must(BeAValidAge).WithMessage("User must be at least 18 years old")
            .LessThan(DateTime.Today).WithMessage("Date of birth cannot be in the future");

        RuleFor(x => x.PhoneNumber)
            .Matches(@"^\+?[1-9]\d{1,14}$")
            .When(x => !string.IsNullOrEmpty(x.PhoneNumber))
            .WithMessage("Invalid phone number format");
    }

    private bool BeAValidAge(DateTime? dateOfBirth)
    {
        if (!dateOfBirth.HasValue) return false;

        var age = DateTime.Today.Year - dateOfBirth.Value.Year;
        if (dateOfBirth.Value.Date > DateTime.Today.AddYears(-age)) age--;

        return age >= 18;
    }
}
```

### Validator con Dependencias Inyectadas

```csharp
public class UpdateProductRequestValidator : AbstractValidator<UpdateProductRequest>
{
    private readonly IProductRepository _productRepository;

    public UpdateProductRequestValidator(IProductRepository productRepository)
    {
        _productRepository = productRepository;

        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Product name is required")
            .MaximumLength(200).WithMessage("Product name must not exceed 200 characters")
            .MustAsync(BeUniqueName).WithMessage("A product with this name already exists");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero")
            .LessThan(1000000).WithMessage("Price must be less than 1,000,000");

        RuleFor(x => x.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("Stock cannot be negative");

        RuleFor(x => x.CategoryId)
            .NotEmpty().WithMessage("Category is required")
            .MustAsync(CategoryExists).WithMessage("Category does not exist");
    }

    private async Task<bool> BeUniqueName(UpdateProductRequest request, string name, CancellationToken cancellationToken)
    {
        var existingProduct = await _productRepository.GetByNameAsync(name, cancellationToken);
        
        // Si existe un producto con ese nombre pero es el mismo que estamos actualizando, es válido
        return existingProduct is null || existingProduct.Id == request.Id;
    }

    private async Task<bool> CategoryExists(Guid categoryId, CancellationToken cancellationToken)
    {
        var category = await _productRepository.GetCategoryByIdAsync(categoryId, cancellationToken);
        return category is not null;
    }
}
```

### Uso en Controller

```csharp
using FluentValidation;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;
    private readonly IValidator<CreateUserRequest> _validator;
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        IUserService userService,
        IValidator<CreateUserRequest> validator,
        ILogger<UsersController> logger)
    {
        _userService = userService;
        _validator = validator;
        _logger = logger;
    }

    // Opción 1: Validación manual explícita
    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        // Validar manualmente
        var validationResult = await _validator.ValidateAsync(request, cancellationToken);

        if (!validationResult.IsValid)
        {
            // Agregar errores al ModelState
            foreach (var error in validationResult.Errors)
            {
                ModelState.AddModelError(error.PropertyName, error.ErrorMessage);
            }

            return ValidationProblem(ModelState);
        }

        var user = await _userService.CreateAsync(request, cancellationToken);
        
        _logger.LogInformation("User {UserId} created successfully", user.Id);

        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    // Opción 2: Si configuraste FluentValidationAutoValidation, la validación es automática
    [HttpPost("auto")]
    public async Task<ActionResult<UserDto>> CreateWithAutoValidation(
        [FromBody] CreateUserRequest request,
        CancellationToken cancellationToken)
    {
        // La validación ya ocurrió automáticamente
        // Si llegamos aquí, el request es válido

        var user = await _userService.CreateAsync(request, cancellationToken);
        return CreatedAtAction(nameof(GetById), new { id = user.Id }, user);
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> GetById(Guid id, CancellationToken cancellationToken)
    {
        var user = await _userService.GetByIdAsync(id, cancellationToken);
        return user is not null ? Ok(user) : NotFound();
    }
}
```

### Uso en Minimal APIs

```csharp
using FluentValidation;

app.MapPost("/api/users", async (
    CreateUserRequest request,
    IValidator<CreateUserRequest> validator,
    IUserService userService,
    CancellationToken cancellationToken) =>
{
    // Validar request
    var validationResult = await validator.ValidateAsync(request, cancellationToken);

    if (!validationResult.IsValid)
    {
        return Results.ValidationProblem(validationResult.ToDictionary());
    }

    var user = await userService.CreateAsync(request, cancellationToken);
    return Results.Created($"/api/users/{user.Id}", user);
});

// Extension method para convertir ValidationResult a Dictionary
public static class ValidationExtensions
{
    public static IDictionary<string, string[]> ToDictionary(this ValidationResult validationResult)
    {
        return validationResult.Errors
            .GroupBy(x => x.PropertyName)
            .ToDictionary(
                g => g.Key,
                g => g.Select(x => x.ErrorMessage).ToArray()
            );
    }
}
```

### Validación Condicional

```csharp
public class CreateOrderRequestValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderRequestValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("Customer ID is required");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must contain at least one item")
            .Must(x => x.Count <= 100).WithMessage("Order cannot contain more than 100 items");

        // Validar cada item
        RuleForEach(x => x.Items)
            .SetValidator(new OrderItemValidator());

        // Validación condicional: si el total > 1000, requerir approval
        RuleFor(x => x.ApprovedBy)
            .NotEmpty()
            .When(x => CalculateTotal(x.Items) > 1000)
            .WithMessage("Orders over $1000 require approval");

        // Validación condicional: shipping address requerida si no es pickup
        RuleFor(x => x.ShippingAddress)
            .NotNull()
            .When(x => x.DeliveryMethod != DeliveryMethod.Pickup)
            .WithMessage("Shipping address is required for delivery orders")
            .SetValidator(new AddressValidator()!)
            .When(x => x.ShippingAddress is not null);
    }

    private decimal CalculateTotal(List<OrderItem> items)
    {
        return items.Sum(i => i.Price * i.Quantity);
    }
}

public class OrderItemValidator : AbstractValidator<OrderItem>
{
    public OrderItemValidator()
    {
        RuleFor(x => x.ProductId)
            .NotEmpty().WithMessage("Product ID is required");

        RuleFor(x => x.Quantity)
            .GreaterThan(0).WithMessage("Quantity must be greater than zero")
            .LessThanOrEqualTo(999).WithMessage("Quantity cannot exceed 999");

        RuleFor(x => x.Price)
            .GreaterThan(0).WithMessage("Price must be greater than zero");
    }
}
```

## Data Annotations (Validación Simple)

Útil para validaciones básicas sin necesidad de FluentValidation:

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateProductRequest
{
    [Required(ErrorMessage = "Product name is required")]
    [StringLength(200, MinimumLength = 3, ErrorMessage = "Name must be between 3 and 200 characters")]
    public string Name { get; set; } = string.Empty;

    [Required]
    [StringLength(1000, ErrorMessage = "Description cannot exceed 1000 characters")]
    public string Description { get; set; } = string.Empty;

    [Required]
    [Range(0.01, 1000000, ErrorMessage = "Price must be between 0.01 and 1,000,000")]
    public decimal Price { get; set; }

    [Range(0, int.MaxValue, ErrorMessage = "Stock cannot be negative")]
    public int Stock { get; set; }

    [Required]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string ContactEmail { get; set; } = string.Empty;

    [Url(ErrorMessage = "Invalid URL format")]
    public string? ProductUrl { get; set; }

    [Phone(ErrorMessage = "Invalid phone number format")]
    public string? ContactPhone { get; set; }

    [RegularExpression(@"^[A-Z]{2}\d{4}$", ErrorMessage = "SKU must be 2 uppercase letters followed by 4 digits")]
    public string? Sku { get; set; }
}
```

### Custom Data Annotation

```csharp
using System.ComponentModel.DataAnnotations;

// Atributo personalizado
public class FutureDateAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(object? value, ValidationContext validationContext)
    {
        if (value is DateTime dateValue)
        {
            if (dateValue <= DateTime.UtcNow)
            {
                return new ValidationResult(ErrorMessage ?? "Date must be in the future");
            }
        }

        return ValidationResult.Success;
    }
}

// Uso
public class CreateEventRequest
{
    [Required]
    public string Title { get; set; } = string.Empty;

    [Required]
    [FutureDate(ErrorMessage = "Event date must be in the future")]
    public DateTime EventDate { get; set; }
}
```

## Manejo Global de Excepciones

### Middleware de Error Handling

```csharp
using System.Net;
using System.Text.Json;

public class GlobalExceptionHandlerMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<GlobalExceptionHandlerMiddleware> _logger;
    private readonly IHostEnvironment _environment;

    public GlobalExceptionHandlerMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlerMiddleware> logger,
        IHostEnvironment environment)
    {
        _next = next;
        _logger = logger;
        _environment = environment;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";

        var (statusCode, message, errorType) = exception switch
        {
            ValidationException => (HttpStatusCode.BadRequest, exception.Message, "ValidationError"),
            UnauthorizedAccessException => (HttpStatusCode.Unauthorized, "Unauthorized access", "UnauthorizedError"),
            KeyNotFoundException => (HttpStatusCode.NotFound, "Resource not found", "NotFoundError"),
            InvalidOperationException => (HttpStatusCode.BadRequest, exception.Message, "InvalidOperation"),
            ArgumentException => (HttpStatusCode.BadRequest, exception.Message, "InvalidArgument"),
            _ => (HttpStatusCode.InternalServerError, "An error occurred while processing your request", "InternalError")
        };

        context.Response.StatusCode = (int)statusCode;

        var response = new
        {
            Type = errorType,
            Title = message,
            Status = (int)statusCode,
            Detail = _environment.IsDevelopment() ? exception.ToString() : null,
            TraceId = context.TraceIdentifier,
            Timestamp = DateTime.UtcNow
        };

        var options = new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };
        var json = JsonSerializer.Serialize(response, options);

        await context.Response.WriteAsync(json);
    }
}

// Registrar en Program.cs
app.UseMiddleware<GlobalExceptionHandlerMiddleware>();
```

### Exception Filter (alternativa para Controllers)

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    private readonly IHostEnvironment _environment;

    public GlobalExceptionFilter(
        ILogger<GlobalExceptionFilter> logger,
        IHostEnvironment environment)
    {
        _logger = logger;
        _environment = environment;
    }

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "An exception occurred");

        var (statusCode, message, errorType) = context.Exception switch
        {
            ValidationException => (400, context.Exception.Message, "ValidationError"),
            UnauthorizedAccessException => (401, "Unauthorized access", "UnauthorizedError"),
            KeyNotFoundException => (404, "Resource not found", "NotFoundError"),
            _ => (500, "An error occurred", "InternalError")
        };

        var problemDetails = new ProblemDetails
        {
            Type = errorType,
            Title = message,
            Status = statusCode,
            Detail = _environment.IsDevelopment() ? context.Exception.ToString() : null,
            Instance = context.HttpContext.Request.Path
        };

        problemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
        problemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        context.Result = new ObjectResult(problemDetails)
        {
            StatusCode = statusCode
        };

        context.ExceptionHandled = true;
    }
}

// Registrar en Program.cs
builder.Services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();
});
```

## Excepciones Personalizadas

```csharp
// Base exception
public abstract class DomainException : Exception
{
    protected DomainException(string message) : base(message) { }
    
    protected DomainException(string message, Exception innerException) 
        : base(message, innerException) { }
}

// Excepciones específicas
public class EntityNotFoundException : DomainException
{
    public EntityNotFoundException(string entityName, Guid id)
        : base($"{entityName} with ID {id} was not found")
    {
        EntityName = entityName;
        EntityId = id;
    }

    public string EntityName { get; }
    public Guid EntityId { get; }
}

public class DuplicateEntityException : DomainException
{
    public DuplicateEntityException(string entityName, string propertyName, string value)
        : base($"{entityName} with {propertyName} '{value}' already exists")
    {
        EntityName = entityName;
        PropertyName = propertyName;
        Value = value;
    }

    public string EntityName { get; }
    public string PropertyName { get; }
    public string Value { get; }
}

public class BusinessRuleViolationException : DomainException
{
    public BusinessRuleViolationException(string rule)
        : base($"Business rule violated: {rule}")
    {
        Rule = rule;
    }

    public string Rule { get; }
}

// Uso
public async Task<Product> GetByIdAsync(Guid id, CancellationToken cancellationToken)
{
    var product = await _repository.GetByIdAsync(id, cancellationToken);
    
    if (product is null)
    {
        throw new EntityNotFoundException(nameof(Product), id);
    }

    return product;
}

public async Task<Product> CreateAsync(CreateProductRequest request, CancellationToken cancellationToken)
{
    var existing = await _repository.GetByNameAsync(request.Name, cancellationToken);
    
    if (existing is not null)
    {
        throw new DuplicateEntityException(nameof(Product), nameof(Product.Name), request.Name);
    }

    // Validación de regla de negocio
    if (request.Price > 10000 && !request.RequiresApproval)
    {
        throw new BusinessRuleViolationException("Products over $10,000 require approval");
    }

    // Crear producto...
}
```

### Manejo Específico de Excepciones Custom

```csharp
private async Task HandleExceptionAsync(HttpContext context, Exception exception)
{
    context.Response.ContentType = "application/json";

    var (statusCode, message, errorType, details) = exception switch
    {
        ValidationException ve => (400, "Validation failed", "ValidationError", new { ve.Message }),
        
        EntityNotFoundException enf => (404, enf.Message, "NotFoundError", new 
        { 
            EntityName = enf.EntityName, 
            EntityId = enf.EntityId 
        }),
        
        DuplicateEntityException de => (409, de.Message, "DuplicateError", new 
        { 
            EntityName = de.EntityName, 
            PropertyName = de.PropertyName, 
            Value = de.Value 
        }),
        
        BusinessRuleViolationException br => (400, br.Message, "BusinessRuleViolation", new 
        { 
            Rule = br.Rule 
        }),
        
        UnauthorizedAccessException => (401, "Unauthorized access", "UnauthorizedError", null),
        
        _ => (500, "An error occurred", "InternalError", null)
    };

    context.Response.StatusCode = statusCode;

    var response = new
    {
        Type = errorType,
        Title = message,
        Status = statusCode,
        Detail = _environment.IsDevelopment() ? exception.ToString() : null,
        AdditionalInfo = details,
        TraceId = context.TraceIdentifier,
        Timestamp = DateTime.UtcNow
    };

    await context.Response.WriteAsync(JsonSerializer.Serialize(response));
}
```

## Problem Details (RFC 7807)

Formato estándar para respuestas de error:

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});

// Usar en endpoints
[HttpGet("{id}")]
public async Task<ActionResult<ProductDto>> GetById(Guid id, CancellationToken cancellationToken)
{
    var product = await _productService.GetByIdAsync(id, cancellationToken);
    
    if (product is null)
    {
        return Problem(
            statusCode: StatusCodes.Status404NotFound,
            title: "Product not found",
            detail: $"Product with ID {id} does not exist",
            type: "https://myapi.com/errors/not-found"
        );
    }

    return Ok(product);
}
```

## Mejores Prácticas

1. **Usar FluentValidation** para validaciones complejas
2. **Validar en el edge** (controllers/endpoints) antes de pasar a servicios
3. **Retornar errores descriptivos** y estructurados
4. **No exponer stack traces** en producción
5. **Loggear excepciones** con contexto suficiente
6. **Usar excepciones personalizadas** para casos de negocio
7. **Implementar validación asíncrona** cuando se requiera acceso a BD
8. **Seguir RFC 7807** (Problem Details) para consistencia
9. **Proveer traceId** en todas las respuestas de error
10. **Validar tanto entrada como reglas de negocio**

---

## Skills Relacionados

- **Para endpoints que usan validación** → `aspnet-webapi.skill.md`
- **Para autenticar requests antes de validar** → `authentication.skill.md`
- **Para validar con datos de BD** → `ef-core.skill.md`
- **Para testing de validadores** → `testing.skill.md`
- **Para configurar validators en DI** → `di-configuration.skill.md`

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

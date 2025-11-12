# Skill: ASP.NET Web API

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** ASP.NET Core Web API  
**Tipo:** Skill  
**Autor:** SWO Team

## Propósito del Skill

Este skill proporciona patrones y mejores prácticas para construir REST APIs con ASP.NET Core. Se enfoca exclusivamente en la capa de API (endpoints, controladores, routing, HTTP).

## Responsabilidad

**✅ Este skill cubre:**
- Estructura de endpoints y controllers
- Routing y HTTP verbs
- Request/Response models (DTOs)
- Versionado de APIs
- Configuración de Swagger/OpenAPI
- CORS y middleware de API
- Status codes y respuestas HTTP

**❌ Este skill NO cubre:**
- Lógica de negocio (ver servicios en `dotnet-chatmode.md`)
- Acceso a datos (ver `ef-core.skill.md`)
- Autenticación (ver `authentication.skill.md`)
- Validación (ver `validation.skill.md`)

## Minimal APIs (.NET 6+)

### Endpoints CRUD Básicos

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// GET - Obtener todos
app.MapGet("/api/products", async (IProductService service) =>
{
    var products = await service.GetAllAsync();
    return Results.Ok(products);
})
.WithName("GetProducts")
.WithTags("Products")
.Produces<IEnumerable<ProductDto>>(StatusCodes.Status200OK);

// GET por ID
app.MapGet("/api/products/{id:guid}", async (Guid id, IProductService service) =>
{
    var product = await service.GetByIdAsync(id);
    return product is not null 
        ? Results.Ok(product) 
        : Results.NotFound(new { message = $"Product {id} not found" });
})
.WithName("GetProductById")
.Produces<ProductDto>(StatusCodes.Status200OK)
.Produces(StatusCodes.Status404NotFound);

// POST - Crear
app.MapPost("/api/products", async (CreateProductRequest request, IProductService service) =>
{
    var product = await service.CreateAsync(request);
    return Results.CreatedAtRoute("GetProductById", new { id = product.Id }, product);
})
.WithName("CreateProduct")
.Produces<ProductDto>(StatusCodes.Status201Created);

// PUT - Actualizar completo
app.MapPut("/api/products/{id:guid}", async (Guid id, UpdateProductRequest request, IProductService service) =>
{
    var updated = await service.UpdateAsync(id, request);
    return updated ? Results.NoContent() : Results.NotFound();
})
.WithName("UpdateProduct")
.Produces(StatusCodes.Status204NoContent)
.Produces(StatusCodes.Status404NotFound);

// DELETE - Eliminar
app.MapDelete("/api/products/{id:guid}", async (Guid id, IProductService service) =>
{
    var deleted = await service.DeleteAsync(id);
    return deleted ? Results.NoContent() : Results.NotFound();
})
.WithName("DeleteProduct")
.Produces(StatusCodes.Status204NoContent);

app.Run();
```

## Controllers (Enfoque Tradicional)

### Controller REST Completo

```csharp
namespace MyApp.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Produces("application/json")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;

    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    /// <summary>
    /// Obtiene todos los productos con paginación
    /// </summary>
    [HttpGet]
    [ProducesResponseType(typeof(PagedResponse<ProductDto>), StatusCodes.Status200OK)]
    public async Task<ActionResult<PagedResponse<ProductDto>>> GetAll(
        [FromQuery] ProductQueryParameters parameters,
        CancellationToken cancellationToken = default)
    {
        _logger.LogInformation("Getting products with parameters: {@Parameters}", parameters);
        
        var result = await _productService.GetPagedAsync(parameters, cancellationToken);
        return Ok(result);
    }

    /// <summary>
    /// Obtiene un producto específico por ID
    /// </summary>
    [HttpGet("{id:guid}", Name = nameof(GetById))]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetById(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        var product = await _productService.GetByIdAsync(id, cancellationToken);
        
        if (product is null)
        {
            return NotFound(new ProblemDetails 
            { 
                Title = "Product not found",
                Detail = $"Product with ID {id} was not found",
                Status = StatusCodes.Status404NotFound
            });
        }

        return Ok(product);
    }

    /// <summary>
    /// Crea un nuevo producto
    /// </summary>
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ProductDto>> Create(
        [FromBody] CreateProductRequest request,
        CancellationToken cancellationToken = default)
    {
        var product = await _productService.CreateAsync(request, cancellationToken);
        
        _logger.LogInformation("Product {ProductId} created", product.Id);
        
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }

    /// <summary>
    /// Actualiza un producto existente
    /// </summary>
    [HttpPut("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        Guid id,
        [FromBody] UpdateProductRequest request,
        CancellationToken cancellationToken = default)
    {
        var updated = await _productService.UpdateAsync(id, request, cancellationToken);
        
        if (!updated)
        {
            return NotFound();
        }

        return NoContent();
    }

    /// <summary>
    /// Elimina un producto
    /// </summary>
    [HttpDelete("{id:guid}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(
        Guid id,
        CancellationToken cancellationToken = default)
    {
        var deleted = await _productService.DeleteAsync(id, cancellationToken);
        
        if (!deleted)
        {
            return NotFound();
        }

        _logger.LogInformation("Product {ProductId} deleted", id);
        return NoContent();
    }
}
```

## DTOs (Data Transfer Objects)

### Response DTOs

```csharp
namespace MyApp.Api.Models;

// DTO de respuesta simple
public record ProductDto
{
    public Guid Id { get; init; }
    public string Name { get; init; } = string.Empty;
    public decimal Price { get; init; }
    public int Stock { get; init; }
}

// DTO de respuesta paginada
public record PagedResponse<T>
{
    public IEnumerable<T> Items { get; init; } = [];
    public int Page { get; init; }
    public int PageSize { get; init; }
    public int TotalCount { get; init; }
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasPrevious => Page > 1;
    public bool HasNext => Page < TotalPages;
}
```

### Request DTOs

```csharp
// Request para crear
public record CreateProductRequest
{
    public required string Name { get; init; }
    public string? Description { get; init; }
    public decimal Price { get; init; }
    public int Stock { get; init; }
}

// Request para actualizar
public record UpdateProductRequest
{
    public string? Name { get; init; }
    public string? Description { get; init; }
    public decimal? Price { get; init; }
    public int? Stock { get; init; }
}

// Query parameters para filtrado
public record ProductQueryParameters
{
    public int Page { get; init; } = 1;
    public int PageSize { get; init; } = 10;
    public string? Category { get; init; }
    public decimal? MinPrice { get; init; }
    public decimal? MaxPrice { get; init; }
    public string? SortBy { get; init; }
    public bool SortDescending { get; init; }
}
```

## Versionado de API

```csharp
using Asp.Versioning;

// Configuración en Program.cs
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
    options.ApiVersionReader = ApiVersionReader.Combine(
        new UrlSegmentApiVersionReader(),
        new HeaderApiVersionReader("X-Api-Version")
    );
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
    options.SubstituteApiVersionInUrl = true;
});

// Controller V1
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new { version = "1.0", data = [] });
    }
}

// Controller V2 con cambios
[ApiController]
[ApiVersion("2.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll([FromQuery] int page = 1, [FromQuery] int pageSize = 10)
    {
        return Ok(new { version = "2.0", page, pageSize, data = [] });
    }
}
```

## Configuración de Swagger/OpenAPI

```csharp
using Microsoft.OpenApi.Models;

builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Version = "v1",
        Title = "Products API",
        Description = "API para gestión de productos",
        Contact = new OpenApiContact
        {
            Name = "Soporte",
            Email = "soporte@ejemplo.com"
        }
    });

    // Incluir comentarios XML
    var xmlFile = $"{Assembly.GetExecutingAssembly().GetName().Name}.xml";
    var xmlPath = Path.Combine(AppContext.BaseDirectory, xmlFile);
    options.IncludeXmlComments(xmlPath);

    // Agregar autenticación JWT a Swagger
    options.AddSecurityDefinition("Bearer", new OpenApiSecurityScheme
    {
        Description = "JWT Authorization header using the Bearer scheme",
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer"
    });

    options.AddSecurityRequirement(new OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme
            {
                Reference = new OpenApiReference
                {
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});
```

## CORS Configuration

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigins", policy =>
    {
        policy.WithOrigins("https://ejemplo.com", "https://app.ejemplo.com")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });

    options.AddPolicy("Development", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

// Usar política según environment
if (app.Environment.IsDevelopment())
{
    app.UseCors("Development");
}
else
{
    app.UseCors("AllowSpecificOrigins");
}
```

## Response Caching

```csharp
// Configuración
builder.Services.AddResponseCaching();
app.UseResponseCaching();

// En controller
[HttpGet]
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
{
    // Cachea por 60 segundos
    return Ok(await _service.GetAllAsync());
}

// Sin caché
[ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
public async Task<ActionResult> GetSensitiveData()
{
    return Ok(await _service.GetSensitiveDataAsync());
}
```

## Mejores Prácticas

1. **Usar route constraints** para type safety (`{id:guid}`, `{id:int}`, `{id:min(1)}`)
2. **Retornar códigos HTTP apropiados** (200 OK, 201 Created, 204 No Content, 400 Bad Request, 404 Not Found)
3. **Usar DTOs** para requests y responses (nunca exponer entidades de dominio)
4. **Implementar paginación** en endpoints que retornan listas
5. **Agregar documentación XML** para Swagger
6. **Usar `CancellationToken`** en todos los métodos async
7. **Log eventos importantes** usando ILogger
8. **Usar `ProblemDetails`** para respuestas de error consistentes
9. **Versionar la API** desde el inicio
10. **Configurar CORS** apropiadamente según environment

## Skills Relacionados

**Para funcionalidad completa, combinar con:**
- `ef-core.skill.md` → Operaciones de base de datos
- `authentication.skill.md` → Seguridad y autenticación
- `validation.skill.md` → Validación de datos de entrada
- `di-configuration.skill.md` → Configuración de servicios
- `dotnet-chatmode.md` → Lógica de negocio y arquitectura

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

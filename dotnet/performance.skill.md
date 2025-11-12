# Skill: Performance Optimization en .NET

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET 6+ / ASP.NET Core  
**Tipo:** Skill Específico  
**Autor:** SWO Team

## Propósito

Este skill se enfoca exclusivamente en **optimización de performance** en aplicaciones .NET. Cubre caching strategies, async patterns, memory optimization, database optimization, y técnicas avanzadas para mejorar rendimiento.

## Lo que CUBRE este Skill

- ✅ Caching (In-Memory, Distributed, Response Caching)
- ✅ Async/Await patterns y best practices
- ✅ Database query optimization
- ✅ Memory management y reducción de allocations
- ✅ Span<T> y Memory<T> para high-performance scenarios
- ✅ Object pooling
- ✅ Compression y response optimization
- ✅ Profiling y diagnostics

## Lo que NO CUBRE este Skill

- ❌ Implementación de endpoints → Ver `aspnet-webapi.skill.md`
- ❌ Configuración de Entity Framework básica → Ver `ef-core.skill.md`
- ❌ Testing de performance → Ver `testing.skill.md`
- ❌ Configuración de servicios → Ver `di-configuration.skill.md`

## Caching Strategies

### In-Memory Caching

**Configuración básica:**
```csharp
// Program.cs
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024; // Límite de items (opcional)
    options.CompactionPercentage = 0.25; // Eliminar 25% cuando se alcance el límite
});
```

**Uso básico:**
```csharp
using Microsoft.Extensions.Caching.Memory;

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly IMemoryCache _cache;
    private readonly ILogger<ProductService> _logger;

    public ProductService(
        IProductRepository repository,
        IMemoryCache cache,
        ILogger<ProductService> logger)
    {
        _repository = repository;
        _cache = cache;
        _logger = logger;
    }

    public async Task<Product?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        var cacheKey = $"product_{id}";

        // Intentar obtener del cache
        if (_cache.TryGetValue(cacheKey, out Product? cachedProduct))
        {
            _logger.LogDebug("Product {ProductId} retrieved from cache", id);
            return cachedProduct;
        }

        // No está en cache, obtener de BD
        _logger.LogDebug("Product {ProductId} not in cache, fetching from database", id);
        var product = await _repository.GetByIdAsync(id, cancellationToken);

        if (product is not null)
        {
            // Agregar al cache con opciones
            var cacheOptions = new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30),
                SlidingExpiration = TimeSpan.FromMinutes(5),
                Priority = CacheItemPriority.Normal,
                Size = 1 // Si configuraste SizeLimit
            };

            _cache.Set(cacheKey, product, cacheOptions);
            _logger.LogDebug("Product {ProductId} added to cache", id);
        }

        return product;
    }

    public async Task<Product> UpdateAsync(Product product, CancellationToken cancellationToken)
    {
        var updated = await _repository.UpdateAsync(product, cancellationToken);

        // Invalidar cache cuando se actualiza
        var cacheKey = $"product_{product.Id}";
        _cache.Remove(cacheKey);
        _logger.LogDebug("Cache invalidated for product {ProductId}", product.Id);

        return updated;
    }
}
```

**GetOrCreateAsync Pattern:**
```csharp
public async Task<List<Product>> GetAllAsync(CancellationToken cancellationToken)
{
    const string cacheKey = "all_products";

    return await _cache.GetOrCreateAsync(cacheKey, async entry =>
    {
        entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
        entry.SlidingExpiration = TimeSpan.FromMinutes(2);

        _logger.LogDebug("Fetching all products from database");
        return await _repository.GetAllAsync(cancellationToken);
    }) ?? new List<Product>();
}
```

### Distributed Caching (Redis)

**NuGet Package:**
```
Microsoft.Extensions.Caching.StackExchangeRedis
```

**Configuración:**
```csharp
// appsettings.json
{
  "Redis": {
    "Configuration": "localhost:6379",
    "InstanceName": "MyApp_"
  }
}

// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration["Redis:Configuration"];
    options.InstanceName = builder.Configuration["Redis:InstanceName"];
});
```

**Uso con IDistributedCache:**
```csharp
using Microsoft.Extensions.Caching.Distributed;
using System.Text.Json;

public class CachedUserService
{
    private readonly IUserRepository _repository;
    private readonly IDistributedCache _cache;
    private readonly ILogger<CachedUserService> _logger;

    public CachedUserService(
        IUserRepository repository,
        IDistributedCache cache,
        ILogger<CachedUserService> logger)
    {
        _repository = repository;
        _cache = cache;
        _logger = logger;
    }

    public async Task<User?> GetByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        var cacheKey = $"user_{id}";

        // Intentar obtener del cache distribuido
        var cachedData = await _cache.GetStringAsync(cacheKey, cancellationToken);

        if (!string.IsNullOrEmpty(cachedData))
        {
            _logger.LogDebug("User {UserId} retrieved from distributed cache", id);
            return JsonSerializer.Deserialize<User>(cachedData);
        }

        // No está en cache, obtener de BD
        var user = await _repository.GetByIdAsync(id, cancellationToken);

        if (user is not null)
        {
            // Serializar y guardar en cache
            var serialized = JsonSerializer.Serialize(user);
            var options = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
                SlidingExpiration = TimeSpan.FromMinutes(15)
            };

            await _cache.SetStringAsync(cacheKey, serialized, options, cancellationToken);
            _logger.LogDebug("User {UserId} added to distributed cache", id);
        }

        return user;
    }

    public async Task RemoveFromCacheAsync(Guid userId, CancellationToken cancellationToken)
    {
        var cacheKey = $"user_{userId}";
        await _cache.RemoveAsync(cacheKey, cancellationToken);
        _logger.LogDebug("User {UserId} removed from cache", userId);
    }
}
```

### Response Caching

**Configuración:**
```csharp
// Program.cs
builder.Services.AddResponseCaching();

var app = builder.Build();

app.UseResponseCaching();
app.UseAuthentication();
app.UseAuthorization();
```

**Uso en Controllers:**
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
    public async Task<ActionResult<List<ProductDto>>> GetAll()
    {
        // Response cacheada por 60 segundos
        var products = await _productService.GetAllAsync(CancellationToken.None);
        return Ok(products);
    }

    [HttpGet("{id}")]
    [ResponseCache(Duration = 120, VaryByQueryKeys = new[] { "id" })]
    public async Task<ActionResult<ProductDto>> GetById(Guid id)
    {
        // Cacheada por 120 segundos, varía según el id
        var product = await _productService.GetByIdAsync(id, CancellationToken.None);
        return product is not null ? Ok(product) : NotFound();
    }

    [HttpGet("search")]
    [ResponseCache(Duration = 30, VaryByQueryKeys = new[] { "q", "page", "size" })]
    public async Task<ActionResult<List<ProductDto>>> Search(
        [FromQuery] string q,
        [FromQuery] int page = 1,
        [FromQuery] int size = 10)
    {
        // Cache varía según parámetros de query
        var results = await _productService.SearchAsync(q, page, size);
        return Ok(results);
    }

    [HttpPost]
    [ResponseCache(NoStore = true, Location = ResponseCacheLocation.None)]
    public async Task<ActionResult<ProductDto>> Create([FromBody] CreateProductRequest request)
    {
        // No cachear POST/PUT/DELETE
        var product = await _productService.CreateAsync(request);
        return CreatedAtAction(nameof(GetById), new { id = product.Id }, product);
    }
}
```

**Response Caching con Minimal APIs:**
```csharp
app.MapGet("/api/products", async (IProductService service) =>
{
    var products = await service.GetAllAsync(CancellationToken.None);
    return Results.Ok(products);
})
.CacheOutput(policy => policy.Expire(TimeSpan.FromMinutes(5)));
```

## Async/Await Best Practices

### Evitar Async Void

```csharp
// ❌ Mal: Async void (solo en event handlers)
public async void ProcessDataAsync()
{
    await _service.ProcessAsync();
}

// ✅ Bien: Async Task
public async Task ProcessDataAsync()
{
    await _service.ProcessAsync();
}
```

### ConfigureAwait(false) en Libraries

```csharp
// En application code (ASP.NET Core), no es necesario
public async Task<User> GetUserAsync(Guid id)
{
    return await _repository.GetByIdAsync(id);
}

// En library code (packages), usar ConfigureAwait(false)
public async Task<User> GetUserLibraryAsync(Guid id)
{
    var user = await _repository.GetByIdAsync(id).ConfigureAwait(false);
    return user;
}
```

### Operaciones Paralelas

```csharp
// ❌ Mal: Await secuencial innecesario
public async Task<OrderSummary> GetOrderSummaryAsync(Guid orderId)
{
    var order = await _orderRepository.GetByIdAsync(orderId);
    var customer = await _customerRepository.GetByIdAsync(order.CustomerId);
    var products = await _productRepository.GetByIdsAsync(order.ProductIds);

    return new OrderSummary(order, customer, products);
}

// ✅ Bien: Ejecutar en paralelo cuando no hay dependencias
public async Task<OrderSummary> GetOrderSummaryAsync(Guid orderId)
{
    var order = await _orderRepository.GetByIdAsync(orderId);

    // Estas dos queries son independientes - ejecutar en paralelo
    var customerTask = _customerRepository.GetByIdAsync(order.CustomerId);
    var productsTask = _productRepository.GetByIdsAsync(order.ProductIds);

    await Task.WhenAll(customerTask, productsTask);

    var customer = await customerTask;
    var products = await productsTask;

    return new OrderSummary(order, customer, products);
}
```

### ValueTask para Hot Paths

```csharp
// Usar ValueTask cuando el resultado puede ser sincrónico (cache hit)
public async ValueTask<Product?> GetProductAsync(Guid id)
{
    var cacheKey = $"product_{id}";

    if (_cache.TryGetValue(cacheKey, out Product? product))
    {
        // Retorno sincrónico - ValueTask evita allocation de Task
        return product;
    }

    // Cache miss - hacer query async
    product = await _repository.GetByIdAsync(id);

    if (product is not null)
    {
        _cache.Set(cacheKey, product, TimeSpan.FromMinutes(10));
    }

    return product;
}
```

## Database Query Optimization

### AsNoTracking para Queries Read-Only

```csharp
// ❌ Mal: Tracking innecesario para read-only
public async Task<List<Product>> GetAllAsync()
{
    return await _context.Products.ToListAsync();
}

// ✅ Bien: AsNoTracking para read-only
public async Task<List<Product>> GetAllAsync()
{
    return await _context.Products
        .AsNoTracking()
        .ToListAsync();
}
```

### Projection para reducir datos transferidos

```csharp
// ❌ Mal: Traer entidades completas cuando solo necesitas algunos campos
public async Task<List<Product>> GetProductNamesAsync()
{
    var products = await _context.Products.ToListAsync();
    return products; // Trae todos los campos
}

// ✅ Bien: Proyectar solo campos necesarios
public async Task<List<ProductNameDto>> GetProductNamesAsync()
{
    return await _context.Products
        .AsNoTracking()
        .Select(p => new ProductNameDto
        {
            Id = p.Id,
            Name = p.Name
        })
        .ToListAsync();
}
```

### Compiled Queries para Queries Frecuentes

```csharp
// Query que se ejecuta frecuentemente
private static readonly Func<ApplicationDbContext, Guid, Task<Product?>> GetProductByIdQuery =
    EF.CompileAsyncQuery((ApplicationDbContext context, Guid id) =>
        context.Products
            .AsNoTracking()
            .FirstOrDefault(p => p.Id == id));

public async Task<Product?> GetByIdAsync(Guid id)
{
    return await GetProductByIdQuery(_context, id);
}
```

### Split Queries para evitar Cartesian Explosion

```csharp
// ❌ Mal: Single query con múltiples includes puede generar duplicación de datos
public async Task<Order?> GetOrderWithDetailsAsync(Guid id)
{
    return await _context.Orders
        .Include(o => o.OrderItems)
        .Include(o => o.Customer)
        .Include(o => o.ShippingAddress)
        .FirstOrDefaultAsync(o => o.Id == id);
}

// ✅ Bien: Split query para relaciones complejas
public async Task<Order?> GetOrderWithDetailsAsync(Guid id)
{
    return await _context.Orders
        .AsSplitQuery() // Ejecuta queries separadas
        .Include(o => o.OrderItems)
        .Include(o => o.Customer)
        .Include(o => o.ShippingAddress)
        .FirstOrDefaultAsync(o => o.Id == id);
}
```

### Batch Updates con ExecuteUpdate

```csharp
// ❌ Mal: Cargar entidades, modificar, y guardar (N+1 queries)
public async Task IncreaseStockAsync(List<Guid> productIds, int amount)
{
    var products = await _context.Products
        .Where(p => productIds.Contains(p.Id))
        .ToListAsync();

    foreach (var product in products)
    {
        product.Stock += amount;
    }

    await _context.SaveChangesAsync();
}

// ✅ Bien: Bulk update en una sola query (EF Core 7+)
public async Task IncreaseStockAsync(List<Guid> productIds, int amount)
{
    await _context.Products
        .Where(p => productIds.Contains(p.Id))
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(p => p.Stock, p => p.Stock + amount));
}
```

## Memory Optimization

### String Concatenation

```csharp
// ❌ Mal: String concatenation en loop
public string BuildCsv(List<Product> products)
{
    string csv = "Id,Name,Price\n";
    foreach (var product in products)
    {
        csv += $"{product.Id},{product.Name},{product.Price}\n"; // Crea nueva string en cada iteración
    }
    return csv;
}

// ✅ Bien: StringBuilder para concatenación en loops
public string BuildCsv(List<Product> products)
{
    var sb = new StringBuilder();
    sb.AppendLine("Id,Name,Price");
    
    foreach (var product in products)
    {
        sb.AppendLine($"{product.Id},{product.Name},{product.Price}");
    }
    
    return sb.ToString();
}
```

### Span<T> y Memory<T> para High-Performance

```csharp
// Procesar array sin allocations adicionales
public static int CountSpaces(string text)
{
    ReadOnlySpan<char> span = text.AsSpan();
    int count = 0;

    foreach (char c in span)
    {
        if (c == ' ')
            count++;
    }

    return count;
}

// String splitting sin allocations con Span
public static void ParseCsvLine(ReadOnlySpan<char> line)
{
    int start = 0;
    int index = 0;

    while (index < line.Length)
    {
        if (line[index] == ',')
        {
            var field = line.Slice(start, index - start);
            ProcessField(field); // Procesar sin crear substring
            start = index + 1;
        }
        index++;
    }

    // Último campo
    if (start < line.Length)
    {
        var field = line.Slice(start);
        ProcessField(field);
    }
}
```

### Object Pooling

```csharp
using Microsoft.Extensions.ObjectPool;

// Configurar pool
builder.Services.AddSingleton<ObjectPoolProvider, DefaultObjectPoolProvider>();
builder.Services.AddSingleton(serviceProvider =>
{
    var provider = serviceProvider.GetRequiredService<ObjectPoolProvider>();
    return provider.Create(new DefaultPooledObjectPolicy<StringBuilder>());
});

// Uso del pool
public class ReportGenerator
{
    private readonly ObjectPool<StringBuilder> _stringBuilderPool;

    public ReportGenerator(ObjectPool<StringBuilder> stringBuilderPool)
    {
        _stringBuilderPool = stringBuilderPool;
    }

    public string GenerateReport(List<Product> products)
    {
        var sb = _stringBuilderPool.Get();
        
        try
        {
            sb.AppendLine("Product Report");
            sb.AppendLine("==============");

            foreach (var product in products)
            {
                sb.AppendLine($"{product.Name}: ${product.Price}");
            }

            return sb.ToString();
        }
        finally
        {
            sb.Clear();
            _stringBuilderPool.Return(sb);
        }
    }
}
```

## Compression

### Response Compression

```csharp
using Microsoft.AspNetCore.ResponseCompression;

// Program.cs
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<BrotliCompressionProvider>();
    options.Providers.Add<GzipCompressionProvider>();
    options.MimeTypes = ResponseCompressionDefaults.MimeTypes.Concat(
        new[] { "application/json", "text/json", "text/plain" });
});

builder.Services.Configure<BrotliCompressionProviderOptions>(options =>
{
    options.Level = System.IO.Compression.CompressionLevel.Fastest;
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = System.IO.Compression.CompressionLevel.SmallestSize;
});

var app = builder.Build();

app.UseResponseCompression(); // Debe estar antes de UseStaticFiles
app.UseStaticFiles();
```

## Profiling y Diagnostics

### MiniProfiler

**NuGet Package:**
```
MiniProfiler.AspNetCore.Mvc
MiniProfiler.EntityFrameworkCore
```

**Configuración:**
```csharp
// Program.cs
builder.Services.AddMiniProfiler(options =>
{
    options.RouteBasePath = "/profiler";
    options.ColorScheme = StackExchange.Profiling.ColorScheme.Auto;
}).AddEntityFramework();

var app = builder.Build();

app.UseMiniProfiler();
```

**Uso:**
```csharp
using StackExchange.Profiling;

public async Task<Product?> GetByIdAsync(Guid id)
{
    using (MiniProfiler.Current.Step("GetProductById"))
    {
        using (MiniProfiler.Current.Step("Check Cache"))
        {
            if (_cache.TryGetValue($"product_{id}", out Product? cached))
                return cached;
        }

        using (MiniProfiler.Current.Step("Query Database"))
        {
            return await _repository.GetByIdAsync(id);
        }
    }
}
```

### Application Insights (Producción)

```csharp
// NuGet: Microsoft.ApplicationInsights.AspNetCore

builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

// Custom metrics
public class ProductService
{
    private readonly TelemetryClient _telemetryClient;

    public ProductService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task<Product> CreateAsync(CreateProductRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            var product = await _repository.AddAsync(request);

            _telemetryClient.TrackMetric("ProductCreationTime", stopwatch.ElapsedMilliseconds);
            _telemetryClient.TrackEvent("ProductCreated", new Dictionary<string, string>
            {
                { "ProductId", product.Id.ToString() },
                { "Category", product.Category }
            });

            return product;
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
    }
}
```

## Mejores Prácticas

1. **Usar caching apropiado** - In-Memory para datos frecuentes, Distributed para multi-server
2. **AsNoTracking para read-only queries**
3. **Proyectar solo campos necesarios** con Select
4. **Operaciones paralelas** con Task.WhenAll cuando sea posible
5. **ValueTask para hot paths** con resultados síncronos frecuentes
6. **StringBuilder para concatenación** en loops
7. **Response compression** para reducir bandwidth
8. **Compiled queries** para queries muy frecuentes
9. **Object pooling** para objetos costosos de crear
10. **Profiling regular** para identificar bottlenecks

---

## Skills Relacionados

- **Para implementar caching en services** → `aspnet-webapi.skill.md`
- **Para optimizar queries** → `ef-core.skill.md`
- **Para configurar caching** → `di-configuration.skill.md`
- **Para testing de performance** → `testing.skill.md`

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

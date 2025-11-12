# Skill: Entity Framework Core

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** Entity Framework Core  
**Tipo:** Skill  
**Autor:** SWO Team

## Propósito del Skill

Este skill proporciona patrones para operaciones de base de datos usando Entity Framework Core. Se enfoca exclusivamente en la capa de acceso a datos.

## Responsabilidad

**✅ Este skill cubre:**
- Configuración de DbContext
- Definición de entidades y relaciones
- Fluent API configuration
- Queries LINQ
- Migraciones
- Repository pattern
- Transacciones
- Optimización de queries

**❌ Este skill NO cubre:**
- Endpoints de API (ver `aspnet-webapi.skill.md`)
- Lógica de negocio (ver `dotnet-chatmode.md`)
- Validación de datos (ver `validation.skill.md`)

## DbContext Configuration

```csharp
namespace MyApp.Infrastructure.Data;

public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    // DbSets
    public DbSet<User> Users => Set<User>();
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        // Aplicar todas las configuraciones del assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);

        // Global query filters (soft delete)
        modelBuilder.Entity<User>().HasQueryFilter(u => !u.IsDeleted);
        modelBuilder.Entity<Product>().HasQueryFilter(p => !p.IsDeleted);
    }

    // Auto-actualizar campos de auditoría
    public override int SaveChanges()
    {
        UpdateAuditFields();
        return base.SaveChanges();
    }

    public override Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
    {
        UpdateAuditFields();
        return base.SaveChangesAsync(cancellationToken);
    }

    private void UpdateAuditFields()
    {
        var entries = ChangeTracker.Entries()
            .Where(e => e.Entity is IAuditable && 
                       (e.State == EntityState.Added || e.State == EntityState.Modified));

        foreach (var entry in entries)
        {
            var entity = (IAuditable)entry.Entity;

            if (entry.State == EntityState.Added)
            {
                entity.CreatedAt = DateTime.UtcNow;
            }

            entity.UpdatedAt = DateTime.UtcNow;
        }
    }
}

// Interface para auditoría
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    DateTime? UpdatedAt { get; set; }
}
```

## Configuración de Entidades con Fluent API

### Configuración Básica

```csharp
namespace MyApp.Infrastructure.Data.Configurations;

public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        // Tabla
        builder.ToTable("Users");

        // Clave primaria
        builder.HasKey(u => u.Id);

        // Propiedades
        builder.Property(u => u.Email)
            .IsRequired()
            .HasMaxLength(256);

        builder.Property(u => u.FirstName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(u => u.LastName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(u => u.CreatedAt)
            .IsRequired()
            .HasDefaultValueSql("GETUTCDATE()");

        // Índices
        builder.HasIndex(u => u.Email)
            .IsUnique()
            .HasDatabaseName("IX_Users_Email");

        builder.HasIndex(u => new { u.FirstName, u.LastName })
            .HasDatabaseName("IX_Users_Name");

        // Relación uno a uno
        builder.HasOne(u => u.Profile)
            .WithOne(p => p.User)
            .HasForeignKey<UserProfile>(p => p.UserId)
            .OnDelete(DeleteBehavior.Cascade);

        // Relación uno a muchos
        builder.HasMany(u => u.Orders)
            .WithOne(o => o.User)
            .HasForeignKey(o => o.UserId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

### Configuración de Product con Precisión Decimal

```csharp
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("Products");

        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Slug)
            .IsRequired()
            .HasMaxLength(250);

        builder.Property(p => p.Description)
            .HasMaxLength(2000);

        // Precisión para decimales (18 dígitos, 2 decimales)
        builder.Property(p => p.Price)
            .IsRequired()
            .HasPrecision(18, 2);

        builder.Property(p => p.Stock)
            .IsRequired()
            .HasDefaultValue(0);

        // Índice único
        builder.HasIndex(p => p.Slug)
            .IsUnique();

        builder.HasIndex(p => p.CategoryId);

        // Relación muchos a uno
        builder.HasOne(p => p.Category)
            .WithMany(c => c.Products)
            .HasForeignKey(p => p.CategoryId)
            .OnDelete(DeleteBehavior.Restrict);
    }
}
```

### Relación Muchos a Muchos

```csharp
// Entidad intermedia para Order-Product
public class OrderItem
{
    public Guid Id { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    public Guid OrderId { get; set; }
    public Order Order { get; set; } = null!;
    
    public Guid ProductId { get; set; }
    public Product Product { get; set; } = null!;
}

// Configuración
public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        builder.ToTable("OrderItems");

        builder.HasKey(oi => oi.Id);

        builder.Property(oi => oi.UnitPrice)
            .HasPrecision(18, 2);

        builder.Property(oi => oi.Quantity)
            .IsRequired();

        // Relaciones
        builder.HasOne(oi => oi.Order)
            .WithMany(o => o.OrderItems)
            .HasForeignKey(oi => oi.OrderId)
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasOne(oi => oi.Product)
            .WithMany(p => p.OrderItems)
            .HasForeignKey(oi => oi.ProductId)
            .OnDelete(DeleteBehavior.Restrict);

        // Índice compuesto único
        builder.HasIndex(oi => new { oi.OrderId, oi.ProductId })
            .IsUnique();
    }
}
```

## Repository Pattern

### Interface Genérica

```csharp
namespace MyApp.Infrastructure.Repositories;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken cancellationToken = default);
    Task<T> AddAsync(T entity, CancellationToken cancellationToken = default);
    Task UpdateAsync(T entity, CancellationToken cancellationToken = default);
    Task DeleteAsync(T entity, CancellationToken cancellationToken = default);
}
```

### Implementación Genérica

```csharp
public class Repository<T> : IRepository<T> where T : class
{
    protected readonly ApplicationDbContext _context;
    protected readonly DbSet<T> _dbSet;

    public Repository(ApplicationDbContext context)
    {
        _context = context;
        _dbSet = context.Set<T>();
    }

    public virtual async Task<T?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
    {
        return await _dbSet.FindAsync(new object[] { id }, cancellationToken);
    }

    public virtual async Task<IEnumerable<T>> GetAllAsync(CancellationToken cancellationToken = default)
    {
        return await _dbSet.AsNoTracking().ToListAsync(cancellationToken);
    }

    public virtual async Task<T> AddAsync(T entity, CancellationToken cancellationToken = default)
    {
        await _dbSet.AddAsync(entity, cancellationToken);
        await _context.SaveChangesAsync(cancellationToken);
        return entity;
    }

    public virtual async Task UpdateAsync(T entity, CancellationToken cancellationToken = default)
    {
        _dbSet.Update(entity);
        await _context.SaveChangesAsync(cancellationToken);
    }

    public virtual async Task DeleteAsync(T entity, CancellationToken cancellationToken = default)
    {
        _dbSet.Remove(entity);
        await _context.SaveChangesAsync(cancellationToken);
    }
}
```

### Repository Específico con Métodos Personalizados

```csharp
public interface IProductRepository : IRepository<Product>
{
    Task<PagedResult<Product>> GetPagedAsync(ProductQueryParameters parameters, CancellationToken cancellationToken = default);
    Task<IEnumerable<Product>> SearchAsync(string query, CancellationToken cancellationToken = default);
    Task<IEnumerable<Product>> GetByCategoryAsync(string category, CancellationToken cancellationToken = default);
}

public class ProductRepository : Repository<Product>, IProductRepository
{
    public ProductRepository(ApplicationDbContext context) : base(context)
    {
    }

    public async Task<PagedResult<Product>> GetPagedAsync(
        ProductQueryParameters parameters,
        CancellationToken cancellationToken = default)
    {
        var query = _dbSet.AsNoTracking().AsQueryable();

        // Filtrado
        if (!string.IsNullOrWhiteSpace(parameters.Category))
        {
            query = query.Where(p => p.Category == parameters.Category);
        }

        if (parameters.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= parameters.MinPrice.Value);
        }

        if (parameters.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= parameters.MaxPrice.Value);
        }

        // Ordenamiento
        query = parameters.SortBy?.ToLower() switch
        {
            "name" => parameters.SortDescending 
                ? query.OrderByDescending(p => p.Name)
                : query.OrderBy(p => p.Name),
            "price" => parameters.SortDescending
                ? query.OrderByDescending(p => p.Price)
                : query.OrderBy(p => p.Price),
            _ => query.OrderBy(p => p.CreatedAt)
        };

        var totalCount = await query.CountAsync(cancellationToken);

        // Paginación
        var items = await query
            .Skip((parameters.Page - 1) * parameters.PageSize)
            .Take(parameters.PageSize)
            .ToListAsync(cancellationToken);

        return new PagedResult<Product>
        {
            Items = items,
            Page = parameters.Page,
            PageSize = parameters.PageSize,
            TotalCount = totalCount
        };
    }

    public async Task<IEnumerable<Product>> SearchAsync(
        string query,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.Name.Contains(query) || (p.Description != null && p.Description.Contains(query)))
            .Take(50)
            .ToListAsync(cancellationToken);
    }

    public async Task<IEnumerable<Product>> GetByCategoryAsync(
        string category,
        CancellationToken cancellationToken = default)
    {
        return await _dbSet
            .AsNoTracking()
            .Where(p => p.Category == category)
            .OrderBy(p => p.Name)
            .ToListAsync(cancellationToken);
    }
}
```

## Patrones de Queries LINQ

### Queries Básicos

```csharp
// Query simple con AsNoTracking (solo lectura)
public async Task<IEnumerable<Product>> GetAllAsync()
{
    return await _context.Products
        .AsNoTracking()
        .Include(p => p.Category)
        .OrderBy(p => p.Name)
        .ToListAsync();
}

// Query con filtrado
public async Task<IEnumerable<Product>> GetActiveProductsAsync()
{
    return await _context.Products
        .AsNoTracking()
        .Where(p => p.Stock > 0 && !p.IsDeleted)
        .ToListAsync();
}

// Proyección a DTO
public async Task<IEnumerable<ProductSummaryDto>> GetProductSummariesAsync()
{
    return await _context.Products
        .AsNoTracking()
        .Select(p => new ProductSummaryDto
        {
            Id = p.Id,
            Name = p.Name,
            Price = p.Price,
            CategoryName = p.Category.Name
        })
        .ToListAsync();
}
```

### Queries Agregados

```csharp
// Promedio
public async Task<decimal> GetAveragePriceAsync(string category)
{
    return await _context.Products
        .Where(p => p.Category == category)
        .AverageAsync(p => p.Price);
}

// Group By
public async Task<IEnumerable<CategoryStats>> GetCategoryStatsAsync()
{
    return await _context.Products
        .GroupBy(p => p.Category)
        .Select(g => new CategoryStats
        {
            Category = g.Key,
            TotalProducts = g.Count(),
            AveragePrice = g.Average(p => p.Price),
            TotalStock = g.Sum(p => p.Stock)
        })
        .ToListAsync();
}

// Contar
public async Task<int> GetProductCountAsync(string category)
{
    return await _context.Products
        .Where(p => p.Category == category)
        .CountAsync();
}

// Verificar existencia
public async Task<bool> ExistsAsync(Guid id)
{
    return await _context.Products
        .AnyAsync(p => p.Id == id);
}
```

### Raw SQL Queries

```csharp
// Query SQL con interpolación (seguro contra SQL injection)
public async Task<IEnumerable<Product>> GetProductsBySqlAsync(string category)
{
    return await _context.Products
        .FromSqlInterpolated($@"
            SELECT * FROM Products 
            WHERE Category = {category} AND IsDeleted = 0")
        .ToListAsync();
}

// Stored procedure
public async Task<IEnumerable<Product>> GetTopProductsAsync(int count)
{
    return await _context.Products
        .FromSqlInterpolated($"EXEC GetTopProducts @Count = {count}")
        .ToListAsync();
}
```

## Transacciones

```csharp
public class OrderService
{
    private readonly ApplicationDbContext _context;

    public async Task<Order> CreateOrderWithItemsAsync(CreateOrderRequest request)
    {
        using var transaction = await _context.Database.BeginTransactionAsync();
        
        try
        {
            // 1. Crear orden
            var order = new Order
            {
                Id = Guid.NewGuid(),
                UserId = request.UserId,
                Status = OrderStatus.Pending,
                CreatedAt = DateTime.UtcNow
            };

            _context.Orders.Add(order);
            await _context.SaveChangesAsync();

            // 2. Agregar items y actualizar stock
            decimal totalAmount = 0;
            foreach (var item in request.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                
                if (product is null)
                    throw new ProductNotFoundException(item.ProductId);

                if (product.Stock < item.Quantity)
                    throw new InsufficientStockException(item.ProductId);

                var orderItem = new OrderItem
                {
                    Id = Guid.NewGuid(),
                    OrderId = order.Id,
                    ProductId = item.ProductId,
                    Quantity = item.Quantity,
                    UnitPrice = product.Price
                };

                _context.OrderItems.Add(orderItem);
                product.Stock -= item.Quantity;
                totalAmount += orderItem.Quantity * orderItem.UnitPrice;
            }

            order.TotalAmount = totalAmount;
            await _context.SaveChangesAsync();

            // 3. Commit si todo fue exitoso
            await transaction.CommitAsync();
            return order;
        }
        catch
        {
            // Rollback en caso de error
            await transaction.RollbackAsync();
            throw;
        }
    }
}
```

## Migraciones

```bash
# Agregar migración
dotnet ef migrations add NombreMigracion --project Infrastructure --startup-project Api

# Actualizar base de datos
dotnet ef database update --project Infrastructure --startup-project Api

# Remover última migración
dotnet ef migrations remove --project Infrastructure --startup-project Api

# Generar script SQL
dotnet ef migrations script --project Infrastructure --startup-project Api --output migration.sql

# Ver migraciones pendientes
dotnet ef migrations list --project Infrastructure --startup-project Api
```

## Optimización de Performance

### Compiled Queries

```csharp
// Query compilado para ejecución frecuente
private static readonly Func<ApplicationDbContext, Guid, Task<Product?>> GetProductByIdQuery =
    EF.CompileAsyncQuery((ApplicationDbContext context, Guid id) =>
        context.Products
            .Include(p => p.Category)
            .FirstOrDefault(p => p.Id == id));

public async Task<Product?> GetByIdOptimizedAsync(Guid id)
{
    return await GetProductByIdQuery(_context, id);
}
```

### Split Queries para Múltiples Includes

```csharp
// Evitar JOIN cartesiano con AsSplitQuery
public async Task<Order?> GetOrderWithDetailsAsync(Guid orderId)
{
    return await _context.Orders
        .Include(o => o.OrderItems)
            .ThenInclude(oi => oi.Product)
        .Include(o => o.User)
        .AsSplitQuery() // Ejecuta queries separados
        .FirstOrDefaultAsync(o => o.Id == orderId);
}
```

### Batch Updates

```csharp
// Actualizar múltiples entidades en un solo SaveChanges
public async Task UpdatePricesAsync(Dictionary<Guid, decimal> priceUpdates)
{
    var productIds = priceUpdates.Keys.ToList();
    var products = await _context.Products
        .Where(p => productIds.Contains(p.Id))
        .ToListAsync();

    foreach (var product in products)
    {
        if (priceUpdates.TryGetValue(product.Id, out var newPrice))
        {
            product.Price = newPrice;
        }
    }

    await _context.SaveChangesAsync(); // Una sola transacción
}
```

## Mejores Prácticas

1. **Usar AsNoTracking** para queries de solo lectura
2. **Evitar lazy loading** en producción (usar Include explícito)
3. **Proyectar a DTOs** cuando sea posible para seleccionar solo columnas necesarias
4. **Crear índices** en columnas frecuentemente consultadas
5. **Implementar repository pattern** para testabilidad
6. **Usar transacciones** para operaciones que requieren consistencia
7. **Aplicar global query filters** para soft deletes
8. **Separar configuraciones** de entidades en clases individuales
9. **Usar compiled queries** para queries ejecutados frecuentemente
10. **Siempre pasar CancellationToken** en métodos async

## Skills Relacionados

**Para funcionalidad completa, combinar con:**
- `aspnet-webapi.skill.md` → Exponer datos via API
- `validation.skill.md` → Validar datos antes de guardar
- `di-configuration.skill.md` → Registrar DbContext y repositories
- `testing.skill.md` → Testear repositories
- `dotnet-chatmode.md` → Arquitectura y patrones de diseño

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

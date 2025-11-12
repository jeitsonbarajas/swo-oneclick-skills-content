# Skill: Testing en .NET

**Generado:** 2025-11-12  
**Versión:** 1.0  
**Tecnología:** .NET 6+ / xUnit / NUnit / MSTest  
**Tipo:** Skill Específico  
**Autor:** SWO Team

## Propósito

Este skill se enfoca exclusivamente en **testing** de aplicaciones .NET. Cubre unit tests, integration tests, mocking, test fixtures, y mejores prácticas para asegurar calidad de código.

## Lo que CUBRE este Skill

- ✅ Unit Tests con xUnit, NUnit, MSTest
- ✅ Integration Tests para APIs y base de datos
- ✅ Mocking con Moq y NSubstitute
- ✅ Test Fixtures y setup/teardown
- ✅ Testing de servicios, repositories, y controllers
- ✅ In-Memory Database para testing
- ✅ WebApplicationFactory para integration tests
- ✅ Code coverage y reporting

## Lo que NO CUBRE este Skill

- ❌ Implementación de servicios → Ver `aspnet-webapi.skill.md`
- ❌ Configuración de Entity Framework → Ver `ef-core.skill.md`
- ❌ Lógica de autenticación → Ver `authentication.skill.md`
- ❌ Implementación de validators → Ver `validation.skill.md`
- ❌ Performance testing → Ver `performance.skill.md`

## Estructura de Proyecto de Tests

```
tests/
├── MyApp.UnitTests/
│   ├── Services/
│   │   ├── UserServiceTests.cs
│   │   └── ProductServiceTests.cs
│   ├── Validators/
│   │   └── CreateUserRequestValidatorTests.cs
│   ├── Helpers/
│   │   └── TestDataBuilder.cs
│   └── MyApp.UnitTests.csproj
│
├── MyApp.IntegrationTests/
│   ├── Controllers/
│   │   ├── UsersControllerTests.cs
│   │   └── ProductsControllerTests.cs
│   ├── Repositories/
│   │   └── ProductRepositoryTests.cs
│   ├── Fixtures/
│   │   └── WebApplicationFactoryFixture.cs
│   └── MyApp.IntegrationTests.csproj
│
└── MyApp.FunctionalTests/
    ├── Scenarios/
    │   └── UserJourneyTests.cs
    └── MyApp.FunctionalTests.csproj
```

## Unit Tests con xUnit

### Instalación de Packages

**NuGet Packages:**
```
xunit
xunit.runner.visualstudio
Microsoft.NET.Test.Sdk
Moq (para mocking)
FluentAssertions (assertions mejoradas - opcional)
```

### Test Básico de Servicio

```csharp
using Xunit;
using Moq;
using FluentAssertions;

namespace MyApp.UnitTests.Services;

public class ProductServiceTests
{
    private readonly Mock<IProductRepository> _mockRepository;
    private readonly Mock<ILogger<ProductService>> _mockLogger;
    private readonly ProductService _sut; // System Under Test

    public ProductServiceTests()
    {
        _mockRepository = new Mock<IProductRepository>();
        _mockLogger = new Mock<ILogger<ProductService>>();
        _sut = new ProductService(_mockRepository.Object, _mockLogger.Object);
    }

    [Fact]
    public async Task GetByIdAsync_WhenProductExists_ReturnsProduct()
    {
        // Arrange
        var productId = Guid.NewGuid();
        var expectedProduct = new Product
        {
            Id = productId,
            Name = "Test Product",
            Price = 99.99m,
            Stock = 10
        };

        _mockRepository
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ReturnsAsync(expectedProduct);

        // Act
        var result = await _sut.GetByIdAsync(productId, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result!.Id.Should().Be(productId);
        result.Name.Should().Be("Test Product");
        result.Price.Should().Be(99.99m);

        _mockRepository.Verify(
            r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task GetByIdAsync_WhenProductDoesNotExist_ReturnsNull()
    {
        // Arrange
        var productId = Guid.NewGuid();
        
        _mockRepository
            .Setup(r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()))
            .ReturnsAsync((Product?)null);

        // Act
        var result = await _sut.GetByIdAsync(productId, CancellationToken.None);

        // Assert
        result.Should().BeNull();
        
        _mockRepository.Verify(
            r => r.GetByIdAsync(productId, It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task CreateAsync_WithValidRequest_CreatesAndReturnsProduct()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Description = "Description",
            Price = 49.99m,
            Stock = 5
        };

        _mockRepository
            .Setup(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync((Product p, CancellationToken ct) => p);

        // Act
        var result = await _sut.CreateAsync(request, CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be(request.Name);
        result.Price.Should().Be(request.Price);

        _mockRepository.Verify(
            r => r.AddAsync(It.Is<Product>(p => 
                p.Name == request.Name && 
                p.Price == request.Price), 
                It.IsAny<CancellationToken>()),
            Times.Once);
    }

    [Fact]
    public async Task CreateAsync_WhenRepositoryThrows_PropagatesException()
    {
        // Arrange
        var request = new CreateProductRequest { Name = "Test" };
        
        _mockRepository
            .Setup(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
            .ThrowsAsync(new InvalidOperationException("Database error"));

        // Act & Assert
        var act = async () => await _sut.CreateAsync(request, CancellationToken.None);
        
        await act.Should().ThrowAsync<InvalidOperationException>()
            .WithMessage("Database error");
    }
}
```

### Test con Theory (Múltiples Casos)

```csharp
public class CalculatorTests
{
    [Theory]
    [InlineData(2, 3, 5)]
    [InlineData(0, 5, 5)]
    [InlineData(-1, 1, 0)]
    [InlineData(100, 200, 300)]
    public void Add_WithDifferentNumbers_ReturnsCorrectSum(int a, int b, int expected)
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Add(a, b);

        // Assert
        result.Should().Be(expected);
    }

    [Theory]
    [MemberData(nameof(GetDivisionTestData))]
    public void Divide_WithVariousInputs_ReturnsExpectedResult(
        decimal dividend, 
        decimal divisor, 
        decimal expected)
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        var result = calculator.Divide(dividend, divisor);

        // Assert
        result.Should().BeApproximately(expected, 0.001m);
    }

    public static IEnumerable<object[]> GetDivisionTestData()
    {
        yield return new object[] { 10m, 2m, 5m };
        yield return new object[] { 9m, 3m, 3m };
        yield return new object[] { 5m, 2m, 2.5m };
        yield return new object[] { 0m, 5m, 0m };
    }

    [Fact]
    public void Divide_ByZero_ThrowsDivideByZeroException()
    {
        // Arrange
        var calculator = new Calculator();

        // Act
        Action act = () => calculator.Divide(10, 0);

        // Assert
        act.Should().Throw<DivideByZeroException>();
    }
}
```

### Test de Validators

```csharp
public class CreateUserRequestValidatorTests
{
    private readonly CreateUserRequestValidator _validator;

    public CreateUserRequestValidatorTests()
    {
        _validator = new CreateUserRequestValidator();
    }

    [Fact]
    public async Task Validate_WithValidRequest_ShouldNotHaveErrors()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            Email = "test@example.com",
            Password = "Pass123!@#",
            FirstName = "John",
            LastName = "Doe",
            DateOfBirth = DateTime.Now.AddYears(-25)
        };

        // Act
        var result = await _validator.ValidateAsync(request);

        // Assert
        result.IsValid.Should().BeTrue();
        result.Errors.Should().BeEmpty();
    }

    [Theory]
    [InlineData("")]
    [InlineData("invalid-email")]
    [InlineData("@example.com")]
    [InlineData("test@")]
    public async Task Validate_WithInvalidEmail_ShouldHaveError(string email)
    {
        // Arrange
        var request = new CreateUserRequest
        {
            Email = email,
            Password = "Pass123!@#",
            FirstName = "John",
            LastName = "Doe",
            DateOfBirth = DateTime.Now.AddYears(-25)
        };

        // Act
        var result = await _validator.ValidateAsync(request);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(CreateUserRequest.Email));
    }

    [Theory]
    [InlineData("short")] // Too short
    [InlineData("nouppercase123!")] // No uppercase
    [InlineData("NOLOWERCASE123!")] // No lowercase
    [InlineData("NoNumbers!")] // No numbers
    [InlineData("NoSpecialChar123")] // No special char
    public async Task Validate_WithInvalidPassword_ShouldHaveError(string password)
    {
        // Arrange
        var request = new CreateUserRequest
        {
            Email = "test@example.com",
            Password = password,
            FirstName = "John",
            LastName = "Doe",
            DateOfBirth = DateTime.Now.AddYears(-25)
        };

        // Act
        var result = await _validator.ValidateAsync(request);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => e.PropertyName == nameof(CreateUserRequest.Password));
    }

    [Fact]
    public async Task Validate_WithUnderageUser_ShouldHaveError()
    {
        // Arrange
        var request = new CreateUserRequest
        {
            Email = "test@example.com",
            Password = "Pass123!@#",
            FirstName = "John",
            LastName = "Doe",
            DateOfBirth = DateTime.Now.AddYears(-15) // 15 years old
        };

        // Act
        var result = await _validator.ValidateAsync(request);

        // Assert
        result.IsValid.Should().BeFalse();
        result.Errors.Should().Contain(e => 
            e.PropertyName == nameof(CreateUserRequest.DateOfBirth) &&
            e.ErrorMessage.Contains("18"));
    }
}
```

## Integration Tests

### WebApplicationFactory Setup

**Fixture para reutilizar en tests:**

```csharp
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remover el DbContext existente
            services.RemoveAll(typeof(DbContextOptions<ApplicationDbContext>));

            // Agregar DbContext con In-Memory database
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDatabase");
            });

            // Crear y seedear la BD
            var serviceProvider = services.BuildServiceProvider();
            using var scope = serviceProvider.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<ApplicationDbContext>();
            
            db.Database.EnsureCreated();
            SeedTestData(db);
        });
    }

    private static void SeedTestData(ApplicationDbContext db)
    {
        db.Products.AddRange(
            new Product { Id = Guid.NewGuid(), Name = "Product 1", Price = 10m, Stock = 100 },
            new Product { Id = Guid.NewGuid(), Name = "Product 2", Price = 20m, Stock = 50 }
        );

        db.SaveChanges();
    }
}
```

### Integration Test de Controller

```csharp
using System.Net;
using System.Net.Http.Json;
using Xunit;
using FluentAssertions;

public class ProductsControllerIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;
    private readonly CustomWebApplicationFactory _factory;

    public ProductsControllerIntegrationTests(CustomWebApplicationFactory factory)
    {
        _factory = factory;
        _client = factory.CreateClient(new WebApplicationFactoryClientOptions
        {
            AllowAutoRedirect = false
        });
    }

    [Fact]
    public async Task GetAll_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/products");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var products = await response.Content.ReadFromJsonAsync<List<ProductDto>>();
        products.Should().NotBeNull();
        products.Should().HaveCountGreaterThan(0);
    }

    [Fact]
    public async Task GetById_WithExistingId_ReturnsProduct()
    {
        // Arrange - Crear producto primero
        var createRequest = new CreateProductRequest
        {
            Name = "Test Product",
            Description = "Test Description",
            Price = 99.99m,
            Stock = 10
        };

        var createResponse = await _client.PostAsJsonAsync("/api/products", createRequest);
        var createdProduct = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        // Act
        var response = await _client.GetAsync($"/api/products/{createdProduct!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        
        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        product.Should().NotBeNull();
        product!.Name.Should().Be("Test Product");
        product.Price.Should().Be(99.99m);
    }

    [Fact]
    public async Task GetById_WithNonExistingId_ReturnsNotFound()
    {
        // Arrange
        var nonExistingId = Guid.NewGuid();

        // Act
        var response = await _client.GetAsync($"/api/products/{nonExistingId}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task Create_WithValidData_ReturnsCreatedProduct()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "New Product",
            Description = "New Description",
            Price = 49.99m,
            Stock = 5
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();

        var product = await response.Content.ReadFromJsonAsync<ProductDto>();
        product.Should().NotBeNull();
        product!.Name.Should().Be(request.Name);
        product.Price.Should().Be(request.Price);
    }

    [Fact]
    public async Task Create_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var request = new CreateProductRequest
        {
            Name = "", // Invalid - required
            Price = -10m, // Invalid - must be positive
            Stock = -5 // Invalid - cannot be negative
        };

        // Act
        var response = await _client.PostAsJsonAsync("/api/products", request);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task Update_WithValidData_ReturnsNoContent()
    {
        // Arrange - Crear producto
        var createRequest = new CreateProductRequest
        {
            Name = "Original Product",
            Price = 100m,
            Stock = 10
        };

        var createResponse = await _client.PostAsJsonAsync("/api/products", createRequest);
        var createdProduct = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        var updateRequest = new UpdateProductRequest
        {
            Id = createdProduct!.Id,
            Name = "Updated Product",
            Price = 150m,
            Stock = 15
        };

        // Act
        var response = await _client.PutAsJsonAsync(
            $"/api/products/{createdProduct.Id}", 
            updateRequest);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verificar que se actualizó
        var getResponse = await _client.GetAsync($"/api/products/{createdProduct.Id}");
        var updatedProduct = await getResponse.Content.ReadFromJsonAsync<ProductDto>();
        updatedProduct!.Name.Should().Be("Updated Product");
        updatedProduct.Price.Should().Be(150m);
    }

    [Fact]
    public async Task Delete_WithExistingId_ReturnsNoContent()
    {
        // Arrange - Crear producto
        var createRequest = new CreateProductRequest
        {
            Name = "Product to Delete",
            Price = 50m,
            Stock = 5
        };

        var createResponse = await _client.PostAsJsonAsync("/api/products", createRequest);
        var createdProduct = await createResponse.Content.ReadFromJsonAsync<ProductDto>();

        // Act
        var response = await _client.DeleteAsync($"/api/products/{createdProduct!.Id}");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.NoContent);

        // Verificar que ya no existe
        var getResponse = await _client.GetAsync($"/api/products/{createdProduct.Id}");
        getResponse.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

### Integration Test con Autenticación

```csharp
public class SecureEndpointsIntegrationTests : IClassFixture<CustomWebApplicationFactory>
{
    private readonly HttpClient _client;

    public SecureEndpointsIntegrationTests(CustomWebApplicationFactory factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetSecureEndpoint_WithoutToken_ReturnsUnauthorized()
    {
        // Act
        var response = await _client.GetAsync("/api/secure/data");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task GetSecureEndpoint_WithValidToken_ReturnsOk()
    {
        // Arrange - Obtener token
        var loginRequest = new LoginRequest("admin@test.com", "Admin123!");
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var authResponse = await loginResponse.Content.ReadFromJsonAsync<AuthResponse>();

        _client.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", authResponse!.AccessToken);

        // Act
        var response = await _client.GetAsync("/api/secure/data");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }

    [Fact]
    public async Task AdminEndpoint_WithUserRole_ReturnsForbidden()
    {
        // Arrange - Login como usuario normal
        var loginRequest = new LoginRequest("user@test.com", "User123!");
        var loginResponse = await _client.PostAsJsonAsync("/api/auth/login", loginRequest);
        var authResponse = await loginResponse.Content.ReadFromJsonAsync<AuthResponse>();

        _client.DefaultRequestHeaders.Authorization = 
            new System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", authResponse!.AccessToken);

        // Act - Intentar acceder a endpoint admin
        var response = await _client.DeleteAsync("/api/products/some-id");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Forbidden);
    }
}
```

## Mocking Avanzado con Moq

```csharp
public class AdvancedMockingTests
{
    [Fact]
    public async Task MockSetup_WithCallback_ExecutesCustomLogic()
    {
        // Arrange
        var mockRepo = new Mock<IProductRepository>();
        Product? capturedProduct = null;

        mockRepo
            .Setup(r => r.AddAsync(It.IsAny<Product>(), It.IsAny<CancellationToken>()))
            .Callback<Product, CancellationToken>((product, ct) =>
            {
                capturedProduct = product;
                product.Id = Guid.NewGuid(); // Simular asignación de ID
            })
            .ReturnsAsync((Product p, CancellationToken ct) => p);

        var service = new ProductService(mockRepo.Object, Mock.Of<ILogger<ProductService>>());

        // Act
        var request = new CreateProductRequest { Name = "Test" };
        var result = await service.CreateAsync(request, CancellationToken.None);

        // Assert
        capturedProduct.Should().NotBeNull();
        capturedProduct!.Name.Should().Be("Test");
        result.Id.Should().NotBeEmpty();
    }

    [Fact]
    public async Task MockSetup_WithSequence_ReturnsDifferentValues()
    {
        // Arrange
        var mockRepo = new Mock<IProductRepository>();
        
        mockRepo
            .SetupSequence(r => r.GetByIdAsync(It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(new Product { Id = Guid.NewGuid(), Name = "First" })
            .ReturnsAsync(new Product { Id = Guid.NewGuid(), Name = "Second" })
            .ReturnsAsync((Product?)null);

        // Act & Assert
        var first = await mockRepo.Object.GetByIdAsync(Guid.NewGuid(), default);
        first!.Name.Should().Be("First");

        var second = await mockRepo.Object.GetByIdAsync(Guid.NewGuid(), default);
        second!.Name.Should().Be("Second");

        var third = await mockRepo.Object.GetByIdAsync(Guid.NewGuid(), default);
        third.Should().BeNull();
    }

    [Fact]
    public void MockVerify_WithSpecificParameters_VerifiesCorrectly()
    {
        // Arrange
        var mockLogger = new Mock<ILogger<ProductService>>();
        var service = new ProductService(Mock.Of<IProductRepository>(), mockLogger.Object);

        // Act
        var productId = Guid.NewGuid();
        // ... llamar método que loggea

        // Assert - Verificar que se loggeó con parámetros específicos
        mockLogger.Verify(
            x => x.Log(
                LogLevel.Information,
                It.IsAny<EventId>(),
                It.Is<It.IsAnyType>((v, t) => v.ToString()!.Contains(productId.ToString())),
                It.IsAny<Exception>(),
                It.IsAny<Func<It.IsAnyType, Exception?, string>>()),
            Times.Once);
    }
}
```

## Test Data Builders

```csharp
public class ProductBuilder
{
    private Guid _id = Guid.NewGuid();
    private string _name = "Default Product";
    private string _description = "Default Description";
    private decimal _price = 99.99m;
    private int _stock = 10;

    public ProductBuilder WithId(Guid id)
    {
        _id = id;
        return this;
    }

    public ProductBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public ProductBuilder WithPrice(decimal price)
    {
        _price = price;
        return this;
    }

    public ProductBuilder WithNoStock()
    {
        _stock = 0;
        return this;
    }

    public Product Build()
    {
        return new Product
        {
            Id = _id,
            Name = _name,
            Description = _description,
            Price = _price,
            Stock = _stock
        };
    }
}

// Uso en tests
public class ProductServiceTestsWithBuilder
{
    [Fact]
    public async Task ProcessOrder_WithOutOfStockProduct_ThrowsException()
    {
        // Arrange
        var product = new ProductBuilder()
            .WithName("Out of Stock Item")
            .WithNoStock()
            .Build();

        var mockRepo = new Mock<IProductRepository>();
        mockRepo.Setup(r => r.GetByIdAsync(product.Id, default))
                .ReturnsAsync(product);

        var service = new ProductService(mockRepo.Object, Mock.Of<ILogger<ProductService>>());

        // Act & Assert
        var act = async () => await service.ProcessOrder(product.Id, 1, default);
        await act.Should().ThrowAsync<InvalidOperationException>()
            .WithMessage("*out of stock*");
    }
}
```

## Mejores Prácticas

1. **Seguir patrón AAA** (Arrange, Act, Assert) en todos los tests
2. **Usar nombres descriptivos** que expliquen qué se testea y qué se espera
3. **Un assert por test** cuando sea posible
4. **Mockear dependencias externas** (BD, APIs, filesystem)
5. **Usar In-Memory Database** para integration tests
6. **Tests deben ser independientes** y poder ejecutarse en cualquier orden
7. **No testear implementación, testear comportamiento**
8. **Mantener tests simples** y fáciles de entender
9. **Usar builders** para crear objetos complejos
10. **Medir code coverage** pero no obsesionarse con 100%

---

## Skills Relacionados

- **Para servicios a testear** → `aspnet-webapi.skill.md`, `ef-core.skill.md`
- **Para validators a testear** → `validation.skill.md`
- **Para auth a testear** → `authentication.skill.md`
- **Para configurar test project** → `di-configuration.skill.md`

---

*Última actualización: 2025-11-12*  
*Parte del Sistema de Skills .NET*

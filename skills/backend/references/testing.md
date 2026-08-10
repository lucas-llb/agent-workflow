# Testing Patterns in .NET Core

## Unit Testing

### 1. Setup Testing Project

```bash
# Create test project
dotnet new xunit -n ProjectName.Tests

# Navigate to test project
cd ProjectName.Tests

# Add required packages
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
dotnet add package Microsoft.NET.Test.Sdk
dotnet add package Moq
dotnet add package FluentAssertions

# Add reference to main project
dotnet add reference ../ProjectName/ProjectName.csproj
```

### 2. Basic Unit Test Structure

```csharp
using Xunit;
using Moq;
using FluentAssertions;

namespace ProjectName.Tests.Services
{
    public class EntityServiceTests
    {
        private readonly Mock<IRepository<Entity>> _repositoryMock;
        private readonly Mock<ILogger<EntityService>> _loggerMock;
        private readonly EntityService _sut; // System Under Test

        public EntityServiceTests()
        {
            _repositoryMock = new Mock<IRepository<Entity>>();
            _loggerMock = new Mock<ILogger<EntityService>>();
            _sut = new EntityService(_repositoryMock.Object, _loggerMock.Object);
        }

        [Fact]
        public async Task GetByIdAsync_WhenEntityExists_ReturnsEntity()
        {
            // Arrange
            var entityId = 1;
            var expectedEntity = new Entity { Id = entityId, Name = "Test" };
            _repositoryMock.Setup(r => r.GetByIdAsync(entityId))
                .ReturnsAsync(expectedEntity);

            // Act
            var result = await _sut.GetByIdAsync(entityId);

            // Assert
            result.Should().NotBeNull();
            result.Id.Should().Be(entityId);
            result.Name.Should().Be("Test");
        }

        [Fact]
        public async Task GetByIdAsync_WhenEntityDoesNotExist_ReturnsNull()
        {
            // Arrange
            var entityId = 999;
            _repositoryMock.Setup(r => r.GetByIdAsync(entityId))
                .ReturnsAsync((Entity?)null);

            // Act
            var result = await _sut.GetByIdAsync(entityId);

            // Assert
            result.Should().BeNull();
        }

        [Theory]
        [InlineData(1)]
        [InlineData(2)]
        [InlineData(3)]
        public async Task GetByIdAsync_WithDifferentIds_WorksCorrectly(int entityId)
        {
            // Arrange
            var entity = new Entity { Id = entityId };
            _repositoryMock.Setup(r => r.GetByIdAsync(entityId))
                .ReturnsAsync(entity);

            // Act
            var result = await _sut.GetByIdAsync(entityId);

            // Assert
            result.Should().NotBeNull();
            result.Id.Should().Be(entityId);
        }

        [Fact]
        public async Task CreateAsync_WhenValidDto_CreatesEntity()
        {
            // Arrange
            var dto = new CreateEntityDto { Name = "New Entity" };
            var createdEntity = new Entity { Id = 1, Name = dto.Name };
            
            _repositoryMock.Setup(r => r.AddAsync(It.IsAny<Entity>()))
                .ReturnsAsync(createdEntity);

            // Act
            var result = await _sut.CreateAsync(dto);

            // Assert
            result.Should().NotBeNull();
            result.Name.Should().Be(dto.Name);
            _repositoryMock.Verify(r => r.AddAsync(It.IsAny<Entity>()), Times.Once);
        }

        [Fact]
        public async Task CreateAsync_WhenRepositoryThrows_ThrowsException()
        {
            // Arrange
            var dto = new CreateEntityDto { Name = "Test" };
            _repositoryMock.Setup(r => r.AddAsync(It.IsAny<Entity>()))
                .ThrowsAsync(new Exception("Database error"));

            // Act
            Func<Task> act = async () => await _sut.CreateAsync(dto);

            // Assert
            await act.Should().ThrowAsync<Exception>()
                .WithMessage("Database error");
        }
    }
}
```

### 3. Testing with Multiple Mocks

```csharp
public class OrderServiceTests
{
    private readonly Mock<IRepository<Order>> _orderRepositoryMock;
    private readonly Mock<IRepository<Product>> _productRepositoryMock;
    private readonly Mock<IEmailService> _emailServiceMock;
    private readonly Mock<ILogger<OrderService>> _loggerMock;
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _orderRepositoryMock = new Mock<IRepository<Order>>();
        _productRepositoryMock = new Mock<IRepository<Product>>();
        _emailServiceMock = new Mock<IEmailService>();
        _loggerMock = new Mock<ILogger<OrderService>>();
        
        _sut = new OrderService(
            _orderRepositoryMock.Object,
            _productRepositoryMock.Object,
            _emailServiceMock.Object,
            _loggerMock.Object);
    }

    [Fact]
    public async Task PlaceOrderAsync_WhenSuccessful_SendsConfirmationEmail()
    {
        // Arrange
        var order = new CreateOrderDto { ProductId = 1, Quantity = 2 };
        var product = new Product { Id = 1, Price = 10.00m };
        
        _productRepositoryMock.Setup(r => r.GetByIdAsync(1))
            .ReturnsAsync(product);
        _orderRepositoryMock.Setup(r => r.AddAsync(It.IsAny<Order>()))
            .ReturnsAsync(new Order { Id = 1 });

        // Act
        await _sut.PlaceOrderAsync(order);

        // Assert
        _emailServiceMock.Verify(
            e => e.SendOrderConfirmationAsync(It.IsAny<Order>()),
            Times.Once);
    }
}
```

## Integration Testing

### 1. Setup Integration Test Project

```bash
dotnet add package Microsoft.AspNetCore.Mvc.Testing
dotnet add package Microsoft.EntityFrameworkCore.InMemory
```

### 2. Create WebApplicationFactory

```csharp
public class CustomWebApplicationFactory<TProgram>
    : WebApplicationFactory<TProgram> where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the app's DbContext registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<ApplicationDbContext>));

            if (descriptor != null)
            {
                services.Remove(descriptor);
            }

            // Add DbContext using in-memory database
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                options.UseInMemoryDatabase("TestDb");
            });

            // Build the service provider
            var sp = services.BuildServiceProvider();

            // Create a scope to obtain a reference to the database context
            using (var scope = sp.CreateScope())
            {
                var scopedServices = scope.ServiceProvider;
                var db = scopedServices.GetRequiredService<ApplicationDbContext>();

                // Ensure the database is created
                db.Database.EnsureCreated();

                // Seed test data
                SeedTestData(db);
            }
        });
    }

    private static void SeedTestData(ApplicationDbContext context)
    {
        context.Entities.AddRange(
            new Entity { Id = 1, Name = "Test Entity 1" },
            new Entity { Id = 2, Name = "Test Entity 2" }
        );
        context.SaveChanges();
    }
}
```

### 3. Integration Test Example

```csharp
public class EntityControllerIntegrationTests 
    : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public EntityControllerIntegrationTests(
        CustomWebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetAll_ReturnsSuccessStatusCode()
    {
        // Act
        var response = await _client.GetAsync("/api/entity");

        // Assert
        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        content.Should().NotBeNullOrEmpty();
    }

    [Fact]
    public async Task GetById_WithValidId_ReturnsEntity()
    {
        // Act
        var response = await _client.GetAsync("/api/entity/1");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var entity = await response.Content.ReadFromJsonAsync<EntityDto>();
        entity.Should().NotBeNull();
        entity!.Id.Should().Be(1);
    }

    [Fact]
    public async Task Create_WithValidData_CreatesEntity()
    {
        // Arrange
        var newEntity = new CreateEntityDto { Name = "New Entity" };

        // Act
        var response = await _client.PostAsJsonAsync("/api/entity", newEntity);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Created);
        var created = await response.Content.ReadFromJsonAsync<EntityDto>();
        created.Should().NotBeNull();
        created!.Name.Should().Be(newEntity.Name);
    }

    [Fact]
    public async Task Create_WithInvalidData_ReturnsBadRequest()
    {
        // Arrange
        var invalidEntity = new CreateEntityDto(); // Missing required fields

        // Act
        var response = await _client.PostAsJsonAsync("/api/entity", invalidEntity);

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### 4. Testing with Authentication

```csharp
public class AuthenticatedIntegrationTests 
    : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public AuthenticatedIntegrationTests(
        CustomWebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    private async Task<string> GetAuthTokenAsync()
    {
        var loginDto = new LoginDto
        {
            Username = "testuser",
            Password = "TestPassword123!"
        };

        var response = await _client.PostAsJsonAsync("/api/auth/login", loginDto);
        var tokenResponse = await response.Content.ReadFromJsonAsync<TokenResponse>();
        return tokenResponse!.Token;
    }

    [Fact]
    public async Task ProtectedEndpoint_WithoutToken_ReturnsUnauthorized()
    {
        // Act
        var response = await _client.GetAsync("/api/protected");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.Unauthorized);
    }

    [Fact]
    public async Task ProtectedEndpoint_WithValidToken_ReturnsSuccess()
    {
        // Arrange
        var token = await GetAuthTokenAsync();
        _client.DefaultRequestHeaders.Authorization = 
            new AuthenticationHeaderValue("Bearer", token);

        // Act
        var response = await _client.GetAsync("/api/protected");

        // Assert
        response.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

## Repository Testing

### 1. In-Memory Database Testing

```csharp
public class EntityRepositoryTests
{
    private ApplicationDbContext CreateContext()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;

        var context = new ApplicationDbContext(options);
        context.Database.EnsureCreated();
        
        return context;
    }

    [Fact]
    public async Task GetByIdAsync_WhenEntityExists_ReturnsEntity()
    {
        // Arrange
        using var context = CreateContext();
        var repository = new Repository<Entity>(context);
        
        var entity = new Entity { Name = "Test" };
        await context.Entities.AddAsync(entity);
        await context.SaveChangesAsync();

        // Act
        var result = await repository.GetByIdAsync(entity.Id);

        // Assert
        result.Should().NotBeNull();
        result!.Name.Should().Be("Test");
    }

    [Fact]
    public async Task AddAsync_AddsEntityToDatabase()
    {
        // Arrange
        using var context = CreateContext();
        var repository = new Repository<Entity>(context);
        var entity = new Entity { Name = "New Entity" };

        // Act
        await repository.AddAsync(entity);

        // Assert
        var savedEntity = await context.Entities.FindAsync(entity.Id);
        savedEntity.Should().NotBeNull();
        savedEntity!.Name.Should().Be("New Entity");
    }
}
```

## Test Data Builders

### 1. Builder Pattern for Test Data

```csharp
public class EntityBuilder
{
    private int _id = 1;
    private string _name = "Default Name";
    private string _email = "test@example.com";

    public EntityBuilder WithId(int id)
    {
        _id = id;
        return this;
    }

    public EntityBuilder WithName(string name)
    {
        _name = name;
        return this;
    }

    public EntityBuilder WithEmail(string email)
    {
        _email = email;
        return this;
    }

    public Entity Build()
    {
        return new Entity
        {
            Id = _id,
            Name = _name,
            Email = _email
        };
    }
}

// Usage in tests
[Fact]
public async Task TestMethod()
{
    var entity = new EntityBuilder()
        .WithName("Custom Name")
        .WithEmail("custom@example.com")
        .Build();

    // Test with entity
}
```

## Fixture Pattern for Shared Setup

```csharp
public class DatabaseFixture : IDisposable
{
    public ApplicationDbContext Context { get; private set; }

    public DatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<ApplicationDbContext>()
            .UseInMemoryDatabase(databaseName: "TestDatabase")
            .Options;

        Context = new ApplicationDbContext(options);
        Context.Database.EnsureCreated();
        
        SeedData();
    }

    private void SeedData()
    {
        Context.Entities.AddRange(
            new Entity { Id = 1, Name = "Entity 1" },
            new Entity { Id = 2, Name = "Entity 2" }
        );
        Context.SaveChanges();
    }

    public void Dispose()
    {
        Context.Database.EnsureDeleted();
        Context.Dispose();
    }
}

public class EntityTests : IClassFixture<DatabaseFixture>
{
    private readonly ApplicationDbContext _context;

    public EntityTests(DatabaseFixture fixture)
    {
        _context = fixture.Context;
    }

    [Fact]
    public void Test1()
    {
        // Use _context
    }

    [Fact]
    public void Test2()
    {
        // Use same _context
    }
}
```

## Testing Best Practices

### 1. AAA Pattern (Arrange-Act-Assert)

```csharp
[Fact]
public async Task MethodName_StateUnderTest_ExpectedBehavior()
{
    // Arrange - Set up test data and dependencies
    var entity = new Entity { Id = 1, Name = "Test" };
    _repositoryMock.Setup(r => r.GetByIdAsync(1)).ReturnsAsync(entity);

    // Act - Execute the method being tested
    var result = await _sut.GetByIdAsync(1);

    // Assert - Verify the expected outcome
    result.Should().NotBeNull();
    result.Id.Should().Be(1);
}
```

### 2. One Assert Per Test (when possible)

```csharp
// Good
[Fact]
public void User_WhenCreated_HasDefaultRole()
{
    var user = new User("testuser");
    user.Role.Should().Be("User");
}

[Fact]
public void User_WhenCreated_IsActive()
{
    var user = new User("testuser");
    user.IsActive.Should().BeTrue();
}

// Acceptable (related assertions)
[Fact]
public void User_WhenCreated_HasCorrectProperties()
{
    var user = new User("testuser");
    
    user.Should().NotBeNull();
    user.Username.Should().Be("testuser");
    user.Role.Should().Be("User");
    user.IsActive.Should().BeTrue();
}
```

### 3. Test Naming Conventions

```csharp
// Pattern: MethodName_StateUnderTest_ExpectedBehavior

[Fact]
public void GetById_WhenEntityExists_ReturnsEntity() { }

[Fact]
public void GetById_WhenEntityDoesNotExist_ReturnsNull() { }

[Fact]
public void Create_WhenDataIsValid_CreatesEntity() { }

[Fact]
public void Create_WhenDataIsInvalid_ThrowsValidationException() { }
```

### 4. FluentAssertions Usage

```csharp
// Collections
result.Should().NotBeNull();
result.Should().HaveCount(3);
result.Should().Contain(x => x.Id == 1);
result.Should().BeInAscendingOrder(x => x.Name);

// Exceptions
await act.Should().ThrowAsync<ValidationException>()
    .WithMessage("*required*");

// Objects
entity.Should().BeEquivalentTo(expectedEntity);
entity.Should().BeOfType<Entity>();

// Strings
result.Name.Should().StartWith("Test");
result.Name.Should().NotBeNullOrWhiteSpace();
```

## Code Coverage

### 1. Generate Coverage Report

```bash
# Install coverlet
dotnet add package coverlet.collector

# Run tests with coverage
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=opencover

# Generate HTML report (install ReportGenerator)
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:coverage.opencover.xml -targetdir:coverage -reporttypes:Html
```

### 2. Coverage Goals

- **Minimum**: 70% code coverage
- **Recommended**: 80-90% code coverage
- **Critical paths**: 100% code coverage

## Performance Testing

```csharp
[Fact]
public async Task GetAll_PerformanceTest()
{
    var stopwatch = Stopwatch.StartNew();
    
    var result = await _sut.GetAllAsync();
    
    stopwatch.Stop();
    stopwatch.ElapsedMilliseconds.Should().BeLessThan(1000);
}
```

## Best Practices Summary

1. **Write tests first** (TDD) or immediately after implementation
2. **Keep tests simple and focused** - One logical assertion per test
3. **Use meaningful test names** - Describe what is being tested
4. **Follow AAA pattern** - Arrange, Act, Assert
5. **Mock external dependencies** - Don't test third-party code
6. **Use builders for complex objects** - Simplify test data creation
7. **Avoid test interdependence** - Tests should run independently
8. **Use InMemory database for integration tests** - Fast and isolated
9. **Test both success and failure paths** - Cover edge cases
10. **Maintain high code coverage** - Aim for 80%+ coverage

---
name: backend-dotnet
description: Professional backend development with .NET Core and Entity Framework Core following Microsoft best practices. Use when developing or maintaining .NET Core applications, including (1) Creating new backend APIs or services, (2) Implementing CRUD operations with EF Core, (3) Setting up database contexts and migrations, (4) Implementing repository patterns and dependency injection, (5) Creating controllers, services, and middleware, (6) Configuring authentication and authorization, (7) Setting up logging and error handling, or any other .NET Core backend development tasks.
---

# Backend Development with .NET Core & Entity Framework Core

This skill provides comprehensive guidance for building professional backend applications using .NET Core and Entity Framework Core, following Microsoft's best practices and industry standards.

## Core Principles

### 1. Clean Architecture

- **Separation of Concerns**: Organize code into layers (Controllers, Services, Repositories, Models)
- **Dependency Injection**: Use built-in DI container for loose coupling
- **SOLID Principles**: Follow SOLID design principles throughout the codebase

### 2. Project Structure (Clean Architecture)

The recommended structure follows Clean Architecture principles with clear separation of concerns:

```
ProjectRoot/
├── .agent/                           # Agent configuration and skills
├── .github/                          # GitHub workflows and actions
│   ├── workflows/
│   └── actions/
├── deploy/                           # Deployment configurations
│   ├── scripts/                      # Deployment scripts
│   ├── base/                         # Base Kubernetes/Docker configs
│   └── overlays/                     # Environment-specific configs
│       ├── qa/
│       └── prod/
└── src/
    ├── backend/                      # Backend solution
    │   ├── ProjectName.Api/          # API Layer - Controllers, Middleware
    │   ├── ProjectName.Application/  # Application Layer - Use Cases, DTOs, Services
    │   ├── ProjectName.Domain/       # Domain Layer - Entities, Interfaces, Business Rules
    │   └── ProjectName.Infra/        # Infrastructure Layer - DbContext, Repositories, External Services
    └── frontend/                     # Frontend application (Next.js)
        ├── src/
        │   ├── (auth)/              # Auth routes
        │   ├── (onboarding)/        # Onboarding routes
        │   ├── (app)/               # App routes
        │   └── api/                 # API routes
        ├── components/              # React components
        └── lib/                     # Utilities and hooks
```

#### Backend Layer Responsibilities

**ProjectName.Api** - Presentation Layer

- Controllers (API endpoints)
- Middleware
- Filters and attributes
- Program.cs configuration
- API-specific models/DTOs

**ProjectName.Application** - Application Layer

- Application services (use cases)
- DTOs (Data Transfer Objects)
- Application interfaces
- Validators (FluentValidation)
- AutoMapper profiles
- CQRS commands/queries (if using MediatR)

**ProjectName.Domain** - Domain Layer

- Entities (domain models)
- Value Objects
- Domain interfaces (IRepository, etc.)
- Domain events
- Business rules and domain logic
- Domain exceptions

**ProjectName.Infra** - Infrastructure Layer

- DbContext and EF Core configurations
- Repository implementations
- External service integrations
- Caching implementations
- Email/SMS services
- File storage services

## Quick Start Patterns

### 1. Creating a New Clean Architecture Solution

Create a new solution with multiple projects following Clean Architecture:

```bash
# Create solution
dotnet new sln -n ProjectName

# Create source directory
mkdir src
cd src

# Create backend directory
mkdir backend
cd backend

# Create Domain layer (no dependencies)
dotnet new classlib -n ProjectName.Domain
dotnet sln ../../ProjectName.sln add ProjectName.Domain

# Create Application layer (depends on Domain)
dotnet new classlib -n ProjectName.Application
dotnet sln ../../ProjectName.sln add ProjectName.Application
cd ProjectName.Application
dotnet add reference ../ProjectName.Domain
dotnet add package FluentValidation
dotnet add package AutoMapper
cd ..

# Create Infrastructure layer (depends on Domain and Application)
dotnet new classlib -n ProjectName.Infra
dotnet sln ../../ProjectName.sln add ProjectName.Infra
cd ProjectName.Infra
dotnet add reference ../ProjectName.Domain
dotnet add reference ../ProjectName.Application
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package Microsoft.EntityFrameworkCore.Design
cd ..

# Create API layer (depends on all layers)
dotnet new webapi -n ProjectName.Api
dotnet sln ../../ProjectName.sln add ProjectName.Api
cd ProjectName.Api
dotnet add reference ../ProjectName.Domain
dotnet add reference ../ProjectName.Application
dotnet add reference ../ProjectName.Infra
cd ..

# Return to root
cd ../..

# Build the solution
dotnet build
```

### 2. DbContext Configuration

Create a properly configured DbContext in `ProjectName.Infra/Data/ApplicationDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using ProjectName.Domain.Entities;

namespace ProjectName.Infra.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }

        // DbSets
        public DbSet<Entity> Entities { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);
            
            // Apply configurations
            modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
        }
    }
}
```

### 3. Entity Configuration

Create entity configurations in `ProjectName.Infra/Data/Configurations/`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using ProjectName.Domain.Entities;

namespace ProjectName.Infra.Data.Configurations
{
    public class EntityConfiguration : IEntityTypeConfiguration<Entity>
    {
        public void Configure(EntityTypeBuilder<Entity> builder)
        {
            builder.HasKey(e => e.Id);
            
            builder.Property(e => e.Name)
                .IsRequired()
                .HasMaxLength(200);
            
            builder.HasIndex(e => e.Email)
                .IsUnique();
        }
    }
}
```

### 4. Repository Pattern

**Domain Layer** - Define interfaces in `ProjectName.Domain/Interfaces/`:

```csharp
namespace ProjectName.Domain.Interfaces
{
    public interface IRepository<T> where T : class
    {
        Task<T?> GetByIdAsync(int id);
        Task<IEnumerable<T>> GetAllAsync();
        Task<T> AddAsync(T entity);
        Task UpdateAsync(T entity);
        Task DeleteAsync(int id);
    }
}
```

**Infrastructure Layer** - Implement repositories in `ProjectName.Infra/Repositories/`:

```csharp
using Microsoft.EntityFrameworkCore;
using ProjectName.Domain.Interfaces;
using ProjectName.Infra.Data;

namespace ProjectName.Infra.Repositories
{
    public class Repository<T> : IRepository<T> where T : class
    {
        protected readonly ApplicationDbContext _context;
        protected readonly DbSet<T> _dbSet;

        public Repository(ApplicationDbContext context)
        {
            _context = context;
            _dbSet = context.Set<T>();
        }

        public virtual async Task<T?> GetByIdAsync(int id)
        {
            return await _dbSet.FindAsync(id);
        }

        public virtual async Task<IEnumerable<T>> GetAllAsync()
        {
            return await _dbSet.ToListAsync();
        }

        public virtual async Task<T> AddAsync(T entity)
        {
            await _dbSet.AddAsync(entity);
            await _context.SaveChangesAsync();
            return entity;
        }

        public virtual async Task UpdateAsync(T entity)
        {
            _dbSet.Update(entity);
            await _context.SaveChangesAsync();
        }

        public virtual async Task DeleteAsync(int id)
        {
            var entity = await GetByIdAsync(id);
            if (entity != null)
            {
                _dbSet.Remove(entity);
                await _context.SaveChangesAsync();
            }
        }
    }
}
```

### 5. Service Layer Pattern

**Application Layer** - Define interfaces in `ProjectName.Application/Interfaces/`:

```csharp
using ProjectName.Application.DTOs;

namespace ProjectName.Application.Interfaces
{
    public interface IEntityService
    {
        Task<EntityDto?> GetByIdAsync(int id);
        Task<IEnumerable<EntityDto>> GetAllAsync();
        Task<EntityDto> CreateAsync(CreateEntityDto dto);
        Task UpdateAsync(int id, UpdateEntityDto dto);
        Task DeleteAsync(int id);
    }
}
```

**Application Layer** - Implement services in `ProjectName.Application/Services/`:

```csharp
using Microsoft.Extensions.Logging;
using ProjectName.Application.DTOs;
using ProjectName.Application.Interfaces;
using ProjectName.Domain.Entities;
using ProjectName.Domain.Interfaces;

namespace ProjectName.Application.Services
{
    public class EntityService : IEntityService
    {
        private readonly IRepository<Entity> _repository;
        private readonly ILogger<EntityService> _logger;

        public EntityService(IRepository<Entity> repository, ILogger<EntityService> logger)
        {
            _repository = repository;
            _logger = logger;
        }

        public async Task<EntityDto?> GetByIdAsync(int id)
        {
            var entity = await _repository.GetByIdAsync(id);
            return entity == null ? null : MapToDto(entity);
        }

        public async Task<IEnumerable<EntityDto>> GetAllAsync()
        {
            var entities = await _repository.GetAllAsync();
            return entities.Select(MapToDto);
        }

        public async Task<EntityDto> CreateAsync(CreateEntityDto dto)
        {
            var entity = new Entity
            {
                Name = dto.Name,
                // Map other properties
            };

            var created = await _repository.AddAsync(entity);
            return MapToDto(created);
        }

        public async Task UpdateAsync(int id, UpdateEntityDto dto)
        {
            var entity = await _repository.GetByIdAsync(id);
            if (entity == null)
                throw new NotFoundException($"Entity with id {id} not found");

            entity.Name = dto.Name;
            // Update other properties

            await _repository.UpdateAsync(entity);
        }

        public async Task DeleteAsync(int id)
        {
            await _repository.DeleteAsync(id);
        }

        private EntityDto MapToDto(Entity entity)
        {
            return new EntityDto
            {
                Id = entity.Id,
                Name = entity.Name
                // Map other properties
            };
        }
    }
}
```

### 6. Controller Pattern

**API Layer** - Create controllers in `ProjectName.Api/Controllers/`:

```csharp
using Microsoft.AspNetCore.Mvc;
using ProjectName.Application.DTOs;
using ProjectName.Application.Interfaces;

namespace ProjectName.Api.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class EntityController : ControllerBase
    {
        private readonly IEntityService _service;
        private readonly ILogger<EntityController> _logger;

        public EntityController(IEntityService service, ILogger<EntityController> logger)
        {
            _service = service;
            _logger = logger;
        }

        [HttpGet]
        [ProducesResponseType(typeof(IEnumerable<EntityDto>), StatusCodes.Status200OK)]
        public async Task<ActionResult<IEnumerable<EntityDto>>> GetAll()
        {
            var entities = await _service.GetAllAsync();
            return Ok(entities);
        }

        [HttpGet("{id}")]
        [ProducesResponseType(typeof(EntityDto), StatusCodes.Status200OK)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<ActionResult<EntityDto>> GetById(int id)
        {
            var entity = await _service.GetByIdAsync(id);
            if (entity == null)
                return NotFound();
            
            return Ok(entity);
        }

        [HttpPost]
        [ProducesResponseType(typeof(EntityDto), StatusCodes.Status201Created)]
        [ProducesResponseType(StatusCodes.Status400BadRequest)]
        public async Task<ActionResult<EntityDto>> Create([FromBody] CreateEntityDto dto)
        {
            var entity = await _service.CreateAsync(dto);
            return CreatedAtAction(nameof(GetById), new { id = entity.Id }, entity);
        }

        [HttpPut("{id}")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> Update(int id, [FromBody] UpdateEntityDto dto)
        {
            await _service.UpdateAsync(id, dto);
            return NoContent();
        }

        [HttpDelete("{id}")]
        [ProducesResponseType(StatusCodes.Status204NoContent)]
        [ProducesResponseType(StatusCodes.Status404NotFound)]
        public async Task<IActionResult> Delete(int id)
        {
            await _service.DeleteAsync(id);
            return NoContent();
        }
    }
}
```

### 7. Program.cs Configuration

Configure services and middleware in `ProjectName.Api/Program.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using ProjectName.Application.Interfaces;
using ProjectName.Application.Services;
using ProjectName.Domain.Interfaces;
using ProjectName.Infra.Data;
using ProjectName.Infra.Repositories;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Database configuration
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Dependency Injection - Infrastructure
builder.Services.AddScoped(typeof(IRepository<>), typeof(Repository<>));

// Dependency Injection - Application Services
builder.Services.AddScoped<IEntityService, EntityService>();

// CORS configuration
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy.WithOrigins("http://localhost:3000") // Frontend URL
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

var app = builder.Build();

// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

## Best Practices

### 1. Async/Await Pattern

- Always use async/await for database operations
- Use `async Task<T>` for methods that return values
- Use `async Task` for void methods

### 2. Error Handling

- Implement global exception handling middleware
- Use try-catch blocks in services
- Return appropriate HTTP status codes
- Log exceptions properly

### 3. DTOs (Data Transfer Objects)

- Separate models from DTOs
- Use DTOs for API requests and responses
- Implement AutoMapper for object mapping

### 4. Validation

- Use Data Annotations for model validation
- Implement FluentValidation for complex scenarios
- Validate in both controller and service layers

### 5. Migrations

```bash
# Create migration
dotnet ef migrations add MigrationName

# Update database
dotnet ef database update

# Remove last migration
dotnet ef migrations remove
```

## Advanced Features

For detailed guidance on specific advanced topics, see:

- **Authentication & Authorization**: See [references/authentication.md](references/authentication.md)
- **Logging & Monitoring**: See [references/logging.md](references/logging.md)
- **Testing Patterns**: See [references/testing.md](references/testing.md)
- **Performance Optimization**: See [references/performance.md](references/performance.md)
- **Security Best Practices**: See [references/security.md](references/security.md)

## Common Patterns

### Unit of Work Pattern

For complex transactions involving multiple repositories, implement the Unit of Work pattern to ensure data consistency.

### CQRS (Command Query Responsibility Segregation)

For complex applications, consider separating read and write operations using CQRS pattern.

### Mediator Pattern

Use MediatR library to implement mediator pattern for decoupling request/response handling.

## Configuration Management

Store sensitive data in:

- `appsettings.json` for non-sensitive configuration
- User Secrets for development
- Environment variables for production
- Azure Key Vault for production secrets

## API Versioning

Implement API versioning for backward compatibility:

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
});
```

## Health Checks

Implement health checks for monitoring:

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>();

app.MapHealthChecks("/health");
```

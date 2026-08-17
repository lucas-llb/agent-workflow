# Logging & Monitoring in .NET Core

## Built-in Logging

### 1. Configure Logging in Program.cs

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddDebug();
builder.Logging.AddEventSourceLogger();

// Configure log levels in appsettings.json
builder.Logging.AddConfiguration(builder.Configuration.GetSection("Logging"));
```

### 2. Configure appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  }
}
```

### 3. Use ILogger in Services

```csharp
public class EntityService : IEntityService
{
    private readonly IRepository<Entity> _repository;
    private readonly ILogger<EntityService> _logger;

    public EntityService(IRepository<Entity> repository, ILogger<EntityService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<EntityDto> CreateAsync(CreateEntityDto dto)
    {
        _logger.LogInformation("Creating entity with name: {Name}", dto.Name);
        
        try
        {
            var entity = await _repository.AddAsync(dto);
            _logger.LogInformation("Successfully created entity with ID: {Id}", entity.Id);
            return entity;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create entity with name: {Name}", dto.Name);
            throw;
        }
    }
}
```

## Serilog Integration

### 1. Install Serilog Packages

```bash
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.File
dotnet add package Serilog.Sinks.Seq
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

### 2. Configure Serilog in Program.cs

```csharp
using Serilog;
using Serilog.Events;

// Configure Serilog
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Microsoft.EntityFrameworkCore", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithThreadId()
    .WriteTo.Console(
        outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File(
        path: "logs/log-.txt",
        rollingInterval: RollingInterval.Day,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}")
    .WriteTo.Seq("http://localhost:5341") // Optional: centralized logging
    .CreateLogger();

try
{
    Log.Information("Starting web application");

    var builder = WebApplication.CreateBuilder(args);

    // Use Serilog
    builder.Host.UseSerilog();

    // ... rest of configuration

    var app = builder.Build();
    
    // Add Serilog request logging
    app.UseSerilogRequestLogging(options =>
    {
        options.MessageTemplate = "HTTP {RequestMethod} {RequestPath} responded {StatusCode} in {Elapsed:0.0000} ms";
        options.EnrichDiagnosticContext = (diagnosticContext, httpContext) =>
        {
            diagnosticContext.Set("RequestHost", httpContext.Request.Host.Value);
            diagnosticContext.Set("UserAgent", httpContext.Request.Headers["User-Agent"].ToString());
        };
    });

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

### 3. Configure Serilog via appsettings.json

```json
{
  "Serilog": {
    "Using": ["Serilog.Sinks.Console", "Serilog.Sinks.File"],
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/log-.txt",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "outputTemplate": "{Timestamp:yyyy-MM-dd HH:mm:ss.fff zzz} [{Level:u3}] {Message:lj}{NewLine}{Exception}"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithMachineName", "WithThreadId"]
  }
}
```

## Structured Logging

### 1. Use Message Templates with Properties

```csharp
// Good - Structured
_logger.LogInformation("User {UserId} logged in from {IpAddress}", userId, ipAddress);

// Bad - String interpolation
_logger.LogInformation($"User {userId} logged in from {ipAddress}");
```

### 2. Create Scoped Logging

```csharp
public async Task<User> CreateUserAsync(CreateUserDto dto)
{
    using (_logger.BeginScope("CreateUser: {Username}", dto.Username))
    {
        _logger.LogInformation("Starting user creation process");
        
        try
        {
            var user = await _repository.AddAsync(dto);
            _logger.LogInformation("User created successfully with ID: {UserId}", user.Id);
            return user;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "User creation failed");
            throw;
        }
    }
}
```

### 3. Log Different Levels Appropriately

```csharp
// Trace - Very detailed information, typically only enabled during development
_logger.LogTrace("Entering method {MethodName} with parameters: {Parameters}", nameof(CreateAsync), parameters);

// Debug - Information useful for debugging, not typically enabled in production
_logger.LogDebug("Cache miss for key: {CacheKey}", cacheKey);

// Information - General flow of application, important business events
_logger.LogInformation("Order {OrderId} placed by user {UserId}", orderId, userId);

// Warning - Unexpected events that don't cause immediate failure
_logger.LogWarning("API rate limit approaching for user {UserId}: {CurrentCount}/{Limit}", userId, count, limit);

// Error - Errors and exceptions that are handled
_logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);

// Critical - Critical failures that require immediate attention
_logger.LogCritical(ex, "Database connection lost. Application cannot continue.");
```

## Global Exception Handling

### 1. Create Exception Handling Middleware

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            await HandleExceptionAsync(context, ex);
        }
    }

    private async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        _logger.LogError(exception, "An unhandled exception occurred");

        var response = context.Response;
        response.ContentType = "application/json";

        var errorResponse = new ErrorResponse
        {
            Message = "An error occurred while processing your request.",
            TraceId = context.TraceIdentifier
        };

        switch (exception)
        {
            case NotFoundException:
                response.StatusCode = StatusCodes.Status404NotFound;
                errorResponse.Message = exception.Message;
                break;

            case ValidationException:
                response.StatusCode = StatusCodes.Status400BadRequest;
                errorResponse.Message = exception.Message;
                break;

            case UnauthorizedAccessException:
                response.StatusCode = StatusCodes.Status401Unauthorized;
                errorResponse.Message = "Unauthorized access";
                break;

            default:
                response.StatusCode = StatusCodes.Status500InternalServerError;
                break;
        }

        await response.WriteAsJsonAsync(errorResponse);
    }
}

// Register middleware in Program.cs
app.UseMiddleware<ExceptionHandlingMiddleware>();
```

### 2. Custom Exception Classes

```csharp
public class NotFoundException : Exception
{
    public NotFoundException(string message) : base(message) { }
    
    public NotFoundException(string name, object key)
        : base($"Entity \"{name}\" ({key}) was not found.") { }
}

public class ValidationException : Exception
{
    public ValidationException(string message) : base(message) { }
    
    public ValidationException(IEnumerable<string> errors)
        : base($"Validation failed: {string.Join(", ", errors)}") { }
}
```

## Application Insights (Azure)

### 1. Install Application Insights

```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

### 2. Configure Application Insights

```csharp
// In Program.cs
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

// In appsettings.json
{
  "ApplicationInsights": {
    "ConnectionString": "InstrumentationKey=your-key-here"
  }
}
```

### 3. Track Custom Events

```csharp
public class OrderService
{
    private readonly TelemetryClient _telemetryClient;

    public OrderService(TelemetryClient telemetryClient)
    {
        _telemetryClient = telemetryClient;
    }

    public async Task PlaceOrderAsync(Order order)
    {
        var stopwatch = Stopwatch.StartNew();
        
        try
        {
            await _repository.AddAsync(order);
            
            _telemetryClient.TrackEvent("OrderPlaced", new Dictionary<string, string>
            {
                { "OrderId", order.Id.ToString() },
                { "UserId", order.UserId.ToString() },
                { "TotalAmount", order.TotalAmount.ToString() }
            });
        }
        catch (Exception ex)
        {
            _telemetryClient.TrackException(ex);
            throw;
        }
        finally
        {
            stopwatch.Stop();
            _telemetryClient.TrackMetric("OrderPlacementDuration", stopwatch.ElapsedMilliseconds);
        }
    }
}
```

## Health Checks & Monitoring

### 1. Configure Health Checks

```csharp
builder.Services.AddHealthChecks()
    .AddDbContextCheck<ApplicationDbContext>("database")
    .AddUrlGroup(new Uri("https://api.example.com/health"), "external-api")
    .AddCheck("self", () => HealthCheckResult.Healthy());

// Add health check UI (optional)
builder.Services.AddHealthChecksUI(options =>
{
    options.SetEvaluationTimeInSeconds(60);
    options.MaximumHistoryEntriesPerEndpoint(50);
})
.AddInMemoryStorage();

// In middleware
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = async (context, report) =>
    {
        context.Response.ContentType = "application/json";
        var response = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                description = e.Value.Description,
                duration = e.Value.Duration
            }),
            totalDuration = report.TotalDuration
        };
        await context.Response.WriteAsJsonAsync(response);
    }
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false
});
```

### 2. Custom Health Check

```csharp
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly ApplicationDbContext _context;

    public DatabaseHealthCheck(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _context.Database.CanConnectAsync(cancellationToken);
            return HealthCheckResult.Healthy("Database connection is healthy");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Database connection failed", ex);
        }
    }
}

// Register
builder.Services.AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database");
```

## Performance Monitoring

### 1. Track Performance Metrics

```csharp
public class PerformanceLoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceLoggingMiddleware> _logger;

    public PerformanceLoggingMiddleware(
        RequestDelegate next,
        ILogger<PerformanceLoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        
        await _next(context);
        
        stopwatch.Stop();
        
        if (stopwatch.ElapsedMilliseconds > 1000)
        {
            _logger.LogWarning(
                "Slow request: {Method} {Path} took {ElapsedMilliseconds}ms",
                context.Request.Method,
                context.Request.Path,
                stopwatch.ElapsedMilliseconds);
        }
    }
}
```

## Best Practices

1. **Use structured logging** - Always use message templates, not string interpolation
2. **Log at appropriate levels** - Use correct log levels for different scenarios
3. **Include context** - Add correlation IDs, user IDs, and request IDs
4. **Don't log sensitive data** - Never log passwords, tokens, or PII
5. **Use scoped logging** - Group related log entries together
6. **Implement global exception handling** - Catch and log all unhandled exceptions
7. **Monitor performance** - Track slow requests and database queries
8. **Set up alerts** - Configure alerts for critical errors
9. **Rotate log files** - Prevent disk space issues
10. **Use centralized logging** - Consider Seq, ELK Stack, or Application Insights for production

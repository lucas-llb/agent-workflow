# Performance Optimization in .NET Core

## Database Performance

### 1. Use AsNoTracking for Read-Only Queries

```csharp
// Good - No tracking overhead for read-only data
public async Task<IEnumerable<EntityDto>> GetAllAsync()
{
    return await _context.Entities
        .AsNoTracking()
        .Select(e => new EntityDto
        {
            Id = e.Id,
            Name = e.Name
        })
        .ToListAsync();
}

// Bad - Tracking overhead when not needed
public async Task<IEnumerable<Entity>> GetAllAsync()
{
    return await _context.Entities.ToListAsync();
}
```

### 2. Eager Loading vs Select Loading

```csharp
// Good - Eager loading with Include
public async Task<Order> GetOrderWithDetailsAsync(int id)
{
    return await _context.Orders
        .Include(o => o.OrderItems)
        .ThenInclude(oi => oi.Product)
        .FirstOrDefaultAsync(o => o.Id == id);
}

// Good - Select loading (projection)
public async Task<OrderDto> GetOrderAsync(int id)
{
    return await _context.Orders
        .Where(o => o.Id == id)
        .Select(o => new OrderDto
        {
            Id = o.Id,
            Total = o.Total,
            Items = o.OrderItems.Select(oi => new OrderItemDto
            {
                ProductName = oi.Product.Name,
                Quantity = oi.Quantity
            }).ToList()
        })
        .FirstOrDefaultAsync();
}

// Bad - Lazy loading (N+1 problem)
public async Task<Order> GetOrderAsync(int id)
{
    var order = await _context.Orders.FindAsync(id);
    // Each access to OrderItems triggers a separate query
    foreach (var item in order.OrderItems)
    {
        var productName = item.Product.Name; // Another query
    }
    return order;
}
```

### 3. Batch Operations

```csharp
// Good - Batch insert
public async Task AddMultipleAsync(IEnumerable<Entity> entities)
{
    await _context.Entities.AddRangeAsync(entities);
    await _context.SaveChangesAsync();
}

// Bad - Individual inserts
public async Task AddMultipleAsync(IEnumerable<Entity> entities)
{
    foreach (var entity in entities)
    {
        await _context.Entities.AddAsync(entity);
        await _context.SaveChangesAsync(); // Multiple database round trips
    }
}
```

### 4. Use Compiled Queries for Frequently Executed Queries

```csharp
private static readonly Func<ApplicationDbContext, int, Task<Entity?>> 
    GetByIdQuery = EF.CompileAsyncQuery(
        (ApplicationDbContext context, int id) =>
            context.Entities.FirstOrDefault(e => e.Id == id));

public async Task<Entity?> GetByIdAsync(int id)
{
    return await GetByIdQuery(_context, id);
}
```

### 5. Pagination

```csharp
public async Task<PagedResult<EntityDto>> GetPagedAsync(int pageNumber, int pageSize)
{
    var totalCount = await _context.Entities.CountAsync();
    
    var entities = await _context.Entities
        .AsNoTracking()
        .OrderBy(e => e.Id)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .Select(e => new EntityDto { Id = e.Id, Name = e.Name })
        .ToListAsync();

    return new PagedResult<EntityDto>
    {
        Items = entities,
        TotalCount = totalCount,
        PageNumber = pageNumber,
        PageSize = pageSize
    };
}
```

### 6. Avoid Select N+1

```csharp
// Bad - N+1 queries
public async Task<List<OrderDto>> GetOrdersWithCustomersAsync()
{
    var orders = await _context.Orders.ToListAsync();
    var result = new List<OrderDto>();
    
    foreach (var order in orders)
    {
        // Each iteration triggers a new query
        var customer = await _context.Customers.FindAsync(order.CustomerId);
        result.Add(new OrderDto
        {
            OrderId = order.Id,
            CustomerName = customer.Name
        });
    }
    
    return result;
}

// Good - Single query with projection
public async Task<List<OrderDto>> GetOrdersWithCustomersAsync()
{
    return await _context.Orders
        .Include(o => o.Customer)
        .Select(o => new OrderDto
        {
            OrderId = o.Id,
            CustomerName = o.Customer.Name
        })
        .ToListAsync();
}
```

## Caching

### 1. In-Memory Caching

```csharp
public class CachedEntityService : IEntityService
{
    private readonly IEntityService _entityService;
    private readonly IMemoryCache _cache;
    private readonly ILogger<CachedEntityService> _logger;

    public CachedEntityService(
        IEntityService entityService,
        IMemoryCache cache,
        ILogger<CachedEntityService> logger)
    {
        _entityService = entityService;
        _cache = cache;
        _logger = logger;
    }

    public async Task<EntityDto?> GetByIdAsync(int id)
    {
        var cacheKey = $"entity_{id}";
        
        if (_cache.TryGetValue(cacheKey, out EntityDto? cachedEntity))
        {
            _logger.LogDebug("Cache hit for {CacheKey}", cacheKey);
            return cachedEntity;
        }

        _logger.LogDebug("Cache miss for {CacheKey}", cacheKey);
        var entity = await _entityService.GetByIdAsync(id);

        if (entity != null)
        {
            var cacheOptions = new MemoryCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                .SetAbsoluteExpiration(TimeSpan.FromHours(1));

            _cache.Set(cacheKey, entity, cacheOptions);
        }

        return entity;
    }

    public async Task UpdateAsync(int id, UpdateEntityDto dto)
    {
        await _entityService.UpdateAsync(id, dto);
        
        // Invalidate cache
        var cacheKey = $"entity_{id}";
        _cache.Remove(cacheKey);
    }
}

// Register in Program.cs
builder.Services.AddMemoryCache();
builder.Services.AddScoped<IEntityService, EntityService>();
builder.Services.Decorate<IEntityService, CachedEntityService>();
```

### 2. Distributed Caching with Redis

```bash
dotnet add package Microsoft.Extensions.Caching.StackExchangeRedis
```

```csharp
// Configure in Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "SampleInstance";
});

// Usage
public class EntityService
{
    private readonly IDistributedCache _cache;

    public async Task<EntityDto?> GetByIdAsync(int id)
    {
        var cacheKey = $"entity_{id}";
        var cachedData = await _cache.GetStringAsync(cacheKey);

        if (cachedData != null)
        {
            return JsonSerializer.Deserialize<EntityDto>(cachedData);
        }

        var entity = await _repository.GetByIdAsync(id);
        
        if (entity != null)
        {
            var serialized = JsonSerializer.Serialize(entity);
            var cacheOptions = new DistributedCacheEntryOptions()
                .SetSlidingExpiration(TimeSpan.FromMinutes(5))
                .SetAbsoluteExpiration(TimeSpan.FromHours(1));
            
            await _cache.SetStringAsync(cacheKey, serialized, cacheOptions);
        }

        return entity;
    }
}
```

### 3. Response Caching

```csharp
// Configure in Program.cs
builder.Services.AddResponseCaching();

app.UseResponseCaching();

// Use in controller
[HttpGet]
[ResponseCache(Duration = 60, Location = ResponseCacheLocation.Any)]
public async Task<ActionResult<IEnumerable<EntityDto>>> GetAll()
{
    var entities = await _service.GetAllAsync();
    return Ok(entities);
}
```

## Async/Await Best Practices

### 1. Use Async All the Way

```csharp
// Good
public async Task<Entity> GetEntityAsync(int id)
{
    var entity = await _repository.GetByIdAsync(id);
    var relatedData = await _relatedService.GetDataAsync(entity.RelatedId);
    return ProcessEntity(entity, relatedData);
}

// Bad - Blocking on async
public Entity GetEntity(int id)
{
    var entity = _repository.GetByIdAsync(id).Result; // Deadlock risk
    return entity;
}
```

### 2. Parallel Async Operations

```csharp
// Good - Parallel execution
public async Task<CombinedResult> GetCombinedDataAsync()
{
    var task1 = _service1.GetDataAsync();
    var task2 = _service2.GetDataAsync();
    var task3 = _service3.GetDataAsync();

    await Task.WhenAll(task1, task2, task3);

    return new CombinedResult
    {
        Data1 = await task1,
        Data2 = await task2,
        Data3 = await task3
    };
}

// Bad - Sequential execution
public async Task<CombinedResult> GetCombinedDataAsync()
{
    var data1 = await _service1.GetDataAsync();
    var data2 = await _service2.GetDataAsync();
    var data3 = await _service3.GetDataAsync();

    return new CombinedResult
    {
        Data1 = data1,
        Data2 = data2,
        Data3 = data3
    };
}
```

### 3. ConfigureAwait in Libraries

```csharp
// In library code (not ASP.NET Core controllers)
public async Task<Entity> GetEntityAsync(int id)
{
    var entity = await _repository.GetByIdAsync(id).ConfigureAwait(false);
    return entity;
}
```

## Connection Pooling

```csharp
// Configure in appsettings.json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyDb;User Id=sa;Password=Pass;Min Pool Size=5;Max Pool Size=100;"
  }
}
```

## API Performance

### 1. Compression

```csharp
builder.Services.AddResponseCompression(options =>
{
    options.EnableForHttps = true;
    options.Providers.Add<GzipCompressionProvider>();
    options.Providers.Add<BrotliCompressionProvider>();
});

builder.Services.Configure<GzipCompressionProviderOptions>(options =>
{
    options.Level = System.IO.Compression.CompressionLevel.Fastest;
});

app.UseResponseCompression();
```

### 2. Output Caching (.NET 7+)

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    options.AddPolicy("Expire30", builder => builder.Expire(TimeSpan.FromSeconds(30)));
});

app.UseOutputCache();

// In controller
[OutputCache(PolicyName = "Expire30")]
[HttpGet]
public async Task<ActionResult<IEnumerable<EntityDto>>> GetAll()
{
    return Ok(await _service.GetAllAsync());
}
```

### 3. Rate Limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.GlobalLimiter = PartitionedRateLimiter.Create<HttpContext, string>(context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User.Identity?.Name ?? context.Request.Headers.Host.ToString(),
            factory: partition => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,
                Window = TimeSpan.FromMinutes(1)
            }));
});

app.UseRateLimiter();
```

## Memory Management

### 1. Use Span<T> and Memory<T>

```csharp
// Good - Reduces allocations
public void ProcessData(ReadOnlySpan<byte> data)
{
    var slice = data.Slice(0, 10);
    // Process slice without allocating new array
}

// Bad - Creates new arrays
public void ProcessData(byte[] data)
{
    var slice = new byte[10];
    Array.Copy(data, 0, slice, 0, 10);
}
```

### 2. Object Pooling

```csharp
public class PooledObjectFactory : DefaultObjectPoolProvider
{
    public override ObjectPool<T> Create<T>(IPooledObjectPolicy<T> policy)
    {
        return new DefaultObjectPool<T>(policy, 100);
    }
}

// Configure
builder.Services.AddSingleton<ObjectPoolProvider, PooledObjectFactory>();

// Usage
public class MyService
{
    private readonly ObjectPool<StringBuilder> _stringBuilderPool;

    public MyService(ObjectPoolProvider poolProvider)
    {
        _stringBuilderPool = poolProvider.Create(new StringBuilderPooledObjectPolicy());
    }

    public string BuildString()
    {
        var builder = _stringBuilderPool.Get();
        try
        {
            builder.Append("Hello");
            builder.Append(" World");
            return builder.ToString();
        }
        finally
        {
            _stringBuilderPool.Return(builder);
        }
    }
}
```

### 3. Avoid String Concatenation in Loops

```csharp
// Good
var builder = new StringBuilder();
foreach (var item in items)
{
    builder.Append(item.Name);
}
var result = builder.ToString();

// Bad
var result = "";
foreach (var item in items)
{
    result += item.Name; // Creates new string each iteration
}
```

## Benchmarking

### 1. BenchmarkDotNet

```bash
dotnet add package BenchmarkDotNet
```

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class PerformanceBenchmarks
{
    private List<int> _data;

    [GlobalSetup]
    public void Setup()
    {
        _data = Enumerable.Range(1, 1000).ToList();
    }

    [Benchmark]
    public int SumWithFor()
    {
        var sum = 0;
        for (int i = 0; i < _data.Count; i++)
        {
            sum += _data[i];
        }
        return sum;
    }

    [Benchmark]
    public int SumWithLinq()
    {
        return _data.Sum();
    }
}

// Run benchmarks
class Program
{
    static void Main(string[] args)
    {
        var summary = BenchmarkRunner.Run<PerformanceBenchmarks>();
    }
}
```

## Monitoring & Profiling

### 1. Application Insights Performance Monitoring

```csharp
_telemetryClient.TrackMetric("DatabaseQueryTime", elapsed.TotalMilliseconds);
_telemetryClient.TrackDependency("SQL", "GetOrders", startTime, elapsed, success);
```

### 2. Custom Performance Middleware

```csharp
public class PerformanceMonitoringMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<PerformanceMonitoringMiddleware> _logger;

    public PerformanceMonitoringMiddleware(
        RequestDelegate next,
        ILogger<PerformanceMonitoringMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        
        await _next(context);
        
        sw.Stop();
        
        var responseTime = sw.ElapsedMilliseconds;
        
        if (responseTime > 1000)
        {
            _logger.LogWarning(
                "Slow request: {Method} {Path} - {ResponseTime}ms",
                context.Request.Method,
                context.Request.Path,
                responseTime);
        }

        context.Response.Headers.Add("X-Response-Time", $"{responseTime}ms");
    }
}
```

## Best Practices Summary

1. **Use AsNoTracking** for read-only queries
2. **Implement caching** at multiple levels (memory, distributed, response)
3. **Use pagination** for large datasets
4. **Avoid N+1 queries** with eager loading or projections
5. **Use async/await** consistently
6. **Parallelize independent operations** with Task.WhenAll
7. **Enable response compression** for API responses
8. **Implement connection pooling** for database connections
9. **Use compiled queries** for frequently executed queries
10. **Monitor performance** with Application Insights or custom middleware
11. **Profile your application** to identify bottlenecks
12. **Use Span<T> and Memory<T>** to reduce allocations
13. **Implement object pooling** for frequently allocated objects
14. **Batch database operations** when possible
15. **Use appropriate cache expiration** strategies

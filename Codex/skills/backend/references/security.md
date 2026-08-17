# Security Best Practices in .NET Core

## Input Validation

### 1. Data Annotations

```csharp
public class CreateUserDto
{
    [Required(ErrorMessage = "Username is required")]
    [StringLength(50, MinimumLength = 3, ErrorMessage = "Username must be between 3 and 50 characters")]
    [RegularExpression(@"^[a-zA-Z0-9_]+$", ErrorMessage = "Username can only contain letters, numbers, and underscores")]
    public string Username { get; set; } = string.Empty;

    [Required(ErrorMessage = "Email is required")]
    [EmailAddress(ErrorMessage = "Invalid email format")]
    public string Email { get; set; } = string.Empty;

    [Required(ErrorMessage = "Password is required")]
    [StringLength(100, MinimumLength = 8, ErrorMessage = "Password must be at least 8 characters")]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]",
        ErrorMessage = "Password must contain uppercase, lowercase, number, and special character")]
    public string Password { get; set; } = string.Empty;

    [Phone(ErrorMessage = "Invalid phone number")]
    public string? PhoneNumber { get; set; }

    [Range(18, 120, ErrorMessage = "Age must be between 18 and 120")]
    public int Age { get; set; }

    [Url(ErrorMessage = "Invalid URL format")]
    public string? Website { get; set; }
}
```

### 2. FluentValidation

```bash
dotnet add package FluentValidation.AspNetCore
```

```csharp
using FluentValidation;

public class CreateUserDtoValidator : AbstractValidator<CreateUserDto>
{
    public CreateUserDtoValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty().WithMessage("Username is required")
            .Length(3, 50).WithMessage("Username must be between 3 and 50 characters")
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username can only contain letters, numbers, and underscores");

        RuleFor(x => x.Email)
            .NotEmpty().WithMessage("Email is required")
            .EmailAddress().WithMessage("Invalid email format");

        RuleFor(x => x.Password)
            .NotEmpty().WithMessage("Password is required")
            .MinimumLength(8).WithMessage("Password must be at least 8 characters")
            .Matches(@"[A-Z]").WithMessage("Password must contain at least one uppercase letter")
            .Matches(@"[a-z]").WithMessage("Password must contain at least one lowercase letter")
            .Matches(@"[0-9]").WithMessage("Password must contain at least one number")
            .Matches(@"[@$!%*?&]").WithMessage("Password must contain at least one special character");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120).WithMessage("Age must be between 18 and 120");
    }
}

// Register in Program.cs
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserDtoValidator>();

// Use in controller
[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateUserDto dto)
    {
        // Validation happens automatically
        if (!ModelState.IsValid)
            return BadRequest(ModelState);

        // Process valid data
        return Ok();
    }
}
```

### 3. Manual Validation

```csharp
public class ValidationService
{
    public ValidationResult ValidateInput(string input)
    {
        var errors = new List<string>();

        if (string.IsNullOrWhiteSpace(input))
            errors.Add("Input cannot be empty");

        if (input.Length > 1000)
            errors.Add("Input too long");

        // Check for SQL injection patterns
        if (ContainsSqlInjectionPattern(input))
            errors.Add("Invalid input detected");

        // Check for XSS patterns
        if (ContainsXssPattern(input))
            errors.Add("Invalid input detected");

        return new ValidationResult
        {
            IsValid = errors.Count == 0,
            Errors = errors
        };
    }

    private bool ContainsSqlInjectionPattern(string input)
    {
        var sqlPatterns = new[]
        {
            @"('|(--)|;|\s(OR|AND)\s)",
            @"(DROP|DELETE|INSERT|UPDATE|EXEC|EXECUTE)\s",
            @"(UNION\s+SELECT)",
            @"(\bxp_\w+)"
        };

        return sqlPatterns.Any(pattern =>
            Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase));
    }

    private bool ContainsXssPattern(string input)
    {
        var xssPatterns = new[]
        {
            @"<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>",
            @"javascript:",
            @"on\w+\s*=",
            @"<iframe",
            @"<object",
            @"<embed"
        };

        return xssPatterns.Any(pattern =>
            Regex.IsMatch(input, pattern, RegexOptions.IgnoreCase));
    }
}
```

## SQL Injection Prevention

### 1. Always Use Parameterized Queries

```csharp
// Good - Parameterized query
public async Task<User?> GetUserByUsernameAsync(string username)
{
    return await _context.Users
        .Where(u => u.Username == username)
        .FirstOrDefaultAsync();
}

// Good - Raw SQL with parameters
public async Task<User?> GetUserByIdAsync(int id)
{
    return await _context.Users
        .FromSqlRaw("SELECT * FROM Users WHERE Id = {0}", id)
        .FirstOrDefaultAsync();
}

// Bad - String concatenation (SQL Injection vulnerable)
public async Task<User?> GetUserByUsernameAsync(string username)
{
    var query = $"SELECT * FROM Users WHERE Username = '{username}'";
    return await _context.Users.FromSqlRaw(query).FirstOrDefaultAsync();
}
```

### 2. Use EF Core LINQ Queries

```csharp
// EF Core automatically parameterizes LINQ queries
public async Task<List<Product>> GetProductsByCategoryAsync(string category)
{
    return await _context.Products
        .Where(p => p.Category == category)
        .ToListAsync();
}
```

## XSS (Cross-Site Scripting) Prevention

### 1. Automatic Encoding in Razor

```html
<!-- Razor automatically encodes output -->
<p>@Model.UserInput</p>

<!-- Explicitly encoding -->
<p>@Html.Encode(Model.UserInput)</p>

<!-- Raw HTML (use with caution) -->
<p>@Html.Raw(Model.TrustedHtml)</p>
```

### 2. Sanitize HTML Input

```bash
dotnet add package HtmlSanitizer
```

```csharp
using Ganss.Xss;

public class HtmlSanitizerService
{
    private readonly HtmlSanitizer _sanitizer;

    public HtmlSanitizerService()
    {
        _sanitizer = new HtmlSanitizer();
        
        // Configure allowed tags
        _sanitizer.AllowedTags.Clear();
        _sanitizer.AllowedTags.Add("p");
        _sanitizer.AllowedTags.Add("b");
        _sanitizer.AllowedTags.Add("i");
        _sanitizer.AllowedTags.Add("u");
        _sanitizer.AllowedTags.Add("br");
    }

    public string Sanitize(string html)
    {
        return _sanitizer.Sanitize(html);
    }
}

// Usage
public async Task<IActionResult> CreatePost([FromBody] CreatePostDto dto)
{
    dto.Content = _htmlSanitizer.Sanitize(dto.Content);
    await _postService.CreateAsync(dto);
    return Ok();
}
```

## CSRF (Cross-Site Request Forgery) Protection

### 1. Enable Antiforgery Tokens

```csharp
// Configure in Program.cs
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-CSRF-TOKEN";
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.SameSite = SameSiteMode.Strict;
});

// Use in controller
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Create([FromBody] CreateDto dto)
{
    await _service.CreateAsync(dto);
    return Ok();
}
```

### 2. SameSite Cookie Policy

```csharp
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.HttpOnly = true;
});
```

## CORS Configuration

```csharp
// Configure CORS properly
builder.Services.AddCors(options =>
{
    options.AddPolicy("ProductionPolicy", policy =>
    {
        policy.WithOrigins("https://yourdomain.com", "https://www.yourdomain.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });

    options.AddPolicy("DevelopmentPolicy", policy =>
    {
        policy.WithOrigins("http://localhost:3000", "http://localhost:4200")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// Use appropriate policy
if (app.Environment.IsDevelopment())
{
    app.UseCors("DevelopmentPolicy");
}
else
{
    app.UseCors("ProductionPolicy");
}

// Never use in production:
// policy.AllowAnyOrigin() - Security risk!
```

## Secrets Management

### 1. Use User Secrets (Development)

```bash
# Initialize user secrets
dotnet user-secrets init

# Set a secret
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;Database=MyDb;..."
dotnet user-secrets set "Jwt:Key" "YourSecretKeyHere"

# List secrets
dotnet user-secrets list

# Remove a secret
dotnet user-secrets remove "Jwt:Key"
```

```csharp
// Automatically loaded in development
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");
var jwtKey = builder.Configuration["Jwt:Key"];
```

### 2. Environment Variables (Production)

```csharp
// Read from environment variables
var connectionString = Environment.GetEnvironmentVariable("CONNECTION_STRING");
var jwtKey = Environment.GetEnvironmentVariable("JWT_KEY");

// Or via configuration
var jwtKey = builder.Configuration["Jwt:Key"]; // Reads from env var JWT__KEY
```

### 3. Azure Key Vault (Production)

```bash
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Identity
```

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

if (!builder.Environment.IsDevelopment())
{
    var keyVaultEndpoint = new Uri(builder.Configuration["KeyVault:Endpoint"]!);
    var credential = new DefaultAzureCredential();
    
    builder.Configuration.AddAzureKeyVault(keyVaultEndpoint, credential);
}

// Access secrets
var secret = builder.Configuration["SecretName"];
```

## HTTPS Enforcement

```csharp
// Configure HTTPS redirection
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status308PermanentRedirect;
    options.HttpsPort = 443;
});

// Configure HSTS (HTTP Strict Transport Security)
builder.Services.AddHsts(options =>
{
    options.MaxAge = TimeSpan.FromDays(365);
    options.IncludeSubDomains = true;
    options.Preload = true;
});

// Use in middleware
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

## Security Headers

```csharp
public class SecurityHeadersMiddleware
{
    private readonly RequestDelegate _next;

    public SecurityHeadersMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // X-Content-Type-Options
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");

        // X-Frame-Options
        context.Response.Headers.Add("X-Frame-Options", "DENY");

        // X-XSS-Protection
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");

        // Referrer-Policy
        context.Response.Headers.Add("Referrer-Policy", "no-referrer");

        // Content-Security-Policy
        context.Response.Headers.Add("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' 'unsafe-inline' 'unsafe-eval'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "font-src 'self'; " +
            "connect-src 'self'; " +
            "frame-ancestors 'none';");

        // Permissions-Policy
        context.Response.Headers.Add("Permissions-Policy",
            "geolocation=(), microphone=(), camera=()");

        await _next(context);
    }
}

// Register middleware
app.UseMiddleware<SecurityHeadersMiddleware>();
```

## Password Security

### 1. Password Hashing with BCrypt

```bash
dotnet add package BCrypt.Net-Next
```

```csharp
using BCrypt.Net;

public class PasswordService
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password, BCrypt.Net.BCrypt.GenerateSalt(12));
    }

    public bool VerifyPassword(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}
```

### 2. Password Policy

```csharp
public class PasswordValidator
{
    public ValidationResult ValidatePassword(string password)
    {
        var errors = new List<string>();

        if (password.Length < 8)
            errors.Add("Password must be at least 8 characters long");

        if (!password.Any(char.IsUpper))
            errors.Add("Password must contain at least one uppercase letter");

        if (!password.Any(char.IsLower))
            errors.Add("Password must contain at least one lowercase letter");

        if (!password.Any(char.IsDigit))
            errors.Add("Password must contain at least one number");

        if (!password.Any(ch => !char.IsLetterOrDigit(ch)))
            errors.Add("Password must contain at least one special character");

        // Check against common passwords
        if (IsCommonPassword(password))
            errors.Add("Password is too common");

        return new ValidationResult
        {
            IsValid = errors.Count == 0,
            Errors = errors
        };
    }

    private bool IsCommonPassword(string password)
    {
        var commonPasswords = new[]
        {
            "password", "123456", "12345678", "qwerty",
            "abc123", "monkey", "1234567", "letmein"
        };

        return commonPasswords.Contains(password.ToLower());
    }
}
```

## Rate Limiting

```csharp
using System.Threading.RateLimiting;

builder.Services.AddRateLimiter(options =>
{
    // Fixed window rate limiter
    options.AddFixedWindowLimiter("fixed", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 5;
    });

    // Sliding window rate limiter
    options.AddSlidingWindowLimiter("sliding", limiterOptions =>
    {
        limiterOptions.PermitLimit = 100;
        limiterOptions.Window = TimeSpan.FromMinutes(1);
        limiterOptions.SegmentsPerWindow = 6;
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 5;
    });

    // Token bucket rate limiter
    options.AddTokenBucketLimiter("token", limiterOptions =>
    {
        limiterOptions.TokenLimit = 100;
        limiterOptions.ReplenishmentPeriod = TimeSpan.FromMinutes(1);
        limiterOptions.TokensPerPeriod = 50;
        limiterOptions.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        limiterOptions.QueueLimit = 5;
    });

    options.OnRejected = async (context, token) =>
    {
        context.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await context.HttpContext.Response.WriteAsync("Rate limit exceeded. Please try again later.", token);
    };
});

app.UseRateLimiter();

// Use on endpoints
[EnableRateLimiting("fixed")]
[HttpGet]
public IActionResult GetAll()
{
    return Ok();
}
```

## Sensitive Data Protection

### 1. Data Protection API

```csharp
builder.Services.AddDataProtection()
    .SetApplicationName("MyApp")
    .PersistKeysToFileSystem(new DirectoryInfo(@"C:\keys"))
    .SetDefaultKeyLifetime(TimeSpan.FromDays(90));

public class EncryptionService
{
    private readonly IDataProtector _protector;

    public EncryptionService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("MyApp.Encryption");
    }

    public string Encrypt(string plainText)
    {
        return _protector.Protect(plainText);
    }

    public string Decrypt(string cipherText)
    {
        return _protector.Unprotect(cipherText);
    }
}
```

### 2. Never Log Sensitive Data

```csharp
// Bad - Logging sensitive data
_logger.LogInformation("User {Username} logged in with password {Password}", username, password);

// Good - Don't log sensitive data
_logger.LogInformation("User {Username} logged in successfully", username);

// Good - Mask sensitive data if you must log it
_logger.LogInformation("Processing credit card ending in {Last4}", creditCard.Substring(creditCard.Length - 4));
```

## API Security Checklist

- [ ] Use HTTPS everywhere
- [ ] Implement authentication (JWT, OAuth, etc.)
- [ ] Implement authorization (role-based, policy-based)
- [ ] Validate all input
- [ ] Use parameterized queries
- [ ] Enable CORS properly (not AllowAnyOrigin in production)
- [ ] Implement rate limiting
- [ ] Add security headers
- [ ] Hash passwords with strong algorithms
- [ ] Use secrets management (not hardcoded secrets)
- [ ] Implement CSRF protection
- [ ] Sanitize HTML input
- [ ] Enable request size limits
- [ ] Implement audit logging
- [ ] Keep dependencies updated
- [ ] Use Security Scanning tools

## Security Scanning

```bash
# Install security scanning tool
dotnet tool install --global security-scan

# Run security scan
dotnet list package --vulnerable
dotnet list package --outdated
```

## Best Practices Summary

1. **Always validate input** - Use Data Annotations or FluentValidation
2. **Use parameterized queries** - Prevent SQL injection
3. **Sanitize HTML** - Prevent XSS attacks
4. **Implement authentication & authorization** - Secure your endpoints
5. **Use HTTPS** - Encrypt data in transit
6. **Store secrets securely** - Never hardcode secrets
7. **Hash passwords** - Use BCrypt or Identity framework
8. **Implement rate limiting** - Prevent abuse
9. **Add security headers** - Protect against common attacks
10. **Enable CORS properly** - Don't allow all origins in production
11. **Keep dependencies updated** - Patch known vulnerabilities
12. **Implement audit logging** - Track security events
13. **Use Data Protection API** - Encrypt sensitive data at rest
14. **Follow principle of least privilege** - Minimize permissions
15. **Regular security audits** - Scan for vulnerabilities

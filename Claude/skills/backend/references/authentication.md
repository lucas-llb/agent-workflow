# Authentication & Authorization in .NET Core

## JWT Authentication

### 1. Install Required Packages

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 2. Configure JWT in Program.cs

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

// Add authentication services
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
    };
});

builder.Services.AddAuthorization();

// In middleware configuration
app.UseAuthentication();
app.UseAuthorization();
```

### 3. Configure appsettings.json

```json
{
  "Jwt": {
    "Key": "YourSuperSecretKeyHere_MinimumLengthRequired",
    "Issuer": "YourApp",
    "Audience": "YourAppUsers",
    "ExpiryMinutes": 60
  }
}
```

### 4. Create Token Service

```csharp
public interface ITokenService
{
    string GenerateToken(User user);
}

public class TokenService : ITokenService
{
    private readonly IConfiguration _configuration;

    public TokenService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public string GenerateToken(User user)
    {
        var securityKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]!));
        var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim(ClaimTypes.Role, user.Role)
        };

        var token = new JwtSecurityToken(
            issuer: _configuration["Jwt:Issuer"],
            audience: _configuration["Jwt:Audience"],
            claims: claims,
            expires: DateTime.Now.AddMinutes(
                Convert.ToDouble(_configuration["Jwt:ExpiryMinutes"])),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

### 5. Create Authentication Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;
    private readonly ITokenService _tokenService;

    public AuthController(IAuthService authService, ITokenService tokenService)
    {
        _authService = authService;
        _tokenService = tokenService;
    }

    [HttpPost("login")]
    [ProducesResponseType(typeof(TokenResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status401Unauthorized)]
    public async Task<ActionResult<TokenResponse>> Login([FromBody] LoginRequest request)
    {
        var user = await _authService.ValidateUserAsync(request.Username, request.Password);
        
        if (user == null)
            return Unauthorized(new { message = "Invalid credentials" });

        var token = _tokenService.GenerateToken(user);
        
        return Ok(new TokenResponse
        {
            Token = token,
            ExpiresIn = Convert.ToInt32(_configuration["Jwt:ExpiryMinutes"]) * 60
        });
    }

    [HttpPost("register")]
    [ProducesResponseType(typeof(UserDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<UserDto>> Register([FromBody] RegisterRequest request)
    {
        var user = await _authService.RegisterUserAsync(request);
        return CreatedAtAction(nameof(GetProfile), new { id = user.Id }, user);
    }

    [Authorize]
    [HttpGet("profile")]
    [ProducesResponseType(typeof(UserDto), StatusCodes.Status200OK)]
    public async Task<ActionResult<UserDto>> GetProfile()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var user = await _authService.GetUserByIdAsync(int.Parse(userId!));
        return Ok(user);
    }
}
```

### 6. Protect Endpoints with [Authorize]

```csharp
[Authorize] // Requires authentication
[HttpGet("protected")]
public IActionResult Protected()
{
    return Ok("This is protected");
}

[Authorize(Roles = "Admin")] // Requires specific role
[HttpGet("admin-only")]
public IActionResult AdminOnly()
{
    return Ok("Admin access");
}

[AllowAnonymous] // Allows anonymous access
[HttpGet("public")]
public IActionResult Public()
{
    return Ok("Public access");
}
```

## Identity Framework Integration

### 1. Install Identity Packages

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

### 2. Configure Identity

```csharp
// Update DbContext to inherit from IdentityDbContext
public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }
}

// Configure Identity in Program.cs
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // Password settings
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = true;
    options.Password.RequiredLength = 8;

    // Lockout settings
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);
    options.Lockout.MaxFailedAccessAttempts = 5;

    // User settings
    options.User.RequireUniqueEmail = true;
})
.AddEntityFrameworkStores<ApplicationDbContext>()
.AddDefaultTokenProviders();
```

### 3. Custom User Model

```csharp
public class ApplicationUser : IdentityUser
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
}
```

## Role-Based Authorization

### 1. Define Roles

```csharp
public static class Roles
{
    public const string Admin = "Admin";
    public const string User = "User";
    public const string Manager = "Manager";
}
```

### 2. Seed Roles

```csharp
public static async Task SeedRolesAsync(RoleManager<IdentityRole> roleManager)
{
    var roles = new[] { Roles.Admin, Roles.User, Roles.Manager };
    
    foreach (var role in roles)
    {
        if (!await roleManager.RoleExistsAsync(role))
        {
            await roleManager.CreateAsync(new IdentityRole(role));
        }
    }
}
```

### 3. Assign Roles to Users

```csharp
public async Task<bool> AssignRoleAsync(string userId, string roleName)
{
    var user = await _userManager.FindByIdAsync(userId);
    if (user == null) return false;

    var result = await _userManager.AddToRoleAsync(user, roleName);
    return result.Succeeded;
}
```

## Policy-Based Authorization

### 1. Define Policies

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("RequireAdminRole", policy => 
        policy.RequireRole(Roles.Admin));
    
    options.AddPolicy("MinimumAge", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
    
    options.AddPolicy("CanEditResource", policy =>
        policy.Requirements.Add(new ResourceOwnerRequirement()));
});
```

### 2. Create Custom Requirements

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}

public class MinimumAgeHandler : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var dateOfBirth = context.User.FindFirst(c => c.Type == "DateOfBirth")?.Value;
        
        if (dateOfBirth == null)
        {
            return Task.CompletedTask;
        }

        var age = DateTime.Today.Year - DateTime.Parse(dateOfBirth).Year;
        
        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}

// Register handler
builder.Services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();
```

### 3. Use Policies

```csharp
[Authorize(Policy = "RequireAdminRole")]
[HttpPost("admin-action")]
public IActionResult AdminAction()
{
    return Ok("Admin action");
}
```

## Refresh Token Implementation

### 1. Refresh Token Model

```csharp
public class RefreshToken
{
    public int Id { get; set; }
    public string Token { get; set; } = string.Empty;
    public int UserId { get; set; }
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public string? RevokedByIp { get; set; }
    public DateTime? RevokedAt { get; set; }
    public bool IsActive => RevokedAt == null && DateTime.UtcNow < ExpiresAt;
}
```

### 2. Generate Refresh Token

```csharp
public string GenerateRefreshToken()
{
    var randomBytes = new byte[64];
    using var rng = RandomNumberGenerator.Create();
    rng.GetBytes(randomBytes);
    return Convert.ToBase64String(randomBytes);
}
```

### 3. Refresh Token Endpoint

```csharp
[HttpPost("refresh-token")]
public async Task<ActionResult<TokenResponse>> RefreshToken([FromBody] RefreshTokenRequest request)
{
    var refreshToken = await _authService.GetRefreshTokenAsync(request.RefreshToken);
    
    if (refreshToken == null || !refreshToken.IsActive)
        return Unauthorized(new { message = "Invalid refresh token" });

    var user = await _authService.GetUserByIdAsync(refreshToken.UserId);
    var newAccessToken = _tokenService.GenerateToken(user);
    var newRefreshToken = _authService.GenerateRefreshToken();
    
    await _authService.RevokeRefreshTokenAsync(request.RefreshToken);
    await _authService.SaveRefreshTokenAsync(user.Id, newRefreshToken);

    return Ok(new TokenResponse
    {
        Token = newAccessToken,
        RefreshToken = newRefreshToken,
        ExpiresIn = 3600
    });
}
```

## Password Hashing

Never store plain text passwords. Use .NET's built-in hashing:

```csharp
public class PasswordHasher
{
    public string HashPassword(string password)
    {
        return BCrypt.Net.BCrypt.HashPassword(password);
    }

    public bool VerifyPassword(string password, string hash)
    {
        return BCrypt.Net.BCrypt.Verify(password, hash);
    }
}

// Or use Identity's PasswordHasher
var hasher = new PasswordHasher<User>();
var hashedPassword = hasher.HashPassword(user, password);
var verificationResult = hasher.VerifyHashedPassword(user, hashedPassword, password);
```

## Best Practices

1. **Never store secrets in code** - Use User Secrets, environment variables, or Azure Key Vault
2. **Use HTTPS** - Always enforce HTTPS in production
3. **Implement rate limiting** - Prevent brute force attacks
4. **Use strong password policies** - Enforce minimum requirements
5. **Implement account lockout** - Protect against brute force
6. **Validate tokens properly** - Check expiration, issuer, audience
7. **Use secure token storage** - Store in httpOnly cookies or secure storage
8. **Implement token revocation** - For logout and security incidents
9. **Use CORS carefully** - Don't allow all origins in production
10. **Log authentication events** - Track login attempts, failures, and successes

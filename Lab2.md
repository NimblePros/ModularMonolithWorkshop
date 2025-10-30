# Lab 2: Configure FastEndpoints, ASP.NET Core Identity, and Produce a Running App

Lab 1 didn't quite get us to a useful application. We still need to add database configuration and ASP.NET Core identity stuff. We're going to be using the [FastEndpoints library](https://fast-endpoints.com/) in this lab, which has a nice feature for automatically discovering endpoints in any associated assembly/project so we won't need to add manual lines to the web host to wire up endpoints from every module.

## 1. Add More Packages

Run the following from the solution root to add the necessary NuGet packages:

```bash
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj package FastEndpoints
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj package FastEndpoints.Security
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj package FastEndpoints.Swagger
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj package Serilog.AspNetCore

dotnet add Nimble.Modulith.Users/Nimble.Modulith.Users.csproj package FastEndpoints
dotnet add Nimble.Modulith.Users/Nimble.Modulith.Users.csproj package FastEndpoints.Security
dotnet add Nimble.Modulith.Users/Nimble.Modulith.Users.csproj package Serilog.AspNetCore
```

## 2. Configure Logging in the Web Host

Open `Program.cs` in your Web project and add the following at the very top:

```csharp
using Nimble.Modulith.Users;
using Serilog;

var logger = Log.Logger = new LoggerConfiguration()
  .Enrich.FromLogContext()
  .WriteTo.Console()
  .CreateLogger();

logger.Information("Starting web host");

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog((_, config) => config.ReadFrom.Configuration(builder.Configuration));
```

Now if your immediately run the app you should see "Starting web host" logged to the console.

## 3. Configure FastEndpoints in the Web Host

Now in `Program.cs` again, add support for FastEndpoints servies after the call to Aspire ServiceDefaults:

```csharp
// Add service defaults (Aspire configuration)
builder.AddServiceDefaults();

// Add FastEndpoints with JWT Bearer Authentication and Authorization
builder.Services.AddFastEndpoints()
    .AddAuthenticationJwtBearer(s =>
    {
        s.SigningKey = builder.Configuration["Auth:JwtSecret"];
    })
    .AddAuthorization()
    .SwaggerDocument();
```

You'll need to add some using statements as well for `FastEndpoints.Security` and `FastEndpoints.Swagger`.

We're mostly going to be using cookies for simplicity in these labs but to get JWT working you'll need to add this to your `appsettings.json` or `appsettings.Development.json`:

```json
  "Auth": {
    "JwtSecret": "really really REALLY long secret key goes here"
  }
```

> **Note**: For development, consider using User Secrets instead of storing the JWT secret in appsettings.json. In production, use environment variables or Azure Key Vault.

Now map FastEndpoints in the request pipeline section, after the calls to auth:

```csharp
// Add authentication and authorization middleware
app.UseAuthentication();
app.UseAuthorization();

app.UseFastEndpoints()
    .UseSwaggerGen();
```

Now you should be able to run the application, but it will fail because there are no endpoints wired up!

```bash
[14:56:47 INF] Starting web host
Unhandled exception. System.InvalidOperationException: FastEndpoints was unable to find any endpoint declarations!
```

## 4. Add a Register Endpoint to the Users Module

In the `Nimble.Modulith.Users` project, add an `Endpoints` folder.

In the `Endpoints` folder, add a new class `Register.cs`.

The class should look like this:

```csharp
using FastEndpoints;
using System.ComponentModel.DataAnnotations;

namespace Nimble.Modulith.Users.Endpoints;

public class RegisterRequest
{
    [Required, EmailAddress]
    public string Email { get; set; } = string.Empty;
    
    [Required, MinLength(6)]
    public string Password { get; set; } = string.Empty;
}

public class RegisterResponse
{
    public string Message { get; set; } = string.Empty;
    public bool Success { get; set; }
    public string? UserId { get; set; }
    public List<string>? Errors { get; set; }
}

public class Register : Endpoint<RegisterRequest, RegisterResponse>
{
    public override void Configure()
    {
        Post("/register");
        AllowAnonymous();
        Summary(s => {
            s.Summary = "Register a new user";
            s.Description = "Creates a new user account";
        });
    }

    public override async Task HandleAsync(RegisterRequest req, CancellationToken ct)
    {
        // For now, just return OK - we'll wire up Identity later
        Response = new RegisterResponse
        {
            Success = true,
            Message = "Registration endpoint is working! (Not yet connected to Identity)"
        };

        await Send.OkAsync(Response, ct);
    }
}
```

Now we should be able to run the application and test it by navigating to the /register endpoint (which youc an also find from /swagger).

### 5. Create the UsersDbContext

In the Users project, create a `Data` folder and add `UsersDbContext.cs`:

```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace Nimble.Modulith.Users.Data;

public class UsersDbContext : IdentityDbContext<IdentityUser>
{
    public UsersDbContext(DbContextOptions<UsersDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Apply Users module specific configurations
        builder.HasDefaultSchema("Users");
    }
}
```

## 6. Add ASP.NET Core Identity and Wire Up From Web Host

We're going to need a database in which to store users, so we'll start by adding that to the Aspire `AppHost.cs` file in the AppHost project. The whole file should look like this:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add SQL Server database for Identity
var sqlServer = builder.AddSqlServer("sqlserver")
    .WithDataVolume();

var usersDb = sqlServer.AddDatabase("usersdb");

// Add the Web API project with database reference
builder.AddProject<Projects.Nimble_Modulith_Web>("webapi")
    .WithReference(usersDb)
    .WaitFor(usersDb);

builder.Build().Run();
```

We'll also need to make sure it has the right references; run this from the solution file folder:

```bash
dotnet add Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj package Aspire.Hosting.SqlServer
```

Note that this will set an environment variable with the a connection string for "usersdb" that will be referenced in the Users module.

Next, we are going to use ASP.NET Core Identity but we want all of the dependency configuration to happen in the Users module. To make this work, each module will expose an extension method for registering its dependencies, and the Web Host will call these extension methods in its Program.cs file.

In the Users project, add a new root level class called `UsersModuleExtensions.cs`. Add the following to it:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Nimble.Modulith.Users.Data;
using Serilog;

namespace Nimble.Modulith.Users;

public static class UsersModuleExtensions
{
    public static IHostApplicationBuilder AddUsersModuleServices(
        this IHostApplicationBuilder builder,
        ILogger logger)
    {
        // Add SQL Server DbContext with Aspire
        builder.AddSqlServerDbContext<UsersDbContext>("usersdb");
       

        // Add Authentication
        builder.Services.AddAuthentication().AddBearerToken(IdentityConstants.BearerScheme);
        builder.Services.AddAuthorizationBuilder();

        // Add Identity with SignInManager support
        builder.Services.AddIdentity<IdentityUser, IdentityRole>(options =>
        {
            // Configure Identity options
            options.Password.RequiredLength = 6;
            options.Password.RequireNonAlphanumeric = false;
            options.Password.RequireDigit = false;
            options.Password.RequireUppercase = false;
            options.Password.RequireLowercase = false;
            
            options.User.RequireUniqueEmail = true;
        })
        .AddEntityFrameworkStores<UsersDbContext>()
        .AddDefaultTokenProviders();

        logger.Information("{Module} module services registered", nameof(UsersModuleExtensions).Replace("ModuleExtensions", ""));

        return builder;
    }

    public static async Task<WebApplication> EnsureUsersModuleDatabaseAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<UsersDbContext>();
        await context.Database.EnsureCreatedAsync();
        
        return app;
    }
}
```

We will also need to add a few NuGet packages. Look at the `Nimble.Modulith.Users.csproj` file and ensure it looks like this:

```xml
<Project Sdk="Microsoft.NET.Sdk">

	<ItemGroup>
		<FrameworkReference Include="Microsoft.AspNetCore.App" />

		<PackageReference Include="FastEndpoints" />
		<PackageReference Include="FastEndpoints.Security" />
		<PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" />
		<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
		<PackageReference Include="Microsoft.EntityFrameworkCore.Design" />
		<PackageReference Include="Aspire.Microsoft.EntityFrameworkCore.SqlServer" />
		<PackageReference Include="Serilog.AspNetCore" />
	</ItemGroup>

	<ItemGroup>
		<ProjectReference Include="..\Nimble.Modulith.Users.Contracts\Nimble.Modulith.Users.Contracts.csproj" />
	</ItemGroup>
</Project>
```

Now go back to the web host `Program.cs` and add calls to these two extension methods.

First, right before you build the app, add the module registration call:

```csharp
// Add module services
builder.AddUsersModuleServices(logger);

var app = builder.Build();
```

(you can remove the OpenApi call if one is there)


Second, add the call to `EnsureUsersModuleDatabaseAsync` after the app has started:

```csharp

// Ensure module databases are created
await app.EnsureUsersModuleDatabaseAsync();

app.Run();

```

## 7. Remove Weather Forecast Stuff

We no longer need the Weather Forecast stuff in Program.cs. Remove these lines:

```csharp
var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", () =>
{
    var forecast =  Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast");



record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}

```

## 8. Add User Endpoints

Now we have all of the dependencies set up; let's update the Register endpoint and add other endpoints for Login, Logout, and Profile.

### Register Endpoint

Update the Register endpoint you created above so it uses UserManager to create a user:

```csharp
public class Register(UserManager<IdentityUser> userManager) : 
    Endpoint<RegisterRequest, RegisterResponse>
{
    private readonly UserManager<IdentityUser> _userManager = userManager;

    public override void Configure()
    {
        Post("/register");
        AllowAnonymous();
        Summary(s => {
            s.Summary = "Register a new user";
            s.Description = "Creates a new user account";
        });
    }

    public override async Task HandleAsync(RegisterRequest req, CancellationToken ct)
    {
        var user = new IdentityUser 
        { 
            UserName = req.Email,
            Email = req.Email 
        };
        
        var result = await _userManager.CreateAsync(user, req.Password);
        
        if (result.Succeeded)
        {
            Response = new RegisterResponse
            {
                Success = true,
                Message = "User created successfully",
                UserId = user.Id
            };
            
            await Send.OkAsync(Response, ct);
        }
        else
        {
            AddError("User registration failed");
            foreach (var error in result.Errors)
            {
                AddError(error.Description);
            }

            await Send.ErrorsAsync(cancellation: ct);
        }
    }
}
```

### Login Endpoint

Now that users can register, the next step is to let them log in. Add `Login.cs` to the Endpoints folder in Users.

```csharp
using FastEndpoints;
using Microsoft.AspNetCore.Identity;
using System.ComponentModel.DataAnnotations;

namespace Nimble.Modulith.Users.Endpoints;

public class LoginRequest
{
    [Required, EmailAddress]
    public string Email { get; set; } = string.Empty;
    
    [Required]
    public string Password { get; set; } = string.Empty;
}

public class LoginResponse
{
    public string Message { get; set; } = string.Empty;
    public bool Success { get; set; }
    public string? Token { get; set; }
}

public class Login(SignInManager<IdentityUser> signInManager) : 
    Endpoint<LoginRequest, LoginResponse>
{
    private readonly SignInManager<IdentityUser> _signInManager = signInManager;

    public override void Configure()
    {
        Post("/login");
        AllowAnonymous();
        Summary(s => {
            s.Summary = "Login with email and password";
            s.Description = "Authenticates a user and returns a token";
        });
    }

    public override async Task HandleAsync(LoginRequest req, CancellationToken ct)
    {       
        var result = await _signInManager.PasswordSignInAsync(
            req.Email, 
            req.Password, 
            isPersistent: false, 
            lockoutOnFailure: false);
        
        if (result.Succeeded)
        {
            Response = new LoginResponse
            {
                Success = true,
                Message = "Login successful",
                Token = "TODO: Generate JWT token" // We'll implement JWT later
            };
            
            await Send.OkAsync(Response, ct);
        }
        else
        {
            await Send.UnauthorizedAsync(ct);
        }
    }
}
```

### Logout Endpoint

Now add a `Logout.cs` file.

```csharp
using FastEndpoints;
using Microsoft.AspNetCore.Identity;

namespace Nimble.Modulith.Users.Endpoints;

public class LogoutResponse
{
    public string Message { get; set; } = string.Empty;
    public bool Success { get; set; }
}

public class Logout(SignInManager<IdentityUser> signInManager) : EndpointWithoutRequest<LogoutResponse>
{
    private readonly SignInManager<IdentityUser> _signInManager = signInManager;

    public override void Configure()
    {
        Post("/logout");
        AllowAnonymous(); // For now, we'll require auth later when we implement JWT properly
        Summary(s => {
            s.Summary = "Logout the current user";
            s.Description = "Signs out the current user";
        });
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        await _signInManager.SignOutAsync();
        
        Response = new LogoutResponse
        {
            Success = true,
            Message = "Logged out successfully"
        };
        
        await Send.OkAsync(Response, ct);
    }
}
```

### Test it out

You can test these endpoints now by running the whole application and using the `Nimble.Modulith.Web.http` file in the web project, or navigate to `/swagger` if you prefer. Update the .http file like so:

```
@Nimble.Modulith.Web_HostAddress = http://localhost:5002

### Register a new user

POST {{Nimble.Modulith.Web_HostAddress}}/register
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "Test123!"
}

### Login - Produces a Cookie for authenticated requests

POST {{Nimble.Modulith.Web_HostAddress}}/login
Content-Type: application/json
{
  "email": "test@example.com",
  "password": "Test123!"
}

### Logout

POST {{Nimble.Modulith.Web_HostAddress}}/logout

### View Profile (if logged in)

GET {{Nimble.Modulith.Web_HostAddress}}/profile

###

```

## Summary

We now have a working solution with a single module that supports registration, login, logout, and user profile access. Note that the web project has no endpoints and delegates all of its logic to the Users module.

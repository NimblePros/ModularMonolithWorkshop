# Lab 5: Add an Email-Sending Module and Fix Up Order Creation

## Overview

In this lab, we'll address several security and architectural concerns that exist after completing Lab 4. We'll also add a new Email module to support password management features.

## Issues to Address

After completing Lab 4, the following concerns need to be addressed:

### 1. Product Management Security
**Current Issue:** Any user can manage products.

**Required Fix:** Implement Role-based authorization so only Admins can manage products (create, update, delete).

### 2. Customer and Order Management Security
**Current Issue:** Any user can create Customers and Orders for any customer.

**Required Fix:** Implement authorization rules:
- Only Admins should be able to create Customers or Orders for other users
- Non-Admin users should be able to:
  - Create a Customer for themselves (if one doesn't exist)
  - Add Orders only to their own Customer record

### 3. Order Pricing Architecture
**Current Issue:** Orders take the price as part of the endpoint/command, allowing clients to set arbitrary prices.

**Required Fix:** Orders should pull the current price from the Products module instead of accepting it from the client. This ensures pricing integrity and prevents price manipulation.

### 4. Customer Password Generation
**Current Issue:** Customer creation doesn't handle password generation or integration with the Users module in any fashion.

**Required Fix:** Newly created customers should receive a randomly generated secure password.

### 5. Password Reset Functionality
**Current Issue:** No password reset mechanism exists. Users created to represent new Customer accounts need to have a way to have their passwords sent to them securely.

**Required Fix:** Users should be able to reset their password, with a new password emailed to them.

## Lab Steps

### Step 1: Create the Email Module

The Email module uses an asynchronous architecture where commands are queued and processed by a background worker. This ensures email sending doesn't block request processing.

#### 1.1: Create the Projects

Create two new projects:

```powershell
dotnet new classlib -n Nimble.Modulith.Email -f net10.0
dotnet new classlib -n Nimble.Modulith.Email.Contracts -f net10.0
dotnet sln add Nimble.Modulith.Email/Nimble.Modulith.Email.csproj
dotnet sln add Nimble.Modulith.Email.Contracts/Nimble.Modulith.Email.Contracts.csproj
```

Add the MailKit package to `Directory.Packages.props`:

```xml
<PackageVersion Include="MailKit" Version="4.14.0" />
```

Update `Nimble.Modulith.Email.csproj` to add package references:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\Nimble.Modulith.Email.Contracts\Nimble.Modulith.Email.Contracts.csproj" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="MailKit" />
    <PackageReference Include="Mediator.Abstractions" />
  </ItemGroup>

  <ItemGroup>
    <FrameworkReference Include="Microsoft.AspNetCore.App" />
  </ItemGroup>

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

Add Mediator.Abstractions to `Nimble.Modulith.Email.Contracts.csproj`:

```powershell
cd Nimble.Modulith.Email.Contracts
dotnet add package Mediator.Abstractions
```

#### 1.2: Create the Contracts

In `Nimble.Modulith.Email.Contracts`, create `SendEmailCommand.cs`:

```csharp
using Mediator;

namespace Nimble.Modulith.Email.Contracts;

public record SendEmailCommand(
  string To,
  string Subject,
  string Body,
  string? From = null) : ICommand;
```

Delete the default `Class1.cs` file.

#### 1.3: Create the Email Module Structure

Create the following folder structure in `Nimble.Modulith.Email`:
- `/Interfaces` - For service interfaces
- `/Services` - For service implementations
- `/Integrations` - For handlers that respond to messages from other modules

#### 1.4: Implement the Queue Service

In `/Interfaces/IQueueService.cs`:

```csharp
namespace Nimble.Modulith.Email.Interfaces;

public interface IQueueService<T>
{
  ValueTask EnqueueAsync(T item, CancellationToken cancellationToken = default);
  ValueTask<T> DequeueAsync(CancellationToken cancellationToken = default);
}
```

In `/Services/ChannelQueueService.cs`:

```csharp
using System.Threading.Channels;
using Nimble.Modulith.Email.Interfaces;

namespace Nimble.Modulith.Email.Services;

public class ChannelQueueService<T> : IQueueService<T>
{
  private readonly Channel<T> _channel;

  public ChannelQueueService()
  {
    // Create an unbounded channel for simplicity
    _channel = Channel.CreateUnbounded<T>();
  }

  public async ValueTask EnqueueAsync(T item, CancellationToken cancellationToken = default)
  {
    await _channel.Writer.WriteAsync(item, cancellationToken);
  }

  public async ValueTask<T> DequeueAsync(CancellationToken cancellationToken = default)
  {
    return await _channel.Reader.ReadAsync(cancellationToken);
  }
}
```

#### 1.5: Implement Email Sending

In the root of the Email project, create `EmailMessage.cs`:

```csharp
namespace Nimble.Modulith.Email;

public record EmailMessage(
  string To,
  string Subject,
  string Body,
  string? From = null);
```

Create `IEmailSender.cs`:

```csharp
namespace Nimble.Modulith.Email;

public interface IEmailSender
{
  Task SendEmailAsync(EmailMessage message, CancellationToken cancellationToken = default);
}
```

Create `EmailSettings.cs`:

```csharp
namespace Nimble.Modulith.Email;

public class EmailSettings
{
  public string SmtpServer { get; set; } = string.Empty;
  public int SmtpPort { get; set; } = 587;
  public bool EnableSsl { get; set; } = true;
  public string Username { get; set; } = string.Empty;
  public string Password { get; set; } = string.Empty;
  public string DefaultFromAddress { get; set; } = "noreply@nimblemodulith.com";
}
```

Create `SmtpEmailSender.cs`:

```csharp
using MailKit.Net.Smtp;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Options;
using MimeKit;

namespace Nimble.Modulith.Email;

public class SmtpEmailSender : IEmailSender
{
  private readonly ILogger<SmtpEmailSender> _logger;
  private readonly EmailSettings _settings;

  public SmtpEmailSender(
    ILogger<SmtpEmailSender> logger,
    IOptions<EmailSettings> settings)
  {
    _logger = logger;
    _settings = settings.Value;
  }

  public async Task SendEmailAsync(EmailMessage message, CancellationToken cancellationToken = default)
  {
    try
    {
      var mimeMessage = new MimeMessage();
      mimeMessage.From.Add(MailboxAddress.Parse(message.From ?? _settings.DefaultFromAddress));
      mimeMessage.To.Add(MailboxAddress.Parse(message.To));
      mimeMessage.Subject = message.Subject;
      mimeMessage.Body = new TextPart("plain")
      {
        Text = message.Body
      };

      using var client = new SmtpClient();
      await client.ConnectAsync(_settings.SmtpServer, _settings.SmtpPort, _settings.EnableSsl, cancellationToken);
      
      if (!string.IsNullOrEmpty(_settings.Username))
      {
        await client.AuthenticateAsync(_settings.Username, _settings.Password, cancellationToken);
      }

      await client.SendAsync(mimeMessage, cancellationToken);
      await client.DisconnectAsync(true, cancellationToken);

      _logger.LogInformation(
        "Email sent successfully to {To} with subject: {Subject}",
        message.To,
        message.Subject);
    }
    catch (Exception ex)
    {
      _logger.LogError(ex, "Failed to send email to {To}", message.To);
      throw;
    }
  }
}
```

#### 1.6: Implement the Command Handler

In `/Integrations/EmailToSend.cs`:

```csharp
namespace Nimble.Modulith.Email.Integrations;

public record EmailToSend(
  string To,
  string Subject,
  string Body,
  string? From = null);
```

In `/Integrations/SendEmailCommandHandler.cs`:

```csharp
using Mediator;
using Nimble.Modulith.Email.Contracts;
using Nimble.Modulith.Email.Interfaces;

namespace Nimble.Modulith.Email.Integrations;

public class SendEmailCommandHandler : ICommandHandler<SendEmailCommand>
{
  private readonly IQueueService<EmailToSend> _queueService;

  public SendEmailCommandHandler(IQueueService<EmailToSend> queueService)
  {
    _queueService = queueService;
  }

  public async ValueTask<Unit> Handle(SendEmailCommand command, CancellationToken cancellationToken)
  {
    var emailToSend = new EmailToSend(
      command.To,
      command.Subject,
      command.Body,
      command.From);

    await _queueService.EnqueueAsync(emailToSend, cancellationToken);

    return Unit.Value;
  }
}
```

#### 1.7: Implement the Background Worker

In the root, create `EmailSendingBackgroundWorker.cs`:

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using Nimble.Modulith.Email.Integrations;
using Nimble.Modulith.Email.Interfaces;

namespace Nimble.Modulith.Email;

public class EmailSendingBackgroundWorker : BackgroundService
{
  private readonly IQueueService<EmailToSend> _queueService;
  private readonly IEmailSender _emailSender;
  private readonly ILogger<EmailSendingBackgroundWorker> _logger;

  public EmailSendingBackgroundWorker(
    IQueueService<EmailToSend> queueService,
    IEmailSender emailSender,
    ILogger<EmailSendingBackgroundWorker> logger)
  {
    _queueService = queueService;
    _emailSender = emailSender;
    _logger = logger;
  }

  protected override async Task ExecuteAsync(CancellationToken stoppingToken)
  {
    _logger.LogInformation("Email sending background worker started");

    while (!stoppingToken.IsCancellationRequested)
    {
      try
      {
        // Process all available emails before sleeping
        bool processedAny = false;
        
        while (true)
        {
          try
          {
            // Try to dequeue with a short timeout to check if queue is empty
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(stoppingToken);
            cts.CancelAfter(TimeSpan.FromMilliseconds(100));
            
            var emailToSend = await _queueService.DequeueAsync(cts.Token);
            
            _logger.LogInformation("Processing email to {To}", emailToSend.To);
            
            var message = new EmailMessage(
              emailToSend.To,
              emailToSend.Subject,
              emailToSend.Body,
              emailToSend.From);

            await _emailSender.SendEmailAsync(message, stoppingToken);
            
            processedAny = true;
          }
          catch (OperationCanceledException)
          {
            // No more messages in queue, break inner loop
            break;
          }
        }

        if (processedAny)
        {
          _logger.LogInformation("Finished processing batch of emails");
        }

        // Sleep for 1 second before checking again
        await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
      }
      catch (Exception ex) when (ex is not OperationCanceledException)
      {
        _logger.LogError(ex, "Error processing email queue");
        await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
      }
    }

    _logger.LogInformation("Email sending background worker stopped");
  }
}
```

#### 1.8: Create Module Extensions

In the root, create `EmailModuleExtensions.cs`:

```csharp
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Nimble.Modulith.Email.Integrations;
using Nimble.Modulith.Email.Interfaces;
using Nimble.Modulith.Email.Services;

namespace Nimble.Modulith.Email;

public static class EmailModuleExtensions
{
  public static IServiceCollection AddEmailModule(
    this IServiceCollection services,
    IConfiguration configuration)
  {
    // Configure email settings
    services.Configure<EmailSettings>(configuration.GetSection("Email"));

    // Register email sender as singleton (thread-safe, no scoped dependencies)
    services.AddSingleton<IEmailSender, SmtpEmailSender>();

    // Register queue service as singleton (shared across all requests)
    services.AddSingleton(typeof(IQueueService<>), typeof(ChannelQueueService<>));

    // Register the command handler
    services.AddScoped<SendEmailCommandHandler>();

    // Register the background worker
    services.AddHostedService<EmailSendingBackgroundWorker>();

    return services;
  }
}
```

#### 1.9: Register the Email Module

Add a reference to the Email project in `Nimble.Modulith.Web.csproj`:

```powershell
cd Nimble.Modulith.Web
dotnet add reference ../Nimble.Modulith.Email/Nimble.Modulith.Email.csproj
```

Update `Program.cs` to register the Email module:

```csharp
using Nimble.Modulith.Email;
// ... other usings

// Add module services
builder.AddUsersModuleServices(logger);
builder.AddProductsModuleServices(logger);
builder.AddCustomersModuleServices(logger);
builder.Services.AddEmailModule(builder.Configuration);
```

#### 1.10: Configure Email Settings

Add email configuration to `appsettings.json` and `appsettings.Development.json`:

```json
{
  "Email": {
    "SmtpServer": "localhost",
    "SmtpPort": 25,
    "EnableSsl": false,
    "Username": "",
    "Password": "",
    "DefaultFromAddress": "noreply@nimblemodulith.com"
  }
}
```

#### 1.11: Add Papercut Docker Container for Email Testing

Papercut is a simple SMTP server that captures emails sent during development, allowing you to view them in a web UI without actually sending emails.

Update `Nimble.Modulith.AppHost/AppHost.cs` to add the Papercut container:

```csharp
using System.Net.Sockets;

var builder = DistributedApplication.CreateBuilder(args);

// Add SQL Server database for Identity
var sqlServer = builder.AddSqlServer("sqlserver")
    .WithDataVolume();

var usersDb = sqlServer.AddDatabase("usersdb");
var productsDb = sqlServer.AddDatabase("productsdb");
var customersDb = sqlServer.AddDatabase("customersdb");

// Papercut SMTP container for email testing
var papercut = builder.AddContainer("papercut", "jijiechen/papercut", "latest")
  .WithEndpoint("smtp", e =>
  {
    e.TargetPort = 25;   // container port
    e.Port = 25;         // host port
    e.Protocol = ProtocolType.Tcp;
    e.UriScheme = "smtp";
  })
  .WithEndpoint("ui", e =>
  {
    e.TargetPort = 37408;
    e.Port = 37408;
    e.UriScheme = "http";
  });

// Add the Web API project with database and Papercut references
var webapi = builder.AddProject<Projects.Nimble_Modulith_Web>("webapi")
    .WithReference(usersDb)
    .WithReference(productsDb)
    .WithReference(customersDb)
    .WithEnvironment("Papercut__Smtp__Url", papercut.GetEndpoint("smtp"))
    .WithEnvironment("Papercut__Ui__Url", papercut.GetEndpoint("ui"))
    .WaitFor(usersDb)
    .WaitFor(productsDb)
    .WaitFor(customersDb)
    .WaitFor(papercut);

builder.Build().Run();
```

**What this does:**

- Creates a Papercut container from the `jijiechen/papercut:latest` Docker image
- Exposes SMTP on port 25 for sending emails
- Exposes the web UI on port 37408 for viewing captured emails
- Passes Papercut endpoint URLs to the web app via environment variables
- Ensures the web app waits for Papercut to start before launching

**Accessing Papercut:**

When you run the Aspire AppHost, you can access the Papercut UI at `http://localhost:37408` to view all emails sent by the application during development.

#### 1.12: Verify the Implementation

Build the solution to ensure everything compiles:

```powershell
dotnet build
```

The Email module is now complete and ready to handle email sending asynchronously. Other modules can send emails by dispatching `SendEmailCommand` via Mediator, and the background worker will process and send them via SMTP to Papercut.

### Step 2: Implement Role-Based Authorization

In this step, we'll add role support to the Users module, create an endpoint to assign users to roles, and use domain events to send email notifications when users are added to roles.

#### 2.1: Seed Roles in the Database

We'll use Entity Framework's `IEntityTypeConfiguration<T>` pattern to configure role seeding in a separate configuration file. This keeps our DbContext clean and makes configurations reusable.

Create `Nimble.Modulith.Users/Data/Config/IdentityRoleConfig.cs`:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Nimble.Modulith.Users.Data.Config;

public class IdentityRoleConfig : IEntityTypeConfiguration<IdentityRole>
{
  public void Configure(EntityTypeBuilder<IdentityRole> builder)
  {
    // Seed Admin and Customer roles
    builder.HasData(
      new IdentityRole
      {
        Id = Guid.NewGuid().ToString(),
        Name = "Admin",
        NormalizedName = "ADMIN",
        ConcurrencyStamp = Guid.NewGuid().ToString()
      },
      new IdentityRole
      {
        Id = Guid.NewGuid().ToString(),
        Name = "Customer",
        NormalizedName = "CUSTOMER",
        ConcurrencyStamp = Guid.NewGuid().ToString()
      }
    );
  }
}
```

Update `Nimble.Modulith.Users/Data/UsersDbContext.cs` to auto-discover all entity configurations:

```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
  base.OnModelCreating(builder);
  
  // Apply Users module specific configurations
  builder.HasDefaultSchema("Users");
  
  // Auto-discover and apply all IEntityTypeConfiguration<T> from this assembly
  builder.ApplyConfigurationsFromAssembly(typeof(UsersDbContext).Assembly);
}
```

**Key Points:**
- **Separation of Concerns:** Configuration is separated from the DbContext into dedicated files
- **Auto-Discovery:** `ApplyConfigurationsFromAssembly` automatically finds and applies all `IEntityTypeConfiguration<T>` implementations
- **Reference by Name:** Other modules reference roles by name ("Admin", "Customer") rather than by ID
- **Generated IDs:** Role IDs are generated automatically; only the names are important
- **Scalability:** As you add more entity configurations, they're automatically picked up

#### 2.2: Create a Design-Time DbContext Factory

EF Core migrations require a way to create the DbContext at design time. Create `Nimble.Modulith.Users/Data/UsersDbContextFactory.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace Nimble.Modulith.Users.Data;

public class UsersDbContextFactory : IDesignTimeDbContextFactory<UsersDbContext>
{
  public UsersDbContext CreateDbContext(string[] args)
  {
    var optionsBuilder = new DbContextOptionsBuilder<UsersDbContext>();
    optionsBuilder.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=NimbleModulith;Trusted_Connection=True;MultipleActiveResultSets=true");

    return new UsersDbContext(optionsBuilder.Options);
  }
}
```

#### 2.3: Create the Migration

Generate an EF Core migration to add the role seeding:

```powershell
cd Nimble.Modulith.Users
dotnet ef migrations add SeedRoles --context UsersDbContext --output-dir Data/Migrations
```

This creates a migration that will insert the Admin and Customer roles when the database is updated.

#### 2.4: Add a Reference to Email.Contracts

The Users module needs to send email commands, so add a reference to the Email.Contracts project:

```powershell
cd Nimble.Modulith.Users
dotnet add reference ../Nimble.Modulith.Email.Contracts/Nimble.Modulith.Email.Contracts.csproj
```

#### 2.5: Create the Domain Event

Domain events allow modules to communicate without tight coupling. Create `Nimble.Modulith.Users/Events/UserAddedToRoleEvent.cs`:

```csharp
using Mediator;

namespace Nimble.Modulith.Users.Events;

public record UserAddedToRoleEvent(
  string UserId,
  string UserEmail,
  string RoleName) : INotification;
```

**Architecture Note:** This event implements `INotification` from Mediator, which allows multiple handlers to respond to it.

#### 2.6: Create the Event Handler

Create `Nimble.Modulith.Users/Events/UserAddedToRoleEventHandler.cs` to send an email when a user is added to a role:

```csharp
using Mediator;
using Microsoft.Extensions.Logging;
using Nimble.Modulith.Email.Contracts;

namespace Nimble.Modulith.Users.Events;

public class UserAddedToRoleEventHandler : INotificationHandler<UserAddedToRoleEvent>
{
  private readonly IMediator _mediator;
  private readonly ILogger<UserAddedToRoleEventHandler> _logger;

  public UserAddedToRoleEventHandler(
    IMediator mediator,
    ILogger<UserAddedToRoleEventHandler> logger)
  {
    _mediator = mediator;
    _logger = logger;
  }

  public async ValueTask Handle(UserAddedToRoleEvent notification, CancellationToken cancellationToken)
  {
    _logger.LogInformation(
      "User {UserId} was added to role {RoleName}, sending email notification",
      notification.UserId,
      notification.RoleName);

    var emailCommand = new SendEmailCommand(
      To: notification.UserEmail,
      Subject: $"You've been added to the {notification.RoleName} role",
      Body: $"Hello,\n\nYou have been added to the {notification.RoleName} role in the Nimble Modulith application.\n\nBest regards,\nThe Nimble Team"
    );

    await _mediator.Send(emailCommand, cancellationToken);

    _logger.LogInformation("Email notification sent to {Email}", notification.UserEmail);
  }
}
```

#### 2.7: Create the AddRoleToUser Endpoint

Create `Nimble.Modulith.Users/Endpoints/AddRoleToUser.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Microsoft.AspNetCore.Identity;
using Nimble.Modulith.Users.Events;

namespace Nimble.Modulith.Users.Endpoints;

public class AddRoleToUserRequest
{
  public string RoleName { get; set; } = string.Empty;
}

public class AddRoleToUserResponse
{
  public string Message { get; set; } = string.Empty;
}

public class AddRoleToUser : Endpoint<AddRoleToUserRequest, AddRoleToUserResponse>
{
  private readonly UserManager<IdentityUser> _userManager;
  private readonly RoleManager<IdentityRole> _roleManager;
  private readonly IMediator _mediator;

  public AddRoleToUser(
    UserManager<IdentityUser> userManager,
    RoleManager<IdentityRole> roleManager,
    IMediator mediator)
  {
    _userManager = userManager;
    _roleManager = roleManager;
    _mediator = mediator;
  }

  public override void Configure()
  {
    Post("/users/{id}/roles");
    AllowAnonymous(); // TODO: Change to require Admin role
  }

  public override async Task HandleAsync(AddRoleToUserRequest req, CancellationToken ct)
  {
    var userId = Route<string>("id")!;

    // Find the user
    var user = await _userManager.FindByIdAsync(userId);
    if (user == null)
    {
      AddError($"User with ID '{userId}' not found");
      await Send.ErrorsAsync(cancellation: ct);
      return;
    }

    // Normalize the role name (handle "admin" -> "Admin")
    var normalizedRoleName = char.ToUpper(req.RoleName[0]) + req.RoleName.Substring(1).ToLower();

    // Check if the role exists (only Admin and Customer are valid)
    if (normalizedRoleName != "Admin" && normalizedRoleName != "Customer")
    {
      AddError($"Role '{normalizedRoleName}' does not exist. Valid roles are: Admin, Customer");
      await Send.ErrorsAsync(cancellation: ct);
      return;
    }

    // Check if user is already in the role
    if (await _userManager.IsInRoleAsync(user, normalizedRoleName))
    {
      AddError($"User is already in the '{normalizedRoleName}' role");
      await Send.ErrorsAsync(cancellation: ct);
      return;
    }

    // Add the user to the role
    var result = await _userManager.AddToRoleAsync(user, normalizedRoleName);
    if (!result.Succeeded)
    {
      foreach (var error in result.Errors)
      {
        AddError(error.Description);
      }
      await Send.ErrorsAsync(cancellation: ct);
      return;
    }

    // Publish domain event for email notification
    var userAddedEvent = new UserAddedToRoleEvent(
      UserId: user.Id,
      UserEmail: user.Email!,
      RoleName: normalizedRoleName
    );

    await _mediator.Publish(userAddedEvent, ct);

    // Send success response
    Response.Message = $"User '{user.Email}' successfully added to role '{normalizedRoleName}'";
  }
}
```

**Key Implementation Details:**

1. **Role Normalization:** Handles case-insensitive role names (e.g., "admin" → "Admin")
2. **Validation:** Checks if user exists, role is valid, and user isn't already in the role
3. **Error Handling:** Uses FastEndpoints pattern: `AddError()` followed by `Send.ErrorsAsync()`
4. **Domain Event:** Publishes `UserAddedToRoleEvent` after successful role assignment
5. **Async Flow:** Event → Handler → Email Command → Queue → Background Worker → SMTP

#### 2.8: Test the Implementation

1. **Start the application** via the Aspire AppHost:
   ```powershell
   cd Nimble.Modulith.AppHost
   dotnet run
   ```

2. **Register a new user** via POST to `/register`:
   ```json
   {
     "email": "testuser@example.com",
     "password": "Test123!"
   }
   ```

3. **Add the user to the Admin role** via POST to `/users/{userId}/roles`:
   ```json
   {
     "roleName": "admin"
   }
   ```

4. **Check Papercut UI** at `http://localhost:37408` to see the email notification

**Expected Email:**
- **To:** testuser@example.com
- **Subject:** You've been added to the Admin role
- **Body:** Notification message about role assignment

#### 2.9: Understanding the Domain Event Pattern

The architecture we've implemented demonstrates several important patterns:

**1. Domain Events for Loose Coupling:**
- The `AddRoleToUser` endpoint doesn't know about email sending
- It publishes an event describing what happened
- Any number of handlers can respond to the event independently

**2. Mediator Auto-Discovery:**
- Handlers implementing `INotificationHandler<T>` are automatically discovered
- No manual registration required in `UsersModuleExtensions`
- Simplifies adding new event handlers

**3. Async Email Processing:**
- Event handler sends `SendEmailCommand`
- Command handler enqueues email to `Channel<T>` queue
- Background worker dequeues and sends via SMTP
- Request completes immediately without waiting for email delivery

**4. Cross-Module Communication:**
- Users module references only `Email.Contracts` (not Email implementation)
- Maintains module boundaries while enabling communication
- Email module implementation can change without affecting Users module

#### 2.10: Next Steps

The remaining items in Step 2:
4. Update Product endpoints to require Admin role
5. Update Customer and Order endpoints with appropriate authorization

These will be completed in subsequent parts of the lab.

### Step 3: Fix Order Pricing

1. Create a contract in Products.Contracts for getting product pricing
2. Implement a handler in the Products module to return product prices
3. Update the Order creation logic to fetch prices from Products module
4. Remove price parameters from Order-related endpoints and commands
5. Add an endpoint to confirm an Order; it should change the Order status to Processing.
5. Email the user when their Order has been placed/confirmed (send command to email module)
6. Raise an OrderCreatedEvent, which should be defined in the Customers.Contracts project, and should include details on the Order and all of its Items (Product Ids, Quantities, and Amounts).

### Step 4: Implement Password Generation

1. Add a password generation utility
2. Update Customer creation to generate a random password
3. Store the password securely using Identity's password hasher
4. Send the generated password via email (command to email module)

### Step 5: Implement Password Reset

1. Create a password reset endpoint
2. Generate a new random password
3. Update the user's password
4. Email the new password to the user (command to email module)

## Success Criteria

After completing this lab, you should have:

- ✅ Role-based authorization protecting Products endpoints
- ✅ Proper authorization on Customer and Order endpoints
- ✅ Orders that fetch prices from the Products module
- ✅ Automatic password generation for new customers
- ✅ Password reset functionality with email notification
- ✅ A working Email module integrated with the application

## Notes

- For simplicity new passwords can just be a portion of a GUID
- In development, we will use Papercut to capture emails sent to localhost port 25
- In production, we would use a cloud-hosted email service
- A real app should have password reset tokens with expiration for better security

## Step 1: Create the Email Module


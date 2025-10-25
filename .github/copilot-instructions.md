# GitHub Copilot Instructions for ModularMonolithWorkshop

## Project Overview
This is a Modular Monolith workshop built with .NET 10 (preview), demonstrating clean architecture patterns with separated modules for Users, Products, Customers, and Email functionality.

## Technology Stack
- **.NET 10.0** (preview RC2)
- **FastEndpoints 7.1.0** - Minimal API alternative
- **Mediator 3.0.1** - CQRS with auto-discovery
- **Entity Framework Core 9.0.9/10.0.0** - Data access
- **ASP.NET Core Identity** - Authentication/Authorization
- **MailKit 4.14.0** - SMTP email sending
- **.NET Aspire 9.5.2** - Cloud-native orchestration

## Architecture Patterns

### Module Structure
Each module follows this pattern:
- `ModuleName/` - Implementation project
- `ModuleName.Contracts/` - Public contracts (commands, queries, DTOs)
- `Data/` - EF Core DbContext and migrations
- `Data/Config/` - Entity configurations using `IEntityTypeConfiguration<T>`
- `Endpoints/` - FastEndpoints endpoint definitions
- `Events/` - Domain events (INotification) and handlers

### Entity Framework Configuration
Use `IEntityTypeConfiguration<T>` for entity configuration in separate files:
```csharp
public class IdentityRoleConfig : IEntityTypeConfiguration<IdentityRole>
{
  public void Configure(EntityTypeBuilder<IdentityRole> builder)
  {
    // Configuration here
  }
}
```

Apply all configurations automatically in DbContext:
```csharp
protected override void OnModelCreating(ModelBuilder builder)
{
  base.OnModelCreating(builder);
  builder.ApplyConfigurationsFromAssembly(typeof(UsersDbContext).Assembly);
}
```

### Mediator Pattern
- **Auto-Discovery**: Handlers implementing `ICommandHandler<T>`, `IQueryHandler<T>`, or `INotificationHandler<T>` are automatically discovered
- **NO manual registration** required in module extensions
- Commands/Queries live in `.Contracts` projects
- Handlers live in implementation projects
- **Cross-Module Handlers**: Handlers for commands/queries/events from other modules' `.Contracts` projects should be placed in an `Integrations/` folder to clearly indicate they handle external contracts
- **Internal vs External Commands**: When a module needs to expose functionality to other modules:
  - Create a public command/query in `.Contracts` project (e.g., `CreateUserCommand`)
  - Create an internal command/query in `UseCases/` folder (e.g., `CreateUserInternalCommand`)
  - Implement the actual business logic in a `UseCases/` handler
  - Create an `Integrations/` handler for the `.Contracts` command that translates to the internal command and delegates via Mediator
  - This keeps internal implementation details separate from external contracts

### Domain Events
Use `INotification` from Mediator for domain events:
```csharp
public record UserAddedToRoleEvent(
  string UserId,
  string UserEmail,
  string RoleName) : INotification;
```

Publish events after business operations:
```csharp
await _mediator.Publish(userAddedEvent, ct);
```

## FastEndpoints API Patterns

### ⚠️ CRITICAL: FastEndpoints API Changes
FastEndpoints 7.x uses a different API than older versions. The `Send` property now has async methods.

### Error Handling Pattern
**Correct approach** - Accumulate errors, then send:
```csharp
if (user == null)
{
  AddError($"User with ID '{userId}' not found");
  await Send.ErrorsAsync(cancellation: ct);
  return;
}

// Multiple errors example
if (!result.Succeeded)
{
  foreach (var error in result.Errors)
  {
    AddError(error.Description);
  }
  await Send.ErrorsAsync(cancellation: ct);
  return;
}
```

**Key points**:
- Use `AddError()` to accumulate error messages
- Use `await Send.ErrorsAsync(cancellation: ct)` to send all errors
- Always include `return;` after sending errors
- Pass the `CancellationToken` parameter as `cancellation: ct`

### Success Response Patterns
**Option 1** - Set Response property (auto sends 200 OK):
```csharp
Response.Message = $"User '{user.Email}' successfully added to role '{normalizedRoleName}'";
// FastEndpoints automatically serializes and sends the response
```

**Option 2** - Explicitly send OK response:
```csharp
await Send.OkAsync(cancellation: ct);
```

**Option 3** - Send with custom response:
```csharp
await Send.OkAsync(new MyResponse { Data = result }, cancellation: ct);
```

### NOT FOUND Pattern
```csharp
if (user == null)
{
  await Send.NotFoundAsync(ct);
  return;
}
```

### ❌ DO NOT USE (Old API)
- ~~`ThrowError()`~~ - Does not exist
- ~~`Send.BadRequestAsync()`~~ - Does not exist on ResponseSender
- ~~`SendAsync()`~~ - Use `Send.OkAsync()` instead

## Dependency Injection Patterns

### Service Lifetimes
- **Singleton**: Services with no scoped dependencies (IEmailSender, IQueueService, ILogger, IOptions)
- **Scoped**: Services with database access (DbContext, handlers)
- **Transient**: Rarely used; prefer scoped for most services

### Background Services
BackgroundService (IHostedService) is registered as singleton:
```csharp
services.AddHostedService<EmailSendingBackgroundWorker>();
```

If background service needs scoped services, inject `IServiceScopeFactory` and create scopes manually. However, if dependencies are singleton-safe (like IEmailSender with only ILogger and IOptions dependencies), register them as singleton for simpler code.

## Email Module Architecture

### Queue-Based Async Processing
```
Command → Handler → Queue → Background Worker → SMTP
```

1. `SendEmailCommand` dispatched via Mediator
2. `SendEmailCommandHandler` enqueues to `Channel<EmailToSend>`
3. `EmailSendingBackgroundWorker` dequeues and sends via `IEmailSender`
4. `SmtpEmailSender` sends via MailKit SMTP client

### Singleton Services
Both `IQueueService<T>` and `IEmailSender` are registered as singletons:
```csharp
services.AddSingleton<IEmailSender, SmtpEmailSender>();
services.AddSingleton(typeof(IQueueService<>), typeof(ChannelQueueService<>));
```

This works because:
- `SmtpEmailSender` only depends on `ILogger` and `IOptions<EmailSettings>` (both singleton-safe)
- `ChannelQueueService<T>` is thread-safe
- Creates `SmtpClient` per-send operation (not shared state)

## Role-Based Authorization

### Role Configuration
- Roles are seeded via EF Core `HasData()` in `IEntityTypeConfiguration<IdentityRole>`
- Use **generated GUIDs** for IDs (not hardcoded strings)
- Reference roles by **name** ("Admin", "Customer") not by ID
- `NormalizedName` must be uppercase ("ADMIN", "CUSTOMER")

### Role Normalization in Endpoints
```csharp
// Handle case-insensitive input: "admin" → "Admin"
var normalizedRoleName = char.ToUpper(req.RoleName[0]) + req.RoleName.Substring(1).ToLower();
```

## Testing with Papercut
- Papercut Docker container captures SMTP emails during development
- SMTP: `localhost:25`
- Web UI: `http://localhost:37408`
- Integrated via Aspire AppHost with `.WaitFor(papercut)`

## Common Patterns

### Cross-Module Communication
- Modules reference other modules' `.Contracts` projects (NOT implementation)
- Use Mediator commands/queries for synchronous communication
- Use domain events (INotification) for asynchronous/decoupled communication

### Endpoint Configuration
```csharp
public override void Configure()
{
  Post("/users/{id}/roles");
  AllowAnonymous(); // or specific authorization
}
```

### Route Parameters
```csharp
var userId = Route<string>("id")!;
```

### Request Handling Template
```csharp
public override async Task HandleAsync(MyRequest req, CancellationToken ct)
{
  // 1. Validate input
  if (/* validation fails */)
  {
    AddError("Error message");
    await Send.ErrorsAsync(cancellation: ct);
    return;
  }

  // 2. Perform business logic
  var result = await _service.DoSomethingAsync(req, ct);

  // 3. Publish domain events if needed
  await _mediator.Publish(new MyDomainEvent(...), ct);

  // 4. Return success response
  Response.MyProperty = result;
}
```

## References
- [FastEndpoints Documentation](https://fast-endpoints.com/)
- [Mediator Documentation](https://github.com/martinothamar/Mediator)
- [.NET Aspire Documentation](https://learn.microsoft.com/dotnet/aspire/)

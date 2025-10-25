# Lab 6: Fix Order Pricing and Implement Password Management

In this lab, we'll implement cross-module communication patterns and improve the order and customer creation workflows.

## Learning Objectives

- Use Mediator to query data from other modules
- Implement domain events for decoupled communication
- Generate and manage user passwords programmatically
- Send transactional emails using the Email module

## Step 1: Fix Order Pricing

Currently, order endpoints require clients to provide product prices, which is a security risk. Products should fetch their prices from the Products module.

### 1.1: Create GetProductPriceQuery Contract

In the Products.Contracts project, create a query that other modules can use to fetch product prices.

**`Nimble.Modulith.Products.Contracts/GetProductPriceQuery.cs`:**

```csharp
using Mediator;

namespace Nimble.Modulith.Products.Contracts;

public record GetProductPriceQuery(int ProductId) : IQuery<decimal>;
```

### 1.2: Implement GetProductPriceQuery Handler

In the Products module, create a handler to return product prices from the database.

**`Nimble.Modulith.Products/UseCases/Queries/GetProductPriceQueryHandler.cs`:**

```csharp
using Mediator;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Contracts;
using Nimble.Modulith.Products.Data;

namespace Nimble.Modulith.Products.UseCases.Queries;

public class GetProductPriceQueryHandler(ProductsDbContext dbContext)
    : IQueryHandler<GetProductPriceQuery, decimal>
{
    public async ValueTask<decimal> Handle(GetProductPriceQuery query, CancellationToken cancellationToken)
    {
        var product = await dbContext.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == query.ProductId, cancellationToken);

        if (product == null)
        {
            throw new InvalidOperationException($"Product with ID {query.ProductId} not found");
        }

        return product.Price;
    }
}
```

### 1.3: Add Products.Contracts Reference to Customers Module

Update the Customers project file to reference Products.Contracts:

**`Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj`:**

```xml
<ItemGroup>
  <ProjectReference Include="..\Nimble.Modulith.Customers.Contracts\Nimble.Modulith.Customers.Contracts.csproj" />
  <ProjectReference Include="..\Nimble.Modulith.Products.Contracts\Nimble.Modulith.Products.Contracts.csproj" />
  <ProjectReference Include="..\Nimble.Modulith.Email.Contracts\Nimble.Modulith.Email.Contracts.csproj" />
  <ProjectReference Include="..\Nimble.Modulith.Users.Contracts\Nimble.Modulith.Users.Contracts.csproj" />
</ItemGroup>
```

### 1.4: Remove UnitPrice from CreateOrderCommand

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/CreateOrderCommand.cs`:**

```csharp
public record CreateOrderItemDto(
    int ProductId,
    string ProductName,
    int Quantity
);  // UnitPrice removed
```

### 1.5: Update CreateOrderHandler to Fetch Prices

Modify the handler to query product prices from the Products module:

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/CreateOrderHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;
using Nimble.Modulith.Products.Contracts;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class CreateOrderHandler(
    IRepository<Order> orderRepository,
    IReadRepository<Customer> customerRepository,
    IMediator mediator) 
    : ICommandHandler<CreateOrderCommand, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        // Verify customer exists
        var customer = await customerRepository.GetByIdAsync(command.CustomerId, ct);
        if (customer is null)
        {
            return Result<OrderDto>.NotFound($"Customer with ID {command.CustomerId} not found");
        }

        // Create order entity
        var order = new Order
        {
            CustomerId = command.CustomerId,
            OrderNumber = $"ORD-{DateTime.UtcNow:yyyyMMddHHmmss}",
            OrderDate = command.OrderDate,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        // Add order items - fetch prices from Products module
        foreach (var itemDto in command.Items)
        {
            // Fetch the product price from the Products module
            decimal unitPrice;
            try
            {
                unitPrice = await mediator.Send(new GetProductPriceQuery(itemDto.ProductId), ct);
            }
            catch (InvalidOperationException ex)
            {
                return Result<OrderDto>.Error($"Failed to get price for product {itemDto.ProductId}: {ex.Message}");
            }

            var item = new OrderItem
            {
                ProductId = itemDto.ProductId,
                ProductName = itemDto.ProductName,
                Quantity = itemDto.Quantity,
                UnitPrice = unitPrice
            };
            order.AddItem(item);
        }

        await orderRepository.AddAsync(order, ct);
        await orderRepository.SaveChangesAsync(ct);

        // Map to DTO
        var dto = new OrderDto(
            order.Id,
            order.CustomerId,
            order.OrderNumber,
            order.OrderDate,
            order.Status.ToString(),
            order.TotalAmount,
            order.Items.Select(i => new OrderItemDto(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList(),
            order.CreatedAt,
            order.UpdatedAt
        );

        return Result<OrderDto>.Success(dto);
    }
}
```

### 1.6: Update Order Endpoint Request DTOs

Remove UnitPrice from the endpoint request DTOs:

**`Nimble.Modulith.Customers/Endpoints/Orders/OrderResponse.cs`:**

```csharp
public record CreateOrderItemRequest(
    int ProductId,
    string ProductName,
    int Quantity
);  // UnitPrice removed

public record AddOrderItemRequest(
    int ProductId,
    string ProductName,
    int Quantity
);  // UnitPrice removed
```

### 1.7: Update Create Endpoint

**`Nimble.Modulith.Customers/Endpoints/Orders/Create.cs`:**

```csharp
var command = new CreateOrderCommand(
    req.CustomerId,
    req.OrderDate,
    req.Items.Select(i => new CreateOrderItemDto(
        i.ProductId,
        i.ProductName,
        i.Quantity  // No UnitPrice
    )).ToList()
);
```

### 1.8: Update AddOrderItemCommand and Handler

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/AddOrderItemCommand.cs`:**

```csharp
public record AddOrderItemCommand(
    int OrderId,
    int ProductId,
    string ProductName,
    int Quantity
) : ICommand<Result<OrderDto>>;  // UnitPrice removed
```

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/AddOrderItemHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;
using Nimble.Modulith.Products.Contracts;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class AddOrderItemHandler(IRepository<Order> repository, IMediator mediator) 
    : ICommandHandler<AddOrderItemCommand, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(AddOrderItemCommand command, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(command.OrderId, ct);

        if (order is null)
        {
            return Result<OrderDto>.NotFound();
        }

        // Fetch the product price from the Products module
        decimal unitPrice;
        try
        {
            unitPrice = await mediator.Send(new GetProductPriceQuery(command.ProductId), ct);
        }
        catch (InvalidOperationException ex)
        {
            return Result<OrderDto>.Error($"Failed to get price for product {command.ProductId}: {ex.Message}");
        }

        var item = new OrderItem
        {
            ProductId = command.ProductId,
            ProductName = command.ProductName,
            Quantity = command.Quantity,
            UnitPrice = unitPrice
        };

        order.AddItem(item);
        order.UpdatedAt = DateTime.UtcNow;

        await repository.UpdateAsync(order, ct);
        await repository.SaveChangesAsync(ct);

        var dto = new OrderDto(
            order.Id,
            order.CustomerId,
            order.OrderNumber,
            order.OrderDate,
            order.Status.ToString(),
            order.TotalAmount,
            order.Items.Select(i => new OrderItemDto(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList(),
            order.CreatedAt,
            order.UpdatedAt
        );

        return Result<OrderDto>.Success(dto);
    }
}
```

### 1.9: Create OrderCreatedEvent

Define a domain event in Customers.Contracts to notify other parts of the system when an order is confirmed:

**`Nimble.Modulith.Customers.Contracts/OrderCreatedEvent.cs`:**

```csharp
using Mediator;

namespace Nimble.Modulith.Customers.Contracts;

public record OrderCreatedEvent(
    int OrderId,
    int CustomerId,
    string CustomerEmail,
    string OrderNumber,
    DateOnly OrderDate,
    decimal TotalAmount,
    List<OrderItemDetails> Items
) : INotification;

public record OrderItemDetails(
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal TotalPrice
);
```

### 1.10: Create ConfirmOrder Command and Handler

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/ConfirmOrderCommand.cs`:**

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public record ConfirmOrderCommand(int OrderId) : ICommand<Result<OrderDto>>;
```

**`Nimble.Modulith.Customers/UseCases/Orders/Commands/ConfirmOrderHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class ConfirmOrderHandler(IRepository<Order> repository) 
    : ICommandHandler<ConfirmOrderCommand, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(ConfirmOrderCommand command, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(command.OrderId, ct);

        if (order is null)
        {
            return Result<OrderDto>.NotFound($"Order with ID {command.OrderId} not found");
        }

        // Change order status to Processing
        order.Status = OrderStatus.Processing;
        order.UpdatedAt = DateTime.UtcNow;

        await repository.UpdateAsync(order, ct);
        await repository.SaveChangesAsync(ct);

        var dto = new OrderDto(
            order.Id,
            order.CustomerId,
            order.OrderNumber,
            order.OrderDate,
            order.Status.ToString(),
            order.TotalAmount,
            order.Items.Select(i => new OrderItemDto(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList(),
            order.CreatedAt,
            order.UpdatedAt
        );

        return Result<OrderDto>.Success(dto);
    }
}
```

### 1.11: Create Confirm Endpoint with Event Publishing

**`Nimble.Modulith.Customers/Endpoints/Orders/Confirm.cs`:**

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.Contracts;
using Nimble.Modulith.Customers.Infrastructure;
using Nimble.Modulith.Customers.UseCases.Customers.Queries;
using Nimble.Modulith.Customers.UseCases.Orders.Commands;
using Nimble.Modulith.Customers.UseCases.Orders.Queries;
using Nimble.Modulith.Email.Contracts;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class Confirm(IMediator mediator, ICustomerAuthorizationService authService) : EndpointWithoutRequest<OrderResponse>
{
    public override void Configure()
    {
        Post("/orders/{id}/confirm");
        Summary(s =>
        {
            s.Summary = "Confirm an order";
            s.Description = "Changes order status to Processing";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var orderId = Route<int>("id");

        // Verify the order exists and get the customer ID
        var orderQuery = new GetOrderByIdQuery(orderId);
        var orderResult = await mediator.Send(orderQuery, ct);

        if (!orderResult.IsSuccess)
        {
            AddError($"Order with ID {orderId} not found");
            await Send.ErrorsAsync(statusCode: 404, cancellation: ct);
            return;
        }

        // Verify user has permission to confirm this order
        var customerQuery = new GetCustomerByIdQuery(orderResult.Value.CustomerId);
        var customerResult = await mediator.Send(customerQuery, ct);

        if (!customerResult.IsSuccess)
        {
            AddError($"Customer with ID {orderResult.Value.CustomerId} not found");
            await Send.ErrorsAsync(statusCode: 404, cancellation: ct);
            return;
        }

        if (!authService.IsAdminOrOwner(User, customerResult.Value.Email))
        {
            AddError("You can only confirm your own orders");
            await Send.ErrorsAsync(statusCode: 403, cancellation: ct);
            return;
        }

        // Confirm the order
        var command = new ConfirmOrderCommand(orderId);
        var result = await mediator.Send(command, ct);

        if (!result.IsSuccess)
        {
            AddError("Failed to confirm order");
            await Send.ErrorsAsync(cancellation: ct);
            return;
        }

        // Publish OrderCreatedEvent
        var orderCreatedEvent = new OrderCreatedEvent(
            result.Value.Id,
            result.Value.CustomerId,
            customerResult.Value.Email,
            result.Value.OrderNumber,
            result.Value.OrderDate,
            result.Value.TotalAmount,
            result.Value.Items.Select(i => new OrderItemDetails(
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList()
        );

        await mediator.Publish(orderCreatedEvent, ct);

        // Send confirmation email
        var emailBody = $@"
Dear Customer,

Your order has been confirmed!

Order Number: {result.Value.OrderNumber}
Order Date: {result.Value.OrderDate:yyyy-MM-dd}
Total Amount: ${result.Value.TotalAmount:F2}

Items:
{string.Join("\n", result.Value.Items.Select(i => $"- {i.ProductName} x {i.Quantity} @ ${i.UnitPrice:F2} = ${i.TotalPrice:F2}"))}

Thank you for your order!
";

        var emailCommand = new SendEmailCommand(
            customerResult.Value.Email,
            $"Order Confirmation - {result.Value.OrderNumber}",
            emailBody
        );

        await mediator.Send(emailCommand, ct);

        // Map to response
        Response = new OrderResponse(
            result.Value.Id,
            result.Value.CustomerId,
            result.Value.OrderNumber,
            result.Value.OrderDate,
            result.Value.Status,
            result.Value.TotalAmount,
            result.Value.Items.Select(i => new OrderItemResponse(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList()
        );
    }
}
```

## Step 2: Implement Password Generation for New Customers

When a new customer is created, we'll automatically create a user account with a generated password and email it to them.

### 2.1: Create PasswordGenerator Utility

**`Nimble.Modulith.Users/Infrastructure/PasswordGenerator.cs`:**

```csharp
namespace Nimble.Modulith.Users.Infrastructure;

public static class PasswordGenerator
{
    /// <summary>
    /// Generates a random password using a portion of a GUID.
    /// In production, consider more secure password generation with specific complexity requirements.
    /// </summary>
    /// <returns>A random password string</returns>
    public static string GeneratePassword()
    {
        // Take first 12 characters of a GUID (removes hyphens for simplicity)
        return Guid.NewGuid().ToString("N")[..12];
    }
}
```

### 2.2: Create CreateUserCommand in Users.Contracts

**`Nimble.Modulith.Users.Contracts/CreateUserCommand.cs`:**

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Users.Contracts;

public record CreateUserCommand(
    string Email,
    string Password
) : ICommand<Result<string>>;  // Returns the user ID
```

### 2.3: Create Internal CreateUserInternalCommand

Following the pattern of separating external contracts from internal implementation:

**`Nimble.Modulith.Users/UseCases/Commands/CreateUserInternalCommand.cs`:**

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Users.UseCases.Commands;

/// <summary>
/// Internal command for creating a user within the Users module.
/// </summary>
public record CreateUserInternalCommand(
    string Email,
    string Password
) : ICommand<Result<string>>;  // Returns the user ID
```

### 2.4: Create Internal Handler

**`Nimble.Modulith.Users/UseCases/Commands/CreateUserInternalCommandHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Microsoft.AspNetCore.Identity;

namespace Nimble.Modulith.Users.UseCases.Commands;

/// <summary>
/// Internal handler for creating users within the Users module.
/// </summary>
public class CreateUserInternalCommandHandler(UserManager<IdentityUser> userManager)
    : ICommandHandler<CreateUserInternalCommand, Result<string>>
{
    public async ValueTask<Result<string>> Handle(CreateUserInternalCommand command, CancellationToken cancellationToken)
    {
        var user = new IdentityUser
        {
            UserName = command.Email,
            Email = command.Email,
            EmailConfirmed = true // Auto-confirm for simplicity
        };

        var result = await userManager.CreateAsync(user, command.Password);

        if (!result.Succeeded)
        {
            var errors = string.Join("; ", result.Errors.Select(e => e.Description));
            return Result<string>.Error($"Failed to create user: {errors}");
        }

        return Result<string>.Success(user.Id);
    }
}
```

### 2.5: Create Integrations Handler

This handler translates the external contract command to the internal command:

**`Nimble.Modulith.Users/Integrations/CreateUserCommandHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Users.Contracts;
using Nimble.Modulith.Users.UseCases.Commands;

namespace Nimble.Modulith.Users.Integrations;

/// <summary>
/// Integration handler that translates external CreateUserCommand (from Contracts)
/// to internal CreateUserInternalCommand for processing by the Users module.
/// </summary>
public class CreateUserCommandHandler(IMediator mediator)
    : ICommandHandler<CreateUserCommand, Result<string>>
{
    public async ValueTask<Result<string>> Handle(CreateUserCommand command, CancellationToken cancellationToken)
    {
        // Translate external command to internal command
        var internalCommand = new CreateUserInternalCommand(
            command.Email,
            command.Password
        );

        // Delegate to internal handler
        return await mediator.Send(internalCommand, cancellationToken);
    }
}
```

### 2.6: Update CreateCustomerHandler

Now update the customer creation handler to create a user account and send welcome email:

**`Nimble.Modulith.Customers/UseCases/Customers/Commands/CreateCustomerHandler.cs`:**

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Email.Contracts;
using Nimble.Modulith.Users.Contracts;

namespace Nimble.Modulith.Customers.UseCases.Customers.Commands;

public class CreateCustomerHandler(IRepository<Customer> repository, IMediator mediator) 
    : ICommandHandler<CreateCustomerCommand, Result<CustomerDto>>
{
    public async ValueTask<Result<CustomerDto>> Handle(CreateCustomerCommand command, CancellationToken ct)
    {
        // Generate a random password
        var password = Guid.NewGuid().ToString("N")[..12]; // First 12 chars of GUID

        // Create Identity user
        var createUserCommand = new CreateUserCommand(command.Email, password);
        var userResult = await mediator.Send(createUserCommand, ct);

        if (!userResult.IsSuccess)
        {
            return Result<CustomerDto>.Error($"Failed to create user account: {userResult.Errors.FirstOrDefault()}");
        }

        // Create customer record
        var customer = new Customer
        {
            FirstName = command.FirstName,
            LastName = command.LastName,
            Email = command.Email,
            PhoneNumber = command.PhoneNumber,
            Address = new Address
            {
                Street = command.Street,
                City = command.City,
                State = command.State,
                PostalCode = command.PostalCode,
                Country = command.Country
            }
        };

        await repository.AddAsync(customer, ct);
        await repository.SaveChangesAsync(ct);

        // Send welcome email with password
        var emailBody = $@"
Welcome to our service!

Your account has been created successfully.

Email: {command.Email}
Temporary Password: {password}

Please log in and change your password as soon as possible.

Best regards,
The Team
";

        var emailCommand = new SendEmailCommand(
            command.Email,
            "Welcome - Your Account Has Been Created",
            emailBody
        );

        await mediator.Send(emailCommand, ct);

        var dto = new CustomerDto(
            customer.Id,
            customer.FirstName,
            customer.LastName,
            customer.Email,
            customer.PhoneNumber,
            new AddressDto(
                customer.Address.Street,
                customer.Address.City,
                customer.Address.State,
                customer.Address.PostalCode,
                customer.Address.Country
            ),
            customer.CreatedAt,
            customer.UpdatedAt
        );

        return Result<CustomerDto>.Success(dto);
    }
}
```

## Step 3: Implement Password Reset

### 3.1: Create ResetPassword Endpoint

**`Nimble.Modulith.Users/Endpoints/ResetPassword.cs`:**

```csharp
using FastEndpoints;
using Mediator;
using Microsoft.AspNetCore.Identity;
using Nimble.Modulith.Email.Contracts;
using Nimble.Modulith.Users.Infrastructure;

namespace Nimble.Modulith.Users.Endpoints;

public class ResetPasswordRequest
{
    public string Email { get; set; } = string.Empty;
}

public class ResetPasswordResponse
{
    public string Message { get; set; } = string.Empty;
    public bool Success { get; set; }
}

public class ResetPassword(UserManager<IdentityUser> userManager, IMediator mediator) : 
    Endpoint<ResetPasswordRequest, ResetPasswordResponse>
{
    public override void Configure()
    {
        Post("/users/reset-password");
        AllowAnonymous(); // Allow anyone to request password reset
        Summary(s => {
            s.Summary = "Reset user password";
            s.Description = "Generates a new password and emails it to the user";
        });
    }

    public override async Task HandleAsync(ResetPasswordRequest req, CancellationToken ct)
    {
        var user = await userManager.FindByEmailAsync(req.Email);
        
        if (user == null)
        {
            // Don't reveal whether user exists or not for security
            Response = new ResetPasswordResponse
            {
                Success = true,
                Message = "If the email exists in our system, a password reset email has been sent."
            };
            return;
        }

        // Generate new password
        var newPassword = PasswordGenerator.GeneratePassword();

        // Remove old password and set new one
        var removeResult = await userManager.RemovePasswordAsync(user);
        if (!removeResult.Succeeded)
        {
            AddError("Failed to reset password");
            await Send.ErrorsAsync(cancellation: ct);
            return;
        }

        var addResult = await userManager.AddPasswordAsync(user, newPassword);
        if (!addResult.Succeeded)
        {
            AddError("Failed to set new password");
            foreach (var error in addResult.Errors)
            {
                AddError(error.Description);
            }
            await Send.ErrorsAsync(cancellation: ct);
            return;
        }

        // Send email with new password
        var emailBody = $@"
Hello,

Your password has been reset successfully.

Your new temporary password is: {newPassword}

Please log in and change your password as soon as possible.

Best regards,
The Team
";

        var emailCommand = new SendEmailCommand(
            user.Email!,
            "Password Reset - New Temporary Password",
            emailBody
        );

        await mediator.Send(emailCommand, ct);

        Response = new ResetPasswordResponse
        {
            Success = true,
            Message = "If the email exists in our system, a password reset email has been sent."
        };
    }
}
```

## Success Criteria for Lab 6

After completing this lab, you should have:

- ✅ Orders that fetch prices from the Products module using cross-module queries
- ✅ Order confirmation endpoint that changes status to Processing
- ✅ OrderCreatedEvent published when orders are confirmed
- ✅ Email notifications sent when orders are confirmed
- ✅ Automatic password generation for new customers
- ✅ User accounts created automatically when customers are created
- ✅ Welcome emails sent with temporary passwords
- ✅ Password reset functionality with email notification

## Key Patterns Demonstrated

1. **Cross-Module Queries**: Products.Contracts.GetProductPriceQuery allows Customers module to fetch prices
2. **Domain Events**: OrderCreatedEvent allows decoupled notification of order creation
3. **Integration Handlers**: Separate external contracts from internal implementation
4. **Transactional Emails**: Use Email.Contracts.SendEmailCommand for all email notifications
5. **Password Management**: Generate secure passwords and manage them through Identity

## Notes

- For simplicity, passwords are generated from GUIDs (first 12 characters)
- In production:
  - Use password reset tokens with expiration instead of immediately sending new passwords
  - Implement proper password complexity requirements
  - Use secure password generation libraries
  - Consider multi-factor authentication
  - Don't email passwords in plain text - use password reset links instead
- The "don't reveal if user exists" pattern in password reset improves security

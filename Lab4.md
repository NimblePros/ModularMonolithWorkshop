# Lab 4: Add a third module using Clean Architecture for Customers and Orders

In this lab, we will add a third module to our modular monolith application for managing Customers and Orders. Unlike the simpler CRUD approach used in the Products module, this module will follow Clean Architecture principles with proper separation of concerns, domain models, application services, and infrastructure concerns.

When we're done, we will have added these new projects to our solution:

- Customers (main module project with domain models, use cases, infrastructure, and endpoints organized in folders)
- Customers.Contracts (shared contracts for the module)

## Clean Architecture Overview

Clean Architecture organizes code into layers with dependencies pointing inward. In this module, we'll organize these layers as folders within the Customers project:

- **Domain Layer** (Domain/): Contains entities, value objects, aggregates, and repository interfaces - **NO external dependencies**
  - `Domain/CustomerAggregate/` - Customer entity and specifications
  - `Domain/OrderAggregate/` - Order entities and specifications
  - `Domain/Common/` - Base entity classes
  - `Domain/Interfaces/` - Repository interfaces
  
- **Application Layer** (UseCases/): Contains commands, queries, handlers, and UseCases DTOs - **depends ONLY on Domain**
  - `UseCases/Customers/` - Customer commands, queries, handlers, and DTOs
  - `UseCases/Orders/` - Order commands, queries, handlers, and DTOs
  
- **Infrastructure Layer** (Infrastructure/): Contains data access, repository implementations - depends on Application and Domain
  - `Infrastructure/Data/` - DbContext, configurations, migrations, repository implementations
  
- **Presentation Layer** (Endpoints/): Contains API endpoints and Endpoint DTOs - communicates with Application via Mediator
  - `Endpoints/Customers/` - Customer endpoints and endpoint DTOs
  - `Endpoints/Orders/` - Order endpoints and endpoint DTOs

**Critical Rule**: Neither Domain nor UseCases should reference Infrastructure types. Handlers use repository interfaces (Domain), not DbContext (Infrastructure).

This module will demonstrate:
- Domain-driven design with rich domain models
- CQRS pattern with commands, queries, and Mediator
- Three distinct DTO types (Details, UseCases DTOs, Endpoint DTOs)
- Repository and Specification patterns for data access
- Dependency inversion - UseCases depend on interfaces, not implementations
- Result pattern for error handling
- Folder-based organization within a single project

## Adding the Customers Module

Run the following commands from the solution directory to create the module structure:

```bash
# Create the Customers.Contracts project
dotnet new classlib -n Nimble.Modulith.Customers.Contracts -o Nimble.Modulith.Customers.Contracts

# Create the Customers project
dotnet new classlib -n Nimble.Modulith.Customers -o Nimble.Modulith.Customers

# Add both projects to the solution in a "Customers Module" solution folder
dotnet sln add Nimble.Modulith.Customers.Contracts/Nimble.Modulith.Customers.Contracts.csproj --solution-folder "Customers Module"
dotnet sln add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj --solution-folder "Customers Module"

# Add reference from Customers to Customers.Contracts
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj reference Nimble.Modulith.Customers.Contracts/Nimble.Modulith.Customers.Contracts.csproj

# Add Customers project reference to Web project
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj reference Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj

# Add NuGet packages to the Customers project
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Ardalis.Result
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Ardalis.Specification
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package FastEndpoints
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Serilog.AspNetCore
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Microsoft.EntityFrameworkCore
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Microsoft.EntityFrameworkCore.SqlServer
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Microsoft.EntityFrameworkCore.Design
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Aspire.Microsoft.EntityFrameworkCore.SqlServer
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Ardalis.Specification.EntityFrameworkCore
dotnet add Nimble.Modulith.Customers/Nimble.Modulith.Customers.csproj package Mediator.Abstractions

# Add Mediator source generation package to the Web project
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj package Mediator.SourceGenerator

# Build the solution to verify everything is set up correctly
dotnet build
```

## Configure the Customers Database

### 1. Add the Customers Database to AppHost

Open `Nimble.Modulith.AppHost/AppHost.cs` and add a new database for customers:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add SQL Server database for Identity
var sqlServer = builder.AddSqlServer("sqlserver")
    .WithDataVolume();

var usersDb = sqlServer.AddDatabase("usersdb");
var productsDb = sqlServer.AddDatabase("productsdb");
var customersDb = sqlServer.AddDatabase("customersdb");

// Add the Web API project with database reference
builder.AddProject<Projects.Nimble_Modulith_Web>("webapi")
    .WithReference(usersDb)
    .WithReference(productsDb)
    .WithReference(customersDb)
    .WaitFor(usersDb)
    .WaitFor(productsDb)
    .WaitFor(customersDb);

builder.Build().Run();
```

### 2. Create the CustomersDbContext

Create a new file `Nimble.Modulith.Customers/Infrastructure/Data/CustomersDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.Infrastructure.Data;

public class CustomersDbContext : DbContext
{
    public CustomersDbContext(DbContextOptions<CustomersDbContext> options)
        : base(options)
    {
    }

    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Apply Customers module specific configurations
        builder.HasDefaultSchema("Customers");
        
        // Apply all configurations from this assembly
        builder.ApplyConfigurationsFromAssembly(typeof(CustomersDbContext).Assembly);
    }
}
```

### 3. Create the DbContext Factory

Create `Nimble.Modulith.Customers/Infrastructure/Data/CustomersDbContextFactory.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace Nimble.Modulith.Customers.Infrastructure.Data;

public class CustomersDbContextFactory : IDesignTimeDbContextFactory<CustomersDbContext>
{
    public CustomersDbContext CreateDbContext(string[] args)
    {
        var optionsBuilder = new DbContextOptionsBuilder<CustomersDbContext>();
        optionsBuilder.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=CustomersDb;Trusted_Connection=True;MultipleActiveResultSets=true");

        return new CustomersDbContext(optionsBuilder.Options);
    }
}
}
```

## Create Domain Models

**Important**: Domain layer has NO dependencies on Infrastructure. All types here are pure domain logic.

### 1. Create Base Entity

Create `Nimble.Modulith.Customers/Domain/Common/EntityBase.cs`:

```csharp
namespace Nimble.Modulith.Customers.Domain.Common;

public abstract class EntityBase
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; set; }
}
```

### 2. Create Customer Entity

Create `Nimble.Modulith.Customers/Domain/CustomerAggregate/Customer.cs`:

```csharp
using Nimble.Modulith.Customers.Domain.Common;

namespace Nimble.Modulith.Customers.Domain.CustomerAggregate;

public class Customer : EntityBase
{
    public string FirstName { get; set; } = string.Empty;
    public string LastName { get; set; } = string.Empty;
    public string Email { get; set; } = string.Empty;
    public string PhoneNumber { get; set; } = string.Empty;
    public Address Address { get; set; } = null!;

    public string FullName => $"{FirstName} {LastName}";
}
```

### 3. Create Address Value Object

Create `Nimble.Modulith.Customers/Domain/CustomerAggregate/Address.cs`:

```csharp
namespace Nimble.Modulith.Customers.Domain.CustomerAggregate;

public class Address
{
    public string Street { get; set; } = string.Empty;
    public string City { get; set; } = string.Empty;
    public string State { get; set; } = string.Empty;
    public string PostalCode { get; set; } = string.Empty;
    public string Country { get; set; } = string.Empty;

    public string FullAddress => $"{Street}, {City}, {State} {PostalCode}, {Country}";
}
```

### 4. Create Order Entity

Create `Nimble.Modulith.Customers/Domain/OrderAggregate/Order.cs`:

```csharp
using System.Diagnostics.CodeAnalysis;
using Nimble.Modulith.Customers.Domain.Common;

namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public class Order : EntityBase
{
    private readonly List<OrderItem> _items = new();

    public int CustomerId { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public DateOnly OrderDate { get; set; }
    public OrderStatus Status { get; set; } = OrderStatus.Pending;
    public decimal TotalAmount => Items.Sum(i => i.TotalPrice);
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(OrderItem item)
    {
        ArgumentNullException.ThrowIfNull(item);

        // Check if an item with the same product already exists
        var existingItem = _items.FirstOrDefault(i => i.ProductId == item.ProductId);
        
        if (existingItem != null)
        {
            // Combine quantities for existing product
            existingItem.Quantity += item.Quantity;
        }
        else
        {
            _items.Add(item);
        }
    }

    public void RemoveItem(OrderItem item)
    {
        ArgumentNullException.ThrowIfNull(item);
        _items.Remove(item);
    }
}
```

**Key Design Decisions:**
- **DateOnly for OrderDate**: Using `DateOnly` instead of `DateTime` since we only care about the date, not the time
- **Encapsulated Collection**: `Items` uses a private backing field `_items` and exposes `IReadOnlyList<OrderItem>` to prevent external modification
- **Calculated TotalAmount**: `TotalAmount` is a computed property that sums item totals, ensuring it's always accurate
- **AddItem with Guard Clauses**: Validates input and combines quantities when the same product is added multiple times
- **RemoveItem**: Safely removes items with null checking

### 5. Create OrderItem Entity

Create `Nimble.Modulith.Customers/Domain/OrderAggregate/OrderItem.cs`:

```csharp
using Nimble.Modulith.Customers.Domain.Common;

namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public class OrderItem : EntityBase
{
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public string ProductName { get; set; } = string.Empty;
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    public decimal TotalPrice => Quantity * UnitPrice;
}
```

### 6. Create OrderStatus Enum

Create `Nimble.Modulith.Customers/Domain/OrderAggregate/OrderStatus.cs`:

```csharp
namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public enum OrderStatus
{
    Pending = 0,
    Processing = 1,
    Shipped = 2,
    Delivered = 3,
    Cancelled = 4
}
```

## Create Repository Interfaces

**Important**: Repository interfaces belong in Domain layer, not Infrastructure.

### 1. Create Generic Repository Interface

Create `Nimble.Modulith.Customers/Domain/Interfaces/IRepository.cs`:

```csharp
using Ardalis.Specification;

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.Interfaces;

public interface IRepository<T> : IRepositoryBase<T> where T : class
{
}
```

### 2. Create Read-Only Repository Interface

Create `Nimble.Modulith.Customers/Domain/Interfaces/IReadRepository.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.Interfaces;

public interface IReadRepository<T> : IReadRepositoryBase<T> where T : class
{
}
```

## Create Specifications

Specifications encapsulate query logic and belong in the Domain layer.

### 1. Create Customer Specifications

Create `Nimble.Modulith.Customers/Domain/CustomerAggregate/Specifications/CustomerByIdSpec.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.CustomerAggregate.Specifications;

public class CustomerByIdSpec : Specification<Customer>
{
    public CustomerByIdSpec(int customerId)
    {
        Query.Where(c => c.Id == customerId);
    }
}
```

Create `Nimble.Modulith.Customers/Domain/CustomerAggregate/Specifications/CustomerByEmailSpec.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.CustomerAggregate.Specifications;

public class CustomerByEmailSpec : Specification<Customer>
{
    public CustomerByEmailSpec(string email)
    {
        Query.Where(c => c.Email == email);
    }
}
```

### 2. Create Order Specifications

Create `Nimble.Modulith.Customers/Domain/OrderAggregate/OrderByIdSpec.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public class OrderByIdSpec : Specification<Order>, ISingleResultSpecification<Order>
{
    public OrderByIdSpec(int orderId)
    {
        Query
            .Where(o => o.Id == orderId)
            .Include(o => o.Items);
    }
}
```

Create `Nimble.Modulith.Customers/Domain/OrderAggregate/OrdersByCustomerSpec.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public class OrdersByCustomerSpec : Specification<Order>
{
    public OrdersByCustomerSpec(int customerId)
    {
        Query.Where(o => o.CustomerId == customerId)
            .Include(o => o.Items)
            .OrderByDescending(o => o.OrderDate);
    }
}
```

## Create Entity Configurations

Entity configurations are Infrastructure concerns.

### 1. Create Customer Configuration

Create `Nimble.Modulith.Customers/Infrastructure/Data/Config/CustomerConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;

namespace Nimble.Modulith.Customers.Infrastructure.Data.Config;

public class CustomerConfiguration : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> builder)
    {
        builder.ToTable("Customers");

        builder.HasKey(c => c.Id);

        builder.Property(c => c.FirstName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(c => c.LastName)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(c => c.Email)
            .IsRequired()
            .HasMaxLength(256);

        builder.HasIndex(c => c.Email)
            .IsUnique();

        builder.Property(c => c.PhoneNumber)
            .HasMaxLength(20);

        // Configure Address as owned entity (value object)
        builder.OwnsOne(c => c.Address, address =>
        {
            address.Property(a => a.Street).HasMaxLength(200);
            address.Property(a => a.City).HasMaxLength(100);
            address.Property(a => a.State).HasMaxLength(100);
            address.Property(a => a.PostalCode).HasMaxLength(20);
            address.Property(a => a.Country).HasMaxLength(100);
        });
    }
}
```

### 2. Create Order Configuration

Create `Nimble.Modulith.Customers/Infrastructure/Data/Config/OrderConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.Infrastructure.Data.Config;

public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("Orders");

        builder.HasKey(o => o.Id);

        builder.Property(o => o.OrderNumber)
            .IsRequired()
            .HasMaxLength(50);

        builder.HasIndex(o => o.OrderNumber)
            .IsUnique();

        builder.Property(o => o.OrderDate)
            .IsRequired();

        builder.Property(o => o.Status)
            .IsRequired()
            .HasConversion<string>();

        builder.Property(o => o.Status)
            .IsRequired()
            .HasConversion<string>();

        // Ignore TotalAmount - it's a calculated property
        builder.Ignore(o => o.TotalAmount);

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey(i => i.OrderId)
            .OnDelete(DeleteBehavior.Cascade)
            .Metadata.PrincipalToDependent?.SetField("_items");
    }
}
```

**Note**: The `.Metadata.PrincipalToDependent?.SetField("_items")` configuration tells EF Core to use the private backing field `_items` for the Items collection, which is necessary since we exposed it as `IReadOnlyList<OrderItem>`. We also use `.Ignore()` for `TotalAmount` since it's a calculated property and should not be persisted to the database.

### 3. Create OrderItem Configuration

Create `Nimble.Modulith.Customers/Infrastructure/Data/Config/OrderItemConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.Infrastructure.Data.Config;

public class OrderItemConfiguration : IEntityTypeConfiguration<OrderItem>
{
    public void Configure(EntityTypeBuilder<OrderItem> builder)
    {
        builder.ToTable("OrderItems");

        builder.HasKey(i => i.Id);

        builder.Property(i => i.ProductName)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(i => i.Quantity)
            .IsRequired();

        builder.Property(i => i.UnitPrice)
            .HasPrecision(18, 2);

        // Ignore TotalPrice - it's a calculated property
        builder.Ignore(i => i.TotalPrice);
    }
}
```

**Note**: Like with `Order.TotalAmount`, we use `.Ignore()` for `OrderItem.TotalPrice` since it's calculated as `Quantity * UnitPrice` and doesn't need to be stored in the database.

## Create Repository Implementations

Repository implementations are Infrastructure and depend on DbContext.

### 1. Create Generic Repository

Create `Nimble.Modulith.Customers/Infrastructure/Data/EfRepository.cs`:

```csharp
using Ardalis.Specification.EntityFrameworkCore;
using Nimble.Modulith.Customers.Domain.Interfaces;

namespace Nimble.Modulith.Customers.Infrastructure.Data;

public class EfRepository<T> : RepositoryBase<T>, IRepository<T> where T : class
{
    public EfRepository(CustomersDbContext dbContext) : base(dbContext)
    {
    }
}
```

### 2. Create Read-Only Repository

Create `Nimble.Modulith.Customers/Infrastructure/Data/EfReadRepository.cs`:

```csharp
using Ardalis.Specification.EntityFrameworkCore;
using Nimble.Modulith.Customers.Domain.Interfaces;

namespace Nimble.Modulith.Customers.Infrastructure.Data;

public class EfReadRepository<T> : RepositoryBase<T>, IReadRepository<T> where T : class
{
    public EfReadRepository(CustomersDbContext dbContext) : base(dbContext)
    {
    }
}
}
```

## Create Contracts (For Cross-Module Communication)

The Contracts project contains types used for communication between modules. These use the "Details" suffix.

### 1. Create Customer Details

Create `Nimble.Modulith.Customers.Contracts/CustomerDetails.cs`:

```csharp
namespace Nimble.Modulith.Customers.Contracts;

/// <summary>
/// Customer details for cross-module communication
/// </summary>
public record CustomerDetails(
    int Id,
    string FirstName,
    string LastName,
    string Email,
    string PhoneNumber,
    AddressDetails Address
);

public record AddressDetails(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country
);
```

### 2. Create Order Details

Create `Nimble.Modulith.Customers.Contracts/OrderDetails.cs`:

```csharp
namespace Nimble.Modulith.Customers.Contracts;

/// <summary>
/// Order details for cross-module communication
/// </summary>
public record OrderDetails(
    int Id,
    int CustomerId,
    string OrderNumber,
    DateTime OrderDate,
    string Status,
    decimal TotalAmount,
    List<OrderItemDetails> Items
);

public record OrderItemDetails(
    int Id,
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal TotalPrice
);
```

## Create Use Cases Layer

**Critical**: Use Cases layer depends ONLY on Domain interfaces (IRepository), never on Infrastructure types (DbContext). Handlers receive repository interfaces via dependency injection.

The Use Cases layer contains commands, queries, handlers, and DTOs used internally within the module.

### 1. Create Customer Use Cases DTOs

Create `Nimble.Modulith.Customers/UseCases/Customers/CustomerDto.cs`:

```csharp
namespace Nimble.Modulith.Customers.UseCases.Customers;

/// <summary>
/// Customer DTO for use cases (includes internal tracking fields)
/// </summary>
public record CustomerDto(
    int Id,
    string FirstName,
    string LastName,
    string Email,
    string PhoneNumber,
    AddressDto Address,
    DateTime CreatedAt,
    DateTime? UpdatedAt
);

public record AddressDto(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country
);
```

### 2. Create Customer Commands

Create `Nimble.Modulith.Customers/UseCases/Customers/Commands/CreateCustomerCommand.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Customers.Commands;

public record CreateCustomerCommand(
    string FirstName,
    string LastName,
    string Email,
    string PhoneNumber,
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country
) : ICommand<Result<CustomerDto>>;
```

Create `Nimble.Modulith.Customers/UseCases/Customers/Commands/CreateCustomerHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.Interfaces;

namespace Nimble.Modulith.Customers.UseCases.Customers.Commands;

public class CreateCustomerHandler(IRepository<Customer> repository) 
    : ICommandHandler<CreateCustomerCommand, Result<CustomerDto>>
{
    public async ValueTask<Result<CustomerDto>> Handle(CreateCustomerCommand command, CancellationToken ct)
    {
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

### 3. Create Customer Queries

Create `Nimble.Modulith.Customers/UseCases/Customers/Queries/GetCustomerByIdQuery.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Customers.Queries;

public record GetCustomerByIdQuery(int Id) : IQuery<Result<CustomerDto>>;
```

Create `Nimble.Modulith.Customers/UseCases/Customers/Queries/GetCustomerByIdHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.CustomerAggregate.Specifications;
using Nimble.Modulith.Customers.Domain.Interfaces;

namespace Nimble.Modulith.Customers.UseCases.Customers.Queries;

public class GetCustomerByIdHandler(IReadRepository<Customer> repository) 
    : IQueryHandler<GetCustomerByIdQuery, Result<CustomerDto>>
{
    public async ValueTask<Result<CustomerDto>> Handle(GetCustomerByIdQuery query, CancellationToken ct)
    {
        var spec = new CustomerByIdSpec(query.Id);
        var customer = await repository.FirstOrDefaultAsync(spec, ct);

        if (customer is null)
        {
            return Result<CustomerDto>.NotFound();
        }

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

Create `Nimble.Modulith.Customers/UseCases/Customers/Queries/ListCustomersQuery.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Customers.Queries;

public record ListCustomersQuery() : IQuery<Result<List<CustomerDto>>>;
```

Create `Nimble.Modulith.Customers/UseCases/Customers/Queries/ListCustomersHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.Interfaces;

namespace Nimble.Modulith.Customers.UseCases.Customers.Queries;

public class ListCustomersHandler(IReadRepository<Customer> repository) 
    : IQueryHandler<ListCustomersQuery, Result<List<CustomerDto>>>
{
    public async ValueTask<Result<List<CustomerDto>>> Handle(ListCustomersQuery query, CancellationToken ct)
    {
        var customers = await repository.ListAsync(ct);
        
        var dtos = customers.Select(c => new CustomerDto(
            c.Id,
            c.FirstName,
            c.LastName,
            c.Email,
            c.PhoneNumber,
            new AddressDto(
                c.Address.Street,
                c.Address.City,
                c.Address.State,
                c.Address.PostalCode,
                c.Address.Country
            ),
            c.CreatedAt,
            c.UpdatedAt
        )).ToList();

        return Result<List<CustomerDto>>.Success(dtos);
    }
}
```

## Create Customer Endpoints

Endpoints contain their own DTOs (for API responses) and communicate with handlers via Mediator.

### 1. Create Endpoint DTOs

Create `Nimble.Modulith.Customers/Endpoints/Customers/CustomerResponse.cs`:

```csharp
namespace Nimble.Modulith.Customers.Endpoints.Customers;

/// <summary>
/// Customer response for API (excludes internal tracking fields like UpdatedAt)
/// </summary>
public record CustomerResponse(
    int Id,
    string FirstName,
    string LastName,
    string Email,
    string PhoneNumber,
    AddressResponse Address
);

public record AddressResponse(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country
);

public record CreateCustomerRequest(
    string FirstName,
    string LastName,
    string Email,
    string PhoneNumber,
    AddressRequest Address
);

public record AddressRequest(
    string Street,
    string City,
    string State,
    string PostalCode,
    string Country
);
```

### 2. Create Customer Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Customers/Create.cs`:

```csharp
using Ardalis.Result;
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Customers;
using Nimble.Modulith.Customers.UseCases.Customers.Commands;

namespace Nimble.Modulith.Customers.Endpoints.Customers;

public class Create(IMediator mediator) : Endpoint<CreateCustomerRequest, CustomerResponse>
{
    public override void Configure()
    {
        Post("/customers");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Create a new customer";
            s.Description = "Creates a new customer with the provided information";
        });
        Tags("customers");
    }

    public override async Task HandleAsync(CreateCustomerRequest req, CancellationToken ct)
    {
        var command = new CreateCustomerCommand(
            req.FirstName,
            req.LastName,
            req.Email,
            req.PhoneNumber,
            req.Address.Street,
            req.Address.City,
            req.Address.State,
            req.Address.PostalCode,
            req.Address.Country
        );

        var result = await mediator.Send(command, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTO to Endpoint Response DTO
        Response = new CustomerResponse(
            result.Value.Id,
            result.Value.FirstName,
            result.Value.LastName,
            result.Value.Email,
            result.Value.PhoneNumber,
            new AddressResponse(
                result.Value.Address.Street,
                result.Value.Address.City,
                result.Value.Address.State,
                result.Value.Address.PostalCode,
                result.Value.Address.Country
            )
        );

        await Send.CreatedAtAsync<GetById>(new { id = result.Value.Id }, generateAbsoluteUrl: false, cancellation: ct);
    }
}
```

### 3. List Customers Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Customers/List.cs`:

```csharp
using Ardalis.Result;
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Customers.Queries;

namespace Nimble.Modulith.Customers.Endpoints.Customers;

public class List(IMediator mediator) : EndpointWithoutRequest<List<CustomerResponse>>
{
    public override void Configure()
    {
        Get("/customers");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "List all customers";
            s.Description = "Returns a list of all customers";
        });
        Tags("customers");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var query = new ListCustomersQuery();
        var result = await mediator.Send(query, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTOs to Endpoint Response DTOs
        Response = result.Value.Select(c => new CustomerResponse(
            c.Id,
            c.FirstName,
            c.LastName,
            c.Email,
            c.PhoneNumber,
            new AddressResponse(
                c.Address.Street,
                c.Address.City,
                c.Address.State,
                c.Address.PostalCode,
                c.Address.Country
            )
        )).ToList();
    }
}
```

### 4. Get Customer By ID Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Customers/GetById.cs`:

```csharp
using Ardalis.Result;
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Customers.Queries;

namespace Nimble.Modulith.Customers.Endpoints.Customers;

public class GetById(IMediator mediator) : EndpointWithoutRequest<CustomerResponse>
{
    public override void Configure()
    {
        Get("/customers/{id}");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Get a customer by ID";
            s.Description = "Returns a single customer by their ID";
        });
        Tags("customers");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var id = Route<int>("id");
        var query = new GetCustomerByIdQuery(id);
        var result = await mediator.Send(query, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTO to Endpoint Response DTO
        Response = new CustomerResponse(
            result.Value.Id,
            result.Value.FirstName,
            result.Value.LastName,
            result.Value.Email,
            result.Value.PhoneNumber,
            new AddressResponse(
                result.Value.Address.Street,
                result.Value.Address.City,
                result.Value.Address.State,
                result.Value.Address.PostalCode,
                result.Value.Address.Country
            )
        );
    }
}
```

## Create Order Use Cases and Endpoints

Now we'll create the Order use cases and endpoints following the same Clean Architecture patterns we used for Customers.

### 1. Create Order Use Cases DTOs

Create `Nimble.Modulith.Customers/UseCases/Orders/OrderDto.cs`:

```csharp
namespace Nimble.Modulith.Customers.UseCases.Orders;

public record OrderDto(
    int Id,
    int CustomerId,
    string OrderNumber,
    DateOnly OrderDate,
    string Status,
    decimal TotalAmount,
    List<OrderItemDto> Items,
    DateTime CreatedAt,
    DateTime? UpdatedAt
);

public record OrderItemDto(
    int Id,
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal TotalPrice
);
```

### 2. Create Order Commands

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/CreateOrderCommand.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public record CreateOrderCommand(
    int CustomerId,
    DateOnly OrderDate,
    List<CreateOrderItemDto> Items
) : ICommand<Result<OrderDto>>;

public record CreateOrderItemDto(
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice
);
```

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/CreateOrderHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.CustomerAggregate;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class CreateOrderHandler(
    IRepository<Order> orderRepository,
    IReadRepository<Customer> customerRepository) 
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

        // Add order items
        foreach (var itemDto in command.Items)
        {
            var item = new OrderItem
            {
                ProductId = itemDto.ProductId,
                ProductName = itemDto.ProductName,
                Quantity = itemDto.Quantity,
                UnitPrice = itemDto.UnitPrice
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

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/AddOrderItemCommand.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public record AddOrderItemCommand(
    int OrderId,
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice
) : ICommand<Result<OrderDto>>;
```

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/AddOrderItemHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class AddOrderItemHandler(IRepository<Order> repository) 
    : ICommandHandler<AddOrderItemCommand, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(AddOrderItemCommand command, CancellationToken ct)
    {
        var spec = new OrderByIdSpec(command.OrderId);
        var order = await repository.SingleOrDefaultAsync(spec, ct);

        if (order is null)
        {
            return Result<OrderDto>.NotFound();
        }

        var item = new OrderItem
        {
            ProductId = command.ProductId,
            ProductName = command.ProductName,
            Quantity = command.Quantity,
            UnitPrice = command.UnitPrice
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

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/DeleteOrderItemCommand.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public record DeleteOrderItemCommand(int OrderId, int OrderItemId) : ICommand<Result<OrderDto>>;
```

Create `Nimble.Modulith.Customers/UseCases/Orders/Commands/DeleteOrderItemHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Commands;

public class DeleteOrderItemHandler(IRepository<Order> repository) 
    : ICommandHandler<DeleteOrderItemCommand, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(DeleteOrderItemCommand command, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(command.OrderId, ct);

        if (order is null)
        {
            return Result<OrderDto>.NotFound("Order not found");
        }

        var item = order.Items.FirstOrDefault(i => i.Id == command.OrderItemId);
        if (item is null)
        {
            return Result<OrderDto>.NotFound("Order item not found");
        }

        order.RemoveItem(item);
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

### 3. Create Order Queries

Create `Nimble.Modulith.Customers/UseCases/Orders/Queries/GetOrderByIdQuery.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Queries;

public record GetOrderByIdQuery(int Id) : IQuery<Result<OrderDto>>;
```

Create `Nimble.Modulith.Customers/UseCases/Orders/Queries/GetOrderByIdHandler.cs`:

```csharp
using Ardalis.Result;
using Ardalis.Specification;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Queries;

public class GetOrderByIdHandler(IReadRepository<Order> repository) 
    : IQueryHandler<GetOrderByIdQuery, Result<OrderDto>>
{
    public async ValueTask<Result<OrderDto>> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        var spec = new OrderByIdSpec(query.Id);
        var order = await repository.FirstOrDefaultAsync(spec, ct);

        if (order is null)
        {
            return Result<OrderDto>.NotFound();
        }

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

Create `Nimble.Modulith.Customers/UseCases/Orders/Queries/ListOrdersByDateQuery.cs`:

```csharp
using Ardalis.Result;
using Mediator;

namespace Nimble.Modulith.Customers.UseCases.Orders.Queries;

public record ListOrdersByDateQuery(DateOnly OrderDate) : IQuery<Result<List<OrderDto>>>;
```

Create `Nimble.Modulith.Customers/UseCases/Orders/Queries/ListOrdersByDateHandler.cs`:

```csharp
using Ardalis.Result;
using Mediator;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Domain.OrderAggregate;

namespace Nimble.Modulith.Customers.UseCases.Orders.Queries;

public class ListOrdersByDateHandler(IReadRepository<Order> repository) 
    : IQueryHandler<ListOrdersByDateQuery, Result<List<OrderDto>>>
{
    public async ValueTask<Result<List<OrderDto>>> Handle(ListOrdersByDateQuery query, CancellationToken ct)
    {
        var spec = new OrdersByDateSpec(query.OrderDate);
        var orders = await repository.ListAsync(spec, ct);

        var dtos = orders.Select(order => new OrderDto(
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
        )).ToList();

        return Result<List<OrderDto>>.Success(dtos);
    }
}
```

We'll also need to add the specification for querying orders by date. Create `Nimble.Modulith.Customers/Domain/OrderAggregate/OrdersByDateSpec.cs`:

```csharp
using Ardalis.Specification;

namespace Nimble.Modulith.Customers.Domain.OrderAggregate;

public class OrdersByDateSpec : Specification<Order>
{
    public OrdersByDateSpec(DateOnly orderDate)
    {
        Query
            .Where(o => o.OrderDate == orderDate)
            .Include(o => o.Items)
            .OrderBy(o => o.CreatedAt);
    }
}
```

## Create Order Endpoints

### 1. Create Order Endpoint DTOs

Create `Nimble.Modulith.Customers/Endpoints/Orders/OrderResponse.cs`:

```csharp
namespace Nimble.Modulith.Customers.Endpoints.Orders;

public record OrderResponse(
    int Id,
    int CustomerId,
    string OrderNumber,
    DateOnly OrderDate,
    string Status,
    decimal TotalAmount,
    List<OrderItemResponse> Items
);

public record OrderItemResponse(
    int Id,
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice,
    decimal TotalPrice
);

public record CreateOrderRequest(
    int CustomerId,
    DateOnly OrderDate,
    List<CreateOrderItemRequest> Items
);

public record CreateOrderItemRequest(
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice
);

public record AddOrderItemRequest(
    int ProductId,
    string ProductName,
    int Quantity,
    decimal UnitPrice
);
```

### 2. Create Order Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Orders/Create.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Orders.Commands;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class Create(IMediator mediator) : Endpoint<CreateOrderRequest, OrderResponse>
{
    public override void Configure()
    {
        Post("/orders");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Create a new order";
            s.Description = "Creates a new order with the provided items";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var command = new CreateOrderCommand(
            req.CustomerId,
            req.OrderDate,
            req.Items.Select(i => new CreateOrderItemDto(
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice
            )).ToList()
        );

        var result = await mediator.Send(command, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTO to Endpoint Response DTO
        await Send.CreatedAtAsync<GetById>(
            new { id = result.Value.Id },
            new OrderResponse(
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
            ),
            generateAbsoluteUrl: false,
            cancellation: ct
        );
    }
}
```

### 3. Add Order Item Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Orders/AddItem.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Orders.Commands;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class AddItem(IMediator mediator) : Endpoint<AddOrderItemRequest, OrderResponse>
{
    public override void Configure()
    {
        Post("/orders/{id}/items");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Add an item to an order";
            s.Description = "Adds a new item to an existing order";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(AddOrderItemRequest req, CancellationToken ct)
    {
        var orderId = Route<int>("id");
        var command = new AddOrderItemCommand(
            orderId,
            req.ProductId,
            req.ProductName,
            req.Quantity,
            req.UnitPrice
        );

        var result = await mediator.Send(command, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTO to Endpoint Response DTO
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

### 4. Delete Order Item Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Orders/DeleteItem.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Orders.Commands;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class DeleteItem(IMediator mediator) : EndpointWithoutRequest<OrderResponse>
{
    public override void Configure()
    {
        Delete("/orders/{id}/items/{itemId}");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Delete an item from an order";
            s.Description = "Removes an item from an existing order";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var orderId = Route<int>("id");
        var itemId = Route<int>("itemId");
        var command = new DeleteOrderItemCommand(orderId, itemId);

        var result = await mediator.Send(command, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        // Map UseCases DTO to Endpoint Response DTO
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

### 5. Get Order By ID Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Orders/GetById.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Orders.Queries;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class GetById(IMediator mediator) : EndpointWithoutRequest<OrderResponse>
{
    public override void Configure()
    {
        Get("/orders/{id}");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "Get an order by ID";
            s.Description = "Returns a single order with all its items";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var id = Route<int>("id");
        var query = new GetOrderByIdQuery(id);
        var result = await mediator.Send(query, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

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

### 6. List Orders By Date Endpoint

Create `Nimble.Modulith.Customers/Endpoints/Orders/ListByDate.cs`:

```csharp
using FastEndpoints;
using Mediator;
using Nimble.Modulith.Customers.UseCases.Orders.Queries;

namespace Nimble.Modulith.Customers.Endpoints.Orders;

public class ListByDate(IMediator mediator) : EndpointWithoutRequest<List<OrderResponse>>
{
    public override void Configure()
    {
        Get("/orders/by-date/{date}");
        AllowAnonymous();
        Summary(s =>
        {
            s.Summary = "List orders by date";
            s.Description = "Returns all orders created on the specified date";
        });
        Tags("orders");
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var dateString = Route<string>("date");
        if (!DateOnly.TryParse(dateString, out var date))
        {
            await Send.ErrorsAsync(cancellation: ct);
            return;
        }

        var query = new ListOrdersByDateQuery(date);
        var result = await mediator.Send(query, ct);

        if (!result.IsSuccess)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        Response = result.Value.Select(o => new OrderResponse(
            o.Id,
            o.CustomerId,
            o.OrderNumber,
            o.OrderDate,
            o.Status,
            o.TotalAmount,
            o.Items.Select(i => new OrderItemResponse(
                i.Id,
                i.ProductId,
                i.ProductName,
                i.Quantity,
                i.UnitPrice,
                i.TotalPrice
            )).ToList()
        )).ToList();
    }
}
```

## Create Module Extensions

Create `Nimble.Modulith.Customers/CustomersModuleExtensions.cs`:

```csharp
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Logging;
using Nimble.Modulith.Customers.Domain.Interfaces;
using Nimble.Modulith.Customers.Infrastructure.Data;

namespace Nimble.Modulith.Customers;

public static class CustomersModuleExtensions
{
    public static IHostApplicationBuilder AddCustomersModuleServices(this IHostApplicationBuilder builder, ILogger logger)
    {
        // Register the DbContext with Aspire
        builder.AddSqlServerDbContext<CustomersDbContext>("customersdb");

        // Register repositories
        builder.Services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>));
        builder.Services.AddScoped(typeof(IReadRepository<>), typeof(EfReadRepository<>));

        return builder;
    }

    public static async Task<WebApplication> EnsureCustomersModuleDatabaseAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<CustomersDbContext>();
        await context.Database.MigrateAsync();
        return app;
    }
}
```

## Update Web Project

### 1. Update Program.cs

Open `Nimble.Modulith.Web/Program.cs` and add the Customers module registration after the Users and Products modules:

```csharp
using Nimble.Modulith.Users;
using Nimble.Modulith.Products;
using Nimble.Modulith.Customers;
using Serilog;
using Mediator;

var logger = Log.Logger = new LoggerConfiguration()
  .Enrich.FromLogContext()
  .WriteTo.Console()
  .CreateLogger();

logger.Information("Starting web host");

var builder = WebApplication.CreateBuilder(args);
builder.Host.UseSerilog((_, config) => config.ReadFrom.Configuration(builder.Configuration));

// Add service defaults (Aspire configuration)
builder.AddServiceDefaults();

// Add Mediator with source generation
builder.Services.AddMediator(options =>
{
    options.ServiceLifetime = ServiceLifetime.Scoped;
});

// Add FastEndpoints with JWT Bearer Authentication and Authorization
builder.Services.AddFastEndpoints()
  .AddAuthenticationJwtBearer(o => o.SigningKey = builder.Configuration["Auth:JwtSecret"]!)
  .AddAuthorization();

// Add Swagger/OpenAPI
builder.Services.SwaggerDocument(o =>
{
    o.DocumentSettings = s =>
    {
        s.Title = "Nimble Modulith API";
        s.Version = "v1";
    };
});

// Register modules using IHostApplicationBuilder pattern
builder.AddUsersModuleServices(logger);
builder.AddProductsModuleServices(logger);
builder.AddCustomersModuleServices(logger);

var app = builder.Build();

// Configure the HTTP request pipeline
app.UseDefaultExceptionHandler();
app.UseAuthentication();
app.UseAuthorization();

// Configure FastEndpoints
app.UseFastEndpoints(config =>
{
    config.Endpoints.RoutePrefix = string.Empty;
});

// Only use Swagger in Development
if (app.Environment.IsDevelopment())
{
    app.UseSwaggerGen();
}

app.MapDefaultEndpoints();

// Ensure databases are created and migrated
await app.EnsureUsersModuleDatabaseAsync();
await app.EnsureProductsModuleDatabaseAsync();
await app.EnsureCustomersModuleDatabaseAsync();

app.Run();
```

## Create and Apply Database Migration

From the solution directory, run the following commands to create and apply the database migration:

```bash
# Navigate to the Customers project
cd Nimble.Modulith.Customers

# Create the initial migration
dotnet ef migrations add InitialCreate --context CustomersDbContext --output-dir Data/Migrations

# Return to solution directory
cd ..

# Run the application to apply migrations
dotnet run --project Nimble.Modulith.AppHost/Nimble.Modulith.AppHost.csproj
```

When the application starts, the migrations will be applied automatically if you've configured automatic migrations in your Web project's startup.

## Test the Customers Module

### 1. Update the HTTP File

Add the following to `Nimble.Modulith.Web/Nimble.Modulith.Web.http`:

```http
### Customers Module Tests

### Create a Customer
POST {{Nimble.Modulith.Web_HostAddress}}/customers
Content-Type: application/json
{
  "firstName": "John",
  "lastName": "Doe",
  "email": "john.doe@example.com",
  "phoneNumber": "+1-555-0123",
  "address": {
    "street": "123 Main St",
    "city": "New York",
    "state": "NY",
    "postalCode": "10001",
    "country": "USA"
  }
}

### List All Customers
GET {{Nimble.Modulith.Web_HostAddress}}/customers

### Get Customer by ID
GET {{Nimble.Modulith.Web_HostAddress}}/customers/1

### Orders Module Tests

### Create an Order
POST {{Nimble.Modulith.Web_HostAddress}}/orders
Content-Type: application/json
{
  "customerId": 1,
  "orderDate": "2025-10-24",
  "items": [
    {
      "productId": 1,
      "productName": "Sample Product",
      "quantity": 2,
      "unitPrice": 19.99
    },
    {
      "productId": 2,
      "productName": "Another Product",
      "quantity": 1,
      "unitPrice": 29.99
    }
  ]
}

### Add Item to Order
POST {{Nimble.Modulith.Web_HostAddress}}/orders/1/items
Content-Type: application/json
{
  "productId": 3,
  "productName": "Third Product",
  "quantity": 1,
  "unitPrice": 15.99
}

### Delete Item from Order
DELETE {{Nimble.Modulith.Web_HostAddress}}/orders/1/items/2

### Get Order by ID
GET {{Nimble.Modulith.Web_HostAddress}}/orders/1

### List Orders by Date
GET {{Nimble.Modulith.Web_HostAddress}}/orders/by-date/2025-10-24

### Update Order Status
PATCH {{Nimble.Modulith.Web_HostAddress}}/orders/1/status
Content-Type: application/json
{
  "status": "Processing"
}
```

## Key Points

### Clean Architecture Benefits

- **Separation of Concerns**: Domain logic is isolated from infrastructure and presentation concerns
- **Testability**: Business logic can be tested independently without dependencies on databases or frameworks
- **Maintainability**: Changes to infrastructure don't affect domain logic
- **Flexibility**: Easy to swap out data access technology or add new presentation layers

### Design Patterns Used

- **CQRS with Mediator**: Commands and Queries separated, communicated via Mediator pattern
- **Repository Pattern**: Abstracts data access logic and provides a collection-like interface
- **Specification Pattern**: Encapsulates query logic in reusable, composable specifications
- **Value Objects**: Address is modeled as an owned entity with no independent identity
- **Result Pattern**: Using Ardalis.Result to handle success/failure without exceptions

### Three Distinct DTO Types

The module demonstrates proper DTO separation:

1. **Details (Contracts project)**: For cross-module communication
   - Example: `CustomerDetails`, `OrderDetails`
   - Purpose: Shared types when other modules need Customer/Order data
   
2. **UseCases DTOs (UseCases folder)**: Internal use within handlers
   - Example: `CustomerDto`, `OrderDto` in UseCases
   - Purpose: Include all internal fields (CreatedAt, UpdatedAt, etc.)
   - Returned in `Result<T>` from handlers
   
3. **Endpoint Response DTOs (Endpoints folder)**: Clean API responses
   - Example: `CustomerResponse`, `OrderResponse`
   - Purpose: Public-facing API, excludes internal tracking fields
   - Mapped from UseCases DTOs before returning to clients

### Data Flow

```
HTTP Request → Endpoint → Command/Query → Mediator → Handler
                                                        ↓
                                                    DbContext
                                                        ↓
                                            Result<UseCasesDto>
                                                        ↓
                                        Map to EndpointResponseDto
                                                        ↓
                                                  HTTP Response
```

### What We've Created

The Customers module now has:
- **Customer Management**: Create, list, and retrieve customers with addresses
- **Order Management**: Create orders, manage order items, and query orders by date
- **Clean Architecture**: Proper separation using folders for domain, use cases, infrastructure, and endpoints
- **Mediator Pattern**: Decoupled communication between endpoints and business logic
- **Rich Domain Models**: Entities with behavior, not just data containers
- **Type Safety**: Three distinct DTO types for different concerns

### Module Features

Customer endpoints:
- **POST /customers** - Create a new customer
- **GET /customers** - List all customers
- **GET /customers/{id}** - Get a specific customer

Order endpoints:
- **POST /orders** - Create a new order with items and a specific order date
- **POST /orders/{id}/items** - Add an item to an existing order
- **DELETE /orders/{id}/items/{itemId}** - Remove an item from an order
- **GET /orders/{id}** - Get a specific order with all its items
- **GET /orders/by-date/{date}** - Get all orders created on a specific date (format: YYYY-MM-DD)

## Congratulations!

You've successfully created a sophisticated Customers and Orders module using Clean Architecture principles. This module demonstrates:
- Folder-based organization with clear separation of concerns
- CQRS pattern with commands, queries, and Mediator
- Three distinct DTO types serving different purposes
- Result pattern for error handling without exceptions
- Domain-driven design with rich domain models
- Repository and specification patterns for data access
- Value objects for type safety
- Complex business logic with order aggregates and item management
- DateOnly type for proper date handling
- **Encapsulated collections** with private backing fields and read-only exposure
- **Guard clauses** to validate domain operations
- **Business rules in the domain** such as combining quantities when the same product is added to an order
- **Calculated properties** like TotalAmount that are always accurate

This architecture scales well and provides a solid foundation for adding more complex business rules and domain logic as your application grows.

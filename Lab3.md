# Lab 3: Add a second module using simple CRUD for Products

In this lab, we will add a second module to our modular monolith application for managing Products. This module will follow a similar structure to the Users module, with a main project and a contracts project, but will focus on simple CRUD operations.

When we're done, we will have added two new projects to our solution:

- Products (main module with endpoints and data access)
- Products.Contracts (shared contracts for the module)

## Adding the Products Module

You can run all the following commands at once from the solution directory. This script will:

1. Create the Products.Contracts class library
2. Create the Products class library
3. Add both projects to the solution in a "Products Module" solution folder
4. Add a reference from Products to Products.Contracts
5. Install the necessary NuGet packages (FastEndpoints, Serilog, EF Core, and Aspire SQL Server)

```bash
# Create the Products.Contracts project
dotnet new classlib -n Nimble.Modulith.Products.Contracts -o Nimble.Modulith.Products.Contracts

# Create the Products project
dotnet new classlib -n Nimble.Modulith.Products -o Nimble.Modulith.Products

# Add both projects to the solution in a "Products Module" solution folder
dotnet sln add Nimble.Modulith.Products.Contracts/Nimble.Modulith.Products.Contracts.csproj --solution-folder "Products Module"
dotnet sln add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj --solution-folder "Products Module"

# Add reference from Products to Products.Contracts
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj reference Nimble.Modulith.Products.Contracts/Nimble.Modulith.Products.Contracts.csproj

# Add Products project reference to Web project
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj reference Nimble.Modulith.Products/Nimble.Modulith.Products.csproj

# Add NuGet packages to the Products project
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package FastEndpoints
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package Serilog.AspNetCore
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package Microsoft.EntityFrameworkCore
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package Microsoft.EntityFrameworkCore.SqlServer
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package Aspire.Microsoft.EntityFrameworkCore.SqlServer

# Build the solution to verify everything is set up correctly
dotnet build
```

## Configure the Products Database

Now we need to configure a separate database for the Products module and wire it up through the Aspire AppHost.

### 1. Add the Products Database to AppHost

Open `Nimble.Modulith.AppHost/AppHost.cs` and add a new database for products. Update the file to include:

```csharp
var builder = DistributedApplication.CreateBuilder(args);

// Add SQL Server database for Identity
var sqlServer = builder.AddSqlServer("sqlserver")
    .WithDataVolume();

var usersDb = sqlServer.AddDatabase("usersdb");
var productsDb = sqlServer.AddDatabase("productsdb");

// Add the Web API project with database reference
builder.AddProject<Projects.Nimble_Modulith_Web>("webapi")
    .WithReference(usersDb)
    .WithReference(productsDb)
    .WaitFor(usersDb)
    .WaitFor(productsDb);

builder.Build().Run();
```

This creates a `productsdb` database on the same SQL Server instance and passes its connection string to the web project.

### 2. Create the ProductsDbContext

Create a new file `Nimble.Modulith.Products/Data/ProductsDbContext.cs`:

```csharp
using Microsoft.EntityFrameworkCore;

namespace Nimble.Modulith.Products.Data;

public class ProductsDbContext : DbContext
{
    public ProductsDbContext(DbContextOptions<ProductsDbContext> options)
        : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Apply Products module specific configurations
        builder.HasDefaultSchema("Products");
    }
}
```

This creates a DbContext specifically for the Products module with its own schema.

### 3. Create ProductsModuleExtensions

Create a new file `Nimble.Modulith.Products/ProductsModuleExtensions.cs`:

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Nimble.Modulith.Products.Data;
using Serilog;

namespace Nimble.Modulith.Products;

public static class ProductsModuleExtensions
{
    public static IHostApplicationBuilder AddProductsModuleServices(
        this IHostApplicationBuilder builder,
        ILogger logger)
    {
        // Add SQL Server DbContext with Aspire
        builder.AddSqlServerDbContext<ProductsDbContext>("productsdb");

        logger.Information("{Module} module services registered", nameof(ProductsModuleExtensions).Replace("ModuleExtensions", ""));

        return builder;
    }

    public static async Task<WebApplication> EnsureProductsModuleDatabaseAsync(this WebApplication app)
    {
        using var scope = app.Services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<ProductsDbContext>();
        await context.Database.EnsureCreatedAsync();
        
        return app;
    }
}
```

This extension class provides methods to register the Products module services and ensure its database is created.

### 4. Update Program.cs to Wire Up Products Module

Update the using statements at the top of `Program.cs` to include:

```csharp
using Nimble.Modulith.Products;
```

Add the Products module services registration after the Users module:

```csharp
// Add module services
builder.AddUsersModuleServices(logger);
builder.AddProductsModuleServices(logger);
```

Add the Products database initialization after the Users database:

```csharp
// Ensure module databases are created
await app.EnsureUsersModuleDatabaseAsync();
await app.EnsureProductsModuleDatabaseAsync();
```

### 5. Add Products Project Reference to Web Project

Add a reference to the Products project from the Web project:

```bash
dotnet add Nimble.Modulith.Web/Nimble.Modulith.Web.csproj reference Nimble.Modulith.Products/Nimble.Modulith.Products.csproj
```

### 6. Build and Verify

Build the solution to ensure everything is configured correctly:

```bash
dotnet build
```

## Create the Product Entity

Now let's create the Product entity model that will represent products in our database.

### 1. Create the Product Model

Create a new file `Nimble.Modulith.Products/Models/Product.cs`:

```csharp
namespace Nimble.Modulith.Products.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime DateCreated { get; set; }
    public string CreatedByUser { get; set; } = string.Empty;
}
```

This creates a basic Product entity with:
- `Id`: Primary key
- `Name`: Product name
- `Description`: Product description
- `DateCreated`: When the product was created (UTC)
- `CreatedByUser`: Username of who created the product

### 2. Add DbSet to ProductsDbContext

Update `Nimble.Modulith.Products/Data/ProductsDbContext.cs` to include a DbSet for Products:

```csharp
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Models;

namespace Nimble.Modulith.Products.Data;

public class ProductsDbContext : DbContext
{
    public ProductsDbContext(DbContextOptions<ProductsDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);
        
        // Apply Products module specific configurations
        builder.HasDefaultSchema("Products");
        
        // Apply all configurations from this assembly
        builder.ApplyConfigurationsFromAssembly(typeof(ProductsDbContext).Assembly);
    }
}
```

### 3. Create Entity Configuration

Create a new file `Nimble.Modulith.Products/Data/Config/ProductConfiguration.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Nimble.Modulith.Products.Models;

namespace Nimble.Modulith.Products.Data.Config;

public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.HasKey(p => p.Id);

        builder.Property(p => p.Name)
            .IsRequired()
            .HasMaxLength(200);

        builder.Property(p => p.Description)
            .HasMaxLength(1000);

        builder.Property(p => p.DateCreated)
            .IsRequired();

        builder.Property(p => p.CreatedByUser)
            .IsRequired()
            .HasMaxLength(100);
    }
}
```

This configuration:
- Sets `Id` as the primary key
- Makes `Name` required with a max length of 200 characters
- Limits `Description` to 1000 characters
- Makes `DateCreated` required
- Makes `CreatedByUser` required with a max length of 100 characters

### 4. Build and Verify

Build the solution to ensure everything compiles:

```bash
dotnet build
```

## Add Database Migrations

Now we'll add EF Core migrations to manage the database schema.

### 1. Add Design-Time DbContext Factory

Create a new file `Nimble.Modulith.Products/Data/ProductsDbContextFactory.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace Nimble.Modulith.Products.Data;

public class ProductsDbContextFactory : IDesignTimeDbContextFactory<ProductsDbContext>
{
    public ProductsDbContext CreateDbContext(string[] args)
    {
        var optionsBuilder = new DbContextOptionsBuilder<ProductsDbContext>();
        
        // This is only used for design-time operations (migrations)
        // The actual connection string comes from Aspire at runtime
        optionsBuilder.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=ProductsDb;Trusted_Connection=True;");
        
        return new ProductsDbContext(optionsBuilder.Options);
    }
}
```

This factory allows EF Core tools to create a DbContext instance at design time for migrations.

### 2. Add EF Core Design Package

Add the `Microsoft.EntityFrameworkCore.Design` package to the Products project:

```bash
dotnet add Nimble.Modulith.Products/Nimble.Modulith.Products.csproj package Microsoft.EntityFrameworkCore.Design
```

### 3. Create Initial Migration

Create the initial migration for the Products database:

```bash
dotnet ef migrations add InitialCreate --project Nimble.Modulith.Products --context ProductsDbContext --output-dir Data/Migrations
```

This will create migration files in the `Data/Migrations` folder with the schema for the Products table.

### 4. Update ProductsModuleExtensions to Use Migrations

Update `Nimble.Modulith.Products/ProductsModuleExtensions.cs` to run migrations instead of using `EnsureCreatedAsync`:

```csharp
public static async Task<WebApplication> EnsureProductsModuleDatabaseAsync(this WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ProductsDbContext>();
    await context.Database.MigrateAsync();
    
    return app;
}
```

Change `EnsureCreatedAsync()` to `MigrateAsync()`. This will automatically apply any pending migrations when the application starts.

### 5. Build and Verify

Build the solution to ensure everything works:

```bash
dotnet build
```

## Create CRUD Endpoints

Now let's create FastEndpoints for all CRUD operations on products. We'll use primary constructor injection to access the `ProductsDbContext`.

### 1. Create the Create Endpoint

Create `Nimble.Modulith.Products/Endpoints/Create.cs`:

```csharp
using FastEndpoints;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Data;
using Nimble.Modulith.Products.Models;

namespace Nimble.Modulith.Products.Endpoints;

public class CreateProductRequest
{
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
}

public class CreateProductResponse
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime DateCreated { get; set; }
    public string CreatedByUser { get; set; } = string.Empty;
}

public class Create(ProductsDbContext dbContext) : Endpoint<CreateProductRequest, CreateProductResponse>
{
    private readonly ProductsDbContext _dbContext = dbContext;

    public override void Configure()
    {
        Post("/products");
        Tags("products");
        Summary(s =>
        {
            s.Summary = "Create a new product";
            s.Description = "Creates a new product with a name and description";
        });
    }

    public override async Task HandleAsync(CreateProductRequest req, CancellationToken ct)
    {
        var product = new Product
        {
            Name = req.Name,
            Description = req.Description,
            DateCreated = DateTime.UtcNow,
            CreatedByUser = User.Identity?.Name ?? "Anonymous"
        };

        _dbContext.Products.Add(product);
        await _dbContext.SaveChangesAsync(ct);

        Response = new CreateProductResponse
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            DateCreated = product.DateCreated,
            CreatedByUser = product.CreatedByUser
        };
    }
}
```

### 2. Create the List Endpoint

Create `Nimble.Modulith.Products/Endpoints/List.cs`:

```csharp
using FastEndpoints;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Data;

namespace Nimble.Modulith.Products.Endpoints;

public class ProductListItem
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime DateCreated { get; set; }
    public string CreatedByUser { get; set; } = string.Empty;
}

public class ListResponse
{
    public List<ProductListItem> Products { get; set; } = new();
}

public class List(ProductsDbContext dbContext) : EndpointWithoutRequest<ListResponse>
{
    private readonly ProductsDbContext _dbContext = dbContext;

    public override void Configure()
    {
        Get("/products");
        Tags("products");
        Summary(s =>
        {
            s.Summary = "List all products";
            s.Description = "Retrieves a list of all products in the system";
        });
    }

    public override async Task HandleAsync(CancellationToken ct)
    {
        var products = await _dbContext.Products
            .Select(p => new ProductListItem
            {
                Id = p.Id,
                Name = p.Name,
                Description = p.Description,
                DateCreated = p.DateCreated,
                CreatedByUser = p.CreatedByUser
            })
            .ToListAsync(ct);

        Response = new ListResponse
        {
            Products = products
        };
    }
}
```

### 3. Create the GetById Endpoint

Create `Nimble.Modulith.Products/Endpoints/GetById.cs`:

```csharp
using FastEndpoints;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Data;

namespace Nimble.Modulith.Products.Endpoints;

public class GetByIdRequest
{
    public int Id { get; set; }
}

public class GetByIdResponse
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime DateCreated { get; set; }
    public string CreatedByUser { get; set; } = string.Empty;
}

public class GetById(ProductsDbContext dbContext) : Endpoint<GetByIdRequest, GetByIdResponse>
{
    private readonly ProductsDbContext _dbContext = dbContext;

    public override void Configure()
    {
        Get("/products/{id}");
        Tags("products");
        Summary(s =>
        {
            s.Summary = "Get a product by ID";
            s.Description = "Retrieves a single product by its ID";
        });
    }

    public override async Task HandleAsync(GetByIdRequest req, CancellationToken ct)
    {
        var product = await _dbContext.Products
            .FirstOrDefaultAsync(p => p.Id == req.Id, ct);

        if (product is null)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        Response = new GetByIdResponse
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            DateCreated = product.DateCreated,
            CreatedByUser = product.CreatedByUser
        };
    }
}
```

### 4. Create the Update Endpoint

Create `Nimble.Modulith.Products/Endpoints/Update.cs`:

```csharp
using FastEndpoints;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Data;

namespace Nimble.Modulith.Products.Endpoints;

public class UpdateProductRequest
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
}

public class UpdateProductResponse
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public string Description { get; set; } = string.Empty;
    public DateTime DateCreated { get; set; }
    public string CreatedByUser { get; set; } = string.Empty;
}

public class Update(ProductsDbContext dbContext) : Endpoint<UpdateProductRequest, UpdateProductResponse>
{
    private readonly ProductsDbContext _dbContext = dbContext;

    public override void Configure()
    {
        Put("/products/{id}");
        Tags("products");
        Summary(s =>
        {
            s.Summary = "Update a product";
            s.Description = "Updates an existing product's name and description";
        });
    }

    public override async Task HandleAsync(UpdateProductRequest req, CancellationToken ct)
    {
        var product = await _dbContext.Products
            .FirstOrDefaultAsync(p => p.Id == req.Id, ct);

        if (product is null)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        product.Name = req.Name;
        product.Description = req.Description;

        await _dbContext.SaveChangesAsync(ct);

        Response = new UpdateProductResponse
        {
            Id = product.Id,
            Name = product.Name,
            Description = product.Description,
            DateCreated = product.DateCreated,
            CreatedByUser = product.CreatedByUser
        };
    }
}
```

### 5. Create the Delete Endpoint

Create `Nimble.Modulith.Products/Endpoints/Delete.cs`:

```csharp
using FastEndpoints;
using Microsoft.EntityFrameworkCore;
using Nimble.Modulith.Products.Data;

namespace Nimble.Modulith.Products.Endpoints;

public class DeleteProductRequest
{
    public int Id { get; set; }
}

public class Delete(ProductsDbContext dbContext) : Endpoint<DeleteProductRequest>
{
    private readonly ProductsDbContext _dbContext = dbContext;

    public override void Configure()
    {
        Delete("/products/{id}");
        Tags("products");
        Summary(s =>
        {
            s.Summary = "Delete a product";
            s.Description = "Deletes a product by its ID";
        });
    }

    public override async Task HandleAsync(DeleteProductRequest req, CancellationToken ct)
    {
        var product = await _dbContext.Products
            .FirstOrDefaultAsync(p => p.Id == req.Id, ct);

        if (product is null)
        {
            await Send.NotFoundAsync(ct);
            return;
        }

        _dbContext.Products.Remove(product);
        await _dbContext.SaveChangesAsync(ct);

        await Send.NoContentAsync(ct);
    }
}
```

### 6. Build and Test

Build the solution to ensure all endpoints compile:

```bash
dotnet build
```

### 7. HTTP File Updates

If you're using the `Nimble.Modulith.Web.http` file for your testing, add this to it:

```
### List Products

GET {{Nimble.Modulith.Web_HostAddress}}/products

### Create a Product (if logged in)
POST {{Nimble.Modulith.Web_HostAddress}}/products
Content-Type: application/json
{
  "name": "Sample Product",
  "description": "This is a sample product.",
  "price": 19.99
}

### View a Product by ID
GET {{Nimble.Modulith.Web_HostAddress}}/products/1

### Update a Product by ID (if logged in)
PUT {{Nimble.Modulith.Web_HostAddress}}/products/1
Content-Type: application/json
{
  "name": "Updated Product",
  "description": "This is an updated product description.",
  "price": 24.99
}

### Delete a Product by ID (if logged in)
DELETE {{Nimble.Modulith.Web_HostAddress}}/products/1

```

## Key Points

### Endpoint Features

- **Primary Constructor Injection**: All endpoints use C# primary constructors to inject `ProductsDbContext` and assign it to a readonly field
- **FastEndpoints v7.x**: Uses the `Response` property to return data and `Send.*` methods for status codes
- **Swagger Documentation**: All endpoints are tagged as "products" and include summary descriptions
- **Authentication**: Endpoints require authentication by default (no `AllowAnonymous()`)
- **Route Patterns**: Routes are at the root level (e.g., `/products`) without `/api/` prefix
- **Automatic Discovery**: FastEndpoints automatically discovers and registers all endpoints from the Products assembly

### What We've Created

The Products module now has a complete CRUD API:
- **POST /products** - Create a new product
- **GET /products** - List all products
- **GET /products/{id}** - Get a specific product
- **PUT /products/{id}** - Update a product
- **DELETE /products/{id}** - Delete a product

All endpoints are grouped under the "products" tag in Swagger UI for easy discovery and testing.

## Congratulations!

You've successfully created a complete Products module with:
- Database configuration and migrations
- Entity model with EF Core configuration
- Full CRUD API endpoints
- Swagger documentation

The Products module follows the same modular pattern as the Users module, demonstrating how to add additional modules to your modular monolith application.




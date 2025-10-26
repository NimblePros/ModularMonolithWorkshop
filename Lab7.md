# Lab 7: Building a Reporting Module with Star Schema

## Overview
This lab builds a comprehensive reporting module that ingests confirmed orders from the Customers module and provides analytical queries using a star schema design.

## Architecture

### Star Schema Design
The reporting database uses a classic star schema with:

**Dimension Tables:**
- `DimDate` - Date dimension with year, quarter, month, day attributes
- `DimCustomer` - Customer information (email, name)
- `DimProduct` - Product catalog (id, name)

**Fact Table:**
- `FactOrders` - Central fact table storing order metrics
  - One row per order item
  - Foreign keys to all dimension tables
  - Measures: quantity, unit price, total price, order total amount
  - Degenerate dimensions: order number, order item ID

### Event-Driven Data Ingestion
```
Customers Module → OrderCreatedEvent → Reporting Module → Star Schema DB
```

1. **Order Confirmation**: When `ConfirmOrderCommand` is executed, it publishes `OrderCreatedEvent`
2. **Event Handling**: `OrderCreatedEventHandler` in Reporting module ingests the data
3. **Dimension Management**: Automatically ensures dimension records exist
4. **Fact Insertion**: Inserts one row per order item into FactOrders
5. **Idempotency**: Prevents duplicate ingestion using unique index on (OrderId, OrderItemId)

## Key Changes to Customers Module

### Order Domain Changes
1. **New OrderStatus**: Added `Confirmed` status between `Pending` and `Processing`
2. **Immutable Confirmed Orders**: `AddItem()` and `RemoveItem()` now throw exceptions if order is not `Pending`
3. **Event Publication**: `ConfirmOrderHandler` now publishes `OrderCreatedEvent` after confirmation

```csharp
public enum OrderStatus
{
    Pending = 0,      // Can add/remove items
    Confirmed = 1,    // Locked for reporting - NO changes allowed
    Processing = 2,
    Shipped = 3,
    Delivered = 4,
    Cancelled = 5
}
```

## Reporting Module Structure

```
Nimble.Modulith.Reporting/
├── Data/
│   ├── ReportingDbContext.cs
│   └── Config/
│       ├── DimDateConfig.cs
│       ├── DimCustomerConfig.cs
│       ├── DimProductConfig.cs
│       └── FactOrderConfig.cs
├── Models/
│   ├── DimDate.cs
│   ├── DimCustomer.cs
│   ├── DimProduct.cs
│   └── FactOrder.cs
├── Ingest/
│   └── OrderCreatedEventHandler.cs
├── Services/
│   ├── IReportService.cs
│   ├── ReportService.cs (using Dapper)
│   └── ReportModels.cs
├── Endpoints/
│   ├── CsvFormatter.cs
│   └── Reports/
│       ├── OrdersReport.cs
│       ├── ProductSalesReport.cs
│       └── CustomerOrdersReport.cs
└── ReportingModuleExtensions.cs
```

## Technology Choices

### EF Core for Schema Management
- Consistent with other modules
- Type-safe entity configurations
- Easy migrations
- `IEntityTypeConfiguration<T>` pattern for organization

### Dapper for Reporting Queries
- Better performance for read-only queries
- Direct SQL control for complex aggregations
- Minimal overhead

## Entity Configuration Details

### Critical: Disable Identity Generation

All dimension table primary keys use `ValueGeneratedNever()` because we control the IDs from source systems:

**DimCustomer Configuration:**
```csharp
public class DimCustomerConfig : IEntityTypeConfiguration<DimCustomer>
{
    public void Configure(EntityTypeBuilder<DimCustomer> builder)
    {
        builder.ToTable("DimCustomer");
        builder.HasKey(c => c.CustomerId);
        
        // Do NOT use identity - we control the IDs from source systems
        builder.Property(c => c.CustomerId)
            .ValueGeneratedNever();
        
        // ... other configurations
    }
}
```

**DimProduct Configuration:**
```csharp
builder.Property(p => p.ProductId)
    .ValueGeneratedNever();
```

**DimDate Configuration:**
```csharp
builder.Property(d => d.DateKey)
    .ValueGeneratedNever(); // DateKey is YYYYMMDD format
```

Without `ValueGeneratedNever()`, EF Core creates identity columns, causing "cannot insert explicit value for identity column" errors.

### Seed Data: DimDate for 2025

The `DimDateConfig` seeds all dates for 2025 using `HasData()`:

```csharp
public class DimDateConfig : IEntityTypeConfiguration<DimDate>
{
    public void Configure(EntityTypeBuilder<DimDate> builder)
    {
        // ... configuration code ...
        
        // Seed all dates for 2025
        var dates = GenerateDateDimension(2025);
        builder.HasData(dates);
    }
    
    private static List<DimDate> GenerateDateDimension(int year)
    {
        var dates = new List<DimDate>();
        var startDate = new DateTime(year, 1, 1);
        var endDate = new DateTime(year, 12, 31);
        
        for (var date = startDate; date <= endDate; date = date.AddDays(1))
        {
            dates.Add(new DimDate
            {
                DateKey = int.Parse(date.ToString("yyyyMMdd")),
                Date = date,
                Year = date.Year,
                Quarter = (date.Month - 1) / 3 + 1,
                Month = date.Month,
                Day = date.Day,
                DayOfWeek = (int)date.DayOfWeek,
                DayName = date.DayOfWeek.ToString(),
                MonthName = date.ToString("MMMM")
            });
        }
        
        return dates;
    }
}
```

This ensures 365 dates are automatically populated when the database is created.

## API Endpoints

### 1. Orders Report
```
GET /reports/orders?startDate=2025-01-01&endDate=2025-12-31&format=json|csv
```

Returns:
- List of orders with totals
- Summary statistics (total orders, revenue, average order value)
- Supports JSON (default) and CSV output

### 2. Product Sales Report
```
GET /reports/product-sales?startDate=2025-01-01&endDate=2025-12-31&format=json|csv
```

Returns:
- Product sales ranked by revenue
- Total quantity sold per product
- Order count per product

### 3. Customer Orders Report
```
GET /reports/customers/{customerId}/orders?format=json|csv
```

Returns:
- All orders for a specific customer
- Customer lifetime metrics (total spent, first/last order dates)

## CSV Export Feature

Reports support CSV export via two mechanisms:

1. **Query parameter**: `?format=csv`
2. **Accept header**: `Accept: text/csv`

The CSV formatter properly escapes special characters and quotes fields containing commas or newlines.

## Database Setup (Aspire)

The `reportingdb` database is configured in the AppHost:

```csharp
var reportingDb = sqlServer.AddDatabase("reportingdb");

var webapi = builder.AddProject<Projects.Nimble_Modulith_Web>("webapi")
    .WithReference(reportingDb)
    .WaitFor(reportingDb);
```

The database is automatically created on first access using `EnsureCreatedAsync()` in the `ReportingModuleExtensions`:

```csharp
public static async Task<WebApplication> EnsureReportingModuleDatabaseAsync(this WebApplication app)
{
    using var scope = app.Services.CreateScope();
    var context = scope.ServiceProvider.GetRequiredService<ReportingDbContext>();
    var logger = scope.ServiceProvider.GetRequiredService<ILogger<ReportingDbContext>>();
    var env = scope.ServiceProvider.GetRequiredService<IHostEnvironment>();
    
    try
    {
        logger.LogInformation("Ensuring reporting database exists...");
        
        // In development, drop and recreate to ensure seed data is applied
        if (env.IsDevelopment())
        {
            logger.LogInformation("Development mode: Dropping and recreating reporting database to ensure seed data...");
            await context.Database.EnsureDeletedAsync();
            var created = await context.Database.EnsureCreatedAsync();
            if (created)
            {
                logger.LogInformation("Reporting database recreated successfully with {DateCount} seed dates", 365);
            }
        }
        else
        {
            // In production, use migrations
            // await context.Database.MigrateAsync();
            await context.Database.EnsureCreatedAsync();
            logger.LogInformation("Reporting database ensured");
        }
    }
    catch (Exception ex)
    {
        logger.LogError(ex, "Failed to ensure reporting database exists");
        throw;
    }
    
    return app;
}
```

**Important Notes:**
- **Development**: Database is dropped and recreated on every startup to ensure seed data (365 dates) is applied
- **Production**: Should use EF Core migrations with `MigrateAsync()` for version control and deployment tracking
- **Seed Data**: The 365 dates for 2025 are automatically inserted via `HasData()` in `DimDateConfig`

## Event Handler Implementation

The `OrderCreatedEventHandler` demonstrates best practices:

1. **Automatic Dimension Management**: Creates dimension records if they don't exist
2. **Idempotency**: Checks for existing fact records before inserting
3. **Transaction Scope**: All dimension and fact inserts in single transaction
4. **Error Logging**: Comprehensive logging with Serilog
5. **DateKey Conversion**: Converts DateOnly to integer key (YYYYMMDD format)

```csharp
private static int ConvertToDateKey(DateOnly date)
{
    return date.Year * 10000 + date.Month * 100 + date.Day;
}
```

## Benefits of This Architecture

1. **Separation of Concerns**: Reporting doesn't impact transactional database
2. **Optimized for Analytics**: Star schema enables fast aggregations
3. **Historical Tracking**: Dimension tables track changes over time
4. **Scalability**: Can add indexes and materialized views without affecting OLTP
5. **Flexibility**: Easy to add new dimensions or metrics
6. **Event-Driven**: Decoupled ingestion via domain events

## Testing the Flow

1. **Create an Order**: `POST /orders`
2. **Add Items**: `POST /orders/{id}/items`
3. **Confirm Order**: `POST /orders/{id}/confirm` (triggers event)
4. **View Reports**: `GET /reports/orders?startDate=...&endDate=...`
5. **Export CSV**: Add `&format=csv` or set `Accept: text/csv` header

## Future Enhancements

1. **More Dimensions**: Add DimTime for hour-level analysis
2. **Slowly Changing Dimensions (SCD)**: Track historical changes to products/customers
3. **Aggregate Tables**: Pre-calculate daily/monthly summaries
4. **Real-time Dashboards**: Add SignalR for live updates
5. **Data Warehouse**: Extend to full ETL pipeline with data quality checks

## Key Learnings

1. **Star Schema Design**: Effective for OLAP workloads
2. **Event-Driven Architecture**: Clean decoupling between modules
3. **Dapper + EF Core**: Best of both worlds (write with EF, read with Dapper)
4. **Flexible Output Formats**: Easy to support multiple content types
5. **Domain Invariants**: Preventing changes to confirmed orders maintains data integrity
6. **ValueGeneratedNever()**: Critical when dimension keys come from source systems, not auto-generated
7. **Seed Data**: Use `HasData()` for reference data like date dimensions
8. **Development vs Production**: Drop/recreate in dev for fresh seed data, use migrations in production

## Sample HTTP Requests

Add these to your `Nimble.Modulith.Web.http` file to test the reporting endpoints:

```http
### Reporting Module Tests

### Get Orders Report (JSON)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/orders?startDate=2025-10-01&endDate=2025-10-31

### Get Orders Report (CSV via query parameter)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/orders?startDate=2025-10-01&endDate=2025-10-31&format=csv

### Get Orders Report (CSV via Accept header)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/orders?startDate=2025-10-01&endDate=2025-10-31
Accept: text/csv

### Get Product Sales Report (JSON)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/product-sales?startDate=2025-10-01&endDate=2025-10-31

### Get Product Sales Report (CSV)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/product-sales?startDate=2025-10-01&endDate=2025-10-31&format=csv

### Get Customer Orders Report (JSON)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/customers/1/orders

### Get Customer Orders Report (CSV)
GET {{Nimble.Modulith.Web_HostAddress}}/reports/customers/1/orders?format=csv
```

**Note**: Make sure to define the variable in your `.http` file:
```http
@Nimble.Modulith.Web_HostAddress = http://localhost:5002
```

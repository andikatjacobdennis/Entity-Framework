### **Prerequisites**
Make sure you have:
- .NET 8 SDK
- SQL Server or a relational database

---

### **Step 1: Create a Console Application**
1. Create a new .NET 8 console project:
   ```bash
   dotnet new console -n EFCoreProductSales
   cd EFCoreProductSales
   ```

2. Install Entity Framework Core:
   ```bash
   dotnet add package Microsoft.EntityFrameworkCore
   dotnet add package Microsoft.EntityFrameworkCore.SqlServer
   dotnet add package Microsoft.EntityFrameworkCore.Tools
   dotnet add package Microsoft.EntityFrameworkCore.Proxies
   ```

---

### **Step 2: Define Models with Constraints**

We will define two main entities: **Product** and **Sale**.

#### **Product.cs**:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;
using System.Collections.Generic;

namespace EFCoreProductSales.Models
{
    public class Product
    {
        [Key] // Primary Key
        public int Id { get; set; }

        [Required] // Not Null
        [StringLength(100)]
        public string Name { get; set; }

        [Column(TypeName = "decimal(18,2)")]
        public decimal Price { get; set; }

        [Required] // Not Null
        [Range(0, int.MaxValue)]
        public int Stock { get; set; }

        [Column(TypeName = "datetime2")]
        public DateTime ManufactureDate { get; set; }

        [ConcurrencyCheck] // Used for concurrency control
        public int Version { get; set; }

        // One-to-Many Relationship: Product -> Sales
        public virtual ICollection<Sale> Sales { get; set; } = new List<Sale>();
    }
}
```

#### **Sale.cs**:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace EFCoreProductSales.Models
{
    public class Sale
    {
        [Key] // Primary Key
        public int Id { get; set; }

        [Required] // Not Null
        public int ProductId { get; set; } // Foreign Key

        [ForeignKey("ProductId")]
        public virtual Product Product { get; set; } // Many-to-One Relationship

        [Column(TypeName = "decimal(18,2)")]
        public decimal SaleAmount { get; set; }

        [Required]
        [Column(TypeName = "datetime2")]
        public DateTime SaleDate { get; set; }
    }
}
```

### **Step 3: Define Many-to-Many Relationship**

If you wanted a **Many-to-Many** relationship (e.g., between `Product` and `Supplier`), you could introduce a join table.

#### **Supplier.cs**:

```csharp
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

namespace EFCoreProductSales.Models
{
    public class Supplier
    {
        [Key] // Primary Key
        public int Id { get; set; }

        [Required] // Not Null
        [StringLength(100)]
        public string Name { get; set; }

        // Many-to-Many Relationship: Supplier -> Products
        public virtual ICollection<ProductSupplier> ProductSuppliers { get; set; } = new List<ProductSupplier>();
    }
}
```

#### **ProductSupplier.cs (Join Table)**:

```csharp
namespace EFCoreProductSales.Models
{
    public class ProductSupplier
    {
        [Required]
        public int ProductId { get; set; }

        [Required]
        public int SupplierId { get; set; }

        public Product Product { get; set; }
        public Supplier Supplier { get; set; }
    }
}
```

---

### **Step 4: Configure the DbContext with Relationships and Seeding**

In your `AppDbContext.cs`, configure relationships using Fluent API and seed initial data.

#### **AppDbContext.cs**:

```csharp
using Microsoft.EntityFrameworkCore;
using EFCoreProductSales.Models;

namespace EFCoreProductSales.Data
{
    public class AppDbContext : DbContext
    {
        public DbSet<Product> Products { get; set; }
        public DbSet<Sale> Sales { get; set; }
        public DbSet<Supplier> Suppliers { get; set; }
        public DbSet<ProductSupplier> ProductSuppliers { get; set; }

        protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.UseSqlServer("Server=(localdb)\\mssqllocaldb;Database=EFCoreProductSalesDb;Trusted_Connection=True;");
            optionsBuilder.UseLazyLoadingProxies(); // Enable Lazy Loading
        }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            // Configure Many-to-Many relationship between Product and Supplier
            modelBuilder.Entity<ProductSupplier>()
                .HasKey(ps => new { ps.ProductId, ps.SupplierId });

            modelBuilder.Entity<ProductSupplier>()
                .HasOne(ps => ps.Product)
                .WithMany(p => p.ProductSuppliers)
                .HasForeignKey(ps => ps.ProductId);

            modelBuilder.Entity<ProductSupplier>()
                .HasOne(ps => ps.Supplier)
                .WithMany(s => s.ProductSuppliers)
                .HasForeignKey(ps => ps.SupplierId);

            // Seeding initial data
            modelBuilder.Entity<Product>().HasData(
                new Product { Id = 1, Name = "Laptop", Price = 1500.00m, Stock = 50, ManufactureDate = DateTime.Parse("2022-10-01"), Version = 1 }
            );
            modelBuilder.Entity<Supplier>().HasData(
                new Supplier { Id = 1, Name = "TechSupplier" }
            );
        }
    }
}
```

---

### **Step 5: Handling Concurrency with Annotations**

Concurrency control ensures that no two users can modify the same data simultaneously.

- Use the `[ConcurrencyCheck]` annotation on the `Version` field in the `Product` entity to track changes.
- On update, EF Core will throw a `DbUpdateConcurrencyException` if another user has modified the same record.

#### **Handling Concurrency in Program.cs**:

```csharp
using EFCoreProductSales.Data;
using EFCoreProductSales.Models;
using Microsoft.EntityFrameworkCore;

class Program
{
    static async Task Main(string[] args)
    {
        using (var context = new AppDbContext())
        {
            try
            {
                var product = await context.Products.FirstOrDefaultAsync(p => p.Id == 1);

                if (product != null)
                {
                    product.Price = 1300.00m;
                    product.Version++;

                    await context.SaveChangesAsync();
                    Console.WriteLine("Product updated successfully.");
                }
            }
            catch (DbUpdateConcurrencyException)
            {
                Console.WriteLine("Concurrency conflict detected. The product was modified by another user.");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"An error occurred: {ex.Message}");
            }
        }
    }
}
```

---

### **Step 6: Migrations and Database Setup**

1. Add a migration for the initial setup:
    ```bash
    dotnet ef migrations add InitialCreate
    ```

2. Apply the migration to update the database:
    ```bash
    dotnet ef database update
    ```

---

### **Step 7: Implementing Foreign Key Relationships**

We have already established one-to-many, many-to-one, and many-to-many relationships. Here's how to load related data.

#### **Lazy Loading**:

EF Core will automatically load related `Sales` when accessing `Product.Sales`.

```csharp
var product = await context.Products.FirstOrDefaultAsync(p => p.Id == 1);
Console.WriteLine($"Product: {product.Name}, Total Sales: {product.Sales.Count}");
```

#### **Eager Loading** (using `Include()`):

You can load related data explicitly using eager loading.

```csharp
var productWithSales = await context.Products
    .Include(p => p.Sales)
    .FirstOrDefaultAsync(p => p.Id == 1);
```

#### **Explicit Loading**:

You can load related data manually:

```csharp
context.Entry(product).Collection(p => p.Sales).Load();
```

---

### **Step 8: Exception Handling**

Handle common EF Core exceptions, such as `DbUpdateException` for database issues and `DbUpdateConcurrencyException` for concurrency control.

```csharp
try
{
    // Code that modifies the database
}
catch (DbUpdateException ex)
{
    Console.WriteLine($"Database update error: {ex.Message}");
}
catch (DbUpdateConcurrencyException ex)
{
    Console.WriteLine("Concurrency conflict occurred.");
}
catch (Exception ex)
{
    Console.WriteLine($"An error occurred: {ex.Message}");
}
```

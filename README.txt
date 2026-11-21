#csproj
ChudoObuv.csproj
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>ChudoObuv</RootNamespace>
    <AssemblyName>ChudoObuv</AssemblyName>
    <ApplicationIcon>Resources\app.ico</ApplicationIcon>
    <Product>Чудо Обувь</Product>
    <BuiltInComInteropSupport>true</BuiltInComInteropSupport>
    <AvaloniaUseCompiledBindingsByDefault>true</AvaloniaUseCompiledBindingsByDefault>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.2.3" />
    <PackageReference Include="Avalonia.Desktop" Version="11.2.3" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.2.3" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Proxies" Version="8.0.11" />
    <PackageReference Include="Pomelo.EntityFrameworkCore.MySql" Version="8.0.2" />
  </ItemGroup>

  <ItemGroup>
    <!-- Логотип и иконка вшиваются в сборку (avares://) -->
    <AvaloniaResource Include="Resources\**" />
    <!-- Фото товаров лежат рядом с exe -->
    <Content Include="Images\**">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>

</Project>


#appsettings
appsettings.json
{
  "ConnectionString": "server=localhost;port=3306;database=chudo_obuv;user=root;password=1234;CharSet=utf8mb4"
}


#Program
Program.cs
using Avalonia;

namespace ChudoObuv;

internal static class Program
{
    [STAThread]
    public static void Main(string[] args) =>
        BuildAvaloniaApp().StartWithClassicDesktopLifetime(args);

    public static AppBuilder BuildAvaloniaApp() =>
        AppBuilder.Configure<App>()
            .UsePlatformDetect()
            .LogToTrace();
}


#App
App.axaml
<Application xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             x:Class="ChudoObuv.App"
             RequestedThemeVariant="Light">
    <Application.Styles>
        <FluentTheme />
    </Application.Styles>

    <Application.Resources>
        <SolidColorBrush x:Key="MainBg" Color="#FFFFFF"/>
        <SolidColorBrush x:Key="AltBg" Color="#D2F6E7"/>
        <SolidColorBrush x:Key="Accent" Color="#70B2AF"/>
        <SolidColorBrush x:Key="AccentDark" Color="#3E7C79"/>
        <SolidColorBrush x:Key="LowStockBg" Color="#FF8080"/>
        <SolidColorBrush x:Key="TextBrush" Color="#1F2D2B"/>
        <SolidColorBrush x:Key="MutedBrush" Color="#4F6B68"/>
        <SolidColorBrush x:Key="ErrorBrush" Color="#C0392B"/>
        <CornerRadius x:Key="Radius">2</CornerRadius>
    </Application.Resources>
</Application>

App.axaml.cs
using System.Globalization;
using Avalonia;
using Avalonia.Controls.ApplicationLifetimes;
using Avalonia.Markup.Xaml;
using ChudoObuv.Models;
using ChudoObuv.Views;

namespace ChudoObuv;

public partial class App : Application
{
    private static ChudoObuvContext? db;
    public static ChudoObuvContext Db => db ??= new ChudoObuvContext();

    public static void ResetDb()
    {
        db?.Dispose();
        db = null;
    }

    public App()
    {
        var ru = new CultureInfo("ru-RU");
        CultureInfo.DefaultThreadCurrentCulture = ru;
        CultureInfo.DefaultThreadCurrentUICulture = ru;
        CultureInfo.CurrentCulture = ru;
        CultureInfo.CurrentUICulture = ru;
    }

    public override void Initialize() => AvaloniaXamlLoader.Load(this);

    public override void OnFrameworkInitializationCompleted()
    {
        if (ApplicationLifetime is IClassicDesktopStyleApplicationLifetime desktop)
        {
            // Приложение начинается с окна авторизации
            desktop.MainWindow = new LoginWindow();
        }
        base.OnFrameworkInitializationCompleted();
    }
}


#Dialogs
Dialogs.cs
using Avalonia;
using Avalonia.Controls;
using Avalonia.Layout;
using Avalonia.Media;

namespace ChudoObuv;

/// <summary>Замена WPF MessageBox: в Avalonia его нет.</summary>
public static class Dialogs
{
    public static Task ShowMessage(Window owner, string text, string title = "Сообщение")
        => Show(owner, text, title, "OK", null);

    public static async Task<bool> Confirm(Window owner, string text, string title = "Подтверждение")
        => await Show(owner, text, title, "Да", "Нет");

    private static async Task<bool> Show(Window owner, string text, string title, string okText, string? cancelText)
    {
        var dlg = new Window
        {
            Title = title,
            Width = 380,
            SizeToContent = SizeToContent.Height,
            CanResize = false,
            WindowStartupLocation = WindowStartupLocation.CenterOwner,
            Background = Brushes.White
        };

        var ok = new Button { Content = okText, MinWidth = 80, HorizontalContentAlignment = HorizontalAlignment.Center };
        ok.Click += (_, _) => dlg.Close(true);

        var buttons = new StackPanel
        {
            Orientation = Orientation.Horizontal,
            HorizontalAlignment = HorizontalAlignment.Right,
            Spacing = 8,
            Margin = new Thickness(0, 18, 0, 0)
        };
        buttons.Children.Add(ok);

        if (cancelText != null)
        {
            var cancel = new Button { Content = cancelText, MinWidth = 80, HorizontalContentAlignment = HorizontalAlignment.Center };
            cancel.Click += (_, _) => dlg.Close(false);
            buttons.Children.Add(cancel);
        }

        dlg.Content = new StackPanel
        {
            Margin = new Thickness(20),
            Children =
            {
                new TextBlock { Text = text, TextWrapping = TextWrapping.Wrap },
                buttons
            }
        };

        var result = await dlg.ShowDialog<bool?>(owner);
        return result == true;
    }
}


#ProductTools
ProductTools.cs
using Avalonia.Controls;
using ChudoObuv.Models;

namespace ChudoObuv;

public static class ProductTools
{
    // Один метод поиска (раньше было два одинаковых: Search и srearch).
    // ?? "" защищает от NullReferenceException, если в БД NULL.
    public static List<Product> Search(List<Product> list, string? text)
    {
        if (string.IsNullOrWhiteSpace(text)) return list;
        text = text.Trim().ToLower();
        return list.Where(p =>
            (p.ProductName ?? "").ToLower().Contains(text) ||
            (p.Composition ?? "").ToLower().Contains(text) ||
            (p.Description ?? "").ToLower().Contains(text) ||
            (p.Manufacturer?.ManufacturerName ?? "").ToLower().Contains(text)).ToList();
    }

    public static List<Product> ByManufacturer(List<Product> list, string? name)
    {
        if (string.IsNullOrEmpty(name) || name == "Все") return list;
        return list.Where(p => p.Manufacturer.ManufacturerName == name).ToList();
    }

    public static List<Product> ByCategory(List<Product> list, string? name)
    {
        if (string.IsNullOrEmpty(name) || name == "Все") return list;
        return list.Where(p => p.CategoryName == name).ToList();
    }

    public static List<Product> BySubcategory(List<Product> list, string? name)
    {
        if (string.IsNullOrEmpty(name) || name == "Все") return list;
        return list.Where(p => p.Subcategory.SubcategoryName == name).ToList();
    }

    public static List<Product> OnlyInStock(List<Product> list)
        => list.Where(p => p.TotalQuantity > 0).ToList();
    public static List<Product> OnlyLowStock(List<Product> list)
        => list.Where(p => p.IsLowStock).ToList();
    public static List<Product> ByPrice(List<Product> list, decimal min, decimal max)
        => list.Where(p => p.Price >= min && p.Price <= max).ToList();
    public static List<Product> BySize(List<Product> list, decimal size)
        => list.Where(p => p.StockItems.Any(s => s.SizeValue == size && s.Quantity > 0)).ToList();

    public static readonly string[] SortNames =
    {
        "Без сортировки",
        "Цена по возростанию",
        "Цена по убыванию",
        "Название А-Я",
        "Название Я-А",
        "Остаток по возростанию",
        "Остаток по убыванию",
    };

    public static List<Product> Sort(List<Product> list, string? sort)
    {
        switch (sort)
        {
            case "Цена по возростанию":    return list.OrderBy(p => p.Price).ToList();
            case "Цена по убыванию":       return list.OrderByDescending(p => p.Price).ToList();
            case "Название А-Я":           return list.OrderBy(p => p.ProductName).ToList();
            case "Название Я-А":           return list.OrderByDescending(p => p.ProductName).ToList();
            case "Остаток по возростанию": return list.OrderBy(p => p.TotalQuantity).ToList();
            case "Остаток по убыванию":    return list.OrderByDescending(p => p.TotalQuantity).ToList();
            default:                       return list.OrderBy(p => p.ProductId).ToList();
        }
    }

    public static List<string> ComboItems(IEnumerable<string> values)
    {
        var result = new List<string> { "Все" };
        result.AddRange(values.Distinct().OrderBy(x => x));
        return result;
    }

    public static decimal AveragePrice(List<Product> list) => list.Count == 0 ? 0 : list.Average(p => p.Price);
    public static decimal MaxPrice(List<Product> list)     => list.Count == 0 ? 0 : list.Max(p => p.Price);
    public static decimal MinPrice(List<Product> list)     => list.Count == 0 ? 0 : list.Min(p => p.Price);
    public static int TotalPairs(List<Product> list)       => list.Sum(p => p.TotalQuantity);
    public static decimal StockValue(List<Product> list)   => list.Sum(p => p.Price * p.TotalQuantity);

    // В Avalonia нет MessageBox, поэтому метод асинхронный и принимает окно-владельца.
    public static async Task<bool> DeleteAsync(Window owner, Product product)
    {
        bool inOrders = product.StockItems.Any(s => s.OrderItems.Count > 0);
        if (inOrders)
        {
            await Dialogs.ShowMessage(owner, "Нельзя удалить: товар есть в заказах.", "Ошибка");
            return false;
        }

        bool yes = await Dialogs.Confirm(owner, $"Удалить товар «{product.ProductName}»?");
        if (!yes) return false;

        App.Db.StockItems.RemoveRange(product.StockItems);
        App.Db.Products.Remove(product);
        App.Db.SaveChanges();
        return true;
    }

    public static void SetPrice(Product product, decimal newPrice)
    {
        product.Price = newPrice;
        App.Db.SaveChanges();
    }

    public static void Discount(Product product, decimal percent)
    {
        product.Price = Math.Round(product.Price * (100 - percent) / 100, 2);
        App.Db.SaveChanges();
    }

    public static void DiscountAll(List<Product> list, decimal percent)
    {
        foreach (var p in list)
            p.Price = Math.Round(p.Price * (100 - percent) / 100, 2);
        App.Db.SaveChanges();
    }

    public static void AddStock(Product product, decimal size, int delta)
    {
        var item = product.StockItems.FirstOrDefault(s => s.SizeValue == size);
        if (item == null)
            product.StockItems.Add(new StockItem { SizeValue = size, Quantity = Math.Max(0, delta) });
        else
            item.Quantity = Math.Max(0, item.Quantity + delta);
        App.Db.SaveChanges();
    }
}


#Models (ЧИСТЫЕ МОДЕЛИ ИЗ СКАФФОЛДА EF CORE БЕЗ [NotMapped])
Models/Category.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Category
{
    public int CategoryId { get; set; }

    public string CategoryName { get; set; } = null!;

    public virtual ICollection<Subcategory> Subcategories { get; set; } = new List<Subcategory>();
}

Models/Manufacturer.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Manufacturer
{
    public int ManufacturerId { get; set; }

    public string ManufacturerName { get; set; } = null!;

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}

Models/Order.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Order
{
    public int OrderId { get; set; }

    public DateOnly OrderDate { get; set; }

    public int UserId { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();

    public virtual User User { get; set; } = null!;
}

Models/OrderItem.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class OrderItem
{
    public int OrderItemId { get; set; }

    public int OrderId { get; set; }

    public int StockItemId { get; set; }

    public int Quantity { get; set; }

    public decimal UnitPrice { get; set; }

    public virtual Order Order { get; set; } = null!;

    public virtual StockItem StockItem { get; set; } = null!;
}

Models/Product.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Product
{
    public int ProductId { get; set; }

    public string ProductName { get; set; } = null!;

    public int ManufacturerId { get; set; }

    public int SubcategoryId { get; set; }

    public string Description { get; set; } = null!;

    public string Composition { get; set; } = null!;

    public string? ImageFile { get; set; }

    public decimal Price { get; set; }

    public virtual Manufacturer Manufacturer { get; set; } = null!;

    public virtual ICollection<StockItem> StockItems { get; set; } = new List<StockItem>();

    public virtual Subcategory Subcategory { get; set; } = null!;
}

Models/Role.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Role
{
    public int RoleId { get; set; }

    public string RoleName { get; set; } = null!;

    public virtual ICollection<User> Users { get; set; } = new List<User>();
}

Models/StockItem.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class StockItem
{
    public int StockItemId { get; set; }

    public int ProductId { get; set; }

    public decimal SizeValue { get; set; }

    public int Quantity { get; set; }

    public virtual ICollection<OrderItem> OrderItems { get; set; } = new List<OrderItem>();

    public virtual Product Product { get; set; } = null!;
}

Models/Subcategory.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class Subcategory
{
    public int SubcategoryId { get; set; }

    public int CategoryId { get; set; }

    public string SubcategoryName { get; set; } = null!;

    public virtual Category Category { get; set; } = null!;

    public virtual ICollection<Product> Products { get; set; } = new List<Product>();
}

Models/User.cs
using System;
using System.Collections.Generic;

namespace ChudoObuv.Models;

public partial class User
{
    public int UserId { get; set; }

    public string LastName { get; set; } = null!;

    public string FirstName { get; set; } = null!;

    public string? Patronymic { get; set; }

    public string Login { get; set; } = null!;

    public int RoleId { get; set; }

    public virtual ICollection<Order> Orders { get; set; } = new List<Order>();

    public virtual Role Role { get; set; } = null!;
}

Models/ChudoObuvContext.cs
using System;
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Models;

public partial class ChudoObuvContext : DbContext
{
    public ChudoObuvContext()
    {
    }

    public ChudoObuvContext(DbContextOptions<ChudoObuvContext> options)
        : base(options)
    {
    }

    public virtual DbSet<Category> Categories { get; set; }

    public virtual DbSet<Manufacturer> Manufacturers { get; set; }

    public virtual DbSet<Order> Orders { get; set; }

    public virtual DbSet<OrderItem> OrderItems { get; set; }

    public virtual DbSet<Product> Products { get; set; }

    public virtual DbSet<Role> Roles { get; set; }

    public virtual DbSet<StockItem> StockItems { get; set; }

    public virtual DbSet<Subcategory> Subcategories { get; set; }

    public virtual DbSet<User> Users { get; set; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
    {
        if (!optionsBuilder.IsConfigured)
        {
            optionsBuilder
                .UseLazyLoadingProxies()
                .UseMySql(AppSettings.ConnectionString, AppSettings.GetServerVersion());
        }
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<Category>(entity =>
        {
            entity.HasKey(e => e.CategoryId).HasName("PRIMARY");

            entity.ToTable("categories");

            entity.HasIndex(e => e.CategoryName, "uq_categories_name").IsUnique();

            entity.Property(e => e.CategoryId).HasColumnName("category_id");
            entity.Property(e => e.CategoryName)
                .HasMaxLength(60)
                .HasColumnName("category_name");
        });

        modelBuilder.Entity<Manufacturer>(entity =>
        {
            entity.HasKey(e => e.ManufacturerId).HasName("PRIMARY");

            entity.ToTable("manufacturers");

            entity.HasIndex(e => e.ManufacturerName, "uq_manufacturers_name").IsUnique();

            entity.Property(e => e.ManufacturerId).HasColumnName("manufacturer_id");
            entity.Property(e => e.ManufacturerName)
                .HasMaxLength(100)
                .HasColumnName("manufacturer_name");
        });

        modelBuilder.Entity<Order>(entity =>
        {
            entity.HasKey(e => e.OrderId).HasName("PRIMARY");

            entity.ToTable("orders");

            entity.HasIndex(e => e.UserId, "ix_orders_user");

            entity.Property(e => e.OrderId).HasColumnName("order_id");
            entity.Property(e => e.OrderDate).HasColumnName("order_date");
            entity.Property(e => e.UserId).HasColumnName("user_id");

            entity.HasOne(d => d.User).WithMany(p => p.Orders)
                .HasForeignKey(d => d.UserId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_orders_user");
        });

        modelBuilder.Entity<OrderItem>(entity =>
        {
            entity.HasKey(e => e.OrderItemId).HasName("PRIMARY");

            entity.ToTable("order_items", tb => tb.HasCheckConstraint("ck_oi_quantity", "quantity > 0"));

            entity.HasIndex(e => e.StockItemId, "ix_oi_stock");

            entity.HasIndex(e => new { e.OrderId, e.StockItemId }, "uq_order_stock").IsUnique();

            entity.Property(e => e.OrderItemId).HasColumnName("order_item_id");
            entity.Property(e => e.OrderId).HasColumnName("order_id");
            entity.Property(e => e.Quantity).HasColumnName("quantity");
            entity.Property(e => e.StockItemId).HasColumnName("stock_item_id");
            entity.Property(e => e.UnitPrice)
                .HasPrecision(10, 2)
                .HasColumnName("unit_price");

            entity.HasOne(d => d.Order).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.OrderId)
                .HasConstraintName("fk_oi_order");

            entity.HasOne(d => d.StockItem).WithMany(p => p.OrderItems)
                .HasForeignKey(d => d.StockItemId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_oi_stock");
        });

        modelBuilder.Entity<Product>(entity =>
        {
            entity.HasKey(e => e.ProductId).HasName("PRIMARY");

            entity.ToTable("products");

            entity.HasIndex(e => e.ManufacturerId, "ix_products_man");

            entity.HasIndex(e => e.SubcategoryId, "ix_products_sub");

            entity.HasIndex(e => new { e.ProductName, e.ManufacturerId }, "uq_products_name_man").IsUnique();

            entity.Property(e => e.ProductId).HasColumnName("product_id");
            entity.Property(e => e.Composition)
                .HasColumnType("text")
                .HasColumnName("composition");
            entity.Property(e => e.Description)
                .HasColumnType("text")
                .HasColumnName("description");
            entity.Property(e => e.ImageFile)
                .HasMaxLength(100)
                .HasComment("имя файла изображения в папке Images приложения")
                .HasColumnName("image_file");
            entity.Property(e => e.ManufacturerId).HasColumnName("manufacturer_id");
            entity.Property(e => e.Price)
                .HasPrecision(10, 2)
                .HasColumnName("price");
            entity.Property(e => e.ProductName)
                .HasMaxLength(200)
                .HasColumnName("product_name");
            entity.Property(e => e.SubcategoryId).HasColumnName("subcategory_id");

            entity.HasOne(d => d.Manufacturer).WithMany(p => p.Products)
                .HasForeignKey(d => d.ManufacturerId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_products_man");

            entity.HasOne(d => d.Subcategory).WithMany(p => p.Products)
                .HasForeignKey(d => d.SubcategoryId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_products_sub");
        });

        modelBuilder.Entity<Role>(entity =>
        {
            entity.HasKey(e => e.RoleId).HasName("PRIMARY");

            entity.ToTable("roles");

            entity.HasIndex(e => e.RoleName, "uq_roles_name").IsUnique();

            entity.Property(e => e.RoleId).HasColumnName("role_id");
            entity.Property(e => e.RoleName)
                .HasMaxLength(50)
                .HasColumnName("role_name");
        });

        modelBuilder.Entity<StockItem>(entity =>
        {
            entity.HasKey(e => e.StockItemId).HasName("PRIMARY");

            entity.ToTable("stock_items", tb => tb.HasCheckConstraint("ck_stock_quantity", "quantity >= 0"));

            entity.HasIndex(e => new { e.ProductId, e.SizeValue }, "uq_stock_product_size").IsUnique();

            entity.Property(e => e.StockItemId).HasColumnName("stock_item_id");
            entity.Property(e => e.ProductId).HasColumnName("product_id");
            entity.Property(e => e.Quantity).HasColumnName("quantity");
            entity.Property(e => e.SizeValue)
                .HasPrecision(3, 1)
                .HasColumnName("size_value");

            entity.HasOne(d => d.Product).WithMany(p => p.StockItems)
                .HasForeignKey(d => d.ProductId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_stock_product");

        });

        modelBuilder.Entity<Subcategory>(entity =>
        {
            entity.HasKey(e => e.SubcategoryId).HasName("PRIMARY");

            entity.ToTable("subcategories");

            entity.HasIndex(e => new { e.CategoryId, e.SubcategoryName }, "uq_subcategories").IsUnique();

            entity.Property(e => e.SubcategoryId).HasColumnName("subcategory_id");
            entity.Property(e => e.CategoryId).HasColumnName("category_id");
            entity.Property(e => e.SubcategoryName)
                .HasMaxLength(60)
                .HasColumnName("subcategory_name");

            entity.HasOne(d => d.Category).WithMany(p => p.Subcategories)
                .HasForeignKey(d => d.CategoryId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_subcategories_category");
        });

        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(e => e.UserId).HasName("PRIMARY");

            entity.ToTable("users");

            entity.HasIndex(e => e.Login, "uq_users_login").IsUnique();

            entity.HasIndex(e => e.RoleId, "ix_users_role");

            entity.Property(e => e.UserId).HasColumnName("user_id");
            entity.Property(e => e.FirstName)
                .HasMaxLength(60)
                .HasColumnName("first_name");
            entity.Property(e => e.LastName)
                .HasMaxLength(60)
                .HasColumnName("last_name");
            entity.Property(e => e.Login)
                .HasMaxLength(50)
                .HasColumnName("login");
            entity.Property(e => e.Patronymic)
                .HasMaxLength(60)
                .HasColumnName("patronymic");
            entity.Property(e => e.RoleId).HasColumnName("role_id");

            entity.HasOne(d => d.Role).WithMany(p => p.Users)
                .HasForeignKey(d => d.RoleId)
                .OnDelete(DeleteBehavior.ClientSetNull)
                .HasConstraintName("fk_users_role");
        });

        OnModelCreatingPartial(modelBuilder);
    }

    partial void OnModelCreatingPartial(ModelBuilder modelBuilder);
}

Models/AppSettings.cs
using System.Text.Json;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Models;
public static class AppSettings
{
    private const string DefaultConnection =
        "server=localhost;port=3306;database=chudo_obuv;user=root;password=1234;CharSet=utf8mb4";

    private static string? connectionString;
    private static ServerVersion? serverVersion;

    public static string ConnectionString
    {
        get
        {
            if (connectionString != null)
                return connectionString;

            connectionString = DefaultConnection;
            
            {
                var file = System.IO.Path.Combine(AppContext.BaseDirectory, "appsettings.json");
                if (System.IO.File.Exists(file))
                {
                    using var doc = JsonDocument.Parse(System.IO.File.ReadAllText(file));
                    if (doc.RootElement.TryGetProperty("ConnectionString", out var value)
                        && !string.IsNullOrWhiteSpace(value.GetString()))
                    {
                        connectionString = value.GetString()!;
                    }
                }
            }
           

            return connectionString;
        }
    }
    public static ServerVersion GetServerVersion()
    {
        return serverVersion ??= ServerVersion.AutoDetect(ConnectionString);
    }
}


#PartialModels (ЧАСТИЧНЫЕ КЛАССЫ С ВЫЧИСЛЯЕМЫМИ СВОЙСТВАМИ И КОРЗИНОЙ)
Models/PartialModels.cs
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations.Schema;
using System.Linq;
using Avalonia.Media;
using Avalonia.Media.Imaging;

namespace ChudoObuv.Models;

public partial class User
{
    [NotMapped]
    public string Fio => string.Join(" ",
        new[] { LastName, FirstName, Patronymic }.Where(s => !string.IsNullOrWhiteSpace(s)));
}

public partial class Product
{
    public const int LowStockLimit = 3;

    [NotMapped]
    public string CategoryText => $"{Subcategory?.Category?.CategoryName ?? ""} / {Subcategory?.SubcategoryName ?? ""}";

    [NotMapped]
    public string CategoryName => Subcategory?.Category?.CategoryName ?? "";

    /// <summary>Доступное для заказа количество суммарно по всем размерам модели.</summary>
    [NotMapped]
    public int TotalQuantity => StockItems.Sum(s => s.Quantity);

    /// <summary>Товары с суммарным количеством <= 3 подсвечиваются в списке.</summary>
    [NotMapped]
    public bool IsLowStock => TotalQuantity <= LowStockLimit;

    [NotMapped]
    public List<StockItem> AvailableStock => StockItems
        .Where(s => s.Quantity > 0)
        .OrderBy(s => s.SizeValue)
        .ToList();

    [NotMapped]
    public string SizesText
    {
        get
        {
            var list = AvailableStock;
            return list.Count == 0
                ? "нет в наличии"
                : string.Join(", ", list.Select(s => s.SizeText));
        }
    }

    [NotMapped]
    public string SizesDisplayText => "Размеры: " + SizesText;

    [NotMapped]
    public string ManufacturerText => "Производитель: " + (Manufacturer?.ManufacturerName ?? "");

    [NotMapped]
    public string CompositionText => "Состав: " + (Composition ?? "");

    [NotMapped]
    public string StockCountDisplay => $"В наличии: {TotalQuantity} пар";

    [NotMapped]
    public string ImagePath => System.IO.Path.Combine(AppContext.BaseDirectory, "Images",
        string.IsNullOrWhiteSpace(ImageFile) ? "picture.png" : ImageFile);

    [NotMapped]
    public string PriceText => Price.ToString("N0") + " ₽";

    [NotMapped]
    public IBrush CardBackground => IsLowStock
        ? Brush.Parse("#FF8080")
        : Brushes.White;

    [NotMapped]
    public Bitmap? ImageBitmap
    {
        get
        {
            var file = System.IO.File.Exists(ImagePath)
                ? ImagePath
                : System.IO.Path.Combine(AppContext.BaseDirectory, "Images", "picture.png");
            if (!System.IO.File.Exists(file)) return null;
            try
            {
                using var stream = System.IO.File.OpenRead(file);
                return Bitmap.DecodeToWidth(stream, 240);
            }
            catch { return null; }
        }
    }
}

public partial class StockItem
{
    [NotMapped]
    public string SizeText => SizeValue.ToString("0.#");

    [NotMapped]
    public string SizeDisplay => $"Размер {SizeText} (в наличии: {Quantity})";

    [NotMapped]
    public string FullDisplay => $"{Product?.ProductName ?? ""} — размер {SizeText} (в наличии: {Quantity})";
}

public partial class Order
{
    [NotMapped]
    public string CustomerFio => User?.Fio ?? "Неизвестный клиент";

    [NotMapped]
    public decimal Total => OrderItems.Sum(i => i.Quantity * i.UnitPrice);

    [NotMapped]
    public int ItemsCount => OrderItems.Sum(i => i.Quantity);

    [NotMapped]
    public string OrderDateText => OrderDate.ToString("dd.MM.yyyy");

    [NotMapped]
    public string SummaryText => $"Заказ №{OrderId} от {OrderDateText} — {CustomerFio}: {ItemsCount} шт. на {Total:N0} ₽";
}

public partial class OrderItem
{
    [NotMapped]
    public string ProductName => StockItem?.Product?.ProductName ?? "Товар";

    [NotMapped]
    public string SizeText => StockItem?.SizeText ?? "";

    [NotMapped]
    public decimal Sum => Quantity * UnitPrice;

    [NotMapped]
    public string DisplayText => $"{ProductName} (р. {SizeText}) — {Quantity} шт. x {UnitPrice:N0} ₽ = {Sum:N0} ₽";
}

public class CartItem
{
    public StockItem StockItem { get; set; } = null!;
    public int Quantity { get; set; }

    public string ProductName => StockItem?.Product?.ProductName ?? "Товар";
    public string SizeText => StockItem?.SizeText ?? "";
    public decimal UnitPrice => StockItem?.Product?.Price ?? 0;
    public decimal Sum => Quantity * UnitPrice;
    public string PriceText => UnitPrice.ToString("N0") + " ₽";
    public string SumText => Sum.ToString("N0") + " ₽";
    public string DisplayText => $"{ProductName} (р. {SizeText}) — {Quantity} шт. по {PriceText} = {SumText}";
}

public static class CartManager
{
    public static List<CartItem> Items { get; } = new();

    public static int TotalCount => Items.Sum(i => i.Quantity);
    public static decimal TotalSum => Items.Sum(i => i.Sum);

    public static void Add(StockItem stock, int qty = 1)
    {
        var existing = Items.FirstOrDefault(i => i.StockItem.StockItemId == stock.StockItemId);
        if (existing != null)
        {
            if (existing.Quantity + qty <= stock.Quantity)
                existing.Quantity += qty;
        }
        else
        {
            Items.Add(new CartItem { StockItem = stock, Quantity = Math.Min(qty, stock.Quantity) });
        }
    }

    public static void Clear() => Items.Clear();
}


#Views
Views/LoginWindow.axaml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="ChudoObuv.Views.LoginWindow"
        x:CompileBindings="False"
        Title="Авторизация — Чудо Обувь"
        Icon="/Resources/app.ico"
        Width="380" SizeToContent="Height" CanResize="False"
        WindowStartupLocation="CenterScreen"
        Background="{DynamicResource MainBg}">
    <StackPanel Margin="28,20,28,24">
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Center">
            <Image Source="/Resources/logo.png" Height="56"/>
            <TextBlock Text="Чудо Обувь" FontSize="26" FontWeight="Bold" Foreground="{DynamicResource TextBrush}"
                       VerticalAlignment="Center" Margin="12,0,0,0"/>
        </StackPanel>

        <TextBlock Text="Вход в систему" FontSize="18" Foreground="{DynamicResource TextBrush}"
                   HorizontalAlignment="Center" Margin="0,16,0,6"/>

        <TextBlock Text="Логин" Foreground="{DynamicResource MutedBrush}" Margin="0,10,0,3"/>
        <TextBox x:Name="tbLogin" Height="32" Padding="6,0" MaxLength="50"
                 VerticalContentAlignment="Center" BorderBrush="{DynamicResource Accent}"/>

        <TextBlock x:Name="tbError" Foreground="{DynamicResource ErrorBrush}" TextWrapping="Wrap"
                   Margin="0,12,0,0" MinHeight="18"/>

        <Button Content="Войти" Height="36" Margin="0,10,0,0" IsDefault="True"
                HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"
                VerticalContentAlignment="Center"
                Background="{DynamicResource Accent}" Foreground="White"
                Click="BtnLogin_Click"/>

        <Button Content="Войти как гость" Height="34" Margin="0,8,0,0"
                HorizontalAlignment="Stretch" HorizontalContentAlignment="Center"
                VerticalContentAlignment="Center"
                Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                Click="BtnGuest_Click"/>
    </StackPanel>
</Window>

Views/LoginWindow.axaml.cs
using System;
using Avalonia.Controls;
using Avalonia.Interactivity;
using ChudoObuv.Models;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Views;

public partial class LoginWindow : Window
{
    /// <summary>Пользователь, вошедший в систему (null = неавторизованный гость).</summary>
    public static User? CurrentUser { get; private set; }

    public LoginWindow()
    {
        InitializeComponent();
        Opened += (_, _) => tbLogin.Focus();
    }

    private void BtnLogin_Click(object? sender, RoutedEventArgs e)
    {
        tbError.Text = "";
        string login = (tbLogin.Text ?? "").Trim();

        if (login.Length == 0)
        {
            tbError.Text = "Введите логин.";
            return;
        }

        try
        {
            var user = App.Db.Users
                .Include(u => u.Role)
                .FirstOrDefault(u => u.Login == login);

            if (user == null)
            {
                tbError.Text = "Пользователь с таким логином не найден.";
                return;
            }

            CurrentUser = user;
            OpenMainWindow();
        }
        catch (Exception ex)
        {
            App.ResetDb();
            tbError.Text = "Ошибка подключения к базе данных: " + ex.Message;
        }
    }

    private void BtnGuest_Click(object? sender, RoutedEventArgs e)
    {
        CurrentUser = null; // Гость (неавторизованный пользователь)
        OpenMainWindow();
    }

    private void OpenMainWindow()
    {
        var products = new Products();
        if (Avalonia.Application.Current?.ApplicationLifetime
            is Avalonia.Controls.ApplicationLifetimes.IClassicDesktopStyleApplicationLifetime desktop)
        {
            desktop.MainWindow = products;
        }
        products.Show();
        Close();
    }
}

Views/Products.axaml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="ChudoObuv.Views.Products"
        x:CompileBindings="False"
        Title="Каталог товаров — Чудо Обувь"
        Icon="/Resources/app.ico"
        Height="820" Width="820" MinWidth="600" MinHeight="450"
        WindowStartupLocation="CenterScreen"
        Background="{DynamicResource MainBg}">
    <DockPanel>
        <!-- Шапка с логотипом, названием и статусом авторизации -->
        <Border DockPanel.Dock="Top" BorderThickness="0,0,0,1" BorderBrush="{DynamicResource Accent}" Padding="14,8" Background="{DynamicResource AltBg}">
            <Grid ColumnDefinitions="Auto,*,Auto">
                <StackPanel Grid.Column="0" Orientation="Horizontal" VerticalAlignment="Center">
                    <Image Source="/Resources/logo.png" Height="50"/>
                    <TextBlock Text="Чудо Обувь" FontSize="22" FontWeight="Bold" Foreground="{DynamicResource TextBrush}"
                               VerticalAlignment="Center" Margin="10,0,0,0"/>
                </StackPanel>

                <StackPanel Grid.Column="2" Orientation="Horizontal" VerticalAlignment="Center" Spacing="10">
                    <TextBlock x:Name="tbUserStatus" Text="Гость" FontSize="14" Foreground="{DynamicResource TextBrush}"
                               VerticalAlignment="Center"/>
                    <Button Content="Выйти" Padding="12,5" Background="{DynamicResource Accent}" Foreground="White"
                            Click="BtnLogout_Click"/>
                </StackPanel>
            </Grid>
        </Border>

        <!-- Панель поиска, фильтрации и сортировки -->
        <WrapPanel DockPanel.Dock="Top" Margin="14,10,14,6">
            <TextBox x:Name="tbSearch" Width="200" Height="32" Margin="0,0,8,6"
                     Watermark="Поиск..."
                     VerticalContentAlignment="Center"
                     ToolTip.Tip="Поиск по названию, составу, производителю, описанию"/>

            <ComboBox x:Name="cbManufacturer" Width="160" Height="32" Margin="0,0,8,6"
                      SelectionChanged="Filter_Changed"/>

            <ComboBox x:Name="cbCategory" Width="140" Height="32" Margin="0,0,8,6"
                      SelectionChanged="Filter_Changed"/>

            <ComboBox x:Name="cbSort" Width="170" Height="32" Margin="0,0,8,6"
                      SelectionChanged="Filter_Changed"/>

            <TextBlock x:Name="tbCount" VerticalAlignment="Center" Margin="8,0,0,6"
                       FontWeight="SemiBold" Foreground="{DynamicResource MutedBrush}"/>
        </WrapPanel>

        <!-- Панель действий: корзина, заказы, удаление -->
        <WrapPanel DockPanel.Dock="Top" Margin="14,0,14,10">
            <Button x:Name="btnAddToCart" Content="В корзину" Height="32" Margin="0,0,8,4"
                    Background="{DynamicResource Accent}" Foreground="White"
                    Click="BtnAddToCart_Click"/>

            <Button x:Name="btnOpenCart" Content="Корзина (0)" Height="32" Margin="0,0,8,4"
                    Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                    Click="BtnOpenCart_Click"/>

            <Button x:Name="btnOrders" Content="Список заказов" Height="32" Margin="0,0,8,4"
                    Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                    Click="BtnOrders_Click"/>

            <Button x:Name="btnDelete" Content="Удалить товар" Height="32" Margin="0,0,8,4"
                    Background="{DynamicResource ErrorBrush}" Foreground="White"
                    Click="BtnDelete_Click"/>
        </WrapPanel>

        <!-- Список товаров (ListBox с DataTemplate и подсветкой #FF8080 при остатке <= 3) -->
        <ListBox x:Name="lbProducts" Margin="14,0,14,14" Background="Transparent"
                 SelectionMode="Single">
            <ListBox.ItemsPanel>
                <ItemsPanelTemplate>
                    <StackPanel Spacing="8"/>
                </ItemsPanelTemplate>
            </ListBox.ItemsPanel>
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <Border Margin="0,0,0,2" Padding="12" CornerRadius="2"
                            Background="{Binding CardBackground}"
                            BorderBrush="{DynamicResource Accent}" BorderThickness="1">
                        <Grid ColumnDefinitions="136,*,Auto">
                            <!-- Фото товара -->
                            <Border Grid.Column="0" Background="White" BorderThickness="1"
                                    BorderBrush="{DynamicResource Accent}"
                                    Width="124" Height="124" VerticalAlignment="Top" HorizontalAlignment="Left">
                                <Image Source="{Binding ImageBitmap}" Stretch="Uniform" Margin="4"/>
                            </Border>

                            <!-- Описание модели -->
                            <StackPanel Grid.Column="1" Margin="14,0,12,0" VerticalAlignment="Center">
                                <TextBlock Text="{Binding ProductName}" FontSize="18" FontWeight="Bold"
                                           Foreground="{DynamicResource TextBrush}"/>
                                <TextBlock Text="{Binding CategoryText}" Margin="0,3,0,0"
                                           Foreground="{DynamicResource MutedBrush}"/>
                                <TextBlock Text="{Binding ManufacturerText}" Margin="0,4,0,0"
                                           Foreground="{DynamicResource TextBrush}"/>
                                <TextBlock Text="{Binding CompositionText}" MaxHeight="40" Margin="0,2,0,0"
                                           TextWrapping="Wrap" Foreground="{DynamicResource TextBrush}"
                                           ToolTip.Tip="{Binding Description}"/>
                                <TextBlock Text="{Binding SizesDisplayText}" Margin="0,3,0,0"
                                           Foreground="{DynamicResource TextBrush}"/>
                            </StackPanel>

                            <!-- Цена и остаток -->
                            <StackPanel Grid.Column="2" MinWidth="130" VerticalAlignment="Center" HorizontalAlignment="Right">
                                <TextBlock Text="{Binding PriceText}" FontSize="22" FontWeight="Bold"
                                           HorizontalAlignment="Right" Foreground="{DynamicResource TextBrush}"/>
                                <TextBlock Text="{Binding StockCountDisplay}" HorizontalAlignment="Right"
                                           Margin="0,6,0,0" Foreground="{DynamicResource MutedBrush}"/>
                            </StackPanel>
                        </Grid>
                    </Border>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </DockPanel>
</Window>

Views/Products.axaml.cs
using System;
using System.Collections.Generic;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using ChudoObuv.Models;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Views;

public partial class Products : Window
{
    private List<Product> _all = new();
    private bool _ready;

    public Products()
    {
        InitializeComponent();

        tbSearch.PropertyChanged += (_, e) =>
        {
            if (e.Property == TextBox.TextProperty) ShowProducts();
        };

        ApplyUserRole();
        LoadData();
    }

    private void ApplyUserRole()
    {
        var user = LoginWindow.CurrentUser;
        if (user == null)
        {
            tbUserStatus.Text = "Гость";
            btnOrders.IsVisible = false;
            btnDelete.IsVisible = false;
        }
        else
        {
            tbUserStatus.Text = $"{user.Fio} ({user.Role.RoleName})";
            int roleId = user.RoleId;
            btnOrders.IsVisible = (roleId == 1 || roleId == 2); // Админ и Менеджер
            btnDelete.IsVisible = (roleId == 1);                // Только Админ
        }

        UpdateCartButton();
    }

    public void UpdateCartButton()
    {
        btnOpenCart.Content = $"Корзина ({CartManager.TotalCount})";
    }

    public void LoadData()
    {
        _ready = false;

        _all = App.Db.Products
            .Include(x => x.Manufacturer)
            .Include(x => x.Subcategory).ThenInclude(s => s.Category)
            .Include(x => x.StockItems)
            .OrderBy(x => x.ProductId)
            .ToList();

        FillCombos();
        _ready = true;
        ShowProducts();
    }

    private void FillCombos()
    {
        cbManufacturer.ItemsSource = ProductTools.ComboItems(_all.Select(p => p.Manufacturer.ManufacturerName));
        cbCategory.ItemsSource = ProductTools.ComboItems(_all.Select(p => p.CategoryName));
        cbSort.ItemsSource = ProductTools.SortNames;

        cbManufacturer.SelectedIndex = 0;
        cbCategory.SelectedIndex = 0;
        cbSort.SelectedIndex = 0;
    }

    private void ShowProducts()
    {
        if (!_ready) return;

        var list = _all;
        list = ProductTools.Search(list, tbSearch.Text);
        list = ProductTools.ByManufacturer(list, cbManufacturer.SelectedItem as string);
        list = ProductTools.ByCategory(list, cbCategory.SelectedItem as string);
        list = ProductTools.Sort(list, cbSort.SelectedItem as string);

        lbProducts.ItemsSource = list;
        tbCount.Text = $"Отображено: {list.Count} из {_all.Count}";
    }

    private void Filter_Changed(object? sender, SelectionChangedEventArgs e) => ShowProducts();

    private async void BtnAddToCart_Click(object? sender, RoutedEventArgs e)
    {
        var product = lbProducts.SelectedItem as Product;
        if (product == null)
        {
            await Dialogs.ShowMessage(this, "Выберите товар в списке для добавления в корзину.");
            return;
        }

        var stock = product.AvailableStock.FirstOrDefault();
        if (stock == null)
        {
            await Dialogs.ShowMessage(this, "Данного товара нет в наличии.");
            return;
        }

        CartManager.Add(stock, 1);
        UpdateCartButton();
        await Dialogs.ShowMessage(this, $"Товар «{product.ProductName}» (размер {stock.SizeText}) добавлен в корзину!");
    }

    private async void BtnOpenCart_Click(object? sender, RoutedEventArgs e)
    {
        var cartWindow = new CartWindow();
        await cartWindow.ShowDialog(this);
        UpdateCartButton();
        LoadData();
    }

    private async void BtnOrders_Click(object? sender, RoutedEventArgs e)
    {
        var ordersWindow = new OrdersWindow();
        await ordersWindow.ShowDialog(this);
        LoadData();
    }

    private async void BtnDelete_Click(object? sender, RoutedEventArgs e)
    {
        var product = lbProducts.SelectedItem as Product;
        if (product == null)
        {
            await Dialogs.ShowMessage(this, "Выберите товар для удаления.");
            return;
        }

        if (await ProductTools.DeleteAsync(this, product))
        {
            LoadData();
            await Dialogs.ShowMessage(this, "Товар успешно удален.");
        }
    }

    private void BtnLogout_Click(object? sender, RoutedEventArgs e)
    {
        var login = new LoginWindow();
        if (Avalonia.Application.Current?.ApplicationLifetime
            is Avalonia.Controls.ApplicationLifetimes.IClassicDesktopStyleApplicationLifetime desktop)
        {
            desktop.MainWindow = login;
        }
        login.Show();
        Close();
    }
}

Views/CartWindow.axaml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="ChudoObuv.Views.CartWindow"
        x:CompileBindings="False"
        Title="Корзина заказов — Чудо Обувь"
        Icon="/Resources/app.ico"
        Height="550" Width="680" MinWidth="500" MinHeight="350"
        WindowStartupLocation="CenterOwner"
        Background="{DynamicResource MainBg}">
    <DockPanel Margin="18">
        <Border DockPanel.Dock="Top" BorderThickness="0,0,0,1" BorderBrush="{DynamicResource Accent}" Padding="0,0,0,10">
            <TextBlock Text="Корзина покупок" FontSize="20" FontWeight="Bold" Foreground="{DynamicResource TextBrush}"/>
        </Border>

        <Border DockPanel.Dock="Bottom" BorderThickness="0,1,0,0" BorderBrush="{DynamicResource Accent}" Padding="0,14,0,0" Margin="0,10,0,0">
            <Grid ColumnDefinitions="*,Auto,Auto">
                <TextBlock x:Name="tbTotalSum" Text="Итого: 0 ₽" FontSize="18" FontWeight="Bold"
                           VerticalAlignment="Center" Foreground="{DynamicResource TextBrush}"/>

                <Button Grid.Column="1" Content="Очистить" Margin="0,0,8,0" Padding="12,6"
                        Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                        Click="BtnClear_Click"/>

                <Button Grid.Column="2" Content="Оформить заказ" Padding="16,6"
                        Background="{DynamicResource Accent}" Foreground="White" FontWeight="Bold"
                        Click="BtnCheckout_Click"/>
            </Grid>
        </Border>

        <ListBox x:Name="lbCartItems" Margin="0,10,0,0" Background="Transparent">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <Border Margin="0,0,0,6" Padding="10" CornerRadius="2"
                            Background="White" BorderBrush="{DynamicResource Accent}" BorderThickness="1">
                        <Grid ColumnDefinitions="*,Auto,Auto">
                            <StackPanel Grid.Column="0" VerticalAlignment="Center">
                                <TextBlock Text="{Binding ProductName}" FontSize="16" FontWeight="SemiBold"
                                           Foreground="{DynamicResource TextBrush}"/>
                                <TextBlock Text="{Binding DisplayText}" Margin="0,4,0,0"
                                           Foreground="{DynamicResource MutedBrush}"/>
                            </StackPanel>

                            <StackPanel Grid.Column="1" Orientation="Horizontal" VerticalAlignment="Center" Margin="10,0">
                                <Button Content="-" Width="28" Height="28" Padding="0"
                                        HorizontalContentAlignment="Center" VerticalContentAlignment="Center"
                                        Tag="{Binding}" Click="BtnMinus_Click"/>
                                <TextBlock Text="{Binding Quantity}" VerticalAlignment="Center" Margin="8,0"
                                           FontWeight="Bold"/>
                                <Button Content="+" Width="28" Height="28" Padding="0"
                                        HorizontalContentAlignment="Center" VerticalContentAlignment="Center"
                                        Tag="{Binding}" Click="BtnPlus_Click"/>
                            </StackPanel>

                            <Button Grid.Column="2" Content="Удалить" Padding="8,4"
                                    Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                                    Tag="{Binding}" Click="BtnRemoveItem_Click"/>
                        </Grid>
                    </Border>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </DockPanel>
</Window>

Views/CartWindow.axaml.cs
using System;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using ChudoObuv.Models;

namespace ChudoObuv.Views;

public partial class CartWindow : Window
{
    public CartWindow()
    {
        InitializeComponent();
        RefreshCart();
    }

    private void RefreshCart()
    {
        lbCartItems.ItemsSource = null;
        lbCartItems.ItemsSource = CartManager.Items.ToList();
        tbTotalSum.Text = $"Итого: {CartManager.TotalSum:N0} ₽ ({CartManager.TotalCount} шт.)";
    }

    private void BtnPlus_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is CartItem item)
        {
            if (item.Quantity < item.StockItem.Quantity)
            {
                item.Quantity++;
                RefreshCart();
            }
        }
    }

    private void BtnMinus_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is CartItem item)
        {
            if (item.Quantity > 1)
            {
                item.Quantity--;
            }
            else
            {
                CartManager.Items.Remove(item);
            }
            RefreshCart();
        }
    }

    private void BtnRemoveItem_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is CartItem item)
        {
            CartManager.Items.Remove(item);
            RefreshCart();
        }
    }

    private void BtnClear_Click(object? sender, RoutedEventArgs e)
    {
        CartManager.Clear();
        RefreshCart();
    }

    private async void BtnCheckout_Click(object? sender, RoutedEventArgs e)
    {
        if (CartManager.Items.Count == 0)
        {
            await Dialogs.ShowMessage(this, "Корзина пуста. Добавьте товары перед оформлением.");
            return;
        }

        int userId;
        if (LoginWindow.CurrentUser != null)
        {
            userId = LoginWindow.CurrentUser.UserId;
        }
        else
        {
            // Для неавторизованного гостя заказ привязывается к первому клиенту в базе
            var defaultClient = App.Db.Users.FirstOrDefault(u => u.RoleId == 3) ?? App.Db.Users.First();
            userId = defaultClient.UserId;
        }

        try
        {
            var order = new Order
            {
                OrderDate = DateOnly.FromDateTime(DateTime.Today),
                UserId = userId
            };

            App.Db.Orders.Add(order);
            App.Db.SaveChanges();

            foreach (var cartItem in CartManager.Items)
            {
                var orderItem = new OrderItem
                {
                    OrderId = order.OrderId,
                    StockItemId = cartItem.StockItem.StockItemId,
                    Quantity = cartItem.Quantity,
                    UnitPrice = cartItem.UnitPrice
                };
                App.Db.OrderItems.Add(orderItem);

                // Уменьшаем количество на складе
                var stock = App.Db.StockItems.Find(cartItem.StockItem.StockItemId);
                if (stock != null)
                {
                    stock.Quantity = Math.Max(0, stock.Quantity - cartItem.Quantity);
                }
            }

            App.Db.SaveChanges();
            CartManager.Clear();
            RefreshCart();

            await Dialogs.ShowMessage(this, $"Заказ №{order.OrderId} успешно оформлен!");
            Close();
        }
        catch (Exception ex)
        {
            await Dialogs.ShowMessage(this, "Ошибка сохранения заказа: " + ex.Message, "Ошибка");
        }
    }
}

Views/OrdersWindow.axaml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="ChudoObuv.Views.OrdersWindow"
        x:CompileBindings="False"
        Title="Управление заказами — Чудо Обувь"
        Icon="/Resources/app.ico"
        Height="650" Width="800" MinWidth="600" MinHeight="400"
        WindowStartupLocation="CenterOwner"
        Background="{DynamicResource MainBg}">
    <DockPanel Margin="18">
        <Border DockPanel.Dock="Top" BorderThickness="0,0,0,1" BorderBrush="{DynamicResource Accent}" Padding="0,0,0,10">
            <Grid ColumnDefinitions="*,Auto">
                <TextBlock Text="Список заказов" FontSize="20" FontWeight="Bold" Foreground="{DynamicResource TextBrush}"/>
                <StackPanel Grid.Column="1" Orientation="Horizontal" Spacing="8">
                    <Button x:Name="btnEditOrder" Content="Редактировать заказ" Padding="12,5"
                            Background="{DynamicResource Accent}" Foreground="White"
                            Click="BtnEditOrder_Click"/>
                    <Button x:Name="btnDeleteOrder" Content="Удалить заказ" Padding="12,5"
                            Background="{DynamicResource ErrorBrush}" Foreground="White"
                            Click="BtnDeleteOrder_Click"/>
                </StackPanel>
            </Grid>
        </Border>

        <Grid ColumnDefinitions="*,12,*">
            <!-- Список всех заказов -->
            <DockPanel Grid.Column="0">
                <TextBlock DockPanel.Dock="Top" Text="Заказы:" FontWeight="SemiBold" Margin="0,8,0,6"
                           Foreground="{DynamicResource MutedBrush}"/>
                <ListBox x:Name="lbOrders" Background="Transparent" SelectionChanged="LbOrders_SelectionChanged">
                    <ListBox.ItemTemplate>
                        <DataTemplate>
                            <Border Margin="0,0,0,6" Padding="10" CornerRadius="2"
                                    Background="White" BorderBrush="{DynamicResource Accent}" BorderThickness="1">
                                <StackPanel>
                                    <TextBlock Text="{Binding SummaryText}" FontWeight="SemiBold" TextWrapping="Wrap"
                                               Foreground="{DynamicResource TextBrush}"/>
                                    <TextBlock Text="{Binding CustomerFio}" Margin="0,3,0,0"
                                               Foreground="{DynamicResource MutedBrush}"/>
                                </StackPanel>
                            </Border>
                        </DataTemplate>
                    </ListBox.ItemTemplate>
                </ListBox>
            </DockPanel>

            <!-- Состав выбранного заказа -->
            <DockPanel Grid.Column="2">
                <TextBlock DockPanel.Dock="Top" Text="Состав выбранного заказа:" FontWeight="SemiBold" Margin="0,8,0,6"
                           Foreground="{DynamicResource MutedBrush}"/>
                <ListBox x:Name="lbOrderItems" Background="Transparent">
                    <ListBox.ItemTemplate>
                        <DataTemplate>
                            <Border Margin="0,0,0,6" Padding="10" CornerRadius="2"
                                    Background="White" BorderBrush="{DynamicResource Accent}" BorderThickness="1">
                                <StackPanel>
                                    <TextBlock Text="{Binding ProductName}" FontWeight="SemiBold"
                                               Foreground="{DynamicResource TextBrush}"/>
                                    <TextBlock Text="{Binding DisplayText}" Margin="0,3,0,0"
                                               Foreground="{DynamicResource MutedBrush}"/>
                                </StackPanel>
                            </Border>
                        </DataTemplate>
                    </ListBox.ItemTemplate>
                </ListBox>
            </DockPanel>
        </Grid>
    </DockPanel>
</Window>

Views/OrdersWindow.axaml.cs
using System;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using ChudoObuv.Models;
using Microsoft.EntityFrameworkCore;

namespace ChudoObuv.Views;

public partial class OrdersWindow : Window
{
    public OrdersWindow()
    {
        InitializeComponent();
        ApplyPermissions();
        LoadOrders();
    }

    private void ApplyPermissions()
    {
        var user = LoginWindow.CurrentUser;
        bool isAdmin = user != null && user.RoleId == 1;
        btnEditOrder.IsVisible = isAdmin;
        btnDeleteOrder.IsVisible = isAdmin;
    }

    public void LoadOrders()
    {
        var orders = App.Db.Orders
            .Include(o => o.User)
            .Include(o => o.OrderItems)
                .ThenInclude(oi => oi.StockItem)
                    .ThenInclude(si => si.Product)
            .OrderByDescending(o => o.OrderId)
            .ToList();

        lbOrders.ItemsSource = orders;
        lbOrderItems.ItemsSource = null;
    }

    private void LbOrders_SelectionChanged(object? sender, SelectionChangedEventArgs e)
    {
        if (lbOrders.SelectedItem is Order order)
        {
            lbOrderItems.ItemsSource = order.OrderItems.ToList();
        }
        else
        {
            lbOrderItems.ItemsSource = null;
        }
    }

    private async void BtnEditOrder_Click(object? sender, RoutedEventArgs e)
    {
        if (lbOrders.SelectedItem is not Order order)
        {
            await Dialogs.ShowMessage(this, "Выберите заказ для редактирования.");
            return;
        }

        var editWindow = new OrderEditWindow(order);
        await editWindow.ShowDialog(this);
        LoadOrders();
    }

    private async void BtnDeleteOrder_Click(object? sender, RoutedEventArgs e)
    {
        if (lbOrders.SelectedItem is not Order order)
        {
            await Dialogs.ShowMessage(this, "Выберите заказ для удаления.");
            return;
        }

        bool ok = await Dialogs.Confirm(this, $"Удалить заказ №{order.OrderId}? Товары вернутся на склад.");
        if (!ok) return;

        try
        {
            // Возвращаем остатки на склад
            foreach (var item in order.OrderItems)
            {
                var stock = App.Db.StockItems.Find(item.StockItemId);
                if (stock != null)
                {
                    stock.Quantity += item.Quantity;
                }
            }

            App.Db.OrderItems.RemoveRange(order.OrderItems);
            App.Db.Orders.Remove(order);
            App.Db.SaveChanges();

            await Dialogs.ShowMessage(this, $"Заказ №{order.OrderId} успешно удален.");
            LoadOrders();
        }
        catch (Exception ex)
        {
            await Dialogs.ShowMessage(this, "Ошибка удаления заказа: " + ex.Message, "Ошибка");
        }
    }
}

Views/OrderEditWindow.axaml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="ChudoObuv.Views.OrderEditWindow"
        x:CompileBindings="False"
        Title="Редактирование заказа — Чудо Обувь"
        Icon="/Resources/app.ico"
        Height="480" Width="600" MinWidth="480" MinHeight="320"
        WindowStartupLocation="CenterOwner"
        Background="{DynamicResource MainBg}">
    <DockPanel Margin="18">
        <Border DockPanel.Dock="Top" BorderThickness="0,0,0,1" BorderBrush="{DynamicResource Accent}" Padding="0,0,0,10">
            <TextBlock x:Name="tbHeader" Text="Редактирование заказа" FontSize="18" FontWeight="Bold"
                       Foreground="{DynamicResource TextBrush}"/>
        </Border>

        <Border DockPanel.Dock="Bottom" BorderThickness="0,1,0,0" BorderBrush="{DynamicResource Accent}" Padding="0,12,0,0" Margin="0,10,0,0">
            <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" Spacing="10">
                <Button Content="Закрыть" Padding="14,6" Background="{DynamicResource AltBg}"
                        Foreground="{DynamicResource TextBrush}" Click="BtnClose_Click"/>
            </StackPanel>
        </Border>

        <ListBox x:Name="lbItems" Margin="0,10,0,0" Background="Transparent">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <Border Margin="0,0,0,6" Padding="10" CornerRadius="2"
                            Background="White" BorderBrush="{DynamicResource Accent}" BorderThickness="1">
                        <Grid ColumnDefinitions="*,Auto,Auto">
                            <StackPanel Grid.Column="0" VerticalAlignment="Center">
                                <TextBlock Text="{Binding ProductName}" FontSize="15" FontWeight="SemiBold"
                                           Foreground="{DynamicResource TextBrush}"/>
                                <TextBlock Text="{Binding DisplayText}" Margin="0,3,0,0"
                                           Foreground="{DynamicResource MutedBrush}"/>
                            </StackPanel>

                            <StackPanel Grid.Column="1" Orientation="Horizontal" VerticalAlignment="Center" Margin="10,0">
                                <Button Content="-" Width="28" Height="28" Padding="0"
                                        HorizontalContentAlignment="Center" VerticalContentAlignment="Center"
                                        Tag="{Binding}" Click="BtnMinus_Click"/>
                                <TextBlock Text="{Binding Quantity}" VerticalAlignment="Center" Margin="8,0"
                                           FontWeight="Bold"/>
                                <Button Content="+" Width="28" Height="28" Padding="0"
                                        HorizontalContentAlignment="Center" VerticalContentAlignment="Center"
                                        Tag="{Binding}" Click="BtnPlus_Click"/>
                            </StackPanel>

                            <Button Grid.Column="2" Content="Удалить" Padding="8,4"
                                    Background="{DynamicResource AltBg}" Foreground="{DynamicResource TextBrush}"
                                    Tag="{Binding}" Click="BtnRemove_Click"/>
                        </Grid>
                    </Border>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </DockPanel>
</Window>

Views/OrderEditWindow.axaml.cs
using System;
using System.Linq;
using Avalonia.Controls;
using Avalonia.Interactivity;
using ChudoObuv.Models;

namespace ChudoObuv.Views;

public partial class OrderEditWindow : Window
{
    private readonly Order _order;

    public OrderEditWindow()
    {
        InitializeComponent();
        _order = new Order();
    }

    public OrderEditWindow(Order order) : this()
    {
        _order = order;
        tbHeader.Text = $"Редактирование заказа №{order.OrderId} ({order.CustomerFio})";
        RefreshItems();
    }

    private void RefreshItems()
    {
        lbItems.ItemsSource = null;
        lbItems.ItemsSource = _order.OrderItems.ToList();
    }

    private async void BtnPlus_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is OrderItem item)
        {
            var stock = App.Db.StockItems.Find(item.StockItemId);
            if (stock != null && stock.Quantity > 0)
            {
                stock.Quantity--;
                item.Quantity++;
                App.Db.SaveChanges();
                RefreshItems();
            }
            else
            {
                await Dialogs.ShowMessage(this, "На складе нет доступного количества этого размера.");
            }
        }
    }

    private async void BtnMinus_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is OrderItem item)
        {
            if (item.Quantity > 1)
            {
                var stock = App.Db.StockItems.Find(item.StockItemId);
                if (stock != null) stock.Quantity++;
                item.Quantity--;
                App.Db.SaveChanges();
                RefreshItems();
            }
            else
            {
                await RemoveItem(item);
            }
        }
    }

    private async void BtnRemove_Click(object? sender, RoutedEventArgs e)
    {
        if (sender is Button btn && btn.Tag is OrderItem item)
        {
            await RemoveItem(item);
        }
    }

    private async Task RemoveItem(OrderItem item)
    {
        bool ok = await Dialogs.Confirm(this, $"Удалить позицию «{item.ProductName}» из заказа?");
        if (!ok) return;

        var stock = App.Db.StockItems.Find(item.StockItemId);
        if (stock != null)
        {
            stock.Quantity += item.Quantity;
        }

        App.Db.OrderItems.Remove(item);
        _order.OrderItems.Remove(item);
        App.Db.SaveChanges();
        RefreshItems();
    }

    private void BtnClose_Click(object? sender, RoutedEventArgs e) => Close();
}

Views/ProductUserControl.axaml
<UserControl xmlns="https://github.com/avaloniaui"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             mc:Ignorable="d" d:DesignWidth="700" d:DesignHeight="160"
             x:Class="ChudoObuv.Views.ProductUserControl"
             x:CompileBindings="False">

    <Border x:Name="Card" Margin="0,0,0,8" Background="Transparent" BorderThickness="2"
            BorderBrush="Transparent">
        <Grid ColumnDefinitions="136,*,Auto">

            <Border Grid.Column="0" Background="White" BorderThickness="1" BorderBrush="{DynamicResource Accent}"
                    Width="124" Height="124" VerticalAlignment="Top" HorizontalAlignment="Left">
                <Image x:Name="Photo" Stretch="Uniform" Margin="4"/>
            </Border>

            <StackPanel Grid.Column="1" Margin="0,0,12,0">
                <TextBlock x:Name="tbTitle" FontSize="17" Foreground="{DynamicResource TextBrush}"/>
                <TextBlock x:Name="tbCategory" Margin="0,2,0,0" Foreground="{DynamicResource MutedBrush}"/>
                <TextBlock x:Name="tbManufacturer" Margin="0,4,0,0" Foreground="{DynamicResource TextBrush}"/>
                <TextBlock x:Name="tbComposition" MaxHeight="40" Margin="0,2,0,0" TextWrapping="Wrap"
                           Foreground="{DynamicResource TextBrush}"/>
                <TextBlock x:Name="tbSizes" Margin="0,2,0,0" Foreground="{DynamicResource TextBrush}"/>
            </StackPanel>

            <StackPanel Grid.Column="2" MinWidth="130">
                <TextBlock x:Name="tbPrice" FontSize="22" HorizontalAlignment="Right" Foreground="{DynamicResource TextBrush}"/>
                <TextBlock x:Name="tbCount" HorizontalAlignment="Right" Margin="0,6,0,0" Foreground="{DynamicResource MutedBrush}"/>
            </StackPanel>
        </Grid>
    </Border>
</UserControl>

Views/ProductUserControl.axaml.cs
using Avalonia.Controls;
using Avalonia.Media;
using Avalonia.Media.Imaging;
using ChudoObuv.Models;

namespace ChudoObuv.Views;

public partial class ProductUserControl : UserControl
{
    public ProductUserControl()
    {
        InitializeComponent();
    }

    public Product? Product { get; set; }

    public void SetSelected(bool selected)
    {
        Card.BorderBrush = selected ? Brush.Parse("#3E7C79") : Brushes.Transparent;
    }

    public void LoadData(Product product)
    {
        Product = product;
        tbTitle.Text = product.ProductName;
        tbCategory.Text = product.CategoryText;
        tbManufacturer.Text = "Производитель: " + product.Manufacturer.ManufacturerName;
        tbComposition.Text = "Состав: " + product.Composition;
        ToolTip.SetTip(tbComposition, product.Description);
        tbSizes.Text = "Размеры: " + product.SizesText;
        tbCount.Text = $"В наличии: {product.TotalQuantity} пар";
        tbPrice.Text = product.PriceText;
        Photo.Source = LoadImage(product.ImagePath);
    }

    private static Bitmap? LoadImage(string path)
    {
        var file = File.Exists(path)
            ? path
            : Path.Combine(AppContext.BaseDirectory, "Images", "picture.png");
        if (!File.Exists(file))
            return null;

        try
        {
            using var stream = File.OpenRead(file);
            return Bitmap.DecodeToWidth(stream, 240);   // как DecodePixelWidth = 240 в WPF
        }
        catch
        {
            return null;
        }
    }
}

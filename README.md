# API .NET Core - Documentação Técnica Completa
## Implementação com DDD, CQRS, Clean Architecture e SQL Queries

## Índice
1. [Estrutura do Projeto](#1-estrutura-do-projeto)
2. [Configuração Inicial](#2-configuração-inicial)
3. [Domain Layer](#3-domain-layer)
4. [Infrastructure Layer](#4-infrastructure-layer)
5. [Application Layer](#5-application-layer)
6. [API Layer](#6-api-layer)
7. [SQL Queries e Dapper](#7-sql-queries-e-dapper)
8. [Testes](#8-testes)
9. [AWS RDS Integration](#9-aws-rds-integration)

## 1. Estrutura do Projeto

```
Solution/
├── src/
│   ├── Company.Project.Domain/
│   │   ├── Entities/
│   │   │   ├── Entity.cs
│   │   │   ├── Product.cs
│   │   │   ├── Order.cs
│   │   │   └── Customer.cs
│   │   ├── Interfaces/
│   │   │   ├── IRepository.cs
│   │   │   ├── IProductRepository.cs
│   │   │   └── IProductQueries.cs
│   │   ├── ValueObjects/
│   │   │   ├── Money.cs
│   │   │   └── Address.cs
│   │   └── Enums/
│   │       └── OrderStatus.cs
│   ├── Company.Project.Infrastructure/
│   │   ├── Context/
│   │   │   └── ApplicationDbContext.cs
│   │   ├── Repositories/
│   │   │   ├── Repository.cs
│   │   │   └── ProductRepository.cs
│   │   ├── Queries/
│   │   │   └── ProductQueries.cs
│   │   └── Configurations/
│   │       └── ProductConfiguration.cs
│   ├── Company.Project.Application/
│   │   ├── Commands/
│   │   │   ├── CreateProductCommand.cs
│   │   │   └── UpdateProductCommand.cs
│   │   ├── Queries/
│   │   │   ├── GetProductQuery.cs
│   │   │   └── GetProductsQuery.cs
│   │   ├── Handlers/
│   │   │   ├── CreateProductHandler.cs
│   │   │   └── GetProductHandler.cs
│   │   ├── DTOs/
│   │   │   └── ProductDto.cs
│   │   └── Mappings/
│   │       └── MappingProfile.cs
│   └── Company.Project.API/
│       ├── Controllers/
│       │   └── ProductsController.cs
│       └── Configurations/
│           └── DependencyInjection.cs
└── tests/
    └── Company.Project.Tests/
        ├── Unit/
        └── Integration/
```

## 2. Configuração Inicial

### 2.1 Pacotes NuGet Necessários

```xml
<!-- Domain -->
<ItemGroup>
  <PackageReference Include="FluentValidation" Version="11.x" />
</ItemGroup>

<!-- Infrastructure -->
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="7.x" />
  <PackageReference Include="Dapper" Version="2.x" />
  <PackageReference Include="MySql.Data" Version="8.x" />
  <PackageReference Include="AWSSDK.RDS" Version="3.x" />
</ItemGroup>

<!-- Application -->
<ItemGroup>
  <PackageReference Include="AutoMapper" Version="12.x" />
  <PackageReference Include="MediatR" Version="12.x" />
</ItemGroup>

<!-- API -->
<ItemGroup>
  <PackageReference Include="Swashbuckle.AspNetCore" Version="6.x" />
</ItemGroup>
```

## 3. Domain Layer

### 3.1 Base Entity
```csharp
public abstract class Entity
{
    public Guid Id { get; protected set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }

    protected Entity()
    {
        Id = Guid.NewGuid();
        CreatedAt = DateTime.UtcNow;
    }

    public void SetUpdatedAt()
    {
        UpdatedAt = DateTime.UtcNow;
    }
}
```

### 3.2 Product Entity
```csharp
public class Product : Entity
{
    public string Name { get; private set; }
    public string Description { get; private set; }
    public Money Price { get; private set; }
    public int StockQuantity { get; private set; }
    public bool Active { get; private set; }

    protected Product() { } // For EF Core

    public Product(string name, string description, decimal price)
    {
        Name = name;
        Description = description;
        Price = new Money(price);
        Active = true;
        Validate();
    }

    public void UpdateStock(int quantity)
    {
        if (quantity < 0)
            throw new DomainException("Quantidade não pode ser negativa");

        StockQuantity = quantity;
        SetUpdatedAt();
    }

    public void UpdatePrice(decimal newPrice)
    {
        Price = new Money(newPrice);
        SetUpdatedAt();
        Validate();
    }

    private void Validate()
    {
        if (string.IsNullOrEmpty(Name))
            throw new DomainException("Nome é obrigatório");

        if (Price.Value < 0)
            throw new DomainException("Preço não pode ser negativo");
    }
}
```

### 3.3 Value Object
```csharp
public class Money : IEquatable<Money>
{
    public decimal Value { get; private set; }

    public Money(decimal value)
    {
        Value = value;
        Validate();
    }

    private void Validate()
    {
        if (Value < 0)
            throw new DomainException("Valor monetário não pode ser negativo");
    }

    public bool Equals(Money other)
    {
        if (other is null) return false;
        return Value == other.Value;
    }

    public override bool Equals(object obj)
    {
        return obj is Money money && Equals(money);
    }

    public override int GetHashCode()
    {
        return Value.GetHashCode();
    }
}
```

## 4. Infrastructure Layer

### 4.1 DbContext
```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}
```

### 4.2 Product Repository
```csharp
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context)
    {
        _context = context;
    }

    public async Task<Product> GetByIdAsync(Guid id)
    {
        return await _context.Products.FindAsync(id);
    }

    public async Task<IEnumerable<Product>> GetAllAsync()
    {
        return await _context.Products.Where(p => p.Active).ToListAsync();
    }

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }

    public async Task UpdateAsync(Product product)
    {
        _context.Entry(product).State = EntityState.Modified;
        await _context.SaveChangesAsync();
    }
}
```

### 4.3 SQL Queries com Dapper
```csharp
public class ProductQueries : IProductQueries
{
    private readonly string _connectionString;

    public ProductQueries(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public async Task<ProductDetailDto> GetProductDetailsAsync(Guid productId)
    {
        using var connection = new MySqlConnection(_connectionString);
        const string sql = @"
            SELECT 
                p.Id,
                p.Name,
                p.Description,
                p.Price_Value as Price,
                p.StockQuantity,
                p.Active,
                p.CreatedAt,
                p.UpdatedAt,
                (SELECT COUNT(*) FROM Orders o 
                 WHERE o.ProductId = p.Id) as TotalOrders,
                (SELECT SUM(oi.Quantity) FROM OrderItems oi 
                 INNER JOIN Orders o ON oi.OrderId = o.Id 
                 WHERE oi.ProductId = p.Id) as TotalUnitsSold
            FROM Products p
            WHERE p.Id = @ProductId";

        return await connection.QueryFirstOrDefaultAsync<ProductDetailDto>(
            sql, new { ProductId = productId });
    }

    public async Task<IEnumerable<ProductSalesDto>> GetTopSellingProductsAsync(
        DateTime startDate, DateTime endDate, int top = 10)
    {
        using var connection = new MySqlConnection(_connectionString);
        const string sql = @"
            SELECT 
                p.Id,
                p.Name,
                SUM(oi.Quantity) as TotalQuantitySold,
                SUM(oi.Quantity * oi.UnitPrice) as TotalRevenue
            FROM Products p
            INNER JOIN OrderItems oi ON p.Id = oi.ProductId
            INNER JOIN Orders o ON oi.OrderId = o.Id
            WHERE o.OrderDate BETWEEN @StartDate AND @EndDate
            GROUP BY p.Id, p.Name
            ORDER BY TotalQuantitySold DESC
            LIMIT @Top";

        return await connection.QueryAsync<ProductSalesDto>(
            sql, new { StartDate = startDate, EndDate = endDate, Top = top });
    }

    public async Task<PaginatedResult<ProductListDto>> GetPaginatedProductsAsync(
        int page, int pageSize, string searchTerm = null)
    {
        using var connection = new MySqlConnection(_connectionString);
        
        var parameters = new DynamicParameters();
        parameters.Add("@Skip", (page - 1) * pageSize);
        parameters.Add("@Take", pageSize);
        parameters.Add("@SearchTerm", $"%{searchTerm}%");

        var whereCause = string.IsNullOrEmpty(searchTerm) 
            ? "" 
            : "WHERE p.Name LIKE @SearchTerm OR p.Description LIKE @SearchTerm";

        var sqlCount = $@"
            SELECT COUNT(*) 
            FROM Products p 
            {whereCause}";

        var sql = $@"
            SELECT 
                p.Id,
                p.Name,
                p.Description,
                p.Price_Value as Price,
                p.StockQuantity,
                p.Active
            FROM Products p
            {whereCause}
            ORDER BY p.Name
            LIMIT @Skip, @Take";

        using var multi = await connection.QueryMultipleAsync(
            $"{sqlCount}; {sql}", parameters);

        var total = await multi.ReadFirstAsync<int>();
        var items = await multi.ReadAsync<ProductListDto>();

        return new PaginatedResult<ProductListDto>
        {
            Items = items.ToList(),
            Total = total,
            Page = page,
            PageSize = pageSize,
            TotalPages = (int)Math.Ceiling(total / (double)pageSize)
        };
    }
}
```

## 5. Application Layer

### 5.1 Commands
```csharp
public class CreateProductCommand : IRequest<ProductDto>
{
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int InitialStock { get; set; }
}

public class CreateProductCommandHandler : IRequestHandler<CreateProductCommand, ProductDto>
{
    private readonly IProductRepository _productRepository;
    private readonly IMapper _mapper;

    public CreateProductCommandHandler(
        IProductRepository productRepository,
        IMapper mapper)
    {
        _productRepository = productRepository;
        _mapper = mapper;
    }

    public async Task<ProductDto> Handle(
        CreateProductCommand request,
        CancellationToken cancellationToken)
    {
        var product = new Product(
            request.Name,
            request.Description,
            request.Price);

        product.UpdateStock(request.InitialStock);

        await _productRepository.AddAsync(product);

        return _mapper.Map<ProductDto>(product);
    }
}
```

### 5.2 Queries
```csharp
public class GetProductDetailsQuery : IRequest<ProductDetailDto>
{
    public Guid Id { get; set; }
}

public class GetProductDetailsQueryHandler 
    : IRequestHandler<GetProductDetailsQuery, ProductDetailDto>
{
    private readonly IProductQueries _productQueries;

    public GetProductDetailsQueryHandler(IProductQueries productQueries)
    {
        _productQueries = productQueries;
    }

    public async Task<ProductDetailDto> Handle(
        GetProductDetailsQuery request,
        CancellationToken cancellationToken)
    {
        return await _productQueries.GetProductDetailsAsync(request.Id);
    }
}
```

### 5.3 DTOs
```csharp
public class ProductDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
    public bool Active { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? UpdatedAt { get; set; }
}

public class ProductDetailDto : ProductDto
{
    public int TotalOrders { get; set; }
    public int TotalUnitsSold { get; set; }
}

public class ProductSalesDto
{
    public Guid Id { get; set; }
    public string Name { get; set; }
    public int TotalQuantitySold { get; set; }
    public decimal TotalRevenue { get; set; }
}

public class PaginatedResult<T>
{
    public List<T> Items { get; set; }
    public int Total { get; set; }
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalPages { get; set; }
}
```

## 6. API Layer

### 6.1 Controller
```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<ProductDetailDto>> GetById(Guid id)
    {
        var query = new GetProductDetailsQuery { Id = id };
        var result = await _mediator.Send(query);

        if (result == null)
            return NotFound();

        return Ok(result);
    }

    [HttpGet]
    public async Task<ActionResult<PaginatedResult<ProductListDto>>> GetPaginated(
        [FromQuery] int page = 1

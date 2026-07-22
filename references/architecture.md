# Architecture Reference

## Layering rules

**Model → Repository → Service → Component.** Each layer only talks to the one directly below it.

- Components inject `IEntityService`, never `IEntityRepository` or `DbContext` directly. This keeps data-access swappable and keeps validation/business-rule logic out of the UI layer.
- Repositories only do data access (EF Core queries). No business rules, no validation beyond what EF Core itself enforces.
- Services own business rules, orchestration across repositories, and any validation beyond `DataAnnotations`.

## Generic base repository

Use a generic base to avoid boilerplate per entity, and override/extend for entities that need custom queries.

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    void Update(T entity);
    void Delete(T entity);
    Task<int> SaveChangesAsync();
}

public class Repository<T> : IRepository<T> where T : class
{
    protected readonly AppDbContext Context;
    protected readonly DbSet<T> DbSet;

    public Repository(AppDbContext context)
    {
        Context = context;
        DbSet = context.Set<T>();
    }

    public async Task<T?> GetByIdAsync(int id) => await DbSet.FindAsync(id);
    public async Task<List<T>> GetAllAsync() => await DbSet.ToListAsync();
    public async Task AddAsync(T entity) => await DbSet.AddAsync(entity);
    public void Update(T entity) => DbSet.Update(entity);
    public void Delete(T entity) => DbSet.Remove(entity);
    public async Task<int> SaveChangesAsync() => await Context.SaveChangesAsync();
}
```

For entity-specific queries, create `IProductRepository : IRepository<Product>` with the extra method, and `ProductRepository : Repository<Product>, IProductRepository`.

## Service layer example

```csharp
public interface IProductService
{
    Task<List<Product>> GetAllAsync();
    Task<Product?> GetByIdAsync(int id);
    Task CreateAsync(Product product);
    Task UpdateAsync(Product product);
    Task DeleteAsync(int id);
}

public class ProductService : IProductService
{
    private readonly IProductRepository _repository;

    public ProductService(IProductRepository repository) => _repository = repository;

    public Task<List<Product>> GetAllAsync() => _repository.GetAllAsync();
    public Task<Product?> GetByIdAsync(int id) => _repository.GetByIdAsync(id);

    public async Task CreateAsync(Product product)
    {
        // business rules beyond DataAnnotations go here
        await _repository.AddAsync(product);
        await _repository.SaveChangesAsync();
    }

    public async Task UpdateAsync(Product product)
    {
        _repository.Update(product);
        await _repository.SaveChangesAsync();
    }

    public async Task DeleteAsync(int id)
    {
        var entity = await _repository.GetByIdAsync(id);
        if (entity is not null)
        {
            _repository.Delete(entity);
            await _repository.SaveChangesAsync();
        }
    }
}
```

## DI registration (Program.cs)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

builder.Services.AddScoped<IProductRepository, ProductRepository>();
builder.Services.AddScoped<IProductService, ProductService>();
```

Repositories and services are `Scoped` — matches the EF Core `DbContext` lifetime and Blazor Server's per-circuit scope. Do not register them as `Singleton`; a shared `DbContext` across users/circuits causes concurrency bugs.

## Blazor Server vs. Blazor Hybrid differences

- **Blazor Server**: DI scope is per-circuit (per user connection). `Scoped` services are safe as above. Avoid holding large in-memory state in components for long-lived circuits — it stays resident server-side.
- **Blazor Hybrid (MAUI)**: DI scope is typically per-app-instance (single user, single process) via `MauiProgram.cs`'s `builder.Services`. SQLite is more common than SQL Server for the local `DbContext`. `IRepository<T>`/`IEntityService` pattern is unchanged.
- **Blazor WebAssembly** (if the project has a `.Client` project or uses `AddInteractiveWebAssemblyComponents()`): there is no server-side `DbContext` reachable directly from the component — the repository layer must sit behind an HTTP API (a minimal API or controller), and the "repository" the component talks to is really an `HttpClient`-based service calling that API. Flag this distinction explicitly to the user if detected, since the scaffolding steps differ (you scaffold an API endpoint too, not just a repository class).

# Component Pattern Reference

Full code examples backing each rule in SKILL.md's "Component pattern conventions" section.

## Official directive order

Per the ASP.NET Core Razor components docs, directives at the top of a `.razor` file follow this order, with no blank lines between them and exactly one blank line before the first markup:

1. `@page`
2. `@rendermode` (.NET 8+, unified Blazor Web App template only)
3. `@using` — `System` namespaces alphabetically, then `Microsoft` namespaces alphabetically, then third-party namespaces alphabetically, then app namespaces alphabetically
4. Other directives (`@attribute`, `@implements`, `@inherits`, `@inject`, etc.), alphabetical by directive/type

```razor
@page "/product-detail/{Id:int}"
@rendermode InteractiveServer
@using System.ComponentModel.DataAnnotations
@using MyApp.Components.Layout
@attribute [Authorize]
@implements IDisposable
@inject IProductService ProductService
@inject NavigationManager Nav

<PageTitle>Product Detail</PageTitle>
```

## List component

```razor
@page "/products"
@inject IProductService ProductService
@inject NavigationManager Nav

<h3>Products</h3>
<a class="btn btn-primary mb-3" href="/products/create">+ New Product</a>

@if (_loading)
{
    <p><em>Loading...</em></p>
}
else if (!_products.Any())
{
    <p>No products yet.</p>
}
else
{
    <table class="table table-striped">
        <thead>
            <tr><th>Name</th><th>Price</th><th></th></tr>
        </thead>
        <tbody>
            @foreach (var p in _products)
            {
                <tr>
                    <td>@p.Name</td>
                    <td>@p.Price.ToString("C")</td>
                    <td>
                        <a class="btn btn-sm btn-secondary" href="@($"/products/edit/{p.Id}")">Edit</a>
                        <button class="btn btn-sm btn-danger" @onclick="() => ConfirmDelete(p)">Delete</button>
                    </td>
                </tr>
            }
        </tbody>
    </table>
}

@code {
    private List<Product> _products = new();
    private bool _loading = true;
    private Product? _pendingDelete;

    protected override async Task OnInitializedAsync()
    {
        _products = await ProductService.GetAllAsync();
        _loading = false;
    }

    private void ConfirmDelete(Product p) => _pendingDelete = p;

    private async Task DeleteConfirmed()
    {
        if (_pendingDelete is null) return;
        await ProductService.DeleteAsync(_pendingDelete.Id);
        _products.Remove(_pendingDelete);
        _pendingDelete = null;
    }
}
```

Notes: data loads in `OnInitializedAsync`, not the constructor; loading/empty states are explicit; delete goes through a confirmation step rather than firing immediately (expand `_pendingDelete` into a real Bootstrap modal in the actual scaffold — this snippet keeps it minimal for readability).

## Shared form component (used by both Create and Edit)

```razor
@* ProductForm.razor *@
<EditForm Model="Model" OnValidSubmit="HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div class="mb-3">
        <label class="form-label">Name</label>
        <InputText class="form-control" @bind-Value="Model.Name" />
        <ValidationMessage For="() => Model.Name" />
    </div>

    <div class="mb-3">
        <label class="form-label">Price</label>
        <InputNumber class="form-control" @bind-Value="Model.Price" />
        <ValidationMessage For="() => Model.Price" />
    </div>

    <button type="submit" class="btn btn-primary">Save</button>
</EditForm>

@code {
    [Parameter, EditorRequired] public Product Model { get; set; } = default!;
    [Parameter] public EventCallback<Product> OnValidSubmit { get; set; }

    private Task HandleValidSubmit() => OnValidSubmit.InvokeAsync(Model);
}
```

## Create / Edit pages hosting the shared form

```razor
@* ProductCreate.razor *@
@page "/products/create"
@inject IProductService ProductService
@inject NavigationManager Nav

<h3>New Product</h3>
<ProductForm Model="_model" OnValidSubmit="Save" />

@code {
    private Product _model = new();

    private async Task Save(Product product)
    {
        await ProductService.CreateAsync(product);
        Nav.NavigateTo("/products");
    }
}
```

```razor
@* ProductEdit.razor *@
@page "/products/edit/{Id:int}"
@inject IProductService ProductService
@inject NavigationManager Nav

<h3>Edit Product</h3>
@if (_model is not null)
{
    <ProductForm Model="_model" OnValidSubmit="Save" />
}

@code {
    [Parameter] public int Id { get; set; }
    private Product? _model;

    protected override async Task OnInitializedAsync()
    {
        _model = await ProductService.GetByIdAsync(Id);
    }

    private async Task Save(Product product)
    {
        await ProductService.UpdateAsync(product);
        Nav.NavigateTo("/products");
    }
}
```

Note `OnParametersSetAsync` vs `OnInitializedAsync`: for `ProductEdit`, `OnInitializedAsync` is sufficient since the route parameter is set before first render and the component isn't reused across different `Id` values without a full navigation. If the same component instance can be reused for a different `Id` (e.g. via `NavLink` inside a persistent layout without a full page reload), reload in `OnParametersSetAsync` instead.

## IDisposable for event subscriptions

```csharp
public partial class SomeComponent : ComponentBase, IDisposable
{
    [Inject] private NavigationManager Nav { get; set; } = default!;

    protected override void OnInitialized()
    {
        Nav.LocationChanged += HandleLocationChanged;
    }

    private void HandleLocationChanged(object? sender, LocationChangedEventArgs e)
    {
        StateHasChanged();
    }

    public void Dispose()
    {
        Nav.LocationChanged -= HandleLocationChanged;
    }
}
```

Any component that subscribes to an event, timer, or JS interop callback must unsubscribe in `Dispose`/`DisposeAsync` — otherwise it leaks on Blazor Server (circuit-scoped, but the component instance and its subscription outlive the visible page) and on Hybrid (long process lifetime).

## Code-behind split (for components >~50 lines of logic)

```razor
@* ProductList.razor *@
@page "/products"
@inject IProductService ProductService
<!-- markup only -->
```

```csharp
// ProductList.razor.cs
public partial class ProductList : ComponentBase
{
    [Inject] private IProductService ProductService { get; set; } = default!;

    private List<Product> _products = new();

    protected override async Task OnInitializedAsync()
    {
        _products = await ProductService.GetAllAsync();
    }
}
```

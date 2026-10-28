---
name: blazor-crud-standards
description: >
  Scaffolds and reviews CRUD interfaces for any Blazor app — unified Blazor Web App template (.NET 8+, static SSR plus InteractiveServer/InteractiveWebAssembly/InteractiveAuto render modes), legacy Blazor Server, Blazor WebAssembly, or Blazor Hybrid (MAUI) — following Microsoft's official architectural and component patterns: EF Core with a repository layer, EditForm plus DataAnnotations validation, correct @rendermode and directive ordering, plain Bootstrap styling, correct lifecycle and DI. Use whenever the user asks to build, scaffold, generate, wire up, or "manage" an entity in a Blazor app (list/create/edit/delete pages, a "manager" page, a CRUD grid), or wants existing Blazor components/services/repositories reviewed or fixed against standard Microsoft patterns — even without saying "Blazor standards" explicitly. Trigger for any .razor file work, any Blazor hosting-model question, or requests to follow "standard Blazor patterns."
---

# Blazor CRUD Standards

A skill for scaffolding new entity CRUD interfaces and reviewing existing Blazor code, always following Microsoft's official architectural and component conventions — across the unified Blazor Web App template, legacy Blazor Server, Blazor WebAssembly, and Blazor Hybrid. Framework-agnostic — no assumption of ABP Framework, Telerik, MudBlazor, or any other add-on library. Default styling is plain Bootstrap (the Blazor project template default).

## Step 0: Detect project shape — never guess silently

Before writing anything, inspect the project to determine which template generation it's on and which hosting model(s) are active. Microsoft's current (.NET 8+) standard is the **unified Blazor Web App template**: a single project can mix static server-side rendering (SSR) with per-page or per-component interactivity, chosen via the `@rendermode` directive rather than a separate project type. Older, still-common project shapes (separate "Blazor Server" / "Blazor WebAssembly" templates, pre-.NET 8) remain valid — match whichever generation the project is actually on.

1. **Identify the template generation and hosting model(s)**:
   - **Unified Blazor Web App (.NET 8+, current standard)** signals: `App.razor` + a `Routes.razor` component, `MapRazorComponents<App>()` with `.AddInteractiveServerComponents()` and/or `.AddInteractiveWebAssemblyComponents()` in `Program.cs`, components under `Components/Pages/` and `Components/Layout/`. Interactivity is per-page/component via `@rendermode InteractiveServer|InteractiveWebAssembly|InteractiveAuto`, or global if set on `Routes`/`HeadOutlet` in `App.razor`. A component with no render mode uses **static SSR** (no interactivity) by design — don't "fix" this unless the user wants it interactive.
   - **Legacy Blazor Server** (pre-.NET 8) signals: `AddServerSideBlazor()`, a `_Host.cshtml`, `Pages/` + `Shared/` folders instead of `Components/Pages/` + `Components/Layout/`.
   - **Legacy Blazor WebAssembly** signals: `builder.RootComponents.Add<App>("#app")` in a WASM `Program.cs`, `wwwroot/index.html` as the entry point, no server-side `DbContext` reachable directly — repository access must go through an HTTP API (see `references/architecture.md`).
   - **Blazor Hybrid (MAUI)** signals: `Microsoft.Maui.Controls` reference, `net8.0-android`/`net8.0-ios` target frameworks, `BlazorWebView` in `MainPage.xaml`.
   - If a project mixes signals or nothing is present yet (brand-new project), **ask the user** which model to target instead of assuming.
2. **Existing conventions** — folder layout (`Models/`, `Services/`, `Repositories/`, and `Components/Pages/`+`Components/Layout/` on the unified template or `Pages/`+`Shared/` on legacy templates), existing `DbContext`, existing naming patterns. Match what's already there rather than introducing a new convention mid-project.
3. **UI library** — default to plain Bootstrap (the official template default). If the project already references Telerik, MudBlazor, Radzen, etc., match the existing library instead of introducing Bootstrap.

## Architecture (layering)

```
Model (POCO + DataAnnotations)
   ↓
IRepository<T> / Repository<T>   — EF Core, injected DbContext, no business logic
   ↓
IEntityService / EntityService   — business logic, injected repository, scoped DI
   ↓
Razor components                 — inject the service, never the repository directly
```

Full rules, DI registration snippets, and the generic base repository pattern are in `references/architecture.md`. Read it before scaffolding the data-access layers.

## Scaffolding a new entity — workflow

When asked to scaffold CRUD for an entity (e.g. "add a Product manager page"):

1. **Model** — POCO with `DataAnnotations` validation attributes (`[Required]`, `[StringLength]`, `[Range]`, etc.). Confirm the fields with the user if not fully specified.
2. **Repository** — interface + EF Core implementation. Use the generic base repository pattern from `references/architecture.md` unless the entity needs custom queries.
3. **Service** — interface + implementation wrapping the repository; this is where validation-beyond-DataAnnotations and business rules live. Register as `Scoped` in `Program.cs`.
4. **Razor components** (see `references/component-patterns.md` for full templates), placed in `Components/Pages/` on the unified template (or `Pages/` on legacy Blazor Server):
   - `{Entity}List.razor` — table view, loading state, empty state, delete confirmation modal
   - `{Entity}Form.razor` — shared `EditForm` component used by both create and edit, with `DataAnnotationsValidator` + `ValidationSummary`
   - `{Entity}Create.razor` / `{Entity}Edit.razor` — thin pages that host the shared form and handle the save/navigate flow
   - File names are **Pascal case** (`ProductList.razor`); `@page` route templates are **kebab case** (`@page "/product-list"`) — this is Microsoft's documented convention, not a style choice.
   - On the unified Blazor Web App template, add `@rendermode InteractiveServer` (or whichever mode the app uses — see "Render modes" below) to each page that needs event handlers/binding to work. Without it the page is static SSR and buttons/forms won't respond.
5. **Wiring** — register `@page` routes, register services/repositories in `Program.cs`, add a nav link in `NavMenu.razor` (`Components/Layout/NavMenu.razor` on the unified template).
6. Use the starter templates in `assets/templates/` as the literal starting point, then adapt field names/types to the entity.

## Render modes (unified Blazor Web App template, .NET 8+)

- A component with no `@rendermode` is **static SSR** — no event handlers, no binding. This is correct and intentional for read-only content; it is not a bug to fix.
- `@rendermode InteractiveServer` — interactive over a SignalR circuit (equivalent to classic Blazor Server behavior).
- `@rendermode InteractiveWebAssembly` — interactive entirely client-side; requires the entity's data access to go through an HTTP API rather than a direct repository/DbContext call (see `references/architecture.md`).
- `@rendermode InteractiveAuto` — server-rendered first, switches to WebAssembly once the client bundle downloads and caches.
- Apply per-page (`@rendermode InteractiveServer` under `@page`) or globally on `Routes`/`HeadOutlet` in `App.razor`. Match whatever the project already does — don't silently switch a static-SSR project to interactive or vice versa.
- Full official directive ordering (`@page`, then `@rendermode`, then `@using`, etc.) is in `references/component-patterns.md`.

## Component pattern conventions (Microsoft standard)

These are non-negotiable defaults — deviate only if the existing project already deviates consistently:

- `EditForm` + `DataAnnotationsValidator` + `ValidationSummary` (or `ValidationMessage` per-field) for every create/edit form. Never hand-roll validation against `InputText` alone.
- Load data in `OnInitializedAsync` (or `OnParametersSetAsync` if it depends on route/parameters) — never in the constructor, never blocking in `OnInitialized`.
- Implement `IDisposable`/`IAsyncDisposable` for any component that subscribes to events, timers, or `NavigationManager.LocationChanged`.
- `[Parameter] public T Prop { get; set; }` for all component parameters — plain auto-properties with no logic in the getter/setter. `[EditorRequired]` where the caller must supply it. Never use the `required` modifier or an `init` accessor on a `[Parameter]` property — Blazor sets parameters via reflection, which bypasses both, so `[EditorRequired]` is the only supported way to signal a required parameter (this is an explicit Microsoft "don't" — a common mistake to watch for in review mode).
- Keep `@code` blocks small; move logic over ~50 lines into a partial class code-behind file (`Component.razor.cs`).
- Call `StateHasChanged()` only after state changes that happen outside the normal Blazor render pipeline (e.g. after an `await` in an event triggered by something other than user interaction, or after a callback from JS interop).
- Respect nullable reference types — don't suppress warnings with `!` as a default habit.
- Use `CascadingParameter` only for genuinely cross-cutting concerns (auth state, theme) — not as a shortcut around passing parameters explicitly.

Full code examples for each of these live in `references/component-patterns.md`.

## Review mode

When asked to review or fix existing Blazor code:

1. Walk the checklist in `references/checklist.md`.
2. Report deviations with a one-line explanation and the fix — **don't silently rewrite** unless the user has asked you to fix it directly rather than review it.
3. Group findings by severity: correctness/bugs first (missing `IDisposable`, blocking async), then convention deviations (naming, layering), then style nits.

## Reference files

- `references/architecture.md` — layering rules, generic repository pattern, DI registration, WebAssembly-specific notes
- `references/component-patterns.md` — full Razor/C# code templates for each convention above
- `references/checklist.md` — the review checklist used in review mode
- `assets/templates/` — copy-and-adapt starter files for Model, Repository, Service, List, Form, Create/Edit pages

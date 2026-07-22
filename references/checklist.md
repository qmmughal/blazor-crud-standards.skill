# Review Checklist

Use in review mode. Check in this order; report findings grouped by severity.

## 1. Correctness / bugs (highest severity)

- [ ] No `async void` event handlers (should be `async Task`)
- [ ] No blocking calls on async code (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) inside components
- [ ] Components that subscribe to events/timers/JS interop implement `IDisposable`/`IAsyncDisposable` and actually unsubscribe
- [ ] `DbContext` is not captured as a field with a longer lifetime than the operation (no manual `Singleton` DbContext)
- [ ] Route parameters (`[Parameter]` bound from `@page` route) are reloaded correctly when the component is reused across navigations — check whether `OnInitializedAsync` should be `OnParametersSetAsync`

## 2. Layering violations

- [ ] Components inject a service (`IEntityService`), not a repository or `DbContext` directly
- [ ] Repositories contain no business rules — pure data access only
- [ ] Business rules / cross-repository orchestration live in the service layer, not scattered across components

## 3. Validation

- [ ] Every create/edit form uses `EditForm` + `DataAnnotationsValidator` (not manual validation against bare `<input>` or `InputText` alone)
- [ ] `ValidationSummary` or per-field `ValidationMessage` is present and rendered
- [ ] Model has appropriate `DataAnnotations` (`[Required]`, `[StringLength]`, `[Range]`, `[EmailAddress]`, etc.) matching the actual constraints

## 4. DI / registration

- [ ] Repositories and services registered as `Scoped` (not `Singleton`, which causes shared-`DbContext` bugs; not `Transient`, which is usually unnecessary overhead for these)
- [ ] Interfaces registered against their implementations (`IProductService` → `ProductService`), components depend on the interface

## 5. Component conventions / style

- [ ] On the unified Blazor Web App template: interactive pages (anything with event handlers or `@bind`) declare `@rendermode`; pages with no `@rendermode` are static SSR — confirm that's intentional, not a missed page
- [ ] Directive order follows the official convention: `@page`, `@rendermode`, `@using` (System → Microsoft → third-party → app, each block alphabetical), then other directives alphabetically — no blank lines between directives, one blank line before markup
- [ ] `[Parameter]` properties use plain `{ get; set; }` with no custom get/set logic, `[EditorRequired]` where the caller must supply a value — never `required` or `init` on a `[Parameter]` property (both are bypassed by Blazor's reflection-based parameter binding and are explicitly called out as unsupported)
- [ ] File names are Pascal case (`ProductList.razor`); routable `@page` templates are kebab case (`/product-list`)
- [ ] `@code` blocks under ~50 lines, or split into a `.razor.cs` partial class
- [ ] `StateHasChanged()` only called where actually needed (after state changes outside the normal render pipeline) — flag redundant calls
- [ ] Nullable reference types respected — flag habitual `!` null-forgiving operator use instead of real null handling
- [ ] `CascadingParameter` used only for genuinely cross-cutting concerns, not as a parameter-passing shortcut

## Reporting format

For each finding: **file:line** (if known) → what's wrong → one-line fix. Group under the five headings above. Don't silently rewrite files during a review — propose fixes and apply them only if the user asks you to fix rather than just review.

# Blazor CRUD Standards — Claude Plugin

A Claude plugin (one Agent Skill) that scaffolds and reviews CRUD interfaces for **any Blazor app** — the unified Blazor Web App template (.NET 8+, static SSR plus `InteractiveServer`/`InteractiveWebAssembly`/`InteractiveAuto` render modes), legacy Blazor Server, Blazor WebAssembly, or Blazor Hybrid (MAUI) — following Microsoft's official architectural and component patterns.

Framework-agnostic: no assumption of ABP Framework, Telerik, MudBlazor, or any other add-on library. Default styling is plain Bootstrap (the official Blazor project template default).

## What it does

Claude invokes this skill automatically whenever you ask it to build, scaffold, generate, wire up, or "manage" an entity in a Blazor app — list/create/edit/delete pages, a "manager" page, a CRUD grid — or to review/fix existing Blazor components, services, or repositories against standard Microsoft patterns. No need to say "Blazor standards" explicitly; it triggers on any `.razor` file work, Blazor hosting-model question, or request to follow "standard Blazor patterns."

**Scaffold mode** generates, in order:
1. **Model** — POCO with `DataAnnotations` validation
2. **Repository** — interface + EF Core implementation (generic base repository pattern)
3. **Service** — interface + implementation wrapping the repository, registered `Scoped`
4. **Razor components** — `{Entity}List.razor`, `{Entity}Form.razor`, `{Entity}Create.razor` / `{Entity}Edit.razor`
5. **Wiring** — `@page` routes, DI registration, nav link

**Review mode** walks a checklist and reports deviations (correctness first, then convention, then style nits) without silently rewriting your code.

Before doing either, the skill inspects the project to detect which Blazor template generation and hosting model is in play, rather than guessing or assuming .NET 8+ unified Blazor Web App by default.

## Repository contents

```
.claude-plugin/
├── plugin.json                           # Plugin manifest (name, version, author, license)
├── marketplace.json                      # Lets Claude Code add this repo as a plugin marketplace
└── icon.png                              # Directory listing icon
skills/blazor-crud-standards/
├── SKILL.md                              # Skill definition (frontmatter + instructions)
├── references/
│   ├── architecture.md                   # Layering rules, generic repository pattern, DI, WASM notes
│   ├── component-patterns.md             # Full Razor/C# code templates for each convention
│   └── checklist.md                      # Review-mode checklist
└── assets/templates/                     # Copy-and-adapt starter files
    ├── Model.cs.template
    ├── Repository.cs.template
    ├── Service.cs.template
    ├── List.razor.template
    ├── Form.razor.template
    └── CreateEdit.razor.template
```

## Architecture enforced by this skill

```
Model (POCO + DataAnnotations)
   ↓
IRepository<T> / Repository<T>   — EF Core, injected DbContext, no business logic
   ↓
IEntityService / EntityService   — business logic, injected repository, scoped DI
   ↓
Razor components                 — inject the service, never the repository directly
```

## Key conventions covered

- `EditForm` + `DataAnnotationsValidator` + `ValidationSummary` for every create/edit form
- Correct `@rendermode` and directive ordering on the unified Blazor Web App template
- Data loading in `OnInitializedAsync`/`OnParametersSetAsync`, never the constructor
- `IDisposable`/`IAsyncDisposable` for components subscribing to events, timers, or `NavigationManager.LocationChanged`
- `[EditorRequired]` instead of `required`/`init` on `[Parameter]` properties (Blazor sets parameters via reflection, which bypasses both)
- Pascal-case file names, kebab-case `@page` routes
- Plain Bootstrap styling by default; matches existing UI library if one is already in the project

## Installation

### Claude Code (plugin)

```bash
claude plugin marketplace add qmmughal/blazor-crud-standards.skill
claude plugin install blazor-crud-standards@qmmughal-blazor
```

Or from inside Claude Code: `/plugin marketplace add qmmughal/blazor-crud-standards.skill`, then
`/plugin install blazor-crud-standards@qmmughal-blazor`.

### Manual skill install

Copy just the skill folder into a skills directory — `~/.claude/skills/` for all projects, or
`<your-project>/.claude/skills/` for one project:

```bash
git clone https://github.com/qmmughal/blazor-crud-standards.skill.git
cp -r blazor-crud-standards.skill/skills/blazor-crud-standards ~/.claude/skills/
```

Keep the folder name `blazor-crud-standards` and the `SKILL.md` filename as-is — Claude discovers
skills by scanning for `SKILL.md` files under `skills/*/`.

## Usage

Once installed, just ask Claude Code naturally — no special invocation syntax needed:

```
"Add a Product manager page with list, create, and edit"
"Scaffold CRUD for a Customer entity"
"Review this Blazor component against Microsoft's standard patterns"
"Why isn't my button click working on this page?"   → triggers render-mode/interactivity review
```

Claude will detect your project's Blazor template generation and hosting model first (unified Blazor Web App, legacy Blazor Server, Blazor WebAssembly, or Blazor Hybrid) and ask before assuming, if the signals are ambiguous or the project is brand new.

## What this plugin runs and sends

Nothing. The plugin is instructions, reference Markdown, and C#/Razor template files only. It has no
hooks, no MCP servers, and no scripts; it makes no network requests, reads no credentials, and
collects no data. Templates are copied into your project only when you ask Claude to scaffold code.

## Trademarks

This is an independent community project, not affiliated with or endorsed by Microsoft or Anthropic.
Blazor, .NET, and Entity Framework Core are named only to describe what the skill works with.

## License

[MIT](./LICENSE)

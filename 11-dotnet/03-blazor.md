# Blazor

> **Tier:** T3 (essentials) · **Related:** [ASP.NET Core MVC](01-aspnet-core-mvc.md), [C# fundamentals](../03-languages/csharp/01-fundamentals.md), [React fundamentals](../06-react/01-fundamentals.md), [Real-time and WebSockets](../04-web-fundamentals/09-realtime-websockets.md), [EF Core and Dapper](02-entity-framework-and-dapper.md)
> **On your resume:** Blazor (skills list), from university work and the .NET side of your final-year project. Your production front-end work is React and React Native, so comparisons with React are your strongest angle.

## Quick revision
- Blazor builds interactive web UIs from **Razor components** (`.razor` files: markup plus C# in an `@code` block) instead of JavaScript.
- **Blazor Server**: components run on the server, and UI events and DOM diffs travel over a SignalR connection called a **circuit**. Small download and direct server access, but every interaction is a network round trip, the server holds state per user, and the UI stops when the connection drops.
- **Blazor WebAssembly**: the .NET runtime and your assemblies download and run in the browser. No per-user server state and offline-capable, but a bigger first download, and all code is visible to the client (no secrets).
- **.NET 8 Blazor Web App**: one project with a render mode per page or component: **Static SSR** (the default, no interactivity), **Interactive Server**, **Interactive WebAssembly**, **Interactive Auto** (Server first, WebAssembly once downloaded).
- **Streaming rendering** sends the page shell immediately and fills in content when async work finishes; **enhanced navigation** makes static SSR links and forms update the page via fetch, SPA-style.
- Components: `[Parameter]` props, `EventCallback<T>` for child-to-parent events, `RenderFragment ChildContent` for nested markup, `CascadingValue`/`[CascadingParameter]` for context.
- Lifecycle: `OnInitialized(Async)` -> `OnParametersSet(Async)` -> render -> `OnAfterRender(Async)(firstRender)` -> `Dispose`. Event handlers trigger re-renders automatically; external triggers need `InvokeAsync(StateHasChanged)`.
- `@bind` is two-way binding (`@bind:event="oninput"` updates per keystroke); forms use `EditForm`, `DataAnnotationsValidator` and `ValidationMessage`.
- DI via `@inject`; on Blazor Server a scoped service lives for the whole circuit, not one request. JS interop goes through `IJSRuntime`, only after the first render.

## 1. Razor components

```html
@page "/counter"
@rendermode InteractiveServer
<h1>Counter</h1>
<p>Current count: @count</p>
<button class="btn btn-primary" @onclick="Increment">Click me</button>

@code {
    private int count;
    private void Increment() => count++;    // the component re-renders automatically after the handler
}
```

A `.razor` file compiles into a C# class. Blazor renders it into a render tree, diffs the new tree against the previous one after each event, and applies only the changes to the DOM: the same idea as React's virtual DOM. The file name is the component name (`ProductCard.razor` is used as `<ProductCard />`), and `@page` makes a component routable. The selling point is full-stack C#: share models and validation between client and server and reuse .NET libraries, with JavaScript interop when you need it.

## 2. Hosting models: Blazor Server vs WebAssembly

| | Blazor Server | Blazor WebAssembly |
|---|---|---|
| Where C# runs | on the server | in the browser, on a WebAssembly .NET runtime |
| UI updates | events go to the server over SignalR; DOM diffs come back | computed locally in the browser |
| First load | small and fast | downloads the runtime and app assemblies (trimming, compression and caching help) |
| Server cost | memory per connected user (circuit state); scale-out needs sticky sessions or a SignalR service | can be served as static files; the server only hosts APIs |
| Offline | no; the UI shows a reconnect overlay when the connection drops | yes, can be a PWA |
| Security | code and secrets stay on the server | assemblies are downloadable: never ship secrets or trust client checks |
| Good fit | internal and line-of-business apps on reliable networks | public apps, offline use, offloading work to clients |

A circuit holds each user's component state in server memory; if the connection is lost and the circuit expires, that in-memory state is gone. Blazor Hybrid (inside .NET MAUI) is a third option that renders components in native apps.

## 3. The .NET 8 Blazor Web App and render modes

| Render mode | Runs on | Interactive | Notes |
|---|---|---|---|
| Static SSR | server | no: plain HTML, forms post back | the default; like Razor Pages, built from components |
| Interactive Server | server, over SignalR | yes | `@rendermode InteractiveServer` |
| Interactive WebAssembly | browser | yes | the component must live in the client project |
| Interactive Auto | server first, WebAssembly on later visits | yes | fast first load plus client-side execution, more moving parts |

```csharp
builder.Services.AddRazorComponents()                 // Program.cs of a Blazor Web App
    .AddInteractiveServerComponents()
    .AddInteractiveWebAssemblyComponents();

app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode()
    .AddInteractiveWebAssemblyRenderMode()
    .AddAdditionalAssemblies(typeof(MyApp.Client._Imports).Assembly);
```

- Set the mode per page or component (`@rendermode InteractiveServer`) or for the whole app on `<Routes @rendermode="InteractiveServer" />` in `App.razor`. Interactive components are **prerendered** on the server by default (fast first paint, SEO), then become interactive. So `OnInitializedAsync` can run twice and JS interop is unavailable during prerendering. Disable it with `@rendermode @(new InteractiveServerRenderMode(prerender: false))`, or carry data across with `PersistentComponentState`.
- **Streaming rendering** keeps static SSR pages responsive when data is slow; **enhanced navigation** intercepts links and form posts in static SSR, fetches the new page and patches the DOM (opt out with `data-enhance-nav="false"`). Static SSR forms need a `FormName` and a `[SupplyParameterFromForm]` model.

```html
@page "/weather"
@attribute [StreamRendering]
@inject WeatherService WeatherApi
@if (forecasts is null) { <p>Loading...</p> }
else { <ul>@foreach (var f in forecasts) { <li>@f.Date.ToShortDateString(): @f.Summary</li> }</ul> }

@code {
    private WeatherForecast[]? forecasts;
    protected override async Task OnInitializedAsync() =>
        forecasts = await WeatherApi.GetForecastsAsync();   // slow call: the page shell streams first
}
```

## 4. Components: parameters, events, child content, cascading values

```html
@* ProductCard.razor *@
<div class="card">
    <h3>@Product.Name</h3>
    @ChildContent
    <button @onclick="() => OnAdd.InvokeAsync(Product)">Add to cart</button>
</div>

@code {
    [Parameter, EditorRequired] public Product Product { get; set; } = default!;
    [Parameter] public EventCallback<Product> OnAdd { get; set; }        // child -> parent
    [Parameter] public RenderFragment? ChildContent { get; set; }        // markup between the tags
    [CascadingParameter] public UserSession? Session { get; set; }       // provided by an ancestor
}
```

```html
@* Shop.razor (parent) *@
<CascadingValue Value="session">
    @foreach (var p in products)
    {
        <ProductCard Product="p" OnAdd="AddToCart">
            <p>@p.Price.ToString("C")</p>
        </ProductCard>
    }
</CascadingValue>

@code {
    private readonly UserSession session = new();
    private readonly List<Product> products = new();
    private readonly List<Product> cart = new();
    private void AddToCart(Product p) => cart.Add(p);   // the parent re-renders after the EventCallback
}
```

- Parameters flow down like React props. Do not modify a `[Parameter]` inside the child: the parent's next render overwrites it. Copy it to a private field, or raise an `EventCallback` so the parent changes it.
- `EventCallback` (rather than a plain `Action`) supports async handlers and re-renders the component that owns the handler. `RenderFragment` is React's `children` (`RenderFragment<T>` builds templated components), and cascading values are React context. `@key` keeps list items matched to the right elements; `@ref` captures a reference.

## 5. Lifecycle and StateHasChanged

| Method | When | Typical use |
|---|---|---|
| `OnInitialized` / `OnInitializedAsync` | once, when the component is first created | load data |
| `OnParametersSet` / `OnParametersSetAsync` | after initialization and whenever the parent passes new parameters | react to parameter changes |
| `OnAfterRender` / `OnAfterRenderAsync(bool firstRender)` | after each render, once the DOM is updated | JS interop, focusing elements |
| `ShouldRender` | before a re-render | skip unnecessary renders |
| `Dispose` / `DisposeAsync` | when the component is removed | unsubscribe from events, stop timers |

```html
@* CartBadge.razor: reacts to a shared CartService *@
@implements IDisposable
@inject CartService Cart
<span>Items: @Cart.Count</span>

@code {
    protected override void OnInitialized() => Cart.Changed += OnCartChanged;
    private void OnCartChanged() => InvokeAsync(StateHasChanged);   // the event may fire on another thread
    public void Dispose() => Cart.Changed -= OnCartChanged;         // unsubscribe, or the component leaks
}
```

Blazor re-renders automatically after UI event handlers, after lifecycle methods and when the parent passes new parameters. Changes that come from elsewhere (timers, service events, background work) need `StateHasChanged()`, wrapped in `InvokeAsync` when not on the renderer's context. During `OnInitializedAsync`, Blazor renders once when the first `await` yields (so show a loading state) and again when it completes.

## 6. Data binding and forms

```html
<input @bind="search" @bind:event="oninput" placeholder="Search" />   @* updates on every keystroke *@

<EditForm Model="form" OnValidSubmit="SaveAsync">
    <DataAnnotationsValidator />
    <ValidationSummary />
    <InputText @bind-Value="form.Email" />
    <ValidationMessage For="() => form.Email" />
    <button type="submit">Sign up</button>
</EditForm>
@code {
    private string search = "";
    private SignupForm form = new();
    private Task SaveAsync() => Users.CreateAsync(form);   // Users: an injected service

    public class SignupForm
    {
        [Required, EmailAddress] public string Email { get; set; } = "";
    }
}
```

`@bind` sets the value and listens to `onchange` by default; `@bind:event`, `@bind:format` and `@bind:after` (run code after the value updates) tune it, and input components use `@bind-Value`. `EditForm` creates an `EditContext`, `DataAnnotationsValidator` applies the same attributes as MVC, and `OnValidSubmit`/`OnInvalidSubmit` split the outcomes. With WebAssembly, validation runs in the browser, so the API must validate again.

## 7. DI, JS interop and state management

```html
@inject IJSRuntime JS
<input @ref="searchBox" />

@code {
    private ElementReference searchBox;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (!firstRender) return;
        await JS.InvokeVoidAsync("app.focus", searchBox);                       // calls window.app.focus(element)
        var saved = await JS.InvokeAsync<string?>("localStorage.getItem", "cart");
    }
}
```

- **DI:** `@inject` (or an `[Inject]` property) resolves services. On Blazor Server a scoped service lives as long as the circuit, often the whole session, so a scoped `DbContext` becomes long-lived and is hit by overlapping events: use `IDbContextFactory<T>` and create a short-lived context per operation. On WebAssembly, scoped behaves like singleton (one per browser tab).
- **JS interop:** `InvokeAsync<T>` and `InvokeVoidAsync` call JavaScript functions by name; JavaScript can call .NET through `DotNetObjectReference`. Interop is not available during prerendering, so call it from `OnAfterRenderAsync`; load JS modules with `JS.InvokeAsync<IJSObjectReference>("import", "./js/app.js")`.
- **State:** component fields for local UI state; parameters and callbacks between parent and child; cascading values for app-wide context such as authentication state; a scoped "state container" service with a `Changed` event (like `CartService` above) for shared state; `[SupplyParameterFromQuery]` for URL state; browser storage (`ProtectedLocalStorage` on Server) for anything that must survive a lost circuit or a reload.

## 8. Blazor vs React

| | Blazor | React |
|---|---|---|
| Language | C# and Razor | JavaScript or TypeScript with JSX |
| Inputs and outputs | `[Parameter]` properties, `EventCallback<T>` | props, callback props |
| Nested markup | `RenderFragment ChildContent` | `children` |
| Shared context | `CascadingValue` / `[CascadingParameter]` | `createContext` / `useContext` |
| Local state | private fields, re-render after events | `useState` |
| Effects and cleanup | `OnInitializedAsync`, `OnAfterRenderAsync`, `Dispose` | `useEffect` with a cleanup function |
| Two-way binding | `@bind` | controlled inputs (`value` plus `onChange`) |
| Where it runs | server (SignalR) or browser (WebAssembly) | browser, with SSR and Server Components through frameworks |
| Ecosystem | smaller; component suites such as MudBlazor and Radzen | very large; React Native for mobile |
| Best fit | .NET teams, internal tools, sharing C# code end to end | public-facing apps, rich ecosystem, JavaScript teams |

## Common pitfalls
- In a .NET 8+ Blazor Web App, forgetting the render mode: the page renders as static SSR and buttons silently do nothing.
- Calling JS interop in `OnInitializedAsync` or during prerendering instead of `OnAfterRenderAsync(firstRender)`.
- Duplicate API calls because `OnInitializedAsync` runs during prerendering and again when interactive.
- Mutating `[Parameter]` properties in a child; forgetting `InvokeAsync(StateHasChanged)` for external events, or forgetting to unsubscribe.
- A scoped `DbContext` living for the whole Blazor Server circuit; secrets or trusted logic shipped in WebAssembly code.

## How to talk about this in an interview
- Lead with your React experience and translate: props are parameters, callbacks are `EventCallback`, context is cascading values, `useEffect` cleanup is `Dispose`. That shows you understand component models, not just one framework's syntax.
- Be honest: Blazor comes from university work and your final-year project. For a Blazor-heavy role you would study render modes in the current .NET version, circuit scaling and component libraries.
- > **Fill in:** whether your final-year project used Blazor (for example, a dashboard showing recognized activities), which hosting model or render mode, and which .NET version.

## Interview questions

**Q1. What is the difference between Blazor Server and Blazor WebAssembly?**
Blazor Server runs components on the server and syncs the UI over a SignalR circuit: tiny download, direct access to server resources and secrets, but every interaction pays network latency, each user consumes server memory, and losing the connection breaks the UI. Blazor WebAssembly runs the .NET runtime in the browser: no per-user server state, offline support and static hosting, but a larger initial download, client-visible code and all data access through APIs.
*Follow-up:* Which would you pick for an internal admin tool? Usually Server: fast to build, users sit on a reliable network, and data access is direct.

**Q2. What are the render modes in a .NET 8 Blazor Web App?**
Static SSR (server renders HTML with no interactivity; the default), Interactive Server (SignalR circuit), Interactive WebAssembly (runs in the browser) and Interactive Auto (starts on the server, switches to WebAssembly once the runtime is cached). The mode is chosen per component or page, or globally on the router, so one app can mix static content pages with interactive widgets.

**Q3. How do parent and child components communicate?**
The parent passes data down through `[Parameter]` properties; the child reports back by invoking an `EventCallback<T>` parameter, which also re-renders the parent. Nested markup is passed as a `RenderFragment` (`ChildContent`), and values needed deep in the tree go through `CascadingValue` and `[CascadingParameter]`. Shared app state can live in a scoped service with a change event.

**Q4. Walk through the component lifecycle. Where do you load data, and where do you call JavaScript?**
`SetParametersAsync`, then `OnInitialized(Async)` once, then `OnParametersSet(Async)` (again whenever parameters change), then render, then `OnAfterRender(Async)` with `firstRender` true the first time, and finally `Dispose` when removed. Load data in `OnInitializedAsync` (or `OnParametersSetAsync` if it depends on parameters); call JavaScript in `OnAfterRenderAsync` when `firstRender` is true, because the DOM exists only after rendering and interop is unavailable during prerendering.

**Q5. When do you need `StateHasChanged`?**
Only when state changes outside Blazor's automatic triggers (UI event handlers, lifecycle methods, new parameters): timer callbacks, events from a shared service, or results arriving from background work. Call it through `InvokeAsync(StateHasChanged)` when you might be on another thread, so the render runs on the renderer's synchronization context.

**Q6. How do forms and validation work in Blazor?**
`EditForm` binds to a model and creates an `EditContext`; input components (`InputText`, `InputNumber`, `InputSelect`) use `@bind-Value`; `DataAnnotationsValidator` applies the model's attributes; `ValidationSummary` and `ValidationMessage` display errors; `OnValidSubmit` runs only when the model is valid. In static SSR the form also needs a `FormName` and a `[SupplyParameterFromForm]` model, and APIs behind WebAssembly apps must validate again.

**Q7. How do DI lifetimes behave differently in Blazor?**
On Blazor Server, a scope is the circuit, so scoped services live for the user's whole connection, not for a single request; long-lived scoped `DbContext` instances become a problem, which is why `IDbContextFactory<T>` is recommended. On WebAssembly everything runs in one browser tab, so scoped services behave like singletons. Singletons on Server are shared by all users and must be thread-safe.

**Q8. Blazor vs React: when would you choose each?**
Blazor fits teams invested in .NET, internal and line-of-business apps, and cases where sharing C# models and validation end to end saves real effort. React fits public-facing products that need a huge ecosystem, mature tooling, SEO-friendly frameworks and code sharing with React Native mobile apps. Both use components, one-way data flow and render-tree diffing, so the concepts transfer.

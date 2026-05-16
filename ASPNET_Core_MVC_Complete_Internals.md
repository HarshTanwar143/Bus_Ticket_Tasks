# ASP.NET Core MVC — Complete Internals Handbook
## Part 1: Architecture, Pipeline & Framework Internals

---

# MODULE 1: THE NATURE OF ASP.NET CORE MVC — WHY IT EXISTS AND HOW IT DIFFERS

## 1.1 What MVC Actually Is (Not What Most Developers Think)

Most developers treat ASP.NET Core MVC as "controllers + views + models." This is catastrophically shallow. MVC is a **composition of multiple sub-frameworks** layered on top of ASP.NET Core's foundational pipeline:

```
ASP.NET Core Host
  └── Kestrel (HTTP transport)
      └── IServer (abstraction)
          └── HttpContext factory
              └── Middleware Pipeline (IApplicationBuilder)
                  └── EndpointMiddleware
                      └── IEndpointRouteBuilder / RoutingMiddleware
                          └── MVC Infrastructure
                              ├── IActionDescriptorCollectionProvider
                              ├── IControllerFactory
                              ├── IModelBinderFactory
                              ├── IFilterFactory
                              ├── IActionInvokerFactory
                              └── IRazorViewEngine (MVC only, not Web API)
```

MVC is not a replacement for ASP.NET Core — it is a **hosted guest** within the ASP.NET Core middleware pipeline. It registers itself as an endpoint handler.

## 1.2 MVC vs Web API: The Real Internal Difference

This is one of the most misunderstood topics.

**Common misconception:** "Web API is for JSON, MVC is for views."
**Reality:** Web API and MVC share ~95% of the same infrastructure. The differences are:

| Concern | ASP.NET Core MVC | ASP.NET Core Web API |
|---|---|---|
| Base class | `Controller` (inherits `ControllerBase`) | `ControllerBase` |
| View support | Yes (`IViewEngine`, Razor) | No |
| `[ApiController]` attribute | Optional | Standard |
| Content negotiation | Manual | Automatic via `IOutputFormatter` |
| Default result on `return View()` | `ViewResult` → Razor render | N/A |
| Model binding source | Form, Route, QueryString, Body | Route, QueryString, Body (with `[ApiController]` Body is inferred) |
| Validation response | Redisplays view with errors | Auto 400 `ValidationProblemDetails` |
| Anti-forgery | Built in via Tag Helpers | Not applied by default |

**Internally:** Both use the same `ActionDescriptor`, same `IActionInvokerFactory`, same `IFilterProvider`, same model binding pipeline. The divergence is in **result execution** — MVC has `ViewResultExecutor`, Web API uses `ObjectResultExecutor`.

## 1.3 The Three Layers of MVC Infrastructure

```
Layer 1: ASP.NET Core Foundation
  - HttpContext
  - IApplicationBuilder
  - IServiceCollection / IServiceProvider
  - IHostedService
  - Kestrel

Layer 2: Routing + Endpoint Infrastructure
  - RouteValueDictionary
  - IEndpointRouteBuilder
  - EndpointDataSource
  - MatcherPolicy
  - IRoutingFeature

Layer 3: MVC Application Infrastructure
  - ApplicationModel (startup-time metadata)
  - ActionDescriptor (compiled action metadata)
  - IControllerFactory
  - IActionInvoker
  - FilterPipeline
  - ModelBinding
  - ResultExecution
  - ViewEngine (MVC only)
```

---

# MODULE 2: STARTUP AND BOOTSTRAPPING INTERNALS

## 2.1 What `AddControllers()` / `AddControllersWithViews()` Actually Does

When you call `builder.Services.AddControllersWithViews()`, you are triggering a complex registration cascade. Let's trace it:

```csharp
// Your Program.cs
builder.Services.AddControllersWithViews();
```

Internally, this calls through a chain:

```
AddControllersWithViews()
  → AddControllersWithViews(IMvcBuilder)
      → AddControllers()              // registers controller infrastructure
          → AddMvcCore()              // registers core MVC services
              ├── ApplicationPartManager
              ├── IActionDescriptorCollectionProvider
              ├── IActionDescriptorChangeProvider
              ├── ActionConstraintCache
              ├── IControllerFactory
              ├── IControllerActivator (DefaultControllerActivator → uses DI)
              ├── IModelBinderFactory
              ├── IModelMetadataProvider (DefaultModelMetadataProvider)
              ├── IObjectModelValidator
              ├── IActionInvokerFactory
              ├── IFilterProvider (DefaultFilterProvider)
              ├── IActionResultExecutor<T> for each result type
              └── MvcRouteHandler
      → AddApiExplorer()
      → AddAuthorization()
      → AddCors()
      → AddDataAnnotations()
      → AddFormatterMappings()
  → AddViews()                       // registers Razor/View infrastructure
      → AddDataAnnotations()
      → IRazorViewEngine
      → IViewComponentActivator
      → ITempDataProvider
      → IHtmlHelper
      └── ITagHelperActivator
```

**Key insight:** `AddMvcCore()` is the true foundation. Everything else is additive. This is why you can call `AddMvcCore()` alone for minimal MVC setups.

## 2.2 ApplicationPartManager — How MVC Discovers Your Controllers

This is rarely understood but critically important.

```csharp
// Internally registered as singleton
services.AddSingleton<ApplicationPartManager>()
```

`ApplicationPartManager` manages:
1. **ApplicationParts** — assemblies/assemblies-as-parts that are scanned for controllers, views, tag helpers
2. **FeatureProviders** — `IApplicationFeatureProvider<T>` that extract features from parts

The flow at startup:

```
WebApplication.Build()
  → IHostedService.StartAsync()
      → ApplicationPartManager scans
          → ControllerFeatureProvider : IApplicationFeatureProvider<ControllerFeature>
              → Foreach assembly in ApplicationParts:
                  → Foreach type in assembly:
                      → IsController() check:
                          ✓ Is public
                          ✓ Is not abstract
                          ✓ Is not generic (or closed generic)
                          ✓ Is not in excluded assemblies
                          ✓ Inherits ControllerBase OR has [Controller] attribute
                          ✓ Does NOT have [NonController] attribute
```

**IsController() logic (simplified from source):**

```csharp
internal static bool IsController(TypeInfo typeInfo)
{
    if (!typeInfo.IsClass) return false;
    if (typeInfo.IsAbstract) return false;
    if (!typeInfo.IsPublic) return false;
    if (typeInfo.ContainsGenericParameters) return false;
    if (typeInfo.IsDefined(typeof(NonControllerAttribute))) return false;
    
    // Must either end in "Controller" suffix OR have [Controller] attribute
    if (!typeInfo.Name.EndsWith("Controller", StringComparison.OrdinalIgnoreCase) 
        && !typeInfo.IsDefined(typeof(ControllerAttribute), inherit: true))
        return false;
    
    return true;
}
```

**Viva Question:** "Can I have a controller class that doesn't end in 'Controller'?"
**Answer:** Yes, if you apply `[Controller]` attribute. The framework checks both naming convention AND attribute. This is the "convention vs. explicit configuration" duality throughout MVC.

## 2.3 ActionDescriptor Construction — The Compiled Action Model

After controller discovery, MVC builds `ActionDescriptor` objects. These are the runtime representation of your actions.

```
ControllerFeature (list of TypeInfo)
  → ApplicationModelFactory.CreateApplicationModel()
      → Builds ApplicationModel (startup-time)
          → ControllerModel (per controller)
              → ActionModel (per action method)
                  → ParameterModel (per parameter)
                      → SelectorModel (per route selector/attribute)
  → ControllerActionDescriptorBuilder.Build()
      → Converts ApplicationModel → ControllerActionDescriptor[]
          (one per action, including multiple selectors = multiple descriptors)
```

**What's stored in ControllerActionDescriptorriptor:**

```csharp
public class ControllerActionDescriptor : ActionDescriptor
{
    public TypeInfo ControllerTypeInfo { get; }      // Reflection TypeInfo
    public MethodInfo MethodInfo { get; }             // Reflection MethodInfo  
    public string ControllerName { get; }
    public string ActionName { get; }
    public IList<FilterDescriptor> FilterDescriptors { get; } // ALL filters merged
    public IDictionary<string, string> RouteValues { get; }
    public AttributeRouteInfo? AttributeRouteInfo { get; }
    // Plus: BoundProperties, Parameters with binding metadata
}
```

**This is built ONCE at startup and cached.** It's a frozen snapshot of reflection metadata. This is why:
- Adding a controller at runtime doesn't work without clearing this cache
- Filters are merged at startup, not per-request
- Route metadata is baked in

## 2.4 `UseRouting()` and `UseEndpoints()` — What They Register

```csharp
// Modern .NET 6+ (these are called implicitly by MapControllers)
app.UseRouting();       // Registers RoutingMiddleware
app.UseEndpoints(...)  // Registers EndpointMiddleware
```

**RoutingMiddleware** (`Microsoft.AspNetCore.Routing.EndpointRoutingMiddleware`):
- Runs `IEndpointSelector` → matches an endpoint
- Stores matched `Endpoint` in `HttpContext.Features.Get<IEndpointFeature>()`
- Does NOT execute the endpoint

**EndpointMiddleware** (`Microsoft.AspNetCore.Routing.EndpointMiddleware`):
- Reads `IEndpointFeature` from HttpContext
- Calls `endpoint.RequestDelegate(httpContext)` → this is the MVC handler

**The bridge between Routing and MVC:**

When `MapControllers()` is called:

```
MapControllers()
  → Registers all ControllerActionDescriptors as Endpoints
  → Each Endpoint has:
      RoutePattern (from attribute routes or conventional routes)
      Metadata (ActionDescriptor, HttpMethodMetadata, AuthorizeData, etc.)
      RequestDelegate = MVC's ActionInvoker invocation
```

---

# MODULE 3: THE COMPLETE REQUEST LIFECYCLE — KESTREL TO HTML RESPONSE

## 3.1 Full Stack Trace (Conceptual)

```
[1] TCP connection arrives at Kestrel
    ↓
[2] Kestrel reads bytes → parses HTTP/1.1 or HTTP/2
    ↓
[3] Kestrel creates HttpContext (DefaultHttpContext)
    ↓
[4] Kestrel hands HttpContext to IHttpApplication<TContext>.ProcessRequestAsync()
    ↓
[5] HostingApplication.ProcessRequestAsync()
    - Creates IServiceScope (per-request DI scope) ← CRITICAL
    - Sets HttpContext.RequestServices = scope.ServiceProvider
    ↓
[6] IApplicationBuilder pipeline begins
    (Each middleware calls _next(context) to proceed)
    ↓
[7] ExceptionHandlingMiddleware (if configured)
    ↓
[8] HSTSMiddleware / HTTPSRedirectionMiddleware
    ↓
[9] StaticFilesMiddleware (short-circuits here for static files)
    ↓
[10] RoutingMiddleware (EndpointRoutingMiddleware)
     - Calls IEndpointSelector.SelectAsync()
     - Runs route matching against all registered endpoints
     - Sets HttpContext.Features → IEndpointFeature
    ↓
[11] CorsMiddleware (if enabled, may short-circuit for preflight)
    ↓
[12] AuthenticationMiddleware
     - Calls IAuthenticationService.AuthenticateAsync()
     - Sets HttpContext.User (ClaimsPrincipal)
    ↓
[13] AuthorizationMiddleware
     - Reads endpoint metadata for [Authorize] attributes
     - May short-circuit with 401/403
    ↓
[14] EndpointMiddleware
     - Reads IEndpointFeature
     - Invokes endpoint.RequestDelegate
    ↓
[15] MVC ActionInvoker Invocation
     ↓
[16] IControllerFactory.CreateController()
     - Uses IControllerActivator (DI-based)
     - Creates controller instance with all dependencies resolved
    ↓
[17] Model Binding
     - IModelBinderFactory creates binders per parameter
     - Reads from route values, query string, form, body
    ↓
[18] Filter Pipeline (Action Filters, Authorization Filters, Resource Filters)
    ↓
[19] Action Method Execution
    ↓
[20] Result Filters
    ↓
[21] IActionResult.ExecuteResultAsync()
     For ViewResult:
       → IViewEngine.FindView()
       → Razor view compilation (or cache hit)
       → IView.RenderAsync()
       → Write HTML to response stream
    ↓
[22] Response flushed through Kestrel
    ↓
[23] IServiceScope.Dispose() — disposes scoped services
    ↓
[24] IControllerFactory.ReleaseController() — disposes controller if IDisposable
```

## 3.2 HttpContext Lifetime and Memory

`DefaultHttpContext` is pooled. ASP.NET Core maintains an `ObjectPool<DefaultHttpContext>` to avoid allocations on every request.

```
Request arrives → Get HttpContext from pool (or new if pool empty)
  → Reset/Initialize HttpContext for this request
  → Process request
  → Return HttpContext to pool
```

**What gets reset between requests:**
- `Request` / `Response` objects are recycled but their streams reset
- `Features` collection is cleared
- `User` is reset to null/empty
- `Items` dictionary is cleared

**Memory implication:** A `HttpContext` object does NOT get garbage collected between requests — it's reused. This means:
- Never store `HttpContext` in a field of a Singleton service (stale after request ends)
- `IHttpContextAccessor` uses `AsyncLocal<HttpContext>` — thread-safe but with async gotchas

## 3.3 The Per-Request DI Scope — Most Critical Concept

```
Request arrives
  ↓
HostingApplication creates IServiceScope
  ServiceScope.ServiceProvider = child of root IServiceProvider
  ↓
HttpContext.RequestServices = scope.ServiceProvider
  ↓
All MVC DI resolutions use HttpContext.RequestServices
  ↓
Request ends
  ↓
scope.Dispose() is called
  ↓
All Scoped services registered in that scope are disposed
  ↓
Any IDisposable scoped services have Dispose() called
```

**This means:**
- `DbContext` (scoped by default in EF Core) → one instance per request, disposed after
- Services resolved via `IServiceProvider` from `HttpContext.RequestServices` are scoped to the request
- Controller itself is created and released per request

**Production trap:** If you inject an `IServiceProvider` into a Singleton service and call `GetRequiredService<MyScopedService>()` — you get a `ScopedService` from the ROOT container, which never gets disposed. This is the "captive dependency" problem.

---

# MODULE 4: ENDPOINT ROUTING INTERNALS (DEEP DIVE)

## 4.1 Why Endpoint Routing Was Introduced (.NET Core 3.0+)

Before endpoint routing, routing was tightly coupled to MVC. Middleware could not know WHICH action would handle a request. This meant:
- CORS middleware had to run before routing
- Authorization middleware couldn't know the target action's `[Authorize]` attributes
- This created ordering nightmares

Endpoint Routing decoupled routing decision from execution:
1. **Route matching** happens early (RoutingMiddleware)
2. **Middleware runs** with knowledge of the matched endpoint
3. **Execution** happens last (EndpointMiddleware)

## 4.2 Route Matching Internals

The route matching system uses a **Decision Tree** (Dispatcher + SequentialMatcher internally).

**At startup:**

```
All routes registered → DfaMatcher built (Deterministic Finite Automaton)
  → Each route segment becomes a state
  → Literal segments get exact match states
  → Parameter segments get parameter matcher states
  → Constraint checking is layered after DFA match
```

**At request time:**

```
URL: /products/42/edit
  ↓
DfaMatcher.FindCandidatesAsync()
  → Walk URL segments against DFA
  → Find candidate endpoints
  ↓
EndpointSelector.SelectAsync()
  → Disambiguate candidates (MatcherPolicy ordering)
  → Run route constraints (IRouteConstraint)
  → Select best match
  ↓
Set HttpContext.Features[IEndpointFeature] = selectedEndpoint
```

**Route constraints are checked AFTER DFA matching.** The DFA does structural matching (segment count, literal vs parameter). Constraints (`{id:int}`, `{name:minlength(3)}`) filter candidates afterward.

## 4.3 Conventional Routing vs Attribute Routing Internally

**Conventional routing:**
```csharp
app.MapControllerRoute("default", "{controller=Home}/{action=Index}/{id?}");
```

This registers a `ConventionalRouteEntry`. At startup, for each `ControllerActionDescriptor` that does NOT have attribute routing, a route is built using this template.

**Attribute routing:**
```csharp
[Route("api/products/{id}")]
```

This bakes the route into the `AttributeRouteInfo` on the `ActionDescriptor`. The endpoint's `RoutePattern` is built directly from this.

**Key difference:** With attribute routing, the URL fully determines the action. With conventional routing, the URL is parsed for `controller`/`action` values, then those values are matched to an `ActionDescriptor`.

## 4.4 Route Value Dictionary — The Hidden State

`RouteValueDictionary` is populated during route matching and flows through:

```
RoutingMiddleware sets IRoutingFeature.RouteData
  → RouteData.Values = {controller: "Products", action: "Edit", id: "42"}
  ↓
Model Binding reads from RouteData.Values
  ↓
UrlHelper reads from RouteData.Values when generating URLs
  ↓
ActionDescriptor matching uses RouteData.Values
```

This dictionary is the "nervous system" of the request — many components read from it.

---

# MODULE 5: CONTROLLER FACTORY AND ACTIVATION INTERNALS

## 5.1 How a Controller Is Created — The Full Chain

```csharp
// Registered services:
services.AddSingleton<IControllerFactory, DefaultControllerFactory>()
services.AddTransient<IControllerActivator, DefaultControllerActivator>()
```

**DefaultControllerFactory.CreateController():**

```
ControllerContext (built from ActionContext + HttpContext)
  ↓
DefaultControllerActivator.Create(controllerContext)
  ↓
ObjectFactory = ActivatorUtilities.CreateFactory(controllerType, Type[0])
  (This is cached per controller type — only built once)
  ↓
ObjectFactory(requestServices, arguments)
  → Resolves constructor parameters from IServiceProvider (requestServices)
  → Creates controller instance via compiled expression (not reflection per-request)
```

**The ObjectFactory is key.** `ActivatorUtilities.CreateFactory()` compiles an expression tree at startup that can create the controller without using Reflection.Invoke per request. This is a significant performance optimization.

**Memory:** The controller instance lives on the heap for the duration of action execution. After `IControllerFactory.ReleaseController()`, if the controller implements `IDisposable`, `Dispose()` is called.

## 5.2 Controller Constructor Parameter Resolution

When the controller is created:

```
Controller constructor: public ProductsController(IProductService svc, ILogger<ProductsController> logger)
  ↓
IServiceProvider (request-scoped) tries to resolve IProductService
  → If registered as Scoped: returns same instance as any other scoped resolve in this request
  → If registered as Transient: creates new instance
  → If registered as Singleton: returns shared instance from root container
  ↓
IServiceProvider tries to resolve ILogger<ProductsController>
  → ILogger<T> is Singleton (logger factory is singleton)
  ↓
Both parameters resolved → ActivatorUtilities creates controller
```

**Common misconception:** "The controller is singleton because the service is singleton."
**Reality:** The controller is ALWAYS transient (new per request). Its DEPENDENCIES might be singleton. The controller holds a reference to a singleton service, but the controller itself is new.

## 5.3 ControllerContext vs ActionContext vs HttpContext

Hierarchy:
```
HttpContext (the raw HTTP plumbing)
  ↓
ActionContext : contains HttpContext + RouteData + ActionDescriptor
  ↓
ControllerContext : ActionContext + metadata specific to controller activation
```

`ControllerContext` is created by `IControllerContextFactory` and stored as a property on the controller base class.

**`Controller.HttpContext`** — this is NOT a direct property. It's:
```csharp
// In ControllerBase:
public HttpContext HttpContext => ControllerContext.HttpContext;
```

The controller doesn't "own" HttpContext — it accesses it through ControllerContext.

---

# MODULE 6: MODEL BINDING INTERNALS

## 6.1 Why Model Binding Is More Complex Than Most Developers Realize

Model binding is the process of mapping HTTP request data (route values, query string, form data, JSON body, headers, cookies) to action method parameters and properties.

The system is built on:
- `IModelBinderFactory` — creates binders
- `IModelBinder` — performs actual binding
- `IModelMetadataProvider` — provides metadata about types
- `IValueProviderFactory` — creates value providers (sources of data)
- `IObjectModelValidator` — validates after binding

## 6.2 Value Providers — The Data Sources

Before binding happens, value providers are assembled:

```
IValueProviderFactory[] (registered in order):
  1. FormValueProviderFactory     → HttpContext.Request.Form
  2. RouteValueProviderFactory    → RouteData.Values
  3. QueryStringValueProviderFactory → HttpContext.Request.Query
```

Each `IValueProviderFactory.CreateValueProviderAsync()` produces an `IValueProvider` that knows how to retrieve values by name.

**Note:** Request body (JSON) is NOT a value provider — it uses `IInputFormatter` directly and is handled by `BodyModelBinder` as a special case.

## 6.3 ModelBinderFactory — How Binders Are Selected

```
For each action parameter:
  ↓
ModelBinderFactory.CreateBinder(ModelBinderFactoryContext)
  ↓
  1. Check explicit [Bind...] attribute on parameter
     → [FromQuery]     → QueryStringValueProvider
     → [FromRoute]     → RouteValueProvider  
     → [FromForm]      → FormValueProvider
     → [FromBody]      → BodyModelBinder (InputFormatter chain)
     → [FromHeader]    → HeaderValueProvider
     → [FromServices]  → ServiceEntryModelBinder (DI resolution)
  
  2. If no attribute: use default inference
     For Web API with [ApiController]:
       - Simple types (string, int, etc.) → [FromQuery] inferred
       - Complex types → [FromBody] inferred
     For plain MVC:
       - Route match first, then QueryString, then Form
```

**The `[ApiController]` inference is done at startup** in `ApiBehaviorApplicationModelProvider`. It modifies the `ParameterModel` objects in the `ApplicationModel`, adding `BindingInfo` metadata.

## 6.4 ComplexObjectModelBinder — Recursive Binding

For complex types (e.g., `CustomerViewModel`), `ComplexObjectModelBinder` does recursive binding:

```
CustomerViewModel
  ├── string Name       → StringModelBinder
  ├── int Age           → SimpleTypeModelBinder
  ├── AddressViewModel  → ComplexObjectModelBinder (recursive)
  │    ├── string Street → StringModelBinder
  │    └── string City   → StringModelBinder
  └── List<string> Tags → CollectionModelBinder → StringModelBinder per item
```

The binder prefix matters: `"Customer.Address.Street"` is how form data must be named to bind to nested types. This is the **naming convention** for complex model binding.

## 6.5 Model Binding Execution Flow (Per Request)

```
ActionExecutor begins
  ↓
ParameterBinder.BindModelAsync() called for each parameter
  ↓
  1. Create ModelBindingContext
     - Set ModelMetadata (type info, display names, validators)
     - Set ValueProvider (composite of all value providers)
     - Set ModelName (parameter name or [Bind] prefix)
  ↓
  2. IModelBinder.BindModelAsync(bindingContext)
     ↓
     - Attempts to read value from IValueProvider
     - Converts string value to target type
     - Sets bindingContext.Result = ModelBindingResult.Success(value)
  ↓
  3. IObjectModelValidator.Validate(model)
     - Runs DataAnnotation validators
     - Populates ModelState dictionary
  ↓
  4. If ModelState.IsValid == false:
     - For MVC: action still executes (you check manually or use filter)
     - For [ApiController]: automatic 400 response (via ModelStateInvalidFilter)
```

## 6.6 ModelState — What It Is Internally

`ModelStateDictionary` is a dictionary of `{fieldName → ModelStateEntry}`.

```csharp
public class ModelStateEntry
{
    public ValueProviderResult RawValue { get; }    // Original string from request
    public object AttemptedValue { get; }           // Attempted bind
    public ModelErrorCollection Errors { get; }      // Validation errors
    public ModelValidationState ValidationState { get; } // Valid/Invalid/Unvalidated/Skipped
}
```

It's stored on `ActionContext.ModelState` and accessible via `Controller.ModelState`.

**Viva trap:** "When is ModelState populated?"
- During model binding (from value providers)
- During `IObjectModelValidator.Validate()` (DataAnnotations)
- You can manually add errors: `ModelState.AddModelError("field", "message")`
- Remote validation adds errors via Ajax call

---

# MODULE 7: THE FILTER PIPELINE INTERNALS

## 7.1 Filters vs Middleware — The Deep Distinction

This is a critical interview topic. Filters and middleware both intercept requests, but they operate at fundamentally different layers.

```
Middleware operates at:
  - HTTP level (HttpContext)
  - Before MVC is even involved
  - No access to ActionDescriptor, ModelState, Controller, ActionResult
  - Applied globally to ALL requests (static files, WebSockets, etc.)

Filters operate at:
  - MVC level (ActionContext, ActionExecutingContext, etc.)
  - Has access to: Controller, Action metadata, Model state, Parameters
  - Applied only to MVC requests
  - Can be scoped to controller or action (not possible with middleware)
```

**When to use each:**
- Authentication, CORS, rate limiting → Middleware (needed before routing)
- Action auditing, transaction wrapping, model validation shortcuts → Filters

## 7.2 Filter Types and Execution Order

```
Filter Execution Order (for Action Execution):

[1] Authorization Filters (IAuthorizationFilter)
    → Can short-circuit (set context.Result)
    ↓
[2] Resource Filters - Before (IResourceFilter.OnResourceExecuting)
    → Can short-circuit, can serve from cache
    ↓
[3] Model Binding occurs
    ↓
[4] Action Filters - Before (IActionFilter.OnActionExecuting)
    → Can short-circuit, can modify parameters
    ↓
[5] ACTION METHOD EXECUTES
    ↓
[6] Action Filters - After (IActionFilter.OnActionExecuted)
    → Can replace result
    ↓
[7] Exception Filters (IExceptionFilter)
    → Catches exceptions from [2]-[6]
    ↓
[8] Result Filters - Before (IResultFilter.OnResultExecuting)
    ↓
[9] RESULT EXECUTES (IActionResult.ExecuteResultAsync)
    ↓
[10] Result Filters - After (IResultFilter.OnResultExecuted)
    ↓
[11] Resource Filters - After (IResourceFilter.OnResourceExecuted)
```

**Authorization filters run BEFORE model binding.** This is intentional — why bind models for unauthorized requests?

## 7.3 How Filters Are Discovered and Merged

**At startup:**

```
For each ActionDescriptor:
  ↓
  1. Global filters (from MvcOptions.Filters)
  2. Controller-level filter attributes
  3. Action-level filter attributes
  → These are merged into ActionDescriptor.FilterDescriptors
     Each FilterDescriptor has Order + Scope (Global=0, Controller=10, Action=20)
```

**At request time:**

```
IFilterProvider[] (ordered):
  1. DefaultFilterProvider — reads FilterDescriptors from ActionDescriptor
  2. (Custom IFilterProvider if registered)
  ↓
FilterFactory.GetAllFilters(actionContext)
  → Creates filter instances
  → For IFilterFactory: calls IFilterFactory.CreateInstance(serviceProvider)
     (allows DI-injected filters via TypeFilterAttribute or ServiceFilterAttribute)
  ↓
FilterFactoryResult contains:
  - Static filters (cached between requests)
  - Dynamic filters (created per request)
```

## 7.4 `IFilterFactory` vs `IFilter` — When DI in Filters Is Needed

```csharp
// WRONG: Can't inject in constructor of Attribute-based filter
public class MyFilter : ActionFilterAttribute
{
    private readonly ILogger _logger; // attributes can't use constructor DI
}

// RIGHT approach 1: ServiceFilter
[ServiceFilter(typeof(MyActionFilter))]
public class HomeController { }
// MyActionFilter is resolved from DI container

// RIGHT approach 2: TypeFilter
[TypeFilter(typeof(MyActionFilter))]
public class HomeController { }
// MyActionFilter is created via ActivatorUtilities (mix of DI + explicit args)
```

`ServiceFilterAttribute` implements `IFilterFactory`:
```csharp
public IFilterMetadata CreateInstance(IServiceProvider serviceProvider)
{
    return (IFilterMetadata)serviceProvider.GetRequiredService(typeof(MyActionFilter));
}
```

This means the filter is resolved from the request-scoped `IServiceProvider` — **scoped services can be injected into filters via ServiceFilter**.

## 7.5 Exception Filters vs Middleware Exception Handling

```
Middleware ExceptionHandler (UseExceptionHandler):
  - Catches ALL unhandled exceptions in the pipeline
  - Global catch-all
  - Gets raw Exception + HttpContext
  - Executes OUTSIDE MVC

Exception Filter (IExceptionFilter):
  - Catches exceptions from MVC pipeline only (action + result execution)
  - Has access to ExceptionContext (includes ActionDescriptor, Controller, etc.)
  - Can suppress exception (context.ExceptionHandled = true)
  - Can provide an ActionResult as response
  - Does NOT catch exceptions from Authorization filters or Result filters
```

**Production pattern:** Use exception filters for MVC-specific error responses (e.g., 404 for entity not found, 422 for validation). Use middleware exception handler as the global safety net.

## 7.6 Async Filter Variants

Every filter has an async version:
- `IAsyncActionFilter` → `OnActionExecutionAsync(context, next)` 
- `IAsyncResultFilter` → `OnResultExecutionAsync(context, next)`

**The `next` delegate is critical.** In async filters, you decide when to call `next()`. Not calling `next()` short-circuits the pipeline.

```csharp
public async Task OnActionExecutionAsync(
    ActionExecutingContext context, 
    ActionExecutionDelegate next)
{
    // BEFORE action
    var sw = Stopwatch.StartNew();
    
    var resultContext = await next(); // Execute action (and inner filters)
    
    // AFTER action
    _logger.LogInformation("Action took {Ms}ms", sw.ElapsedMilliseconds);
    
    // resultContext.Exception is set if action threw
    // resultContext.Result is the IActionResult
}
```

---

# MODULE 8: ACTION EXECUTION INTERNALS

## 8.1 IActionInvokerFactory — The Bridge

After the filter pipeline setup, `IActionInvokerFactory.CreateInvoker()` creates the actual invoker:

```
IActionInvokerFactory
  → ControllerActionInvokerProvider.OnProvidersExecuting()
      → Creates ControllerActionInvokerCacheEntry (cached per action):
          - FilterFactoryResult
          - ObjectMethodExecutor (compiled delegate for action method)
          - ActionMethodExecutor (handles sync/async/value-task variants)
      → Returns ControllerActionInvoker
```

## 8.2 ObjectMethodExecutor — Zero-Reflection Action Execution

`ObjectMethodExecutor` is one of the most important internal classes. It:
1. At startup: compiles an expression tree for the action method
2. At runtime: executes the compiled delegate (NOT `MethodInfo.Invoke()`)

```csharp
// Conceptually what ObjectMethodExecutor compiles:
// Original: public IActionResult Edit(int id, ProductViewModel model)
// Compiled to something like:
Func<object, object[], object> executor = (controller, args) =>
    ((ProductsController)controller).Edit((int)args[0], (ProductViewModel)args[1]);
```

This avoids the performance penalty of reflection per request.

## 8.3 How the Framework Handles Sync vs Async Actions

`ActionMethodExecutor` is abstract with concrete implementations:

```
ActionMethodExecutors (checked in order):
  1. VoidActionMethodExecutor        — void return type
  2. SyncActionResultExecutor        — IActionResult (sync)
  3. AsyncActionResultExecutor       — Task<IActionResult>
  4. AsyncObjectResultExecutor       — Task<T> (Web API returns)
  5. VoidTaskResultExecutor          — Task (void async)
```

The framework selects the appropriate executor at startup based on the action method's return type. At runtime, the correct code path is taken without per-request type-checking.

## 8.4 What Happens When an Action Returns `null`

```csharp
public IActionResult GetProduct() => null; // what happens?
```

If an action returns null:
- `ActionMethodExecutor` wraps it in `EmptyResult`
- `EmptyResult.ExecuteResultAsync()` writes nothing (just sets 200 status code if not already set)

This is intentional behavior — null is treated as "empty response."

---
# ASP.NET Core MVC — Complete Internals Handbook
## Part 2: IActionResult, Razor Internals, View Rendering, State Management

---

# MODULE 9: IACTIONRESULT EXECUTION PIPELINE

## 9.1 What IActionResult Actually Is

`IActionResult` is just an interface:
```csharp
public interface IActionResult
{
    Task ExecuteResultAsync(ActionContext context);
}
```

The result knows how to write itself to the HTTP response. The separation between "deciding the result" (action execution) and "writing the result" (result execution) is fundamental MVC design. This allows:
- Filters to intercept and replace results
- Unit testing (verify the returned type without HTTP)
- Content negotiation

## 9.2 IActionResultExecutor — The Hidden Partner

Every result type has a companion executor registered in DI:

```csharp
// Registered at startup:
services.AddSingleton<IActionResultExecutor<ViewResult>, ViewResultExecutor>()
services.AddSingleton<IActionResultExecutor<JsonResult>, JsonResultExecutor>()
services.AddSingleton<IActionResultExecutor<RedirectResult>, RedirectResultExecutor>()
services.AddSingleton<IActionResultExecutor<FileResult>, FileResultExecutor>()
services.AddSingleton<IActionResultExecutor<ObjectResult>, ObjectResultExecutor>()
// ... etc
```

When `result.ExecuteResultAsync(context)` is called, most results delegate to their executor:

```csharp
// ViewResult.ExecuteResultAsync (simplified):
public async Task ExecuteResultAsync(ActionContext context)
{
    var executor = context.HttpContext.RequestServices
        .GetRequiredService<IActionResultExecutor<ViewResult>>();
    await executor.ExecuteAsync(context, this);
}
```

**Why this pattern?** The result holds DATA (view name, status code, model). The executor holds BEHAVIOR (how to write to response). This makes results lightweight and testable, while allowing executors to be replaced via DI.

## 9.3 ObjectResult — The Most Complex Result Type

`ObjectResult` is the workhorse of Web API. When you `return Ok(myObject)`, you get `OkObjectResult : ObjectResult`.

**Content negotiation flow:**

```
ObjectResult.ExecuteResultAsync()
  ↓
ObjectResultExecutor.ExecuteAsync()
  ↓
  1. Determine status code (200 for Ok, 201 for Created, etc.)
  ↓
  2. Content Negotiation:
     - Read Accept header from request
     - Compare against registered IOutputFormatters
     - Formatters (in order):
         SystemTextJsonOutputFormatter  (application/json)
         XmlDataContractSerializerOutputFormatter (application/xml, if registered)
         StringOutputFormatter          (text/plain)
     - Select best matching formatter
     - If no match and ReturnHttpNotAcceptable=true → 406
  ↓
  3. IOutputFormatter.WriteAsync(OutputFormatterWriteContext)
     - Serializes object to response body stream
     - Sets Content-Type header
```

## 9.4 ViewResult vs PartialViewResult vs ViewComponentResult

```
ViewResult:
  - Finds full layout (via _ViewStart.cshtml chain)
  - Renders master page + body
  - Most expensive

PartialViewResult:
  - No layout lookup
  - Just renders the partial view template
  - Used for AJAX partial updates

ViewComponentResult:
  - Invokes IViewComponent
  - ViewComponent is a mini-controller for reusable UI
  - Has its own Invoke/InvokeAsync method
  - Renders its own view (Views/Shared/Components/ComponentName/Default.cshtml)
```

---

# MODULE 10: RAZOR VIEW ENGINE INTERNALS

## 10.1 What Razor Actually Is

Razor is a **template language** that compiles `.cshtml` files into C# classes that, when executed, write HTML to a `TextWriter`.

The compilation pipeline:

```
.cshtml file (source)
  ↓
RazorProjectEngine.Process()
  ↓
RazorCodeDocument produced
  ↓
RazorCSharpDocument (C# source code)
  ↓
Roslyn compiler
  ↓
Assembly (compiled view as DLL)
  ↓
Cached in memory
```

**The compiled output for a simple view:**

```cshtml
<!-- Source .cshtml -->
@model ProductViewModel
<h1>@Model.Name</h1>
```

Compiles to roughly:
```csharp
public class Views_Products_Index : RazorPage<ProductViewModel>
{
    public override async Task ExecuteAsync()
    {
        WriteLiteral("<h1>");
        Write(Model.Name);  // HTML-encodes by default
        WriteLiteral("</h1>");
    }
}
```

`Write()` HTML-encodes. `WriteLiteral()` writes raw. This is how `@Model.Name` protects against XSS while `@Html.Raw()` bypasses it.

## 10.2 View Discovery — How Razor Finds Views

`IViewEngine.FindView()` or `IViewEngine.GetView()` performs view discovery:

```
FindView(controllerContext, viewName, isMainPage):
  ↓
RazorViewEngine checks ViewLocationFormats (in order):
  1. "/Views/{1}/{0}.cshtml"   → {1}=ControllerName, {0}=ViewName
  2. "/Views/Shared/{0}.cshtml"
  (For Areas: "/Areas/{2}/Views/{1}/{0}.cshtml", etc.)
  ↓
For each location:
  IRazorPageFactoryProvider.CreateFactory(path)
    → Checks if compiled page exists in memory cache
    → If not: attempts to compile from disk
    → If disk file not found: continue to next location
  ↓
If found: returns ViewEngineResult.Found(view)
If not found: returns ViewEngineResult.NotFound(searched locations)
```

**The searched locations are included in the 500 error when view not found** — this is the "view not found" exception message you see in development.

## 10.3 ViewLocationExpander — Customizing View Discovery

You can intercept view discovery by implementing `IViewLocationExpander`:

```csharp
services.Configure<RazorViewEngineOptions>(options =>
{
    options.ViewLocationExpanders.Add(new TenantViewLocationExpander());
});

public class TenantViewLocationExpander : IViewLocationExpander
{
    public IEnumerable<string> ExpandViewLocations(
        ViewLocationExpanderContext context, 
        IEnumerable<string> viewLocations)
    {
        // Prepend tenant-specific path
        var tenant = context.Values["tenant"];
        yield return $"/Tenants/{tenant}/Views/{{1}}/{{0}}.cshtml";
        foreach (var location in viewLocations) yield return location;
    }
    
    public void PopulateValues(ViewLocationExpanderContext context)
    {
        // Store any data needed for expansion in context.Values
        // These values become part of the cache key!
        context.Values["tenant"] = GetTenant(context.ActionContext);
    }
}
```

**Important:** Values stored in `PopulateValues()` become part of the view location cache key. If you don't store the value, the cached location won't vary by tenant.

## 10.4 Razor Compilation: Runtime vs Build-Time

**Two modes:**

**1. Runtime compilation (default in development):**
```
File on disk → compiled on first request → cached in memory
File changes → re-compiled on next request
```

**2. Build-time compilation (production, via RazorCompileOnBuild):**
```
dotnet build → Razor files compiled to assembly → shipped as DLL
No disk files needed at runtime
Faster startup (no compilation), faster first request
```

**Enabling runtime compilation in production:**
```csharp
builder.Services.AddControllersWithViews()
    .AddRazorRuntimeCompilation(); // nuget: Microsoft.AspNetCore.Mvc.Razor.RuntimeCompilation
```

**Why use build-time?** Startup time, no Roslyn overhead at runtime, smaller deployed footprint (no .cshtml files needed), compile-time error detection.

## 10.5 _ViewStart.cshtml and Layout Chain

`_ViewStart.cshtml` is executed before every view in its directory (and subdirectories).

```
Rendering Products/Index.cshtml:
  ↓
  1. Find _ViewStart.cshtml files up the tree:
     /Views/Products/_ViewStart.cshtml (if exists)
     /Views/_ViewStart.cshtml (most common)
  ↓
  2. Execute _ViewStart.cshtml first:
     @{ Layout = "_Layout"; }
  ↓
  3. Execute Index.cshtml:
     Renders the body section
  ↓
  4. Execute _Layout.cshtml:
     @RenderBody() → inserts body from step 3
     @RenderSection("scripts", required: false) → optional sections
```

**_ViewImports.cshtml** is similar but only imports:
- `@using` directives
- `@addTagHelper` directives
- `@model` (type defaults)
These are merged hierarchically and applied to all views.

## 10.6 View Rendering Internal Steps

```
ViewResultExecutor.ExecuteAsync(actionContext, viewResult)
  ↓
  1. IViewEngine.FindView() — locate the view
  ↓
  2. Create ViewContext:
     - HttpContext
     - ActionContext
     - ViewData (typed ViewDataDictionary<TModel>)
     - TempData
     - TextWriter (the response body writer)
  ↓
  3. IView.RenderAsync(viewContext)
     ↓
     RazorView.RenderAsync():
       a. Execute _ViewStart chain (sets Layout)
       b. Execute main view (RazorPage.ExecuteAsync)
          → Writes HTML chunks to StringWriter (body buffer)
       c. If Layout set:
          → Execute layout page
          → At @RenderBody(): flush body buffer
          → At @RenderSection(): flush section buffers
       d. Final output → Response.Body stream
```

**The double-buffering is important:** View content is first written to a `StringWriter` (in memory). Only after the layout is fully rendered does it get written to the actual response stream. This allows sections to be defined in views but rendered in layout headers.

---

# MODULE 11: TAG HELPERS INTERNALS

## 11.1 What Tag Helpers Are Under the Hood

Tag Helpers are C# classes that participate in Razor view rendering by targeting specific HTML elements/attributes.

```cshtml
<a asp-controller="Products" asp-action="Edit" asp-route-id="@Model.Id">Edit</a>
```

This is NOT a method call. During Razor compilation, the compiler:
1. Detects `asp-controller`, `asp-action`, `asp-route-id` attributes
2. Matches to `AnchorTagHelper` (which targets `<a>` elements with these attributes)
3. Generates code to instantiate and invoke `AnchorTagHelper`

## 11.2 Tag Helper Execution Flow

```
Razor page renders <a asp-controller="..." ...>
  ↓
Generated code:
  AnchorTagHelper tagHelper = new AnchorTagHelper(IUrlHelperFactory, ...);
  // Properties set:
  tagHelper.Controller = "Products";
  tagHelper.Action = "Edit";
  tagHelper.RouteValues["id"] = Model.Id.ToString();
  ↓
tagHelper.ProcessAsync(TagHelperContext, TagHelperOutput)
  ↓
AnchorTagHelper.Process():
  - Calls IUrlHelper.Action("Edit", "Products", new { id = ... })
  - IUrlHelper → uses RouteValueDictionary → generates URL
  - Sets output.Attributes["href"] = generatedUrl
  ↓
TagHelperOutput rendered as final HTML
```

## 11.3 Tag Helper Discovery — @addTagHelper

`@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers` in `_ViewImports.cshtml`:

This tells the Razor compiler to scan the named assembly for all classes implementing `ITagHelper` and register them.

At compile time:
```
RazorEngine processes @addTagHelper directive
  → TagHelperDescriptorProvider scans assembly
  → Creates TagHelperDescriptor for each tag helper (metadata: target element, attributes)
  → These descriptors are cached
  → Used during Razor compilation to know which elements have tag helpers
```

**Tag helpers are resolved per-view, per-element match, at compilation time** — not at runtime via reflection.

## 11.4 Form Tag Helpers and Anti-Forgery Integration

```cshtml
<form asp-controller="Products" asp-action="Create" method="post">
```

`FormTagHelper` automatically:
1. Generates the `action` URL
2. **Injects the anti-forgery token hidden field** if the method is POST/PUT/PATCH/DELETE

This means using `<form asp-...>` implicitly adds anti-forgery protection.

---

# MODULE 12: ANTI-FORGERY TOKEN INTERNALS

## 12.1 Why CSRF Protection Exists in MVC

In a server-rendered app, the browser holds the session cookie. Any site can make the browser send a POST to your app — the browser automatically includes the cookie. The attacker doesn't need to read the cookie — just triggering the request is enough.

Anti-forgery tokens solve this by requiring a secret that:
1. Is embedded in the rendered HTML form (hidden field)
2. Must match a value in a cookie

An attacker can't read the hidden field from the HTML (same-origin policy), so they can't include it in their forged request.

## 12.2 Token Generation and Validation Internals

**Token structure:**

```
Each token pair:
  - Cookie token: AntiforgeryToken stored as encrypted cookie
  - Field token: Hidden input value (different random value, but linked to cookie token)

Both tokens together form a valid pair.
```

**Generation (in view rendering):**

```cshtml
@Html.AntiForgeryToken()
<!-- OR automatically via Form Tag Helper -->
```

```
IAntiforgery.GetAndStoreTokens(HttpContext)
  ↓
  1. Check if cookie token already exists for this session
  2. If not: generate new cookie token, set cookie
  3. Generate new field token (cryptographically linked to cookie token via HMAC)
  4. Return AntiforgeryTokenSet { CookieToken, FormFieldName, HeaderName, RequestToken }
```

**Validation (in action execution):**

```
[ValidateAntiForgeryToken] attribute is an Authorization Filter
  ↓
IAntiforgery.ValidateRequestAsync(HttpContext)
  ↓
  1. Read cookie token from cookie
  2. Read field token from form data (or request header X-XSRF-TOKEN for AJAX)
  3. Validate both tokens match (via HMAC verification)
  4. If invalid → throw AntiforgeryValidationException → 400 response
```

## 12.3 Anti-Forgery with AJAX and SPAs

For AJAX requests, the token can be passed as a header:
```javascript
// Read from meta tag or cookie
const token = document.querySelector('meta[name="antiforgery-token"]').content;
fetch('/api/products', {
    method: 'POST',
    headers: { 'X-XSRF-TOKEN': token },
    body: JSON.stringify(data)
});
```

ASP.NET Core's `IAntiforgery` checks both form fields AND `X-XSRF-TOKEN` header.

**Production consideration:** The `__RequestVerificationToken` cookie is HTTP-only by default. For SPAs, you might need to configure it as non-HTTP-only so JavaScript can read it.

---

# MODULE 13: VIEWDATA, VIEWBAG, TEMPDATA INTERNALS

## 13.1 ViewData — The Weakly-Typed Dictionary

```csharp
public ViewDataDictionary ViewData { get; set; }
```

`ViewDataDictionary` is backed by an `IDictionary<string, object>` plus a special `Model` property.

**Internally:**
- Created by `IViewDataDictionaryFactory` per request
- Flows from Controller → View (passed in ViewContext)
- Shares state: same object passed to layout, partials

**ViewData["Key"]** — requires casting on read. Type safety is zero.

**The Model property:**
```csharp
// In ViewDataDictionary<TModel>:
public TModel Model { get; set; }  // Strongly typed
```

When you use `@model ProductViewModel` in a view, Razor generates `RazorPage<ProductViewModel>` which uses `ViewDataDictionary<ProductViewModel>`. The `Model` property is type-safe.

## 13.2 ViewBag — Dynamic Wrapper

`ViewBag` is a `DynamicViewData` wrapper over `ViewData`:

```csharp
// Controller.ViewBag is defined as:
public dynamic ViewBag
{
    get
    {
        if (_viewBag == null) _viewBag = new DynamicViewData(() => ViewData);
        return _viewBag;
    }
}
```

`DynamicViewData` implements `DynamicObject`:
- `TryGetMember` → reads `ViewData[name]`
- `TrySetMember` → writes `ViewData[name]`

So `ViewBag.Foo = "bar"` is exactly `ViewData["Foo"] = "bar"`.

**Performance:** `DynamicViewData` uses `dynamic` — requires runtime DLR invocation. **Do not use in hot paths.** For production code, prefer typed ViewModels or ViewData with explicit casting.

## 13.3 TempData — The Most Misunderstood State

TempData survives ONE redirect. This is its defining characteristic.

**Internal flow:**

```
Action 1: TempData["Message"] = "Saved!"
  ↓
Response: Redirect(302) to Action 2
  ↓
TempData provider serializes TempData to storage:
  - Default: CookieTempDataProvider (cookie-based)
  - Alternative: SessionStateTempDataProvider
  ↓
Browser follows redirect → Request for Action 2
  ↓
TempData provider reads from storage → deserializes
  ↓
Action 2: var msg = TempData["Message"]  // "Saved!"
  ↓
TempData["Message"] is MARKED for deletion
  ↓
After response: marked entries deleted from storage
```

**"Keep" and "Peek":**
- `TempData.Keep("Message")` — prevents deletion after this request
- `TempData.Peek("Message")` — reads WITHOUT marking for deletion

**CookieTempDataProvider internals:**
```
Serialize Dictionary<string, object> → JSON → Base64 → AES-GCM encrypt → Cookie value
```

The encryption uses ASP.NET Core Data Protection. Without Data Protection configured, TempData won't work across server instances (load balancing).

## 13.4 Session State Internals

Session in ASP.NET Core is NOT in-memory per default like classic ASP.NET. It requires a backing store.

**Setup:**
```csharp
builder.Services.AddDistributedMemoryCache(); // or Redis, SQL, etc.
builder.Services.AddSession(options => {
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.HttpOnly = true;
    options.Cookie.IsEssential = true;
});
app.UseSession(); // Must be after UseRouting, before UseEndpoints
```

**Request flow:**

```
Request arrives with session cookie (or no cookie on first visit)
  ↓
SessionMiddleware.InvokeAsync():
  1. Read session ID from cookie
  2. Create DistributedSession object (lazy — not loaded yet)
  3. Set ISessionFeature.Session = distributedSession
  ↓
Action accesses HttpContext.Session.GetString("key"):
  1. Checks if session loaded
  2. If not: IDistributedCache.GetAsync(sessionId) → load from backing store
  3. Deserialize to Dictionary<string, byte[]>
  4. Return value
  ↓
Session modified (SetString, etc.):
  - Changes tracked in-memory
  ↓
SessionMiddleware OnResponseCompleted:
  1. If session modified: IDistributedCache.SetAsync(sessionId, serialized, expiry)
  2. Set/refresh session cookie
```

**Memory:** Session data is kept in memory (in the `DistributedSession` object) for the duration of the request, then persisted to the backing store.

**Serialization:** Session stores `byte[]` values. `GetString`/`SetString` uses UTF-8 encoding. For complex objects, you must manually serialize to JSON and store as string.

---

# MODULE 14: AUTHENTICATION AND AUTHORIZATION IN MVC DEEP DIVE

## 14.1 The Authentication Middleware Flow

```
AuthenticationMiddleware.InvokeAsync():
  ↓
  1. IAuthenticationSchemeProvider.GetDefaultAuthenticateSchemeAsync()
     → Returns the default scheme name (e.g., "Cookies" or "Bearer")
  ↓
  2. IAuthenticationService.AuthenticateAsync(context, schemeName)
     → Calls IAuthenticationHandler.AuthenticateAsync()
     For Cookie auth:
       - Reads authentication cookie
       - Decrypts using Data Protection
       - Deserializes ClaimsPrincipal
       - Checks expiry
       → Returns AuthenticateResult.Success(ticket)
     For JWT Bearer:
       - Reads Authorization: Bearer <token> header
       - Validates signature, expiry, issuer, audience
       → Returns AuthenticateResult.Success(ticket)
  ↓
  3. If success: HttpContext.User = ticket.Principal
     If failure: HttpContext.User = new ClaimsPrincipal(anonymous identity)
  ↓
  4. Call next middleware (authorization)
```

## 14.2 Authorization Middleware vs Authorization Filter

**Two separate layers:**

```
AuthorizationMiddleware (step [13] in pipeline):
  - Runs for ALL endpoints with authorization metadata
  - Checks endpoint metadata for IAuthorizeData (from [Authorize] attributes)
  - Calls IAuthorizationService.AuthorizeAsync()
  - If failed: calls IAuthorizationMiddlewareResultHandler
    → 401 if not authenticated
    → 403 if authenticated but not authorized
    → This short-circuits before reaching MVC
  
Authorization Filter (IAuthorizationFilter, runs in filter pipeline):
  - Runs inside MVC, after endpoint matched
  - More granular: can check ActionDescriptor, route values
  - Older pattern; less preferred than policy-based authorization
```

**Modern recommendation:** Use AuthorizationMiddleware with policy-based authorization. Avoid `IAuthorizationFilter` for new code.

## 14.3 Cookie Authentication Flow in MVC

**Sign in:**
```csharp
await HttpContext.SignInAsync(
    CookieAuthenticationDefaults.AuthenticationScheme,
    new ClaimsPrincipal(new ClaimsIdentity(claims, "Cookies")),
    new AuthenticationProperties { IsPersistent = rememberMe });
```

Internally:
```
IAuthenticationService.SignInAsync()
  → CookieAuthenticationHandler.HandleSignInAsync()
      1. Creates AuthenticationTicket (ClaimsPrincipal + properties)
      2. Serialize ticket to bytes
      3. Encrypt with Data Protection (AES-GCM + HMAC)
      4. Optionally chunk if too large (multi-chunk cookies)
      5. Set-Cookie header in response
```

**On subsequent requests:**
```
Cookie arrives in request
  → CookieAuthenticationHandler.HandleAuthenticateAsync()
      1. Read cookie value
      2. Decrypt + deserialize → AuthenticationTicket
      3. Check IsPersistent, expiry
      4. Optionally call ValidatePrincipal event for server-side invalidation
      5. Return ClaimsPrincipal
```

## 14.4 JWT in MVC Apps vs API Apps

**Difference in behavior:**

| Scenario | Cookie Auth (MVC) | JWT Bearer (API) |
|---|---|---|
| Token storage | HTTP-only cookie (server-set) | Client controls (localStorage, memory) |
| Automatic send | Browser sends cookie automatically | Client must set Authorization header |
| XSS risk | Low (HTTP-only) | High if stored in localStorage |
| CSRF risk | High (needs anti-forgery) | Low (header-based) |
| Statelessness | Partially (session can be server-side) | Fully stateless |
| Expiry handling | Sliding window possible | Must refresh token explicitly |

**MVC apps typically use cookies. APIs use JWT.** In a hybrid architecture:
- MVC app renders server-side views with cookie auth
- MVC app calls its own Web API using an `HttpClient` with cookie propagation OR by passing JWT obtained after cookie sign-in

## 14.5 Data Protection — The Cryptographic Heart

ASP.NET Core Data Protection is used by:
- Cookie authentication (encrypts ticket)
- Anti-forgery tokens (encrypts/signs tokens)
- TempData (encrypts cookie TempData)
- Session (signs session ID)

```
IDataProtectionProvider
  → IDataProtector = provider.CreateProtector("purpose-string")
  
protector.Protect(plaintext byte[]) → ciphertext
protector.Unprotect(ciphertext byte[]) → plaintext (or throws if invalid)
```

**Production critical:** If you don't persist the Data Protection key ring:
- After app restart, all encrypted cookies are invalid → users logged out
- In load-balanced environments, server B can't decrypt cookies encrypted by server A

```csharp
// Persist keys to shared location:
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(...)  // Azure
    .PersistKeysToFileSystem(new DirectoryInfo("/shared/keys"))  // File system
    .ProtectKeysWithAzureKeyVault(...)   // Encrypt the keys themselves
```

---

# MODULE 15: RAZOR PAGES VS MVC — INTERNAL ARCHITECTURE

## 15.1 How Razor Pages Differ Internally

Razor Pages is a page-focused framework built on the same MVC infrastructure but with a different programming model.

```
MVC:
  URL → Route → Controller class → Action method → View file
  (separate C# class from view)

Razor Pages:
  URL → Route → PageModel class → Handler method (OnGet, OnPost, etc.)
  (PageModel is in same .cshtml file as code-behind)
```

**Internally, Razor Pages is MVC with:**
- `PageActionDescriptor` instead of `ControllerActionDescriptor`
- `PageHandlerMethod` selected by `IPageHandlerMethodSelector`
- `PageContext` instead of `ControllerContext`
- `IPageActivatorProvider` instead of `IControllerActivator`

The filter pipeline, model binding, and result execution are identical.

---

# MODULE 16: VIEWMODELS DEEP DIVE

## 16.1 ViewModel Design Patterns and Data Flow

The ViewModel flow in MVC:

```
Controller Action (GET):
  1. Query domain model from repository/service
  2. Map domain model → ViewModel (Automapper or manual)
  3. Return View(viewModel)
     ↓
View renders ViewModel properties
  ↓

Controller Action (POST):
  1. Model binding maps form data → ViewModel (or input model)
  2. Validate ViewModel (DataAnnotations, FluentValidation)
  3. If invalid: return View(viewModel) to redisplay with errors
  4. If valid: map ViewModel → domain model
  5. Save via service/repository
  6. Redirect (PRG pattern)
```

## 16.2 Why Separate ViewModels from Domain Models

Production reasons:
1. **Shape mismatch:** Domain model has ALL data; view needs SUBSET
2. **Security:** Domain model may have fields you don't want exposed (e.g., PasswordHash)
3. **Aggregation:** View might combine data from multiple domain models
4. **Validation:** View validation rules differ from domain validation rules
5. **Display logic:** `FullName = $"{First} {Last}"` is view concern, not domain concern

**Mass assignment vulnerability:** If you bind directly to domain model:
```csharp
// DANGEROUS:
public IActionResult Update([Bind] User user) { db.Update(user); }
// Attacker can POST user.IsAdmin=true and your binding will set it

// SAFE:
public IActionResult Update(UpdateUserViewModel vm)
{
    var user = db.Users.Find(vm.Id);
    user.Name = vm.Name; // Only update allowed fields
    db.SaveChanges();
}
```

## 16.3 [BindingBehavior] and [Bind] Attributes

```csharp
// Whitelist properties for binding:
[Bind("Name,Email")]
public IActionResult Update(User user) { }

// Or use BindingBehavior on model:
public class CreateUserViewModel
{
    [BindRequired]   // Must be present in request
    public string Name { get; set; }
    
    [BindNever]      // Never bound from request
    public string SecurityStamp { get; set; }
}
```

Internally, `[BindNever]` sets `BindingInfo.BindingSource = BindingSource.None` in the `ModelMetadata` — the model binder skips the property.

---

# MODULE 17: MODEL VALIDATION INTERNALS

## 17.1 Validation Pipeline

```
After model binding completes:
  ↓
IObjectModelValidator.Validate(actionContext, validationState, prefix, model)
  ↓
ValidationVisitor traverses the object graph:
  - For each property:
    - Check ModelMetadata.ValidatorMetadata (DataAnnotations)
    - Run each IModelValidator (DataAnnotationsModelValidator)
      → validator.Validate(context) → IEnumerable<ModelValidationResult>
    - Add errors to ModelStateDictionary
  - Recursively validate nested objects
```

## 17.2 DataAnnotations vs FluentValidation Internally

**DataAnnotations validation flow:**

```
[Required], [MaxLength(100)], [EmailAddress] etc.
  ↓
These are ValidationAttribute subclasses
  ↓
DataAnnotationsModelValidatorProvider wraps them in IModelValidator
  ↓
Called during ValidateAsync by ValidationVisitor
  ↓
ValidationAttribute.IsValid(value, validationContext) called
  ↓
Errors added to ModelState
```

**FluentValidation (third-party) integration:**
```
Registers custom IModelValidator implementations
  OR
Intercepts via IObjectModelValidator replacement
  ↓
Runs fluent rules and populates ModelState
```

## 17.3 ModelState.IsValid — What It Really Checks

```csharp
if (!ModelState.IsValid) 
{
    return View(model);
}
```

`IsValid` is `true` only when ALL entries in `ModelStateDictionary` have `ModelValidationState.Valid`. A single `ModelValidationState.Invalid` or `ModelValidationState.Unvalidated` can affect this.

**When is something Unvalidated?** When the bound value was never validated (e.g., deep in object graph, skipped due to MaxValidationDepth setting). By default `MaxValidationDepth` is 32 levels.

## 17.4 [ApiController] and Automatic Validation Response

With `[ApiController]`:
```
ModelStateInvalidFilter runs as an action filter (Order = -2000):
  ↓
  if (!context.ModelState.IsValid)
  {
      var factory = context.HttpContext.RequestServices
          .GetRequiredService<ProblemDetailsFactory>();
      var problemDetails = factory.CreateValidationProblemDetails(
          context.HttpContext, context.ModelState);
      context.Result = new BadRequestObjectResult(problemDetails);
  }
```

This runs **before** your action method, so you never even reach your action code. This is why `[ApiController]` actions don't need `if (!ModelState.IsValid)` checks.

---

# MODULE 18: MVC FRONTEND CALLING BACKEND API — ARCHITECTURE PATTERNS

## 18.1 Server-Side Rendered MVC vs API Architecture

```
Pattern 1: Pure Server-Side Rendering (Classic MVC)
  Browser → [HTTP GET] → MVC Controller
  Controller → queries database/services directly
  Controller → returns View(model)
  MVC renders HTML on server → sends full HTML to browser
  
  Pros: SEO, simpler architecture, no CORS
  Cons: Full page reloads, tight coupling, harder to scale frontend
  
Pattern 2: MVC + API (Hybrid)
  Browser → [HTTP GET] → MVC Controller (renders shell HTML)
  Browser → [HTTP GET/POST/Ajax] → Web API Controller (returns JSON)
  JavaScript → updates DOM with JSON data
  
  Pros: Dynamic UX, partial updates, API reuse
  Cons: Complexity, CORS, anti-forgery token handling for AJAX

Pattern 3: BFF (Backend-for-Frontend)
  Browser → [HTTP] → MVC App (BFF)
  MVC App → [HTTP/gRPC] → Microservice APIs (internal)
  MVC renders aggregated data as HTML
  
  Pros: Aggregation, security boundary, schema translation
  Cons: Extra network hop, more infrastructure
```

## 18.2 HttpClient Internally in MVC Controllers

When MVC calls an API internally:

```csharp
// Register (in Program.cs):
builder.Services.AddHttpClient<IProductApiClient, ProductApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
});

// Use in controller:
public class ProductsController : Controller
{
    private readonly IProductApiClient _api;
    public ProductsController(IProductApiClient api) => _api = api;
    
    public async Task<IActionResult> Index()
    {
        var products = await _api.GetProductsAsync();
        return View(products);
    }
}
```

**IHttpClientFactory internals:**
- Creates/pools `HttpMessageHandler` instances (manages DNS refresh, connection pooling)
- Each named/typed client gets handlers from the pool
- Handlers are recycled every 2 minutes by default (avoids DNS staleness with long-lived HttpClient)
- Not creating a new `HttpClient` per request (that exhausts socket connections)

## 18.3 Passing Authentication Between MVC and API

**Scenario:** MVC app (cookie-auth) calling internal API (JWT-auth)

```
Option 1: Token exchange
  - User logs in via MVC (gets cookie)
  - MVC app gets JWT from auth server using client credentials
  - MVC calls API with JWT Bearer header

Option 2: Cookie forwarding (for same domain)
  - HttpClient configured to forward cookies
  - Works if both MVC and API share same auth domain

Option 3: User delegation (OAuth2 On-Behalf-Of)
  - MVC receives user's JWT
  - MVC exchanges for delegated token
  - API validates delegated token

Production code:
  services.AddHttpClient<IProductApiClient>()
      .AddHttpMessageHandler<BearerTokenHandler>();
  
  // BearerTokenHandler reads token from IHttpContextAccessor
  // and adds it to outgoing request
```

---

# MODULE 19: ASYNC/AWAIT AND THREAD BEHAVIOR IN MVC

## 19.1 ASP.NET Core Thread Model

Classic ASP.NET (System.Web) used a thread-per-request model from a fixed thread pool. ASP.NET Core uses a fundamentally different model:

```
Kestrel I/O threads (configured count):
  - Read HTTP data
  - Call into application pipeline
  
.NET Thread Pool (worker threads):
  - Execute application code
  - When async I/O is awaited, thread returns to pool
  - When I/O completes, a (possibly different) thread resumes
```

**No SynchronizationContext in ASP.NET Core.** This is a critical difference from ASP.NET (System.Web) which used `AspNetSynchronizationContext`.

## 19.2 Async Action Methods — How MVC Handles Them

```csharp
public async Task<IActionResult> Index()
{
    var products = await _productService.GetAllAsync(); // I/O
    return View(products);
}
```

**Execution flow:**

```
MVC thread: calls action method
  ↓
action method runs synchronously until first await
  ↓
await _productService.GetAllAsync() → if I/O not complete:
  - State machine captures local variables
  - Returns incomplete Task to MVC
  - MVC thread returns to thread pool (NOT blocked)
  ↓
[Time passes — I/O completes]
  ↓
ThreadPool thread picks up continuation
  - Restores state machine
  - Continues running action method
  - (May be a DIFFERENT thread than before the await)
  ↓
return View(products) → MVC gets the completed Task<IActionResult>
  ↓
Result execution continues
```

**Key insight:** There is NO thread affinity in ASP.NET Core async code. The `HttpContext` flows via `AsyncLocal` (part of `ExecutionContext`), not thread-local storage.

## 19.3 `ConfigureAwait(false)` in Library Code

```csharp
// Library/service code:
public async Task<Product> GetProductAsync(int id)
{
    var data = await _db.Products.FindAsync(id).ConfigureAwait(false);
    // ConfigureAwait(false): don't need to resume on specific context
    return MapToProduct(data);
}
```

In ASP.NET Core: `ConfigureAwait(false)` is technically unnecessary (no SynchronizationContext), but it's still a good practice in library code because:
1. The library might be used in other contexts (WPF, WinForms) that DO have SynchronizationContext
2. It signals intent: "I don't care which thread I resume on"

**Warning:** Never use `ConfigureAwait(false)` then access `HttpContext` — in a context-free thread, `IHttpContextAccessor.HttpContext` may be null.

## 19.4 Deadlock Patterns (Classic ASP.NET vs Core)

**In Classic ASP.NET (System.Web):** Deadlock when blocking on async code:
```csharp
// DEADLOCK in classic ASP.NET:
var result = GetProductAsync().Result; // Blocks thread
// Async continuation needs the same SynchronizationContext thread that's blocked
// = deadlock
```

**In ASP.NET Core:** This doesn't deadlock (no SynchronizationContext), but it STILL blocks a thread unnecessarily, reducing scalability:
```csharp
// BAD (blocks thread, no deadlock but wasteful):
var result = GetProductAsync().Result;

// GOOD:
var result = await GetProductAsync();
```

---

# MODULE 20: SCOPED/TRANSIENT/SINGLETON IN MVC LIFECYCLE

## 20.1 Lifetime Behavior Per Request

```
Request lifetime (one HTTP request):

Singleton service:
  - Created once when first requested (or at startup if called eagerly)
  - Shared across ALL requests, ALL users
  - Lives until application shuts down
  - Same instance in every controller in every request
  
Scoped service:
  - Created once per request (when IServiceScope is created)
  - Shared within a single request (same instance for controller, filters, services called during that request)
  - Disposed when request ends (IServiceScope.Dispose())
  - Different instance in different requests
  
Transient service:
  - New instance EVERY time it's resolved from IServiceProvider
  - Even within the same request: multiple resolutions = multiple instances
  - Disposed when the scope is disposed (if IDisposable)
  
Example:
  ProductsController constructor called
    → IProductService resolved: SCOPED → instance A (for this request)
  
  Inside ProductsController.Index():
    → _productService.GetAll() → instance A does work
  
  A filter on ProductsController
    → IProductService resolved: same SCOPED → still instance A
  
  New request comes in
    → IProductService resolved: new SCOPED → instance B
```

## 20.2 Captive Dependency Anti-Pattern

```
Singleton A depends on Scoped B:
  
  A is created once (singleton)
  A holds reference to B
  B is supposed to be per-request, but A captured it at startup
  B is effectively singleton — WRONG BEHAVIOR
  
ASP.NET Core detects this at startup in development mode:
  InvalidOperationException: "Cannot consume scoped service 'B' from singleton 'A'"
```

**Solution:** If a singleton needs per-request behavior, inject `IServiceScopeFactory` and create a scope manually:

```csharp
public class SingletonBackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public async Task DoWorkAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var scopedService = scope.ServiceProvider.GetRequiredService<IProductService>();
        await scopedService.DoWorkAsync();
        // scope disposed here → scopedService disposed
    }
}
```

## 20.3 Controller Lifetime (Common Misconception)

**Misconception:** "Controllers are transient because they're created per request."

**Reality:** Controllers are technically **not registered in the DI container as transient**. They're activated by `IControllerActivator` on each request. The distinction:

- If you DO register your controller explicitly (e.g., `services.AddTransient<ProductsController>()`), DI is used
- By default, `DefaultControllerActivator` uses `ActivatorUtilities.CreateInstance()` — uses DI for dependencies but the controller itself is NOT in the service registry
- This is intentional — controllers shouldn't be resolved directly from DI in general code

---
# ASP.NET Core MVC — Complete Internals Handbook
## Part 3: Advanced Internals, Viva Questions, End-to-End Lifecycle, Debugging

---

# MODULE 21: MIDDLEWARE-TO-MVC TRANSITION INTERNALS

## 21.1 The Exact Handoff Point

The transition from "middleware world" to "MVC world" is one of the least understood aspects. Let's trace it exactly.

```
EndpointMiddleware.InvokeAsync(HttpContext context):
  ↓
  var endpoint = context.GetEndpoint();  // read from IEndpointFeature
  
  if (endpoint == null) 
  {
      await _next(context);  // pass to next middleware (404 path)
      return;
  }
  
  var requestDelegate = endpoint.RequestDelegate;
  // requestDelegate IS MVC's entry point
  ↓
  await requestDelegate(context);
  // This calls into MVC's ActionInvoker
```

**What is `requestDelegate` for an MVC action?**

During `MapControllers()` / `UseEndpoints()`, each `ControllerActionDescriptor` is wrapped:

```csharp
// Simplified from source (Microsoft.AspNetCore.Mvc.Infrastructure):
var requestDelegate = async (HttpContext ctx) =>
{
    var routeData = ctx.GetRouteData();
    var actionDescriptor = endpoint.Metadata.GetMetadata<ActionDescriptor>();
    var actionContext = new ActionContext(ctx, routeData, actionDescriptor);
    
    var invokerFactory = ctx.RequestServices
        .GetRequiredService<IActionInvokerFactory>();
    var invoker = invokerFactory.CreateInvoker(actionContext);
    
    await invoker.InvokeAsync();
};
```

This is the bridge. The `RequestDelegate` is a closure that:
1. Reads route data from `HttpContext`
2. Gets the `ActionDescriptor` from endpoint metadata
3. Creates an `ActionContext`
4. Resolves `IActionInvokerFactory` from DI
5. Creates the invoker
6. Calls `InvokeAsync()`

## 21.2 ControllerActionInvoker — The Heart of MVC Execution

`ControllerActionInvoker` orchestrates the entire MVC execution:

```
ControllerActionInvoker.InvokeAsync():
  ↓
  InvokeFilterPipelineAsync():
    ↓
    // Builds filter pipeline state machine
    // State machine iterates through filters in correct order
    
    State: AuthorizationFilters
      → For each IAuthorizationFilter:
          authFilter.OnAuthorization(context)
          If context.Result set → short circuit → go to ResultExecution
    ↓
    State: ResourceFilters (Before)
      → For each IResourceFilter:
          resourceFilter.OnResourceExecuting(context)
          If context.Result set → short circuit
    ↓
    State: ModelBinding
      → ParameterBinder.BindModelAsync() for each parameter
      → IObjectModelValidator.Validate()
    ↓
    State: ActionFilters (Before) + Action Execution + ActionFilters (After)
      → This is a nested delegate chain
    ↓
    State: ResultFilters (Before) + Result Execution + ResultFilters (After)
    ↓
    State: ResourceFilters (After)
```

The state machine is implemented as a `Next()` method that advances through states. Each state can set `context.Result` to short-circuit. When short-circuited, execution jumps to ResultExecution state.

## 21.3 Short-Circuit Mechanics

```
ActionExecutingContext.Result is set by a filter:
  ↓
ControllerActionInvoker detects Result != null
  ↓
Action method is SKIPPED
  ↓
Result Filters run
  ↓
Result.ExecuteResultAsync() runs
  ↓
Response written
```

This is used by:
- Authorization filters (401/403)
- Resource filters (cache hit)
- Action filters (request validation)
- `ModelStateInvalidFilter` (auto 400 for [ApiController])

---

# MODULE 22: PERFORMANCE INTERNALS AND OPTIMIZATIONS

## 22.1 What the Framework Caches

Understanding what's cached is crucial for debugging and optimization:

```
STARTUP-TIME CACHE (lives for app lifetime):
  ├── ApplicationModel (controller/action/parameter metadata)
  ├── ControllerActionDescriptor[] (route + filter + binding metadata)
  ├── Route matcher DFA (compiled automaton)
  ├── ObjectMethodExecutor (compiled expression trees for each action)
  ├── ActionMethodExecutor selection (sync/async/void determination)
  ├── ModelMetadata (type information, validators, display names)
  │    Stored in ConcurrentDictionary<Type, ModelMetadata>
  ├── ObjectFactory per controller type (ActivatorUtilities compiled factory)
  └── Tag helper descriptors (per assembly)

REQUEST-TIME CACHE (per-request, then discarded):
  ├── IValueProvider[] (created from factories each request)
  ├── Filter instances (for static filters; dynamic filters recreated)
  └── Model binder instances (from IModelBinderFactory cache)

VIEW COMPILATION CACHE:
  ├── Compiled Razor assemblies (keyed by view path + file hash)
  ├── Layout + partial view compiled pages
  └── View location cache (keyed by controller/action + expander values)
```

## 22.2 ModelMetadata Cache — Deep Dive

`DefaultModelMetadataProvider` maintains a cache:

```csharp
// Internal cache structure:
ConcurrentDictionary<ModelMetadataIdentity, ModelMetadataCacheEntry>
```

`ModelMetadataIdentity` is:
- For type: `{ Type, PropertyName = null }`
- For property: `{ Type, PropertyName, ContainerType }`
- For parameter: `{ Type, ParameterInfo }`

**First request for a type triggers metadata construction:**
```
Type → Reflect all properties
  → For each property:
      - DataAnnotation attributes
      - Display attributes
      - Validation attributes
      - [BindNever], [Required] etc.
  → Build ModelMetadata tree
  → Cache it
```

**Production implication:** First request to a new endpoint type is slower (metadata construction). After that, it's O(1) cache lookup. This is one reason for "warmup" requests in production.

## 22.3 Response Compression and Buffering

By default, ASP.NET Core **does not buffer** the response. Writing to `Response.Body` goes directly to Kestrel's output buffer.

**For Razor views:** The view writes to a `StringWriter` (in-memory buffer), then flushes to `Response.Body` after layout rendering is complete. This is an exception to the non-buffering rule — necessary because layout needs to wrap the view content.

**Response compression middleware:**
```
ResponseCompressionMiddleware wraps Response.Body:
  → Compresses output stream
  → Chooses algorithm based on Accept-Encoding header
  → GZip or Brotli
  → Only if Content-Type is in allowed types
```

**MinimumCompressionSize:** Default 1KB — responses smaller than this aren't compressed (overhead not worth it).

## 22.4 Connection and Thread Pool Tuning

```
Kestrel configuration:
  options.Limits.MaxConcurrentConnections = 100; // null = unlimited
  options.Limits.MaxRequestBodySize = 30_000_000; // 30MB default
  
Thread Pool:
  ThreadPool.SetMinThreads(workerThreads, completionPortThreads);
  // Default: CPU count for both
  // For I/O heavy workloads: increase to avoid thread pool starvation
  
Thread pool starvation pattern:
  Many sync-over-async calls (Result, .Wait())
  → All threads blocked
  → New work can't be scheduled
  → Request timeouts cascade
  Solution: use async all the way down
```

---

# MODULE 23: MOST COMMONLY MISUNDERSTOOD CONCEPTS

## 23.1 "ViewBag and ViewData are different things"
**Reality:** ViewBag is literally `dynamic` wrapper over ViewData. `ViewBag.Foo = 1` is `ViewData["Foo"] = 1`. They share the same dictionary.

## 23.2 "Filters run before middleware"
**Reality:** Middleware runs BEFORE filters. Filters are INSIDE MVC, which is INSIDE middleware. Order: Middleware → Endpoint routing → MVC → Filters.

## 23.3 "HttpContext is thread-safe"
**Reality:** `HttpContext` is NOT thread-safe. Accessing it from multiple threads (e.g., `Parallel.ForEach` in an action) can corrupt state. Never access `HttpContext` concurrently. Use `IHttpContextAccessor` carefully — it uses `AsyncLocal` which flows through `await` but not through `Task.Run()` without explicit capture.

## 23.4 "Session is automatic in ASP.NET Core"
**Reality:** Session requires explicit setup: `AddSession()`, `AddDistributedCache()`, `UseSession()`. And you must use it BEFORE writing to the response. Session middleware must appear before MVC in the pipeline.

## 23.5 "[ApiController] just adds swagger support"
**Reality:** `[ApiController]` makes significant behavioral changes:
- Automatic model validation (400 before action executes)
- Automatic binding source inference (complex types → [FromBody])
- Problem details error responses
- Route attribute required on controller

## 23.6 "TempData is just session"
**Reality:** By default, TempData uses COOKIES (not session). `CookieTempDataProvider` encrypts data into a cookie. To use session as backing store: `services.AddSingleton<ITempDataProvider, SessionStateTempDataProvider>()`.

## 23.7 "DbContext is safe to use as singleton"
**Reality:** `DbContext` is NOT thread-safe. It must be scoped. Using it as singleton means concurrent requests share the same context → race conditions, data corruption. EF Core will throw if you try to register it as singleton (it detects this).

## 23.8 "Routing attributes don't affect filter order"
**Reality:** Filter ORDER is determined by `Order` property and `Scope` (Global=0, Controller=10, Action=20). Route attributes have nothing to do with filter ordering. A global filter with Order=-1000 runs before any attribute-based filter regardless of route.

## 23.9 "Constructor injection and property injection are equivalent in MVC"
**Reality:** MVC supports constructor injection natively. Property injection requires `[FromServices]` attribute on the property, which is a MODEL BINDING source, not true DI property injection. The controller still gets its required dependencies through the constructor; `[FromServices]` on action parameters or properties is handled by `ServiceEntryModelBinder`.

## 23.10 "async void actions are fine"
**Reality:** `async void` actions are NOT supported and cause unpredictable behavior. MVC expects `Task` or `Task<IActionResult>`. With `async void`, exceptions are unhandled and the MVC pipeline can't await completion. Always use `async Task<IActionResult>`.

## 23.11 "Model binding reads the request body multiple times"
**Reality:** `Request.Body` is a non-rewindable stream. Once read by `[FromBody]` binding, it's exhausted. You CANNOT read it again. To read it multiple times, enable request body buffering:
```csharp
app.Use(async (context, next) =>
{
    context.Request.EnableBuffering();
    await next();
});
```

## 23.12 "Returning IActionResult vs ActionResult<T> is cosmetic"
**Reality:** `ActionResult<T>` enables:
- Implicit conversion (return `model` instead of `Ok(model)`)
- Type information for API documentation tools (Swagger knows the response type)
- Compile-time checking that the returned model is of the right type

```csharp
// With IActionResult:
public IActionResult GetProduct(int id)
    => Ok(new Product()); // Swagger doesn't know return type

// With ActionResult<T>:
public ActionResult<Product> GetProduct(int id)
    => new Product(); // implicit conversion, Swagger knows type
```

---

# MODULE 24: DEBUGGING SCENARIOS AND PRODUCTION PATTERNS

## 24.1 Debugging "View Not Found" Errors

```
ViewEngineResult.NotFound exception includes:
  "The view 'Edit' was not found. The following locations were searched:
   /Views/Products/Edit.cshtml
   /Views/Shared/Edit.cshtml"

Debug checklist:
  1. Check view filename matches action name exactly (case-sensitive on Linux)
  2. Check controller name prefix matches (ProductsController → Views/Products/)
  3. Check if view is in wrong folder
  4. If using Areas: check area registration and view folder structure
  5. Check build action is "Content" (not excluded from deployment)
  6. In Docker/Linux: filenames ARE case-sensitive (Windows is not)
     → /Views/Products/Edit.cshtml works on Windows
     → /Views/products/edit.cshtml fails on Linux
```

## 24.2 Debugging Model Binding Failures

```
Symptom: Action parameter is null or default, form data is not binding

Debug steps:
  1. Log value providers: add logging to ModelBinderFactory
  
  2. Check form field names match parameter/property names:
     Form: <input name="ProductName"> 
     Model: public string Name { get; set; } ← MISMATCH
     Fix: <input name="Name"> or [Bind] attribute
  
  3. For nested models, check prefix:
     <input name="Address.Street"> for Address.Street
  
  4. For collections:
     <input name="Tags[0]"> <input name="Tags[1]">
     OR <input name="Tags"> multiple times (depends on binder)
  
  5. Check Content-Type header:
     application/x-www-form-urlencoded → form binding
     multipart/form-data → form + file binding
     application/json → body binding (needs [FromBody])
  
  6. If using [FromBody]: only ONE parameter can be [FromBody]
     Body is read once
```

## 24.3 Debugging Filter Execution Order

```csharp
// To see filter order, enable logging:
builder.Services.AddLogging(logging => 
    logging.AddFilter("Microsoft.AspNetCore.Mvc", LogLevel.Debug));

// Or implement a debug filter:
public class DebugFilter : IOrderedFilter, IActionFilter
{
    public int Order => -9999; // Run first
    
    public void OnActionExecuting(ActionExecutingContext context)
    {
        var filters = context.Filters.Select(f => f.GetType().Name);
        Console.WriteLine($"Filters: {string.Join(", ", filters)}");
    }
    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

## 24.4 Debugging Memory Leaks in MVC Apps

Common memory leak patterns:

```
1. Singleton capturing HttpContext:
   public class MySingleton
   {
       private readonly HttpContext _ctx; // WRONG: captured reference
       // Fix: use IHttpContextAccessor and access .HttpContext per use
   }

2. Static collection growing unbounded:
   private static List<Request> _log = new(); // WRONG: never cleared
   // Fix: use IMemoryCache with expiry, or IDistributedCache

3. IDisposable not disposed in controller:
   public IActionResult Index()
   {
       var resource = new ExpensiveResource(); // WRONG: never disposed
       // Fix: using statement, or inject as scoped service (framework disposes)
   }

4. Event subscriptions not unsubscribed:
   // If controller subscribes to singleton events, it's never collected
   // Fix: controller implements IDisposable, unsubscribes in Dispose()

5. Large ViewBag/ViewData objects:
   ViewBag.HugeList = _db.Entities.ToList(); // 50,000 items
   // These stay in memory during rendering
   // Fix: paginate, stream data
```

## 24.5 Debugging Anti-Forgery Token Failures

```
Error: "The anti-forgery token could not be decrypted"

Causes and solutions:
  1. Data Protection keys rotated/changed → old tokens invalid
     Fix: Configure persistent key storage (Azure Blob, filesystem)
  
  2. Load balancer: request encrypted on Server A, validated on Server B
     Server B can't decrypt Server A's token
     Fix: Shared Data Protection key ring
  
  3. Form submitted without hidden token field
     Fix: Use <form asp-...> tag helper (auto-injects token)
         OR add @Html.AntiForgeryToken() manually
  
  4. AJAX request missing X-XSRF-TOKEN header
     Fix: Read token from cookie/meta and set header
  
  5. Token cookie blocked (CORS, SameSite=Strict policy)
     Fix: Review cookie policy for cross-origin scenarios
```

---

# MODULE 25: ENTERPRISE ARCHITECTURE PATTERNS WITH ASP.NET CORE MVC

## 25.1 Clean Architecture Layering with MVC

```
Presentation Layer (ASP.NET Core MVC):
  Controllers — thin orchestrators
  ViewModels — presentation-specific models
  Filters — cross-cutting concerns (auth, logging)
  Tag Helpers — reusable UI components
  
Application Layer (between Presentation and Domain):
  IProductService interface
  ProductService implementation
  Commands / Queries (CQRS pattern)
  DTOs for service boundaries
  Validators (FluentValidation)
  
Domain Layer:
  Domain entities
  Value objects
  Domain events
  Business rules / invariants
  Repository interfaces
  
Infrastructure Layer:
  EF Core DbContext
  Repository implementations
  External service clients (HttpClient)
  Caching implementations
```

**Rule:** Controllers should NEVER directly access DbContext. Controllers → Service → Repository → DbContext. This keeps controllers testable and thin.

## 25.2 CQRS with MVC

```csharp
// Command:
public class CreateProductCommand
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}

// Controller with MediatR:
public class ProductsController : Controller
{
    private readonly IMediator _mediator;
    
    public async Task<IActionResult> Create(CreateProductViewModel vm)
    {
        if (!ModelState.IsValid) return View(vm);
        
        var command = new CreateProductCommand { 
            Name = vm.Name, 
            Price = vm.Price 
        };
        
        var productId = await _mediator.Send(command);
        
        TempData["Success"] = "Product created";
        return RedirectToAction(nameof(Index));
    }
}
```

## 25.3 Global Exception Handling Strategy

```csharp
// Program.cs — layered error handling:

// Layer 1: Developer exception page (development only)
if (app.Environment.IsDevelopment())
    app.UseDeveloperExceptionPage();
else
{
    // Layer 2: Custom error page (production)
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}

// Layer 3: ProblemDetails middleware (for API endpoints)
app.UseStatusCodePages(); // Handles 404, etc.

// Layer 4: MVC Exception Filter (action-level)
builder.Services.Configure<MvcOptions>(options =>
{
    options.Filters.Add<GlobalExceptionFilter>(); // Custom global filter
});

// GlobalExceptionFilter:
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger _logger;
    
    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, "Unhandled exception");
        
        if (context.Exception is EntityNotFoundException ex)
        {
            context.Result = new NotFoundObjectResult(new { message = ex.Message });
            context.ExceptionHandled = true;
            return;
        }
        
        if (context.Exception is ValidationException vex)
        {
            context.Result = new UnprocessableEntityObjectResult(vex.Errors);
            context.ExceptionHandled = true;
        }
        // Other exception types fall through to middleware handler
    }
}
```

---

# MODULE 26: COMPLETE VIVA / INTERVIEW QUESTION BANK

## Section A: Conceptual Depth Questions

**Q1: Explain the complete lifecycle of an HTTP request in ASP.NET Core MVC from the moment it arrives at the server.**

**Answer:** (Refer to Module 3, but summarize as):
TCP connection → Kestrel HTTP parsing → HttpContext creation (from pool) → Per-request IServiceScope created → Middleware pipeline executed in order → RoutingMiddleware matches endpoint using DFA → AuthenticationMiddleware sets HttpContext.User → AuthorizationMiddleware enforces policies → EndpointMiddleware calls RequestDelegate → MVC: ActionContext built → ControllerFactory creates controller via ActivatorUtilities → ParameterBinder runs model binding (ValueProviders + ModelBinders) → IObjectModelValidator validates → Filter pipeline executes (Authorization → Resource → Action → Result) → Action method executes via ObjectMethodExecutor (compiled delegate) → IActionResult returned → Result filters → Result.ExecuteResultAsync() → For ViewResult: ViewEngine.FindView → Razor compilation or cache hit → RazorPage.ExecuteAsync writes to StringWriter → Layout rendered wrapping body → Flushed to Response.Body → IServiceScope.Dispose() → Controller released.

---

**Q2: How does ASP.NET Core avoid reflection overhead per request for action invocation?**

**Answer:** `ObjectMethodExecutor` compiles an expression tree from the action's `MethodInfo` at startup. The compiled delegate is cached per action descriptor. At runtime, instead of `MethodInfo.Invoke()` (which uses reflection), the compiled delegate is called directly. Similarly, `ActivatorUtilities.CreateFactory()` compiles the controller constructor call. The result is near-native invocation speed per request.

---

**Q3: What is the difference between IActionFilter and IAsyncActionFilter, and which should you prefer?**

**Answer:** `IActionFilter` has `OnActionExecuting` and `OnActionExecuted` — two separate methods with no control over the continuation between them. `IAsyncActionFilter` has a single `OnActionExecutionAsync(context, next)` where YOU call `next()` to execute the action. Prefer `IAsyncActionFilter` when:
- You need to wrap action execution with try/catch
- You need to measure time around the action
- You need to conditionally skip action execution
- You want to handle both before/after logic with shared variables

`IActionFilter` is fine for simple pre/post concerns with no shared state.

---

**Q4: How does TempData survive a redirect, and what happens to it after being read?**

**Answer:** TempData is persisted by `ITempDataProvider`. The default `CookieTempDataProvider` serializes the dictionary to JSON, encrypts with Data Protection, and stores it in a cookie. On redirect, the browser sends this cookie with the next request. `TempDataDictionary.Load()` deserializes it. When `TempData["key"]` is read, the key is marked for deletion. After the response completes, `TempData.Save()` is called, which writes back the dictionary minus the marked-for-deletion keys. `TempData.Keep("key")` removes the deletion mark. `TempData.Peek("key")` reads without marking.

---

**Q5: Explain captive dependency problem with a concrete MVC example.**

**Answer:**
```csharp
// PROBLEM:
public class EmailService : IEmailService  // Registered as Singleton
{
    private readonly IUserContext _userContext; // Registered as Scoped
    
    public EmailService(IUserContext userContext)
    {
        _userContext = userContext; // Captured at singleton creation time
        // _userContext is now effectively singleton
        // First user's context leaks into all subsequent requests
    }
}

// FIX:
public class EmailService : IEmailService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public EmailService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task SendAsync(string message)
    {
        using var scope = _scopeFactory.CreateScope();
        var userContext = scope.ServiceProvider.GetRequiredService<IUserContext>();
        // Fresh scoped service per operation
    }
}
```

---

**Q6: Why does [ApiController] cause model validation to return 400 before the action executes?**

**Answer:** `ApiBehaviorApplicationModelProvider` (registered in `AddApiExplorer()` chain) detects `[ApiController]` attribute at startup and adds `ModelStateInvalidFilter` to the action's filter collection with `Order = -2000` (extremely early). This runs as an Action Filter before the action method. Inside, it checks `ActionContext.ModelState.IsValid`. If false, it creates `ValidationProblemDetails` using `ProblemDetailsFactory` and sets `context.Result = new BadRequestObjectResult(problemDetails)`, short-circuiting the pipeline before the action runs.

---

**Q7: How does the DFA route matcher work internally?**

**Answer:** The Deterministic Finite Automaton (DFA) matcher is built at startup by `DfaMatcherBuilder`. It processes all registered route templates and builds a decision tree. Each node in the tree represents a URL segment position. Literal segments create exact-match transitions. Parameter segments create wildcard transitions. At request time, the URL is split into segments and walked through the DFA in O(n) time where n is segment count — NOT O(routes). Constraint checking happens after DFA filtering. This means adding more routes doesn't linearly slow down routing — it remains fast because the DFA is pre-compiled.

---

**Q8: Explain how Razor prevents XSS by default.**

**Answer:** In Razor, `@expression` calls `Write(value)` which internally uses `HtmlEncoder.Default.Encode(value.ToString())`. This HTML-encodes characters like `<`, `>`, `"`, `&`, turning them into `&lt;`, `&gt;`, `&quot;`, `&amp;`. An attacker's `<script>alert('xss')</script>` becomes `&lt;script&gt;...&lt;/script&gt;` — displayed as text, not executed. `@Html.Raw(value)` bypasses encoding — only use for trusted content. `WriteLiteral()` (used for literal HTML in .cshtml) also bypasses encoding — but it's only called for your static HTML, never for dynamic data.

---

**Q9: What is the difference between UseExceptionHandler middleware and IExceptionFilter?**

**Answer:**
- `UseExceptionHandler`: runs at middleware level, catches ALL exceptions from the entire pipeline below it. Has access only to `HttpContext` and `Exception`. Typically re-executes a different endpoint (`/Error`). Cannot set an `IActionResult` — must write to `HttpContext.Response` directly or via re-execution.
- `IExceptionFilter`: runs inside MVC filter pipeline. Catches exceptions from Action Filters, Action execution, and Result Filter execution (NOT from Authorization filters or outside MVC). Can set `context.Result = new IActionResult` which goes through Result Filters and normal result execution. Has access to `ActionDescriptor`, `Controller`, `RouteData`.
- Use exception filter for domain-specific MVC error handling (entity not found → 404). Use middleware handler as global catch-all safety net.

---

**Q10: Explain how session is loaded and what "lazy loading" means in context of ISession.**

**Answer:** When `UseSession()` adds `SessionMiddleware`, it doesn't immediately load session data. It creates a `DistributedSession` object and registers it via `ISessionFeature`. This object has `_isAvailable = false` and data is null initially. When code first calls `HttpContext.Session.GetString("key")`, `DistributedSession.LoadAsync()` is triggered — it calls `IDistributedCache.GetAsync(sessionId)` to fetch the data from the backing store. This is lazy loading: the network/storage call only happens when session data is actually needed. If your request never reads session, no network call is made. On response completion, if session was modified, `CommitAsync()` serializes and writes back to the cache.

---

## Section B: Interview Trap Questions

**Trap Q1: "Can you inject a scoped service into a singleton using constructor injection in ASP.NET Core?"**

**Trap:** Many say "no." 
**Nuanced answer:** In Development, ASP.NET Core will throw an `InvalidOperationException` at startup because it validates the service graph (scope validation is enabled). In Production (or when validation is disabled), it technically WORKS but gives wrong behavior — the scoped service becomes effectively singleton (captured in singleton's constructor). Always verify: does your production environment have scope validation enabled?

```csharp
// To disable validation (not recommended):
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = false; // Dangerous
});
```

---

**Trap Q2: "Does ViewResult write directly to the response stream?"**

**Trap:** Many say "yes."
**Reality:** No. `RazorView.RenderAsync()` first writes to a `StringWriter` (memory). Only after the full layout is rendered (which needs to wrap the view body) does it flush to `Response.Body`. This is necessary because the layout can add content AFTER the body (footer sections), and the layout renders sections defined in the view.

---

**Trap Q3: "Are filters singletons?"**

**Trap:** This has no single answer.
**Reality:** It depends on how they're applied:
- Attribute-based filters (directly on controller/action): One instance per attribute application. The SAME instance is reused per request (because attributes are class-level metadata, created once). This means you CANNOT inject scoped services via constructor.
- `[ServiceFilter(typeof(MyFilter))]`: MyFilter resolved from DI per request (or per scope). Supports scoped injection.
- `[TypeFilter(typeof(MyFilter))]`: New instance per request via `ActivatorUtilities`. Supports DI + extra args.
- Global filters added via `MvcOptions.Filters.Add(new MyFilter())`: The instance you add is reused (singleton-like).
- Global filters added via `MvcOptions.Filters.AddService<MyFilter>()`: Resolved per request.

---

**Trap Q4: "Does async action method mean the request is handled on multiple threads?"**

**Trap:** Many say "yes, it switches threads."
**Reality:** It MIGHT switch threads at an `await` point (the continuation may resume on a different thread pool thread), but there's no explicit multi-threading. The request is handled sequentially — just potentially on different threads at different points. The `HttpContext` is accessible throughout because it flows via `ExecutionContext` (which `AsyncLocal` uses), not thread-local storage. No concurrent thread access occurs in a normal async action.

---

**Trap Q5: "When is the controller disposed?"**

**Trap:** Many say "when the request ends" or "never."
**Reality:** `IControllerFactory.ReleaseController(context, controller)` is called after result execution. `DefaultControllerFactory` calls `IControllerActivator.Release()`. `DefaultControllerActivator.Release()` checks if the controller implements `IDisposable` — if yes, calls `Dispose()`. This happens BEFORE the request IServiceScope is disposed. If the controller needs scoped service cleanup, that's handled by the scope disposal separately.

---

**Trap Q6: "Does UseRouting() need to be called before UseAuthentication()?"**

**Trap:** Many say it doesn't matter.
**Reality:** Yes, `UseRouting()` must come BEFORE `UseAuthentication()` and `UseAuthorization()`. Authentication middleware needs to know the endpoint metadata (e.g., which authentication schemes are required) to make decisions. `UseRouting()` populates `IEndpointFeature` so that `UseAuthorization()` can read `[Authorize]` attributes from the matched endpoint. Wrong order = authorization runs without endpoint context = no policy-based decisions.

Correct order:
```csharp
app.UseRouting();           // 1. Match endpoint
app.UseCors();              // 2. CORS needs endpoint info
app.UseAuthentication();    // 3. Who is this user?
app.UseAuthorization();     // 4. What can they do?
app.MapControllers();       // 5. Execute
```

---

## Section C: "What Actually Happens When..." Questions

**"What actually happens when you call `return RedirectToAction("Index")`?"**

```
Controller.RedirectToAction("Index") is called
  ↓
Returns RedirectToActionResult(actionName: "Index", 
                               controllerName: null, 
                               routeValues: null, 
                               permanent: false)
  ↓
Filter pipeline: result filters run
  ↓
RedirectToActionResult.ExecuteResultAsync(context):
  ↓
  IUrlHelper.Action("Index", null, null)
    → reads current controller from RouteData
    → generates URL using route table
  ↓
  HttpContext.Response.StatusCode = 302
  HttpContext.Response.Headers["Location"] = generatedUrl
  ↓
Response sent to browser
  ↓
Browser follows Location header (new GET request)
```

---

**"What actually happens when you call `View()` with no arguments?"**

```
Controller.View() → View(viewName: null, model: null)
  ↓
ViewResult created with:
  ViewName = null
  Model = null (ViewData.Model remains null)
  ↓
ViewResultExecutor.ExecuteAsync():
  ↓
  viewName = null → use ActionContext.ActionDescriptor.ActionName
  (e.g., "Index" for Index action)
  ↓
  IViewEngine.FindView(context, "Index", isMainPage: true)
    → Search /Views/Products/Index.cshtml → found
  ↓
  ViewResultExecutor sets response:
    StatusCode = 200 (if not already set)
    Content-Type = "text/html; charset=utf-8"
  ↓
  IView.RenderAsync(viewContext) → Razor rendering pipeline
```

---

**"What actually happens when model binding fails for a required field?"**

```
Request: POST /products/create with no "Name" field in form body
  ↓
ParameterBinder.BindModelAsync() for CreateProductViewModel:
  ↓
  ComplexObjectModelBinder attempts to bind each property
  ↓
  For "Name" property:
    IValueProvider.ContainsPrefix("Name") → false (not in form)
    → No value available
    → bindingContext.Result = ModelBindingResult.Failed()
  ↓
  DataAnnotations validation runs:
    [Required] on Name property → value is null → fails
    ModelState.AddModelError("Name", "The Name field is required.")
  ↓
ModelState.IsValid = false
  ↓
  For [ApiController]:
    ModelStateInvalidFilter fires BEFORE action
    → context.Result = BadRequestObjectResult(ValidationProblemDetails)
    → action method never called
  
  For plain MVC Controller:
    Action method IS called
    You check: if (!ModelState.IsValid) return View(model);
    Razor renders view with asp-validation-for tag helpers showing errors
```

---

**"What actually happens on first Razor view render (cold start)?"**

```
First request for /Products/Index (build-time compilation disabled):
  ↓
ViewResultExecutor: FindView("Index")
  ↓
RazorPageFactoryProvider: check CompiledPageCache → MISS
  ↓
IRazorPageLoader: load from disk
  ↓
RazorProjectEngine.Process("/Views/Products/Index.cshtml"):
  1. Read .cshtml source from IRazorProjectFileSystem
  2. Parse: RazorSyntaxTree built
  3. Lower: C# code generation
  4. RazorCSharpDocument contains generated C# source
  ↓
Roslyn CSharpCompilation:
  - Parse generated C# into SyntaxTree
  - Reference all loaded assemblies (including your project's)
  - Emit to MemoryStream → Assembly bytes
  ↓
Assembly.Load(bytes) → loaded into AppDomain
  ↓
Compiled page type extracted via reflection
  ↓
Stored in CompiledPageCache (keyed by path + file content hash)
  ↓
RazorPage instance created → ExecuteAsync() runs → HTML generated
  
Second request for same view:
  CompiledPageCache HIT → compiled type retrieved
  RazorPage instance created → ExecuteAsync() runs
  (No disk read, no Roslyn, no reflection after first load)
```

---

# MODULE 27: END-TO-END REQUEST LIFECYCLE — ULTRA DEEP DIVE

## Complete Journey: `GET /Products/Edit/42` in an MVC Application

### Phase 1: Transport Layer (Kestrel)

```
[1.1] TCP SYN received by OS → passed to Kestrel's socket acceptor
      Kestrel runs on I/O threads (configured via KestrelServerOptions.Limits)

[1.2] Kestrel's Http1Connection (or Http2Connection) parses bytes:
      - Method: GET
      - Path: /Products/Edit/42
      - Headers: Host, Accept, Cookie, etc.
      HTTP headers stored in HeaderDictionary (pooled, zero-allocation where possible)

[1.3] Kestrel creates/retrieves DefaultHttpContext from ObjectPool<DefaultHttpContext>
      Initializes:
        - HttpContext.Request (DefaultHttpRequest)
        - HttpContext.Response (DefaultHttpResponse) 
        - HttpContext.Features collection (cleared)
        - HttpContext.Connection (IP, port)

[1.4] Kestrel calls IHttpApplication<HostingApplication.Context>.ProcessRequestAsync()
```

### Phase 2: Hosting Layer

```
[2.1] HostingApplication.ProcessRequestAsync():
      - Creates IServiceScope from root IServiceProvider
        scope = rootServiceProvider.CreateScope()
      - Sets HttpContext.RequestServices = scope.ServiceProvider
        (All subsequent DI resolutions use this scoped provider)
      - Captures ILogger, DiagnosticListener for request events
      - Records request start time for logging

[2.2] Calls into IApplicationBuilder's compiled pipeline
      The pipeline is a delegate chain compiled once at startup:
      Delegate pipeline = middleware1(middleware2(middleware3(...(endpoint_middleware))))
```

### Phase 3: Middleware Pipeline

```
[3.1] ExceptionHandlerMiddleware / DeveloperExceptionPageMiddleware:
      Wraps the rest in try/catch
      Calls await _next(context)

[3.2] HSTSMiddleware:
      Adds Strict-Transport-Security header
      Calls await _next(context)

[3.3] HttpsRedirectionMiddleware:
      If HTTP and configured: 301/302 to HTTPS
      (Assuming HTTPS: passes through)
      Calls await _next(context)

[3.4] StaticFilesMiddleware:
      Checks if path matches a static file
      /Products/Edit/42 → not a static file
      Calls await _next(context)

[3.5] RoutingMiddleware (EndpointRoutingMiddleware):
      Calls IEndpointSelector.SelectAsync(candidates, context)
      
      DfaMatcher.FindCandidatesAsync("/Products/Edit/42"):
        Segment 1: "Products" → matches {controller} AND "Products" literal routes
        Segment 2: "Edit" → matches {action} AND "Edit" literal routes  
        Segment 3: "42" → matches {id?} parameter
        Candidates: [{controller=Products, action=Edit, id=42}]
      
      For each candidate: check route constraints
        {id} with :int constraint? → "42" is valid int → passes
      
      Selected endpoint: ControllerActionEndpoint for ProductsController.Edit(int id)
      
      context.Features.Set<IEndpointFeature>(new EndpointFeature { Endpoint = ep })
      context.Features.Set<IRouteValuesFeature>(new RouteValueFeature { 
          RouteValues = { controller="Products", action="Edit", id="42" } 
      })
      
      Calls await _next(context)

[3.6] CorsMiddleware (if enabled):
      Reads endpoint CORS metadata
      GET request (simple request): passes through
      Calls await _next(context)

[3.7] AuthenticationMiddleware:
      IAuthenticationSchemeProvider.GetDefaultAuthenticateSchemeAsync()
        → Returns "Cookies" (configured scheme)
      
      CookieAuthenticationHandler.AuthenticateAsync():
        - Reads ".AspNetCore.Auth" cookie from request
        - IDataProtector.Unprotect(cookieValue) → decrypts AuthTicket bytes
        - Deserializes ClaimsPrincipal (User, Claims)
        - Checks ticket expiry and validity
        - Returns AuthenticateResult.Success(ticket)
      
      HttpContext.User = ticket.Principal
      (Now: HttpContext.User.Identity.IsAuthenticated = true)
      Calls await _next(context)

[3.8] AuthorizationMiddleware:
      Reads endpoint metadata for IAuthorizeData
      ProductsController has [Authorize] attribute → policy: default
      
      IAuthorizationService.AuthorizeAsync(HttpContext.User, resource, policy):
        - RequireAuthenticatedUserRequirement: HttpContext.User.IsAuthenticated = true → passes
        - Any role/claim requirements → evaluated
        → AuthorizationResult.Success()
      
      Access granted. Calls await _next(context)
      (If denied: IAuthorizationMiddlewareResultHandler writes 403 → pipeline stops)

[3.9] EndpointMiddleware:
      Reads IEndpointFeature → gets endpoint
      var requestDelegate = endpoint.RequestDelegate
      await requestDelegate(context)  ← ENTERS MVC
```

### Phase 4: MVC Infrastructure

```
[4.1] RequestDelegate (MVC closure) executes:
      - Gets RouteData from IRoutingFeature
      - Gets ActionDescriptor from endpoint.Metadata
        ActionDescriptor = ControllerActionDescriptor for ProductsController.Edit
      - Creates ActionContext { HttpContext, RouteData, ActionDescriptor }

[4.2] IActionInvokerFactory.CreateInvoker(actionContext):
      ControllerActionInvokerProvider.OnProvidersExecuting():
        
        a. FilterFactory.GetAllFilters(actionContext):
           - Global filters (from MvcOptions)
           - Controller-level: [Authorize] (already handled), [ValidateAntiForgeryToken]?
           - Action-level: none additional
           - Merge → OrderedFilters[]
           
           Filter instances created:
           - Static filters (attribute-based): reused instances
           - ServiceFilter filters: resolved from context.HttpContext.RequestServices
        
        b. ControllerActionInvokerCacheEntry retrieved:
           - Contains ObjectMethodExecutor for Edit(int id)
           - Contains ActionMethodExecutor (SyncActionResultExecutor for non-async)
        
        c. Returns ControllerActionInvoker(actionContext, cacheEntry, filters)
```

### Phase 5: Controller Activation

```
[5.1] IControllerFactory.CreateController(controllerContext):
      DefaultControllerFactory → DefaultControllerActivator.Create():
      
      ObjectFactory factory = cache["ProductsController"]  // cached at startup
      // factory = compiled delegate: (sp, args) => new ProductsController(...)
      
      factory(HttpContext.RequestServices, Array.Empty<object>()):
        → IServiceProvider.GetRequiredService<IProductService>()
            Returns: scoped ProductService instance (created for THIS request)
        → IServiceProvider.GetRequiredService<ILogger<ProductsController>>()
            Returns: singleton ILogger (from logger factory)
        → new ProductsController(productService, logger)
      
      Controller instance created on heap
      controller.ControllerContext = controllerContext (ActionContext + metadata)
      controller.MetadataProvider = (from DI, singleton)
      controller.ModelStateDictionary = new ModelStateDictionary()
      controller.TempData = tempDataDictionaryFactory.GetTempData(HttpContext)
        (TempData loaded from CookieTempDataProvider → decrypted from cookie → deserialized)
      controller.ViewData = new ViewDataDictionary<object>(metadataProvider, modelState)
      controller.Url = IUrlHelper (from IUrlHelperFactory, created per request)
```

### Phase 6: Filter Pipeline — Authorization

```
[6.1] ControllerActionInvoker.InvokeFilterPipelineAsync():
      State machine begins.
      
      State: InvokeNextAuthorizationFilterAsync
      
      AuthorizationFilter (if any custom IAuthorizationFilter added):
        context = AuthorizationFilterContext { ActionDescriptor, HttpContext, Filters, RouteData }
        filter.OnAuthorization(context)
        If context.Result set → short circuit to result execution
        
      [In our case: Authorization handled by middleware, no additional filter]
      State machine advances.
```

### Phase 7: Resource Filters

```
[7.1] State: InvokeNextResourceFilter
      IResourceFilter.OnResourceExecuting(context)
      
      [Example: OutputCacheFilter]
      Check if cached response exists for this request:
        Cache key = RouteData + Query string
        Cache MISS → continue
      
      If cache HIT:
        context.Result = CachedResult
        → Short circuit to result execution (skips model binding, action)
      
      [In our case: no resource filter / cache miss]
      State machine advances.
```

### Phase 8: Model Binding

```
[8.1] ParameterBinder.BindModelAsync() for parameter: int id
      
      IModelBinder selection:
        Parameter: int id (no attributes) 
        No explicit [From...] attribute
        Conventional MVC: check route first, then query, then form
        
        ModelBinderFactory checks ModelMetadata for int:
          → SimpleTypeModelBinder
      
      Value providers assembled:
        RouteValueProvider: { controller="Products", action="Edit", id="42" }
        QueryStringValueProvider: {} (no query string)
        FormValueProvider: N/A (GET request)
      
      SimpleTypeModelBinder.BindModelAsync():
        CompositeValueProvider.GetValue("id")
          → RouteValueProvider.GetValue("id") → ValueProviderResult("42")
        
        TypeConverter<int>.ConvertFrom("42") → 42 (int)
        
        bindingContext.Result = ModelBindingResult.Success(42)
      
      Validation for int:
        No [Required] on value type
        ModelState["id"] = Valid
      
      ModelState.IsValid = true
```

### Phase 9: Action Filters (Before)

```
[9.1] State: InvokeNextActionFilter
      
      ActionExecutingContext created:
        Controller = ProductsController instance
        ActionArguments = { "id": 42 }
        HttpContext, RouteData, ActionDescriptor
      
      IActionFilter.OnActionExecuting(context):
        [Example: LoggingActionFilter]
        _logger.LogInformation("Action {Action} executing", context.ActionDescriptor.DisplayName)
        
      If context.Result set → skip action, go to result execution
      
      [No short circuit in our case]
```

### Phase 10: Action Execution

```
[10.1] ControllerActionInvoker invokes action via ObjectMethodExecutor:
       
       compiledDelegate(controller, new object[] { 42 })
       → ProductsController.Edit(42) executes
       
       Inside Edit(int id):
         // Example implementation:
         var product = await _productService.GetByIdAsync(id);
         
         [await: thread returns to pool while DB query runs]
         [DB query completes: continuation scheduled on thread pool]
         [New thread resumes execution]
         
         if (product == null) return NotFound();
         
         var viewModel = new ProductEditViewModel
         {
             Id = product.Id,
             Name = product.Name,
             Price = product.Price
         };
         
         return View(viewModel);  // Returns ViewResult
       
       ObjectMethodExecutor awaits the Task<IActionResult> (if async)
       Returns IActionResult (ViewResult) to invoker
```

### Phase 11: Action Filters (After)

```
[11.1] ActionExecutedContext created:
       Result = ViewResult (from action return)
       Canceled = false, Exception = null
       
       IActionFilter.OnActionExecuted(context):
         [Example: LoggingActionFilter]
         _logger.LogInformation("Action completed, result: {Result}", context.Result)
       
       Filters can replace context.Result here
       Filters can set context.Exception (re-throw or suppress)
```

### Phase 12: Exception Filters

```
[12.1] If exception was thrown during [9]-[11]:
       IExceptionFilter.OnException(context):
         context.Exception = the thrown exception
         context.ExceptionHandled = false initially
         
         filter can:
           Set context.Result = new ObjectResult(errorDto) { StatusCode = 404 }
           Set context.ExceptionHandled = true → exception swallowed
         
         If not handled → propagates up to middleware ExceptionHandler
```

### Phase 13: Result Filters (Before)

```
[13.1] ResultExecutingContext created:
       Result = ViewResult
       Controller = ProductsController instance
       
       IResultFilter.OnResultExecuting(context):
         Can replace context.Result
         Can set context.Cancel = true → skip result execution
```

### Phase 14: Result Execution

```
[14.1] ViewResult.ExecuteResultAsync(actionContext):
       → Delegates to ViewResultExecutor.ExecuteAsync(actionContext, viewResult)
       
       ViewResultExecutor:
         1. viewName = viewResult.ViewName ?? actionContext.ActionDescriptor.ActionName
                     = null ?? "Edit" = "Edit"
         
         2. IViewEngine.FindView(context, "Edit", isMainPage: true):
            Try: /Views/Products/Edit.cshtml → exists? YES
            Return: ViewEngineResult.Found("Edit", RazorView)
         
         3. Set response status:
            HttpContext.Response.StatusCode = 200 (if not set)
            HttpContext.Response.ContentType = "text/html; charset=utf-8"
         
         4. Create ViewContext:
            ViewContext {
              HttpContext = context.HttpContext,
              Writer = null (will be set),
              ViewData = controller.ViewData (with Model = ProductEditViewModel),
              TempData = controller.TempData,
              FormContext = new FormContext(),
              ...
            }
         
         5. IView.RenderAsync(viewContext):
```

### Phase 15: Razor Rendering

```
[15.1] RazorView.RenderAsync(viewContext):
       
       a. Execute _ViewStart chain:
          /Views/_ViewStart.cshtml found:
            @{ Layout = "_Layout"; }
          Sets viewContext.Layout = "_Layout"
       
       b. Execute main view (ProductEdit.cshtml):
          bodyWriter = new StringWriter() ← in-memory buffer
          viewContext.Writer = bodyWriter
          
          RazorPage<ProductEditViewModel>.ExecuteAsync():
            WriteLiteral("<div class=\"container\">");
            WriteLiteral("<h2>Edit Product</h2>");
            WriteLiteral("<form method=\"post\" action=\"");
            Write(Url.Action("Save", "Products")); // HTML-encoded URL
            WriteLiteral("\">");
            
            // Anti-forgery token (from Form Tag Helper):
            IAntiforgery.GetAndStoreTokens(HttpContext):
              → Generate/read cookie token
              → Generate field token
              → Set cookie (if new)
            WriteLiteral("<input type=\"hidden\" name=\"__RequestVerificationToken\" value=\"");
            Write(fieldToken); // the anti-forgery token
            WriteLiteral("\">");
            
            // Tag Helper: <input asp-for="Name" />
            InputTagHelper.Process():
              → Reads ModelMetadata for Name property
              → Generates: <input type="text" id="Name" name="Name" value="Product X">
            
            WriteLiteral("</form></div>");
            
            // Define sections:
            DefineSection("scripts", async () => {
                WriteLiteral("<script src=\"/js/product-edit.js\"></script>");
            });
          
          bodyContent = bodyWriter.ToString() ← body HTML captured
       
       c. Layout rendering:
          If Layout set ("_Layout"):
            layoutView = FindView(context, "_Layout", isMainPage: false)
            
            layoutWriter = new StreamWriter(HttpContext.Response.Body)
            viewContext.Writer = layoutWriter
            
            Execute _Layout.cshtml:
              WriteLiteral("<!DOCTYPE html><html><head>...");
              WriteLiteral("<title>");
              Write(ViewBag.Title ?? "Default Title");
              WriteLiteral("</title></head><body>");
              
              @RenderBody() → flushes bodyContent to layoutWriter
              
              WriteLiteral("<footer>...</footer>");
              
              @await RenderSectionAsync("scripts", required: false)
                → executes DefineSection("scripts") delegate
                → WriteLiteral("<script src=\"/js/product-edit.js\"></script>");
              
              WriteLiteral("</body></html>");
            
            layoutWriter.Flush() ← writes to Response.Body stream
```

### Phase 16: Result Filters (After)

```
[16.1] IResultFilter.OnResultExecuted(context):
       Result has been executed
       Can inspect but cannot change the result (response already written)
       Typically used for logging, cleanup
```

### Phase 17: Resource Filters (After)

```
[17.1] IResourceFilter.OnResourceExecuted(context):
       [Example: OutputCacheFilter]
         Store the response in cache for future requests
```

### Phase 18: Response Transmission

```
[18.1] Response.Body (Kestrel's PipeWriter) flushed
       Kestrel chunks the HTML into HTTP response:
         HTTP/1.1 200 OK
         Content-Type: text/html; charset=utf-8
         Content-Length: 4821
         Set-Cookie: .AspNetCore.Auth=...; Path=/; HttpOnly; Secure; SameSite=Lax
         Set-Cookie: __RequestVerificationToken=...; Path=/; SameSite=Strict
         
         [HTML body]

[18.2] Kestrel sends bytes over TCP to client
```

### Phase 19: Cleanup

```
[19.1] IControllerFactory.ReleaseController(context, controller):
       DefaultControllerActivator.Release():
         controller implements IDisposable? → Dispose() called
         controller = null (eligible for GC)

[19.2] IServiceScope.Dispose():
       All Scoped services registered in this scope are disposed:
         ProductService → if IDisposable → Dispose()
         DbContext (if scoped) → Dispose() → closes DB connection
         Any other scoped IDisposable → Dispose()

[19.3] HttpContext returned to ObjectPool<DefaultHttpContext>:
       State reset for reuse in next request

[19.4] TempData cleanup:
       TempData.Save() called:
         Keys read during this request removed
         Remaining keys serialized
         If empty: delete TempData cookie
         If not empty: update cookie (with remaining keys)

[19.5] Request logging:
       ILogger writes request completion log:
         "HTTP GET /Products/Edit/42 responded 200 in 45.3ms"
```

---

# MODULE 28: PERFORMANCE BOTTLENECKS AND PRODUCTION OPTIMIZATION

## 28.1 Top Performance Bottlenecks

**1. Synchronous database calls in actions**
```csharp
// BAD — blocks thread
var products = _db.Products.ToList();

// GOOD — async I/O
var products = await _db.Products.ToListAsync();
```

**2. N+1 query problems**
```csharp
// BAD:
var orders = await _db.Orders.ToListAsync();
foreach (var o in orders)
    o.Customer = await _db.Customers.FindAsync(o.CustomerId); // N queries

// GOOD:
var orders = await _db.Orders.Include(o => o.Customer).ToListAsync(); // 1 query
```

**3. ViewBag for large datasets**
```csharp
// BAD:
ViewBag.AllCategories = _db.Categories.ToList(); // thousands of rows
// These live in memory until view renders

// GOOD: Use typed ViewModel with lazy/paged loading
```

**4. Missing output caching for read-heavy views**
```csharp
// GOOD: Cache rendered views
[OutputCache(Duration = 60, VaryByQueryKeys = new[] { "page" })]
public async Task<IActionResult> Catalog() { }
```

**5. Synchronous file reads in views**
```cshtml
// BAD in view:
@System.IO.File.ReadAllText("/config/something.json")
// Blocking I/O on rendering thread

// GOOD: Read in controller, pass in ViewModel
```

**6. Large response bodies without compression**
```csharp
// Add response compression:
builder.Services.AddResponseCompression(opts =>
{
    opts.EnableForHttps = true;
    opts.Providers.Add<BrotliCompressionProvider>();
    opts.Providers.Add<GzipCompressionProvider>();
});
app.UseResponseCompression(); // Before static files middleware
```

## 28.2 Hidden Framework Optimizations You Should Know

**1. Route DFA is O(segments), not O(routes)**
Adding 1000 routes doesn't slow routing proportionally. DFA lookup is O(URL segment count).

**2. ObjectMethodExecutor eliminates reflection per request**
Action invocation is via compiled delegate, not `MethodInfo.Invoke()`.

**3. ModelMetadata is cached indefinitely**
First request builds metadata for a type; subsequent requests reuse it.

**4. Compiled Razor views skip Roslyn on subsequent requests**
After first compilation, only the `ExecuteAsync()` delegate runs.

**5. HttpContext pooling**
`DefaultHttpContext` is pooled — allocation only on pool exhaustion.

**6. Filter static caching**
Static filters (non-DI) are identified and reused across requests without recreation.

---

# MODULE 29: ADVANCED DEBUGGING REFERENCE

## 29.1 Enabling Detailed MVC Logging

```csharp
// appsettings.Development.json:
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore.Mvc": "Debug",
      "Microsoft.AspNetCore.Routing": "Debug",
      "Microsoft.AspNetCore.Authentication": "Debug",
      "Microsoft.AspNetCore.Authorization": "Debug"
    }
  }
}
```

This reveals:
- Route matching decisions and candidates
- Model binding attempts and failures
- Filter execution order
- View search paths
- Authentication results

## 29.2 Inspecting ActionDescriptor at Runtime

```csharp
// In any action:
public IActionResult Debug()
{
    var provider = HttpContext.RequestServices
        .GetRequiredService<IActionDescriptorCollectionProvider>();
    
    var descriptors = provider.ActionDescriptors.Items
        .OfType<ControllerActionDescriptor>()
        .Select(d => new {
            d.ControllerName,
            d.ActionName,
            Filters = d.FilterDescriptors.Select(f => f.Filter.GetType().Name),
            d.AttributeRouteInfo?.Template,
            d.RouteValues
        });
    
    return Json(descriptors);
}
```

## 29.3 Diagnosing Slow Requests

```csharp
// Custom action filter for timing:
public class TimingFilter : IAsyncActionFilter
{
    private readonly ILogger<TimingFilter> _logger;
    
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, 
        ActionExecutionDelegate next)
    {
        var sw = Stopwatch.StartNew();
        var resultContext = await next();
        sw.Stop();
        
        _logger.LogWarning(
            "Action {Controller}.{Action} took {ElapsedMs}ms. " +
            "Exception: {Exception}",
            context.Controller.GetType().Name,
            context.ActionDescriptor.DisplayName,
            sw.ElapsedMilliseconds,
            resultContext.Exception?.Message);
    }
}

// Register globally:
services.Configure<MvcOptions>(o => o.Filters.Add<TimingFilter>());
```

---

# APPENDIX: QUICK REFERENCE — INTERNAL COMPONENT MAP

```
STARTUP COMPONENTS:
  ApplicationPartManager        → discovers controller assemblies
  ApplicationModelFactory       → builds startup-time model
  ControllerActionDescriptorProvider → builds ActionDescriptors
  DfaMatcherBuilder             → compiles route DFA
  ObjectMethodExecutorCache     → compiles action delegates
  ModelMetadataCache            → caches type metadata

REQUEST COMPONENTS:
  HostingApplication            → creates IServiceScope per request
  EndpointRoutingMiddleware     → runs DFA matcher
  EndpointMiddleware            → invokes RequestDelegate
  ControllerActionInvoker       → orchestrates filter + action pipeline
  DefaultControllerFactory      → creates/releases controller
  DefaultControllerActivator    → uses ActivatorUtilities
  ParameterBinder               → model binding per parameter
  DefaultModelBindingContext    → holds binding state
  CompositeValueProvider        → aggregates Route + Query + Form providers
  IObjectModelValidator         → runs DataAnnotation validation
  FilterFactory                 → resolves filter instances
  ObjectMethodExecutor          → invokes compiled action delegate

RESULT EXECUTION:
  ViewResultExecutor            → finds view, sets up ViewContext
  RazorViewEngine               → view discovery + compilation
  RazorView                     → orchestrates _ViewStart + view + layout
  RazorPage<TModel>             → compiled view class
  ITagHelperActivator           → creates tag helper instances
  IAntiforgery                  → generates/validates CSRF tokens
  ITempDataProvider             → serializes/deserializes TempData
  IDataProtector                → encrypts cookies, tokens, TempData

STATE:
  HttpContext                   → pooled, per-request lifespan
  IServiceScope                 → per-request DI scope
  ViewDataDictionary            → Controller → View data channel
  ModelStateDictionary          → binding + validation errors
  RouteValueDictionary          → route data for the request
  TempDataDictionary            → cross-redirect data channel
```

---

*End of ASP.NET Core MVC Complete Internals Handbook — .NET 8/9 Architecture*
*Topics covered: 29 Modules | 40+ Internal Flow Diagrams | Complete E2E Lifecycle | Viva Bank*

# Lesson 1 — Orientation: From Zero to Your First API

**Goal:** Understand what a .NET Web API is, see how the two reference repos are laid out, and run your own minimal API in ~15 minutes.

---

## 1. What is a .NET Web API?

A **Web API** in .NET is an HTTP server that exposes endpoints (URLs) returning data — usually JSON. It's built on **ASP.NET Core**, the cross-platform web framework that ships with the .NET SDK.

The key building blocks:

| Concept            | What it is                                                          |
| ------------------ | ------------------------------------------------------------------- |
| **Project**        | A `.csproj` file + source code, compiled into a DLL or executable.  |
| **Solution**       | A `.sln` / `.slnx` file grouping multiple projects.                 |
| **Program.cs**     | Entry point. Configures services and HTTP pipeline.                 |
| **Endpoint**       | A route (`GET /todos/{id}`) bound to a handler (a method).          |
| **DI container**   | Registers services so they can be injected into handlers.           |

You already have everything: `dotnet --version` returned `10.0.203`.

---

## 2. The two reference repos, side by side

Open these in your editor as we go:

### `CleanArchitecture/src/` — single API, layered
```
src/
├── Domain/          ← Business entities & rules. NO dependencies on other layers.
├── Application/     ← Use cases (CQRS commands/queries). Depends on Domain only.
├── Infrastructure/  ← EF Core, identity, external services. Implements Application interfaces.
├── Web/             ← HTTP layer (controllers, minimal APIs). Composition root.
├── AppHost/         ← .NET Aspire orchestration (local dev runner).
├── ServiceDefaults/ ← Shared telemetry/health-check setup.
└── Shared/          ← DTOs shared with clients.
```

The arrows of dependency point **inward** toward `Domain`. That's the whole point of Clean Architecture: business rules don't know about databases or HTTP.

### `eShop/src/` — many APIs, microservices
```
src/
├── Catalog.API/     ← Product catalog service
├── Basket.API/      ← Shopping basket service
├── Ordering.API/    ← Orders (note: also has Ordering.Domain + Ordering.Infrastructure)
├── Identity.API/    ← Auth (IdentityServer/Duende)
├── Webhooks.API/    ← Webhook subscriptions
├── EventBus/        ← Abstractions for async messaging
├── EventBusRabbitMQ/← RabbitMQ implementation
├── eShop.AppHost/   ← Aspire orchestrator — runs ALL services together
├── WebApp/          ← Blazor frontend
└── ...
```

Notice eShop **only** applies the full layered split (`Domain` / `Infrastructure`) to `Ordering` — the most complex domain. The simpler services (`Catalog.API`, `Basket.API`) are **single-project APIs**. That's a real-world lesson: **layering has a cost; only pay it where complexity demands it.**

---

## 3. Both repos share a backbone: .NET Aspire

Look at `CleanArchitecture/src/AppHost/` and `eShop/src/eShop.AppHost/`. Both projects use **.NET Aspire**, Microsoft's local-dev orchestrator. It:

- Starts your APIs, databases (Postgres, Redis), message brokers (RabbitMQ), etc. with one `dotnet run`.
- Wires up service discovery so `Catalog.API` can call `http://basket-api` without hardcoded URLs.
- Provides a dashboard at `http://localhost:18888` showing logs/traces across services.

You don't need to learn Aspire today — just know that `AppHost` is where the *whole system* boots, and individual API projects are still runnable on their own.

---

## 4. Hands-on: your first API

We'll build a tiny "todo" minimal API. This mirrors how `Catalog.API` is structured — flat, single-project, with endpoints in `Program.cs`.

### Step 1 — Create the project

```bash
cd E:/1applications/CSharp-Learning/hands-on/lesson-01-hello-api
dotnet new webapi -n HelloApi --use-minimal-apis
cd HelloApi
```

`dotnet new webapi` scaffolds a working API. `--use-minimal-apis` skips the controller boilerplate.

### Step 2 — Replace `Program.cs`

Open `Program.cs` and replace its contents with:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

var todos = new List<Todo>
{
    new(1, "Learn .NET", false),
    new(2, "Read CleanArchitecture repo", false),
};

app.MapGet("/todos", () => todos);

app.MapGet("/todos/{id:int}", (int id) =>
    todos.FirstOrDefault(t => t.Id == id) is { } todo
        ? Results.Ok(todo)
        : Results.NotFound());

app.MapPost("/todos", (Todo todo) =>
{
    todos.Add(todo);
    return Results.Created($"/todos/{todo.Id}", todo);
});

app.Run();

record Todo(int Id, string Title, bool Done);
```

### Step 3 — Run it

```bash
dotnet run
```

You'll see something like `Now listening on: http://localhost:5xxx`.

### Step 4 — Try it

In another terminal:

```bash
curl http://localhost:5xxx/todos
curl http://localhost:5xxx/todos/1
curl -X POST http://localhost:5xxx/todos -H "Content-Type: application/json" -d "{\"id\":3,\"title\":\"Ship it\",\"done\":false}"
```

Stop the server with `Ctrl+C`.

---

## 5. What you just learned

- **Project anatomy.** `Program.cs` is the entry point. It configures a `WebApplication` and maps endpoints.
- **Minimal APIs.** Endpoints are lambdas. No controller class needed for simple cases.
- **Records.** `record Todo(...)` is a one-line immutable type — used heavily in both reference repos for DTOs.
- **`Results`.** The return type for endpoints. `Ok`, `NotFound`, `Created` are typed helpers that produce correct HTTP responses.

---

## 6. Compare what you wrote to the real thing

Open `eShop/src/Catalog.API/Program.cs` and `eShop/src/Catalog.API/Apis/CatalogApi.cs`. Notice:

- `Program.cs` is short — it just wires services and calls `app.MapCatalogApi()`.
- The actual endpoint definitions live in a separate file (`CatalogApi.cs`) as an **extension method**: `public static RouteGroupBuilder MapCatalogApi(this RouteGroupBuilder app)`.

This is the **next refactor** you'd do as your API grows past ~5 endpoints. We'll cover that in Lesson 2.

---

## 7. Checklist before Lesson 2

- [ ] You ran `HelloApi` and hit all three endpoints successfully.
- [ ] You opened `eShop/src/Catalog.API/Program.cs` and skimmed it.
- [ ] You opened `CleanArchitecture/src/Web/Program.cs` and skimmed it.
- [ ] You can name the four Clean Architecture layers without looking.

When you're ready, ask for **Lesson 2: Endpoint groups, DI, and the request pipeline**.
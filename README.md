# PFound.DependencyContainer

A lightweight constructor-injection DI container for Unity. Register services fluently, `Build()`
to wire the graph, then resolve by lifetime. Constructor arguments are injected by picking the
greediest constructor whose parameters are all registered; circular dependencies fail fast. Pure
C# — no engine reference, no `MonoBehaviour`, no scene presence.

## Model

- **Lifetimes** (`ServiceLifetime`): `Singleton` (one shared instance per container), `Scoped` (one
  per `IServiceScope`), `Transient` (a fresh instance per resolve).
- **Build phase.** `Register(...)` / `RegisterInstance(...)` mutate the graph; `Build()` locks it and
  eagerly constructs every singleton (running each `IInitializableService.Initialize()`). Registering
  after `Build()` throws.
- **Ownership.** The container tracks every `IDisposable` it *creates* and disposes them in reverse
  order on `Dispose()`; a pre-built instance passed to `RegisterInstance` is owned by the caller.

## Public API (`DependencyContainer`)

**Registration** (before `Build`): `Register<TImpl>()` → `ServiceBuilder<TImpl>` with
`.As<TService>()` (alias, shared instance for singletons), `.WithLifetime(ServiceLifetime)`,
`.WithFactory(c => …)`; `RegisterInstance<TService>(instance)`.

**Wiring**: `Build()` (returns the container for chaining).

**Resolution**: `Get<T>()`, `TryGet<T>(out T)`, `GetService(Type)` (`IServiceProvider`),
`CreateScope()` → `IServiceScope` (`Get<T>` / `TryGet<T>` / `Dispose`).

**Teardown**: `Dispose()`.

**Hooks**: implement `IInitializableService.Initialize()` to run one-time setup right after a
service is constructed and injected.

## Setup / wiring

Pure library — `new DependencyContainer()`, no scene object or lifecycle host. The consumer owns
the instance: build it once at startup, keep the reference (e.g. in your bootstrap object, or hand
it to other systems), and `Dispose()` it on shutdown.

```csharp
var container = new DependencyContainer();
container.Register<SaveSystem>().As<ISaveSystem>();
container.Register<AudioService>().WithLifetime(ServiceLifetime.Singleton);
container.RegisterInstance<IClock>(clock);          // caller-owned instance
container.Build();                                   // eager singletons + Initialize() hooks

var save = container.Get<ISaveSystem>();
using (var scope = container.CreateScope()) { var perScope = scope.Get<Foo>(); }
// on shutdown:
container.Dispose();
```

Main-thread only. `CreateScope()` throws before `Build()`. There is no static/singleton accessor —
if you want global reach, register the container (or a service locator over it) into your own
bootstrap seam.

## Layout

- `Runtime/` — `DependencyContainer` + `ServiceContracts` (`ServiceLifetime`,
  `IInitializableService`, `IServiceScope`). Assembly `PFound.DependencyContainer`
  (`noEngineReferences`, `autoReferenced:false`).
- `Tests/` — NUnit suite.

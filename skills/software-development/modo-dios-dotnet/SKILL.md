---
name: modo-dios-dotnet
description: "Use when working with .NET, C#, WPF, WinForms, ASP.NET Core, Entity Framework, MAUI, Blazor, or any .NET ecosystem project. Modo dios experto: código óptimo, patrones avanzados, zero bugs, rendimiento máximo."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [windows, linux, macos]
metadata:
  hermes:
    tags: [dotnet, csharp, wpf, aspnet, ef-core, maui, blazor, winforms, visual-studio, rider]
    related_skills: [writing-plans, test-driven-development, systematic-debugging, spike]
---

# Modo Dios .NET — C#, WPF, y Todo el Ecosistema

## Filosofía

Cuando esta skill está activa, el agente opera en **modo dios** para .NET:

- Código de máxima calidad, sin atajos.
- Patrones correctos desde el inicio — no "después lo arreglo".
- Conocimiento profundo del runtime, GC, JIT, y IL.
- Cero tolerancia a antipatrones.
- Explicaciones claras en español, código en inglés (convención estándar).

## Cuándo Usar

- **Siempre que se toque .NET** — C#, F#, VB.NET, cualquier proyecto del ecosistema.
- WPF, WinForms, MAUI, Blazor, ASP.NET Core, Entity Framework, SignalR, gRPC.
- Consultas sobre rendimiento, memoria, GC, async/await, threading.
- Migraciones entre versiones de .NET Framework → .NET Core → .NET 5+.
- Configuración de proyectos, MSBuild, NuGet, CI/CD para .NET.
- Debugging avanzado, dump analysis, WinDbg/SOS.

## Tabla de Versiones .NET

| Versión | LTS | Lanzamiento | Fin de Soporte | Runtime |
|:--------|:----|:------------|:---------------|:--------|
| .NET 10 | No | Nov 2025 (preview) | — | net10.0 |
| .NET 9 | No | Nov 2024 | May 2026 | net9.0 |
| **.NET 8** | **Sí** | Nov 2023 | Nov 2026 | net8.0 |
| .NET 7 | No | Nov 2022 | May 2024 | net7.0 |
| **.NET 6** | **Sí** | Nov 2021 | Nov 2024 | net6.0 |
| .NET 5 | No | Nov 2020 | May 2022 | net5.0 |
| .NET Core 3.1 | Sí | Dic 2019 | Dic 2022 | netcoreapp3.1 |
| .NET Framework 4.8.1 | Sí | Ago 2022 | Indefinido (Windows) | net481 |

**Regla de oro:** Siempre apuntar al último LTS (.NET 8 en 2025-2026). Solo usar versiones anteriores si el cliente/proyecto lo exige explícitamente.

## C# — Patrones y Convenciones

### Estructura de Archivos

```csharp
// ─── Usings fuera del namespace (C# 10+) ───
using System.Text;
using Microsoft.Extensions.Logging;

namespace MiEmpresa.MiApp.Feature;

// ─── 1. Delegados y tipos anidados ───
public delegate void DataChangedHandler(object sender, DataChangedEventArgs e);

// ─── 2. Clase principal ───
public sealed class ProcesadorDatos : IDisposable
{
    // 3a. Constantes y campos estáticos readonly
    private const int BufferSize = 4096;
    private static readonly char[] Separators = [' ', ',', ';']; // C# 12 collection expr

    // 3b. Campos privados (con underscore)
    private readonly ILogger<ProcesadorDatos> _logger;
    private bool _disposed;

    // 4. Constructor
    public ProcesadorDatos(ILogger<ProcesadorDatos> logger) => _logger = logger;

    // 5. Propiedades públicas
    public int TotalProcesado { get; private set; }

    // 6. Métodos públicos
    public async Task<Resultado> ProcesarAsync(Stream input, CancellationToken ct = default)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        ArgumentNullException.ThrowIfNull(input);

        // ... lógica

        return new Resultado(TotalProcesado);
    }

    // 7. IDisposable
    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}
```

### Reglas de Estilo (Estándar Microsoft + extras)

| Regla | Ejemplo |
|:------|:--------|
| `PascalCase` para públicos | `public class MiClase`, `public int Total` |
| `_camelCase` para privados | `private readonly ILogger _logger;` |
| `camelCase` para parámetros | `void Metodo(string nombreParam)` |
| `PascalCase` sin underscore en constantes | `private const int MaxRetries = 3;` |
| Nombres asíncronos con sufijo `Async` | `Task<Producto> ObtenerAsync()` |
| `var` cuando el tipo es obvio | `var lista = new List<int>();` |
| Tipo explícito cuando NO es obvio | `Dictionary<string, User> cache = ObtenerCache();` |
| Expression-bodied para una línea | `public int Suma => A + B;` |
| File-scoped namespaces (C# 10+) | `namespace MiApp.Feature;` |
| Primary constructors (C# 12+) para clases simples | `public class Punto(int x, int y);` |

### Nullabilidad (NRT — Nullable Reference Types)

**SIEMPRE habilitado en proyectos nuevos:**

```xml
<!-- .csproj -->
<Nullable>enable</Nullable>
<WarningsAsErrors>nullable</WarningsAsErrors>
```

```csharp
// ✓ BIEN — explícito
public string? DescripcionOpcional { get; set; }
public string Nombre { get; set; } = string.Empty;

// ✗ MAL — ambiguo (pre-C# 8)
public string Nombre { get; set; } // ¿nullable o no?
```

### Async/Await — Reglas de Oro

```csharp
// ✓ BIEN — async hasta arriba (no bloquear)
public async Task<Producto> ObtenerAsync(int id)
{
    return await _db.Productos.FindAsync(id);
}

// ✓ BIEN — ConfigureAwait(false) en librerías (hasta .NET 8)
public async Task<string> LeerArchivoAsync(string ruta)
{
    return await File.ReadAllTextAsync(ruta).ConfigureAwait(false);
}

// ✗ MAL — .Result / .Wait() → DEADLOCK
public Producto Obtener(int id) => ObtenerAsync(id).Result; // NUNCA

// ✗ MAL — async void (solo para event handlers)
public async void CargarDatos() { ... } // Solo en UI event handlers

// ✓ BIEN — CancellationToken siempre en APIs async
public async Task ProcesarAsync(CancellationToken ct = default)
{
    await Task.Delay(1000, ct); // Respeta cancelación
}
```

### Patrones Clave

**1. Result Pattern (en vez de excepciones para flujo):**

```csharp
public readonly record struct Resultado<T>
{
    public T? Valor { get; }
    public Error? Error { get; }
    public bool EsExito => Error is null;
    
    private Resultado(T valor) { Valor = valor; }
    private Resultado(Error error) { Error = error; }
    
    public static Resultado<T> Exito(T valor) => new(valor);
    public static Resultado<T> Falla(Error error) => new(error);
}
```

**2. Options Pattern (configuración tipada):**

```csharp
public class OpcionesApi
{
    public const string Seccion = "Api";
    
    [Required]
    public string UrlBase { get; init; } = string.Empty;
    
    [Range(1, 60)]
    public int TimeoutSegundos { get; init; } = 30;
}

// Program.cs
builder.Services
    .AddOptions<OpcionesApi>()
    .Bind(builder.Configuration.GetSection(OpcionesApi.Seccion))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

**3. Decorator Pattern con Scrutor (DI):**

```csharp
services.Decorate<IRepositorio, RepositorioConCache>();
services.Decorate<IRepositorio, RepositorioConLogging>();
```

### LINQ — Performance Tips

```csharp
// ✓ BIEN — evitar múltiples enumeraciones
var lista = query.ToList(); // Materializar una vez

// ✓ BIEN — métodos de extensión eficientes
if (items is List<T> list)
    return list.Exists(x => x.Activo); // List<T>.Exists, no allocs extra

// ✗ MAL — Count() en IEnumerable (posible O(n))
items.Count(); // ← Puede iterar toda la colección
// ✓ Usar .Count (propiedad) si es ICollection<T> o materializar antes

// ✓ BIEN — Span<T> para segmentos sin alloc
Span<byte> buffer = stackalloc byte[256]; // stack, zero alloc

// ✗ MAL — múltiples ToList() encadenados
items.Where(x => x.Activo).ToList().Select(x => x.Nombre).ToList();
// ✓ Una sola materialización
items.Where(x => x.Activo).Select(x => x.Nombre).ToList();
```

## WPF — Windows Presentation Foundation

### Arquitectura Recomendada: MVVM + CommunityToolkit.Mvvm

```xml
<!-- .csproj -->
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.*" />
<PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="8.*" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.*" />
```

### ViewModel Moderno (Source Generators)

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class MainViewModel : ObservableObject
{
    // [ObservableProperty] genera propiedad pública + OnXxxChanged partial
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(NombreCompleto))]
    private string _nombre = string.Empty;

    [ObservableProperty]
    private string _apellido = string.Empty;

    // Propiedad calculada
    public string NombreCompleto => $"{Nombre} {Apellido}".Trim();

    // [RelayCommand] genera IRelayCommand (con soporte CanExecute)
    [RelayCommand(CanExecute = nameof(PuedeGuardar))]
    private async Task GuardarAsync(CancellationToken ct)
    {
        // Lógica async automáticamente manejada (botón deshabilitado mientras)
        await Task.Delay(500, ct);
    }

    private bool PuedeGuardar() => !string.IsNullOrWhiteSpace(Nombre);
}
```

### XAML — Buenas Prácticas

```xml
<!-- ✓ Usar recursos y estilos -->
<Window.Resources>
    <Style x:Key="TextoLlamativo" TargetType="TextBlock">
        <Setter Property="FontSize" Value="16"/>
        <Setter Property="Foreground" Value="{StaticResource ColorPrimario}"/>
    </Style>
</Window.Resources>

<!-- ✓ Binding con FallbackValue y TargetNullValue -->
<TextBlock Text="{Binding Nombre, FallbackValue='Cargando...', TargetNullValue='Sin nombre'}"/>

<!-- ✓ DataTemplate por tipo (sin DataTemplateSelector si es simple) -->
<DataTemplate DataType="{x:Type vm:UsuarioViewModel}">
    <local:UsuarioControl/>
</DataTemplate>
```

### DI en WPF (Generic Host)

```csharp
// App.xaml.cs
public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        _host = Host.CreateDefaultBuilder()
            .ConfigureServices((context, services) =>
            {
                // ViewModels
                services.AddSingleton<MainViewModel>();
                services.AddTransient<DetalleViewModel>();

                // Views
                services.AddSingleton<MainWindow>();
                services.AddTransient<DetalleWindow>();

                // Servicios
                services.AddHttpClient<IApiService, ApiService>();
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();
        var mainWindow = _host.Services.GetRequiredService<MainWindow>();
        mainWindow.DataContext = _host.Services.GetRequiredService<MainViewModel>();
        mainWindow.Show();
        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.StopAsync();
        _host.Dispose();
        base.OnExit(e);
    }
}
```

### WPF Rendimiento — Checklist

- [ ] **Virtualización activada** — `VirtualizingStackPanel.IsVirtualizing="True"` (por defecto en ListBox/ListView)
- [ ] **Container Recycling** — `VirtualizingStackPanel.VirtualizationMode="Recycling"`
- [ ] **Deferred Scrolling** — `ScrollViewer.IsDeferredScrollingEnabled="True"`
- [ ] **Freezables congelados** — `SolidColorBrush` y geometrías como `Freezable.Freeze()`
- [ ] **Bindings con PresentationTraceSources** para debug en DEBUG
- [ ] **Evitar converter allocation** — Preferir `IValueConverter.Convert()` que no aloque

## ASP.NET Core

### Minimal API vs Controllers

| Criterio | Minimal API | Controllers |
|:---------|:-----------|:------------|
| Complejidad baja-media | ✓ Recomendado | Sobrecarga innecesaria |
| Complejidad alta (+20 endpoints) | Se complica | ✓ Recomendado |
| Microservicios | ✓ Ideal | Pesado |
| Swagger/OpenAPI | Con `WithOpenApi()` | ✓ Nativo |

**Minimal API moderna (C# 12):**

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddDbContext<AppDbContext>(o => o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IProductoService, ProductoService>();

var app = builder.Build();

app.MapGet("/api/productos", async (IProductoService svc, CancellationToken ct) =>
    Results.Ok(await svc.ObtenerTodosAsync(ct)))
   .WithName("GetProductos")
   .WithOpenApi()
   .Produces<Producto[]>(200)
   .Produces(500)
   .CacheOutput("Default30s");

app.MapPost("/api/productos", async (ProductoDto dto, IProductoService svc, CancellationToken ct) =>
{
    var producto = await svc.CrearAsync(dto, ct);
    return Results.Created($"/api/productos/{producto.Id}", producto);
})
   .WithName("CreateProducto")
   .WithOpenApi()
   .Produces<Producto>(201)
   .Produces<ValidationProblem>(400)
   .AddEndpointFilter<ValidationFilter<ProductoDto>>();

app.Run();
```

### Middleware Pipeline (Orden Correcto)

```
ExceptionHandler → HSTS → HttpsRedirection → StaticFiles
→ Routing → CORS → Authentication → Authorization
→ UseEndpoints (MVC/Minimal API)
```

### Entity Framework Core — Buenas Prácticas

```csharp
// ✓ UseSplitQueries para evitar explosión cartesiana
var ordenes = await _db.Ordenes
    .Include(o => o.Items).ThenInclude(i => i.Producto)
    .Include(o => o.Cliente)
    .AsSplitQuery() // ← Múltiples SELECTs, evita JOINs monstruosos
    .ToListAsync(ct);

// ✓ AsNoTracking para solo-lectura
var productos = await _db.Productos
    .AsNoTracking()
    .Where(p => p.Activo)
    .ToListAsync(ct);

// ✓ Compiled queries para queries calientes
private static readonly Func<AppDbContext, int, Task<Producto?>> ProductoPorId =
    EF.CompileAsyncQuery((AppDbContext ctx, int id) =>
        ctx.Productos.AsNoTracking().FirstOrDefault(p => p.Id == id));

// ✓ Proyecciones en vez de Include para solo-lectura
var resumen = await _db.Ordenes
    .Select(o => new OrdenResumenDto
    {
        o.Id,
        o.Cliente.Nombre,
        TotalItems = o.Items.Count,
        Total = o.Items.Sum(i => i.Cantidad * i.PrecioUnitario)
    })
    .ToListAsync(ct);
```

### Global Exception Handling (.NET 8+)

```csharp
// IExceptionHandler — nativo desde .NET 8
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext ctx, Exception ex, CancellationToken ct)
    {
        logger.LogError(ex, "Error no manejado: {Mensaje}", ex.Message);

        var (status, title) = ex switch
        {
            NotFoundException => (404, "Recurso no encontrado"),
            ValidationException => (400, "Error de validación"),
            UnauthorizedAccessException => (403, "Acceso denegado"),
            _ => (500, "Error interno del servidor")
        };

        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(
            new ProblemDetails { Status = status, Title = title }, ct);
        return true;
    }
}

// Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
```

## .NET MAUI

### Arquitectura

```xml
<!-- Shell-based navigation -->
<Shell x:Class="MiApp.AppShell">
    <TabBar>
        <ShellContent Title="Inicio" ContentTemplate="{DataTemplate local:MainPage}" Route="main"/>
        <ShellContent Title="Config" ContentTemplate="{DataTemplate local:ConfigPage}" Route="config"/>
    </TabBar>
</Shell>
```

```csharp
// Navegación tipada con CommunityToolkit.Maui
await Shell.Current.GoToAsync($"{nameof(DetallePage)}?id={producto.Id}");
```

### Performance MAUI

```xml
<!-- ListView → CollectionView (más rápido, virtualización) -->
<CollectionView ItemsSource="{Binding Productos}"
                RemainingItemsThreshold="5"
                RemainingItemsThresholdReached="OnLoadMore">
    <CollectionView.ItemTemplate>
        <DataTemplate x:DataType="vm:ProductoItemViewModel">
            <VerticalStackLayout Padding="10">
                <Label Text="{Binding Nombre}" FontSize="18"/>
                <Label Text="{Binding Precio, StringFormat='{0:C}'}"/>
            </VerticalStackLayout>
        </DataTemplate>
    </CollectionView.ItemTemplate>
</CollectionView>
```

## Rendimiento .NET — Checklist Universal

### Memoria

- [ ] `Span<T>` / `Memory<T>` en hot paths (stack alloc)
- [ ] `ArrayPool<T>` para buffers temporales
- [ ] `stackalloc` para arrays pequeños y temporales
- [ ] `StringBuilder` con capacidad inicial estimada
- [ ] Evitar boxing (genéricos en vez de `object`)
- [ ] `record struct` para DTOs pequeños (copia por valor, no heap)
- [ ] `IEnumerable<T>` no se materializa sin necesidad
- [ ] `string.Intern()` para strings repetidos

### Async

- [ ] `ValueTask` en vez de `Task` cuando es síncrono la mayoría de veces
- [ ] `ConfigureAwait(false)` en librerías (innecesario en .NET 8+ ASP.NET Core)
- [ ] `Task.WhenAll` para paralelismo real (no `await` en loop)
- [ ] `Channel<T>` para producer-consumer (mejor que `BlockingCollection`)
- [ ] `SemaphoreSlim` para throttling de async

### Diagnóstico

```bash
# dotnet-counters — rendimiento en vivo
dotnet-counters monitor --process-id <pid> System.Runtime

# dotnet-trace — perfilado
dotnet-trace collect --process-id <pid> --duration 00:00:30

# dotnet-dump — análisis de volcado
dotnet-dump collect --process-id <pid>
dotnet-dump analyze dump.dmp
> dumpheap -stat          # objetos por tipo
> dumpheap -type MiTipo   # inspeccionar tipo específico
> gcroot <addr>           # ¿quién referencia este objeto?
```

## Migración: .NET Framework → .NET Core/5+

### Plan de Migración

1. **Analizar dependencias** — `portability-analyzer` + `try-convert`
2. **Migrar .csproj a SDK-style** — `dotnet try-convert`
3. **Migrar libraries primero** (sin dependencias UI)
4. **Migrar tests** a `Microsoft.NET.Test.Sdk` + `xunit` / `nunit`
5. **Migrar ASP.NET** — `System.Web` → `Microsoft.AspNetCore`
6. **Migrar WPF** — .NET Core 3.1+ tiene soporte WPF (solo Windows)
7. **Migrar WinForms** — soporte desde .NET Core 3.1

```bash
# Herramientas de migración
dotnet tool install -g try-convert
try-convert --project MiProyecto.csproj

# Analizar compatibilidad
# https://github.com/microsoft/dotnet-apiport
```

## Errores Comunes (Antipatrones .NET)

1. **`async void`** fuera de event handlers → Crash silencioso de la app.
2. **`.Result` / `.Wait()`** en código async → Deadlock garantizado en UI threads.
3. **`string + string`** en loops → Usar `StringBuilder` o `string.Concat`.
4. **`List<T>`** como parámetro → Usar `IReadOnlyList<T>` o `IEnumerable<T>`.
5. **No hacer dispose** de `HttpClient` / `Stream` / `DbContext` → Memory leaks.
6. **`HttpClient` nuevo por request** → Socket exhaustion. Usar `IHttpClientFactory`.
7. **`throw ex`** en vez de `throw` → Pierde el stack trace original.
8. **`ConfigureAwait(false)`** en ASP.NET Core moderno → Ya no es necesario (.NET Core no tiene SynchronizationContext).
9. **`lock(this)`** o `lock(typeof(MiClase))` → Deadlocks ocultos. Usar `private readonly object _lock = new();`.
10. **UI thread bloqueado** en WPF/MAUI → Usar async o `Task.Run` + `Dispatcher.Invoke`.
11. **Lógica de negocio específica en templates/proyectos base** — Métodos como descuentos, precios, reglas de afiliación pertenecen a la aplicación concreta (ej. "LA OFRENDA"), no al template (NovaCore). El template solo debe contener infraestructura, utilidades compartidas y abstracciones genéricas. Cada aplicación que herede del template agrega su propia lógica de negocio. *Ver `references/nova-core-architecture.md` para el detalle completo de la arquitectura del proyecto.*

## Comandos Útiles

```bash
# Nuevo proyecto
dotnet new wpf -n MiApp          # WPF
dotnet new maui -n MiApp         # MAUI
dotnet new webapi -n MiApi       # ASP.NET Core Web API
dotnet new blazor -n MiApp       # Blazor WebAssembly

# Build y test
dotnet build -c Release
dotnet test --filter "FullyQualifiedName~MiTest" -l "console;verbosity=detailed"
dotnet test --collect:"XPlat Code Coverage"

# Análisis estático
dotnet format --verify-no-changes
dotnet format analyzers --verify-no-changes

# NuGet
dotnet list package --outdated
dotnet list package --vulnerable
dotnet nuget locals all --clear

# Benchmarking
dotnet run -c Release -- --filter "*MiBenchmark*"

# Publicación
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true
dotnet publish -c Release -r linux-x64 -p:PublishAot=true  # Native AOT (solo APIs compatibles)
```

## Verificación de Calidad

- [ ] Nullable Reference Types habilitados y sin warnings
- [ ] `dotnet format --verify-no-changes` pasa limpio
- [ ] EditorConfig presente con reglas de estilo consistentes
- [ ] Tests unitarios para lógica de negocio (xUnit / NUnit)
- [ ] `dotnet test` pasa al 100%
- [ ] Sin `.Result` / `.Wait()` en código async
- [ ] `IHttpClientFactory` en vez de `new HttpClient()`
- [ ] `IDisposable` implementado en clases con recursos no manejados
- [ ] `ConfigureAwait(false)` solo en librerías (si < .NET 8)
- [ ] Sin `async void` fuera de event handlers de UI
- [ ] Colecciones expuestas como `IReadOnlyList<T>` (no `List<T>`)
- [ ] Opciones de configuración con `IOptions<T>` + validación
- [ ] Health checks configurados en ASP.NET Core

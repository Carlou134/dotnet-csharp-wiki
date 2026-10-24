# Inyección de dependencias y ciclos de vida

## En una frase

Con inyección de dependencias, cada clase declara en su constructor lo que necesita y el contenedor de .NET lo crea y lo entrega; el ciclo de vida (*singleton*, *scoped*, *transient*) **no se elige por costumbre** sino según el estado que guarda el servicio y lo que necesitan sus dependencias.

-----

## Antes de empezar

Conviene que ya sepas:

* Interfaces y constructores: [POO](../../01-csharp-core-and-runtime/04-poo/README.md).
* Registro de servicios en `Program.cs`: [Host, configuración y pipeline HTTP](../00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).
* Controladores delgados: [Controladores y rutas por atributo](01-Controladores%20y%20rutas%20por%20atributo.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Inyección de dependencias (DI):** técnica en la que una clase recibe sus dependencias desde afuera (normalmente por constructor) en lugar de crearlas con `new`.
* **Contenedor de DI:** componente que sabe cómo construir cada servicio registrado y sus dependencias.
* **Ciclo de vida (*lifetime*):** cuánto vive una instancia: toda la aplicación, una petición o una resolución.
* **Scope:** unidad de vida para los servicios *scoped*; en ASP.NET Core, una petición HTTP.
* **Dependencia cautiva (*captive dependency*):** servicio de vida corta atrapado dentro de uno de vida más larga.
* **Service locator:** antipatrón en el que una clase pide servicios al contenedor en lugar de recibirlos.

-----

## El problema

Un equipo aprende dos reglas: "en Web API, *scoped* por defecto" y "*singleton* hay que evitarlo". Las aplica a todo:

```csharp
builder.Services.AddScoped<ICacheDeCotizaciones, CacheDeCotizaciones>();  // guarda cotizaciones 10 minutos
builder.Services.AddScoped<IContadorDeVisitas, ContadorEnMemoria>();
builder.Services.AddScoped<IPedidoRepository, PedidoRepository>();       // usa el DbContext
```

Lo que pasa:

1. La caché **nunca devuelve nada**: cada petición recibe una instancia nueva y vacía. Cada request llama al proveedor de cotizaciones, que empieza a responder `429 Too Many Requests`.
2. El contador de visitas siempre vale 1.
3. Alguien "lo arregla" pasando todo a *singleton*. La caché funciona, pero `PedidoRepository` ahora vive para siempre con un `DbContext` que debía morir al final de cada petición. En Development la app ni arranca; en producción, errores intermitentes de concurrencia.

Las consecuencias:

* servicios con estado que pierden el estado en cada petición;
* servicios sin estado compartidos de forma insegura entre hilos;
* reglas memorizadas que no explican **por qué**.

-----

## Cómo funciona

### 1. El problema que resuelve DI

```csharp
// ❌ La clase decide qué implementación usar y cómo construirla
public sealed class PedidoService
{
    private readonly PedidoRepository _repo = new(new AppDbContext(...));
}

// ✔ La clase declara qué necesita; quién lo construye es problema de otro
public sealed class PedidoService(IPedidoRepository repo) { }
```

Con `new`, `PedidoService` conoce la cadena de conexión, el tipo concreto del repositorio y cómo armarlo. No puedes probarlo sin base de datos ni cambiar la implementación sin editarlo. Con inyección por constructor, solo depende del contrato.

### 2. Registrar y resolver

```csharp
builder.Services.AddScoped<IPedidoRepository, PedidoRepository>();   // "cuando pidan IPedidoRepository, crea un PedidoRepository"
builder.Services.AddScoped<PedidoService>();                          // sin interfaz también se puede
```

```text
Petición HTTP → el controlador necesita PedidoService
                  → PedidoService necesita IPedidoRepository
                        → PedidoRepository necesita AppDbContext
                              → AppDbContext necesita DbContextOptions
El contenedor arma el grafo completo, de abajo hacia arriba.
```

### 3. Los tres ciclos de vida

| Lifetime | Una instancia por... | Elígelo cuando el servicio... | Ejemplos |
| --- | --- | --- | --- |
| Singleton | Aplicación | Guarda estado compartido **seguro entre hilos**, o no guarda estado y es caro de crear | Cachés, `IHttpClientFactory`, `TimeProvider`, configuración |
| Scoped | Petición HTTP | Guarda estado **de la petición** o depende de algo scoped | `DbContext`, repositorios sobre EF Core, usuario actual |
| Transient | Resolución | No guarda estado y es liviano | Validadores, calculadoras, mapeadores |

La pregunta no es "¿cuál se usa en APIs?", sino:

```text
¿Guarda estado?
 ├─ No ──────────────────────────► transient (o singleton si es caro de crear y seguro entre hilos)
 └─ Sí ─┬─ ¿De toda la app? ──────► singleton, y debe ser seguro entre hilos
        └─ ¿De la petición? ──────► scoped
Y además: nunca más largo que la dependencia más corta que tenga.
```

**Sobre los mitos:**

* "*Singleton* hay que evitarlo": falso. Es el lifetime correcto para cachés y clientes compartidos. Lo que hay que evitar es un singleton **con estado mutable sin protección** o que **dependa de algo scoped**.
* "*Transient* es ineficiente": crear un objeto liviano cuesta nanosegundos. El riesgo real de transient es otro: si implementa `IDisposable` y se resuelve desde el contenedor raíz, se guarda hasta que la aplicación termina.

### 4. La regla de oro: no capturar vidas cortas

```text
Singleton (vive siempre)
  └── Scoped (debía morir al final de la petición)  ❌ queda atrapado para siempre
```

En Development, ASP.NET Core activa `ValidateScopes` y `ValidateOnBuild`, y detecta el error al arrancar:

```text
Cannot consume scoped service 'IPedidoRepository' from singleton 'ICacheDeCotizaciones'.
```

En producción esa validación está apagada por defecto (cuesta tiempo de arranque): el error no aparece, la dependencia queda cautiva y falla de formas raras. Por eso conviene probar el arranque en Development.

### 5. Usar algo scoped desde un singleton

Si un singleton (por ejemplo, un `BackgroundService`) necesita un servicio scoped, crea un scope por unidad de trabajo:

```csharp
public sealed class LimpiezaNocturna(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await using var scope = scopes.CreateAsyncScope();
        var repo = scope.ServiceProvider.GetRequiredService<IPedidoRepository>();
        await repo.BorrarBorradoresViejosAsync(ct);
    }   // el scope descarta el repositorio y su DbContext
}
```

### 6. ¿Siempre una interfaz?

| La interfaz aporta cuando... | Sobra cuando... |
| --- | --- |
| Hay o habrá varias implementaciones (pagos con distintos proveedores) | Hay una sola implementación sin dependencias externas |
| Es un límite con infraestructura (base, HTTP, archivos, reloj) que quieres reemplazar en pruebas | Es lógica pura que se prueba directamente con su clase |
| Es un contrato entre capas o módulos | Solo duplica la firma de cada método |

"Siempre interfaz + clase" no es una ley. Una clase concreta también se registra e inyecta (`AddScoped<PedidoService>()`).

### 7. Registros con fábrica

```csharp
builder.Services.AddScoped<IServicioSaludo>(sp =>
    new ServicioSaludo(sp.GetRequiredService<ILogger<ServicioSaludo>>(), idioma: "es"));
```

Una fábrica es legítima cuando la construcción necesita lógica o valores que el contenedor no conoce. Lo que **no** conviene es una fábrica que hace `new` de las dependencias en lugar de pedirlas a `sp`: esas instancias quedan fuera del control del contenedor (lifetime, *dispose*, configuración).

### 8. Varios registros, claves y `TryAdd`

```csharp
builder.Services.AddScoped<INotificador, NotificadorEmail>();
builder.Services.AddScoped<INotificador, NotificadorSms>();
// Pedir INotificador → el ÚLTIMO registrado (Sms)
// Pedir IEnumerable<INotificador> → ambos, en orden de registro

builder.Services.AddKeyedSingleton<IAlmacen, AlmacenS3>("archivos");   // .NET 8
public sealed class Subidas([FromKeyedServices("archivos")] IAlmacen almacen) { }

builder.Services.TryAddScoped<IReloj, RelojSistema>();   // solo registra si no hay otro
```

### 9. Antipatrón: service locator

```csharp
// ❌ Las dependencias reales quedan ocultas
public sealed class PedidoService(IServiceProvider sp)
{
    public void Crear() => sp.GetRequiredService<IPedidoRepository>().Agregar(...);
}
```

El constructor ya no dice qué necesita la clase; los errores de registro aparecen en tiempo de ejecución y las pruebas tienen que armar un contenedor. Pide las dependencias por constructor. La excepción razonable es infraestructura como el `IServiceScopeFactory` del punto 5.

-----

## Ejemplo completo

Consola que muestra qué instancia recibe cada consumidor. `Lifetimes.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.DependencyInjection" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Microsoft.Extensions.DependencyInjection;

var services = new ServiceCollection();
services.AddSingleton<Reloj>();
services.AddScoped<Carrito>();
services.AddTransient<CalculadoraDescuento>();
services.AddScoped<ServicioPedidos>();
services.AddScoped<Auditoria>();

using var provider = services.BuildServiceProvider();

// Cada scope simula una petición HTTP
for (int peticion = 1; peticion <= 2; peticion++)
{
    using var scope = provider.CreateScope();
    var pedidos = scope.ServiceProvider.GetRequiredService<ServicioPedidos>();
    var auditoria = scope.ServiceProvider.GetRequiredService<Auditoria>();

    Console.WriteLine($"Petición {peticion}: pedidos   -> carrito #{pedidos.Carrito.Id}, calculadora #{pedidos.Calculadora.Id}, reloj #{pedidos.Reloj.Id}");
    Console.WriteLine($"            auditoría -> carrito #{auditoria.Carrito.Id}, calculadora #{auditoria.Calculadora.Id}");
}

// Dependencia cautiva: un singleton que pide un scoped
var mal = new ServiceCollection();
mal.AddScoped<Carrito>();
mal.AddSingleton<CacheDePrecios>();

using var providerValidado = mal.BuildServiceProvider(new ServiceProviderOptions { ValidateScopes = true });
try
{
    providerValidado.GetRequiredService<CacheDePrecios>();
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Error: {ex.Message}");
}

public sealed class Reloj
{
    private static int _creados;
    public int Id { get; } = Interlocked.Increment(ref _creados);
}

public sealed class Carrito
{
    private static int _creados;
    public int Id { get; } = Interlocked.Increment(ref _creados);
}

public sealed class CalculadoraDescuento
{
    private static int _creadas;
    public int Id { get; } = Interlocked.Increment(ref _creadas);
}

public sealed class ServicioPedidos(Carrito carrito, CalculadoraDescuento calculadora, Reloj reloj)
{
    public Carrito Carrito { get; } = carrito;
    public CalculadoraDescuento Calculadora { get; } = calculadora;
    public Reloj Reloj { get; } = reloj;
}

public sealed class Auditoria(Carrito carrito, CalculadoraDescuento calculadora)
{
    public Carrito Carrito { get; } = carrito;
    public CalculadoraDescuento Calculadora { get; } = calculadora;
}

public sealed class CacheDePrecios(Carrito carrito)
{
    public Carrito Carrito { get; } = carrito;
}
```

Salida:

```text
Petición 1: pedidos   -> carrito #1, calculadora #1, reloj #1
            auditoría -> carrito #1, calculadora #2
Petición 2: pedidos   -> carrito #2, calculadora #3, reloj #1
            auditoría -> carrito #2, calculadora #4
Error: Cannot consume scoped service 'Carrito' from singleton 'CacheDePrecios'.
```

Qué observar:

* **Scoped:** dentro de la petición 1, pedidos y auditoría comparten el carrito #1; la petición 2 recibe otro (#2).
* **Transient:** cada consumidor recibió su propia calculadora, incluso dentro de la misma petición (#1 y #2).
* **Singleton:** el reloj #1 es el mismo en las dos peticiones.
* Con `ValidateScopes`, el contenedor rechazó el singleton que pedía un scoped. ASP.NET Core activa esta validación en Development; aquí se activó a mano porque es una consola.

-----

## Errores comunes

**1. Registrar como scoped un servicio que guarda estado de toda la aplicación.**
Qué pasa: cachés que nunca devuelven nada, contadores que siempre valen 1.
Por qué: cada petición recibe una instancia nueva.
Arreglo: singleton seguro entre hilos (`ConcurrentDictionary`, `lock`, `IMemoryCache`).

**2. Singleton que depende de un scoped.**
Qué pasa: `InvalidOperationException: Cannot consume scoped service ... from singleton ...` en Development; dependencia cautiva silenciosa en producción.
Por qué: el singleton se construye una vez y retiene su dependencia para siempre.
Arreglo: bajar el lifetime del consumidor o crear un scope con `IServiceScopeFactory` por unidad de trabajo.

**3. Singleton con estado mutable sin sincronizar.**
Qué pasa: datos corruptos o `InvalidOperationException: Collection was modified` bajo carga.
Por qué: todas las peticiones concurrentes usan la misma instancia.
Arreglo: colecciones concurrentes, `lock` o rediseñar sin estado mutable.

**4. Fábricas que hacen `new` de las dependencias.**
Qué pasa: instancias que nunca se descartan, configuración duplicada y lifetimes ignorados.
Por qué: lo que creas con `new` dentro de la fábrica no lo administra el contenedor.
Arreglo: pide las dependencias con `sp.GetRequiredService<T>()`.

**5. Inyectar `IServiceProvider` para resolver todo a mano.**
Qué pasa: dependencias ocultas, errores en ejecución y pruebas que necesitan un contenedor.
Por qué: service locator.
Arreglo: dependencias explícitas en el constructor.

**6. Transient `IDisposable` resuelto desde el proveedor raíz.**
Qué pasa: memoria que crece: el contenedor guarda cada instancia para descartarla al final, y el final es el apagado de la app.
Por qué: el proveedor raíz es un scope que vive toda la aplicación.
Arreglo: resuelve dentro de un scope, o no registres como transient objetos desechables que se piden muchas veces.

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** contenedor de DI integrado (`Microsoft.Extensions.DependencyInjection`).
* **ASP.NET Core 2.0:** `ValidateScopes` activado por defecto en Development.
* **.NET Core 3.0:** `ValidateOnBuild`, que valida todos los registros al construir el proveedor.
* **.NET 6:** `CreateAsyncScope` para descartar servicios `IAsyncDisposable`; Minimal APIs que reciben servicios de DI como parámetros del handler.
* **.NET 8:** *keyed services* (`AddKeyedScoped`, `[FromKeyedServices]`).
* **C# 12:** constructores primarios: inyección por constructor en una línea.

-----

## Cuándo sí y cuándo no

**Usa DI cuando:** una clase depende de infraestructura (base, HTTP, archivos, reloj) o de servicios con su propio ciclo de vida. Es el modo normal de trabajar en ASP.NET Core.

**No inyectes cuando:** es un valor o un objeto de dominio (`new Pedido(...)`, `new Dinero(...)`). Las entidades y value objects se crean con `new`; DI es para servicios.

**No elijas lifetime por regla fija:** decide por el estado que guarda el servicio y por sus dependencias.

-----

## Resumen en 5 líneas

1. DI: la clase declara lo que necesita en el constructor; el contenedor arma el grafo.
2. Singleton para estado compartido seguro o servicios caros sin estado; scoped para estado de la petición; transient para servicios livianos sin estado.
3. Nunca un lifetime más largo que el de sus dependencias: un singleton no puede depender de un scoped.
4. Interfaces cuando aportan (límites, varias implementaciones, pruebas), no por ritual.
5. Fábricas sí, `new` de dependencias dentro de ellas no; service locator tampoco.

-----

## Para profundizar

<details>
<summary>Cómo resuelve el contenedor un singleton concurrente</summary>

Si dos peticiones piden un singleton que todavía no existe al mismo tiempo, el contenedor sincroniza la creación: el constructor corre una sola vez. Eso protege la **creación**, no el uso: los métodos del singleton siguen ejecutándose concurrentemente desde todas las peticiones.

</details>

<details>
<summary>Contenedores de terceros</summary>

Autofac, Lamar o DryIoc agregan módulos, interceptores, registro por convenciones avanzadas y decoradores. Para registro por escaneo y decoradores sobre el contenedor de Microsoft, Scrutor suele alcanzar. Antes de cambiar de contenedor, pregúntate qué función concreta te falta: el integrado cubre la gran mayoría de las aplicaciones.

</details>

-----

## En entrevista

### Respuesta corta (junior)

Inyección de dependencias es recibir las dependencias por constructor en vez de crearlas con `new`. En ASP.NET Core se registran en `builder.Services` con tres lifetimes: singleton (una instancia para toda la app), scoped (una por petición) y transient (una cada vez que se pide).

### Respuesta ampliada (semi-senior)

Elijo el lifetime por el estado del servicio: singleton para cachés o clientes compartidos, siempre seguros entre hilos; scoped para `DbContext` y lo que depende de él; transient para servicios livianos sin estado. La regla que no se rompe es que un servicio no viva más que sus dependencias: un singleton que captura un scoped es una dependencia cautiva, que `ValidateScopes` detecta en Development. Desde singletons o `BackgroundService` creo scopes con `IServiceScopeFactory`. Uso interfaces en límites de infraestructura o con varias implementaciones, evito el service locator y conozco keyed services, `TryAdd` y la resolución de `IEnumerable<T>`.

### Preguntas frecuentes de seguimiento

**1. ¿Por qué el `DbContext` es scoped?**
Porque guarda estado de una unidad de trabajo (change tracker) y no es seguro entre hilos.

**2. ¿Un scoped puede depender de un singleton?**
Sí: lo contrario es el problema.

**3. ¿Qué devuelve el contenedor si registré dos implementaciones?**
La última al pedir `T`; todas al pedir `IEnumerable<T>`.

**4. ¿Cómo uso un servicio scoped en un `BackgroundService`?**
Creando un scope con `IServiceScopeFactory.CreateAsyncScope()` en cada unidad de trabajo.

-----

## Práctica

**Ejercicio 1.** Elige el lifetime de cada servicio y justifícalo: (a) `TasaDeCambioCache` que guarda tasas 10 minutos; (b) `UsuarioActual` que lee los claims del token de la petición; (c) `CalculadoraImpuestos` sin estado; (d) `FacturaRepository` que usa `AppDbContext`.

<details>
<summary>Solución</summary>

* (a) **Singleton**, con almacenamiento seguro entre hilos (`IMemoryCache` o `ConcurrentDictionary`): el estado es de toda la aplicación.
* (b) **Scoped**: el usuario cambia en cada petición.
* (c) **Transient** (o singleton, si fuera caro de crear): no guarda estado.
* (d) **Scoped**: depende del `DbContext`, que es scoped; no puede vivir más que él.

</details>

**Ejercicio 2.** Este singleton falla al arrancar en Development. Arréglalo sin cambiar su lifetime.

```csharp
public sealed class ResumenDiario(AppDbContext db) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        int total = await db.Pedidos.CountAsync(ct);
    }
}
```

<details>
<summary>Solución</summary>

```csharp
public sealed class ResumenDiario(IServiceScopeFactory scopes) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await using var scope = scopes.CreateAsyncScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        int total = await db.Pedidos.CountAsync(ct);
    }
}
```

Los hosted services son singleton. En lugar de capturar el `DbContext`, el servicio crea un scope por ejecución; al terminar el `await using`, el contexto se descarta. `IDbContextFactory<AppDbContext>` es una alternativa igual de válida.

</details>

-----

## Siguiente lección

[Middleware personalizado](03-Middleware%20personalizado.md)

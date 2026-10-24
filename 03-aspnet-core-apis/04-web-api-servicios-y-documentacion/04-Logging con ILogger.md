# Logging con ILogger

## En una frase

`ILogger<T>` es la API de logging de .NET: escribes mensajes con **plantillas** y niveles, la configuración decide qué niveles se guardan por categoría, y un *provider* (consola, Serilog, OpenTelemetry) decide **dónde** van, sin cambiar tu código.

-----

## Antes de empezar

Conviene que ya sepas:

* Inyección de dependencias: [Inyección de dependencias y ciclos de vida](02-Inyeccion%20de%20dependencias%20y%20ciclos%20de%20vida.md).
* Configuración por entorno (`appsettings.{Environment}.json`): [Host, configuración y pipeline HTTP](../00-fundamentos-de-aspnet-core/01-Host%20configuracion%20y%20pipeline%20HTTP.md).

Palabras nuevas (también están en el [Glosario](Glosario.md)):

* **Categoría:** nombre que identifica quién escribe el log; con `ILogger<T>`, es el nombre completo de `T`.
* **Nivel de log:** severidad del mensaje, de `Trace` (0) a `Critical` (5).
* **Plantilla de mensaje (*message template*):** texto con marcadores con nombre (`{PedidoId}`) cuyos valores se guardan como propiedades separadas.
* **Log estructurado:** registro que guarda cada valor como campo consultable, no solo como texto.
* **Provider (proveedor):** componente que envía los logs a un destino: consola, archivo, plataforma de logs.
* **Sink:** en Serilog, el destino de los logs (archivo, Seq, Elasticsearch, base de datos).

-----

## El problema

Un servicio de pedidos registra así:

```csharp
_logger.LogInformation($"Pedido {pedido.Id} creado para {cliente.Email} con tarjeta {pago.NumeroTarjeta}");
_logger.LogInformation($"Calculando descuento para el pedido {pedido.Id}");
_logger.LogInformation($"Error: {ex.Message}");
```

Y para "tener trazabilidad", un *provider* casero inserta cada log en una tabla `Logs` de la misma base de datos, en cada petición.

Lo que pasa:

1. **Datos sensibles en los logs.** Emails y números de tarjeta terminan en archivos y tablas que mucha más gente puede leer que la base de clientes. Eso es un incidente de seguridad y un problema legal.
2. **No se puede buscar.** "Todos los logs del pedido 42" exige buscar texto. Con interpolación, el `42` está pegado al mensaje; no existe un campo `PedidoId`.
3. **Todo es `Information`.** El detalle de cálculo (que debía ser `Debug`) y los errores (que debían ser `Error`) tienen el mismo nivel: imposible filtrar.
4. **La excepción se pierde.** `ex.Message` sin el objeto `ex` descarta el tipo y el *stack trace*.
5. **La tabla de logs empeora la caída.** Cuando la base se pone lenta, cada log también es lento; cuando se cae, los logs que explicarían la caída **no se pueden escribir**.

Las consecuencias:

* filtración de datos personales;
* logs que no responden preguntas concretas;
* un sistema de diagnóstico que falla junto con lo que debía diagnosticar.

-----

## Cómo funciona

### 1. `ILogger<T>` y la categoría

```csharp
public sealed class PedidoService(ILogger<PedidoService> logger)
{
    public void Crear(int pedidoId) => logger.LogInformation("Pedido {PedidoId} creado", pedidoId);
}
```

ASP.NET Core ya registra el logging: solo pides `ILogger<T>` por constructor. La categoría es `MiApp.Servicios.PedidoService`, y la configuración puede filtrar por ese nombre o por cualquier prefijo (`MiApp.Servicios`).

### 2. Niveles

| Nivel | Valor | Para | ¿En producción? |
| --- | --- | --- | --- |
| `Trace` | 0 | Detalle extremo (cada paso, datos internos) | No |
| `Debug` | 1 | Diagnóstico durante el desarrollo | Normalmente no |
| `Information` | 2 | Hechos del flujo normal: "pedido creado" | Sí |
| `Warning` | 3 | Algo inesperado que no impidió seguir: reintento, valor por defecto | Sí |
| `Error` | 4 | Una operación falló | Sí |
| `Critical` | 5 | La aplicación no puede seguir: sin espacio en disco, sin base | Sí |
| `None` | 6 | Desactiva el log de esa categoría | — |

Un nivel mínimo deja pasar ese nivel **y los superiores**. Con `Information`, no salen `Debug` ni `Trace`.

### 3. Configuración por categoría y entorno

```json
// appsettings.json (producción)
"Logging": {
  "LogLevel": {
    "Default": "Information",
    "Microsoft.AspNetCore": "Warning",
    "Microsoft.EntityFrameworkCore.Database.Command": "Warning"
  }
}
```

```json
// appsettings.Development.json
"Logging": {
  "LogLevel": {
    "Default": "Debug",
    "Microsoft.EntityFrameworkCore.Database.Command": "Information"
  }
}
```

* Gana la regla con el prefijo **más largo** que coincide con la categoría.
* En Development ves el SQL que genera EF Core; en producción, no.
* Cambiar el nivel es cambiar configuración, no código. Con `reloadOnChange` (por defecto en `appsettings.json`), incluso sin reiniciar.

### 4. Plantillas, no interpolación

```csharp
// ❌ Interpolación: un solo texto, sin campos
logger.LogInformation($"Pedido {pedidoId} creado para el cliente {clienteId}");

// ✔ Plantilla: texto + propiedades PedidoId y ClienteId
logger.LogInformation("Pedido {PedidoId} creado para el cliente {ClienteId}", pedidoId, clienteId);
```

```text
Con plantilla, una plataforma de logs guarda:
  mensaje:   "Pedido 42 creado para el cliente 7"
  plantilla: "Pedido {PedidoId} creado para el cliente {ClienteId}"
  PedidoId:  42
  ClienteId: 7
→ puedes consultar PedidoId = 42, o contar logs por plantilla.
```

Además, con interpolación el `string` se arma **aunque el nivel esté desactivado**; con plantilla, no. El analizador **CA2254** advierte cuando la plantilla no es constante.

Los valores se asocian a los marcadores **por posición**, no por nombre: `{PedidoId}` toma el primer argumento.

### 5. Excepciones

```csharp
catch (Exception ex)
{
    logger.LogError(ex, "No se pudo confirmar el pedido {PedidoId}", pedidoId);
    throw;
}
```

El primer parámetro es la excepción completa: tipo, mensaje, *stack trace* y excepciones internas. Y no registres **y** relances en cada capa: terminarías con el mismo error cinco veces. Regístralo donde se maneja (por ejemplo, en el `IExceptionHandler` global).

### 6. Alto rendimiento: `[LoggerMessage]`

```csharp
public static partial class Logs
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Pedido {PedidoId} cancelado por {Motivo}")]
    public static partial void PedidoCancelado(this ILogger logger, int pedidoId, string motivo);
}

logger.PedidoCancelado(42, "cliente");
```

Un generador de código crea el método: sin *boxing* de los valores, sin analizar la plantilla en cada llamada y con chequeo de nivel incluido. Además, los logs quedan centralizados, con `EventId` estables y parámetros tipados. Es la forma recomendada en código que se ejecuta mucho.

### 7. Dónde terminan los logs

| Destino | Cómo | Para |
| --- | --- | --- |
| Consola | Incluido (`AddConsole`, `AddJsonConsole`) | Desarrollo y contenedores (el orquestador recoge la salida estándar) |
| Serilog / NLog | Se conectan como *provider* de `ILogger` | Muchos destinos (*sinks*), enriquecimiento, formato |
| Plataforma de logs (Seq, Elasticsearch, Loki, Application Insights) | *Sink* de Serilog u OpenTelemetry (OTLP) | Búsqueda, alertas, retención, correlación con trazas |
| Tabla en tu base de datos | *Sink* de base de datos | Auditoría de negocio puntual, con cuidado |

Tu código sigue usando `ILogger`; cambiar el destino es configuración. Ojo con los nombres: **Serilog** y **NLog** son bibliotecas de logging; **Seq** es un servidor donde se guardan y consultan logs.

**Sobre la tabla de logs en la misma base:** se puede, pero tiene costos. Cada log compite con las consultas del negocio, y cuando la base falla perdés justo los logs que explican la falla. Si necesitas logs consultables, usa una plataforma de logs. Si necesitas **auditoría** ("quién cambió qué precio"), eso es un requisito de negocio: modélalo como datos propios, no como logs.

### 8. Qué no registrar

* Contraseñas, tokens, números de tarjeta, claves de API: **nunca**.
* Datos personales (email, documento, dirección): solo si es imprescindible, y enmascarados.
* Cuerpos completos de peticiones y respuestas: casi nunca; si hace falta, con `HttpLogging` y campos filtrados.

.NET 8 agregó `Microsoft.Extensions.Compliance.Redaction` para marcar datos sensibles y que se enmascaren automáticamente en los logs.

-----

## Ejemplo completo

Consola con el *provider* de consola y filtros por categoría. `Logging.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Logging.Console" Version="10.0.0" />
  </ItemGroup>
</Project>
```

`Program.cs`:

```csharp
using Microsoft.Extensions.Logging;

// El using descarta la fábrica al final: así se vacía la cola del logger de consola
using var factory = LoggerFactory.Create(builder => builder
    .AddSimpleConsole()
    .SetMinimumLevel(LogLevel.Information)        // por defecto: Information o superior
    .AddFilter("Pagos", LogLevel.Warning));       // la categoría Pagos: solo Warning o superior

ILogger pedidos = factory.CreateLogger("Pedidos");
ILogger pagos = factory.CreateLogger("Pagos");

pedidos.LogDebug("Calculando descuentos del pedido {PedidoId}", 42);                 // filtrado: Debug < Information
pedidos.LogInformation("Pedido {PedidoId} creado para el cliente {ClienteId}", 42, 7);
pagos.LogInformation("Pago del pedido {PedidoId} iniciado", 42);                    // filtrado: Pagos exige Warning
pagos.LogWarning("Pago del pedido {PedidoId} reintentado {Intentos} veces", 42, 2);
pedidos.LogError(new InvalidOperationException("Stock insuficiente"), "No se pudo confirmar el pedido {PedidoId}", 42);
pedidos.PedidoCancelado(42, "cliente");

public static partial class Logs
{
    [LoggerMessage(EventId = 1001, Level = LogLevel.Information, Message = "Pedido {PedidoId} cancelado por {Motivo}")]
    public static partial void PedidoCancelado(this ILogger logger, int pedidoId, string motivo);
}
```

Salida:

```text
info: Pedidos[0]
      Pedido 42 creado para el cliente 7
warn: Pagos[0]
      Pago del pedido 42 reintentado 2 veces
fail: Pedidos[0]
      No se pudo confirmar el pedido 42
      System.InvalidOperationException: Stock insuficiente
info: Pedidos[1001]
      Pedido 42 cancelado por cliente
```

Qué observar:

* Seis llamadas, cuatro logs: el `Debug` quedó por debajo del mínimo y el `Information` de `Pagos` por debajo de su filtro.
* Cada entrada muestra nivel abreviado (`info`, `warn`, `fail`), categoría y `EventId` entre corchetes. El método generado tiene el `EventId` 1001; los demás, 0.
* La excepción salió completa debajo del mensaje porque se pasó como primer parámetro, no como `ex.Message`. (Como no se lanzó, no tiene *stack trace*.)
* El formato de consola es para humanos. Para que una plataforma de logs reciba las propiedades `PedidoId` y `ClienteId` como campos, usa `AddJsonConsole()`, Serilog u OpenTelemetry.

-----

## Errores comunes

**1. Interpolar en lugar de usar plantillas.**
Qué pasa: logs sin propiedades consultables, `string` armado aunque el nivel esté apagado y advertencia CA2254.
Por qué: el *provider* recibe un texto ya terminado.
Arreglo: `LogInformation("Pedido {PedidoId}", id)`.

**2. Registrar datos sensibles.**
Qué pasa: tokens, contraseñas o tarjetas en archivos y plataformas con acceso amplio.
Por qué: los logs se copian, se retienen y se comparten mucho más que la base.
Arreglo: no registrar esos valores; enmascarar datos personales; redacción automática con `Microsoft.Extensions.Compliance.Redaction`.

**3. `LogError(ex.Message)`.**
Qué pasa: sin tipo de excepción, sin *stack trace*, sin excepciones internas.
Por qué: solo se registra un texto.
Arreglo: `LogError(ex, "Plantilla {Dato}", dato)`.

**4. Todo en `Information`.**
Qué pasa: producción llena de ruido, costos de almacenamiento y errores difíciles de encontrar.
Por qué: los niveles pierden significado.
Arreglo: `Debug` para detalle técnico, `Information` para hechos del negocio, `Warning`/`Error` para lo anormal.

**5. Escribir logs de forma síncrona en la base del negocio.**
Qué pasa: más latencia en cada petición y pérdida de logs justo cuando la base falla.
Por qué: el diagnóstico depende del mismo recurso que falla.
Arreglo: plataforma de logs con *sink* asíncrono o con *buffer*; auditoría de negocio como datos propios.

**6. Registrar y relanzar en cada capa.**
Qué pasa: el mismo error aparece varias veces, con *stack traces* parciales.
Por qué: cada `catch` intermedio vuelve a registrar.
Arreglo: registra donde manejas la excepción; en las capas intermedias, déjala pasar (o agrega contexto con scopes).

-----

## Según la versión de .NET

* **ASP.NET Core 1.0:** `Microsoft.Extensions.Logging`, `ILogger<T>` y providers integrados.
* **ASP.NET Core 2.0:** `LoggerMessage.Define` para logs de alto rendimiento y filtros por categoría en configuración.
* **.NET 5:** formateadores de consola (`AddSimpleConsole`, `AddJsonConsole`, `AddSystemdConsole`).
* **.NET 6:** generador de código `[LoggerMessage]` e `HttpLogging`.
* **.NET 8:** redacción de datos sensibles (`Microsoft.Extensions.Compliance.Redaction`) y `FakeLogger` para pruebas.

-----

## Cuándo sí y cuándo no

**Registra cuando:** ocurre un hecho del negocio relevante (pedido creado, pago rechazado), algo anormal (reintento, valor por defecto) o un error.

**Usa `Debug` cuando:** el detalle solo sirve mientras desarrollas o diagnosticas un problema concreto.

**No registres cuando:** el dato es sensible, el mensaje se repite miles de veces por segundo sin aportar información, o la necesidad real es una **métrica** (contar peticiones, medir latencias): para eso existen las métricas ([Métricas con Meter](../../02-architecture-and-design-patterns/04-observabilidad-con-opentelemetry/02-Metricas%20con%20Meter.md)).

-----

## Resumen en 5 líneas

1. Pide `ILogger<T>` por constructor; la categoría es el nombre de `T`.
2. Seis niveles; la configuración por categoría y entorno decide qué se guarda.
3. Plantillas con marcadores con nombre, nunca interpolación: dan campos consultables.
4. Excepciones como primer parámetro; `[LoggerMessage]` en código caliente.
5. El destino es configuración (consola, Serilog, OTLP); nunca registres datos sensibles.

-----

## Para profundizar

<details>
<summary>Serilog en ASP.NET Core en cinco líneas</summary>

```csharp
builder.Services.AddSerilog((services, config) => config
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.Seq("http://localhost:5341"));
```

Con el paquete `Serilog.AspNetCore` (y `Serilog.Sinks.Seq` para el destino Seq), Serilog reemplaza a los providers integrados. Tu código sigue usando `ILogger<T>`. `app.UseSerilogRequestLogging()` reemplaza los varios logs por petición de ASP.NET Core por uno solo con método, ruta, código y duración.

</details>

<details>
<summary>Scopes y correlación con trazas</summary>

`logger.BeginScope(...)` agrega propiedades (como `PedidoId`) a todos los logs escritos dentro del bloque. Combinado con OpenTelemetry, cada log lleva además el `TraceId` de la petición, y desde un error puedes saltar a la traza completa. Lo viste en [Logs estructurados y correlación](../../02-architecture-and-design-patterns/04-observabilidad-con-opentelemetry/03-Logs%20estructurados%20y%20correlacion.md).

</details>

-----

## En entrevista

### Respuesta corta (junior)

Uso `ILogger<T>` inyectado por constructor y elijo el nivel según la importancia: `Information` para hechos normales, `Warning` y `Error` para problemas. Los niveles se configuran en `appsettings.json`, y en producción normalmente desde `Information` para arriba.

### Respuesta ampliada (semi-senior)

Escribo logs con plantillas, no con interpolación, para que cada valor sea un campo consultable y no se arme el texto si el nivel está apagado; en código caliente uso `[LoggerMessage]`. Configuro niveles por categoría y entorno (por ejemplo, el SQL de EF Core solo en Development), registro excepciones como primer parámetro y una sola vez, donde se manejan. Envío los logs a una plataforma (Seq, Elasticsearch, OTLP) mediante Serilog u OpenTelemetry, sin acoplar el código al destino, y nunca registro datos sensibles. Distingo logs de métricas y de auditoría de negocio.

### Preguntas frecuentes de seguimiento

**1. ¿Cuál es el nivel más bajo?**
`Trace` (0). `None` (6) desactiva por completo.

**2. ¿Por qué no interpolar en un log?**
Porque se pierden las propiedades estructuradas y el texto se construye aunque el nivel esté desactivado.

**3. ¿Guardar logs en la base de datos es buena práctica?**
Puede servir para casos puntuales, pero acopla el diagnóstico al recurso que puede fallar. Lo habitual es una plataforma de logs.

**4. ¿Serilog reemplaza a `ILogger`?**
No en tu código: se conecta como provider y tú sigues escribiendo con `ILogger<T>`.

-----

## Práctica

**Ejercicio 1.** Corrige estas tres líneas.

```csharp
_logger.LogInformation($"Usuario {email} inició sesión con token {token}");
_logger.LogInformation($"Consulta SQL: {sql}");
catch (Exception ex) { _logger.LogError("Falló el pago: " + ex.Message); }
```

<details>
<summary>Solución</summary>

```csharp
_logger.LogInformation("Usuario {UsuarioId} inició sesión", usuario.Id);
_logger.LogDebug("Consulta SQL: {Sql}", sql);
catch (Exception ex) { _logger.LogError(ex, "Falló el pago del pedido {PedidoId}", pedidoId); }
```

* Fuera el email y el token: se identifica al usuario por su id interno.
* El SQL es detalle técnico: `Debug`, y con plantilla.
* La excepción va completa como primer parámetro, con un dato que permita buscar el caso.

</details>

**Ejercicio 2.** Escribe la configuración para que en producción solo se vean `Warning` o superiores de ASP.NET Core y EF Core, pero `Information` de tu código (espacio de nombres `Tienda`).

<details>
<summary>Solución</summary>

```json
"Logging": {
  "LogLevel": {
    "Default": "Warning",
    "Tienda": "Information"
  }
}
```

`Default` cubre todo lo que no tenga una regla más específica (incluidos `Microsoft.AspNetCore` y `Microsoft.EntityFrameworkCore`). La regla `Tienda` gana para tus categorías porque su prefijo es más largo y coincide con ellas.

</details>

-----

## Siguiente lección

[Documentación con OpenAPI](05-Documentacion%20con%20OpenAPI.md)
